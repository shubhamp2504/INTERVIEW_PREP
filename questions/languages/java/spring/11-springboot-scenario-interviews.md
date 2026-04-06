# 🌱 Spring Boot — Scenario-Based Interview Questions (Q1–Q12)

> **Source**: Frequently asked Spring Boot scenario questions from recent Java interviews (2026)  
> **Coverage**: Multiple main methods, open-closed API design, synchronous/parallel API patterns, @Transactional pitfalls, @OneToMany ownership, connection pool exhaustion, production slowness debugging, query optimization, API layering, annotation usage  
> **Level**: 2–5 YOE Java Backend Developer

---

<a id="q1"></a>
## Q1. Is it possible for a Spring Boot project to contain more than one main method? If yes, which one will execute?

### 📝 One-Liner
Yes, multiple `main()` methods can exist in different classes, but only the one configured as the **start class** in `pom.xml` / `build.gradle` or `MANIFEST.MF` runs.

### 🔑 Quick Answer
Java allows `public static void main(String[])` in every class — it's not restricted to one. Spring Boot, however, needs ONE entry point class annotated with `@SpringBootApplication`. When running `mvn spring-boot:run` or `java -jar`, the **start class** is resolved via: (1) `<start-class>` in `pom.xml`, (2) `Main-Class` in `MANIFEST.MF`, (3) the class with `@SpringBootApplication`. If multiple `@SpringBootApplication` classes exist, the build plugin will fail unless you explicitly configure the start class. *(Haan, multiple main methods ho sakte hain — lekin Spring Boot sirf wahi chalayega jo start-class mein configure hai, baaki ignore ho jaayenge)*

### 📖 How It Works (Detailed Explanation)

**Resolution order:**
```
1. pom.xml → <start-class> property
   <properties>
       <start-class>com.example.MyApplication</start-class>
   </properties>

2. spring-boot-maven-plugin → mainClass configuration
   <plugin>
       <configuration>
           <mainClass>com.example.MyApplication</mainClass>
       </configuration>
   </plugin>

3. MANIFEST.MF → Main-Class attribute (inside JAR)

4. Auto-detection → looks for @SpringBootApplication
   (fails if multiple found without explicit config)
```

**Real scenario**: You might have a second main method for a **batch job launcher** or **data migration script** in the same project. Configure them as separate Maven profiles or Gradle tasks.

### 🗣️ Answering Approach
"Yes, multiple main methods can exist in a Spring Boot project — Java has no restriction on that. But Spring Boot needs one designated entry point. It resolves which main to run via the start-class configuration in pom.xml or the spring-boot-maven-plugin's mainClass property. If multiple @SpringBootApplication classes exist without explicit config, the build plugin throws an error. In practice, I've seen secondary main methods used for batch processors or migration scripts, configured as separate Maven profiles."

### ⚡ Remember
- Multiple `main()` = valid Java; single `@SpringBootApplication` = Spring Boot convention
- Explicit `<start-class>` or `<mainClass>` resolves ambiguity
- Use Maven profiles for secondary entry points (batch, migration)

---

<a id="q2"></a>
## Q2. If we have a login API, how to add login via OTP, CAPTCHA, or username/password without creating a new API or modifying the existing one?

> 🔥 **Frequently asked** — tests Open-Closed Principle (OCP) practical application

### 📝 One-Liner
Use the **Strategy Pattern** with a factory — define a `LoginStrategy` interface, create implementations for each auth type, and resolve the strategy dynamically based on the request.

### 🔑 Quick Answer
This is a classic **Open-Closed Principle** (SOLID → O) implementation. Create a `LoginStrategy` interface with an `authenticate()` method. Implement `UsernamePasswordStrategy`, `OtpStrategy`, `CaptchaStrategy`. Use a `LoginStrategyFactory` that resolves the strategy based on a `loginType` field in the request. The existing controller endpoint remains unchanged — it delegates to the factory. Adding a new auth method means creating a new Strategy class only — no code modification needed. *(Strategy Pattern + Factory use karo — naya login type add karna ho toh sirf naya class banao, existing code touch nahi karna)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Step 1: Strategy Interface
public interface LoginStrategy {
    LoginResponse authenticate(LoginRequest request);
    String getType(); // "OTP", "PASSWORD", "CAPTCHA"
}

