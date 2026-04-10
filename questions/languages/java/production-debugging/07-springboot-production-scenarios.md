# 🛠️ Spring Boot Production — Scenario-Based Debugging Questions (Q1–Q15)

> **Source**: Real production scenario questions for Spring Boot applications  
> **Coverage**: Deploy failures, configuration issues, connection pools, async pitfalls, circuit breakers, Docker differences  
> **Level**: 3+ YOE (Senior Java/Spring Boot Developer — production mindset)  
> **Key**: Interviewers want to see your debugging THOUGHT PROCESS, not textbook answers

---

<a id="q1"></a>
## Q1. App works locally but fails in production. First 5 things you check?

### 📝 One-Liner
Environment delta — the difference between local and production configs, dependencies, infrastructure, and runtime behavior.

### 🔑 Quick Answer
**5-step checklist:**
1. **Profile/Config**: Which `application-{profile}.yml` is active? Env vars overriding properties? Check `spring.profiles.active` in prod
2. **Dependencies**: Same JDK version? Same library versions? (`java -version`, compare `pom.xml` vs deployed JAR)
3. **Connectivity**: Can the app reach DB, Redis, Kafka, external APIs from production network? DNS resolution, firewall rules, VPN
4. **Resource limits**: Docker memory limit, CPU throttling, file descriptor limit, connection pool sizing for prod scale
5. **Logs**: Check startup logs for bean creation failures, port conflicts, health check failures, permission denied

*(Locally sab perfect — prod mein fail. Pehle check karo: config, JDK version, network connectivity, resource limits, startup logs)*

### 📖 How It Works (Detailed Explanation)

| Check | Local | Production |
|-------|-------|------------|
| Profile | `default` or `dev` | `prod` (different DB, Redis, secrets) |
| JDK | OpenJDK 17.0.8 | Maybe 17.0.2 (different GC defaults) |
| DB | H2 / local MySQL | RDS / Aurora (network latency, SSL required) |
| Memory | 8GB available | Docker limit 512MB |
| Files | Local filesystem | Read-only container filesystem |
| Env vars | `.env` file | Kubernetes secrets / vault |

### 🗣️ Answering Approach
"I follow a systematic checklist. First, I verify which Spring profile is active in production and compare the resolved config with local. Second, I check JDK and library version parity. Third, I test network connectivity from the production host to all downstream dependencies — many issues are firewall or DNS-related. Fourth, I check resource limits — Docker memory caps, CPU throttling. Fifth, I read the full startup log — Spring Boot logs active profile, datasource URL, and bean creation output on startup."

### ⚡ Remember
- Most local-vs-prod failures are CONFIG or NETWORK differences
- Always check: profiles, connectivity, resource limits, startup logs
- Actuator `/env` and `/configprops` endpoints reveal resolved config

---

<a id="q2"></a>
## Q2. APIs were fast locally but slow in production. Narrow it down.

### 📝 One-Liner
Production has network latency, real data volume, real concurrency, and shared resource contention — any of which can expose slow paths invisible locally.

### 🔑 Quick Answer
**Narrowing approach:**
1. **Is it ALL APIs or specific ones?** — All = infrastructure issue. Specific = code/query issue
2. **Compare data volume** — Local: 100 rows; Prod: 10M rows. Missing indexes, full table scans
3. **Network hops** — Local: in-process; Prod: API → LB → container → DB (each hop adds latency)
4. **Connection pool** — Local: pool never saturates; Prod: 200 concurrent requests exhaust HikariCP pool
5. **Downstream** — Local: mock/fast; Prod: real service with latency, rate limits, authentication overhead
6. **GC pressure** — Local: light load, mild GC; Prod: high allocation rate, frequent pauses

*(Local mein fast kyunki data kam, concurrency kam, sab same machine pe. Prod mein network, data size, pool saturation — sab alag)*

