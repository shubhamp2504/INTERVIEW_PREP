# 🧩 In-Memory Data Grids & Actor Concurrency (Q22–Q23)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q22"></a>
## Q22. In-Memory Data Grids (Hazelcast, Infinispan)

### 📝 One-Liner
An IMDG (In-Memory Data Grid) stores data **distributed across a cluster in RAM** — providing ultra-fast access with automatic partitioning, replication, and compute capabilities beyond simple caching (Hazelcast, Infinispan, Apache Ignite).

### 🔑 Quick Answer
**What**: a distributed in-memory key-value store that partitions data across cluster nodes, replicates for fault tolerance, and supports compute-on-data (execute code where data lives — no network transfer). **vs. Redis**: Redis is a centralized cache/store; IMDG is a distributed grid where data is co-located with compute. **Hazelcast**: embedded (same JVM) or client-server mode. Distributed Map, Queue, Lock, ExecutorService. Near-cache for local reads. Auto-discovery (multicast/TCP). **Infinispan**: Red Hat, embedded in WildFly/JBoss. Supports multiple modes: local, replicated (all nodes have all data), distributed (partitioned), scattered. Can be used as Hibernate L2 cache. **Use cases**: session storage, distributed caching, real-time analytics, collocated compute (process data where it lives), near-real-time event processing. *(IMDG = RAM mein distributed data store — cache se zyada — compute bhi data ke paas hota hai)*

### 📖 How It Works
```
Distributed Data Grid:
  ┌────────────┐  ┌────────────┐  ┌────────────┐
  │  Node 1    │  │  Node 2    │  │  Node 3    │
  │            │  │            │  │            │
  │ Partition  │  │ Partition  │  │ Partition  │
  │ 0 (primary)│  │ 1 (primary)│  │ 2 (primary)│
  │ 1 (backup) │  │ 2 (backup) │  │ 0 (backup) │
  │            │  │            │  │            │
  │ Near-cache │  │ Near-cache │  │ Near-cache │
  │ (hot local)│  │ (hot local)│  │ (hot local)│
  └────────────┘  └────────────┘  └────────────┘
  
  PUT("key1", value):
    hash("key1") → Partition 1 → Node 2 (primary)
    → backup on Node 3 (async replication)
  
  GET("key1"):
    hash("key1") → Partition 1 → Node 2
    OR near-cache hit (local) → 0 network latency!

Compute on Data (EntryProcessor):
  Instead of:  GET data → transfer to client → process → PUT back
  Do:          Send code TO the data → execute ON the node → no transfer
  
  → Massive savings when processing large values
  → Like stored procedures but in-memory + distributed

Node Failure:
  Node 2 dies → Node 3 promotes backup of Partition 1 to primary
  → Node 1 creates new backup of Partition 1
  → Automatic, transparent to application
```

### 💻 Code Example
```java
// Hazelcast — Embedded Mode (same JVM)
HazelcastInstance hz = Hazelcast.newHazelcastInstance();
IMap<String, UserSession> sessions = hz.getMap("sessions");

// Distributed Map operations
sessions.put("session-123", new UserSession("Shubham", Instant.now()),
    30, TimeUnit.MINUTES); // TTL
UserSession session = sessions.get("session-123");

// EntryProcessor — compute on data (no network transfer of value)
sessions.executeOnKey("session-123", (EntryProcessor<String, UserSession, Void>) entry -> {
    UserSession s = entry.getValue();
    s.setLastAccess(Instant.now());
    s.incrementPageViews();
    entry.setValue(s);  // updates IN-PLACE on the owning node
    return null;
});

// Distributed SQL (Hazelcast 5+)
hz.getSql().execute("SELECT * FROM sessions WHERE page_views > ?", 10)
    .forEach(row -> System.out.println(row.getObject("user_name")));

// Near-Cache configuration (local read cache for hot data)
NearCacheConfig nearCacheConfig = new NearCacheConfig()
    .setInMemoryFormat(InMemoryFormat.OBJECT)
    .setMaxIdleSeconds(300)
    .setEvictionConfig(new EvictionConfig()
        .setMaxSizePolicy(MaxSizePolicy.ENTRY_COUNT)
        .setSize(1000));

MapConfig mapConfig = new MapConfig("sessions")
    .setNearCacheConfig(nearCacheConfig)
    .setBackupCount(1);  // 1 sync backup per partition

// Infinispan — as Hibernate L2 Cache
// hibernate.cfg.xml or application.properties:
// spring.jpa.properties.hibernate.cache.use_second_level_cache=true
// spring.jpa.properties.hibernate.cache.region.factory_class=
//   org.infinispan.hibernate.cache.v60.InfinispanRegionFactory
@Entity
@Cacheable
@Cache(usage = CacheConcurrencyStrategy.READ_WRITE)
public class Product {
    @Id private Long id;
    private String name;
    private BigDecimal price;
}
```

