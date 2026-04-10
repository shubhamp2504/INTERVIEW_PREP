# 🏢 TCS & Capgemini — Java Backend Developer Interview (Q1–Q21)

> **Source**: Recent interview experience for Java Backend Developer role at TCS and Capgemini  
> **Coverage**: Concurrency (locks, volatile, atomic), Maven lifecycle, Spring Security in microservices, distributed transactions, interface methods, auto-configuration, bean lifecycle, Java 17 features (records, var), SOLID, CQRS, Kafka idempotency, load balancing, SQL, DSA  
> **Level**: 3+ YOE Java Backend Developer  
> **Key**: Covers both deep Java internals AND practical microservice architecture

---

<a id="q1"></a>
## Q1. Difference between synchronized keyword and ReentrantLock. What is the trade-off?

### 📝 One-Liner
`synchronized` is simpler (implicit lock/unlock, no manual management), while `ReentrantLock` offers explicit control — tryLock, fairness, interruptibility, multiple conditions — at the cost of more code and risk of forgetting `unlock()`.

### 🔑 Quick Answer
**`synchronized`**: Built-in keyword, auto-acquires and releases lock, no deadlock from forgetting release, supports only one implicit condition (wait/notify). **`ReentrantLock`**: Explicit lock/unlock (must use try-finally), supports `tryLock(timeout)`, fairness policy (`new ReentrantLock(true)`), interruptible lock acquisition, multiple `Condition` objects. **Trade-off**: `synchronized` = simpler, less error-prone; `ReentrantLock` = more powerful but more dangerous (forgetting unlock = deadlock). *(synchronized = automatic, simple. ReentrantLock = manual control, powerful but risky — unlock bhool gaye toh deadlock pakka)*

### 📖 How It Works (Detailed Explanation)

| Feature | `synchronized` | `ReentrantLock` |
|---------|---------------|-----------------|
| Lock/Unlock | Automatic (block scope) | Manual (`lock()` / `unlock()`) |
| TryLock with timeout | ❌ Not possible | ✅ `tryLock(5, TimeUnit.SECONDS)` |
| Fairness | No guarantee (OS decides) | ✅ `new ReentrantLock(true)` — FIFO |
| Interruptible | ❌ Thread waits indefinitely | ✅ `lockInterruptibly()` |
| Multiple conditions | ❌ One implicit (wait/notify) | ✅ `lock.newCondition()` — multiple |
| Performance | Optimized in modern JVMs (biased/lightweight locking) | Slightly slower (explicit acquire) |
| Deadlock risk | Low (auto-release) | High (forget `unlock()` = deadlock) |
| Read-Write split | ❌ | ✅ `ReadWriteLock` for concurrent reads |

### 🗣️ Answering Approach
"synchronized is simpler — the JVM handles lock acquisition and release automatically, so there's no risk of forgetting to unlock. ReentrantLock is more powerful — it supports tryLock with timeout (useful to avoid deadlocks), fairness ordering, interruptible lock acquisition, and multiple Condition objects. The trade-off is complexity vs control. I use synchronized for simple mutual exclusion and ReentrantLock when I need tryLock, timeouts, or read-write separation with ReadWriteLock. In Spring Boot, most synchronization is at a higher level — @Transactional isolation, ConcurrentHashMap, or distributed locks with Redis."

### 💻 Code
```java
// ✅ synchronized — simple, auto-release
public synchronized void transfer(Account from, Account to, double amount) {
    from.debit(amount);
    to.credit(amount);
} // Lock released automatically

// ✅ ReentrantLock — explicit control with tryLock
private final ReentrantLock lock = new ReentrantLock(true); // fair

public boolean transferWithTimeout(Account from, Account to, double amount) 
        throws InterruptedException {
    if (lock.tryLock(5, TimeUnit.SECONDS)) {  // try for 5s, avoid deadlock
        try {
            from.debit(amount);
            to.credit(amount);
            return true;
        } finally {
            lock.unlock(); // ⚠️ MUST be in finally block
        }
    }
    return false; // couldn't acquire lock in time
}
```

### ⚠️ Pitfalls / Gotchas
- **ReentrantLock without finally**: Forgetting `unlock()` in finally = permanent deadlock
- **Fair locks are slower**: FIFO ordering has overhead — use only when starvation is a problem
- **Virtual threads + synchronized** = **pinning** — use ReentrantLock with virtual threads (Java 21)

### ⚡ Remember
- `synchronized` = simple + safe; `ReentrantLock` = powerful + dangerous
- Always `unlock()` in `finally` block
- `tryLock(timeout)` = avoid deadlocks; fairness = prevent starvation
- Virtual threads: prefer `ReentrantLock` over `synchronized`

---

<a id="q2"></a>
## Q2. HikariCP — What is it?

### 📝 One-Liner
HikariCP is the fastest JDBC connection pool for Java — Spring Boot's default since 2.0 — managing a pool of reusable DB connections to avoid the overhead of creating new connections per request.

### 🔑 Quick Answer
**HikariCP** = High-performance JDBC connection pool. It pre-creates a pool of DB connections and reuses them instead of creating/closing connections per request (which takes ~250ms each). Spring Boot auto-configures it. Key settings: `maximumPoolSize` (max connections, default 10), `minimumIdle` (min idle connections), `connectionTimeout` (wait time for connection, default 30s), `idleTimeout` (close idle connection after, default 10min). Under load, if pool exhausts → threads wait → `connectionTimeout` → `SQLTransientConnectionException`. *(HikariCP DB connection ka pool manage karta hai — har request pe naya connection banana expensive hai, pool se reuse karna fast hai)*

### 📖 How It Works (Detailed Explanation)

```
Without Pool:                      With HikariCP:
Request → Create Connection        Request → Borrow from Pool
        → Execute Query                    → Execute Query
        → Close Connection                 → Return to Pool (reuse!)
        ≈ 250ms overhead                   ≈ 0ms overhead
```

**Key configuration (application.yml):**
```yaml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20        # max connections (default: 10)
      minimum-idle: 5              # min idle connections
      connection-timeout: 30000    # wait 30s for connection (ms)
      idle-timeout: 600000         # close idle after 10min
      max-lifetime: 1800000        # force close after 30min
      pool-name: MyAppPool
      leak-detection-threshold: 60000  # warn if connection not returned in 60s
```

**Pool sizing formula:**
```
connections = ((CPU cores * 2) + effective_disk_spindles)
Example: 4 cores → (4 * 2) + 1 = 9-10 connections
```

### 🗣️ Answering Approach
"HikariCP is the default JDBC connection pool in Spring Boot — it's the fastest pool available for Java. It pre-creates database connections and reuses them, avoiding the expensive overhead of establishing new TCP connections and DB handshakes per request. Key tuning parameters are maximumPoolSize, connectionTimeout, and leak-detection-threshold. The typical formula is 2× CPU cores + 1 for pool size. In production, I monitor active, idle, and pending connections via Micrometer metrics. Pool exhaustion — where all connections are in use — is one of the most common production issues I debug."

