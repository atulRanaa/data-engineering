# 3. Database Systems

The term "database" categorizes wildly different software systems depending on the algorithmic structure used to write data to the disk. 

!!! info "Historical Context: The Rise of NoSQL"
    In the 2000s, Web 2.0 companies (Google, Facebook, Amazon) generated unstructured traffic data at a scale that traditional relational databases (Oracle, MySQL) literally could not physically handle. In response, Amazon published the Dynamo paper (2007) and Google published Bigtable (2006), sparking the "NoSQL" (Not Only SQL) revolution which traded relational joins and strict ACID consistency for horizontal scalability.

---

## 3.1 OLTP vs OLAP vs HTAP

Database architectures are generally divided into three workload categories:

### OLTP (Online Transaction Processing)
- **Examples:** PostgreSQL, MySQL.
- **Goal:** Sub-millisecond reads/writes of single rows. e.g., "Deduct $5 from John's checking account."
- **Internal Storage:** Row-oriented (B-Trees). Data for an entire row is stored contiguously on the hard drive. 

### OLAP (Online Analytical Processing)
- **Examples:** Snowflake, Amazon Redshift, Google BigQuery.
- **Goal:** Aggregating billions of rows. e.g., "What was the average revenue across all distinct zip codes in 2024?"
- **Internal Storage:** Column-oriented (Parquet format style). Instead of storing rows together, all values for the `revenue` column are stored completely contiguously on the SSD. If the query doesn't ask for the `first_name` column, the hard drive head completely skips scanning that section of the disk, saving terabytes of I/O.

### HTAP (Hybrid Transactional/Analytical Processing)
- **Examples:** TiDB, Google Spanner.
- **Goal:** Doing both simultaneously without complex ETL pipelines syncing the OLTP DB to the OLAP DB. HTAP databases dynamically store data in *both* row format (for fast inserts) and column format (for fast background analytics).

### Comparison Table

| Property | OLTP | OLAP | HTAP |
|---|---|---|---|
| **Primary Use** | Real-time transactions | Historical analytics | Both simultaneously |
| **Query Type** | `INSERT`, `UPDATE`, single-row `SELECT` | `SUM`, `AVG`, `GROUP BY` over billions of rows | Mixed workloads |
| **Storage Layout** | Row-oriented (B-Tree) | Column-oriented (Parquet/ORC) | Dual (Row + Column replicas) |
| **Latency** | < 10ms | Seconds to minutes | Varies by operation |
| **Schema** | Highly normalized (3NF) | Denormalized (Star/Snowflake) | Flexible |
| **Concurrency** | Thousands of short txns | Few long-running scans | Medium |
| **Examples** | PostgreSQL, MySQL, Oracle | Snowflake, Redshift, BigQuery | TiDB, SAP HANA, MemSQL |

---

## 3.2 SQL vs. NoSQL

The choice between SQL and NoSQL comes down to strict consistency (CAP Theorem) and the predictability of the schema.

### Relational / SQL
These databases enforce strict ACID guarantees. If a bank transfer crashes midway, a relational database guarantees the system automatically Rolls Back the transaction, preventing money from vanishing.
- **Limitation:** To guarantee strict Consistency, relational databases traditionally operate on a single Primary node. This makes them difficult to scale horizontally (adding more servers).

### Document NoSQL (MongoDB)
Data is stored as nested BSON (JSON) documents. 
**Internal Insight:** MongoDB allows incredible developer velocity because it doesn't enforce a schema. You can add a `friends_list` array variable to User 1 without having to run a blocking `ALTER TABLE` statement that locks out the database for an hour.
- **Limitation:** Documents can grow unruly, and `JOIN`s across millions of JSON documents are highly inefficient.

### Wide-Column NoSQL (Cassandra / Bigtable)
Built explicitly to write petabytes of data continuously without ever dropping a connection. They rely on eventual consistency and peer-to-peer masterless architectures. 

### Graph Databases (Neo4j / Amazon Neptune)

!!! info "Why Graphs?"
    Consider a social network query: *"Find all friends-of-friends of Alice who also like Jazz."* In a relational DB, this requires deeply nested self-JOINs that become exponentially slower with each hop. A Graph Database stores data as **Nodes** (entities) and **Edges** (relationships), traversing connections in $O(1)$ per hop regardless of dataset size.

- **Use Cases:** Social networks, fraud detection (tracing money flow), knowledge graphs, recommendation engines.
- **Query Language:** Cypher (Neo4j) — `MATCH (a:Person)-[:FRIENDS_WITH]->(b)-[:LIKES]->(g:Genre {name: 'Jazz'}) RETURN b`

### Database Type Comparison

| Type | Data Model | Query Language | Scaling | Best For | Trade-off |
|---|---|---|---|---|---|
| **Relational** | Tables + Rows | SQL | Vertical (single node) | Financial transactions, ERP | Rigid schema, hard to shard |
| **Document** | Nested JSON/BSON | MongoDB Query, SQL-like | Horizontal (sharding) | Content management, user profiles | No JOINs, denormalized |
| **Wide-Column** | Rows + Column families | CQL (Cassandra) | Horizontal (masterless ring) | IoT telemetry, time-series logs | Eventual consistency |
| **Graph** | Nodes + Edges | Cypher, Gremlin | Varies | Social networks, fraud detection | Poor for bulk analytics |
| **Key-Value** | Key → Binary blob | GET/SET API | Horizontal (hash-based) | Session caching, leaderboards | No complex queries |

??? example "Python Snippet: MongoDB Document Insertion"
    Using the PyMongo driver to insert unstructured, nested data directly into an OLTP NoSQL store.
    ```python
    import pymongo

    client = pymongo.MongoClient("mongodb://localhost:27017/")
    db = client["application_db"]
    users_col = db["users"]

    # Schema-less: We can embed lists and nested dictionaries effortlessly
    user_document = {
        "user_id": 998,
        "name": "Alice Wonderland",
        "email": "alice@gmail.com",
        "history": [
            {"event": "login", "timestamp": "2024-01-01T10:00:00Z"},
            {"event": "purchase", "item": "laptop", "price": 1200.00}
        ]
    }

    x = users_col.insert_one(user_document)
    print(f"Inserted Document ID: {x.inserted_id}")
    ```

---

## 3.3 NewSQL (The Best of Both Worlds)

!!! info "Historical Context: Google Spanner"
    In 2012, engineers realized that NoSQL's "eventual consistency" was a nightmare for application developers who had to write custom code to handle data conflicts. Google published the **Spanner** paper, proving you could have a globally distributed, horizontally scalable database that still provided strict SQL ACID transactions, enabled by physically synchronizing atomic clocks (TrueTime) across data centers.

Following Spanner, the industry rallied around **NewSQL** (CockroachDB, YugabyteDB, TiDB). These systems distribute data using Raft consensus across thousands of nodes but still expose a standard Postgres/MySQL interface to the developer, effectively sunsetting the necessity of NoSQL for strict financial transactions.

---

!!! abstract "References & Papers"
    - **Spanner: Google's Globally Distributed Database** (Corbett et al., OSDI 2012).
    - **Dynamo: Amazon's Highly Available Key-value Store** (DeCandia et al., 2007).
    - **Designing Data-Intensive Applications** - Chapter 1 (Reliability, Scalability, and Maintainability) and Chapter 5 (Replication) explain why NoSQL architectures were chosen and the subsequent NewSQL shift.
