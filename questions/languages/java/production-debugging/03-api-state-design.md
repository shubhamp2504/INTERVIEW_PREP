# 🔥 Production Debugging — API, State & Design Flaws (Q12–Q16)

> **"Production just broke. Let's see how you think."**
> Symptoms → Hypothesis → Diagnosis → Fix

---

<a id="q12"></a>
## Q12. A REST endpoint works fine with few users but crashes when thousands of requests arrive. Why might that happen?

### 📝 One-Liner
**Resource exhaustion under load** — Tomcat thread pool (200 default) saturates, HikariCP connection pool (10 default) saturates, or downstream service slows down → timeout cascade → all threads stuck waiting → new requests rejected or queued indefinitely.

### 🔑 Quick Answer
Works with few users because resources (threads, connections, file handles) aren't contested. Under thousands: **(1) Thread pool exhaustion**: 200 Tomcat threads all busy → accept-count queue (100) fills → OS drops connections → client gets "Connection refused." **(2) Connection pool exhaustion**: 200 threads fight for 10 DB connections → 190 wait for connectionTimeout (default 30s!) → latency 30s+ → cascading timeouts. **(3) Downstream dependency slowdown**: payment service takes 10s instead of 100ms → threads hold connections for 10x longer → amplifies all exhaustion. **(4) Heap pressure**: each request allocates objects → 1000 concurrent requests = GC storms → stop-the-world pauses → further slowdown. **(5) Open file/socket limits**: Linux default `ulimit -n 1024` → exceeded → "Too many open files". *(Kam users pe sab theek — zyada load pe resources khatam ho jaate hain — pool sizes aur timeouts tune karo)*

### 📖 How to Diagnose
```
What crashes look like:

  1. Connection Refused (client side):
     → Tomcat thread pool + accept queue FULL → OS drops connection
     → Check: server.tomcat.threads.max, server.tomcat.accept-count

  2. HikariPool-1 - Connection is not available, request timed out after 30000ms:
     → All 10 connections busy, waited 30s, gave up
     → Check: spring.datasource.hikari.maximum-pool-size
     → Check: spring.datasource.hikari.connection-timeout

  3. java.net.SocketTimeoutException: Read timed out:
     → Downstream service didn't respond in time
     → Check: RestTemplate/WebClient timeout configuration (default = INFINITE!)

  4. java.io.IOException: Too many open files:
     → OS file descriptor limit hit
     → $ ulimit -n → if 1024, increase to 65535
     → Also check for connection/file leaks (connections not closed)

Load test progression:
  10 users:   50ms avg latency ✅
  100 users:  55ms avg latency ✅ (slight increase)
  500 users:  200ms avg (DB pool contention starting)
  1000 users: 3s avg (DB pool waiting dominates)
  2000 users: 30s avg → timeouts → errors → crash
```

### 💻 Code — Production-Quality Configuration
```yaml
# application.yml — tuned for high traffic
server:
  tomcat:
    threads:
      max: 200
      min-spare: 50
    accept-count: 200
    connection-timeout: 5000    # 5s to accept connection, not default 60s

spring:
  datasource:
    hikari:
      maximum-pool-size: 30     # 30 not 10 (default)
      minimum-idle: 10
      connection-timeout: 3000  # 3s not 30s (default) — fail fast
      max-lifetime: 1800000     # 30 min, recycle before DB closes
      leak-detection-threshold: 30000
```

```java
// ✅ RestTemplate with TIMEOUTS (default is INFINITE!)
@Bean
public RestTemplate restTemplate() {
    var factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(3000);   // 3s to establish connection
    factory.setReadTimeout(5000);      // 5s to read response
    return new RestTemplate(factory);
}

// ✅ Circuit Breaker — prevent cascade when downstream fails
@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
public PaymentResponse callPaymentService(PaymentRequest request) {
    return restTemplate.postForObject(paymentUrl, request, PaymentResponse.class);
}

public PaymentResponse paymentFallback(PaymentRequest request, Throwable t) {
    return PaymentResponse.pendingRetry(request.getOrderId());
}
```

### ⚡ Remember
- **Default pool sizes are too small** for production (HikariCP=10, Tomcat=200)
- **Default timeouts are too long or INFINITE** → set explicitly *(Timeouts set karo — default infinite hai toh thread kabhi release nahi hota)*
- **RestTemplate/WebClient default = no timeout** → set connect + read timeouts
- **Circuit breaker** on all downstream calls to prevent cascade
- Load test: 10x expected peak traffic to find the breaking point