### ⚠️ Pitfalls / Gotchas
- **Pool exhaustion**: Long-running queries or connection leaks drain the pool
- **Too large pool**: More connections ≠ faster — DB has connection limits too
- **Leak detection**: Enable `leak-detection-threshold` to catch unreturned connections
- **@Transactional scope**: Large transactional methods hold connections longer

### ⚡ Remember
- Spring Boot default since 2.0 — no extra dependency needed
- Pool size formula: `(CPU cores × 2) + 1`
- Monitor: active, idle, pending, blocked, timeout metrics
- `leak-detection-threshold` = lifesaver for finding connection leaks

---

<a id="q3"></a>
## Q3. What is the volatile keyword?

### 📝 One-Liner
`volatile` guarantees visibility — a write to a volatile variable by one thread is immediately visible to all other threads — but it does NOT guarantee atomicity (no compound operations like `count++`).

### 🔑 Quick Answer
Without `volatile`, each thread may cache variables in CPU registers/L1 cache and never see updates from other threads. `volatile` forces read/write directly from main memory — ensuring **visibility**. But `volatile` does NOT make `count++` atomic (read → increment → write is 3 steps). For atomic compound operations, use `AtomicInteger`. Use `volatile` for: flags (`volatile boolean running`), double-checked locking singleton, publish-subscribe patterns. *(volatile = "hamesha main memory se padho/likho" — lekin count++ jaise operations ke liye kaam nahi karega, atomic chahiye)*

### 📖 How It Works (Detailed Explanation)

```
Without volatile:                 With volatile:
Thread-1: running = true          Thread-1: running = true
  (writes to CPU cache)             (writes to MAIN MEMORY)

Thread-2: while(running)          Thread-2: while(running)
  (reads from own CPU cache)         (reads from MAIN MEMORY)
  → may never see false!             → always sees latest value ✅
```

| Feature | `volatile` | `synchronized` | `AtomicInteger` |
|---------|-----------|----------------|-----------------|
| Visibility | ✅ | ✅ | ✅ |
| Atomicity | ❌ | ✅ | ✅ (CAS-based) |
| Blocking | ❌ (lock-free) | ✅ (acquires lock) | ❌ (lock-free) |
| Use case | Flags, state publish | Critical sections | Counters, accumulators |

### 🗣️ Answering Approach
"volatile ensures visibility across threads — any write is flushed to main memory and subsequent reads always fetch from main memory. It prevents threads from seeing stale cached values. However, volatile doesn't provide atomicity — operations like count++ require read, modify, write which can still race. For counters, I use AtomicInteger which uses CAS (Compare-And-Swap) for lock-free atomic operations. I use volatile for boolean flags like shutdown signals and for the double-checked locking singleton pattern."

### 💻 Code
```java
// ✅ volatile for shutdown flag
class Worker implements Runnable {
    private volatile boolean running = true; // visible across threads
    
    public void run() {
        while (running) { // always reads from main memory
            doWork();
        }
    }
    
    public void stop() {
        running = false; // immediately visible to worker thread
    }
}

// ❌ volatile does NOT fix this (not atomic)
private volatile int count = 0;
count++; // read → increment → write = 3 ops, still races!

// ✅ Use AtomicInteger instead
private final AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // atomic CAS operation
```

### ⚡ Remember
- `volatile` = visibility (main memory read/write); NOT atomicity
- Use for: flags, state published across threads, double-checked locking
- For counters: `AtomicInteger` / `AtomicLong` (CAS-based, lock-free)

---

<a id="q4"></a>
## Q4. Difference between volatile keyword and Atomic class (AtomicInteger, AtomicLong)?

### 📝 One-Liner
`volatile` guarantees visibility only; `Atomic` classes guarantee both visibility AND atomicity using lock-free CAS (Compare-And-Swap) operations — crucial for counters and accumulators.

### 🔑 Quick Answer
**`volatile`**: Ensures read/write go to main memory (visibility). But `count++` on volatile is still NOT atomic (3 steps: read, increment, write — another thread can interfere between steps). **`AtomicInteger`**: Uses CAS (Compare-And-Swap) hardware instruction — reads current value, computes new value, atomically writes ONLY if current value hasn't changed. If another thread changed it, CAS retries (spin). This makes `incrementAndGet()` both visible AND atomic without locks. *(volatile sirf dikhata hai — "value change ho gayi." AtomicInteger dikhata bhi hai aur safely change bhi karta hai — bina lock ke)*

### 📖 How It Works (Detailed Explanation)

```
volatile count = 5:
Thread-1: READ 5 → ADD 1 → WRITE 6
Thread-2: READ 5 → ADD 1 → WRITE 6  ← RACE! Expected 7, got 6

AtomicInteger count = 5:
Thread-1: CAS(expected=5, new=6) → SUCCESS → count = 6
Thread-2: CAS(expected=5, new=6) → FAIL (count is now 6, not 5) → RETRY
Thread-2: CAS(expected=6, new=7) → SUCCESS → count = 7  ✅
```

| Operation | `volatile` | `AtomicInteger` |
|-----------|-----------|-----------------|
| Simple read/write | ✅ Safe | ✅ Safe |
| `count++` | ❌ Race condition | ✅ `incrementAndGet()` |
| `count = count + 5` | ❌ Race condition | ✅ `addAndGet(5)` |
| Compare-and-set | ❌ Not available | ✅ `compareAndSet(old, new)` |
| Performance | Faster (no CAS overhead) | Slightly slower (CAS loop) |

### 🗣️ Answering Approach
"volatile provides visibility — writes go to main memory, reads come from main memory. But it doesn't help with compound operations like count++. AtomicInteger solves this using CAS — Compare-And-Swap — a hardware-level atomic instruction. When a thread wants to increment, it reads the current value, computes the new value, then atomically writes it only if the current value hasn't changed. If another thread modified it in between, CAS detects the mismatch and retries. This is lock-free and generally faster than synchronized for simple counters."

### ⚡ Remember
- `volatile` = visibility only; `Atomic*` = visibility + atomicity
- CAS = Compare-And-Swap (hardware instruction, lock-free)
- Use `volatile` for simple flags; `Atomic*` for counters and accumulators
- Atomic classes: `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference`

---

<a id="q5"></a>
## Q5. What is the Maven lifecycle? (Detailed: maven package, maven install, maven deploy)

### 📝 One-Liner
Maven has 3 lifecycles (default, clean, site). The default lifecycle phases run in order: `validate → compile → test → package → verify → install → deploy` — each phase executes all preceding phases.

### 🔑 Quick Answer
Running `mvn package` automatically runs: validate → compile → test-compile → test → package. Running `mvn install` adds → verify → install (copies to local `.m2` repo). Running `mvn deploy` adds → deploy (uploads to remote repo like Nexus/Artifactory). Each command runs ALL preceding phases in order. *(Maven lifecycle sequential hai — `mvn install` bologe toh compile, test, package sab pehle chalega, phir install hoga local .m2 mein)*

### 📖 How It Works (Detailed Explanation)

**Default Lifecycle (7 key phases):**