// ✅ Step 2: Implementations
@Component
public class PasswordLoginStrategy implements LoginStrategy {
    public LoginResponse authenticate(LoginRequest request) {
        // validate username + password from DB
        return new LoginResponse(generateJwt(user));
    }
    public String getType() { return "PASSWORD"; }
}

@Component
public class OtpLoginStrategy implements LoginStrategy {
    public LoginResponse authenticate(LoginRequest request) {
        // verify OTP from cache/DB
        return new LoginResponse(generateJwt(user));
    }
    public String getType() { return "OTP"; }
}

// ✅ Step 3: Factory (auto-discovers strategies from Spring context)
@Component
public class LoginStrategyFactory {
    private final Map<String, LoginStrategy> strategies;
    
    public LoginStrategyFactory(List<LoginStrategy> strategyList) {
        this.strategies = strategyList.stream()
            .collect(Collectors.toMap(LoginStrategy::getType, Function.identity()));
    }
    
    public LoginStrategy getStrategy(String type) {
        return Optional.ofNullable(strategies.get(type))
            .orElseThrow(() -> new UnsupportedLoginException(type));
    }
}

// ✅ Step 4: Controller — UNCHANGED for new login types
@PostMapping("/api/login")
public ResponseEntity<LoginResponse> login(@RequestBody LoginRequest request) {
    LoginStrategy strategy = loginStrategyFactory.getStrategy(request.getLoginType());
    return ResponseEntity.ok(strategy.authenticate(request));
}
```

**Adding CAPTCHA login later**: Just create `CaptchaLoginStrategy implements LoginStrategy` — Spring auto-discovers it, factory picks it up. Zero changes to controller or existing strategies.

### 🗣️ Answering Approach
"I'd apply the Strategy Pattern combined with Spring's component scanning. I create a LoginStrategy interface with authenticate() and getType() methods. Each auth mechanism — password, OTP, CAPTCHA — gets its own implementation. A factory class collects all strategies from Spring context and resolves by login type. The controller stays unchanged: it takes the request, asks the factory for the right strategy, and delegates. Adding a new login method means creating one new class — no existing code is modified. This is the Open-Closed Principle in action."

### ⚡ Remember
- **Strategy + Factory + Spring DI** = cleanest OCP implementation
- Factory uses `List<LoginStrategy>` injection — Spring auto-discovers all implementations
- Controller never changes — new auth type = new @Component class only
- Cross-ref: [SOLID Principles → oops-patterns/01-oop-principles.md](../../oops-patterns/01-oop-principles.md)

---

<a id="q3"></a>
## Q3. If an API calls another API sequentially (one-by-one, waiting for each to complete), what is this model called?

### 📝 One-Liner
**Synchronous (blocking) execution model** — also called **sequential chaining** or **orchestration pattern** in microservices context.

### 🔑 Quick Answer
When API A calls API B, waits for response, then calls API C using B's result — this is **synchronous/blocking execution**. In Spring Boot: `RestTemplate` (blocking), `WebClient.block()` (blocking wrapper on reactive). In microservices, this pattern is called **orchestration** — a coordinator service calls other services in sequence. The problem: latency compounds (total = sum of all call latencies) and one slow service blocks everything. *(Ek API doosre ko call kare aur wait kare — yeh synchronous/blocking model hai — total time sab ka sum hoga)*

### 📖 How It Works (Detailed Explanation)

```
Synchronous Chain:
  Client → Service A → Service B → Service C → DB
           wait...      wait...     wait...
           ←200ms       ←300ms      ←150ms
  Total latency = 200 + 300 + 150 = 650ms (cumulative)
