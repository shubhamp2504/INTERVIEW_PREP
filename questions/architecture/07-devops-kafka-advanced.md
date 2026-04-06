# 🏗️ DevOps, Kafka Advanced & Cross-cutting Architecture (Q1–Q10)

> **Source**: "Citi Bank Senior Java" + "Java Backend Round 1 & 2" + "Top 20 System Design" compilations (2026)  
> **Coverage**: Kafka deep dive (ordering, streams, error handling), DevOps comparisons (K8s vs OpenShift, TDD vs BDD), inter-service security, caching strategies  
> **Level**: 4+ YOE, interviews that probe beyond surface-level knowledge  
> **Cross-refs**: General Kafka in [system-design/03](../system-design/03-ecommerce-payment-systems.md), event sourcing in [system-design/02](../system-design/02-coordination-failover-eventsourcing.md), deployments in [cloud-devops/02](../cloud-devops/02-cloud-infra-processing.md)

---

<a id="q1"></a>
## Q1. How do you ensure Kafka message ordering across multiple topics?

### 📝 One-Liner
You **cannot** guarantee ordering across topics. Use a single topic with partitioning by business key, or if multiple topics are required, use timestamps + a resequencing buffer at the consumer.

### 🔑 Quick Answer
Kafka guarantees ordering **only within a single partition**. Across topics, there is no ordering guarantee. **Strategies:**
1. **Single topic + partition key** — all related events go to the same partition of one topic (e.g., partition by `orderId`). This is the simplest and preferred approach.
2. **Multiple topics + consumer resequencing** — if different topics are unavoidable (e.g., `order-created`, `payment-completed`), add a timestamp or sequence number to each event. Consumer reads from both topics and uses a resequencing buffer to process in order.
3. **Orchestrator pattern** — a central service issues events in sequence; downstream services process only when triggered.

*(Alag alag topics mein order guarantee nahi hota — ek hi topic mein partition key se order maintain karo)*

### 📖 How It Works (Detailed Explanation)

```
❌ Problem — Two topics, no ordering:
  Topic: order-events   →  [order-created at T1]  →  consumer reads at T3
  Topic: payment-events →  [payment-done at T2]   →  consumer reads at T1
  Consumer sees payment before order!

✅ Solution 1 — Single topic, partition by orderId:
  Topic: order-lifecycle
    Partition 0: [order-123-created, order-123-paid, order-123-shipped]
    Partition 1: [order-456-created, order-456-paid]
  → All events for order-123 are in partition 0, in order

✅ Solution 2 — Multi-topic with sequence:
  Event: { "orderId": "123", "seq": 1, "type": "CREATED", "ts": 1000 }
  Event: { "orderId": "123", "seq": 2, "type": "PAID",    "ts": 1050 }
  Consumer: buffer events, process in seq order per orderId
```

### 🗣️ Answering Approach
"Kafka guarantees ordering only within a single partition, not across topics. My preferred approach is using a single topic with a partition key based on the entity ID — for example, orderId as the key ensures all events for one order land in the same partition. If multiple topics are architecturally necessary, I add a sequence number to each event and implement a resequencing buffer in the consumer that holds events until the expected sequence arrives. But honestly, I try to avoid the multi-topic ordering problem by designing around single-topic approaches."

---

<a id="q2"></a>
## Q2. How does Kafka Streams know if a message has been consumed? (Consumer Offset Management)

### 📝 One-Liner
Kafka tracks a **committed offset** per consumer-group per partition in the `__consumer_offsets` internal topic. Consumers control when to commit — auto-commit or manual after processing.

### 🔑 Quick Answer
1. **Offset** = position of the next message to read in a partition
2. **Committed offset** = last position the consumer has confirmed processing
3. Stored in internal topic `__consumer_offsets`
4. **Auto-commit** (`enable.auto.commit=true`): commits every 5s — risk of at-least-once (message reprocessed on restart)
5. **Manual commit** (`commitSync()` / `commitAsync()`): consumer commits after successful processing — enables exactly-once semantics with idempotent writes

