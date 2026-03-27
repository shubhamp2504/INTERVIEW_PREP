# 🌐 Distributed Systems Fundamentals (Q1–Q6)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. CAP Theorem

### 📝 One-Liner
In a distributed system you can guarantee only **two of three**: Consistency, Availability, Partition Tolerance — and since network partitions are inevitable, the real choice is **CP vs AP**.

### 🔑 Quick Answer
**C (Consistency)**: every read returns the most recent write (all nodes see same data simultaneously). **A (Availability)**: every request gets a non-error response (system always responds). **P (Partition Tolerance)**: system works even when network between nodes is broken. **Network partitions WILL happen** in distributed systems (cables fail, switches die), so you MUST choose P. The real tradeoff: **CP** (consistent but may refuse requests during partition — bank accounts, inventory) vs **AP** (available but may return stale data during partition — social media feeds, DNS). *(Network partition hoga hi — isliye choice hai: sahi data do ya hamesha jawab do)*

### 📖 How It Works
```
CAP Theorem:

       C (Consistency)
      / \
     /   \
   CP     CA ← impossible in distributed systems
   /       \     (no partition tolerance = single node)
  P ─────── A
  (Partition   (Availability)
   Tolerance)

Network Partition scenario:
  Node A ←──✗ network broken ✗──→ Node B
  Client writes "balance=500" to Node A

  CP choice (e.g., banking):
    Node B refuses reads until it syncs with A
    → Consistent ✅ but Unavailable for Node B clients ❌

  AP choice (e.g., social feed):
    Node B returns stale data (old balance)
    → Available ✅ but Inconsistent ❌ (stale read)

Real-world choices:
  CP systems: ZooKeeper, HBase, MongoDB (default), etcd, Redis Cluster
    → Refuses requests if can't guarantee consistency
  AP systems: Cassandra, DynamoDB, CouchDB, DNS
    → Always responds, eventual consistency
  CA systems: traditional single-node RDBMS (PostgreSQL, MySQL single)
    → Not distributed, so partition tolerance isn't needed
```

### 🗣️ How to Say in Interview
"The CAP theorem states that a distributed system can provide at most two of three guarantees: Consistency, Availability, and Partition Tolerance. Since network partitions are unavoidable in distributed systems, the practical tradeoff is between CP and AP. CP systems like ZooKeeper or HBase prioritize consistency — they may become unavailable during a partition to ensure all nodes agree on the same data. AP systems like Cassandra or DynamoDB prioritize availability — they always respond but may return stale data during partitions, achieving eventual consistency. In my projects, I choose CP for financial data where correctness is critical, and AP for read-heavy non-critical data like user activity feeds where eventual consistency is acceptable."

### ⚠️ Pitfalls / Gotchas
- **CAP is about partitions** — when there's NO partition, you get both C and A. CAP only kicks in during failure *(Partition hone pe choice karna padta hai — normal state mein dono milte hain)*
- **CP doesn't mean "no availability"** — it means temporarily unavailable DURING partition
- Real systems are **not purely CP or AP** — they tune per-query (e.g., Cassandra configurable consistency level)
- **PACELC** is a better model: if Partition → C or A; Else → Latency or Consistency

### ⚡ Remember
- **C**onsistency + **A**vailability + **P**artition Tolerance — pick 2 (really CP vs AP)
- Partitions are inevitable → **P is mandatory**
- **CP** = banking, inventory (correctness > availability)
- **AP** = social feeds, DNS (availability > freshness)
- PACELC extends CAP with latency tradeoff