### 🔗 Cross-References
- Q1 (01-jvm-memory-performance) → Slow API diagnosis (GC, thread pool, connection pool)
- architecture/01 → Circuit Breakers (Resilience4j)

---

<a id="q13"></a>
## Q13. A background job sometimes runs twice even though it was scheduled once. What might cause this?

### 📝 One-Liner
**No distributed lock / no idempotency** — multiple app instances each run the same `@Scheduled` task, or the scheduler fires again before the previous execution finishes, or a retry mechanism re-triggers without deduplication.

### 🔑 Quick Answer
**(1) Multiple instances**: deployed on 3 pods in K8s → each pod has the same `@Scheduled(cron = "0 0 * * * *")` → job runs 3 times. The scheduler runs IN-PROCESS, not centrally. **(2) Overlapping execution**: job takes 40 min, scheduled every 30 min, no `@Scheduled(fixedDelay)` → second execution starts before first finishes. **(3) Retry on timeout**: job succeeds but response times out → caller retries → job runs again. **(4) Message broker redelivery**: SQS/RabbitMQ message processing succeeds but ACK fails → message redelivered → processed again. **Fix**: distributed lock (ShedLock), idempotency keys, or centralized scheduler (Quartz cluster mode). *(Multiple pods pe @Scheduled = har pod pe job chalega — ShedLock se ek hi pe chalao)*

### 📖 How to Diagnose
```
Symptoms:
  → Emails sent twice, reports generated twice, duplicate DB entries
  → Problem appeared after scaling to multiple instances
  → Problem appeared after increase in job duration

Diagnosis:
  1. CHECK: How many application instances are running?
     → If N instances → @Scheduled runs N times (once per instance)

  2. CHECK: Does fixedRate overlap?
     @Scheduled(fixedRate = 60_000)  → fires every 60s REGARDLESS of previous completion
     @Scheduled(fixedDelay = 60_000) → fires 60s AFTER previous completes ← safer
     
  3. CHECK: Is there a retry mechanism?
     → Spring Retry, Resilience4j Retry → may re-trigger on timeout
     → Message broker → at-least-once delivery = can redeliver

Multi-instance problem:
  Pod 1: @Scheduled(cron="0 0 * * * *") → runs ✅
  Pod 2: @Scheduled(cron="0 0 * * * *") → runs ✅  ← duplicate!
  Pod 3: @Scheduled(cron="0 0 * * * *") → runs ✅  ← duplicate!
  → Job runs 3 times — each instance has its own scheduler
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: @Scheduled runs on EVERY instance
@Scheduled(cron = "0 0 2 * * *")  // 2 AM daily
public void generateDailyReport() {
    reportService.generate(); // runs on ALL pods!
}

// ✅ FIX: ShedLock — distributed lock ensures only one instance runs
// pom.xml: net.javacrumbs.shedlock:shedlock-spring + shedlock-provider-jdbc-template

@Configuration
@EnableSchedulerLock(defaultLockAtMostFor = "PT30M")  // max lock 30 min
public class ShedLockConfig {
    @Bean
    public LockProvider lockProvider(DataSource dataSource) {
        return new JdbcTemplateLockProvider(dataSource);
    }
}

@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "dailyReport",
    lockAtLeastFor = "PT5M",   // hold lock min 5 min (prevent rapid re-execution)
    lockAtMostFor = "PT30M")   // max 30 min (release if instance crashes)
public void generateDailyReport() {
    reportService.generate(); // only ONE pod runs this ✅
}

// ✅ FIX: Idempotent job (safe even if it runs twice)
@Scheduled(cron = "0 0 2 * * *")
@SchedulerLock(name = "dailyReport")
public void generateDailyReport() {
    LocalDate today = LocalDate.now();
    if (reportRepository.existsByDate(today)) {
        log.info("Report already generated for {}, skipping", today);
        return; // idempotent — safe to call multiple times
    }
    reportService.generateForDate(today);
}
```

### ⚡ Remember
- **@Scheduled runs per-instance** — N pods = N executions *(3 pods = 3 baar job chalega — ShedLock lagao)*
- **ShedLock** = DB-based distributed lock for @Scheduled → only one instance runs
- **fixedDelay > fixedRate** — prevents overlapping execution
- **Idempotency** = design job so running twice produces same result (safety net)
- For complex scheduling: **Quartz** in cluster mode (DB-backed coordination)