| Phase | What Happens | Command |
|-------|-------------|---------|
| **validate** | Validate project structure, POM correctness | Automatic |
| **compile** | Compile src/main/java → target/classes | `mvn compile` |
| **test** | Run unit tests (src/test/java) | `mvn test` |
| **package** | Create JAR/WAR in target/ | `mvn package` |
| **verify** | Run integration tests, quality checks | `mvn verify` |
| **install** | Copy artifact to local ~/.m2/repository | `mvn install` |
| **deploy** | Upload artifact to remote repo (Nexus/Artifactory) | `mvn deploy` |

```
mvn package  → validate → compile → test → PACKAGE
mvn install  → validate → compile → test → package → verify → INSTALL
mvn deploy   → validate → compile → test → package → verify → install → DEPLOY
```

**Other lifecycles:**
- **clean**: `mvn clean` → deletes target/ directory
- **site**: `mvn site` → generates project documentation

**`<distributionManagement>` tag in pom.xml:**
```xml
<distributionManagement>
    <repository>
        <id>releases</id>
        <url>https://nexus.company.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>snapshots</id>
        <url>https://nexus.company.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```
This configures WHERE `mvn deploy` uploads the artifact — required for CI/CD pipelines.

### 🗣️ Answering Approach
"Maven's default lifecycle has ordered phases: validate, compile, test, package, verify, install, deploy. When I run mvn package, it executes validate through package — compiling code, running tests, and creating the JAR. mvn install goes further — builds the package and copies it to my local .m2 repository so other projects on my machine can use it as a dependency. mvn deploy does everything plus uploads the artifact to a remote repository like Nexus, configured via the distributionManagement tag in pom.xml. In our CI/CD pipeline, we run mvn deploy on the main branch to publish artifacts to Nexus for other services to consume."

### ⚡ Remember
- `mvn package` = compile + test + JAR/WAR
- `mvn install` = package + copy to local `.m2`
- `mvn deploy` = install + upload to remote (Nexus/Artifactory)
- `distributionManagement` = remote repo URL for `mvn deploy`
- Each phase runs ALL preceding phases

---

<a id="q6"></a>
## Q6. How did you implement Spring Security in distributed microservices?

### 📝 One-Liner
API Gateway authenticates JWT tokens, each microservice validates the token independently (stateless), and inter-service calls use service-to-service tokens or propagated user tokens.

### 🔑 Quick Answer
**Architecture**: Client → API Gateway (auth) → Microservices (token validation). **Implementation**: (1) **API Gateway** — validates JWT, extracts claims, routes request with token in header. (2) **Each microservice** — Spring Security filter validates JWT signature (public key), extracts roles/permissions, applies `@PreAuthorize`. (3) **Inter-service** — WebClient propagates user JWT or uses service-account tokens (client_credentials flow). **Token issuer**: Keycloak/Auth0/custom auth-service issues JWT with claims (userId, roles, permissions). *(Gateway pe authenticate karo, microservice pe authorize karo — har service independently JWT validate karti hai, kisi pe trust nahi)*

### 📖 How It Works (Detailed Explanation)

```
Client → [JWT Token] → API Gateway
                           │
                    ┌──────┼──────┐
                    ▼      ▼      ▼
              Order API  User API  Payment API
              (validates  (validates (validates
               JWT)       JWT)      JWT)
```

**Key components:**
1. **Auth Service / Keycloak**: Issues JWT with claims
2. **API Gateway**: Rate limiting + initial token validation
3. **JwtAuthFilter**: Each microservice has a filter that validates token signature + expiry
4. **@PreAuthorize**: Method-level authorization (`@PreAuthorize("hasRole('ADMIN')")`)
5. **Inter-service**: Propagate Authorization header or use service tokens

### 🗣️ Answering Approach
"In our microservices architecture, I implemented security using JWT with an API Gateway pattern. The auth service (Keycloak) issues JWT tokens containing user ID, roles, and permissions. The API Gateway does initial authentication — validates the token signature and expiry. Each downstream microservice independently validates the JWT using the public key — this is stateless, no session needed. For authorization, I use @PreAuthorize annotations at the controller/service level. For inter-service communication, we propagate the user's JWT token via WebClient interceptors. For machine-to-machine calls, we use OAuth2 client_credentials flow with service-specific tokens."

### 💻 Code
```java
// JWT validation filter in each microservice
@Component
public class JwtAuthFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest req, HttpServletResponse res,
            FilterChain chain) throws ServletException, IOException {
        String token = extractToken(req.getHeader("Authorization"));
        if (token != null && jwtUtil.validateToken(token)) {
            Claims claims = jwtUtil.extractClaims(token);
            var auth = new UsernamePasswordAuthenticationToken(
                claims.getSubject(), null, extractAuthorities(claims));
            SecurityContextHolder.getContext().setAuthentication(auth);
        }
        chain.doFilter(req, res);
    }
}

// Inter-service call — propagate JWT
@Bean
public WebClient.Builder webClientBuilder() {
    return WebClient.builder()
        .filter((req, next) -> {
            String token = SecurityContextHolder.getContext()
                .getAuthentication().getCredentials().toString();
            return next.exchange(ClientRequest.from(req)
                .header("Authorization", "Bearer " + token).build());
        });
}
```

### ⚡ Remember
- Gateway = authenticate; Microservice = authorize (stateless JWT validation)
- Each service validates JWT independently — never trust other services blindly
- Inter-service: propagate user JWT or use client_credentials tokens
- Keycloak/Auth0 = token issuer; Spring Security = token validator

---

<a id="q7"></a>
## Q7. How to maintain transactions in a distributed database?

### 📝 One-Liner
Use the Saga pattern (choreography or orchestration) for microservices, or 2PC (Two-Phase Commit) for strong consistency — Saga is preferred for availability, 2PC for strict ACID.

### 🔑 Quick Answer
In microservices, distributed transactions can't use traditional ACID (each service has its own DB). **Saga Pattern** (preferred): Break transaction into local steps, each service commits its own DB, compensating actions undo on failure. Two types: **Choreography** (event-driven, each service reacts to events) and **Orchestration** (central orchestrator coordinates steps). **2PC** (Two-Phase Commit): Coordinator asks all services to prepare, then commit/rollback — guarantees consistency but blocks all services during vote phase (poor availability). *(Microservices mein @Transactional ek service se doosri service tak kaam nahi karta — Saga pattern use karo: har step ka undo bhi define karo)*

### 📖 How It Works (Detailed Explanation)

**Saga Choreography (event-driven):**
```
Order Service → publishes OrderCreated event
    → Payment Service → deducts money → publishes PaymentCompleted
        → Inventory Service → reserves stock → publishes StockReserved
            → Shipping Service → creates shipment

FAILURE at Payment? → publishes PaymentFailed
    → Order Service → compensates: cancel order
```

**Saga Orchestration (central coordinator):**
```
Saga Orchestrator:
  Step 1: Call Order Service → create order
  Step 2: Call Payment Service → deduct money
  Step 3: Call Inventory Service → reserve stock
  FAILURE at Step 3?
    → Compensate Step 2: refund payment
    → Compensate Step 1: cancel order
```