### 🔗 Follow-ups
- [Q2 → Consistency Models (eventual, strong, causal)](#q2)
- [Q5 → Data Sharding (partition tolerance in practice)](#q5)

---

<a id="q2"></a>
## Q2. Consistency Models

### 📝 One-Liner
Strong consistency (read always returns latest write — slow), eventual consistency (reads may be stale but converge — fast), causal consistency (respects cause-effect ordering — balanced).

### 🔑 Quick Answer
**Strong Consistency**: after a write completes, ALL subsequent reads (from any node) return that value. Like a single-node database. Cost: high latency (must wait for all replicas to agree). Example: bank balance, inventory count. **Eventual Consistency**: after a write, reads MAY return stale data, but eventually ALL replicas converge to the same value. Cost: stale reads possible. Example: social media likes, DNS propagation. **Causal Consistency**: if event A causes event B, everyone sees A before B. Unrelated events may appear in different order on different nodes. Balances correctness and performance. **Read-your-writes**: after YOU write, YOU always see your own write (others may see stale). Common in web apps. *(Strong = sab ko turant latest dikhao; Eventual = thodi der mein sab ek jaisa ho jaayega; Causal = cause-effect ka order maintain karo)*

### 📖 How It Works
```
Strong Consistency:
  Client writes X=5 → waits for ALL 3 replicas to confirm → returns success
  Any client reads X → guaranteed to get 5

  Write: [Node A: X=5] ──sync──→ [Node B: X=5] ──sync──→ [Node C: X=5]
         → all confirmed → respond "success" to client
  Read from ANY node: X=5 ✅

Eventual Consistency:
  Client writes X=5 → Node A confirms immediately → async replication
  Client reads from Node B (hasn't received update yet) → may get X=3 (old)
  ... seconds later, all nodes have X=5

  Write: [Node A: X=5] ──async──→ [Node B: X=3→5] ──async──→ [Node C: X=3→5]
         → respond "success" immediately (only A confirmed)
  Read from Node B (before sync): X=3 ❌ (stale but eventually correct)

Causal Consistency:
  User posts "I got promoted!" (event A)
  User's friend comments "Congratulations!" (event B, caused by A)

  All nodes must show A before B (cause before effect)
  But unrelated posts can appear in any order on different nodes

Read-Your-Writes:
  You update your profile name to "Shubham"
  You immediately see "Shubham" (your own write visible to you)
  Other users might still see old name for a few seconds (eventual)
```

### 🗣️ How to Say in Interview
"I choose the consistency model based on the business requirement. For financial transactions and inventory where correctness is critical, I use strong consistency — all replicas must agree before the write is acknowledged. For social media feeds or analytics dashboards, eventual consistency is sufficient — reads may be slightly stale but the system is much faster and more available. Causal consistency is useful for messaging systems where the order of cause and effect must be preserved but unrelated messages can be reordered. In practice with Cassandra, I configure consistency levels per query — writes with QUORUM for important data and reads with ONE for high-throughput analytics."

### 🆚 vs. Comparison
| Model | Guarantee | Latency | Example |
|-------|-----------|---------|---------|
| Strong | Latest write always | High (sync all replicas) | Bank balance, ZooKeeper |
| Linearizable | Strong + real-time order | Highest | etcd, Spanner |
| Causal | Cause-effect order | Medium | Messaging systems |
| Read-your-writes | You see your own writes | Low-Medium | Web app sessions |
| Eventual | Converges eventually | Lowest ⭐ | DNS, Cassandra (CL=ONE) |

### ⚡ Remember
- **Strong** = all reads return latest write (slow, correct) *(Sab ko ek hi data dikhega — slow but safe)*
- **Eventual** = reads may be stale, converge over time (fast, AP)
- **Causal** = preserves cause→effect ordering
- Configurable per-query in most distributed DBs (Cassandra, DynamoDB)
- Most web apps use **read-your-writes** + eventual for others

### 🔗 Follow-ups
- [Q1 → CAP Theorem (C vs A tradeoff)](#q1)
- [Q4 → Consensus Algorithms (how nodes agree)](#q4)

---

<a id="q3"></a>
## Q3. Distributed System Architectures

### 📝 One-Liner
Client-Server (centralized), Peer-to-Peer (decentralized), Microservices (modular services), Event-Driven (async messaging), Lambda (batch + stream), Master-Worker (central coordination + distributed execution).

### 🔑 Quick Answer
**Client-Server**: clients send requests to centralized server(s). Simple, most web apps. **Peer-to-Peer (P2P)**: no central server, all nodes are equal (BitTorrent, blockchain). **Microservices**: application split into small, independently deployable services communicating via REST/messaging. **Event-Driven**: services communicate through events via message broker (Kafka/RabbitMQ). Decoupled, scalable. **Master-Worker**: master distributes tasks to worker nodes (Spark driver + executors, MapReduce). **Lambda Architecture**: batch layer (accurate, slow) + speed layer (fast, approximate) + serving layer. **CQRS**: separate read and write models for different scaling needs. *(Har architecture ka use case alag hai — microservices sabse common enterprise mein)*

### 📖 How It Works
```
Architecture Patterns:

1. CLIENT-SERVER (most web apps):
   Clients  ──REST──→  Server(s)  ──→  Database
   Simple, centralized, easy to reason about

2. MICROSERVICES (enterprise):
   ┌────────┐  ┌────────┐  ┌────────┐
   │Order   │  │Payment │  │Inventory│
   │Service │  │Service │  │Service  │
   └───┬────┘  └───┬────┘  └───┬────┘
       └─────Kafka/REST────────┘
   Independent deploy, own DB, team ownership

3. EVENT-DRIVEN (async, decoupled):
   Producers → [Event Bus/Kafka] → Consumers
   Fire-and-forget, eventual consistency

4. MASTER-WORKER (data processing):
   ┌────────┐
   │ Master │ → distributes tasks
   └───┬────┘
   ┌───┴───┐ ┌───────┐ ┌───────┐
   │Worker1│ │Worker2│ │Worker3│ → execute in parallel
   └───────┘ └───────┘ └───────┘
   Example: Spark Driver + Executors

5. LAMBDA ARCHITECTURE (big data):
   Raw Data ──→ Batch Layer (MapReduce, accurate)
       │                      ↘
       └──→ Speed Layer (Storm, real-time) → Serving Layer → Queries
   Combines accuracy of batch with speed of streaming
```

### 🗣️ How to Say in Interview
"I choose architecture based on the problem. For most enterprise applications, microservices architecture — each service owns its domain, data, and lifecycle — combined with event-driven communication through Kafka for cross-service events. For data processing, master-worker pattern like Apache Spark where the driver distributes tasks to executors. For real-time analytics on massive datasets, lambda architecture combining batch processing for accuracy with stream processing for low latency. The key is to not over-architect — I start with a modular monolith and extract microservices when team boundaries and scaling requirements justify the operational complexity."

### 🆚 vs. Comparison
| Architecture | Coupling | Scalability | Complexity | Best For |
|-------------|---------|------------|-----------|----------|
| Monolith | Tight | Vertical | Low ⭐ | Small teams, MVP |
| Microservices | Loose ⭐ | Horizontal ⭐ | High | Large teams, enterprise |
| Event-Driven | Loosest ⭐ | Horizontal ⭐ | High | Real-time, async |
| Master-Worker | Medium | Horizontal | Medium | Data processing |
| P2P | None | Self-scaling | Very High | File sharing, blockchain |

### ⚡ Remember
- **Microservices** = most common enterprise pattern (independent deploy, own DB)
- **Event-Driven** = decoupled async (Kafka) *(Event publish karo, bhool jao)*
- **Master-Worker** = data processing (Spark, MapReduce)
- **Lambda** = batch + stream for big data
- Start **monolith** → extract services as needed — don't over-architect

### 🔗 Follow-ups
- [Q1 → CAP Theorem (tradeoffs in distributed systems)](#q1)
- [Q5 → Data Sharding](#q5)
- Q5 (architecture/03) → Microservices with Spring Cloud

---

<a id="q4"></a>
## Q4. Consensus Algorithms (Paxos, Raft)

### 📝 One-Liner
Consensus algorithms let distributed nodes agree on a single value despite failures — Paxos (theoretical, complex) and Raft (practical, understandable) both use leader election + majority agreement (quorum).

### 🔑 Quick Answer
**Problem**: in a distributed system with N nodes, how do all nodes agree on a value when some nodes may crash or messages may be delayed? **Consensus** = getting a majority (quorum) of nodes to agree. **Paxos**: first consensus algorithm (Lamport, 1989). Correct but notoriously complex to implement. Used in Google's Chubby, Spanner. **Raft**: designed as "understandable Paxos" (2014). Three roles: **Leader** (handles all writes), **Followers** (replicate leader's log), **Candidates** (during election). Uses **log replication** + **leader election** with term numbers. Used in etcd, CockroachDB, Consul. **Quorum**: majority must agree = `(N/2) + 1`. With 5 nodes, quorum = 3, tolerates 2 failures. *(Consensus = bahut saare nodes mein se majority ko ek value pe agree karaana — Raft sabse practical hai)*

### 📖 How It Works
```
Raft Algorithm — Three Phases:

1. LEADER ELECTION:
   All nodes start as Followers. If no heartbeat from Leader:
   Follower → becomes Candidate → requests votes
   If gets majority votes → becomes Leader
   
   Term 1: [Leader: A] [Follower: B] [Follower: C] [Follower: D] [Follower: E]
   
   A crashes! No heartbeat...
   
   Term 2: B becomes Candidate → gets votes from C, D, E
           [Dead: A] [Leader: B] [Follower: C] [Follower: D] [Follower: E]

2. LOG REPLICATION:
   Client → writes to Leader → Leader appends to log
   → Leader replicates to Followers → waits for majority (quorum) ACK
   → Leader commits → notifies Followers to commit
   
   Leader B: [log: X=5] → replicate to C, D, E
   C: ACK ✅  D: ACK ✅  E: (slow) ...
   Quorum (3/5) reached → COMMIT!

3. SAFETY:
   Only nodes with up-to-date logs can become Leader
   → Prevents committed entries from being lost

Quorum Table:
  Nodes: 3 → Quorum: 2 → Tolerates: 1 failure
  Nodes: 5 → Quorum: 3 → Tolerates: 2 failures ⭐
  Nodes: 7 → Quorum: 4 → Tolerates: 3 failures
```

### 🗣️ How to Say in Interview
"Consensus algorithms solve the fundamental problem of getting distributed nodes to agree on a value despite node failures and network delays. Raft is the most widely used practical consensus algorithm — it works by electing a leader who handles all client writes. The leader replicates its log to followers and waits for a majority quorum to acknowledge before committing. If the leader fails, followers detect the missing heartbeat, start an election, and a new leader is elected. The quorum requirement — a majority of nodes — ensures that committed values are never lost since any majority overlaps with the previous. In practice, I use systems built on Raft like etcd for Kubernetes coordination and Consul for service discovery, rather than implementing consensus directly."

### ⚠️ Pitfalls / Gotchas
- **Never implement your own consensus** — use battle-tested systems (etcd, ZooKeeper, Consul) *(Khud Paxos/Raft implement mat karo — existing systems use karo)*
- **Even number of nodes** = bad (split vote possible). Always use odd: 3, 5, 7
- **More nodes ≠ better** — more nodes = higher write latency (more ACKs needed). 5 is the sweet spot
- **Raft leader bottleneck** — all writes go through leader. For write-heavy, consider multi-Raft (sharded)

### 🆚 vs. Comparison
| Aspect | Paxos | Raft | ZAB (ZooKeeper) |
|--------|-------|------|-----------------|
| Complexity | Very high | Moderate ⭐ | High |
| Understandability | Difficult | Easy ⭐ | Moderate |
| Leader-based | Multi-decree | Single leader ⭐ | Single leader |
| Used in | Spanner, Chubby | etcd, Consul, CockroachDB | ZooKeeper |
| Correctness | Proven | Proven | Proven |

### ⚡ Remember
- **Consensus** = distributed nodes agreeing on a value (despite failures)
- **Raft** = Leader election + log replication + quorum commit *(Leader likhta hai, majority confirm karti hai)*
- **Quorum** = (N/2)+1 — use **odd nodes** (3, 5, 7)
- 5 nodes = tolerates 2 failures (production sweet spot)
- Don't implement yourself — use etcd, ZooKeeper, Consul

### 🔗 Follow-ups
- [Q1 → CAP Theorem (consensus = CP choice)](#q1)
- Q14 (system-design/02) → Zookeeper uses ZAB consensus

---

<a id="q5"></a>
## Q5. Data Sharding and Partitioning

### 📝 One-Liner
Sharding splits data across multiple database nodes by a shard key — horizontal partitioning (rows) distributes load, but cross-shard queries and rebalancing are hard.

### 🔑 Quick Answer
**Sharding** = splitting a database table across multiple independent database instances (shards). Each shard holds a subset of data. **Why**: single DB can't handle 1B rows or 100K QPS — sharding distributes load. **Strategies**: **(1) Hash-based** — `hash(userId) % numShards`. Even distribution, but hard to range query. **(2) Range-based** — shard by date range or ID range. Good for range queries, but hot spots possible. **(3) Directory-based** — lookup table maps key → shard. Flexible, but lookup table = bottleneck. **Vertical partitioning** = splitting columns into separate tables/services (user profile vs user activity). **Horizontal partitioning** = splitting rows across shards (users 1-1M on shard 1, 1M-2M on shard 2). *(Sharding = data ko multiple databases mein baanto — ek DB se limit cross ho gayi toh)*

### 📖 How It Works
```
Horizontal Sharding (by userId):

  Users table: 10 million rows → split across 4 shards

  Hash-based: shard = hash(userId) % 4
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Shard 0      │  │ Shard 1      │  │ Shard 2      │  │ Shard 3      │
  │ hash%4 = 0   │  │ hash%4 = 1   │  │ hash%4 = 2   │  │ hash%4 = 3   │
  │ ~2.5M users  │  │ ~2.5M users  │  │ ~2.5M users  │  │ ~2.5M users  │
  └──────────────┘  └──────────────┘  └──────────────┘  └──────────────┘

  Query: getUserById(123)
  → shard = hash(123) % 4 = 3 → query Shard 3 only ✅

  Query: getUsersByCity("Mumbai")
  → which shard? ALL of them! → scatter-gather (slow) ❌

Consistent Hashing (solves rebalancing):
  Hash ring: 0 ────────────── 2³² 
              S1    S2    S3    S4
  
  Add shard S5: only keys between S4 and S5 move
  → Minimal data movement (not full reshuffle)

Rebalancing problem with naive hash:
  Before: hash(key) % 4 → shards 0-3
  After:  hash(key) % 5 → ALMOST ALL keys map to different shards!
  → Massive data migration. Consistent hashing avoids this.
```

### 🗣️ How to Say in Interview
"Sharding distributes data across multiple database instances to scale beyond what a single node can handle. I choose the sharding strategy based on access patterns. Hash-based sharding — hash of the shard key modulo number of shards — gives even distribution but makes range queries expensive since they require scatter-gather across all shards. Range-based sharding is good for time-series data where queries are usually by date range, but it can create hot spots on the most recent shard. For production, I use consistent hashing rather than simple modulo, because adding or removing a shard only requires moving a small fraction of data instead of rehashing everything. The biggest challenges are cross-shard queries, which require coordination, and choosing the right shard key — a bad key leads to hot spots where one shard handles disproportionate load."

### ⚠️ Pitfalls / Gotchas
- **Wrong shard key** → hot spots (all active users on one shard) *(Galat shard key = ek shard pe sara load — toh sharding ka fayda hi nahi)*
- **Cross-shard JOINs** are very expensive or impossible — denormalize data
- **Auto-increment IDs** don't work across shards — use UUIDs or Snowflake IDs
- **Transactions across shards** need 2PC or Saga — much harder than single-DB transactions
- **Resharding** (adding more shards) requires data migration — use consistent hashing to minimize

### 🆚 vs. Comparison
| Strategy | Distribution | Range Query | Hot Spots | Resharding |
|----------|-------------|-------------|-----------|------------|
| Hash-based | Even ⭐ | Scatter-gather | Unlikely | Full reshuffle |
| Consistent hash | Even ⭐ | Scatter-gather | Unlikely | Minimal ⭐ |
| Range-based | Uneven | Efficient ⭐ | Likely | Medium |
| Directory | Flexible ⭐ | Depends | Configurable | Easy ⭐ |

### ⚡ Remember
- **Sharding** = split rows across DB instances by shard key
- **Hash-based** = even distribution, bad for range queries
- **Range-based** = good for time-series, risk of hot spots
- **Consistent hashing** = minimal data movement on reshard *(Consistent hashing = shard add/remove karne pe kam data move hota hai)*
- Cross-shard queries = scatter-gather (expensive)
- UUID/Snowflake IDs for distributed unique keys

### 🔗 Follow-ups
- [Q6 → Distributed Databases (implement sharding)](#q6)
- [Q1 → CAP Theorem (sharding + replication tradeoffs)](#q1)

---

<a id="q6"></a>
## Q6. Distributed Databases (Cassandra, MongoDB, HBase)

### 📝 One-Liner
Cassandra (AP, wide-column, masterless, massive write throughput), MongoDB (CP default, document-based, flexible schema), HBase (CP, wide-column on HDFS, strong consistency + big data scanning).

### 🔑 Quick Answer
**Cassandra**: masterless ring architecture (no single point of failure), AP by default (tunable consistency per query), wide-column store, designed for massive write throughput across datacenters. Use for: time-series, IoT, activity logs. **MongoDB**: document store (JSON-like BSON), CP by default (primary + secondaries), flexible schema with rich queries, aggregation pipeline. Use for: content management, user profiles, catalogs. **HBase**: wide-column store on top of HDFS (Hadoop), CP (strong consistency), RegionServer architecture, great for large-scale sequential reads/writes. Use for: analytics, big data with random read/write on Hadoop data. *(Cassandra = write-heavy distributed; MongoDB = flexible documents; HBase = Hadoop pe strong consistency)*

### 📖 How It Works
```
CASSANDRA (AP, masterless):
  ┌─────┐  ┌─────┐  ┌─────┐  ┌─────┐
  │Node1│──│Node2│──│Node3│──│Node4│── (ring topology)
  └─────┘  └─────┘  └─────┘  └─────┘
  - No master → no SPOF
  - Data replicated to N nodes (replication factor)
  - Write: hash(partition key) → responsible nodes → write to N replicas
  - Consistency level per query: ONE, QUORUM, ALL
  - QUORUM read + QUORUM write = strong consistency (R + W > N)

MONGODB (CP, primary-secondary):
  ┌─────────┐  replication  ┌──────────┐
  │ Primary  │──────────────│Secondary1│
  │ (reads + │              │(read only)│
  │  writes) │──────────────│Secondary2│
  └─────────┘              └──────────┘
  - Primary handles all writes
  - Secondaries async replicate (configurable writeConcern)
  - If Primary dies → election for new Primary (Raft-like)
  - Rich queries: aggregation pipeline, indexes, text search

HBASE (CP, on HDFS):
  ┌─────────────┐
  │  HMaster    │ → assigns regions to RegionServers
  └─────┬───────┘
  ┌─────┴─────┐  ┌───────────┐  ┌───────────┐
  │RegionSvr1 │  │RegionSvr2 │  │RegionSvr3 │
  │ (regions) │  │ (regions) │  │ (regions) │
  └───────────┘  └───────────┘  └───────────┘
        └──── stored on HDFS (distributed filesystem) ────┘
  - Strong consistency (single RegionServer per region)
  - Great for sequential scans on massive datasets
```

### 🆚 vs. Comparison
| Aspect | Cassandra | MongoDB | HBase |
|--------|-----------|---------|-------|
| Data model | Wide-column | Document (BSON) | Wide-column |
| CAP | AP (tunable) | CP (default) | CP |
| Architecture | Masterless ring | Primary-Secondary | Master + RegionServers |
| Write perf | Excellent ⭐ | Good | Good |
| Query lang | CQL (SQL-like) | MQL + Aggregation ⭐ | Scan/Get API |
| Schema | Fixed-ish | Flexible ⭐ | Column families |
| Best for | Time-series, IoT, logs | Content, catalogs, profiles | Big data + Hadoop |
| SPOF | None ⭐ | Primary (auto-failover) | HMaster (HA mode) |

### ⚡ Remember
- **Cassandra** = AP, masterless, insane write throughput, CQL *(Write-heavy + multi-datacenter = Cassandra)*
- **MongoDB** = CP, flexible docs, rich queries, aggregation
- **HBase** = CP, Hadoop ecosystem, strong consistency + big data
- Choose based on: data model + consistency needs + ecosystem
- All three support horizontal scaling via sharding/partitioning

### 🔗 Follow-ups
- [Q5 → Data Sharding (how these DBs partition)](#q5)
- [Q1 → CAP Theorem (CP vs AP)](#q1)
- [Q2 → Consistency Models (tunable consistency)](#q2)
