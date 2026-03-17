# 05. Consensus and Failure (The "What If" Scenarios)

In 2026, global enterprises rely on globally distributed databases like Google Spanner, CockroachDB, and TiDB. These systems promise "Five Nines" (99.999%) of availability and strict serializable transactions across continents.

But what actually happens internally when a fiber optic cable across the Atlantic is severed, partitioning the New York data center from the London data center?

The answer lies in the deeply mathematical world of **Consensus Protocols**.

---

## 1. The Two Generals Problem

To understand consensus, we must start with the fundamental impossibility proof of distributed computing: **The Two Generals Problem**.

Imagine two generals, General A and General B, situated on opposite hills, looking down at a city they want to attack.
- They can only win if they both attack at the exact same time.
- They can only communicate by sending messengers through the enemy city valley.

General A sends a messenger: *"Let's attack at dawn."*
But General A cannot attack until he receives confirmation from B. If he attacks alone, he dies.
General B receives the message and sends a messenger back: *"I agree, dawn."*
But General B cannot attack until he receives confirmation that A *received* his confirmation. If B's messenger was captured, A won't attack, and B will die alone.

This leads to an infinite loop of required acknowledgments. **Mathematically, it is proven impossible to achieve absolute certainty over an unreliable network.**

---

## 2. The Paxos and Raft Algorithms

Because absolute certainty is impossible, computer scientists designed algorithms that achieve **Consensus** (agreement) among $N$ nodes as long as a strict majority $\left(\lfloor N/2 \rfloor + 1\right)$ of nodes remain alive and can communicate.

### Paxos
Invented by Leslie Lamport in 1989 (and famously published as a Greek mythological allegory), Paxos was the theoretical foundation of distributed consensus for decades, used in Google's Chubby lock service and Spanner.

However, Paxos is notoriously difficult to understand and even harder to implement correctly. The state space for failure recovery is massive.

### Raft
Created by Diego Ongaro and John Ousterhout in 2014 specifically to be understandable, **Raft** is the reigning standard in 2026 (powering etcd, Kubernetes, CockroachDB, Kafka's KRaft core).

Raft decomposes consensus into three specific sub-problems:
1. **Leader Election:** The cluster elects a single Leader.
2. **Log Replication:** The Leader accepts all client writes, appends them to its local log, and broadcasts them to the Followers.
3. **Safety:** If a Leader crashes, the new Leader must be guaranteed to contain all previously committed entries.

#### The Mathematics of Raft Quorums

Let $Q_{write}$ be the set of nodes that acknowledge a write, and $Q_{read}$ be the set of nodes participating in a leader election.
Raft enforces strict consistency using Quorum Intersection:
$$ |Q_{write}| + |Q_{read}| > N $$

Because the quorums must overlap, it is guaranteed that at least one node in the new Leader Election quorum $Q_{read}$ has seen the latest write from $Q_{write}$.

A piece of data is considered **Committed** (safe to acknowledge to the user) only when the Leader receives a successful append response from a majority of nodes.
For a 5-node cluster, 3 nodes must successfully write the data to their physical SSD before the transaction is finalized. 

---

## 3. "What If" Scenario: CockroachDB Global Partition

Let's walk through a catastrophic failure of a globally distributed SQL database powered by Raft, like CockroachDB.

**The Setup:**
A user's row is stored in a Raft group spanning 3 geographical regions: `us-east` (Leader), `eu-west` (Follower 1), and `ap-northeast` (Follower 2).

**The Disaster:**
The trans-Atlantic fiber cable is cut. The `us-east` data center completely loses connectivity to `eu-west` and `ap-northeast`. 

**The Immediate Effect (Milliseconds 0-500):**
1. The Leader (`us-east`) receives a client `UPDATE` request.
2. It writes to its local log and sends RPC requests over the open internet to the two followers.
3. The requests timeout. The Leader only has 1 vote (itself). It cannot achieve a majority quorum (2 out of 3).
4. *The `us-east` cluster stalls the write and returns a timeout error to the client.* (This is the "C" in CAP Theorem — choosing Consistency over Availability).

**The Raft Leader Election Phase (Milliseconds 500-2000):**
1. Over in `eu-west`, the Follower node's "Heartbeat Timer" expires because it hasn't heard from the `us-east` leader in 500ms.
2. The `eu-west` node transitions to **Candidate** state and increments its internal `term` counter.
3. It votes for itself and sends an `RequestVote` RPC to `ap-northeast`.
4. `ap-northeast` receives the request. It compares the Candidate's log against its own. Seeing that they are equally up-to-date, it grants its vote.
5. The `eu-west` node now has 2 votes (a majority). It instantly transitions to **Leader** state.

**The Recovery (Second 2+):**
The database routing layer detects the new Leader. All incoming SQL traffic for this row is immediately routed to Europe. The database is fully operational again, with zero data loss, completely automatically, in under 2 seconds.

### The Code: Internal Raft State Machine
*Conceptual source inspired by `coreos/etcd/raft`:*

```go
// The heartbeat tick function inside a Raft Follower
func (r *raft) tickElection() {
    r.electionElapsed++
    
    // If we haven't heard from the leader in a random timeout interval...
    if r.promotable() && r.pastElectionTimeout() {
        r.electionElapsed = 0
        r.Step(pb.Message{From: r.id, Type: pb.MsgHup}) // Trigger election
    }
}

// Handling the election trigger
func (r *raft) campaign(t CampaignType) {
    r.becomeCandidate()
    r.term++
    r.vote = r.id // Vote for self
    
    // Broadcast RequestVote to all peers
    for id := range r.prs {
        if id == r.id { continue }
        r.send(pb.Message{Term: r.term, To: id, Type: pb.MsgVote})
    }
}
```

---

## 4. Google Spanner and the TrueTime API

While Raft solves consensus for a *single* group of nodes (a single partition), what happens when a banking transaction spans two completely different partitions? You need Distributed Transactions (Two-Phase Commit, 2PC).

The challenge with 2PC across the globe is **Time**.
If you commit Transaction A in New York at 12:00:00.001, and Transaction B in London at 12:00:00.002, how do you mathematically prove which one happened first? Standard NTP servers have millisecond clock drift. The servers could be out of sync, violating Strict Serializability.

In 2012, Google solved this physical boundary with **TrueTime**.
Google placed atomic clocks and GPS receivers in every single one of their data centers. The TrueTime API does not return a timestamp. It returns an *interval* $[t.earliest, t.latest]$ representing the bounds of clock uncertainty.

$$ \epsilon = t.latest - t.earliest $$
(In Spanner, $\epsilon$ is strictly guaranteed to be less than 7 milliseconds).

To guarantee strict serializable ordering, Google Spanner literally introduces **Commit Wait**. Before replying success to the user, the database forces the thread to physically wait out the uncertainty window $\epsilon$. It simply sleeps for 7 milliseconds. 
This ensures that by the time the user receives the acknowledgement, the transaction's timestamp is mathematically guaranteed to be fully in the past, globally, forever.

This is the ultimate intersection of physics, hardware clocks, mathematical consensus, and database engineering.