*(Consumer apna offset khud track karta hai — auto-commit matlab har 5 second mein commit, manual commit matlab processing ke baad commit)*

### 📖 How It Works (Detailed Explanation)

```
Partition 0: [msg0, msg1, msg2, msg3, msg4, msg5, ...]
                                   ↑             ↑
                           committed offset   current position
                           (confirmed done)   (reading here)

If consumer crashes and restarts:
  → Reads from committed offset (msg3), NOT current position
  → msg3, msg4 will be reprocessed (at-least-once)

To prevent reprocessing:
  1. Commit after every message: commitSync() → slowest, safest
  2. Commit in batches: commitSync() every 100 messages → balanced
  3. Idempotent consumer: even if reprocessed, result is same
```

**Manual commit pattern (recommended):**
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        processMessage(record);  // business logic
    }
    consumer.commitSync();  // commit after successful processing
}
```

### 🗣️ Answering Approach
"Each consumer group has offsets stored in the __consumer_offsets topic. The committed offset represents the last message the consumer has confirmed processing. With auto-commit, Kafka commits the offset periodically regardless of whether processing succeeded — which can cause data loss if the consumer crashes after commit but before processing. I prefer manual commit: I process the batch first, then call commitSync(). Combined with idempotent processing on the consumer side, this gives me effectively exactly-once semantics."

---

<a id="q3"></a>
## Q3. What happens if Kafka consumer encounters an error during message persistence? How to handle deduplication?

### 📝 One-Liner
Use a dead-letter queue for poison messages, retry with backoff, and deduplication via idempotent writes (unique message ID + database constraint or Redis dedup cache).

### 🔑 Quick Answer
**Error handling:**
1. **Transient error** (DB timeout) → Retry with exponential backoff (3 attempts)
2. **Permanent error** (invalid message) → Send to Dead Letter Queue (DLQ) → Don't block partition
3. **Offset management** → Only commit offset after successful processing or DLQ write

**Deduplication:**
1. **Idempotency key** — each message has a unique ID (e.g., `eventId`)
2. **Database constraint** — `UNIQUE(event_id)` column → duplicate insert fails gracefully
3. **Redis dedup cache** — `SETNX event:{eventId}` with TTL → if exists, skip processing

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Spring Kafka error handling with retry + DLQ
@Configuration
public class KafkaConfig {
    @Bean
    public DefaultErrorHandler errorHandler(KafkaTemplate<String, String> template) {
        // DLQ producer
        DeadLetterPublishingRecoverer recoverer = 
            new DeadLetterPublishingRecoverer(template);
        
        // Retry 3 times with backoff, then send to DLQ
        return new DefaultErrorHandler(recoverer, 
            new FixedBackOff(1000L, 3));  // 1s interval, 3 attempts
    }
}

// ✅ Idempotent consumer with DB constraint
@Transactional
public void processEvent(OrderEvent event) {
    // Check if already processed
    if (processedEventRepo.existsByEventId(event.getEventId())) {
        log.info("Duplicate event {}, skipping", event.getEventId());
        return;
    }
    // Process
    orderService.processOrder(event);
    // Mark as processed
    processedEventRepo.save(new ProcessedEvent(event.getEventId()));
}
```

### 🗣️ Answering Approach
"I handle errors in two categories: transient and permanent. For transient errors like database timeouts, I retry with exponential backoff — typically 3 attempts. For permanent errors like deserialization failures, I publish to a dead letter queue so the message doesn't block the partition. For deduplication, each message carries a unique eventId. Before processing, I check a database table with a UNIQUE constraint on eventId — if it already exists, I skip. This handles the case where a message was processed but the offset commit failed, causing reprocessing."

---

<a id="q4"></a>
## Q4. What happens when a class implements two interfaces with conflicting default methods?

