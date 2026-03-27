# 🔗 Coordination, Failover & Event Sourcing (Q7–Q10)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q7"></a>
## Q7. Apache ZooKeeper for Distributed Coordination

### 📝 One-Liner
ZooKeeper is a centralized **coordination service** for distributed systems — it provides leader election, distributed locks, configuration management, and service discovery using a hierarchical znode data model with watches.

### 🔑 Quick Answer
**What**: a CP system (strong consistency via ZAB consensus protocol) that stores data in a tree of **znodes** (like a filesystem). **Core primitives**: **(1) Ephemeral nodes** — auto-deleted when client session ends (used for service registration: alive? node exists). **(2) Sequential nodes** — auto-incrementing suffix (used for distributed locks and leader election). **(3) Watches** — one-time event notifications when a znode changes (used for config change detection). **Typical uses**: leader election (smallest sequential ephemeral node wins), distributed locks, config management, service discovery. **Ensemble**: 3 or 5 ZK servers (odd number) using ZAB consensus. *(ZooKeeper = distributed systems ka traffic police — sab ko coordinate karta hai)*

### 📖 How It Works
```
ZooKeeper Data Model (like a filesystem):
  /
  ├── /services
  │   ├── /services/order-service
  │   │   ├── /services/order-service/instance001 (ephemeral)
  │   │   └── /services/order-service/instance002 (ephemeral)
  │   └── /services/payment-service
  │       └── /services/payment-service/instance001 (ephemeral)
  ├── /config
  │   └── /config/db-url = "jdbc:mysql://prod-db:3306"
  └── /leader-election
      ├── /leader-election/candidate-0000000001 (ephemeral+sequential)
      └── /leader-election/candidate-0000000002 (ephemeral+sequential)

Leader Election:
  1. Each node creates /leader-election/candidate-000000000N (ephemeral+sequential)
  2. Node with LOWEST sequence number = leader
  3. Other nodes watch the node just BEFORE them (not the leader — avoids thundering herd)
  4. If leader dies → ephemeral node deleted → next node gets notified → becomes leader

ZAB Consensus (ZooKeeper Atomic Broadcast):
  ┌────────┐
  │ Leader │ → handles all write requests
  └───┬────┘
      │ broadcast write to followers
  ┌───┴───┐  ┌──────────┐  ┌──────────┐
  │Follow1│  │ Follow2  │  │ Follow3  │
  └───────┘  └──────────┘  └──────────┘
  → Quorum ACK → commit
  → Reads can go to ANY node (linearizable with sync())
```

### 💻 Code Example
```java
// Using Apache Curator (high-level ZooKeeper client)

// 1. Leader Election
LeaderSelector leaderSelector = new LeaderSelector(
    curatorClient, "/leader-election",
    new LeaderSelectorListenerAdapter() {
        @Override
        public void takeLeadership(CuratorFramework client) throws Exception {
            System.out.println("I am the leader now!");
            // do leader work...
            // method returns → leadership released
            Thread.sleep(Long.MAX_VALUE); // hold leadership
        }
    });
leaderSelector.autoRequeue(); // re-enter election when leadership lost
leaderSelector.start();

// 2. Distributed Lock
InterProcessMutex lock = new InterProcessMutex(curatorClient, "/locks/my-resource");
if (lock.acquire(10, TimeUnit.SECONDS)) {
    try {
        // critical section — only one JVM can execute this
        processExclusiveResource();
    } finally {
        lock.release();
    }
}

// 3. Service Discovery
ServiceDiscovery<InstanceDetails> discovery = ServiceDiscoveryBuilder.builder(InstanceDetails.class)
    .client(curatorClient)
    .basePath("/services")
    .thisInstance(ServiceInstance.<InstanceDetails>builder()
        .name("order-service")
        .address("192.168.1.10")
        .port(8080)
        .build())
    .build();
discovery.start(); // registers ephemeral node — auto-removed on crash
```