### 🗣️ How to Say in Interview
"An in-memory data grid like Hazelcast goes beyond simple caching — it distributes data across cluster nodes with automatic partitioning and replication, and supports compute-on-data via EntryProcessors where code executes on the node that owns the data, eliminating network transfer of large values. I use Hazelcast embedded in Spring Boot for distributed session storage and real-time aggregations. The near-cache feature provides sub-millisecond reads for hot data by caching frequently accessed entries locally on each node. For JPA heavy applications, I configure Infinispan as Hibernate's L2 cache — entities are cached in the grid and shared across all application instances, dramatically reducing database load. The key advantage over Redis is data locality and embedded mode — no separate infrastructure to manage."

### ⚠️ Pitfalls / Gotchas
- **Serialization cost** — data must be serializable across nodes. Use efficient serializers (Hazelcast Portable, Protobuf) not Java serialization *(Java serialization slow hai — Portable ya Protobuf use karo)*
- **Memory pressure** — all data in RAM. Monitor heap closely, configure eviction policies
- **Split-brain** — network partition can create two sub-clusters. Configure merge policies (latest-update-wins)
- **Near-cache inconsistency** — local cache may be stale. Configure invalidation strategy
- **Class loading** — embedded mode: all nodes must have same class versions on classpath

### 🆚 vs. Comparison
| Aspect | Hazelcast | Redis | Infinispan |
|--------|-----------|-------|-----------|
| Architecture | Embedded or client-server | Client-server only | Embedded or client-server |
| Distribution | Auto-partitioned ⭐ | Manual sharding / Cluster | Auto-partitioned ⭐ |
| Compute on data | EntryProcessor ⭐ | Lua scripts | Interceptors |
| Near-cache | Built-in ⭐ | Client-side caching | Built-in |
| Persistence | Write-through/behind | RDB/AOF | Cache stores |
| JPA Integration | Limited | None | Hibernate L2 Cache ⭐ |
| Language | Java-native ⭐ | Any (protocol) | Java-native |

### ⚡ Remember
- **IMDG** = distributed RAM storage with partitioning + replication + compute
- **Hazelcast** = embedded Java, EntryProcessor (compute on data), near-cache *(Data ke paas code bhejo — network latency bachao)*
- **Infinispan** = Red Hat, Hibernate L2 cache integration
- **Near-cache** = local hot data copy for sub-ms reads
- Use for: sessions, real-time aggregations, collocated compute
- Watch: serialization, memory pressure, split-brain

