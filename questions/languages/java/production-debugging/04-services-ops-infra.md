# 🔥 Production Debugging — Services, Ops & Infrastructure (Q17–Q20)

> **"Production just broke. Let's see how you think."**
> Symptoms → Hypothesis → Diagnosis → Fix

---

<a id="q17"></a>
## Q17. A microservice call blocks your main request threads for a long time. What should be investigated?

### 📝 One-Liner
**Missing timeouts on the HTTP client** (default = INFINITE wait!) and **no circuit breaker** — when the downstream service is slow/down, your threads hang forever waiting for a response, exhausting your thread pool and cascading the failure to your service.

### 🔑 Quick Answer
**(1) No connect/read timeout**: `RestTemplate` and `WebClient` have **no default timeout** — a request to a dead service will wait forever, holding the Tomcat thread. 200 threads × infinite wait = entire service frozen. **(2) No circuit breaker**: even after detecting the downstream is slow, the code keeps calling it → every new request gets stuck → cascade. Circuit breaker (Resilience4j) opens after N failures → returns fallback immediately → threads freed. **(3) No bulk-heading**: all downstream calls use the same thread pool → one slow service starves all other services. Bulkhead isolates: 20 threads for payment, 20 for inventory → slow payment doesn't affect inventory. **(4) Synchronous chain**: A → B → C → D. If D slows down, C's threads hold, then B's, then A's → entire chain frozen. *(Downstream service slow hai aur timeout set nahi hai — thread hang forever karega)*

### 📖 How to Diagnose
```
Symptoms:
  → API latency suddenly 10-30 seconds
  → Thread dump shows most threads WAITING on socket.read()
  → Error logs show: "Read timed out" (if timeout IS set)
  → Or NO errors, just slow (if timeout NOT set — threads hang silently!)

Diagnosis:
  1. THREAD DUMP:
     $ jstack <pid> | grep -A 5 "RUNNABLE\|TIMED_WAITING"
     → Many threads in "java.net.SocketInputStream.read()" → waiting on downstream
     → Which service? Check the URL in the call stack

  2. CHECK TIMEOUT CONFIGURATION:
     → RestTemplate: is HttpComponentsClientHttpRequestFactory configured?
     → WebClient: is .timeout() or reactor.netty.http.client.HttpClient configured?
     → If no timeout → THAT'S THE BUG

  3. CHECK CIRCUIT BREAKER:
     → Is Resilience4j/Hystrix on the downstream call?
     → Actuator: /actuator/circuitbreakers → state (CLOSED/OPEN/HALF_OPEN)

Cascade failure:
  Client → [Your Service] → [Payment Service (slow: 30s)]
                ↑
  Thread hangs 30s per request → 200 threads hang → YOUR service is now slow
  → Client retries → 2x the load → faster death spiral
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: RestTemplate with NO timeouts (default = infinite)
@Bean
public RestTemplate restTemplate() {
    return new RestTemplate(); // connect timeout: ∞, read timeout: ∞
}

// ✅ FIX: Explicit timeouts + circuit breaker + bulkhead
@Bean
public RestTemplate restTemplate() {
    var factory = new HttpComponentsClientHttpRequestFactory();
    factory.setConnectTimeout(2000);   // 2s to connect
    factory.setReadTimeout(5000);      // 5s to read response
    return new RestTemplate(factory);
}

// ✅ Circuit Breaker (Resilience4j)
// application.yml:
resilience4j:
  circuitbreaker:
    instances:
      paymentService:
        sliding-window-size: 10
        failure-rate-threshold: 50      # open after 50% failures in window
        wait-duration-in-open-state: 30s  # try again after 30s
        permitted-number-of-calls-in-half-open-state: 3
  
  timelimiter:
    instances:
      paymentService:
        timeout-duration: 3s            # hard timeout for any call

  bulkhead:
    instances:
      paymentService:
        max-concurrent-calls: 20        # max 20 threads for payment calls
        max-wait-duration: 500ms        # wait max 500ms for a slot

@CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
@Bulkhead(name = "paymentService")
@TimeLimiter(name = "paymentService")
public CompletableFuture<PaymentResponse> callPayment(PaymentRequest req) {
    return CompletableFuture.supplyAsync(() ->
        restTemplate.postForObject(paymentUrl, req, PaymentResponse.class));
}

public CompletableFuture<PaymentResponse> paymentFallback(
        PaymentRequest req, Throwable t) {
    log.warn("Payment service unavailable, queuing for retry: {}", t.getMessage());
    return CompletableFuture.completedFuture(PaymentResponse.pendingRetry());
}
```

