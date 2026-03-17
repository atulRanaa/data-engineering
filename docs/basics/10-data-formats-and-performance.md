# 10. Data Formats and Performance

You cannot optimize a query if you do not understand how the bytes are literally arranged on the SSD. Analytical formats (Columnar) and Transactional formats (Row-based) are diametrically opposed in design.

!!! info "Historical Context: The Invention of Columnar Storage"
    In the 2000s, traditional databases stored data row-by-row (CSV or B-Trees). If you had a 1-Billion row table and ran `SELECT SUM(salary) FROM employees`, the hard drive physically had to scan every single byte of every single row (names, addresses, phone numbers) just to find the integer `salary` tucked into each row. It was disastrously slow. In 2005, the **C-Store** academic paper formally proposed column-oriented architectures. In 2013, Twitter and Cloudera released **Apache Parquet**, standardizing the columnar format for the Hadoop ecosystem.

---

## 10.1 Row vs. Columnar Formats

### Apache Avro (Row-based)
Used for streaming data (Kafka).
A JSON file containing `{ "id": 1, "name": "Bob", "age": 30 }` wastes disk space because it repeats the strings `"id"`, `"name"`, and `"age"` for every row.
Avro stores the schema once at the top of the file as JSON. Below it, it stores the data as pure raw binary (`00000001 010000100110111101100010 00011110`). The serialization is incredibly fast and highly compressible.

### Apache Parquet (Column-based)
Used for data lakes (S3, Iceberg).
Instead of storing `Row 1, Row 2, Row 3`, Parquet stores `All IDs`, then `All Names`, then `All Ages`. 

**Internal Insight:** Parquet does not just store vectors; it stores them in **Row Groups** (e.g., 256MB chunks). At the end of every Row Group is a metadata strictly defining the `MIN` and `MAX` value for every column in that chunk.
When Spark runs `SELECT * WHERE age = 60`, it downloads the tiny metadata footers over the network first. If Row Group 1 says its Age range is `[18, 55]`, Spark completely skips downloading that 256MB block (Predicate Pushdown).

### Apache ORC (Optimized Row Columnar)

!!! info "Historical Context: ORC vs Parquet — The Hadoop Wars"
    ORC was created by Hortonworks specifically for the Hive ecosystem, while Parquet was created by Twitter + Cloudera for the broader Hadoop landscape. For years, the Hadoop community was split. ORC won in the Hive-centric world because it natively supported ACID transactions (INSERT, UPDATE, DELETE) on Hadoop tables—something Parquet could not do until Delta Lake arrived in 2020.