### 🗣️ Answering Approach
"I'd narrow it down systematically. If all APIs are slow, it's infrastructure — network latency, DNS, or resource constraints. If specific APIs are slow, I'd compare the SQL execution plan with production data volume — queries fast on 100 rows may full-scan on 10M. I'd check HikariCP metrics — are threads waiting for connections? I'd look at downstream service latency — external APIs may have rate limits or SLA degradation. Finally, I'd check GC metrics — high allocation rate under load causes more frequent pauses."

### ⚡ Remember
- All APIs slow → infrastructure; Specific API slow → code/query
- Check: data volume + indexes, connection pool metrics, downstream latency
- Tools: Spring Actuator metrics, Micrometer + Grafana, slow query log

---

<a id="q3"></a>
## Q3. application.properties changes not reflecting in prod. Why?

### 📝 One-Liner
Profile override, environment variable override, cached config, or wrong JAR deployed — the property IS being read, just not from where you expect.

### 🔑 Quick Answer
Property resolution order (higher overrides lower): (1) **Command-line args** (`--server.port=9090`), (2) **Environment variables** (`SERVER_PORT=9090`), (3) **`application-{profile}.properties`** (profile-specific), (4) **`application.properties`** (default). If your change is in `application.properties` but production uses `application-prod.properties` with the same key, the profile-specific value wins. Also: (5) **ConfigServer** may override, (6) **Kubernetes ConfigMap** mounted as file, (7) **Docker layer caching** — old JAR deployed. *(Property change kiya but reflect nahi ho raha — check karo: profile override, env variable, ya purana JAR deploy hua hai)*

### 🗣️ Answering Approach
"I'd check the actual resolved property value using Actuator /env endpoint. Spring has a strict property source ordering — environment variables beat application.properties, profile-specific files beat default. If I changed application.properties but production activates the 'prod' profile, and application-prod.properties has that same key, my change is overridden. I'd also verify the correct JAR is deployed — Docker layer caching or CI/CD issues can deploy stale artifacts."

### ⚡ Remember
- Property precedence: cmd args > env vars > profile-specific > default
- Use Actuator `/env` to see actual resolved values
- Docker layer caching → stale JAR → changes not reflected

---

<a id="q4"></a>
## Q4. CPU usage is low but requests time out. What's the bottleneck?

### 📝 One-Liner
Threads are waiting (I/O-bound), not computing — the bottleneck is a downstream dependency, lock contention, or exhausted resource pool.

### 🔑 Quick Answer
Same root cause as Java Runtime Q2 (→ `production-debugging/06-java-runtime-scenarios.md` Q2) but Spring Boot specific investigation: (1) **HikariCP pool exhausted** — check `hikaricp.connections.pending` metric, (2) **RestTemplate blocking** — synchronous HTTP call to slow downstream, (3) **@Transactional holding connection too long** — business logic inside transaction, (4) **Redis BLPOP/BRPOP** — blocking Redis operations, (5) **File I/O** — reading large files synchronously. *(CPU free but timeout — threads I/O pe block hain. HikariCP pool, RestTemplate call, ya @Transactional ke andar slow logic — check karo)*

### 🗣️ Answering Approach
"Low CPU with timeouts means threads are blocked on I/O, not computing. In Spring Boot, I'd check HikariCP connection pool metrics first — if all connections are in use, new requests wait. Then I'd look at RestTemplate/WebClient calls to downstream services — are they timing out? I'd also check if @Transactional methods contain slow non-DB logic that holds the connection unnecessarily. Thread dump would show me exactly what each thread is waiting on."

### ⚡ Remember
- Spring Boot specific: HikariCP metrics, RestTemplate timeouts, @Transactional scope
- Cross-ref: `production-debugging/06-java-runtime-scenarios.md` Q2 for general JVM diagnosis

---

<a id="q5"></a>
## Q5. Multiple beans of same type causing startup failure. How do you fix it?

### 📝 One-Liner
`NoUniqueBeanDefinitionException` — Spring can't autowire when multiple beans match the type. Fix with `@Primary`, `@Qualifier`, or `@ConditionalOnMissingBean`.