### ⚡ Remember
- **RestTemplate / WebClient default timeout = INFINITE** → always configure explicitly *(Default timeout infinite hai — set nahi kiya toh thread kabhi nahi chhootega)*
- **Circuit breaker** = stop calling failed service after threshold → return fallback
- **Bulkhead** = isolate thread pools per downstream → one slow service can't starve others
- **TimeLimiter** = hard timeout even if HTTP client timeout fails
- Defence layers: timeout → retry (with backoff) → circuit breaker → fallback

### 🔗 Cross-References
- architecture/01 → Circuit Breakers (Resilience4j patterns)
- Q12 (03-api-state-design) → REST crashes under load (timeout cascade)

---

<a id="q18"></a>
## Q18. Your application logs suddenly increase disk I/O and slow down the service. Why?

### 📝 One-Liner
**Synchronous logging on the request thread** — every `log.info()` calls `write()` to disk, and under high traffic the file I/O blocks request threads. Or a debug log level was accidentally enabled in production, generating 100x the log volume.

### 🔑 Quick Answer
**(1) Synchronous file appender**: default Logback `FileAppender` writes to disk on the calling thread. Under 1000 req/s with 5 log lines each = 5000 disk writes/sec → disk I/O saturates → request threads block on `write()` → latency spikes. **(2) Debug/Trace log level enabled**: someone set `logging.level.root=DEBUG` in production → 100x log volume → disk full + I/O thrashing. **(3) Logging large objects**: `log.info("Request: {}", requestBody)` where requestBody is a 10MB JSON → serialized via `toString()` → massive disk writes + CPU for serialization. **(4) Log file rotation failed**: log file grew to 50GB, single file on slow disk → seek time increases. **(5) Console appender in Docker**: stdout goes to Docker logging driver → if driver is slow (fluentd, json-file with no rotation) → backpressure to application. *(Logging synchronous hai toh request thread disk write pe block hota hai — async appender lagao)*

### 📖 How to Diagnose
```
Symptoms:
  → Latency spikes correlated with log volume
  → disk I/O utilization near 100% (iostat/Grafana)
  → Thread dump shows threads in java.io.FileOutputStream.write()
  → Disk usage growing rapidly (df -h)

Diagnosis:
  1. CHECK LOG LEVEL:
     → /actuator/loggers → is root level DEBUG? → set to INFO/WARN
     → Per-package: is Hibernate SQL logging on? (org.hibernate.SQL=DEBUG)

  2. CHECK APPENDER TYPE:
     → logback.xml: <appender class="ch.qos.logback.core.FileAppender"> → SYNC ❌
     → Should be: <appender class="ch.qos.logback.classic.AsyncAppender"> → ASYNC ✅

  3. CHECK LOG SIZE:
     → ls -lh /var/log/app/ → is any file >1GB? → rotation broken
     → du -sh /var/log/ → total log storage

  4. CHECK what's being logged:
     → grep for large toString() or payload logging
     → log.debug("Full response: {}", response) ← could be 10MB!
```