### 🔗 Cross-References
- spring-batch/01 Q19 → Idempotency in Spring Batch (JobInstance uniqueness)
- spring-batch/15 → Production scenario: preventing duplicate processing
- system-design/02 Q8 → Distributed Locks (ZooKeeper/Redis)

---

<a id="q14"></a>
## Q14. After adding caching, users start seeing outdated responses. What design issue might exist?

### 📝 One-Liner
**Cache invalidation failure** — data updated in the database but the cache still holds the old value. Missing TTL, no write-through/invalidation strategy, or race condition between cache write and DB write.

### 🔑 Quick Answer
**(1) No TTL**: cache entry lives forever → data changed in DB but cache never refreshes. **(2) No invalidation on write**: update goes to DB but nobody tells the cache → stale read on next request. **(3) Cache-aside race condition**: Thread A reads cache miss → queries DB → Thread B updates DB + invalidates cache → Thread A writes OLD value back to cache → stale forever. **(4) Multi-instance cache**: local in-memory cache on Pod 1 updated, Pod 2's cache still has old value. **(5) Read-through not updated**: cache fetches from DB on miss, but DB was updated between TTL window. **The hardest problem in CS**: "There are only two hard things: cache invalidation and naming things." *(Cache mein purana data — TTL lagao, write pe invalidate karo, ya write-through use karo)*

### 📖 How to Diagnose
```
Symptoms:
  → User updates profile, still sees old name on next page load
  → Admin updates price, customers see old price
  → Data correct in DB but wrong in API response
  → Problem disappears after restarting the service (cache cleared)

Diagnosis:
  1. IS THERE A TTL?
     → No TTL = cached forever = stale eventually
     → TTL too long (24h) for frequently changing data

  2. IS THERE INVALIDATION ON WRITE?
     → Write to DB → does it evict/update the cache?
     → If using @Cacheable + @CacheEvict → is @CacheEvict on the write method?

  3. MULTIPLE INSTANCES?
     → Local cache (Caffeine/EhCache) per instance → Pod 1 evicts, Pod 2 doesn't
     → Distributed cache (Redis) → should be consistent IF properly invalidated

  4. RACE CONDITION (cache-aside pattern):
     T1: cache.get("user:1") → MISS
     T1: db.query("SELECT * FROM users WHERE id=1") → {name: "old"}
       T2: db.update("UPDATE users SET name='new' WHERE id=1")
       T2: cache.evict("user:1")
     T1: cache.put("user:1", {name: "old"})  ← STALE! Eviction happened before put
     → Cache now has old value, indefinitely (until next TTL or evict)
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: @Cacheable without eviction on write
@Cacheable("users")
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}

public User updateUser(Long id, UserUpdateDto dto) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(dto.getName());
    return userRepository.save(user);  // DB updated, cache still has old value!
}

// ✅ FIX: @CacheEvict on write + TTL
@Cacheable(value = "users", key = "#id")
public User getUser(Long id) {
    return userRepository.findById(id).orElseThrow();
}

@CacheEvict(value = "users", key = "#id")  // evict on update
public User updateUser(Long id, UserUpdateDto dto) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(dto.getName());
    return userRepository.save(user);
}

// ✅ FIX: TTL as safety net (even if invalidation missed)
@Bean
public CacheManager cacheManager() {
    CaffeineCacheManager manager = new CaffeineCacheManager("users");
    manager.setCaffeine(Caffeine.newBuilder()
        .expireAfterWrite(Duration.ofMinutes(5))   // max stale = 5 min
        .maximumSize(10_000));
    return manager;
}

// ✅ FIX: Write-through pattern (update cache AND DB atomically)
@CachePut(value = "users", key = "#id")  // updates cache with returned value
public User updateUser(Long id, UserUpdateDto dto) {
    User user = userRepository.findById(id).orElseThrow();
    user.setName(dto.getName());
    return userRepository.save(user);  // returned value goes into cache
}

// For multi-instance: use Redis (distributed cache) instead of local Caffeine
// application.yml:
// spring.cache.type=redis
// spring.data.redis.host=redis-cluster
// spring.cache.redis.time-to-live=300000  # 5 min TTL
```

### ⚡ Remember
- **Always set TTL** — safety net even if invalidation logic has bugs *(TTL na lagao toh cache mein purana data hamesha rehta hai)*
- **@CacheEvict on every write** or **@CachePut** (update cache with new value)
- **Multi-instance** → use distributed cache (Redis), not local-only (Caffeine)
- **Write-through** (update cache on write) > **cache-aside** (evict on write, fetch on read miss)
- Cache-aside race condition → TTL limits the staleness window