| Approach | Consistency | Availability | Complexity |
|----------|-------------|-------------|------------|
| **2PC** | Strong (ACID) | Low (blocking) | Medium |
| **Saga Choreography** | Eventual | High | High (event tracking) |
| **Saga Orchestration** | Eventual | High | Medium (centralized logic) |

### 🗣️ Answering Approach
"For distributed transactions across microservices, I use the Saga pattern because 2PC is blocking and doesn't scale well. In our order system, I implemented Saga Orchestration: the Order Orchestrator coordinates steps — create order, deduct payment, reserve inventory. Each step commits to its own database. If payment fails, the orchestrator triggers compensating actions — cancel the order. We use Kafka for reliable event delivery between services. For idempotency, each service checks if the step was already processed before executing. The key design principle is: every action must have a compensating action."

### ⚡ Remember
- Saga = eventual consistency + compensating actions; 2PC = strong consistency + blocking
- Choreography = decentralized (events); Orchestration = centralized (coordinator)
- Each step: local commit + compensating action defined
- Kafka/RabbitMQ for reliable event delivery between saga steps

---

<a id="q8"></a>
## Q8. In pom.xml, what is the `<distributionManagement>` tag? Why do we use it?

### 📝 One-Liner
`<distributionManagement>` configures the remote repository URL where `mvn deploy` uploads your built artifact (JAR/WAR) — typically Nexus or Artifactory in enterprise CI/CD.

### 🔑 Quick Answer
When you run `mvn deploy`, Maven needs to know WHERE to upload the artifact. `<distributionManagement>` specifies: **release repo** (for stable versions like `1.0.0`) and **snapshot repo** (for development versions like `1.0.0-SNAPSHOT`). Without it, `mvn deploy` fails with "no distribution management information." The credentials for these repos are stored in `~/.m2/settings.xml` (matched by repository `<id>`). *(distributionManagement = Maven ko batao ki `deploy` karne pe artifact kahan upload karna hai — Nexus/Artifactory ka URL)*

### 📖 How It Works (Detailed Explanation)

```xml
<!-- pom.xml -->
<distributionManagement>
    <repository>
        <id>company-releases</id>
        <name>Release Repository</name>
        <url>https://nexus.company.com/repository/maven-releases/</url>
    </repository>
    <snapshotRepository>
        <id>company-snapshots</id>
        <name>Snapshot Repository</name>
        <url>https://nexus.company.com/repository/maven-snapshots/</url>
    </snapshotRepository>
</distributionManagement>
```

```xml
<!-- ~/.m2/settings.xml (credentials) -->
<servers>
    <server>
        <id>company-releases</id>  <!-- matches <id> in pom.xml -->
        <username>${env.NEXUS_USER}</username>
        <password>${env.NEXUS_PASS}</password>
    </server>
</servers>
```

### 🗣️ Answering Approach
"distributionManagement in pom.xml configures the target repository for mvn deploy. It has two sections: repository for release versions and snapshotRepository for snapshot builds. When our CI/CD pipeline runs mvn deploy, it uploads the JAR to Nexus at the configured URL. The authentication credentials are stored in ~/.m2/settings.xml, matched by the repository ID. This enables other teams to pull our library as a Maven dependency without needing our source code."

### ⚡ Remember
- `<repository>` = release versions; `<snapshotRepository>` = SNAPSHOT versions
- Credentials in `~/.m2/settings.xml`, matched by `<id>`
- Without it, `mvn deploy` fails
- Used in CI/CD to publish artifacts to Nexus/Artifactory

---

<a id="q9"></a>
## Q9. Is it possible to write private methods in an interface?

### 📝 One-Liner
Yes, since Java 9 — interfaces can have `private` methods to share common logic between `default` and `static` methods without exposing it to implementing classes.

### 🔑 Quick Answer
Java 9 introduced **private methods in interfaces** — both `private` and `private static`. They're used to avoid code duplication between `default` methods. The private method is NOT visible to implementing classes or subinterfaces. Before Java 9, you had to duplicate code across default methods or extract to a utility class. *(Java 9 se interface mein private method likh sakte ho — default methods ke beech common logic share karne ke liye, implementing class ko dikhta nahi)*

### 📖 How It Works (Detailed Explanation)

```java
public interface Logger {
    // public abstract (traditional)
    void log(String message);
    
    // default method (Java 8)
    default void logInfo(String msg)  { logFormatted("INFO", msg); }
    default void logError(String msg) { logFormatted("ERROR", msg); }
    default void logWarn(String msg)  { logFormatted("WARN", msg); }
    
    // ✅ private method (Java 9) — shared logic, NOT visible to implementors
    private void logFormatted(String level, String msg) {
        System.out.println("[" + level + "] " + java.time.LocalDateTime.now() + " - " + msg);
    }
    
    // ✅ private static method (Java 9)
    private static String sanitize(String input) {
        return input.replaceAll("[^a-zA-Z0-9 ]", "");
    }
}
```

### 🗣️ Answering Approach
"Yes, since Java 9. Private methods in interfaces serve as helper methods for default and static methods — avoiding code duplication without exposing internal logic to implementing classes. Before Java 9, if two default methods shared common logic, you had to either duplicate the code or use a separate utility class. Now you can extract the shared logic into a private method within the interface itself."

### ⚡ Remember
- Java 9+ feature — `private` and `private static` in interfaces
- Purpose: DRY between default methods
- NOT visible to implementing classes
- Interface methods through versions: abstract (Java 1) → default + static (Java 8) → private (Java 9)

---

<a id="q10"></a>
## Q10. How many types of methods can we write in an interface?

### 📝 One-Liner
Five types: **abstract** (Java 1), **default** (Java 8), **static** (Java 8), **private** (Java 9), **private static** (Java 9).

### 🔑 Quick Answer

| Type | Since | Syntax | Inherited? | Need body? |
|------|-------|--------|-----------|-----------|
| Abstract | Java 1 | `void doWork();` | Yes (must implement) | No |
| Default | Java 8 | `default void log() {...}` | Yes (can override) | Yes |
| Static | Java 8 | `static void util() {...}` | No (call via InterfaceName) | Yes |
| Private | Java 9 | `private void helper() {...}` | No (internal only) | Yes |
| Private Static | Java 9 | `private static void util() {...}` | No (internal only) | Yes |

### 🗣️ Answering Approach
"There are five types. Abstract methods — the traditional contract that implementing classes must fulfill. Default methods introduced in Java 8 — provide a body, inherited by implementors, can be overridden. Static methods in Java 8 — utility methods called via the interface name, not inherited. Private methods in Java 9 — helper methods for default methods, not visible to implementors. And private static methods in Java 9 — helper methods for static methods. This evolution made interfaces more powerful while maintaining backward compatibility."

### ⚡ Remember
- **5 types**: abstract, default, static, private, private static
- Java 8 = default + static; Java 9 = private + private static
- Default = inherited; static = not inherited; private = internal DRY

---

<a id="q11"></a>
## Q11. What are @EnableAutoConfiguration and @Qualifier?