### 💻 Code — Bug → Fix
```xml
<!-- ❌ BUG: Synchronous file appender (default) -->
<appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>/var/log/app/application.log</file>
    <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
        <fileNamePattern>/var/log/app/application-%d{yyyy-MM-dd}.%i.log</fileNamePattern>
        <maxFileSize>100MB</maxFileSize>
        <maxHistory>30</maxHistory>
        <totalSizeCap>3GB</totalSizeCap>
    </rollingPolicy>
    <encoder><pattern>%d{ISO8601} [%thread] %-5level %logger{36} - %msg%n</pattern></encoder>
</appender>

<!-- ✅ FIX: Wrap in AsyncAppender (log on separate thread) -->
<appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
    <queueSize>1024</queueSize>           <!-- buffer 1024 log events -->
    <discardingThreshold>20</discardingThreshold> <!-- discard DEBUG/TRACE when 80% full -->
    <neverBlock>true</neverBlock>          <!-- NEVER block request thread for logging -->
    <appender-ref ref="FILE" />
</appender>

<root level="INFO">                       <!-- INFO not DEBUG in production! -->
    <appender-ref ref="ASYNC_FILE" />
</root>
```

```java
// ❌ BUG: Logging large objects
log.info("Processing request: {}", objectMapper.writeValueAsString(request));
// Serializes 10MB body to string on EVERY request!

// ✅ FIX: Log only identifiers, use lazy evaluation
log.info("Processing request id={}, type={}", request.getId(), request.getType());

// ✅ FIX: Guard expensive logging with level check
if (log.isDebugEnabled()) {
    log.debug("Full request: {}", objectMapper.writeValueAsString(request));
}

// ✅ FIX: Use Lombok @Slf4j lazy toString (parameterized message)
log.debug("Request: {}", request); // toString() called ONLY if DEBUG is enabled!
// → Logback skips toString() if level is above DEBUG
```

### ⚡ Remember
- **AsyncAppender** = logs on separate thread → never blocks request thread *(Async appender lagao — request thread disk write pe block nahi hoga)*
- **neverBlock=true** = drop log events rather than block application
- **Production = INFO level** (not DEBUG!) → change via `/actuator/loggers` at runtime
- **Never log full payloads** — log IDs and metadata only
- **Log rotation**: max 100MB per file, 3GB total, 30 days retention
- Docker: configure logging driver rotation (`--log-opt max-size=100m max-file=3`)

---

<a id="q19"></a>
## Q19. Your service sometimes processes the same request twice. What design issue might cause this?

### 📝 One-Liner
**Missing idempotency** — network retries (client timeout → retry), at-least-once message delivery (Kafka/SQS), or load balancer retry all cause duplicate requests. Without an **idempotency key** mechanism, each duplicate is treated as a new request.

### 🔑 Quick Answer
**(1) Client retry on timeout**: client sends request → server processes it → response lost (network) → client retries → server processes AGAIN. **(2) At-least-once messaging**: Kafka consumer processes message → crashes before committing offset → rebalance → message redelivered → processed again. **(3) Load balancer retry**: request to Pod 1 times out → LB retries to Pod 2 → both pods process it. **(4) Browser double-submit**: user clicks "Pay" button twice before response loads. **Fix**: every mutating operation needs an **idempotency key** — a unique identifier (UUID) sent with the request → server checks "did I already process this key?" → if yes, return cached response. *(Duplicate request = network retry ya message redelivery — idempotency key se rokho)*

### 📖 How to Diagnose
```
Symptoms:
  → Customer charged twice
  → Duplicate email/notification sent
  → Inventory decremented twice for one order
  → Happens intermittently, more often under high latency

Diagnosis:
  1. CHECK for retries:
     → Client code: is there a retry interceptor? (Spring Retry, Resilience4j)
     → Load balancer: is retry configured on 503/timeout?
     → Kafka consumer: auto.offset.commit vs manual commit?

  2. LOOK for duplicate processing evidence:
     → Same orderId with two different transaction IDs
     → Two audit log entries for same operation, seconds apart
     → Duplicate DB inserts (if no unique constraint on business key)

  3. TIMELINE:
     t=0.0s: Client sends POST /payments (requestId=abc-123)
     t=2.0s: Server starts processing
     t=3.0s: Client timeout (3s) → client retries POST /payments (requestId=abc-456)
     t=3.5s: Server finishes first request → response to nobody (client already retried)
     t=5.5s: Server finishes second request → duplicate payment!
```

