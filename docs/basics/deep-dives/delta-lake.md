# Deep Dive: Delta Lake vs Apache Iceberg

Historically, Data Lakes were just collections of dumb files scattered across cheap cloud storage like Amazon S3 or Hadoop HDFS. If you had 1,000 Parquet files representing a massive log table, you couldn't safely run a SQL `UPDATE` or `DELETE` statement. Distributed readers would collide with distributed writers, corrupting the table.

**Lakehouse Data Formats** (Delta Lake, Apache Iceberg, Apache Hudi) solved this. They bring ACID database guarantees (Atomicity, Consistency, Isolation, Durability) directly to your cheap cloud object storage, eliminating the need for expensive Data Warehouses for many workloads.

---

## 1. History & Origins

The "Data Lakehouse" paradigm emerged around 2017 to bridge the gap between Data Lakes (cheap storage, unstructured data) and Data Warehouses (expensive compute, strict SQL rules). 

1. **Delta Lake (2017):** Developed internally by Databricks, it was originally a proprietary storage layer for their customers struggling with massive S3 Parquet corruption. Databricks open-sourced it in 2019 to accelerate industry adoption, paving the way for the "Lakehouse" architecture.
2. **Apache Iceberg (2018):** Developed by Netflix. Netflix had petabytes of data on S3 and their Hive Metastore queries were taking upwards of 30 minutes just to *list* the files in a folder before a Spark job could even begin reading. Iceberg was built specifically to solve the "S3 `LIST` metadata bottleneck."

---

## 2. Core Architecture and Internal Insights

Both formats operate on the same mathematical principle: separating the *State* of the table from the *Physical Files*.

### The Transaction Log
A Lakehouse table is a directory containing raw Parquet files (the data) alongside a dedicated metadata directory (the Transaction Log).

In Delta Lake, the `_delta_log/` directory contains JSON files detailing exactly which physical Parquet files represent the table at "Version 01", "Version 02", etc.
If a Spark cluster writes a new Parquet file to S3, the table doesn't "know" about it until the cluster writes a new JSON commit file physically verifying the transaction. 

**Internal Insight: Optimistic Concurrency Control (OCC)**
What happens if two Spark clusters try to update a table simultaneously? They use OCC. 
Cluster A and Cluster B both read Version 01. Both write their modified Parquet files to S3. Both attempt to legally commit "Version 02.json" simultaneously. Delta/Iceberg relies on the Cloud Provider's atomic file creation guarantees (S3 conditional writes). Whoever creates `02.json` first wins. The loser catches the exception, re-reads the log, sees what Cluster A did, mathematically verifies if the datasets conflict, and simply tries again for `03.json`.

### Time Travel and Schema Evolution
Because the Transaction Log keeps an immutable record of every `ADD FILE` and `REMOVE FILE` command, old data is never immediately deleted. You can simply command Spark to read the state of the JSON log exactly as it looked 5 days ago, completely restoring the database to that second in time (Time Travel).

---

## 3. Hands-on Code Example

Below is a complete PySpark script demonstrating how to write, conditionally update, and time-travel on a Delta Lake table.

```python
from pyspark.sql import SparkSession
from delta.tables import DeltaTable

# 1. Initialize Spark with Delta dependencies
spark = SparkSession.builder \
    .appName("DeltaLakehouseExample") \
    .config("spark.sql.extensions", "io.delta.sql.DeltaSparkSessionExtension") \
    .config("spark.sql.catalog.spark_catalog", "org.apache.spark.sql.delta.catalog.DeltaCatalog") \
    .getOrCreate()

# 2. Write an initial DataFrame as a Delta Table (Version 0)
data = [(1, "Alice", 500), (2, "Bob", 300)]
columns = ["user_id", "name", "lifetime_value"]
df = spark.createDataFrame(data, columns)

df.write.format("delta").mode("overwrite").save("s3://lakehouse/users_table")

# 3. Read the Delta Table
deltaTable = DeltaTable.forPath(spark, "s3://lakehouse/users_table")

# 4. Perform an ACID Upsert (MERGE INTO)
# Imagine we received a stream of updates. Bob's value increased, and Charlie is new.
updates_data = [(2, "Bob", 600), (3, "Charlie", 100)]
updates_df = spark.createDataFrame(updates_data, columns)

deltaTable.alias("target") \
  .merge(
    updates_df.alias("source"),
    "target.user_id = source.user_id"
  ) \
  .whenMatchedUpdateAll() \
  .whenNotMatchedInsertAll() \
  .execute()

# 5. Time Travel!
# Oh no, the UPSERT was a mistake! Let's read the data exactly as it was 
# before the merge happened (Version 0).
df_original = spark.read.format("delta").option("versionAsOf", 0).load("s3://lakehouse/users_table")
df_original.show()
```

---

!!! abstract "References & Foundational Reading"
    - **Original Paper:** *Delta Lake: High-Performance ACID Table Storage over Cloud Object Stores* (Armbrust et al., VLDB 2020).
    - **Original Paper/Docs:** *Apache Iceberg: A Table Format for Huge Analytic Datasets* (Netflix).
    - **Designing Data-Intensive Applications:** Chapter 7 (Transactions) covers ACID properties and concurrency control algorithms like OCC necessary for Lakehouses.