### 📝 One-Liner
`@EnableAutoConfiguration` tells Spring Boot to auto-configure beans based on classpath dependencies. `@Qualifier` disambiguates when multiple beans of the same type exist.

### 🔑 Quick Answer
**@EnableAutoConfiguration**: Part of `@SpringBootApplication` composite annotation. Scans classpath — if `spring-data-jpa` is present, auto-configures DataSource, EntityManager, TransactionManager. If `spring-web` is present, configures DispatcherServlet, embedded Tomcat. Uses `META-INF/spring.factories` (pre-Boot 3) or `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` (Boot 3+). **@Qualifier**: When you have `@Bean NotificationService emailService()` and `@Bean NotificationService smsService()`, Spring can't auto-wire by type alone. `@Qualifier("emailService")` tells Spring which one you want. *(EnableAutoConfiguration = Spring Boot apne aap figure karta hai kya configure karna hai. Qualifier = ek type ke multiple beans hain toh batao kaunsa chahiye)*

### 📖 How It Works (Detailed Explanation)

```java
// @SpringBootApplication = @EnableAutoConfiguration + @ComponentScan + @Configuration
@SpringBootApplication
public class MyApp { ... }

// What auto-configuration does:
// Classpath has: spring-boot-starter-data-jpa
// → Auto-configures: DataSource, EntityManagerFactory, TransactionManager
// Classpath has: spring-boot-starter-web
// → Auto-configures: DispatcherServlet, Tomcat, Jackson ObjectMapper

// ✅ @Qualifier example
@Service
public class OrderService {
    @Autowired
    @Qualifier("emailNotification")  // which bean to inject
    private NotificationService notificationService;
}

@Configuration
public class NotificationConfig {
    @Bean("emailNotification")
    public NotificationService emailService() { return new EmailNotification(); }
    
    @Bean("smsNotification")
    public NotificationService smsService() { return new SmsNotification(); }
}
```

### 🗣️ Answering Approach
"@EnableAutoConfiguration is part of @SpringBootApplication. It inspects the classpath and auto-configures beans — if spring-data-jpa is present, it configures DataSource and EntityManager; if spring-web is present, it configures Tomcat and DispatcherServlet. You can exclude specific configurations with @SpringBootApplication(exclude = DataSourceAutoConfiguration.class). @Qualifier solves ambiguity when multiple beans of the same type exist. Instead of Spring guessing which NotificationService to inject, @Qualifier specifies the exact bean by name."

### ⚡ Remember
- `@EnableAutoConfiguration` = inside `@SpringBootApplication` (classpath → beans)
- Exclude auto-config: `@SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})`
- `@Qualifier` = disambiguate multiple beans of same type
- Alternative to `@Qualifier`: `@Primary` marks one bean as the default

---

<a id="q12"></a>
## Q12. What are bean lifecycles?

### 📝 One-Liner
Spring bean lifecycle: **Instantiate** → **Populate properties (DI)** → **BeanNameAware/BeanFactoryAware** → **BeanPostProcessor.before** → **@PostConstruct / InitializingBean** → **Ready to use** → **@PreDestroy / DisposableBean** → **Destroyed**.

### 🔑 Quick Answer
The full lifecycle: (1) Bean instantiated via constructor, (2) Dependencies injected (DI), (3) Aware interfaces called (BeanNameAware, ApplicationContextAware), (4) `BeanPostProcessor.postProcessBeforeInitialization()`, (5) `@PostConstruct` / `InitializingBean.afterPropertiesSet()`, (6) Bean ready for use, (7) On shutdown: `@PreDestroy` / `DisposableBean.destroy()`. *(Bean banta hai → dependencies inject hoti hain → initialization callbacks → use karo → shutdown pe cleanup)*

### 📖 How It Works (Detailed Explanation)

```
1. Instantiation       → new Bean() (constructor)
2. Property injection  → @Autowired, @Value setters
3. Aware callbacks     → setBeanName(), setApplicationContext()
4. BPP beforeInit      → BeanPostProcessor.postProcessBeforeInitialization()
5. Initialization      → @PostConstruct → InitializingBean.afterPropertiesSet()
6. BPP afterInit       → BeanPostProcessor.postProcessAfterInitialization()
   ──────── Bean is READY ────────
7. Usage               → Application uses the bean
   ──────── Shutdown ────────
8. Destruction         → @PreDestroy → DisposableBean.destroy()
```

```java
@Component
public class MyService implements InitializingBean, DisposableBean {
    @PostConstruct
    public void init() { /* runs after DI, before ready */ }
    
    @Override
    public void afterPropertiesSet() { /* InitializingBean callback */ }
    
    @PreDestroy
    public void cleanup() { /* runs before bean is destroyed */ }
    
    @Override
    public void destroy() { /* DisposableBean callback */ }
}
```

### 🗣️ Answering Approach
"The Spring bean lifecycle has clear phases. First, the bean is instantiated and dependencies are injected. Then aware interfaces are called if implemented — setBeanName, setApplicationContext. BeanPostProcessor's beforeInitialization runs, followed by @PostConstruct and InitializingBean.afterPropertiesSet for custom initialization. After BeanPostProcessor's afterInitialization, the bean is ready. On application shutdown, @PreDestroy runs followed by DisposableBean.destroy. In practice, I use @PostConstruct for initialization logic like loading caches and @PreDestroy for cleanup like closing connections."

### ⚡ Remember
- **Init order**: Constructor → DI → Aware → BPP before → `@PostConstruct` → `afterPropertiesSet()` → BPP after
- **Destroy order**: `@PreDestroy` → `DisposableBean.destroy()`
- `@PostConstruct` = most common initialization hook
- BeanPostProcessor = cross-cutting (logging, proxying, AOP)

### 🔗 Cross-references
- Spring bean lifecycle detailed → [spring/01-spring-framework-internals.md](../languages/java/spring/01-spring-framework-internals.md)

---

<a id="q13"></a>
## Q13. Java 17 features.

### 📝 One-Liner
Key Java 17 features: Sealed classes, Records, Pattern matching for instanceof, Switch expressions, Text blocks, Helpful NullPointerExceptions, Strong encapsulation of JDK internals.

### 🔑 Quick Answer

| Feature | Example | Why It Matters |
|---------|---------|---------------|
| **Sealed classes** | `sealed class Shape permits Circle, Rect` | Controlled inheritance + exhaustive switch |
| **Records** | `record User(String name, int age) {}` | Immutable DTOs with less boilerplate |
| **Pattern matching instanceof** | `if (obj instanceof String s)` (no cast) | Eliminates explicit casting |
| **Switch expressions** | `int r = switch(x) { case 1 -> 10; }` | Value-returning switch, arrow syntax |
| **Text blocks** | `"""multi-line string"""` | Clean SQL, JSON, HTML templates |
| **Helpful NPE** | `a.b.c.d` → "Cannot invoke because a.b is null" | Pinpoints exact null reference |
| **Strong JDK encapsulation** | `--illegal-access=deny` default | Can't access internal JDK APIs |