### 🔗 Follow-ups
- [Q23 → Akka Actor Model (alternative concurrent architecture)](#q23)
- Q7 (architecture/02) → Caching Architecture (Redis vs IMDG)

---

<a id="q23"></a>
## Q23. Akka Actor-Based Concurrency

### 📝 One-Liner
The Actor Model replaces shared-state concurrency (locks, synchronized) with **message-passing** — each actor has private state, processes one message at a time, and communicates only via async messages — Akka is the JVM implementation (now Pekko after license change).

### 🔑 Quick Answer
**Problem with threads + locks**: shared mutable state + locks = deadlocks, race conditions, hard to reason about. **Actor Model**: each actor is a lightweight entity with: **(1)** private mutable state (no sharing), **(2)** a mailbox (message queue), **(3)** behavior (how to process each message). Actors process **one message at a time** (no locking needed) and communicate only by sending messages (fire-and-forget). **Supervision**: parent actor supervises children — if child fails, parent decides: restart, stop, escalate, or resume. No try/catch everywhere. **Akka** (Lightbend) → relicensed to BSL. **Apache Pekko**: open-source fork of Akka (community-driven, compatible API). **Akka Cluster**: actors distributed across JVMs, location-transparent messaging. **Use cases**: high-concurrency systems (10M+ entities), IoT device management, real-time trading, game servers, chat systems. *(Actor = ek choti entity jo sirf messages se baat karti hai — no locks, no shared state)*

### 📖 How It Works
```
Actor Model:
  ┌──────────────┐   message    ┌──────────────┐
  │   Actor A    │─────────────→│   Actor B    │
  │  ┌────────┐  │              │  ┌────────┐  │
  │  │ State  │  │              │  │ State  │  │
  │  │(private)│  │              │  │(private)│  │
  │  └────────┘  │              │  └────────┘  │
  │  ┌────────┐  │   message    │  ┌────────┐  │
  │  │Mailbox │  │←─────────────│  │Mailbox │  │
  │  │ [msg3] │  │              │  │ [msg1] │  │
  │  │ [msg2] │  │              │  │        │  │
  │  │ [msg1] │  │              │  │        │  │
  │  └────────┘  │              │  └────────┘  │
  └──────────────┘              └──────────────┘
  Process one message at a time → no locks needed!

Supervision Hierarchy:
  /user (guardian)
    ├── /user/orderProcessor (supervisor)
    │   ├── /user/orderProcessor/validator-1
    │   ├── /user/orderProcessor/validator-2
    │   └── /user/orderProcessor/validator-3
    │       (validator-3 throws exception)
    │       → orderProcessor decides: restart validator-3
    │       → state reset, no other actors affected
    └── /user/paymentProcessor

vs. Traditional Threads:
  Threads: 10,000 threads max (stack memory ~1MB each = 10GB)
  Actors: 10,000,000 actors easily (few hundred bytes each)
  → Actors are extremely lightweight
```

### 💻 Code Example
```java
// Akka Typed (or Apache Pekko equivalent)
// Define messages (protocol)
public sealed interface OrderCommand {
    record PlaceOrder(String orderId, BigDecimal amount, ActorRef<OrderEvent> replyTo)
        implements OrderCommand {}
    record CancelOrder(String orderId, ActorRef<OrderEvent> replyTo)
        implements OrderCommand {}
}

public sealed interface OrderEvent {
    record OrderPlaced(String orderId) implements OrderEvent {}
    record OrderCancelled(String orderId) implements OrderEvent {}
    record OrderFailed(String orderId, String reason) implements OrderEvent {}
}

// Actor behavior
public class OrderActor extends AbstractBehavior<OrderCommand> {
    private final Map<String, Order> orders = new HashMap<>(); // private state!

    public static Behavior<OrderCommand> create() {
        return Behaviors.setup(OrderActor::new);
    }

    private OrderActor(ActorContext<OrderCommand> context) {
        super(context);
    }

    @Override
    public Receive<OrderCommand> createReceive() {
        return newReceiveBuilder()
            .onMessage(PlaceOrder.class, this::onPlaceOrder)
            .onMessage(CancelOrder.class, this::onCancelOrder)
            .build();
    }

    private Behavior<OrderCommand> onPlaceOrder(PlaceOrder cmd) {
        // No synchronization needed — one message at a time!
        Order order = new Order(cmd.orderId(), cmd.amount(), "PLACED");
        orders.put(cmd.orderId(), order);
        cmd.replyTo().tell(new OrderPlaced(cmd.orderId()));
        return this; // same behavior
    }

    private Behavior<OrderCommand> onCancelOrder(CancelOrder cmd) {
        Order order = orders.get(cmd.orderId());
        if (order != null) {
            order.setStatus("CANCELLED");
            cmd.replyTo().tell(new OrderCancelled(cmd.orderId()));
        } else {
            cmd.replyTo().tell(new OrderFailed(cmd.orderId(), "Not found"));
        }
        return this;
    }
}

// Spawn and use actors
ActorSystem<OrderCommand> system = ActorSystem.create(
    OrderActor.create(), "order-system");

system.tell(new PlaceOrder("ORD-1", new BigDecimal("99.99"), replyAdapter));

// Supervision — restart child on failure
Behavior<OrderCommand> supervisedBehavior = Behaviors.supervise(OrderActor.create())
    .onFailure(Exception.class, SupervisorStrategy.restart()
        .withLimit(3, Duration.ofMinutes(1))); // max 3 restarts/min
```

### 🗣️ How to Say in Interview
"The Actor Model fundamentally changes how we handle concurrency — instead of shared mutable state protected by locks, each actor owns its private state and communicates only through asynchronous messages. Since actors process one message at a time, there are no race conditions or deadlocks by design. This is particularly powerful when you need to manage millions of concurrent entities — like IoT devices or user sessions — because actors are extremely lightweight compared to threads. Supervision trees replace try/catch error handling — a parent actor defines a strategy for child failures: restart, stop, or escalate. I've used Akka for high-throughput order processing where each order is an actor, enabling us to handle millions of concurrent orders without complex locking. With Lightbend's license change, I'd recommend Apache Pekko for new projects as it's the community fork with a compatible API."

### ⚠️ Pitfalls / Gotchas
- **Don't block inside actors** — blocking one actor blocks its dispatcher thread, affecting all actors on that dispatcher. Use async I/O or dedicated blocking dispatcher *(Actor ke andar block mat karo — doosre actors bhi ruk jaayenge)*
- **Mailbox overflow** — if producer sends faster than consumer processes, mailbox grows → OOM. Use backpressure or bounded mailbox
- **Ask pattern overhead** — `ask` (request-reply) creates a temporary actor + timeout. Prefer `tell` (fire-and-forget) when possible
- **Testing** — actor testing requires `TestProbe` to intercept messages. More complex than unit testing regular classes
- **Akka → Pekko** — Akka relicensed to BSL 1.1. For OSS: use Apache Pekko (drop-in replacement, import change)

### 🆚 vs. Comparison
| Aspect | Threads + Locks | Actor Model (Akka/Pekko) | Virtual Threads (Java 21) |
|--------|----------------|------------------------|--------------------------|
| State | Shared mutable | Private per actor ⭐ | Shared (same as threads) |
| Synchronization | Locks, synchronized | Messages ⭐ (no locks) | Locks (but cheap to block) |
| Concurrency units | ~10K threads | ~10M actors ⭐ | ~10M virtual threads ⭐ |
| Error handling | try/catch | Supervision trees ⭐ | try/catch |
| Distribution | Manual | Cluster (transparent) ⭐ | Single JVM only |
| Complexity | Medium | High (paradigm shift) | Low ⭐ |
| Best for | Simple concurrency | Massive-scale entities ⭐ | I/O-bound high concurrency ⭐ |

### ⚡ Remember
- **Actor** = private state + mailbox + behavior (no shared state, no locks)
- Process **one message at a time** → no race conditions *(Ek actor ek waqt mein ek message process karta hai — lock ki zaroorat nahi)*
- **Supervision** = parent handles child failures (restart/stop/escalate)
- Actors are lightweight (~300 bytes) vs threads (~1MB stack)
- **Pekko** = OSS fork of Akka (use for new projects)
- Don't block in actors; use tell over ask

### 🔗 Follow-ups
- [Q22 → In-Memory Data Grids (distributed state alternative)](#q22)
- Q1-Q10 (multithreading files) → Traditional Java concurrency