### 📝 One-Liner
**Compiler error** — the class must override the conflicting method and explicitly choose which interface's implementation to use via `InterfaceName.super.method()`.

### 🔑 Quick Answer
Java 8+ allows default methods in interfaces. When a class implements two interfaces with the same default method signature, the **compiler throws an error** — it doesn't silently pick one. The class must provide its own implementation:

```java
interface Printable { default void log() { System.out.println("Printable"); } }
interface Loggable  { default void log() { System.out.println("Loggable"); } }

// ❌ Won't compile — "class inherits unrelated defaults for log()"
// class MyClass implements Printable, Loggable { }

// ✅ Must override and choose
class MyClass implements Printable, Loggable {
    @Override
    public void log() {
        Printable.super.log();  // explicitly delegate to Printable's default
    }
}
```

### 📖 How It Works (Detailed Explanation)

**Diamond Problem Resolution Rules (Java):**
1. **Class wins over interface** — if the class has a concrete method, it overrides any interface default
2. **Sub-interface wins over super-interface** — more specific interface takes priority
3. **Conflict = compiler error** — if neither rule applies, the class must explicitly override

```
Rule 1: Class wins
  interface A { default void go() { "A"; } }
  class B { void go() { "B"; } }
  class C extends B implements A { }
  → C.go() = "B" (class B wins)

Rule 2: Sub-interface wins
  interface A { default void go() { "A"; } }
  interface B extends A { default void go() { "B"; } }
  class C implements A, B { }
  → C.go() = "B" (B is more specific)

Rule 3: Ambiguity = must override
  interface A { default void go() { "A"; } }
  interface B { default void go() { "B"; } }
  class C implements A, B {
      @Override
      public void go() { A.super.go(); } // developer chooses
  }
```

### 🗣️ Answering Approach
"When a class implements two interfaces with the same default method, the compiler won't compile — it's a deliberate design decision to avoid the diamond problem. The developer must override the method and explicitly choose using `InterfaceName.super.methodName()`. Java's resolution rules are: class always wins over interface, more specific sub-interface wins over parent interface, and if neither applies, the developer must resolve the ambiguity manually. This is safer than languages that silently pick one."

---

<a id="q5"></a>
## Q5. TDD vs BDD — what is the difference and when to use each?

### 📝 One-Liner
**TDD** = developer tests code units (Red-Green-Refactor), **BDD** = business-readable scenarios linking requirements to tests (Given-When-Then). BDD is TDD with a shared language.

### 🔑 Quick Answer

| Aspect | TDD | BDD |
|--------|-----|-----|
| **Focus** | Code correctness (unit level) | Business behavior (feature level) |
| **Written by** | Developers | Developers + QA + BA (collaborative) |
| **Syntax** | `assertEquals`, `assertTrue` | `Given-When-Then` (Gherkin) |
| **Granularity** | Method/class level | User story / acceptance criteria |
| **Tools** | JUnit, pytest, Jest | Cucumber, SpecFlow, Behave |
| **When to use** | Library code, algorithms, utilities | APIs, user flows, business rules |

*(TDD developer ke liye hai — code level testing. BDD business ke liye hai — non-technical log bhi samajh sake)*

### 📖 How It Works (Detailed Explanation)

```
TDD cycle:
  1. RED: Write a failing test
     @Test void shouldCalculateDiscount() {
         assertEquals(90, calculator.apply(100, 10)); // fails — method doesn't exist
     }
  2. GREEN: Write minimal code to pass
     public int apply(int price, int pct) { return price - (price * pct / 100); }
  3. REFACTOR: Clean up, keep tests green

BDD cycle:
  1. DISCOVER: Discuss scenarios with BA, QA
  2. FORMULATE: Write Gherkin scenarios
     Feature: Discount Calculation
       Scenario: Apply percentage discount
         Given a product priced at 100
         When a 10% discount is applied
         Then the final price should be 90
  3. AUTOMATE: Implement step definitions
     @Given("a product priced at {int}")
     public void productPriced(int price) { this.price = price; }
```