### 🗣️ Answering Approach
"Java 17 is an LTS release with several important features. Records eliminate boilerplate for immutable data classes — I use them for DTOs and API responses. Sealed classes let me control the inheritance hierarchy — only permitted classes can extend, enabling exhaustive switch patterns. Pattern matching for instanceof removes explicit casts. Text blocks clean up multi-line strings like SQL queries. Helpful NullPointerExceptions tell you exactly which reference was null in a chain call. And strong encapsulation of JDK internals means reflection-based hacks on internal APIs no longer work by default."

### 🔗 Cross-references
- Java versions features deep dive → [core/17-java-versions-features.md](../languages/java/core/17-java-versions-features.md)

### ⚡ Remember
- Java 17 = LTS (Long-Term Support) — current enterprise standard
- Records = immutable DTOs; Sealed = controlled hierarchy; Pattern matching = no casts
- Migration from Java 11: check for internal API usage (`--add-opens` may be needed)

---

<a id="q14"></a>
## Q14. What is the Record class? How to store data from a query result? How to fetch the first record?

### 📝 One-Liner
`record` (Java 16+) is an immutable data carrier with auto-generated constructor, getters, equals, hashCode, toString — perfect for DTOs. Map query results via constructor, fetch first with `stream().findFirst()` or `LIMIT 1`.

### 🔑 Quick Answer
`record Employee(int id, String name, double salary) {}` — compiler generates: all-args constructor, `id()`, `name()`, `salary()` accessors, `equals()`, `hashCode()`, `toString()`. To store query data: use JPA's constructor expression or `@SqlResultSetMapping`. To fetch first record: `findFirst()` from stream, or `LIMIT 1` in SQL, or Spring's `findTop1By...()`. *(Record = immutable data class — boilerplate likne ki zarurat nahi. Query result store karne ke liye constructor expression use karo)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Define a Record
public record EmployeeDTO(int id, String name, double salary) {}

// ✅ Store query data into Record (JPQL constructor expression)
@Query("SELECT new com.example.dto.EmployeeDTO(e.id, e.name, e.salary) " +
       "FROM Employee e WHERE e.department = :dept")
List<EmployeeDTO> findByDept(@Param("dept") String dept);

// ✅ Fetch first record
// Option 1: Stream
Optional<EmployeeDTO> first = employees.stream().findFirst();

// Option 2: Spring Data JPA derived query
Optional<EmployeeDTO> findTop1ByDepartmentOrderBySalaryDesc(String dept);

// Option 3: JPQL with LIMIT (Hibernate 6+)
@Query("SELECT e FROM Employee e ORDER BY e.salary DESC LIMIT 1")
EmployeeDTO findHighestPaid();

// ✅ Records with List data
List<EmployeeDTO> employees = resultList.stream()
    .map(row -> new EmployeeDTO(
        (int) row[0],
        (String) row[1],
        (double) row[2]))
    .toList();

EmployeeDTO firstEmp = employees.getFirst(); // Java 21 or .get(0)
```

**Record rules:**
- Implicitly `final` — cannot extend or be extended
- Fields are `private final` — truly immutable
- Can implement interfaces but cannot extend classes
- Can have custom methods, static methods, and compact constructors

### 🗣️ Answering Approach
"Records are immutable data carriers introduced in Java 16. They auto-generate the constructor, accessor methods, equals, hashCode, and toString — eliminating boilerplate for DTOs. For storing query results, I use JPQL constructor expressions or map native query results via streams. For fetching the first record, I use Stream.findFirst() in code, Spring Data's findTop1By* for JPA, or LIMIT 1 in SQL. In our project, all API response DTOs are records because they're immutable and concise."

### ⚡ Remember
- Record = final + private final fields + auto getters + equals + hashCode + toString
- JPQL: `SELECT new DTO(...)` for direct mapping
- First record: `stream().findFirst()` or `findTop1By...()` or `LIMIT 1`

---

<a id="q15"></a>
## Q15. What is the `var` keyword? Can we use it as a generic type?

### 📝 One-Liner
`var` (Java 10) = local variable type inference — the compiler infers the type from the right-hand side. It is NOT a generic type and cannot be used for fields, method parameters, or return types.

### 🔑 Quick Answer
`var list = new ArrayList<String>();` — compiler infers `ArrayList<String>`. It's syntactic sugar for local variables only — the type is still statically determined at compile time (not dynamic like JavaScript's `var`). **Cannot be used as generic type**: `var` is not a type — you can't do `List<var>`, `public var getUser()`, or set it as a field type. It works only as a local variable declaration. *(var = compiler ke liye shortcut — type likhne ki zarurat nahi but compile time pe type fixed hota hai)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Valid uses of var
var name = "Shubham";          // infers String
var list = List.of(1, 2, 3);   // infers List<Integer>
var map = new HashMap<String, List<Integer>>(); // saves typing!
var stream = list.stream().filter(x -> x > 1); // infers Stream<Integer>

// ✅ var in for-loops and try-with-resources
for (var entry : map.entrySet()) { ... }  // infers Map.Entry<String, List<Integer>>
try (var reader = new BufferedReader(new FileReader("f.txt"))) { ... }

// ❌ INVALID uses
var x;                    // no initializer → can't infer
var y = null;             // null → can't infer type
var z = {1, 2, 3};       // array initializer → can't infer
private var name = "X";  // field → not allowed
public var getName() {}  // return type → not allowed
void process(var input)  // parameter → not allowed
List<var> items;          // generic type → NOT a type!

// ❌ var is NOT a generic type parameter
// List<var> = INVALID — var is not a type, it's a declaration keyword
```

### 🗣️ Answering Approach
"var is local variable type inference introduced in Java 10. The compiler infers the type from the initializer — so var list = new ArrayList<String>() is equivalent to ArrayList<String> list = new ArrayList<>(). It's compile-time inference, not dynamic typing — the type is fixed at compilation. It cannot be used for fields, method parameters, return types, or as a generic type parameter. You can't write List<var> because var is not a type — it's a context keyword that only works in local variable declarations."

### ⚡ Remember
- `var` = local variables only (Java 10+)
- Compile-time inference (NOT dynamic typing)
- NOT a type → can't use in generics (`List<var>`), fields, params, return types
- Best for: complex generic types (saves typing), for-each loops, try-with-resources

---

<a id="q16"></a>
## Q16. SOLID design principles.

### 📝 One-Liner
**S**ingle Responsibility, **O**pen-Closed, **L**iskov Substitution, **I**nterface Segregation, **D**ependency Inversion — five principles for maintainable, extensible OOP design.

### 🔑 Quick Answer

| Principle | Meaning | Spring Boot Example |
|-----------|---------|-------------------|
| **S**ingle Responsibility | One class = one reason to change | `OrderService` handles orders, `EmailService` handles emails — not mixed |
| **O**pen-Closed | Open for extension, closed for modification | Strategy pattern: add `UPIPayment` without modifying `PaymentProcessor` |
| **L**iskov Substitution | Subtypes must be substitutable for parent | `List<Shape> shapes` — works with Circle, Rectangle without knowing type |
| **I**nterface Segregation | Small, focused interfaces | `Readable` + `Writable` instead of `ReadWritable` |
| **D**ependency Inversion | Depend on abstractions, not concretions | `@Autowired NotificationService` (interface), Spring injects EmailNotification |