```

**Spring Boot implementation:**
```java
// ✅ Synchronous — each call waits for previous to complete
public OrderResponse placeOrder(OrderRequest request) {
    // Call 1: Validate inventory (waits for response)
    InventoryResponse inventory = restTemplate.postForObject(
        "http://inventory-service/api/check", request, InventoryResponse.class);
    
    // Call 2: Process payment (waits for Call 1 to finish first)
    PaymentResponse payment = restTemplate.postForObject(
        "http://payment-service/api/charge", request, PaymentResponse.class);
    
    // Call 3: Create shipment (waits for Call 2)
    ShipmentResponse shipment = restTemplate.postForObject(
        "http://shipping-service/api/create", request, ShipmentResponse.class);
    
    return new OrderResponse(inventory, payment, shipment);
}
```

**Downsides**: Latency stacking, tight coupling, one failure blocks entire chain, thread blocked during wait (wastes server resources).

### 🗣️ Answering Approach
"This is the synchronous or blocking execution model. Each API call waits for the previous one to complete before proceeding. In microservices, this is also called the orchestration pattern where a coordinator service drives the sequence. The key downside is latency stacking — total response time equals the sum of all individual call times. If any service is slow, the entire chain is slow. In Spring Boot, RestTemplate and synchronous Feign Client follow this model. For scenarios where calls are independent, I'd prefer the asynchronous/parallel model instead."

---

<a id="q4"></a>
## Q4. If APIs execute concurrently (parallel execution), what is this model called?

### 📝 One-Liner
**Asynchronous (non-blocking) parallel execution model** — fire multiple calls simultaneously and aggregate results.

### 🔑 Quick Answer
When independent API calls execute simultaneously (not waiting for each other), it's the **asynchronous parallel execution** model. In Spring Boot: `CompletableFuture` with `@Async`, `WebClient` (reactive/non-blocking), or `Virtual Threads` (Java 21+). Total latency = **max** of individual call times (not sum). In microservices, independent parallel calls are a form of **scatter-gather pattern**. *(Sab API ek saath call karo, sabse slow wala kitna time leta hai — utna hi total time lagega)*

### 📖 How It Works (Detailed Explanation)

```
Parallel Execution:
  Client → Service A → Service B (simultaneously)
                     → Service C (simultaneously)
           ←200ms     ←300ms     ←150ms
  Total latency = max(200, 300, 150) = 300ms (parallel)
  vs Sequential  = 200 + 300 + 150  = 650ms (sequential)
```

**Spring Boot implementation:**
```java
// ✅ Parallel with CompletableFuture
public OrderResponse placeOrder(OrderRequest request) {
    CompletableFuture<InventoryResponse> inventoryFuture = CompletableFuture.supplyAsync(
        () -> restTemplate.postForObject("http://inventory-service/api/check", request, InventoryResponse.class));
    
    CompletableFuture<PaymentResponse> paymentFuture = CompletableFuture.supplyAsync(
        () -> restTemplate.postForObject("http://payment-service/api/charge", request, PaymentResponse.class));
    
    CompletableFuture<ShipmentResponse> shipmentFuture = CompletableFuture.supplyAsync(
        () -> restTemplate.postForObject("http://shipping-service/api/create", request, ShipmentResponse.class));
    
    // Wait for ALL to complete
    CompletableFuture.allOf(inventoryFuture, paymentFuture, shipmentFuture).join();
    
    return new OrderResponse(inventoryFuture.join(), paymentFuture.join(), shipmentFuture.join());
}