### 💻 Code — Implementing Idempotency
```java
// ✅ Idempotency Key Pattern
@RestController
public class PaymentController {

    @PostMapping("/payments")
    public ResponseEntity<PaymentResponse> createPayment(
            @RequestHeader("Idempotency-Key") String idempotencyKey,
            @RequestBody PaymentRequest request) {

        // 1. Check if already processed
        Optional<PaymentResponse> existing = idempotencyStore.get(idempotencyKey);
        if (existing.isPresent()) {
            return ResponseEntity.ok(existing.get()); // return cached response
        }

        // 2. Process the payment
        PaymentResponse response = paymentService.process(request);

        // 3. Store result keyed by idempotency key (with TTL)
        idempotencyStore.put(idempotencyKey, response, Duration.ofHours(24));

        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
}

// Idempotency store (Redis-backed)
@Service
public class IdempotencyStore {
    @Autowired private StringRedisTemplate redis;
    @Autowired private ObjectMapper mapper;

    public Optional<PaymentResponse> get(String key) {
        String json = redis.opsForValue().get("idempotency:" + key);
        if (json == null) return Optional.empty();
        return Optional.of(mapper.readValue(json, PaymentResponse.class));
    }

    public void put(String key, PaymentResponse response, Duration ttl) {
        redis.opsForValue().set("idempotency:" + key,
            mapper.writeValueAsString(response), ttl);
    }
}

// ✅ Kafka idempotent consumer
@KafkaListener(topics = "order-events")
public void handleOrderEvent(ConsumerRecord<String, OrderEvent> record) {
    String eventId = record.key(); // unique event ID
    
    if (processedEventRepository.existsByEventId(eventId)) {
        log.info("Already processed event {}, skipping", eventId);
        return; // idempotent — skip duplicate
    }

    orderService.process(record.value());
    processedEventRepository.save(new ProcessedEvent(eventId, Instant.now()));
}

// ✅ Database-level idempotency (UPSERT)
@Modifying
@Query(value = "INSERT INTO payments (idempotency_key, amount, status) " +
    "VALUES (:key, :amount, 'COMPLETED') " +
    "ON CONFLICT (idempotency_key) DO NOTHING", nativeQuery = true)
int createPaymentIdempotent(@Param("key") String key, @Param("amount") BigDecimal amount);
// Returns 0 if already exists → no duplicate
```

### ⚡ Remember
- **Idempotency key** = UUID in request header → server deduplicates *(Har mutating request ke saath unique key bhejo — duplicate check karo)*
- **At-least-once** is the default for most systems → design for it
- **Three levels of protection**: (1) idempotency key check (2) DB unique constraint (3) UPSERT
- **Kafka**: manual offset commit AFTER processing, store processed event IDs
- **Client**: generate `Idempotency-Key: <UUID>` in the frontend/client SDK

### 🔗 Cross-References
- spring-batch/01 Q19 → Idempotency in Spring Batch
- spring-batch/15 → Three-layer duplicate prevention

---

<a id="q20"></a>
## Q20. A Java service deployed on multiple instances starts behaving inconsistently. What shared-state issue could exist?

### 📝 One-Liner
**Local in-memory state treated as global** — one instance's local cache, in-memory sessions, static variables, or local file storage are invisible to other instances, causing each pod to see a different "truth."

### 🔑 Quick Answer
**(1) Local cache inconsistency**: Pod 1 caches user data, Pod 2 hasn't cached it yet. User updates profile via Pod 1 (cache updated), next request goes to Pod 2 (stale cache or cache miss → old DB data if replication lag). **(2) In-memory session**: `HttpSession` stored in Tomcat's memory → user request goes to Pod 1 (session exists), next request goes to Pod 2 (no session → forced to re-login). **(3) Static variables / singletons**: `static Map<String, Config> config` loaded at startup → Pod 1 refreshes config, Pod 2 still has old config. **(4) Local file system**: writing to `/tmp/report.pdf` → only on Pod 1. Subsequent request to Pod 2 → file not found. **(5) Scheduled task inconsistency**: `@Scheduled` job runs on all instances but modifies shared DB without coordination → conflicts. *(Multiple pods — sab ka local state alag hai — distributed state store use karo)*