### 🗣️ Answering Approach
"SOLID is five design principles. Single Responsibility — in my project, OrderService only handles order logic, notification is separate. Open-Closed — when we added UPI payment, we created a new UPIPaymentStrategy without modifying the existing PaymentProcessor — Strategy pattern. Liskov Substitution — any NotificationService implementation can replace another without breaking the notification flow. Interface Segregation — we have separate Auditable, Soft-Deletable, and Timestamped interfaces instead of one fat Entity interface. Dependency Inversion — our services depend on repository interfaces, not JPA implementations — Spring DI handles the wiring."

### ⚡ Remember
- **S** = one reason to change; **O** = extend don't modify; **L** = subtypes are interchangeable
- **I** = small interfaces; **D** = depend on abstractions
- Spring Boot embodies D naturally with DI + IoC

### 🔗 Cross-references
- OOP principles → [oops-patterns/01-oop-principles.md](../oops-patterns/01-oop-principles.md)

---

<a id="q17"></a>
## Q17. What is CQRS?

### 📝 One-Liner
CQRS (Command Query Responsibility Segregation) separates read and write models — commands mutate state (write DB), queries read data (read-optimized DB) — enabling independent scaling and optimization.

### 🔑 Quick Answer
Traditional CRUD: Same model for reads and writes. **CQRS**: **Command side** — handles create/update/delete, writes to primary DB (normalized). **Query side** — handles reads, reads from denormalized read store (optimized for queries). Sync between them via domain events (Kafka/RabbitMQ). Benefits: read and write scale independently, read model optimized for queries (no JOINs), write model optimized for integrity. *(CQRS = likhne ka model alag, padhne ka model alag — read ka DB query-friendly banao, write ka DB integrity-focused banao)*

### 📖 How It Works (Detailed Explanation)

```
Traditional:  Client → Same Service → Same DB (CRUD model)

CQRS:
  Write side:  Client → Command Handler → Write DB (normalized)
                                              │ domain event
                                              ▼
  Read side:   Client → Query Handler  → Read DB (denormalized, fast)
```

| Aspect | Write Side | Read Side |
|--------|-----------|-----------|
| Model | Domain entities (normalized) | DTOs/projections (denormalized) |
| DB | PostgreSQL (ACID, normalized) | Elasticsearch / Redis / materialized views |
| Operations | Create, Update, Delete | Read-only queries |
| Scaling | Scale for write throughput | Scale for read throughput |
| Consistency | Strong (ACID) | Eventually consistent |

### 🗣️ Answering Approach
"CQRS separates the read and write models of a system. The command side handles creates, updates, and deletes, writing to a normalized database optimized for data integrity. The query side handles reads from a denormalized store optimized for query performance — no complex JOINs needed. Domain events synchronize the two sides, typically via Kafka. In our e-commerce system, order creation writes to PostgreSQL, and a Kafka consumer updates an Elasticsearch index for fast order search. This lets us scale reads and writes independently."

### ⚡ Remember
- CQRS = separate read model and write model
- Write: normalized DB (integrity); Read: denormalized DB (performance)
- Sync via events (Kafka/RabbitMQ) → eventually consistent
- Use when: read/write patterns differ significantly, high read:write ratio

### 🔗 Cross-references
- Event Sourcing + CQRS → [system-design/02-coordination-failover-eventsourcing.md](../system-design/02-coordination-failover-eventsourcing.md)

---

<a id="q18"></a>
## Q18. How can we make Kafka Producer idempotent?

### 📝 One-Liner
Set `enable.idempotence=true` — Kafka assigns a Producer ID and sequence number to each message, detecting and deduplicating retries at the broker level.

### 🔑 Quick Answer
**Idempotent producer** ensures exactly-once delivery per partition even on retries. Enable with: `properties.put("enable.idempotence", "true")`. Kafka assigns each producer a **Producer ID (PID)** and each message a **sequence number**. If a message is retried (network failure), the broker checks if that PID+sequence was already written — if yes, it acknowledges without re-writing (dedup). Requires `acks=all` (set automatically), `max.in.flight.requests.per.connection ≤ 5`, and `retries > 0`. *(Idempotent producer = retry karne pe duplicate message nahi banega — Kafka broker PID + sequence number se detect karta hai ki ye message pehle aa chuka hai)*

### 📖 How It Works (Detailed Explanation)

```
Without Idempotence:
Producer → sends msg → Broker saves → ACK lost (network)
Producer → retries  → Broker saves AGAIN → DUPLICATE!

With Idempotence (enable.idempotence=true):
Producer → sends msg (PID=1, seq=5) → Broker saves → ACK lost
Producer → retries  (PID=1, seq=5) → Broker checks: "seq=5 already!"
                                    → Broker ACKs without saving → NO DUPLICATE ✅
```

```java
// ✅ Idempotent Kafka Producer configuration
@Configuration
public class KafkaProducerConfig {
    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> props = new HashMap<>();
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);  // key setting
        props.put(ProducerConfig.ACKS_CONFIG, "all");               // auto-set
        props.put(ProducerConfig.RETRIES_CONFIG, Integer.MAX_VALUE); // auto-set
        props.put(ProducerConfig.MAX_IN_FLIGHT_REQUESTS_PER_CONNECTION, 5);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(props);
    }
}
```

**For exactly-once across partitions** → use **Kafka Transactions**:
```java
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "order-producer-1");
// Then: producer.beginTransaction() → send() → commitTransaction()
```

### 🗣️ Answering Approach
"To make a Kafka producer idempotent, I set enable.idempotence to true. This tells Kafka to assign a Producer ID and sequence number to each message. If a send is retried due to a transient failure, the broker detects the duplicate via the PID and sequence — it acknowledges without writing again. This guarantees at-least-once becomes effectively exactly-once per partition. For exactly-once across multiple partitions or topics, I use Kafka transactions with a transactional.id, wrapping sends in beginTransaction/commitTransaction."

### ⚡ Remember
- `enable.idempotence=true` = exactly-once per partition (dedup retries)
- Kafka uses PID + sequence number for deduplication
- Auto-sets: `acks=all`, `retries=MAX`, `max.in.flight ≤ 5`
- For cross-partition exactly-once → Kafka Transactions (`transactional.id`)

### 🔗 Cross-references
- Kafka advanced patterns → [architecture/07-devops-kafka-advanced.md](../architecture/07-devops-kafka-advanced.md)

---

<a id="q19"></a>
## Q19. Load Balancer policies — if 3 instances are running, how to check which is available?

### 📝 One-Liner
Load balancers use health checks (HTTP `/actuator/health`) + distribution policies: Round Robin, Least Connections, Weighted, IP Hash — the health check determines instance availability.

