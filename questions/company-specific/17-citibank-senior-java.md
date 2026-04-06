# 🏦 Citi Bank — Senior Java Developer Interview (Q1–Q18)

> **Source**: Real Citi Bank interview experience — Senior Java/Kafka Developer (2026)  
> **Format**: Interview narrative with detailed answers + cross-references  
> **Level**: 5-8 YOE Senior Java Developer (Kafka-heavy role)  
> **Theme**: Heavy on Kafka internals, Java advanced, architecture patterns, deployment strategies

---

<a id="q1"></a>
## Q1. What is Kafka? Why do we use it?

### 📝 One-Liner
Kafka = distributed event streaming platform for high-throughput, fault-tolerant messaging between microservices with persistent log storage.

### 🔑 Quick Answer
**Apache Kafka** is a distributed event streaming platform used for:
1. **Async communication** between microservices (decoupling producers/consumers)
2. **Event-driven architecture** (order placed → inventory deducted → notification sent)
3. **Log aggregation** (centralized log collection from all services)
4. **Stream processing** (real-time analytics, fraud detection)
5. **Data pipeline** (CDC from DB → data lake/warehouse)

**Why Kafka over RabbitMQ/SQS?** Kafka retains messages (log-based); consumers can replay. Handles millions of messages/sec. Built-in partitioning for parallelism.

*(Kafka ek distributed messaging system hai — producers events publish karte hain, consumers subscribe karte hain. Kafka messages replay bhi kar sakta hai — yeh RabbitMQ se alag hai)*

### ⚡ Remember
- Cross-ref: [Kafka in ecommerce → system-design/03-ecommerce-payment-systems.md](../system-design/03-ecommerce-payment-systems.md)

---

<a id="q2"></a>
## Q2. Explain Zookeeper, Broker, Topic, Partition in Kafka

### 📝 One-Liner
**Broker** = Kafka server node, **Topic** = named channel for messages, **Partition** = ordered subset of a topic for parallelism, **Zookeeper** = cluster coordinator (being replaced by KRaft).

### 🔑 Quick Answer

| Component | Role | Analogy |
|-----------|------|---------|
| **Broker** | Kafka server node that stores data | Post office branch |
| **Cluster** | Group of brokers working together | All post office branches in a city |
| **Topic** | Named category/feed of messages | A specific mailbox (e.g., "orders") |
| **Partition** | Ordered, immutable sequence within a topic | Numbered slots in the mailbox |
| **Zookeeper** | Cluster coordinator — leader election, config | City postal HQ managing branches |
| **KRaft** | Kafka's own consensus (replacing ZK, Kafka 3.3+) | Internal management, no external dependency |

```
Kafka Cluster:
  Broker 0 ── Topic: orders ── Partition 0: [msg0, msg1, msg2]
  Broker 1 ── Topic: orders ── Partition 1: [msg3, msg4, msg5]
  Broker 2 ── Topic: orders ── Partition 2: [msg6, msg7]
                                             ↑ replicated across brokers
  
  Zookeeper: manages broker registration, leader election
  KRaft (new): brokers self-manage without external Zookeeper
```

### 🗣️ Answering Approach
"A Kafka cluster consists of multiple brokers — each broker is a server that stores partitions. Topics are logical channels named by business domain like 'orders' or 'payments'. Each topic is split into partitions for parallelism — messages within a partition are strictly ordered. Zookeeper handles cluster metadata and leader election, but Kafka 3.3+ introduced KRaft mode which eliminates the Zookeeper dependency. In production, I configure replication factor of 3 so each partition has copies on 3 brokers for fault tolerance."

---

<a id="q3"></a>
## Q3. How do you ensure message ordering across multiple Kafka topics?

### 📝 One-Liner
You can't — ordering is only guaranteed within a single partition. Use single-topic with partition key, or multi-topic with sequence numbers and resequencing buffer.

