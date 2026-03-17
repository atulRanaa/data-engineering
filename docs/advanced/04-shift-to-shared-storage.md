# 04. The Shift to Shared-Storage Architecture

For the entire decade spanning 2010 to 2020, the undisputed king of distributed database architecture was **Shared-Nothing**. Systems like Cassandra, Hadoop HDFS, MongoDB, and early CockroachDB all relied on the premise that compute and storage must be tightly coupled on the same physical server to avoid network bottlenecks.

By 2026, the industry has almost entirely abandoned Shared-Nothing in favor of **Shared-Storage** architectures (e.g., Snowflake, Amazon Aurora, Neon, Databricks Serverless).

To understand why this massive architectural shift occurred, we must examine the physics of modern networking and the economic realities of cloud computing.

---

## 1. The Flaw in Shared-Nothing

In a typical Shared-Nothing database (like Cassandra), every node has its own CPU, RAM, and attached SSD. If Node A holds rows 1-1,000, and Node B holds rows 1,001-2,000, they share nothing.

**The Problem: Elasticity is Impossible.**
Suppose your e-commerce site experiences a massive Black Friday traffic spike. Your CPUs are at 100% utilization. You need to add more compute power immediately.

In a Shared-Nothing architecture, you cannot add "just CPUs." You must add brand new servers (CPU + Disk). When the new server boots up, it is empty. The database must now pause and physically copy terabytes of data from the existing nodes over the network to the new node to rebalance the cluster. This process (resharding) can take **days**. By the time the cluster has scaled up, Black Friday is over.

---

## 2. The Shared-Storage Revolution

In a **Shared-Storage** architecture, Compute and Storage are completely decoupled.

1. **The Compute Layer:** A fleet of stateless CPU/RAM nodes (the database engine) that execute SQl queries. They have local NVMe SSDs, but *only* for caching.
2. **The Storage Layer:** A massive, distributed file system (like Amazon S3 or a custom distributed block store) that holds the actual canonical data.

**The Benefit: Instant Elasticity.**
When Black Friday hits, you can spin up 100 new Compute nodes in 3 seconds. Because they are stateless, they don't need to copy any data. They simply start pulling the necessary blocks from the central Storage Layer. When traffic drops, you instantly shut them down. You independently scale compute (per second) and storage (per byte).

---

## 3. Why It Became Possible: NVMe-over-Fabrics

If Shared-Storage is so much better for elasticity, why didn't we do it in 2010? 
Because in 2010, fetching data over a standard 1 Gigabit Ethernet network was 1,000x slower than reading it from a local hard drive.

The tipping point was a massive hardware innovation: **NVMe-over-Fabrics (NVMe-oF)** and 100+ Gigabit cloud networking.

By 2020, cloud providers had deployed RDMA (Remote Direct Memory Access) and custom hypervisors (like AWS Nitro). Suddenly, a Compute Node could read a block of data from a remote Storage server over the network *almost exactly as fast* as reading it from a local SSD plugged directly into the motherboard.

The network bottleneck disappeared.

---

## 4. Deconstructing Amazon Aurora

*Source literature: "Amazon Aurora: Design Considerations for High Throughput Cloud-Native Relational Databases" (SIGMOD 2017 & 2018).*

Amazon Aurora is the most famous example of breaking the traditional monolithic database (MySQL/PostgreSQL) into a Shared-Storage architecture.

In standard MySQL (InnoDB), a single `UPDATE` query generates an insane amount of disk I/O. It must write to:
1. The Redo Log (WAL)
2. The Binlog
3. The actual Data Pages (Doublewrite buffer)
4. The Undo Log

If you have a primary node and remote Read Replicas, the primary must send all this actual block data over the network, choking the network bandwidth.

### The Aurora Innovation: "The Log is the Database"
Aurora completely redesigned this. The Aurora Compute Node (where your SQL runs) **never writes data pages**. It *only* writes Redo Log records.

It streams this tiny Redo Log over the network to a custom, massively distributed Storage Fleet. The Storage Nodes themselves are intelligent. They receive the Redo Log, and *they* compute the new data pages asynchronously in the background.

```java
// Conceptual Aurora Storage Node Logic (Simplified)
public class AuroraStorageNode {
    private BlockStore disk;
    
    public void onReceiveRedoLog(RedoLogRecord log) {
        // 1. Immediately append to local NVMe WAL (Fast)
        disk.appendLog(log);
        
        // 2. Acknowledge to Compute Node immediately (Low Latency)
        sendAckToComputeNode();
        
        // 3. In the background, asynchronously apply the math
        // to turn the log into an actual Database Page
        asyncApplyThread.submit(() -> {
            DataPage oldPage = disk.readPage(log.getPageId());
            DataPage newPage = applyRedo(oldPage, log);
            disk.writePage(newPage);
        });
    }
}
```

By pushing the heavy lifting of page materialization down into the storage layer, Aurora reduced network traffic by **35x** compared to standard MySQL, allowing it to handle exponentially higher write throughput on the exact same hardware.

---

## 5. The 2026 Extension: Serverless Postgres (Neon)

While Aurora decoupled compute from storage, systems like **Neon** (open-source Serverless Postgres) took it a step further in the 2020s by separating compute, storage, *and* the WAL service.

Neon allows you to instantly spin up "Database Branches" (like Git branches). Because the storage layer is heavily versioned (using Copy-on-Write techniques in Rust), creating an exact clone of a 10TB production database for a staging environment takes **0.5 seconds** and uses absolutely zero extra disk space until you actually modify the clone.

This is the ultimate evolution of data infrastructure: the database itself has become a lightweight, ephemeral compute function hovering above an infinite, version-controlled storage plane.