// ✅ With WebClient (reactive, non-blocking)
public Mono<OrderResponse> placeOrderReactive(OrderRequest request) {
    Mono<InventoryResponse> inventory = webClient.post().uri("/api/check")...;
    Mono<PaymentResponse> payment = webClient.post().uri("/api/charge")...;
    
    return Mono.zip(inventory, payment)
        .map(tuple -> new OrderResponse(tuple.getT1(), tuple.getT2()));
}
```

### 🗣️ Answering Approach
"This is the asynchronous parallel execution model, also called scatter-gather in distributed systems. Independent API calls fire simultaneously, and we aggregate results when all complete. In Spring Boot, I implement this with CompletableFuture.supplyAsync() for the blocking approach or WebClient with Mono.zip() for the reactive approach. With Java 21's virtual threads, we can also use structured concurrency. The benefit is latency = max(individual times) instead of sum. I only use parallel when calls are truly independent — if call B depends on call A's result, they must stay sequential."

### ⚡ Remember
- **Sequential**: latency = sum, use when calls are dependent
- **Parallel**: latency = max, use when calls are independent
- `CompletableFuture.allOf()` for blocking, `Mono.zip()` for reactive
- **Scatter-gather** = microservices term for parallel fan-out + aggregate

---

<a id="q5"></a>
## Q5. In Hibernate @OneToMany mapping, where do you place the annotation? Explain how it works.

### 📝 One-Liner
`@OneToMany` goes on the **parent entity's collection field** (the "one" side); `@ManyToOne` goes on the **child entity's FK field** (the "many" side). The child owns the relationship.

### 🔑 Quick Answer
In a Department → Employee relationship (one department has many employees): `@OneToMany` is placed on `List<Employee>` in the `Department` class. `@ManyToOne` is placed on `Department department` in the `Employee` class. The **child (Employee) owns the relationship** via `@JoinColumn`. Parent uses `mappedBy` to indicate it's the inverse side. Without `mappedBy`, Hibernate creates a **join table** — which is usually undesirable. *(Parent class mein @OneToMany lagta hai collection pe, child class mein @ManyToOne lagta hai FK field pe — child owner hai relationship ka)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Parent — Department (the "one" side)
@Entity
public class Department {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @OneToMany(mappedBy = "department", cascade = CascadeType.ALL, orphanRemoval = true)
    private List<Employee> employees = new ArrayList<>();
    
    // Helper methods for bidirectional sync
    public void addEmployee(Employee emp) {
        employees.add(emp);
        emp.setDepartment(this);  // sync both sides!
    }
}

// ✅ Child — Employee (the "many" side, OWNS the relationship)
@Entity
public class Employee {
    @Id @GeneratedValue
    private Long id;
    private String name;
    
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "department_id")  // FK column in Employee table
    private Department department;
}
```

**Database:**
```sql
-- Employee table has the FK (child owns the relationship)
CREATE TABLE employee (
    id BIGINT PRIMARY KEY,
    name VARCHAR(255),
    department_id BIGINT REFERENCES department(id)  -- FK here
);
```

**Common mistakes:**
| Mistake | Problem | Fix |
|---------|---------|-----|
| No `mappedBy` | Hibernate creates unnecessary join table | Add `mappedBy = "department"` |
| No `@JoinColumn` on child | Default column naming may not match | Explicit `@JoinColumn(name = "department_id")` |
| Eager loading on `@OneToMany` | N+1 queries, OOM on large collections | Use `FetchType.LAZY` (default for OneToMany) |
| Forgetting bidirectional sync | Parent/child out of sync in same persistence context | Use helper methods |

### 🗣️ Answering Approach
"@OneToMany goes on the parent entity's collection field — like List<Employee> in the Department class. @ManyToOne goes on the child entity's foreign key field — the Department reference in Employee. The child is the owning side of the relationship because it has the foreign key column. The parent uses mappedBy to indicate it's the inverse side. Without mappedBy, Hibernate creates an unnecessary join table. I always use FetchType.LAZY on the @ManyToOne side and helper methods for bidirectional sync to avoid common pitfalls."

### ⚡ Remember
- `@OneToMany(mappedBy)` = parent (inverse side)
- `@ManyToOne` + `@JoinColumn` = child (owning side, has FK)
- Always use `FetchType.LAZY` — N+1 prevention
- Cross-ref: [JPA relationships → database/01-jpa-sql-transactions.md](../../database/01-jpa-sql-transactions.md)

---

<a id="q6"></a>
## Q6. @Transactional is present but rollback doesn't happen — why?

> 🔥 **Frequently asked** — common production pitfall

### 📝 One-Liner
Top causes: checked exception (default = no rollback), self-invocation (proxy bypass), wrong propagation, or non-public method.

### 🔑 Quick Answer
**5 common reasons rollback fails:**
1. **Checked exception** — `@Transactional` only rolls back on **unchecked exceptions** (RuntimeException) by default. Checked exceptions do NOT trigger rollback unless you add `rollbackFor = Exception.class`.
2. **Self-invocation** — calling a @Transactional method from another method in the same class bypasses the proxy → no transaction.
3. **Non-public method** — Spring AOP proxies only intercept public methods.
4. **Wrong propagation** — `REQUIRES_NEW` creates a separate transaction that commits independently.
5. **Exception caught internally** — if you catch the exception before it reaches the proxy, Spring never sees it. *(Sabse common galti: checked exception throw karte ho lekin @Transactional sirf RuntimeException pe rollback karta hai by default)*