### 🔗 Cross-References
- architecture/02 → Caching Architecture Patterns (cache-aside, write-through, write-behind)

---

<a id="q15"></a>
## Q15. Your Java service suddenly throws ClassCastException in production but not in testing. Why?

### 📝 One-Liner
**Type erasure + unchecked casts** (generic type info lost at runtime), **different classpath/classloader** between test and prod, **serialization/deserialization** returning wrong type, or **cache storing mixed types** under same key.

### 🔑 Quick Answer
**(1) Generics type erasure**: `List<Order>` becomes `List` at runtime. If cache returns `List<Payment>` but code casts to `List<Order>` → no compile error (unchecked warning suppressed) → runtime ClassCastException. **(2) Serialization mismatch**: Redis/Kafka stores serialized object with old class version → deserialization produces incompatible object → cast fails. Different in test because test uses fresh cache. **(3) Classloader difference**: prod uses different classloader (OSGI, app server, Spring DevTools) → same class loaded by two classloaders = two different Class objects → `com.example.Order` ≠ `com.example.Order` from different classloaders! **(4) Cache pollution**: same cache key stores different types in different code paths → `cache.get("result")` returns unexpected type. **(5) Heap pollution**: varargs + generics → `@SafeVarargs` misused. *(Prod mein ClassCastException — type erasure, serialization mismatch, ya classloader alag hai)*

### 📖 How to Diagnose
```
Stack trace clues:

1. Type erasure:
   ClassCastException: com.example.Payment cannot be cast to com.example.Order
   at com.example.Service.getOrders(Service.java:42)
   → Check: is the method returning a generic type from a cache/deserializer?

2. Classloader:
   ClassCastException: com.example.Order cannot be cast to com.example.Order
   → SAME class name! Two different classloaders loaded it
   → Common with: Spring DevTools, WAR in Tomcat, OSGi

3. Serialization:
   ClassCastException: java.util.LinkedHashMap cannot be cast to com.example.Order
   → Jackson deserializes without type info → returns Map instead of POJO
   → Redis GenericJackson2JsonRedisSerializer vs Jackson2JsonRedisSerializer

4. Diagnosis steps:
   → Read the full stack trace (which line? what types?)
   → Check if both types are from same ClassLoader:
     objA.getClass().getClassLoader() vs objB.getClass().getClassLoader()
   → Check serializer configuration (Redis, Kafka, JSON)
   → Search for @SuppressWarnings("unchecked") in codebase ← red flag
```

### 💻 Code — Common Causes → Fixes
```java
// ❌ BUG: Generic type erasure + cache
@Cacheable("result")
public List<Order> getOrders(String key) {
    return orderRepository.findByKey(key);
}

// Another method puts a DIFFERENT type in same cache name!
@Cacheable("result")
public List<Payment> getPayments(String key) {
    return paymentRepository.findByKey(key);
}
// If both called with same key → second call returns first's cached List
// ClassCastException: Payment cannot be cast to Order

// ✅ FIX: Use distinct cache names
@Cacheable("orders")   // separate cache
public List<Order> getOrders(String key) { ... }

@Cacheable("payments")  // separate cache
public List<Payment> getPayments(String key) { ... }

// ❌ BUG: Jackson deserializing to Map instead of POJO (Redis)
// RedisTemplate with default serializer → objects stored as LinkedHashMap
Object cached = redisTemplate.opsForValue().get("order:123");
Order order = (Order) cached;  // ClassCastException: LinkedHashMap cannot be cast to Order

// ✅ FIX: Configure proper Redis serializer with type info
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory factory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(factory);
    template.setKeySerializer(new StringRedisSerializer());

    ObjectMapper mapper = new ObjectMapper();
    mapper.activateDefaultTyping(mapper.getPolymorphicTypeValidator(),
        ObjectMapper.DefaultTyping.NON_FINAL);  // stores type info with JSON

    template.setValueSerializer(new GenericJackson2JsonRedisSerializer(mapper));
    return template;
}
```

### ⚡ Remember
- **Type erasure** = generics gone at runtime → `List<Order>` is just `List` *(Runtime pe generic type info nahi hota — cast galat ho sakta hai)*
- **Same class + different classloader = ClassCastException** (same name, different Class object)
- **`LinkedHashMap cannot be cast to X`** = Jackson deserialized without type info → configure serializer
- **Search for `@SuppressWarnings("unchecked")`** — each one is a potential ClassCastException
- **Separate cache names** per return type — never reuse cache name for different types

