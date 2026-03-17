# 7. Distributed Systems & Scaling

When data exceeds the capacity of a single physical server, engineers must scale out horizontally. This fundamentally changes software engineering from managing logic to managing **Physics and Failure**.

!!! info "Historical Context: The CAP Theorem"
    In the late 1990s, Eric Brewer formulated the CAP Theorem, which mathematically proved that a distributed database can only provide two of the three following guarantees: **Consistency** (Every read receives the most recent write), **Availability** (Every request receives a non-error response), or **Partition Tolerance** (The system continues to operate despite arbitrary network failures dropping messages). Because networks *will* fail, you must choose between CP (Consistency) and AP (Availability). This dictated the design of every single NoSQL database for the next 20 years.

---

## 7.1 Sharding vs. Replication

To scale a database, you manipulate the dataset in two ways.

### Replication (High Availability)
Copying the exact same data to multiple servers.
- **Leader-Follower:** All writes go to the Leader. The Leader streams the writes to Followers. Reads can scale infinitely by reading from Followers, but if the Leader dies, write operations are completely blocked until a new Leader is elected via Consensus algorithms (e.g., Raft).
- **Multi-Leader:** Complex, used for multi-datacenter setups. Two users on different continents can write identical rows simultaneously, creating horrific conflict resolution problems (often solved by CRDTs - Conflict-free Replicated Data Types, or "Last Writer Wins" timestamps).

### Sharding / Partitioning (High Throughput)
Splitting a massive dataset into isolated chunks (Shards) spread across different servers.
**Internal Insight:** The most dangerous problem in Data Engineering is the **Hotspot** (or data skew). If you shard Twitter by User ID, and Elon Musk tweets, 100% of the traffic will hit the single server containing his shard, immediately crashing it while the other 99 servers sit idle.

??? example "Consistent Hashing Algorithm (Python Concept)"
    To avoid hotspots when a server joins or leaves a cluster, NoSQL databases like DynamoDB and Cassandra use **Consistent Hashing**. Instead of `server = hash(key) % N`, servers and data are placed on a massive numerical circle.
    ```python
    import hashlib

    # The 32-bit Hash Ring (0 to 4,294,967,295)
    ring = {}
    servers = ["192.168.1.1", "192.168.1.2", "192.168.1.3"]

    def hash_key(key):
        return int(hashlib.md5(key.encode('utf-8')).hexdigest(), 16) % (2**32)

    # 1. Place the 3 servers on the physical ring
    for server in servers:
        ring[hash_key(server)] = server
    
    # 2. To insert a user row, hash their ID
    user_hash = hash_key("user_123_data")

    # 3. Find the first server clockwise on the ring!
    # If a server crashes, the data automatically falls onto the next server
    # without having to re-hash millions of other rows.
    assigned_server = next(
        (ring[k] for k in sorted(ring.keys()) if k >= user_hash), None
    )
    if not assigned_server: 
        assigned_server = ring[sorted(ring.keys())[0]]  # Wrap around
    ```

---

## 7.2 Consensus: Solving the Two Generals Problem

If 5 servers are sharing data, how do they agree on the exact order of events if the network switch connecting them is losing 10% of their packets? 

This is the mathematically unsolvable **Two Generals Problem**. In practice, we solve it using Consensus Quorums.

**Internal Insight:** You must always have an **odd number** of servers (3, 5, or 7) in a consensus cluster (like ZooKeeper or etcd). If you have 4, and the network splits in half (2 vs 2), neither side can establish a mathematical majority ($N/2 + 1$). The entire system will freeze in a "Split Brain" scenario.

### Consensus Algorithm Comparison

| Property | Paxos (1989) | Raft (2014) | Zab (ZooKeeper) |
|---|---|---|---|
| **Inventor** | Leslie Lamport | Diego Ongaro (Stanford) | Yahoo Research |
| **Design Goal** | Mathematical correctness | Understandability | ZooKeeper-specific |
| **Leader Election** | Complex (multi-round) | Simple (randomized timeout) | Integrated with atomic broadcast |
| **Log Replication** | Separate from election | Unified with election | Tightly coupled |
| **Membership Change** | Joint consensus (complex) | Single-server changes | Static configuration |
| **Used By** | Google Chubby, Spanner | etcd, Consul, CockroachDB, TiKV | Apache ZooKeeper, Kafka (legacy) |
| **Understandability** | Notoriously hard | Designed to be taught | Moderate |

---

## 7.3 Resource Management (YARN vs Kubernetes)

When you submit a Spark job asking for 500 CPUs, something must allocate those CPUs from a pool of physical servers. That is the **Resource Manager**.

### YARN (Yet Another Resource Negotiator)
- **Era:** Hadoop (2012-2020). YARN is baked directly into the Hadoop distribution.
- **Architecture:** A central `ResourceManager` accepts requests from `ApplicationMasters` (one per Spark/MapReduce job). It allocates `Containers` (CPU + RAM slices) on `NodeManagers` running on each server.
- **Limitation:** Hadoop-centric. If you need to run a Go microservice alongside a Spark job, YARN cannot schedule it.

### Kubernetes
- **Era:** Cloud-native (2015+). Language/framework agnostic.
- **Architecture:** The `kube-scheduler` places `Pods` (containers) onto `Nodes` based on resource requests (`cpu: "4"`, `memory: "16Gi"`). The **Spark Operator** on K8s lets you submit Spark jobs as YAML manifests, and Kubernetes handles everything YARN used to do—plus auto-scaling, rolling upgrades, and health checks.
- **Advantage:** You can run Spark, Kafka, Flink, Airflow, and your custom Python services all on the same Kubernetes cluster, sharing resources dynamically.

| Feature | YARN | Kubernetes |
|---|---|---|
| **Scope** | Hadoop ecosystem only | Any containerized workload |
| **Scheduling Unit** | Container (JVM) | Pod (Docker container) |
| **Auto-scaling** | Manual (add nodes) | Horizontal Pod Autoscaler |
| **Multi-tenancy** | Queues (Fair/Capacity) | Namespaces + Resource Quotas |
| **Ecosystem** | Spark, MapReduce, Tez, Hive | Spark, Flink, Kafka, Airflow, ML, APIs |
| **Cloud-Native** | No (designed for on-prem) | Yes (EKS, GKE, AKS) |

---

!!! abstract "References & Papers"
    - **Brewer's Conjecture and the Feasibility of Consistent, Available, Partition-Tolerant Web Services** (Gilbert and Lynch, 2002). The formal mathematical proof of the CAP Theorem.
    - **In Search of an Understandable Consensus Algorithm** (Ongaro and Ousterhout, USENIX 2014). The Raft paper.
    - **Designing Data-Intensive Applications** - Chapter 8 (The Trouble with Distributed Systems) and Chapter 9 (Consistency and Consensus) explain the horrors of network partitions and clock drift.