### 📖 How It Works (Detailed Explanation)

```java
// ❌ Problem 1: Checked exception — NO rollback by default
@Transactional
public void updateOrder(Order order) throws IOException {
    orderRepository.save(order);
    fileService.upload(order.getReceipt()); // throws IOException (checked)
    // DB changes are COMMITTED even though IOException was thrown!
}

// ✅ Fix: Add rollbackFor
@Transactional(rollbackFor = Exception.class)
public void updateOrder(Order order) throws IOException { ... }

// ❌ Problem 2: Self-invocation — proxy bypassed
@Service
public class OrderService {
    public void processOrder(Order order) {
        // THIS calls @Transactional method internally — proxy NOT involved!
        this.saveOrder(order);  // NO transaction!
    }
    
    @Transactional
    public void saveOrder(Order order) {
        orderRepository.save(order);
    }
}

// ✅ Fix: Inject self or extract to another service
@Autowired private OrderService self; // inject the proxy
public void processOrder(Order order) {
    self.saveOrder(order);  // goes through proxy → transaction works
}

// ❌ Problem 3: Non-public method
@Transactional
private void saveOrder(Order order) { ... } // IGNORED — must be public

// ❌ Problem 4: Exception caught internally
@Transactional
public void updateOrder(Order order) {
    try {
        orderRepository.save(order);
        externalService.notify(order); // throws RuntimeException
    } catch (Exception e) {
        log.error("Failed", e); // Exception swallowed — proxy never sees it!
    }
    // Transaction COMMITS because no exception propagated
}
```

### 🗣️ Answering Approach
"The most common reason is checked exceptions. @Transactional only rolls back on RuntimeException by default — if a checked exception like IOException is thrown, the transaction commits. The fix is rollbackFor = Exception.class. The second common issue is self-invocation — calling a @Transactional method from within the same class bypasses the Spring proxy, so no transaction is applied. The fix is to inject self or refactor into a separate service. Third is exception swallowing — if you catch the exception inside the method, Spring never sees it. And fourth, @Transactional only works on public methods due to Spring AOP limitations."

### ⚡ Remember
- Default rollback: only `RuntimeException` and `Error`, NOT checked exceptions
- Self-invocation = proxy bypass → no transaction
- Exception caught inside = Spring never sees it → no rollback
- Cross-ref: [@Transactional internals → spring/](../spring/)

---

<a id="q7"></a>
## Q7. Connection pool gets exhausted under load — what causes this?

### 📝 One-Liner
Leaked connections (not closed), long-running queries holding connections, @Transactional on long operations, undersized pool, or external API calls inside transactions.

### 🔑 Quick Answer
**Root causes (in order of frequency):**
1. **Connection leaks** — code that opens connections manually but doesn't close in `finally`
2. **Long-running transactions** — `@Transactional` on methods that call external APIs or do heavy processing, holding the DB connection the entire time
3. **N+1 queries** — lazy loading triggers dozens of queries, each holding pool time
4. **Undersized pool** — default HikariCP pool = 10 connections, insufficient for production
5. **Slow queries** — unindexed queries holding connections for seconds

### 📖 How It Works (Detailed Explanation)

```
HikariCP Pool (10 connections):
  Request 1 → gets conn #1 → long-running query (5s) → ❌ holds connection
  Request 2 → gets conn #2 → calls external API inside @Transactional (3s) → ❌
  Request 3 → gets conn #3
  ...
  Request 10 → gets conn #10
  Request 11 → WAITS (pool exhausted) → ConnectionTimeoutException after 30s!
```

**Diagnosis steps:**
```yaml
# ✅ Enable HikariCP leak detection
spring:
  datasource:
    hikari:
      leak-detection-threshold: 30000  # warn if connection held > 30s
      maximum-pool-size: 20
      connection-timeout: 5000         # fail fast if pool exhausted
      
# ✅ Monitor via Actuator /metrics
hikaricp.connections.active     # currently borrowed
hikaricp.connections.idle       # available
hikaricp.connections.pending    # waiting threads
hikaricp.connections.timeout    # timed out requests
```