ORC stores data in **Stripes** (similar to Parquet's Row Groups, typically 250MB). Each Stripe contains:

- **Index Data:** Bloom filters and min/max statistics for each column — enabling aggressive skip-scanning.
- **Row Data:** Columnar-encoded values.
- **Stripe Footer:** Column encoding metadata.

**Key Advantage over Parquet:** ORC has built-in lightweight indexes (Bloom Filters) per Stripe. This makes point lookups (e.g., `WHERE user_id = 12345`) significantly faster than scanning Parquet Row Group footers.

### Format Comparison Table

| Feature | Parquet | ORC | Avro | JSON / CSV |
|---|---|---|---|---|
| **Layout** | Columnar | Columnar | Row-based (binary) | Row-based (text) |
| **Compression** | Excellent (column homogeneity) | Excellent (+ built-in indexes) | Good | Poor |
| **Schema** | Embedded in footer | Embedded in file tail | Embedded in header (JSON) | None enforced |
| **Schema Evolution** | Limited (append columns) | Limited | Full (forward + backward) | None |
| **ACID Support** | Via Delta Lake / Iceberg | Native (Hive ACID) | No | No |
| **Column Pruning** | Yes (skip unneeded columns) | Yes | No (must read full row) | No |
| **Predicate Pushdown** | Yes (Row Group min/max) | Yes (Stripe Bloom filters) | No | No |
| **Splittable** | Yes (Row Groups) | Yes (Stripes) | Yes (blocks) | CSV: Yes, JSON: No |
| **Best For** | Data Lakes, OLAP, ML | Hive/Hadoop, Transactional Lakes | Kafka streaming, RPC | Ad-hoc exchange, APIs |
| **Ecosystem** | Universal (Spark, Flink, Trino) | Hive, Spark, Presto | Kafka, Avro RPC | Everything |

??? example "PyArrow Code: Inspecting Parquet Metadata Footers"
    A data engineer must verify that Predicate Pushdown is mathematically possible by inspecting the Parquet file's internal physics.
    ```python
    import pyarrow.parquet as pq

    # 1. Read the Parquet file metadata directly from the OS
    parquet_file = pq.ParquetFile('data/sales_data.parquet')
    
    print(f"Total Row Groups: {parquet_file.num_row_groups}")

    # 2. Inspect the metadata footer of the first Row Group
    group_zero = parquet_file.metadata.row_group(0)
    
    # Let's say column 2 is 'salary'
    salary_metadata = group_zero.column(2)
    
    # Spark looks at these two exact numbers to decide if it should download the chunk!
    print(f"Salary Min: {salary_metadata.statistics.min}")
    print(f"Salary Max: {salary_metadata.statistics.max}")
    ```

---

## 10.2 Compression Algorithms

When you store identical data types next to each other sequentially (e.g., 100,000 integers in a column), the compression ratio skyrockets.

### Compression Codec Comparison

| Codec | Compression Ratio | Decompression Speed | CPU Cost | Default In | Best For |
|---|---|---|---|---|---|
| **Snappy** (Google) | Moderate (~2x) | Very Fast (~500 MB/s) | Low | Parquet, BigQuery | Hot data, real-time queries |
| **GZIP** | High (~5x) | Slow (~100 MB/s) | High | HTTP, legacy archives | Cold storage, small files |
| **ZStandard** (Facebook) | High (~5x) | Fast (~400 MB/s) | Medium | Kafka, modern systems | Best general-purpose choice |
| **LZ4** | Low (~1.5x) | Extremely Fast (~800 MB/s) | Very Low | RocksDB, ClickHouse | Ultra-low-latency KV stores |
| **Brotli** (Google) | Very High (~6x) | Moderate | High | Web content (HTTP) | Static asset compression |

**Internal Insight: Run-Length Encoding (RLE) and Dictionary Encoding**
Parquet uses native columnar tricks before even applying Snappy. 
If a column contains `[US, US, US, US, CA, CA]`, Parquet uses RLE to store it physically as `4xUS, 2xCA`. It converts billions of characters into simple mathematical multipliers.

---

## 10.3 Indices (B-Tree vs LSM-Tree vs Inverted Index)

If you need to find one row out of a billion in an OLTP database (Postgres or Kafka/Cassandra), the engine must use an Index.

- **B-Trees (PostgreSQL):** A self-balancing search tree. It allows $O(\log n)$ read access. The OS must overwrite physical pages on the disk randomly when new data arrives, which is slow for HDDs but fine for NVMe SSDs.
- **LSM-Trees (Cassandra, RocksDB):** Optimized purely for sequential writes. It buffers inserts in a memory MemTable, then flushes them to disk as immutable, sorted files (SSTables). Background compactions merge these files continuously. Read slower, but write infinitely faster.
- **Inverted Indexes (Elasticsearch, Lucene):** Maps every unique *term* to the list of *document IDs* that contain it. When you search `"kubernetes error"`, Elasticsearch intersects the posting lists for `kubernetes` and `error` in microseconds, even across billions of log lines.

| Index Type | Read Speed | Write Speed | Space Overhead | Best For |
|---|---|---|---|---|
| **B-Tree** | $O(\log n)$ — Fast | Moderate (random I/O) | Low | OLTP point lookups (Postgres) |
| **LSM-Tree** | Moderate (compaction lag) | $O(1)$ amortized — Very Fast | High (write amplification) | High-write workloads (Cassandra) |
| **Inverted** | Very Fast (term lookup) | Slow (index rebuild) | Very High | Full-text search (Elasticsearch) |

---

!!! abstract "References & Papers"
    - **C-Store: A Column-oriented DBMS** (Stonebraker et al., VLDB 2005). The architectural birth of modern columnar formats.
    - **Dremel: Interactive Analysis of Web-Scale Datasets** (Melnik et al., VLDB 2010). Google's paper explaining exactly how protobufs and columnar nesting evolved into Parquet.
    - **RCFile: A Fast and Space-efficient Data Placement Structure in MapReduce-based Warehouse Systems** (He et al., ICDE 2011). The precursor to ORC.
    - **Designing Data-Intensive Applications** - Chapter 3 (Storage and Retrieval) explains B-Trees, LSM-Trees, and Column-Oriented Storage.