### 🔑 Quick Answer
When two beans implement the same interface, Spring doesn't know which to inject. **Fixes:**
1. **`@Primary`** — marks one as the default choice
2. **`@Qualifier("beanName")`** — explicitly specify which bean
3. **`@ConditionalOnMissingBean`** — only create if no other bean of this type exists
4. **Collection injection** — `List<MyService>` injects ALL implementations
5. **`@Profile`** — different beans for different profiles

*(Same type ke 2 beans hain — Spring confuse ho raha hai kise inject kare. @Primary ya @Qualifier se bata do)*

### 💻 Code
```java
// Two implementations
@Service
public class EmailNotificationService implements NotificationService { }

@Service
@Primary  // ← This one is injected by default
public class SmsNotificationService implements NotificationService { }

// OR explicit qualifier
@Autowired
@Qualifier("emailNotificationService")
private NotificationService service;

// OR conditional (auto-config pattern)
@Bean
@ConditionalOnMissingBean(NotificationService.class)
public NotificationService defaultNotification() {
    return new SmsNotificationService();
}
```

### ⚡ Remember
- `@Primary` = default choice when multiple match
- `@Qualifier` = explicit selection by name
- `@ConditionalOnMissingBean` = create only if missing (auto-config pattern)

---

<a id="q6"></a>
## Q6. @Transactional rollback not happening. What went wrong?

### 📝 One-Liner
Spring `@Transactional` only rolls back on **unchecked exceptions** (RuntimeException) by default. Checked exceptions do NOT trigger rollback unless explicitly configured.

