# 10. Lakehouse Commit Logs: ACID on S3

*Source references: `delta-io/delta` and `apache/iceberg`*

How do you implement ACID transactions (Atomicity, Consistency, Isolation, Durability) on top of Amazon S3? 

S3 is highly durable and provides strong read-after-write consistency, but it does **not** provide cross-file transaction support or file locking. If two Spark clusters try to update a table simultaneously, the last one to overwrite the file wins, and data is corrupted.

Delta Lake, Apache Iceberg, and Apache Hudi all solve this using the exact same mathematical principle: **Optimistic Concurrency Control (OCC)** applied to a central transaction log.

---

## 1. The Structure of a Commit

When you insert data into a Delta Table, the Spark driver generates two things:
1. Parquet files containing the new rows (e.g., `part-001.parquet`). S3 blindly accepts these files.
2. A JSON log file representing the commit.

The transaction log directory (`_delta_log/`) is strictly ordered:
`000000000000.json` (Commit 0)
`000000000001.json` (Commit 1)
`000000000002.json` (Commit 2)

If a Spark job wants to append data, it reads the log to find the latest version (e.g., Version 2). It writes its Parquet files. Then, it attempts to write its commit log as `000000000003.json`.

---

## 2. Optimistic Concurrency Control (OCC)

What happens if Cluster A and Cluster B both read Version 2, write their own Parquet files, and *both* attempt to write `000000000003.json` to S3 at the exact same millisecond? S3 doesn't have an `atomic_increment` function. 

You must guarantee that only one writer succeeds, and the other fails.

**The Magic S3 Feature: `PutIfAbsent` (or Conditional Writes)**
In 2024, AWS introduced conditional writes to S3 (specifically `PutObject` with HTTP `If-None-Match: *`). Delta/Iceberg relies entirely on this physical mechanism.

If Cluster A's HTTP request arrives 1 ms before Cluster B's:
1. Cluster A's request creates `003.json`.
2. Cluster B's request receives `HTTP 412 Precondition Failed` because `003.json` already exists.

Because Delta uses **Optimistic** concurrency, Cluster B doesn't just crash. It does the following loop:

```java
// Conceptual Optimistic Concurrency Control Loop inside Delta/Iceberg

int attempt = 0;
while (attempt < MAX_RETRIES) {
    long currentVersion = deltaLog.getLatestVersion(); // e.g., Version 2
    long targetVersion = currentVersion + 1;           // Trying for Version 3
    
    // 1. Generate the logical Parquet append actions
    List<Action> actions = buildActions(dataframe);
    
    try {
        // 2. ATOMIC WRITE: Attempt to create exactly 003.json
        // On S3, this uses the If-None-Match conditional write!
        storageClient.putIfAbsent(
            path = "_delta_log/" + format(targetVersion) + ".json", 
            content = serialize(actions)
        );
        
        return "SUCCESS"; // Transaction Committed!
        
    } catch (FileAlreadyExistsException e) {
        // 3. Cluster A beat us! We lost the race for Version 3.
        
        // 4. Read what Cluster A did (download 003.json)
        Commit otherCommit = readLog(targetVersion);
        
        // 5. Mathematically verify if Cluster A's actions conflict with ours.
        // E.g., Did they DELETE the exact same partition we are modifying?
        if (conflicts(actions, otherCommit.actions)) {
            throw new ConcurrentModificationException("Transaction Failed!");
        }
        
        // No logical conflict! Re-run the while loop, aiming for Version 4.
        attempt++;
    }
}
```

### The Conflict Resolution Math

The `conflicts()` function is the core intellect of the Lakehouse format.

If Cluster A added new users to the `country=US` partition, and Cluster B simultaneously deleted users in the `country=EU` partition, they are writing to completely different Parquet files. The `conflicts()` function evaluates the bounding box metadata of the two commits, sees zero overlap, and allows Cluster B to immediately proceed to attempt `004.json` without recomputing its entire DataFrame.

This elegant use of Optimistic loops and S3 HTTP Preconditions is what transformed dumb object storage into enterprise-grade ACID Data Warehouses.