### 🗣️ Answering Approach
"TDD is developer-focused — write a failing unit test first, implement minimal code to pass, then refactor. BDD extends this to business behavior — scenarios are written in Gherkin (Given-When-Then) that non-technical stakeholders can read and validate. I use TDD for utility code, algorithms, and internal logic where the developer is the primary audience. I use BDD when requirements come from business stakeholders and acceptance criteria need to be formally mapped to tests — it creates a shared language between developers, QA, and product."

---

<a id="q6"></a>
## Q6. Kubernetes vs OpenShift — key differences

### 📝 One-Liner
OpenShift = enterprise Kubernetes distribution by Red Hat with built-in CI/CD, image registry, developer tools, stricter security defaults, and commercial support. K8s is the engine; OpenShift is the car.

### 🔑 Quick Answer

| Feature | Kubernetes | OpenShift |
|---------|-----------|-----------|
| **Vendor** | CNCF (open source) | Red Hat (enterprise) |
| **Installation** | Manual / cloud-managed (EKS/GKE/AKS) | Installer (IPI) or managed (ROSA/ARO) |
| **CI/CD** | External (Jenkins, GitHub Actions) | Built-in (Tekton Pipelines, S2I) |
| **Image Registry** | External (DockerHub, ECR) | Integrated internal registry |
| **Security** | Permissive defaults | Strict SCCs (no root by default) |
| **Networking** | CNI plugins (choice) | OVN-Kubernetes (opinionated) |
| **UI** | Dashboard (basic) | Web Console (full-featured) |
| **Support** | Community | Red Hat 24/7 commercial support |
| **Cost** | Free | Licensed per node |

*(Kubernetes = building blocks, khud sab setup karo. OpenShift = ready-made enterprise platform jismein sab kuch built-in hai)*

### 🗣️ Answering Approach
"Kubernetes is the foundation — it's the open-source container orchestrator. OpenShift is Red Hat's enterprise distribution built on top of Kubernetes with everything a developer needs out of the box: built-in CI/CD pipelines via Tekton, an integrated image registry, stricter security defaults that don't allow running containers as root by default, and a full-featured web console. The tradeoff is cost — OpenShift is licensed per node. I'd recommend vanilla Kubernetes when the team has strong DevOps skills and wants flexibility, and OpenShift when the enterprise needs security compliance, commercial support, and a self-service developer experience."

---

<a id="q7"></a>
## Q7. How do you secure communication between microservices?

### 📝 One-Liner
mTLS for transport encryption, API gateway for auth, service mesh for zero-trust, JWT propagation for identity, and network policies for isolation.

### 🔑 Quick Answer
**Security layers (defense in depth):**
1. **Transport**: mTLS (mutual TLS) — both sides present certificates → encrypted + authenticated
2. **Identity**: JWT propagation — gateway validates user token, services propagate it internally
3. **Authorization**: Each service validates permissions for its resources (not just "is token valid")
4. **Network**: K8s NetworkPolicies — services can only talk to allowed services
5. **Service Mesh**: Istio/Linkerd handle mTLS automatically between sidecars

### 📖 How It Works (Detailed Explanation)

```
External request flow:
  Client → API Gateway (validates JWT, rate limits)
         → Service A (mTLS) → Service B (mTLS) → DB

Internal security:
  1. mTLS: Every service has a certificate
     Service A ←cert→ Service B (both verify each other's identity)
     
  2. JWT propagation:
     Gateway extracts userId from JWT → passes as X-User-Id header
     Each service checks: does this user have permission for THIS resource?
     
  3. Network policies (K8s):
     Order Service → can talk to → Payment Service, Inventory Service
     Order Service → CANNOT talk to → User Service (no direct path)
     
  4. Service Mesh (Istio):
     Sidecar proxy handles mTLS automatically
     Traffic policies enforce which services can communicate
```