### 🔑 Quick Answer
> **Full answer in**: [architecture/07-devops-kafka-advanced.md Q1](../architecture/07-devops-kafka-advanced.md#q1)

**Summary**: Kafka guarantees ordering only within a partition. For cross-topic ordering: (1) redesign to single topic with partition key, (2) add sequence numbers and build a consumer resequencing buffer, or (3) use an orchestrator pattern.

---

<a id="q4"></a>
## Q4. Stream API vs Parallel Stream — when to use in real projects?

### 📝 One-Liner
Stream for sequential I/O-bound processing; parallelStream for CPU-bound computation on large collections (10K+ elements) with no shared mutable state.

### 🔑 Quick Answer

| Aspect | stream() | parallelStream() |
|--------|----------|-------------------|
| **Execution** | Single thread (main) | Fork-Join pool (multiple threads) |
| **Order** | Guaranteed | May not maintain encounter order |
| **Overhead** | None | Thread spawning, merging |
| **Best for** | I/O operations, small collections | CPU-heavy ops on 10K+ elements |
| **Danger** | None | Shared state → race conditions |

```java
// ✅ Good: CPU-bound, large collection, no shared state
List<Double> results = hugeList.parallelStream()  // 100K+ elements
    .map(item -> computeExpensiveScore(item))      // CPU-intensive
    .collect(Collectors.toList());

// ❌ Bad: small collection — overhead > benefit
List<String> small = names.parallelStream()        // 50 elements
    .map(String::toUpperCase)                       // trivial operation
    .collect(Collectors.toList());
// parallelStream overhead is slower than sequential!
```

### 🗣️ Answering Approach
"I use parallelStream only when three conditions are met: the collection is large (10K+), the operation is CPU-intensive, and there's no shared mutable state. For I/O operations, parallel streams are actually harmful because they share the common Fork-Join pool — one slow I/O call blocks threads for other parallel operations across the entire JVM. In practice, I use regular streams for 90% of cases. For I/O parallelism, I use CompletableFuture with a custom executor instead."

### ⚡ Remember
- Cross-ref: [Parallel streams deep dive → core/07-java-interview-tricky.md](../languages/java/core/07-java-interview-tricky.md)
- Cross-ref: [CompletableFuture parallel model → spring/11 Q4](../languages/java/spring/11-springboot-scenario-interviews.md#q4)

---

<a id="q5"></a>
## Q5. What's new in Java 17?

### 📝 One-Liner
Sealed classes, pattern matching for instanceof, records, text blocks, switch expressions, and strong encapsulation of JDK internals.

### 🔑 Quick Answer
**Key Java 17 features (LTS):**
1. **Sealed Classes** — `sealed class Shape permits Circle, Square` — restrict inheritance
2. **Pattern Matching for instanceof** — `if (obj instanceof String s)` — no casting needed
3. **Records** — `record Point(int x, int y) {}` — immutable data carriers
4. **Text Blocks** — `"""multi-line string"""` — clean SQL, JSON in code
5. **Switch Expressions** — `var result = switch(x) { case 1 -> "one"; ... };`
6. **Strong Encapsulation** — JDK internals locked down by default

### ⚡ Remember
- Cross-ref: [Java 8→21 evolution → core/17-java-evolution-8to21.md](../languages/java/core/17-java-evolution-8to21.md)

---

<a id="q6"></a>
## Q6. Kafka Streams — how do you know if a message has been consumed? How to handle errors during persistence with deduplication?

### 📝 One-Liner
Consumer offset tracking in `__consumer_offsets` topic + manual commit after processing + DLQ for errors + idempotent writes for deduplication.

### 🔑 Quick Answer
> **Full answer in**: [architecture/07-devops-kafka-advanced.md Q2](../architecture/07-devops-kafka-advanced.md#q2) (offset management) + [Q3](../architecture/07-devops-kafka-advanced.md#q3) (error handling + dedup)

**Summary**: Kafka tracks committed offsets per consumer-group/partition. Manual commit after processing ensures at-least-once delivery. Dedup via idempotent writes (UNIQUE constraint on eventId) + DLQ for poison messages.

---

<a id="q7"></a>
## Q7. What is Event Sourcing?

### 📝 One-Liner
Store state changes as a sequence of immutable events, not as current state — replay events to reconstruct state at any point in time.

### 🔑 Quick Answer
Instead of storing `Order { status: SHIPPED, total: 100 }`, store:
```
Event 1: OrderCreated { orderId: 123, items: [...], total: 100 }
Event 2: PaymentReceived { orderId: 123, amount: 100 }
Event 3: OrderShipped { orderId: 123, trackingId: "TRK456" }
```

**Benefits**: Full audit trail, temporal queries ("what was the state at T2?"), event replay, CQRS pairing.
**Challenges**: Event schema evolution, eventual consistency, snapshot management for performance.

### ⚡ Remember
- Cross-ref: [Event sourcing deep dive → system-design/02-coordination-failover-eventsourcing.md](../system-design/02-coordination-failover-eventsourcing.md)

---

<a id="q8"></a>
## Q8. Default method conflict — what happens when a class implements two interfaces with the same default method?

### 📝 One-Liner
Compiler error — must override and explicitly choose via `InterfaceName.super.method()`.

### 🔑 Quick Answer
> **Full answer in**: [architecture/07-devops-kafka-advanced.md Q4](../architecture/07-devops-kafka-advanced.md#q4)

**Summary**: Java's diamond problem resolution — compiler forces explicit choice. Three rules: class wins over interface, sub-interface wins over super, ambiguity requires manual override.

---

<a id="q9"></a>
## Q9. SOLID principles — explain with examples

### 📝 One-Liner
Five OOP design principles for maintainable, extensible code: Single Responsibility, Open-Closed, Liskov Substitution, Interface Segregation, Dependency Inversion.

### 🔑 Quick Answer
> **Full answer in**: [oops-patterns/01-oop-principles.md](../oops-patterns/01-oop-principles.md) + [oops-patterns/02-oop-real-project-examples.md Q5](../oops-patterns/02-oop-real-project-examples.md#q5)

**Quick summary with banking context (Citi-specific):**
- **S**: `TransactionService` handles transactions, `NotificationService` handles alerts — not one class doing both
- **O**: New payment method (crypto) → add `CryptoPaymentStrategy`, don't modify existing PaymentService
- **L**: `SavingsAccount` and `CheckingAccount` both work wherever `Account` is expected
- **I**: `Depositable`, `Withdrawable`, `InterestBearing` — fixed deposits don't implement Withdrawable
- **D**: `TransactionService` depends on `AccountRepository` interface, not `JpaAccountRepository`

---

<a id="q10"></a>
## Q10. Liskov Substitution Principle — why do we need inheritance?

### 📝 One-Liner
Liskov says: anywhere a parent type is used, a child type must be substitutable without breaking behavior. Inheritance enables code reuse + polymorphism when LSP is respected.

### 🔑 Quick Answer
**Liskov Substitution Principle (LSP):**
- If `Bird` has `fly()`, then `Penguin extends Bird` violates LSP because penguin can't fly
- Fix: separate `FlyingBird` and `NonFlyingBird` hierarchies, or use composition over inheritance

**Why we need inheritance:**
1. **Code reuse** — `BaseEntity` provides id, audit fields to all entities
2. **Polymorphism** — `List<Account>` can hold Savings, Checking, Fixed — process uniformly
3. **Template Method** — base class defines algorithm skeleton, subclass overrides specific steps

**When to prefer composition over inheritance:**
- When the child doesn't truly "is-a" parent (Penguin is NOT a FlyingBird)
- When you need behavior from multiple sources (Java doesn't support multiple inheritance)
- "Favor composition over inheritance" — Gang of Four

### 🗣️ Answering Approach
"Liskov Substitution ensures that inheritance isn't misused. If I have a Transaction class with a reverse() method, any subclass like WireTransfer or InternalTransfer must support reverse() without breaking. If CashDeposit can't be reversed, it shouldn't extend Transaction — that violates LSP. I'd restructure with a Reversible interface instead. We need inheritance for genuine 'is-a' relationships where polymorphism simplifies the code — in banking, all account types are genuine specializations of Account."

---

<a id="q11"></a>
## Q11. TDD vs BDD — what's the difference?

### 📝 One-Liner
TDD = developer tests code units (Red-Green-Refactor). BDD = business-readable scenarios (Given-When-Then). BDD extends TDD to include non-technical stakeholders.

### 🔑 Quick Answer
> **Full answer in**: [architecture/07-devops-kafka-advanced.md Q5](../architecture/07-devops-kafka-advanced.md#q5)

**Banking context**: "At Citi, TDD for internal utility code and algorithms. BDD for regulatory features where business analysts validate scenarios: Given a wire transfer over $10,000, When submitted, Then compliance alert is triggered."

---

<a id="q12"></a>
## Q12. Canary vs Blue-Green deployment — when to use which?

### 📝 One-Liner
**Blue-Green** = two identical environments, instant switchover. **Canary** = gradual rollout (1% → 10% → 50% → 100%) with monitoring at each stage.

### 🔑 Quick Answer

| Aspect | Blue-Green | Canary |
|--------|-----------|--------|
| **Rollout** | 0% → 100% instant switch | 1% → 10% → 50% → 100% gradual |
| **Risk** | Full blast if bug exists | Limited blast radius (only canary %) |
| **Cost** | 2× infrastructure (both envs running) | 1× infra + canary instances |
| **Rollback** | Instant (switch back to blue) | Remove canary instances |
| **Best for** | Small teams, simple apps | Large-scale, high-traffic systems |

### ⚡ Remember
- Cross-ref: [Deployment strategies → cloud-devops/02-cloud-infra-processing.md](../cloud-devops/02-cloud-infra-processing.md)
- Cross-ref: [Zero-downtime K8s → system-design/08-backend-scenario-debugging.md Q14](../system-design/08-backend-scenario-debugging.md#q14)

---

<a id="q13"></a>
## Q13. Orchestration vs Choreography in microservices

### 📝 One-Liner
**Orchestration** = central coordinator controls the flow (saga orchestrator). **Choreography** = each service reacts to events independently (event-driven, no central controller).

### 🔑 Quick Answer

| Aspect | Orchestration | Choreography |
|--------|--------------|--------------|
| **Control** | Central orchestrator | Decentralized (events) |
| **Coupling** | Services coupled to orchestrator | Services loosely coupled |
| **Visibility** | Easy to see full flow | Must trace events across services |
| **Complexity** | Orchestrator becomes bottleneck/SPoF | Event chain hard to debug |
| **Example** | Order saga orchestrator calls payment → inventory → shipping | OrderCreated event → PaymentService listens, InventoryService listens |

**Banking context**: "For Citi's wire transfer flow (compliance check → debit → credit → notification), I'd use orchestration because the flow is strict, sequential, and auditable. For notification fan-out after a transaction, choreography — each notification channel (email, SMS, push) reacts independently."

### ⚡ Remember
- Cross-ref: [Orchestration vs choreography → system-design/01-distributed-systems-fundamentals.md](../system-design/01-distributed-systems-fundamentals.md)

---

<a id="q14"></a>
## Q14. Kubernetes vs OpenShift

### 📝 One-Liner
OpenShift = enterprise Kubernetes by Red Hat with built-in CI/CD, image registry, strict security, and commercial support. K8s is the engine; OpenShift is the car.

### 🔑 Quick Answer
> **Full answer in**: [architecture/07-devops-kafka-advanced.md Q6](../architecture/07-devops-kafka-advanced.md#q6)

**Banking context**: "Citi likely uses OpenShift because of enterprise support, strict security defaults (SCCs prevent running as root — critical for banking compliance), built-in image scanning, and Red Hat's 24/7 support SLA. Vanilla K8s requires building all these controls yourself."

---

<a id="q15"></a>
## Q15. What happens when you use a mutable object as a HashMap key?

### 📝 One-Liner
If the key's hashCode changes after insertion, the entry becomes unreachable — you can't find it, can't remove it, and the map effectively leaks memory.

### 🔑 Quick Answer
```java
// ❌ Mutable key — silent data loss
List<String> key = new ArrayList<>(List.of("a", "b"));
Map<List<String>, String> map = new HashMap<>();
map.put(key, "value");

key.add("c");  // mutates key → hashCode changes!

map.get(key);  // null! — looks in wrong bucket
map.get(List.of("a", "b"));  // null! — original key's bucket, but equals() fails
// The entry is LOST — still in memory, but unreachable
```

**Rules**: HashMap keys should be immutable: `String`, `Integer`, `record`, or custom classes with final fields and immutable hashCode/equals.

### ⚡ Remember
- Cross-ref: [HashMap keys deep dive → core/06-java-collections-gotchas.md](../languages/java/core/06-java-collections-gotchas.md)

---

<a id="q16"></a>
## Q16. What are the advantages of checked exceptions?

### 📝 One-Liner
Checked exceptions force callers to handle failure cases at compile time — making error paths explicit and preventing silent failures.

### 🔑 Quick Answer

| Aspect | Checked | Unchecked |
|--------|---------|-----------|
| **Enforcement** | Compiler forces handling | No compile-time check |
| **Usage** | Recoverable failures (IO, network) | Programming errors (NPE, IndexOOB) |
| **Advantage** | Explicit error contract | Less boilerplate |
| **Disadvantage** | Verbose, catches leak up the call chain | Easy to forget handling |

**Banking context**: `InsufficientFundsException` as checked → every caller MUST handle insufficient balance. Can't accidentally ignore it.

### ⚡ Remember
- Cross-ref: [Exception handling → core/04-java8-exception-handling.md](../languages/java/core/04-java8-exception-handling.md)

---

<a id="q17"></a>
## Q17. What is "effectively final" in Java?

### 📝 One-Liner
A variable that is not declared `final` but is never reassigned after initialization — required for lambda captures and anonymous classes.

### 🔑 Quick Answer
```java
// ✅ Effectively final — never reassigned
String name = "Shubham";  // not declared final, but never reassigned
Runnable r = () -> System.out.println(name);  // compiles — effectively final

// ❌ Not effectively final — reassigned
String name = "Shubham";
name = "Pathak";  // reassigned!
Runnable r = () -> System.out.println(name);  // WON'T COMPILE
```

**Why**: Lambdas capture variable values, not references. If the variable could change after capture, the behavior would be confusing/unsafe.

### ⚡ Remember
- Cross-ref: [Effectively final → core/13-java-language-deep-dive.md](../languages/java/core/13-java-language-deep-dive.md)

---

<a id="q18"></a>
## Q18. What is CQRS? When would you use it?

### 📝 One-Liner
**Command Query Responsibility Segregation** — separate write models (commands) from read models (queries) with different databases/schemas optimized for each.

### 🔑 Quick Answer

```
Traditional: Same model for reads and writes
  App → OrderEntity → PostgreSQL ← reads and writes

CQRS: Separate models
  Writes: App → Command Handler → Write DB (normalized, PostgreSQL)
                    ↓ (event published)
  Reads:  App → Query Handler  → Read DB (denormalized, Elasticsearch/Redis)
```

**When to use:**
- Read and write patterns are very different (complex writes, simple reads or vice versa)
- Read-heavy system that needs materialized views
- Paired with Event Sourcing for full audit + temporal queries

**Banking context**: "Transaction processing (writes) needs ACID guarantees on PostgreSQL. Account statement queries (reads) need fast search across millions of transactions — optimized in Elasticsearch. CQRS separates these concerns."

### ⚡ Remember
- Cross-ref: [CQRS + Event Sourcing → system-design/02-coordination-failover-eventsourcing.md](../system-design/02-coordination-failover-eventsourcing.md)