---

<a id="q16"></a>
## Q16. Your application throws IllegalStateException under heavy traffic. What situation might cause this?

### 📝 One-Liner
**Concurrent access to a stateful object not designed for multi-threading** — reusing a closed/exhausted connection, using a non-thread-safe object (like `SimpleDateFormat`) across threads, or calling an operation in an invalid lifecycle state (e.g., writing to a closed stream).

### 🔑 Quick Answer
**(1) Connection pool exhausted + reuse of closed connection**: thread gets a connection from pool, connection times out or is reclaimed by pool while thread holds a reference → thread uses stale connection → IllegalStateException: "Connection is closed." **(2) Non-thread-safe objects shared**: `SimpleDateFormat`, `StringBuilder`, `EntityManager` used concurrently → internal state corrupted → IllegalStateException. **(3) Stream used twice**: Java Streams are single-use → calling terminal operation twice → `IllegalStateException: stream has already been operated upon`. **(4) Servlet response already committed**: filter/controller writes to response, then another filter tries to redirect → `IllegalStateException: Response already committed`. **(5) Incorrect lifecycle**: thread calls `start()` on an already-started thread → IllegalStateException. *(Heavy traffic pe IllegalStateException — shared stateful object concurrent access se corrupt ho gaya)*

### 📖 How to Diagnose
```
Stack trace clues:

1. "Connection is closed" / "Session is closed":
   → JPA EntityManager or DB connection reused after pool reclaimed it
   → OSIV (Open Session In View) disabled but lazy load in controller

2. "stream has already been operated upon":
   → Stored a Stream in a variable and tried to use it twice
   → Streams are single-use!

3. "Response already committed":
   → Two filters or handler + filter both write to HttpServletResponse
   → Check filter order and early response writes

4. SimpleDateFormat corruption:
   → Intermittent parse errors or wrong dates
   → SimpleDateFormat is NOT thread-safe (mutable internal Calendar)

5. Diagnosis:
   → Read FULL stack trace — which object is in illegal state?
   → Is that object shared across threads (singleton bean field)?
   → Is there a lifecycle issue (used after close)?
```

### 💻 Code — Common Causes → Fixes
```java
// ❌ BUG: SimpleDateFormat shared (not thread-safe)
@Service
public class DateService {
    private final SimpleDateFormat sdf = new SimpleDateFormat("yyyy-MM-dd"); // SHARED!

    public Date parse(String dateStr) throws ParseException {
        return sdf.parse(dateStr); // concurrent access → corrupted internal state
    }
}

// ✅ FIX: Use DateTimeFormatter (immutable, thread-safe)
@Service
public class DateService {
    private static final DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy-MM-dd");

    public LocalDate parse(String dateStr) {
        return LocalDate.parse(dateStr, formatter); // thread-safe ✅
    }
}

// ❌ BUG: Reusing a Stream
Stream<Order> orderStream = orders.stream().filter(o -> o.isActive());
long count = orderStream.count();        // terminal operation — stream consumed
List<Order> list = orderStream.toList(); // IllegalStateException: already operated upon!

// ✅ FIX: Create new stream each time
long count = orders.stream().filter(Order::isActive).count();
List<Order> list = orders.stream().filter(Order::isActive).toList();

// ❌ BUG: Lazy loading outside transaction (Session closed)
@GetMapping("/users/{id}")
public UserDto getUser(@PathVariable Long id) {
    User user = userService.findById(id); // transaction ends here
    user.getOrders().size(); // LazyInitializationException or IllegalStateException
    // Session already closed!
}

// ✅ FIX: Fetch eagerly in service or use DTO projection
@Transactional(readOnly = true)
public UserDto findById(Long id) {
    User user = userRepository.findById(id).orElseThrow();
    return new UserDto(user.getName(), user.getOrders().size()); // loaded within tx
}
```

### ⚡ Remember
- **IllegalStateException = object used in wrong state** (closed, consumed, corrupted)
- **SimpleDateFormat** is NOT thread-safe → use `DateTimeFormatter` *(SimpleDateFormat = thread-safe nahi — DateTimeFormatter use karo)*
- **Streams are single-use** → never store in a variable and reuse
- **Session/Connection closed** → check transaction boundaries and OSIV setting
- Under load, concurrent access to shared mutable objects triggers state corruption

### 🔗 Cross-References
- core/02 → Streams (single-use, lazy evaluation)
- database/01 → JPA transactions, lazy loading, OSIV