### 📖 How to Diagnose
```
Symptoms:
  → User logs in, then gets "logged out" on next click
  → Feature toggle shows different values on different page loads
  → Report generated on one request, "not found" on next request
  → Intermittent — depends on which pod handles the request

Diagnosis:
  1. IDENTIFY local state in the application:
     → Search for: static fields, @Scope("singleton") caches, HttpSession
     → Search for: local file writes (new File(), Files.write())
     → Search for: in-memory caches without distributed backend
  
  2. VERIFY by calling specific pods:
     → curl http://pod1:8080/api/users/1 → returns updated data
     → curl http://pod2:8080/api/users/1 → returns old data
     → Different response = local state problem

  3. MULTI-INSTANCE CHECKLIST:
     □ Session storage → Redis/DB (not local Tomcat)
     □ Cache → Redis/Hazelcast (not local Caffeine alone)
     □ File storage → S3/shared volume (not local /tmp)
     □ Config → ConfigMap/Config Server (not static variable)
     □ @Scheduled → ShedLock or Quartz cluster (not per-instance)
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: In-memory session (lost when request goes to different pod)
// Default Spring Boot: session in Tomcat memory → per-pod

// ✅ FIX: Redis session store (shared across all pods)
// pom.xml: spring-session-data-redis
// application.yml:
spring:
  session:
    store-type: redis
    redis:
      flush-mode: on-save
      namespace: myapp:session

// ❌ BUG: Local Caffeine cache — each pod has its own cache
@Bean
public CacheManager cacheManager() {
    return new CaffeineCacheManager("users"); // LOCAL to this pod!
}

// ✅ FIX: Redis cache (shared) with local near-cache as L1
@Bean
public CacheManager cacheManager(RedisConnectionFactory factory) {
    RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
        .entryTtl(Duration.ofMinutes(10))
        .serializeValuesWith(SerializationPair.fromSerializer(
            new GenericJackson2JsonRedisSerializer()));
    return RedisCacheManager.builder(factory).cacheDefaults(config).build();
}

// ❌ BUG: Local file storage
@PostMapping("/upload")
public String upload(@RequestParam MultipartFile file) {
    File dest = new File("/tmp/uploads/" + file.getOriginalFilename());
    file.transferTo(dest); // stored on THIS pod only!
    return dest.getAbsolutePath();
}

// ✅ FIX: S3 / shared storage
@PostMapping("/upload")
public String upload(@RequestParam MultipartFile file) {
    String key = "uploads/" + UUID.randomUUID() + "/" + file.getOriginalFilename();
    s3Client.putObject(PutObjectRequest.builder()
        .bucket("myapp-uploads").key(key).build(),
        RequestBody.fromInputStream(file.getInputStream(), file.getSize()));
    return key; // accessible from any pod
}

// ❌ BUG: Feature flag in static variable
public class FeatureFlags {
    private static boolean newCheckoutEnabled = false; // different per pod!
    public static void enable() { newCheckoutEnabled = true; } // only this pod
}

// ✅ FIX: Centralized config (Config Server, Redis, DB)
@Service
public class FeatureFlags {
    @Autowired private StringRedisTemplate redis;
    
    public boolean isEnabled(String flag) {
        return "true".equals(redis.opsForValue().get("feature:" + flag));
    }
}
```

### ⚡ Remember
- **Local = per-pod; Distributed = shared** — decide for each piece of state
- **Session → Redis** (`spring-session-data-redis`) *(Session Redis mein rakho — nahi toh pod badle pe logout ho jaayega)*
- **Cache → Redis** (or Hazelcast for IMDG) — not just local Caffeine
- **Files → S3 / shared volume** — not local /tmp
- **Config → Config Server / Redis** — not static variables
- **@Scheduled → ShedLock** — not per-pod execution
- **12-Factor App rule #6**: processes are **stateless** — store state in backing services

### 🔗 Cross-References
- Q13 (03-api-state-design) → Background job runs twice (multi-instance @Scheduled)
- Q14 (03-api-state-design) → Cache staleness (multi-instance cache)
- architecture/04 → In-Memory Data Grids (Hazelcast distributed state)