### 🗣️ Answering Approach
"I implement defense in depth. At the transport layer, mTLS between services ensures encrypted communication and mutual authentication. At the identity layer, the API gateway validates the user's JWT and propagates identity to internal services. Each service then authorizes the request for its own resources — not just token validation but actual permission checks. At the network layer, Kubernetes NetworkPolicies restrict which services can communicate. In production, I prefer a service mesh like Istio that handles mTLS automatically via sidecar proxies without application code changes."

---

<a id="q8"></a>
## Q8. How do you design a database schema for scalability?

### 📝 One-Liner
Start with proper normalization, add strategic denormalization for read performance, use partitioning for large tables, and design for sharding from day one (shard key selection).

### 🔑 Quick Answer

| Strategy | When | How |
|----------|------|-----|
| **Normalization** | Default starting point | Eliminate redundancy, enforce consistency |
| **Denormalization** | Read-heavy, join-heavy queries | Pre-compute joins into materialized views or columns |
| **Indexing** | Slow queries | Composite indexes matching query patterns |
| **Partitioning** | Single table > 100M rows | Range (by date) or hash (by ID) partitioning |
| **Read replicas** | Read-heavy workload | Route reads to replicas, writes to primary |
| **Sharding** | Write-heavy > single DB limit | Horizontal sharding by business key (tenant_id, region) |

### 📖 How It Works (Detailed Explanation)

```
Scalability path:
  Stage 1 (< 1M rows): Normalized schema + proper indexes → sufficient
  Stage 2 (1-100M rows): Add read replicas + query optimization + caching
  Stage 3 (100M+ rows): Table partitioning (by date for time-series)
  Stage 4 (multi-billion): Horizontal sharding by shard key

Shard key selection (most critical decision):
  ✅ Good: tenant_id (even distribution, queries stay within shard)
  ✅ Good: region_id (data locality matches access pattern)
  ❌ Bad: auto-increment id (hot shard — all new data goes to latest shard)
  ❌ Bad: timestamp (same issue — time-based hotspot)
```

### 🗣️ Answering Approach
"I start with a properly normalized schema — premature denormalization is worse than premature optimization. As the data grows, I add read replicas for read-heavy workloads and caching for hot data. When tables exceed 100 million rows, I partition by range (date) or hash (ID). For extreme scale, horizontal sharding becomes necessary — the shard key is the most critical decision. I choose a key that distributes data evenly and matches query patterns: tenant_id or region work well. I avoid auto-increment IDs or timestamps as shard keys because they create hotspots."

---

<a id="q9"></a>
## Q9. How does load balancing work in backend systems?

### 📝 One-Liner
Distribute incoming requests across multiple server instances using algorithms like round-robin, least-connections, or consistent hashing — at L4 (transport) or L7 (application) layer.

### 🔑 Quick Answer

| Algorithm | How | Best For |
|-----------|-----|----------|
| **Round Robin** | Requests distributed equally in rotation | Homogeneous instances, stateless |
| **Weighted Round Robin** | More powerful instances get more requests | Mixed instance types |
| **Least Connections** | Sends to instance with fewest active connections | Variable request duration |
| **IP Hash** | Same client IP → same server | Session affinity (avoid if possible) |
| **Consistent Hashing** | Distributed across ring, minimal redistribution on change | Cache layers, distributed systems |

**L4 vs L7:**
- **L4 (Transport)**: NLB/HAProxy TCP — fast, no content inspection, can't route by URL
- **L7 (Application)**: ALB/Nginx HTTP — can route by path, header, cookie; URL-based routing

### 📖 How It Works (Detailed Explanation)