**Common fixes:**
| Cause | Fix |
|-------|-----|
| External API call inside @Transactional | Move API call outside transaction scope |
| Slow queries | Add indexes, EXPLAIN ANALYZE, batch operations |
| Connection leak | Use try-with-resources, Spring manages when using JPA/Spring Data |
| Undersized pool | HikariCP formula: connections = (cores × 2) + spindle_count |
| N+1 queries | JOIN FETCH, @BatchSize, EntityGraph |

### 🗣️ Answering Approach
"The most common cause I've seen is long-running transactions — @Transactional on methods that call external APIs. The DB connection is held for the entire transaction duration, including API wait time. The fix is to move external calls outside the transaction scope. Second is connection leaks — code that opens connections manually without proper close. I enable HikariCP's leak-detection-threshold to catch these at 30 seconds. Third is undersized pool — default 10 connections is rarely enough for production. I tune using the formula: connections = (cores × 2) + spindle_count, and monitor via Actuator metrics."

### ⚡ Remember
- **Never call external APIs inside @Transactional** — releases the DB connection
- HikariCP pool formula: `(cores × 2) + spindle_count`
- Enable leak detection: `leak-detection-threshold: 30000`
- Cross-ref: [HikariCP sizing → spring/07-project-infrastructure-decisions.md](../spring/07-project-infrastructure-decisions.md)

---

<a id="q8"></a>
## Q8. API works locally but is slow in production — what might be different?

### 📝 One-Liner
Check: data volume, connection pool size, network latency to DB/services, DNS resolution, JVM memory/GC, missing indexes on production DB, external API response times.

### 🔑 Quick Answer
**Common production-vs-local differences:**
1. **Data volume** — local: 100 rows; production: 10M rows → query that scans all rows
2. **Network latency** — local: DB on localhost (0ms); production: DB in different AZ (5-20ms per query)
3. **Connection pooling** — local: plenty of connections; production: shared pool under contention
4. **Missing indexes** — production DB schema may differ from local Flyway/Liquibase state
5. **JVM settings** — local: default heap; production: may be undersized or GC-constrained
6. **External services** — local: mocks; production: real services with latency
7. **DNS resolution** — service discovery overhead in K8s/cloud

### 📖 How It Works (Detailed Explanation)

**Debugging checklist:**
```
□ Data volume → SELECT COUNT(*) on affected tables
□ Query plan  → EXPLAIN ANALYZE the slow query (production vs local)
□ Network     → ping DB from pod, check inter-service latency
□ Pool        → Actuator /metrics/hikaricp.connections.* 
□ GC pressure → -Xlog:gc, check GC pause duration
□ External    → Trace all outgoing calls with Sleuth/Zipkin
□ DNS         → nslookup timing for service names
□ Env config  → diff application-prod.yml vs application-local.yml
```

### 🗣️ Answering Approach
"The first thing I check is data volume. Locally we test with 100 records; production has millions. A query that works fine locally may be doing full table scans in production. I run EXPLAIN ANALYZE on the slow query in production. Second is network — locally the DB is on the same machine, but production may have 5-20ms round-trip per query. With N+1 issues, that multiplies fast. Third, I check external service response times — we use mocks locally but real services in production. I use distributed tracing with Zipkin or Sleuth to pinpoint exactly which call is slow."

---

<a id="q9"></a>
## Q9. How can we optimize database queries or transactions to improve performance?

### 📝 One-Liner
Index frequently queried columns, eliminate N+1, use batch operations, optimize SELECT (only needed columns), use read replicas, cache hot data, and keep transactions short.

### 🔑 Quick Answer
**10 optimization strategies (in priority order):**
1. **Add indexes** — on WHERE, JOIN, ORDER BY columns (EXPLAIN ANALYZE first)
2. **Fix N+1 queries** — JOIN FETCH, @EntityGraph, @BatchSize
3. **Batch operations** — `saveAll()` instead of save-in-loop; `spring.jpa.properties.hibernate.jdbc.batch_size=50`
4. **SELECT only needed columns** — Projections/DTOs instead of full entity fetch
5. **Pagination** — never `findAll()` on large tables; use `Pageable`
6. **Read replicas** — route reads to replica, writes to primary
7. **Caching** — Redis/Caffeine for frequently accessed, rarely changed data
8. **Connection pool tuning** — HikariCP sizing, statement caching
9. **Short transactions** — minimize time inside @Transactional
10. **Query optimization** — avoid `SELECT *`, use EXISTS instead of COUNT, avoid OR (use UNION)