### 🔑 Quick Answer
**Top causes of no rollback:**
1. **Checked exception** — `@Transactional` only rolls back on `RuntimeException` and `Error` by default. Fix: `@Transactional(rollbackFor = Exception.class)`
2. **Self-invocation** — calling `@Transactional` method from same class bypasses proxy. Fix: inject self or refactor
3. **Private method** — `@Transactional` on private method is ignored (proxy can't intercept). Fix: make public
4. **Exception caught internally** — try-catch swallows exception → Spring never sees it → no rollback
5. **Wrong transaction manager** — multiple datasources, annotation bound to wrong manager

*(Rollback nahi ho raha — pehle check karo: checked exception toh nahi hai? Same class se call toh nahi? Exception catch toh nahi kar rahe? Private method toh nahi?)*

### 💻 Code
```java
// ❌ No rollback — checked exception
@Transactional
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("fail"); // ← Checked exception → NO rollback!
}

// ✅ Fix: Explicit rollback for checked exceptions
@Transactional(rollbackFor = Exception.class)
public void process() throws IOException {
    repo.save(entity);
    throw new IOException("fail"); // ← NOW rolls back
}

// ❌ No rollback — self-invocation bypasses proxy
public void outer() {
    this.inner(); // ← Direct call, proxy not involved!
}

@Transactional
public void inner() { /* ... */ }
```

### 🗣️ Answering Approach
"The first thing I check is the exception type — @Transactional only rolls back on unchecked exceptions by default. If a checked exception is thrown, the transaction commits. Fix: add `rollbackFor = Exception.class`. Second, I check for self-invocation — if a non-transactional method in the same class calls a @Transactional method, the proxy is bypassed. Third, I check if the exception was caught inside the method — if the code catches the exception, Spring never sees it and proceeds with commit."

### ⚡ Remember
- Checked exception = NO rollback (default). Add `rollbackFor = Exception.class`
- Self-invocation = proxy bypass. Inject self or extract to separate bean
- Caught exception = Spring never sees it → commits

---

<a id="q7"></a>
## Q7. Database connection pool exhausts. Diagnose root cause.

### 📝 One-Liner
Connections are acquired but not returned — long-running transactions, connection leaks (unclosed in try-catch), or pool sized too small for production concurrency.

### 🔑 Quick Answer
**Diagnosis checklist:**
1. **Leaked connections** — `@Transactional` missing on service method → connection acquired by JPA but never committed/released
2. **Long transactions** — business logic inside `@Transactional` that calls external APIs → connection held during HTTP call
3. **N+1 queries** — hundreds of queries per request → connection held longer
4. **Pool too small** — HikariCP default `maximumPoolSize=10` → 10 concurrent requests saturate it
5. **Connection timeout** — slow queries hold connections → others wait → timeout cascade

**HikariCP key configs:**
```yaml
spring.datasource.hikari:
  maximum-pool-size: 20      # Match: (core_count * 2) + spindle_count
  minimum-idle: 5
  connection-timeout: 30000  # 30s wait for connection
  leak-detection-threshold: 60000  # Log warning if connection held > 60s
```

### 🗣️ Answering Approach
"I'd first enable HikariCP's leak-detection-threshold to log stack traces when connections are held too long. Then I'd check for transactions that include non-DB operations — like calling an external API inside @Transactional, which holds a connection during the HTTP round trip. I'd also verify the pool size matches production concurrency — HikariCP's formula is (core_count × 2) + disk_spindle_count for most use cases. N+1 queries are another common cause — hundreds of queries per request extend connection hold time."

### ⚡ Remember
- `leak-detection-threshold` = most useful HikariCP config for debugging
- Don't call external APIs inside `@Transactional`
- Pool formula: `(CPU cores × 2) + disk spindle count`

---

<a id="q8"></a>
## Q8. Scheduled jobs affecting API latency. How to isolate?

### 📝 One-Liner
Scheduled jobs and API requests share the same thread pool, connection pool, and memory — a heavy batch job starves API resources.

### 🔑 Quick Answer
**Problems:**
1. **Shared connection pool** — scheduled job uses 15 of 20 connections → APIs can't get connections
2. **Shared thread pool** — `@Scheduled` runs on Spring's scheduler pool → CPU-intensive job steals threads
3. **GC pressure** — batch job creates many objects → GC pauses affect API latency
4. **DB locks** — scheduled job locks rows → API queries wait

**Isolation fixes:**
1. **Separate datasource** for batch jobs with its own connection pool
2. **Dedicated scheduler thread pool**: `spring.task.scheduling.pool.size=5`
3. **@Async with custom executor** for heavy jobs
4. **Run scheduled jobs on separate instance** (worker node pattern)
5. **Rate-limit batch operations** — chunk processing with delays

*(Scheduled job aur API same pool share karte hain — job heavy hai toh API ke liye resources nahi bachte. Alag pool ya alag instance use karo)*

### 🗣️ Answering Approach
"The root cause is resource sharing. Scheduled jobs and API endpoints share connection pools, thread pools, and heap. A heavy batch job consuming most DB connections starves API requests. I'd isolate by: giving the batch job a separate datasource with its own connection pool, configuring a dedicated thread pool for scheduled tasks, and ideally running batch jobs on a separate worker instance that doesn't serve API traffic."

### ⚡ Remember
- Separate connection pools for batch vs API
- Worker node pattern: dedicated instance for scheduled jobs
- `spring.task.scheduling.pool.size` for scheduler thread pool

---

<a id="q9"></a>
## Q9. Docker behavior differs from local. What diverged?

### 📝 One-Liner
Docker imposes resource limits (memory/CPU capping), uses a different filesystem, networking stack, timezone, locale, and potentially a different JDK build — all invisible locally.

### 🔑 Quick Answer
**Common divergences:**
1. **Memory limit** — Container limit 512MB but JVM defaults to 25% of HOST memory → OOM kill. Fix: `-XX:MaxRAMPercentage=75`
2. **Timezone** — Container uses UTC, local uses IST → scheduled jobs run at wrong time. Fix: `TZ=Asia/Kolkata`
3. **Filesystem** — Container fs is read-only/ephemeral → temp file creation fails
4. **DNS** — Docker DNS resolver differs → hostname resolution fails for internal services
5. **CPU throttling** — cgroup CPU limit → JVM calculates wrong `availableProcessors()` → wrong thread pool sizes
6. **Locale** — Different default encoding → file reading/String conversion breaks
7. **JDK build** — Local: Oracle; Docker: Eclipse Temurin → minor behavior differences

### 🗣️ Answering Approach
"The biggest Docker gotcha for JVM apps is memory. If the container limit is 512MB but the JVM calculates heap based on the host's 16GB RAM, the kernel OOM-kills the container. I use `-XX:MaxRAMPercentage=75` to respect container limits. Second is timezone — containers default to UTC. Third is CPU — cgroup limits affect `Runtime.availableProcessors()`, which affects default thread pool sizes. I explicitly set thread pool sizes rather than relying on availableProcessors()."

### ⚡ Remember
- JVM must respect container limits: `-XX:MaxRAMPercentage=75`
- Set timezone explicitly: `TZ=Asia/Kolkata`
- Set thread pools explicitly (don't rely on `availableProcessors()` in containers)

---

<a id="q10"></a>
## Q10. New deploy but old behavior persists. Why?

### 📝 One-Liner
Stale artifact — cached Docker layer, load balancer still routing to old instance, classpath conflict with old JAR, or client-side caching (CDN/browser).

### 🔑 Quick Answer
**Checklist:**
1. **Docker layer cache** — `docker build` reused cached layer → old code. Fix: `--no-cache` or proper layer ordering
2. **Rolling deployment in progress** — some pods still running old version. Check pod versions
3. **CDN/browser cache** — static assets cached. Fix: cache-busting with content hash
4. **Maven/Gradle local cache** — `~/.m2` has old snapshot. Fix: `mvn clean install -U`
5. **Spring Boot DevTools** — remote restart using old classpath
6. **Class conflict** — old dependency JAR has same class → classpath order determines which loads

*(Deploy kiya but purana behavior dikh raha hai — purana JAR deploy hua, CDN cache, ya rolling update abhi complete nahi hua)*

### 🗣️ Answering Approach
"I'd verify the deployed artifact first — check the build version or git commit hash embedded in the JAR. If it's correct, I'd check if the load balancer is still routing to old instances during a rolling deployment. For frontend issues, I'd check CDN and browser caching. For backend, I'd check if the Docker build used a cached layer — proper Dockerfile layering and --no-cache flag help. I always embed the git commit hash in the Spring Boot info endpoint for production verification."

### ⚡ Remember
- Embed build version in `/actuator/info` — instant production verification
- Docker: proper layer ordering + `--no-cache` for critical deploys
- Rolling deploy: verify ALL instances updated, not just one

---

<a id="q11"></a>
## Q11. Logs missing in production but present locally. Why?

### 📝 One-Liner
Log level mismatch, wrong appender, log configuration override, or containerized stdout not captured by the log aggregator.

### 🔑 Quick Answer
**Causes:**
1. **Log level** — Local: `DEBUG`; Prod: `WARN` → DEBUG/INFO logs filtered out
2. **Appender** — Local: console appender; Prod: file appender writing to path that doesn't exist
3. **Config override** — `logback-spring.xml` overridden by profile-specific config
4. **Container logging** — App logs to file, but container logging driver only captures stdout/stderr
5. **Async appender** — queue full → logs dropped (discardingThreshold setting)
6. **MDC not propagated** — in async/threaded context, MDC values lost → filtered by MDC-based appender

### 🗣️ Answering Approach
"First, I'd check the effective log level in production via Actuator /loggers endpoint — it often differs from local. Then I'd verify the log output destination — if the app logs to a file but the container log driver only captures stdout, logs are lost. Async appenders can silently drop logs when the queue is full. I always use structured JSON logging in production with console output for container-based deployments."

### ⚡ Remember
- Check effective level: `GET /actuator/loggers/{package}`
- Container logging: log to stdout, not file
- Async appender `discardingThreshold` can silently drop logs

---

<a id="q12"></a>
## Q12. @Async made performance worse. What went wrong?

### 📝 One-Liner
Without a custom executor, `@Async` uses `SimpleAsyncTaskExecutor` which creates a **new thread for every call** — no pooling, unbounded thread creation, OS thrashing.

### 🔑 Quick Answer
**Problems:**
1. **Default executor** — `SimpleAsyncTaskExecutor` creates new thread per task → thousands of threads → context switching overhead → OS thrashing
2. **No backpressure** — unbounded task submission → memory exhaustion from queued Runnables
3. **Exception swallowed** — `@Async void` methods swallow exceptions silently (no Future to check)
4. **Wrong use case** — making a synchronous DB call @Async doesn't help if DB connection pool is the bottleneck
5. **Missing `@EnableAsync`** — annotation present but not activated → runs synchronously

### 💻 Code
```java
// ❌ Uses SimpleAsyncTaskExecutor — unbounded thread creation
@Async
public void processOrder(Order order) { /* ... */ }

// ✅ Custom executor with bounded pool and queue
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("orderExecutor")
    public Executor orderExecutor() {
        ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
        exec.setCorePoolSize(5);
        exec.setMaxPoolSize(10);
        exec.setQueueCapacity(100);
        exec.setRejectedExecutionHandler(new CallerRunsPolicy());
        exec.setThreadNamePrefix("order-async-");
        return exec;
    }
}

@Async("orderExecutor")  // ← Use named executor
public CompletableFuture<Result> processOrder(Order order) {
    // return CompletableFuture for error tracking
}
```

### ⚡ Remember
- Always define custom `ThreadPoolTaskExecutor` with bounded pool + queue
- Return `CompletableFuture` instead of `void` → track errors
- `CallerRunsPolicy` = backpressure (caller thread does the work if queue full)

---

<a id="q13"></a>
## Q13. Circuit breaker stays open and never recovers. Why?

### 📝 One-Liner
The downstream service recovered, but the circuit breaker's **half-open** probe still fails — either health check endpoint differs from actual API, timeout is too aggressive, or no half-open transition configured.

### 🔑 Quick Answer
**Causes:**
1. **Half-open failure** — probe request still fails (different error from original) → circuit stays open
2. **Health check isn't representative** — `/health` returns 200 but the actual API still has issues
3. **Timeout too low** — downstream is slow but working; probe times out → treated as failure
4. **No half-open state** — misconfigured to go directly back to closed → all traffic floods recovering service → fails again
5. **Sliding window not resetting** — old failure count still in window → new successes don't close circuit

### 🗣️ Answering Approach
"I'd check the circuit breaker's state and metrics via Actuator. If it's stuck open, I'd verify the half-open probe is testing the right endpoint with a realistic timeout. Then I'd check the sliding window configuration — some implementations require N consecutive successes in half-open to close, and if any probe fails, it goes back to open. I'd also ensure the wait duration in open state is appropriate — too long delays recovery, too short floods the recovering service."

### 💻 Code
```yaml
# Resilience4j configuration
resilience4j.circuitbreaker:
  instances:
    paymentService:
      slidingWindowSize: 10
      failureRateThreshold: 50        # Open if 50% fail
      waitDurationInOpenState: 30s     # Wait 30s before probe
      permittedNumberOfCallsInHalfOpenState: 3  # 3 probes
      minimumNumberOfCalls: 5
      automaticTransitionFromOpenToHalfOpenEnabled: true
```

### ⚡ Remember
- Half-open probes must use realistic timeout and correct endpoint
- `automaticTransitionFromOpenToHalfOpenEnabled=true` → auto-probe
- Monitor via Actuator: `/actuator/circuitbreakers`

---

<a id="q14"></a>
## Q14. Adding more resources (CPU/RAM) didn't help. Why?

### 📝 One-Liner
The bottleneck isn't CPU or memory — it's I/O bound (DB, network), lock contention, or an algorithmic issue (O(n²) doesn't care about CPU speed).

### 🔑 Quick Answer
More resources help only when the bottleneck IS those resources. Doesn't help when: (1) **I/O-bound** — faster CPU doesn't speed up network calls or disk reads, (2) **Lock contention** — more CPU, but threads still wait for same lock, (3) **Algorithmic complexity** — O(n²) on 1M records is slow regardless of hardware, (4) **DB bottleneck** — app has more memory but DB is the constraint, (5) **Sequential processing** — code processes items sequentially → more cores unused. *(Zyada CPU/RAM diya but koi improvement nahi — bottleneck I/O, lock, ya algorithm mein hai — hardware se solve nahi hoga)*

### 🗣️ Answering Approach
"Before adding resources, I identify the actual bottleneck. If threads spend 90% of time waiting for DB responses, adding CPU won't help — I need to optimize queries or add a cache layer. If it's lock contention, I need to reduce lock scope or switch to lock-free data structures. If it's an O(n²) algorithm, no amount of hardware fixes that — I need a better algorithm. I profile first with JFR or async-profiler to see where time is actually spent."

### ⚡ Remember
- Profile FIRST to identify the actual bottleneck
- Amdahl's Law: if 90% of work is sequential, scaling to 100 cores gives only 10% gain
- I/O-bound: add cache, optimize queries, use async I/O
- CPU-bound: then more CPU/cores actually helps

---

<a id="q15"></a>
## Q15. A specific Spring Boot decision you made that caused production issues. What did you learn?

### 📝 One-Liner
This is a **behavioral/experience question** — interviewers want a real story showing: mistake → diagnosis → fix → lesson learned. No right answer, but show growth.

### 🔑 Quick Answer
**Example answer framework:**

**Scenario**: Used `@Transactional` on a service method that called an external payment API inside the transaction boundary. Locally it worked fine (fast network to mock server). In production, the API call took 3-5 seconds → DB connection held for 3-5 seconds per request → under load, HikariCP pool exhausted → all requests started timing out.

**Root cause**: @Transactional scope was too broad — non-DB work inside transaction boundary.

**Fix**: Extracted the payment API call outside the @Transactional method. DB operations happened in a narrow transaction, API call happened separately with its own error handling.

**Lesson**: Keep @Transactional scope minimal — only wrap actual DB operations. External calls, message sends, file I/O should be OUTSIDE the transaction boundary.

### 🗣️ Answering Approach
"In a previous project, I had a @Transactional method that included an external API call. Locally it was fast because the mock responded in milliseconds. In production, the API took 3-5 seconds, which meant DB connections were held for that duration. Under 200 concurrent requests, the HikariCP pool of 20 connections was instantly exhausted. I fixed it by narrowing the transaction boundary — DB operations in a minimal transaction, external calls outside. The lesson: always keep @Transactional scope as narrow as possible, and never include non-DB I/O inside a transaction."

### ⚡ Remember
- This is a BEHAVIORAL question — share a real story (mistake → fix → lesson)
- Good topics: @Transactional scope, caching misconfiguration, wrong GC choice, missing timeouts
- Show GROWTH: "I now always verify transaction boundaries in code review"
- Avoid: blaming others, trivial issues, or saying "never made a production mistake"

---

## 🔗 Cross-References

| Topic | Detailed Coverage |
|-------|------------------|
| Thread blocking (Q4) | [production-debugging/06-java-runtime-scenarios.md](./06-java-runtime-scenarios.md) Q2 |
| Connection pool tuning (Q7) | [spring/11-springboot-scenario-interviews.md](../spring/11-springboot-scenario-interviews.md) |
| @Transactional pitfalls (Q6) | [spring/12-springboot-rest-jpa-advanced.md](../spring/12-springboot-rest-jpa-advanced.md) Q14, Q25 |
| Docker JVM behavior (Q9) | [company-specific/20-tcs-capgemini-java-backend.md](../../../company-specific/20-tcs-capgemini-java-backend.md) |
| Scaling issues (Q14) | [production-debugging/06-java-runtime-scenarios.md](./06-java-runtime-scenarios.md) Q15 |

---

[← Back to Production Debugging](./README.md) | [← Back to Java](../README.md) | [← Back to Home](../../../../README.md)