```
Typical production setup:
  DNS → CloudFlare (CDN + DDoS)
    → AWS ALB (L7, routes by path)
       /api/orders → Order Service instances (3x)
       /api/users  → User Service instances (2x)
       /static/*   → S3 + CloudFront

Health checks:
  ALB checks /actuator/health every 10s
  Unhealthy instance → removed from rotation within 30s
  Returns to rotation after 3 consecutive healthy checks
```

### 🗣️ Answering Approach
"Load balancing distributes traffic across instances. I use L7 (application layer) load balancers like AWS ALB for HTTP services — they can route by URL path, headers, and enable features like sticky sessions. The algorithm depends on the use case: round-robin for stateless services with homogeneous instances, least-connections when request processing times vary significantly. Health checks are critical — the load balancer pings /actuator/health and automatically removes unhealthy instances. I prefer to keep services stateless so any instance can handle any request, avoiding session affinity."

---

<a id="q10"></a>
## Q10. How do you implement caching strategies? (Redis, GemFire, Multi-layer)

### 📝 One-Liner
Multi-layer caching: L1 (in-process Caffeine/Guava for hot data) → L2 (distributed Redis/GemFire for shared cache) → Database, with explicit invalidation and appropriate TTLs per layer.

### 🔑 Quick Answer

| Layer | Technology | Latency | Use Case |
|-------|-----------|---------|----------|
| **L1: In-process** | Caffeine, Guava | ~1μs | Frequently accessed, rarely changing config |
| **L2: Distributed** | Redis, GemFire | ~1ms | Shared across instances, session data, user profiles |
| **L3: CDN** | CloudFront | ~5ms | Static assets, public API responses |
| **DB** | PostgreSQL | ~5-50ms | Source of truth |

**Redis vs GemFire:**

| Aspect | Redis | GemFire (Apache Geode) |
|--------|-------|----------------------|
| Model | Key-value (strings, hashes, sets) | Region-based (rich object model) |
| Consistency | Eventually consistent (replication lag) | Strong consistency (synchronous replication) |
| Query | Basic key lookup, Lua scripts | OQL (SQL-like queries on cached objects) |
| Use case | Session, counters, rate limiting | Financial data, inventory, real-time analytics |
| Cost | Lower, widely adopted | Higher, enterprise (Pivotal/VMware) |

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Spring Boot multi-layer caching
@Configuration
public class CacheConfig {
    @Bean
    public CacheManager caffeineCacheManager() {
        return new CaffeineCacheManager("localProducts"); // L1: in-process
    }
    
    @Bean
    public RedisCacheManager redisCacheManager(RedisConnectionFactory factory) {
        return RedisCacheManager.builder(factory)
            .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofMinutes(10)))  // L2: 10-min TTL
            .build();
    }
}

// ✅ Cache-aside pattern with explicit invalidation
@Cacheable(value = "products", key = "#id", cacheManager = "redisCacheManager")
public Product getProduct(Long id) { return repo.findById(id).orElseThrow(); }

@CacheEvict(value = "products", key = "#product.id")
@Transactional
public Product updateProduct(Product product) { return repo.save(product); }
```

### 🗣️ Answering Approach
"I implement multi-layer caching. L1 is in-process cache using Caffeine — sub-microsecond reads for hot data like configuration. L2 is distributed cache using Redis — shared across all instances for user sessions and frequently queried data. Each layer has appropriate TTLs and explicit invalidation on writes. For Redis vs GemFire: Redis is my default for most use cases — it's battle-tested, widely supported, and handles sessions, counters, and rate limiting well. GemFire is appropriate for financial applications requiring strong consistency and rich querying on cached objects — but it comes with significantly higher operational complexity."

### ⚡ Remember
- Cross-ref: [Cache causing inconsistency → system-design/08 Q6](../system-design/08-backend-scenario-debugging.md#q6)
- Cross-ref: [Connection pooling → spring/07](../languages/java/spring/07-springboot-performance-production.md), [HikariCP → spring/11 Q7](../languages/java/spring/11-springboot-scenario-interviews.md#q7)