### 📖 How It Works (Detailed Explanation)

```java
// ❌ N+1 Problem — 1 query for orders + N queries for items
List<Order> orders = orderRepository.findAll(); // 1 query
orders.forEach(o -> o.getItems().size()); // N queries (lazy load each)

// ✅ Fix: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.status = :status")
List<Order> findOrdersWithItems(@Param("status") String status);

// ❌ Save in loop
items.forEach(item -> itemRepository.save(item)); // N individual INSERTs

// ✅ Batch save
itemRepository.saveAll(items); // 1 batch INSERT (with batch_size config)

// ✅ DTO Projection (only fetch needed columns)
public interface OrderSummary {
    Long getId();
    String getStatus();
    BigDecimal getTotal();
}
List<OrderSummary> findByStatus(String status); // SELECT id, status, total only
```

### 🗣️ Answering Approach
"I start with EXPLAIN ANALYZE to identify the bottleneck — is it a missing index, full table scan, or sequential reads? The most common wins are: fixing N+1 queries with JOIN FETCH, adding indexes on WHERE and JOIN columns, and using batch operations for bulk writes. For read-heavy systems, I add a caching layer with Redis for hot data. I also keep transactions as short as possible — long transactions hold DB connections and locks. And I always paginate — findAll() on a million-row table is never acceptable."

### ⚡ Remember
- **EXPLAIN ANALYZE first** — don't guess, measure
- N+1 is the #1 JPA performance killer
- Batch size: `hibernate.jdbc.batch_size=50` in properties
- Cross-ref: [N+1 problem → database/01-jpa-sql-transactions.md](../../database/01-jpa-sql-transactions.md)

---

<a id="q10"></a>
## Q10. Request goes through API Gateway → Controller → Service → Database and is taking too long — how to optimize?

### 📝 One-Liner
Trace each layer with distributed tracing, identify the slowest span, then apply targeted optimization — don't shotgun optimize.

### 🔑 Quick Answer
**Layer-by-layer optimization:**

| Layer | Common Issues | Optimization |
|-------|--------------|--------------|
| **API Gateway** | Rate limiting, auth overhead, routing latency | Cache auth tokens, optimize route matching |
| **Controller** | Large request/response serialization | Compress responses (gzip), paginate |
| **Service** | Sequential external calls, heavy processing | Parallel calls (CompletableFuture), async processing (Kafka) |
| **Database** | Slow queries, N+1, missing indexes, lock contention | Indexes, JOIN FETCH, batch, read replicas, caching |

**Tracing approach:**
```
Use Distributed Tracing (Zipkin/Jaeger/Sleuth):
  Total: 2500ms
  ├── Gateway auth: 50ms
  ├── Controller deserialization: 20ms
  ├── Service logic: 200ms
  ├── External API call: 1500ms  ← BOTTLENECK!
  └── DB query: 730ms            ← SECONDARY
```

### 🗣️ Answering Approach
"I never optimize blindly. First, I set up distributed tracing with Spring Cloud Sleuth and Zipkin to see exact time per layer. Once I identify the bottleneck — usually it's either a slow DB query or a slow external service call — I apply targeted fixes. For DB: EXPLAIN ANALYZE the query, add indexes, fix N+1. For external calls: make them async or parallel if independent, add caching for repeated calls, set proper timeouts with circuit breakers. For the service layer: move non-critical processing to async queues with Kafka."

---

<a id="q11"></a>
## Q11. What annotations have you used in your Spring Boot projects? Give examples.

### 📝 One-Liner
Interview expects practical usage context, not just listing — group by layer and explain why you chose each.

### 🔑 Quick Answer