### 🗣️ How to Say in Interview
"ZooKeeper is a distributed coordination service that provides strongly consistent primitives for building distributed systems. It stores data in a hierarchical tree of znodes, with two key node types: ephemeral nodes that auto-delete when the client disconnects — perfect for service registration and health detection — and sequential nodes with auto-incrementing suffixes used for ordering, like leader election and fair locks. In production, I typically use the Curator framework rather than the raw ZooKeeper API. For leader election, each candidate creates an ephemeral sequential node, and the one with the lowest sequence becomes leader. Others watch only the node immediately before them to avoid thundering herd. When the leader dies, only the next candidate gets notified and takes over."

### ⚠️ Pitfalls / Gotchas
- **Not for large data** — ZK keeps entire dataset in memory. Max ~1MB per znode. It's for coordination, not storage *(ZooKeeper mein data store mat karo — sirf coordination ke liye hai)*
- **Session timeout** — if GC pause > session timeout, ZK thinks client is dead → ephemeral nodes deleted
- **Watch is one-time** — after triggering, you must re-register the watch
- **Thundering herd** — don't make all nodes watch the leader; each watches its predecessor
- **etcd** is the modern alternative (K8s uses etcd, Raft-based, simpler API)

### 🆚 vs. Comparison
| Aspect | ZooKeeper | etcd | Consul |
|--------|-----------|------|--------|
| Consensus | ZAB | Raft | Raft |
| Data model | Hierarchical znodes | Flat key-value | Key-value + services |
| Language | Java | Go | Go |
| Used by | Kafka, HBase, Hadoop | Kubernetes ⭐ | HashiCorp ecosystem |
| Watch model | One-time | Persistent stream ⭐ | Blocking queries |
| Service discovery | Via ephemeral nodes | Via leases | Built-in ⭐ |

### ⚡ Remember
- **ZooKeeper** = CP coordination service (ZAB consensus, quorum)
- **Ephemeral nodes** = auto-delete on disconnect → health/registration
- **Sequential nodes** = auto-increment → leader election, locks
- **Watches** = one-time notifications on znode changes
- Use **Curator** library (not raw ZK API)
- Modern alternative: **etcd** (K8s), **Consul** (HashiCorp)