### 🔑 Quick Answer
**Health checks**: Load balancer periodically calls each instance's health endpoint (`/actuator/health`). If an instance doesn't respond or returns unhealthy status, it's removed from the pool. **Policies**: **Round Robin** (cyclic), **Least Connections** (send to least busy), **Weighted** (more traffic to powerful instances), **IP Hash** (same client → same instance for session affinity). In Spring Boot: Spring Cloud LoadBalancer + Eureka/Consul for service discovery. *(Load balancer health check karta hai — instance healthy hai toh traffic bhejo, unhealthy hai toh pool se nikaal do)*

### 📖 How It Works (Detailed Explanation)

```
Load Balancer
  │
  ├── Health Check every 10s: GET /actuator/health
  │   Instance-1: ✅ 200 OK → ACTIVE
  │   Instance-2: ✅ 200 OK → ACTIVE
  │   Instance-3: ❌ Timeout → REMOVED from pool
  │
  ├── Policy: Least Connections
  │   Instance-1: 15 active connections
  │   Instance-2: 8 active connections
  │   → Next request → Instance-2 ✅
```

| Policy | How It Works | Best For |
|--------|-------------|----------|
| **Round Robin** | Cyclic: 1 → 2 → 3 → 1 → 2 → 3 | Equal-capacity instances |
| **Least Connections** | Route to instance with fewest active connections | Varying request durations |
| **Weighted Round Robin** | Instance-1 gets 3x, Instance-2 gets 1x | Mixed-capacity instances |
| **IP Hash** | Hash(client IP) → always same instance | Session affinity (sticky sessions) |
| **Random** | Random instance selection | Simple, stateless services |

### 🗣️ Answering Approach
"Load balancers determine availability through health checks — periodic HTTP calls to each instance's health endpoint. In Spring Boot, I configure /actuator/health to include disk, DB, and custom health indicators. If an instance fails 3 consecutive health checks, it's removed from the pool. For distribution, I choose the policy based on the use case: Round Robin for stateless APIs, Least Connections when request durations vary (some endpoints are slow), and IP Hash when we need session affinity. In our microservices, we use Spring Cloud LoadBalancer with Eureka for client-side load balancing."

### ⚡ Remember
- Health check = `/actuator/health` (Spring Boot default)
- Round Robin = simplest; Least Connections = smart for varying loads
- Client-side LB: Spring Cloud LoadBalancer + Eureka/Consul
- Server-side LB: AWS ALB, Nginx, HAProxy

---

<a id="q20"></a>
## Q20. Write a query to fetch employees with salary more than the average of all employees' salaries.

### 📝 One-Liner
Use a subquery: `SELECT * FROM employees WHERE salary > (SELECT AVG(salary) FROM employees)`.

### 🔑 Quick Answer
```sql
SELECT e.id, e.name, e.department, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees);
```
The subquery `(SELECT AVG(salary) FROM employees)` computes the average across all employees. The outer query filters employees whose salary exceeds that average. *(Average salary nikalo sabka, phir usme se zyada salary wale employees filter karo)*

### 📖 How It Works (Detailed Explanation)

```sql
-- Method 1: Subquery (universal SQL)
SELECT e.id, e.name, e.department, e.salary
FROM employees e
WHERE e.salary > (SELECT AVG(salary) FROM employees);

-- Method 2: Using CTE (Common Table Expression)
WITH avg_sal AS (
    SELECT AVG(salary) AS avg_salary FROM employees
)
SELECT e.id, e.name, e.department, e.salary
FROM employees e, avg_sal
WHERE e.salary > avg_sal.avg_salary;

-- Method 3: Using HAVING with GROUP BY (department-wise)
SELECT department, name, salary
FROM employees e1
WHERE salary > (
    SELECT AVG(salary) FROM employees e2 
    WHERE e2.department = e1.department   -- correlated subquery
)
ORDER BY department, salary DESC;

-- JPQL equivalent in Spring Data
@Query("SELECT e FROM Employee e WHERE e.salary > (SELECT AVG(e2.salary) FROM Employee e2)")
List<Employee> findAboveAverage();
```

### 🗣️ Answering Approach
"I'd use a subquery approach: SELECT employees WHERE salary is greater than a subquery that computes AVG(salary). For department-wise comparison, I'd use a correlated subquery where each employee's salary is compared against their department's average. In Spring Data JPA, I'd write this as a @Query with JPQL. For performance on large tables, I'd ensure there's an index on the salary column."

### ⚡ Remember
- Subquery: `WHERE salary > (SELECT AVG(salary) FROM employees)`
- CTE: Cleaner for complex queries with multiple references to the average
- Correlated subquery: For per-department comparison (slower — runs per row)
- Index on `salary` column improves performance

---

<a id="q21"></a>
## Q21. Find the first unique character and its index. Example: String input = "hackathon"

### 📝 One-Liner
Count character frequency with a `LinkedHashMap` (preserves insertion order), then iterate to find the first character with count 1.

### 🔑 Quick Answer
Use `LinkedHashMap` to count frequency (preserves insertion order), then find the first entry with count == 1. For "hackathon": h=2, a=2, c=1 → first unique = 'c' at index 2. *(Pehle frequency count karo LinkedHashMap mein, phir pehla character dhundho jiski frequency 1 hai — LinkedHashMap order maintain karta hai)*

### 💻 Code
```java
// ✅ Method 1: LinkedHashMap (preserves insertion order)
public static int firstUniqueChar(String input) {
    LinkedHashMap<Character, Integer> freq = new LinkedHashMap<>();
    for (char c : input.toCharArray()) {
        freq.merge(c, 1, Integer::sum);
    }
    for (Map.Entry<Character, Integer> entry : freq.entrySet()) {
        if (entry.getValue() == 1) {
            char ch = entry.getKey();
            System.out.println("First unique: '" + ch + "' at index " + input.indexOf(ch));
            return input.indexOf(ch);
        }
    }
    return -1; // no unique character
}

// ✅ Method 2: Java 8 Streams
public static OptionalInt firstUniqueStream(String input) {
    Map<Character, Long> freq = input.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(Function.identity(), 
                 LinkedHashMap::new, Collectors.counting()));
    
    return freq.entrySet().stream()
        .filter(e -> e.getValue() == 1)
        .mapToInt(e -> input.indexOf(e.getKey()))
        .findFirst();
}

// Test: "hackathon"
// h=2, a=2, c=1, k=1, t=1, o=1, n=1
// First unique: 'c' at index 2
```

### ⚠️ Pitfalls / Gotchas
- Use `LinkedHashMap`, NOT `HashMap` — HashMap doesn't guarantee insertion order
- `indexOf()` returns the FIRST occurrence — correct for this problem
- Edge case: empty string → return -1; all duplicates → return -1

### ⚡ Remember
- `LinkedHashMap` = HashMap + insertion order (key for this problem)
- frequency map → first entry with count 1 → `indexOf()` for position
- Java 8: `Collectors.groupingBy` with `LinkedHashMap::new` supplier

### 🔗 Cross-references
- Stream coding problems → [core/08-streams-coding-basic.md](../languages/java/core/08-streams-coding-basic.md)

---

[← Back to Company Specific](./README.md) | [← Back to Home](../../README.md)