| Layer | Annotations | When I Use |
|-------|------------|------------|
| **Configuration** | `@SpringBootApplication`, `@Configuration`, `@Bean`, `@Value`, `@ConfigurationProperties` | App setup, external config binding |
| **Web** | `@RestController`, `@GetMapping`, `@PostMapping`, `@RequestBody`, `@PathVariable`, `@RequestParam` | REST endpoint definition |
| **Service** | `@Service`, `@Transactional`, `@Async`, `@Cacheable` | Business logic, transactions, caching |
| **Data** | `@Entity`, `@Id`, `@OneToMany`, `@ManyToOne`, `@Repository`, `@Query` | JPA entity mapping, custom queries |
| **Validation** | `@Valid`, `@NotNull`, `@Size`, `@Email` | Input validation at controller boundary |
| **Exception** | `@ControllerAdvice`, `@ExceptionHandler`, `@ResponseStatus` | Global error handling |
| **Security** | `@PreAuthorize`, `@Secured`, `@EnableWebSecurity` | Endpoint authorization |
| **Testing** | `@SpringBootTest`, `@MockBean`, `@DataJpaTest`, `@WebMvcTest` | Integration & slice tests |

### 🗣️ Answering Approach
"I'll walk through by layer. At the configuration level, I use @Configuration with @Bean for custom beans and @ConfigurationProperties for type-safe external config. For web, @RestController with @GetMapping/@PostMapping — I prefer the HTTP-method-specific annotations over @RequestMapping for readability. In the service layer, @Transactional for database operations and @Cacheable with Redis for frequently accessed data. For data, @Entity with @ManyToOne relationships, and @Query for complex custom queries. For cross-cutting concerns, @ControllerAdvice for global exception handling and @Aspect for logging. In tests, @WebMvcTest for controller-only tests and @DataJpaTest for repository tests."

---

<a id="q12"></a>
## Q12. Design an API with proper layering (Controller, Service, Repository) and implement Global Exception Handling.

### 📝 One-Liner
Three-layer architecture with DTOs at boundaries + `@ControllerAdvice` for centralized error responses with consistent error format.

### 🔑 Quick Answer

```java
// ✅ Controller Layer
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {
    private final EmployeeService employeeService;
    
    @PostMapping
    public ResponseEntity<EmployeeDTO> create(@Valid @RequestBody CreateEmployeeRequest request) {
        return ResponseEntity.status(HttpStatus.CREATED)
            .body(employeeService.createEmployee(request));
    }
    
    @GetMapping("/{id}")
    public ResponseEntity<EmployeeDTO> getById(@PathVariable Long id) {
        return ResponseEntity.ok(employeeService.getEmployee(id));
    }
}

// ✅ Service Layer
@Service
@Transactional(readOnly = true)
public class EmployeeService {
    private final EmployeeRepository repository;
    
    @Transactional
    public EmployeeDTO createEmployee(CreateEmployeeRequest request) {
        if (repository.existsByEmail(request.email())) {
            throw new DuplicateResourceException("Employee with email already exists");
        }
        Employee saved = repository.save(request.toEntity());
        return EmployeeDTO.from(saved);
    }
    
    public EmployeeDTO getEmployee(Long id) {
        return repository.findById(id)
            .map(EmployeeDTO::from)
            .orElseThrow(() -> new ResourceNotFoundException("Employee", id));
    }
}

// ✅ Repository Layer
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    boolean existsByEmail(String email);
}

// ✅ Global Exception Handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse("NOT_FOUND", ex.getMessage(), LocalDateTime.now()));
    }
    
    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(DuplicateResourceException ex) {
        return ResponseEntity.status(HttpStatus.CONFLICT)
            .body(new ErrorResponse("CONFLICT", ex.getMessage(), LocalDateTime.now()));
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest()
            .body(new ErrorResponse("VALIDATION_ERROR", message, LocalDateTime.now()));
    }
    
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR)
            .body(new ErrorResponse("INTERNAL_ERROR", "Something went wrong", LocalDateTime.now()));
    }
}

// ✅ Consistent Error Response
public record ErrorResponse(String code, String message, LocalDateTime timestamp) {}
```

### ⚡ Remember
- **Controller**: request validation, HTTP concerns, NO business logic
- **Service**: business logic, transactions, orchestration
- **Repository**: data access only
- `@RestControllerAdvice` = centralized, consistent error handling
- Custom exceptions per domain (not generic RuntimeException)