### 🔗 Follow-ups
- [Q8 → Distributed Locks (locks with ZK and Redis)](#q8)
- Q4 (system-design/01) → Consensus Algorithms (ZAB vs Raft)

---

<a id="q8"></a>
## Q8. Distributed Locks (ZooKeeper vs Redis)

### 📝 One-Liner
Distributed locks ensure **mutual exclusion across multiple JVMs/services** — ZooKeeper locks are CP (correct but slower), Redis Redlock is fast but has edge cases where it can fail.

### 🔑 Quick Answer
**Why**: in a distributed system, `synchronized` and `ReentrantLock` only work within a single JVM. You need a distributed lock when multiple services must coordinate access to a shared resource (e.g., only one service processes a payment). **ZooKeeper lock**: create ephemeral sequential znode → lowest sequence wins lock → on unlock/crash, ephemeral deleted → next node gets lock. CP, correct, but slower (network round-trips to ZK). **Redis lock (Redlock)**: `SET key value NX PX 30000` (set if not exists with TTL). Fast but risky: if Redis master fails before replicating the lock to slave, two clients can hold the "same" lock. **Redlock** (Martin Kleppmann controversy): acquire lock on N/2+1 independent Redis instances. Better than single-node but still debated. *(Distributed lock = multiple servers mein sirf ek ko kaam karne do — ZK safe hai, Redis fast hai)*

### 📖 How It Works
```
ZooKeeper Lock (correct, CP):
  1. Client A creates /locks/resource/lock-0000000001 (ephemeral+sequential)
  2. Client B creates /locks/resource/lock-0000000002 (ephemeral+sequential)
  3. Both check: "Am I the lowest?" → A is lowest → A has lock
  4. B watches A's node (predecessor)
  5. A finishes → deletes node (or crashes → ephemeral auto-deleted)
  6. B gets notification → B is now lowest → B has lock
  
  Guarantees: mutual exclusion ✅, no deadlocks (ephemeral) ✅, fairness (FIFO) ✅

Redis Lock (simple, AP-ish):
  Client A: SET my-lock "owner-A" NX PX 30000
  → returns OK (lock acquired, expires in 30s)
  
  Client B: SET my-lock "owner-B" NX PX 30000
  → returns nil (lock exists, retry)
  
  Client A finishes: DEL my-lock (only if value == "owner-A")
  → Lua script for atomic check-and-delete:
     if redis.call("get", KEYS[1]) == ARGV[1]
     then return redis.call("del", KEYS[1])
     else return 0 end

  Problem: Client A's GC pause > 30s TTL → lock expires → B acquires → BOTH in critical section!
  
Redlock (multi-node Redis):
  5 independent Redis instances
  Client tries to acquire lock on all 5
  If gets lock on >= 3 (majority) within timeout → lock acquired
  Still debated (Kleppmann vs Antirez) for correctness guarantees
```

### 💻 Code Example
```java
// ZooKeeper Lock (Curator)
InterProcessMutex lock = new InterProcessMutex(curatorClient, "/locks/payment-processing");
if (lock.acquire(10, TimeUnit.SECONDS)) {
    try {
        processPayment(orderId);
    } finally {
        lock.release(); // if JVM crashes, ephemeral node auto-deleted
    }
}

// Redis Lock (Redisson — recommended over manual SET NX)
RLock lock = redissonClient.getLock("payment-lock:" + orderId);
try {
    // waitTime=10s, leaseTime=30s (auto-release after 30s)
    if (lock.tryLock(10, 30, TimeUnit.SECONDS)) {
        try {
            processPayment(orderId);
        } finally {
            lock.unlock();
        }
    }
} catch (InterruptedException e) {
    Thread.currentThread().interrupt();
}

// Redisson's Watchdog: if leaseTime not set,
// Redisson auto-extends lock every 10s while holder is alive
// → prevents GC pause expiry problem
RLock lock = redissonClient.getLock("my-lock");
lock.lock(); // leaseTime=-1 → watchdog enabled
try {
    doWork();
} finally {
    lock.unlock();
}
```

### 🗣️ How to Say in Interview
"For distributed mutual exclusion, I choose between ZooKeeper and Redis based on correctness requirements. ZooKeeper locks using Curator's InterProcessMutex are CP — they guarantee mutual exclusion even during failures because ephemeral nodes are automatically deleted when a client disconnects, and the ZAB consensus ensures all nodes agree. For performance-critical scenarios where absolute correctness isn't life-or-death, I use Redis with Redisson which provides a watchdog mechanism that automatically extends the lock while the holder is alive, preventing the classic GC-pause expiry problem. I avoid manual SET NX implementations because it's easy to get the unlock logic wrong — Redisson handles the Lua-script-based atomic unlock and lease extension."

### ⚠️ Pitfalls / Gotchas
- **GC pause + TTL expiry** = two clients in critical section (use fencing tokens to detect) *(GC pause zyada ho gaya toh lock expire ho jaata hai — doosra client lock le leta hai)*
- **Redis replication lag** — master fails before replicating lock → slave promotes → lock lost
- **Forgetting to unlock** → deadlock (use try/finally always)
- **Fencing token**: include a monotonically increasing token with each lock acquisition. The protected resource rejects requests with old tokens → safety even if lock is falsely released

### ⚡ Remember
- **ZK lock** = ephemeral + sequential, correct (CP), slower *(ZK lock safe hai — crash pe auto-release)*
- **Redis lock** = SET NX + TTL, fast, but edge cases (use Redisson)
- **Redisson watchdog** = auto-extend lease while alive
- **Fencing tokens** = monotonic counter to detect stale locks
- Always use **try/finally** to release locks

### 🔗 Follow-ups
- [Q7 → ZooKeeper (coordination primitives)](#q7)
- [Q10 → Failover Mechanisms (what happens when lock holder dies)](#q10)

---

<a id="q9"></a>
## Q9. Event Sourcing + CQRS

### 📝 One-Liner
Event Sourcing stores **every state change as an immutable event** (not just current state); CQRS separates **read** and **write** models so each can be optimized independently.

### 🔑 Quick Answer
**Event Sourcing**: instead of storing current state (balance=500), store all events that led to it: `AccountCreated(1000)`, `Withdrawn(300)`, `Deposited(100)`, `Withdrawn(300)`. Current state = replay all events. **Benefits**: complete audit trail, time-travel debugging, derive new projections from same events. **CQRS (Command Query Responsibility Segregation)**: separate the write model (handles commands, validates business rules, stores events) from the read model (optimized denormalized views for queries). Write to event store → project events into read-optimized views (separate DB). **Together**: Event Sourcing provides the event log; CQRS projects those events into query-optimized read models. *(Event Sourcing = har change record karo; CQRS = likhne aur padhne ka model alag rakho)*

### 📖 How It Works
```
TRADITIONAL (just save current state):
  Account: { id: 1, balance: 500 }
  → No history of HOW we got to 500

EVENT SOURCING (save every change):
  Event Store:
  | Seq | Event              | Data               | Timestamp  |
  |-----|--------------------|--------------------|------------|
  | 1   | AccountCreated     | { balance: 1000 }  | 2024-01-01 |
  | 2   | MoneyWithdrawn     | { amount: 300 }    | 2024-01-05 |
  | 3   | MoneyDeposited     | { amount: 100 }    | 2024-01-10 |
  | 4   | MoneyWithdrawn     | { amount: 300 }    | 2024-01-15 |
  
  Current state: replay events → 1000 - 300 + 100 - 300 = 500
  State at Jan 5: replay events 1-2 → 1000 - 300 = 700 (time travel!)

CQRS (separate read + write):
  ┌────────────────┐         ┌──────────────────────┐
  │ COMMAND SIDE   │         │ QUERY SIDE           │
  │ (Write Model)  │         │ (Read Model)         │
  │                │         │                      │
  │ Validate rules │  event  │ Denormalized views   │
  │ Emit events    │──────→  │ Optimized for reads  │
  │ Event Store    │  async  │ Separate DB (Elastic) │
  └────────────────┘         └──────────────────────┘
      ↑ Commands               ↑ Queries
  (PlaceOrder)              (GetOrdersByUser)
  
  Write DB: normalized (events)
  Read DB: denormalized (pre-joined, indexed for queries)
  → Eventual consistency between write and read side
```

### 💻 Code Example
```java
// Event Sourcing — Events
public sealed interface AccountEvent {
    record AccountCreated(String accountId, BigDecimal initialBalance) implements AccountEvent {}
    record MoneyDeposited(String accountId, BigDecimal amount) implements AccountEvent {}
    record MoneyWithdrawn(String accountId, BigDecimal amount) implements AccountEvent {}
}

// Aggregate — rebuild state from events
public class AccountAggregate {
    private String accountId;
    private BigDecimal balance = BigDecimal.ZERO;
    private final List<AccountEvent> uncommittedEvents = new ArrayList<>();

    // Rebuild from history
    public static AccountAggregate fromHistory(List<AccountEvent> events) {
        AccountAggregate account = new AccountAggregate();
        events.forEach(account::apply);
        return account;
    }

    // Command → validate → emit event
    public void withdraw(BigDecimal amount) {
        if (balance.compareTo(amount) < 0) {
            throw new InsufficientFundsException("Balance: " + balance);
        }
        apply(new MoneyWithdrawn(accountId, amount));
        uncommittedEvents.add(new MoneyWithdrawn(accountId, amount));
    }

    // Apply event to update state
    private void apply(AccountEvent event) {
        switch (event) {
            case AccountCreated e -> { accountId = e.accountId(); balance = e.initialBalance(); }
            case MoneyDeposited e -> balance = balance.add(e.amount());
            case MoneyWithdrawn e -> balance = balance.subtract(e.amount());
        }
    }
}

// CQRS — Read-side projection (event handler)
@Component
public class AccountProjection {
    @Autowired private AccountReadRepository readRepo; // separate read DB

    @EventHandler
    public void on(MoneyDeposited event) {
        AccountView view = readRepo.findById(event.accountId());
        view.setBalance(view.getBalance().add(event.amount()));
        view.setLastTransaction("Deposit: " + event.amount());
        readRepo.save(view); // update denormalized read model
    }
}
```

### 🗣️ How to Say in Interview
"Event sourcing stores every state change as an immutable event rather than just overwriting current state. This gives you a complete audit trail, the ability to time-travel by replaying events to any point, and flexibility to create new read models by projecting the same events differently. I combine it with CQRS to separate the write model — where commands are validated against business rules and events are emitted — from the read model — where events are projected into denormalized views optimized for specific query patterns. The tradeoff is eventual consistency between write and read sides, and increased complexity. I use frameworks like Axon or EventStore for implementation. Event sourcing is particularly valuable in domains with strong audit requirements like finance or compliance."

### ⚠️ Pitfalls / Gotchas
- **Event schema evolution** — changing event structure is hard (use upcasting or versioned events) *(Event ka format change karna mushkil hai — versioning plan karo pehle se)*
- **Eventual consistency** — read model lags behind write. UI may show stale data
- **Snapshots needed** — replaying 10M events per aggregate is slow → periodically snapshot aggregate state
- **Not for simple CRUD** — overkill for basic apps. Use when audit trail and complex domain matter
- **Event ordering** — across partitions, ordering is not guaranteed → use partition keys wisely

### 🆚 vs. Comparison
| Aspect | Traditional CRUD | Event Sourcing + CQRS |
|--------|-----------------|----------------------|
| Storage | Current state only | All events (immutable log) |
| Audit trail | Add separate audit table | Built-in ⭐ |
| Complexity | Low ⭐ | High |
| Time travel | Not possible | Replay to any point ⭐ |
| Read perf | Same DB for read/write | Separate optimized read DB ⭐ |
| Consistency | Strong (same DB) ⭐ | Eventual (async projection) |
| Best for | Simple CRUD apps | Complex domains, finance, audit |

### ⚡ Remember
- **Event Sourcing** = store changes, not state *(Current state = sab events replay karo)*
- **CQRS** = separate write model (commands) from read model (queries)
- Events are **immutable** — never delete or update events
- **Snapshots** to avoid replaying millions of events
- Use for audit-heavy domains (finance, compliance). Don't over-apply.

### 🔗 Follow-ups
- Q9 (architecture/03) → Event-Driven with Kafka (event transport)
- [Q10 → Failover Mechanisms (recovery from event log)](#q10)

---

<a id="q10"></a>
## Q10. Failover Mechanisms

### 📝 One-Liner
Failover is automatic switching to a standby system when the primary fails — **active-passive** (standby takes over), **active-active** (both serve traffic, one absorbs other's load), using health checks (heartbeats) and DNS/load-balancer rerouting.

### 🔑 Quick Answer
**Active-Passive (hot standby)**: primary handles all traffic, secondary replicates data but sits idle. On primary failure, secondary promotes (DNS update or VIP failover). Simple but wastes standby resources. Example: RDS Multi-AZ. **Active-Active**: both nodes handle traffic simultaneously. On one failure, the other absorbs all traffic. Better resource utilization but requires data synchronization (conflict resolution). Example: multi-region DynamoDB Global Tables. **Health checks**: heartbeat pings (every 5-10s), TCP/HTTP health probes from load balancer or orchestrator. **Failover types**: **(1) Database** — primary/replica promotion (RDS, PostgreSQL streaming replication). **(2) Application** — K8s pod restart, load balancer removes unhealthy instance. **(3) DNS** — Route53 health check → failover to secondary region. *(Failover = primary fail ho jaye toh backup turant le le — downtime minimize karo)*

### 📖 How It Works
```
ACTIVE-PASSIVE:
  Normal:  Client → Load Balancer → [Primary ✅] ──replication──→ [Standby 💤]
  Failure: Client → Load Balancer → [Primary ❌] 
           LB detects failure (health check timeout)
           → Switch to standby
  After:   Client → Load Balancer → [Standby ✅ (now primary)]

ACTIVE-ACTIVE:
  Normal:  Client → LB → [Node A ✅] ←──sync──→ [Node B ✅]
           Traffic split 50/50
  Failure: Client → LB → [Node A ❌]
           LB removes A from pool
  After:   Client → LB → [Node B ✅] (handles 100%)

K8s Failover (Pod level):
  ┌──────────────────────────────────┐
  │ ReplicaSet: 3 pods desired       │
  │                                  │
  │   Pod1 ✅  Pod2 ✅  Pod3 ❌      │
  │                                  │
  │ Pod3 fails health check          │
  │ → Kubelet kills Pod3             │
  │ → K8s schedules Pod4 on new node │
  │                                  │
  │   Pod1 ✅  Pod2 ✅  Pod4 ✅      │
  └──────────────────────────────────┘

Database Failover (PostgreSQL):
  Primary ──streaming replication──→ Replica
  Primary crashes → detect via pg_isready or heartbeat
  → Promote replica: pg_ctl promote
  → Update connection string (PgBouncer/HAProxy)
  → Application reconnects to new primary
  RTO (Recovery Time Objective): 30-60 seconds typical
```

### 🗣️ How to Say in Interview
"I implement failover at multiple levels. At the application tier, Kubernetes handles pod failures automatically through liveness probes and ReplicaSets — if a pod fails, K8s terminates it and schedules a replacement. At the database tier, I use active-passive with streaming replication — for AWS RDS Multi-AZ, the standby in another AZ is automatically promoted on primary failure with under 60 seconds failover time. For cross-region failover, I use active-active with DynamoDB Global Tables or Route53 DNS failover with health checks. The key metrics are RTO — how quickly we recover — and RPO — how much data we can afford to lose. For critical financial systems, I target zero RPO with synchronous replication and less than 30 seconds RTO."

### ⚠️ Pitfalls / Gotchas
- **Split-brain** — both nodes think they're primary (use fencing/STONITH to kill the old primary) *(Split-brain = dono primary ban gaye — data corrupt ho sakta hai)*
- **Async replication lag** — failover to replica may lose recent writes (RPO > 0)
- **DNS TTL** — DNS-based failover is limited by client TTL caching (set low TTL like 60s)
- **Cascading failures** — failover increases load on remaining nodes → they also fail → use circuit breakers
- **False positives** — aggressive health check timeouts may cause unnecessary failover (flapping)

### 🆚 vs. Comparison
| Aspect | Active-Passive | Active-Active |
|--------|---------------|--------------|
| Resource usage | Standby wasted 💤 | Both utilized ⭐ |
| Complexity | Low ⭐ | High (sync/conflict resolution) |
| Failover speed | Seconds (promote) | Instant ⭐ (absorb traffic) |
| Data sync | Async replication | Bidirectional sync |
| Cost | Medium | Higher (double compute) |
| Conflict risk | None ⭐ | Write conflicts possible |

### ⚡ Remember
- **Active-Passive** = standby waits, promotes on failure (RDS Multi-AZ, PostgreSQL streaming)
- **Active-Active** = both serve traffic, one absorbs on failure
- **Health checks** = heartbeats, HTTP probes, TCP checks
- **RTO** = recovery time, **RPO** = data loss tolerance
- Watch for **split-brain** (fencing) and **cascading failures** (circuit breakers)

### 🔗 Follow-ups
- [Q7 → ZooKeeper (leader election for failover coordination)](#q7)
- Q20 (architecture/01) → Circuit Breakers (prevent cascading failure)
