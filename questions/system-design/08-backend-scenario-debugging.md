# 🏠 Backend Scenario-Based Debugging & System Design (Q1–Q14)

> **Source**: "Top 20 Java System Design" + "ONE INTERVIEW THREE LAYERS" + "Java Backend Round 1 & 2" compilations (2026)  
> **Coverage**: Production debugging scenarios, system design patterns, scalability strategies — questions that separate developers from backend engineers  
> **Level**: 3+ YOE, targeting high-paying Java backend roles  
> **Cross-refs**: Many foundational topics covered in existing files — this file focuses on NEW debugging scenarios and designs not yet in the repo

---

<a id="q1"></a>
## Q1. Your service is slow but CPU and memory are normal — what do you check first?

### 📝 One-Liner
Check thread states, I/O waits, DB connection pool, external service latency, and garbage collection pauses — the bottleneck is likely waiting, not computing.

### 🔑 Quick Answer
If CPU and memory are normal but the service is slow, the bottleneck is almost certainly **I/O or waiting**, not computation. Check: (1) **Thread dump** — are threads BLOCKED or WAITING? (2) **DB connection pool** — are threads waiting for connections? (3) **External service calls** — high latency or timeouts? (4) **GC pauses** — even with normal heap, long GC pauses cause latency spikes. (5) **Network I/O** — DNS resolution delays, serialization overhead. *(CPU aur memory normal hai lekin slow hai — toh problem I/O mein hai, DB pool mein hai, ya external calls mein wait ho raha hai)*

### 📖 How It Works (Detailed Explanation)

```
CPU normal + Memory normal + Slow response = I/O BOUND problem

Debugging steps:
1. Thread dump (jstack <pid> or Actuator /threaddump)
   → Look for BLOCKED, WAITING, TIMED_WAITING states
   → Many threads waiting on same lock? → Lock contention
   → Many threads in socket read? → External call bottleneck

2. DB connection pool (Actuator /metrics/hikaricp.*)
   → hikaricp.connections.pending > 0? → Pool exhaustion
   → hikaricp.connections.active = maximum? → All connections busy

3. Distributed tracing (Zipkin/Jaeger)
   → Which span is taking 90% of the time?
   
4. GC logs (-Xlog:gc)
   → Long STW (stop-the-world) pauses? → GC issue
   
5. Slow query log (MySQL: slow_query_log, PG: log_min_duration_statement)
   → Unindexed queries taking seconds?
```

### 🗣️ Answering Approach
"When CPU and memory are normal but the service is slow, I immediately look at thread states. I take a thread dump and check — are threads BLOCKED on locks or WAITING on I/O? Most likely they're waiting for: database connections (pool exhaustion), responses from external services, or lock contention. I check HikariCP metrics first — if pending connections are high, that's the culprit. Then I check distributed traces to find which external call is slowest. GC pauses are another hidden cause — even with normal heap, a badly tuned collector can cause 500ms+ stop-the-world pauses."

---

<a id="q2"></a>
## Q2. Increasing thread pool size didn't improve performance — what could be the reason?

### 📝 One-Liner
The bottleneck isn't thread count — it's a shared resource: database connection pool, single lock, external service rate limit, or Amdahl's Law limit on parallelizable work.

### 🔑 Quick Answer
Adding more threads only helps if threads were the bottleneck. **Reasons more threads don't help:**
1. **DB connection pool is the real bottleneck** — 200 threads but only 10 DB connections → 190 threads wait anyway
2. **Lock contention** — all threads serialize on synchronized block or row-level lock
3. **External API rate limit** — third-party allows only 100 req/sec regardless of threads
4. **Amdahl's Law** — if 80% of work is sequential (single-threaded), more threads only speed up 20%
5. **Context switching overhead** — too many threads → CPU spends time switching, not working

### 📖 How It Works (Detailed Explanation)

```
Scenario: Thread pool 50 → 200, but throughput same

The chain is only as fast as its slowest link:

  Thread 1  ─╲
  Thread 2  ──╲───→ DB Pool (10 connections) ──→ Database
  Thread 3  ──╱     ↑ BOTTLENECK!
  ...       ─╱      max 10 concurrent queries
  Thread 200 ─→ waits...

Fix: increase DB pool -- OR -- reduce DB time per query

  Thread 1  ─╲
  Thread 2  ──╲───→ synchronized block ──→ result
  Thread 3  ──╱     ↑ SERIAL!
  ...       ─╱      only 1 thread at a time

Fix: reduce critical section, use concurrent data structures
```

### 🗣️ Answering Approach
"More threads only help if threads are the bottleneck. Most often, the real bottleneck is downstream: the database connection pool is maxed out, or there's lock contention serializing all threads. I'd check: is the DB pool size proportional to the thread pool? Are threads spending time in synchronized blocks? Is there an external rate limit? I've seen cases where increasing threads from 50 to 200 actually made things worse — the context switching overhead exceeded any parallelism benefit. The fix is to identify and widen the actual bottleneck."

---

<a id="q3"></a>
## Q3. Retries between services start causing system overload — why?

### 📝 One-Liner
**Retry storm** — when a failing service gets overwhelmed by exponentially multiplying retry requests from all its callers, creating a cascading failure.

### 🔑 Quick Answer
When Service A retries 3 times, and Service B (behind A) also retries 3 times → one failed request becomes 3 × 3 = 9 requests at Service C. With 1000 concurrent failures, that's 9000 requests hitting an already struggling service. This is a **retry storm** or **retry amplification**. **Fixes**: (1) **Exponential backoff with jitter** (spread retries over time), (2) **Circuit breaker** (stop retrying when failure rate is high), (3) **Retry budget** (max 10% of requests can be retries), (4) **Limit retry layers** (only the closest caller retries, not every hop).

### 📖 How It Works (Detailed Explanation)

```
Without protection — Retry Amplification:

  1000 requests → Service A (retry 3x) → Service B (retry 3x) → Service C
  Service C fails → 1000 × 3 × 3 = 9,000 requests hitting C!
  Service C crashes harder → more retries → system collapse

With protection:
  1000 requests → Service A (circuit breaker: OPEN after 50% failures)
                → Returns fallback immediately 
                → Only 100 requests reach Service C
                → C recovers
```

**Correct retry pattern:**
```java
// ✅ Resilience4j configuration
@Retry(name = "serviceC", fallbackMethod = "fallback")
@CircuitBreaker(name = "serviceC", fallbackMethod = "fallback")

// application.yml
resilience4j:
  retry:
    instances:
      serviceC:
        max-attempts: 3
        wait-duration: 500ms
        enable-exponential-backoff: true  # 500 → 1000 → 2000ms
        randomized-wait: true            # jitter prevents thundering herd
  circuitbreaker:
    instances:
      serviceC:
        failure-rate-threshold: 50       # open at 50% failures
        wait-duration-in-open-state: 10s
```

### 🗣️ Answering Approach
"This is called a retry storm. When multiple layers of services all retry independently, the failed service gets exponentially more requests. A single request failing can become 9 or 27 requests downstream. The fix is three-fold: use exponential backoff with jitter to spread retries over time, add a circuit breaker to stop retrying when the failure rate is high, and implement a retry budget — only 10% of total requests can be retries. I also limit which layer retries — typically only the immediate caller should retry, not every hop."

---

<a id="q4"></a>
## Q4. Circuit breaker remains open even when service is healthy — why?

### 📝 One-Liner
The half-open state isn't being triggered (check wait duration), health check endpoint isn't what the circuit breaker monitors, or the sliding window still contains old failures.

### 🔑 Quick Answer
**Common causes:**
1. **Wait duration too long** — `waitDurationInOpenState` set to 60s but you're checking at 30s
2. **Sliding window not reset** — count-based window still has old failures from before recovery
3. **Health != circuit breaker** — circuit breaker checks actual request failures, not /health endpoint
4. **Slow start after half-open** — only `permittedNumberOfCallsInHalfOpenState` calls are allowed, and they may fail if the service is slow to warm up
5. **Different failure types** — circuit breaker counts timeouts as failures, and the "healthy" service may still be slow

### 📖 How It Works (Detailed Explanation)

```
Circuit Breaker States:

CLOSED ──(failure rate > threshold)──→ OPEN
   ↑                                      │
   │                                      │ waits: waitDurationInOpenState
   │                                      ↓
   └──(success rate > threshold)──── HALF_OPEN
                                     ↑
                                  permits N calls
                                  → all succeed? → CLOSED
                                  → any fail? → back to OPEN
```

**Debugging:**
```yaml
# ✅ Check current state via Actuator
GET /actuator/circuitbreakerevents

# ✅ Tune for faster recovery
resilience4j:
  circuitbreaker:
    instances:
      serviceX:
        wait-duration-in-open-state: 5s     # shorter recovery check
        permitted-calls-in-half-open: 5      # more probe calls
        sliding-window-size: 10              # smaller window = faster reset
        failure-rate-threshold: 50
```

### 🗣️ Answering Approach
"First I check the wait-duration-in-open-state — if it's 60 seconds, the breaker won't try the service until that time passes, even if the service recovered in 10 seconds. Second, I check the half-open configuration — permitted-calls-in-half-open determines how many probe requests are sent. If those probe requests happen to fail (maybe the service is still warming up), the breaker goes right back to OPEN. Third, I verify what counts as 'failure' — slow responses exceeding the timeout count as failures even if the service technically returned a response."

---

<a id="q5"></a>
## Q5. Adding more instances didn't improve performance — what might be the bottleneck?

### 📝 One-Liner
The bottleneck is a shared resource that doesn't scale horizontally: database, distributed lock, single-partition Kafka topic, or stateful service with session affinity.

### 🔑 Quick Answer
Horizontal scaling fails when the bottleneck is **not the application layer**:
1. **Database** — 10 app instances but 1 DB → DB is the bottleneck (connection saturation, lock contention)
2. **Single Kafka partition** — consumers can't parallelize beyond partition count
3. **Distributed lock** — all instances compete for the same lock → serialized
4. **Shared cache** — single Redis instance saturated
5. **Hot key problem** — all requests hitting the same DB row/cache key

### 📖 How It Works (Detailed Explanation)

```
Horizontal scaling only works when your bottleneck scales too:

✅ Scales:    App instances (stateless)
❌ Doesn't:   Database (vertical bottleneck)
❌ Doesn't:   Kafka partition (consumer limit = partition count)
❌ Doesn't:   Distributed lock (serialized by design)

  Instance 1 ──╲
  Instance 2 ──── Database (1 instance, maxed out) ← BOTTLENECK
  Instance 3 ──╱

Fix options:
  1. Read replicas (scale reads)
  2. Database sharding (scale writes)
  3. Caching (reduce DB load)
  4. Async processing (decouple with queues)
```

### 🗣️ Answering Approach
"When adding instances doesn't help, the bottleneck is a shared resource that doesn't scale with the application tier. Most commonly it's the database — all instances hit the same DB, which becomes saturated with connections and locks. I check: is DB CPU at 100%? Are connections maxed? Is there lock contention on hot rows? Fixes depend on the bottleneck: read replicas for read-heavy, sharding for write-heavy, caching to reduce DB load, or async processing to decouple request handling from heavy operations."

---

<a id="q6"></a>
## Q6. Cache improved speed but caused inconsistent data — why?

### 📝 One-Liner
**Cache invalidation failure** — cache serves stale data because it wasn't updated/evicted when the source data changed.

### 🔑 Quick Answer
Classic "There are only two hard things in CS: cache invalidation and naming things." **Common causes:**
1. **Write-behind cache** — data updated in DB but cache not invalidated
2. **TTL too long** — cache holds stale data for hours
3. **Multi-instance cache inconsistency** — Instance A updates DB + local cache, Instance B still has old cached value
4. **Race condition** — read-from-cache and write-to-DB happen concurrently → stale read wins
5. **Cache-aside without invalidation** — populate cache on miss, but never evict on update

### 📖 How It Works (Detailed Explanation)

**Consistency patterns:**

| Pattern | Consistency | Performance | Use When |
|---------|-------------|-------------|----------|
| **Cache-aside** | Eventual | Best read perf | Read-heavy, tolerance for brief staleness |
| **Write-through** | Strong | Slower writes | Consistency is critical |
| **Write-behind** | Eventual | Best write perf | Analytics, non-critical data |
| **Read-through** | Eventual | Good | Simple caching needs |

**Fix for consistency:**
```java
// ✅ Cache-aside with explicit invalidation
@Cacheable("products")
public Product getProduct(Long id) { return repo.findById(id).orElseThrow(); }

@CacheEvict(value = "products", key = "#product.id")
@Transactional
public Product updateProduct(Product product) { return repo.save(product); }

// ✅ Or use short TTL + eventual consistency
@Cacheable(value = "products", cacheManager = "shortTtlCacheManager") // 30s TTL
```

### 🗣️ Answering Approach
"The root cause is cache invalidation — when data is updated in the database, the corresponding cache entry isn't evicted or updated. In a multi-instance setup, Instance A may update the DB and invalidate its local cache, but Instance B still serves the old cached value. The fix depends on consistency requirements: for strong consistency, use @CacheEvict on every write operation and centralized Redis cache (not local). For eventual consistency, use short TTLs. I always use @CacheEvict on update/delete operations and prefer centralized Redis over local caches in multi-instance deployments."

---

<a id="q7"></a>
## Q7. System works in testing but fails under real traffic — why?

### 📝 One-Liner
Testing doesn't replicate production: data volume, concurrency, network latency, third-party behavior, edge-case user input, and infrastructure configuration differences.

### 🔑 Quick Answer
**Common gaps between testing and production:**
1. **Concurrency** — tests run sequentially; production has 1000 concurrent users → race conditions, deadlocks
2. **Data volume** — test DB has 100 rows; production has 10M → full table scans, OOM
3. **Third-party behavior** — test uses mocks/sandbox that always succeed; production APIs rate-limit, timeout, return errors
4. **Infrastructure** — different JVM settings, DNS, load balancer config, TLS overhead
5. **Edge-case inputs** — test data is clean; real users send Unicode, emojis, null fields, SQL injection
6. **Network** — test is localhost; production crosses AZs with 5-20ms latency per hop

### 📖 How It Works (Detailed Explanation)

| Testing Gap | Why It Passes Tests | Why It Fails in Prod |
|-------------|--------------------|--------------------|
| Race condition | Tests are sequential | Concurrent requests cause dirty reads |
| Connection pool | Single test thread | 200 concurrent threads exhaust 10-connection pool |
| Memory leak | Tests are short-lived | Production runs for days → gradual OOM |
| Timeout | Mock returns instantly | Real API takes 5s under load |
| GC pressure | Small data sets | Large heap → long GC pauses |

**Prevention:**
- Load testing before release (JMeter, Gatling at 2× expected peak)
- Chaos engineering (kill services, inject latency)
- Staging environment that mirrors production config
- Integration tests with real external services (not just mocks)

### 🗣️ Answering Approach
"The core issue is that testing doesn't replicate production conditions. The biggest gaps I've seen: concurrency — race conditions only appear under load; data volume — queries that work on 100 rows fail on 10 million; and third-party APIs — mocks always succeed but real APIs rate-limit and timeout. My prevention strategy: mandatory load testing at 2× expected peak before every release, a staging environment that mirrors production infrastructure, and integration tests with real external services for critical paths."

---

<a id="q8"></a>
## Q8. Your service's p99 latency spiked from 80ms to 2s overnight — walk through your debugging process.

> 🔥 **Frequently asked** — tests real production debugging methodology

### 📝 One-Liner
Check recent deployments → APM/traces for slow spans → DB slow query log → GC logs → external dependency health → infrastructure changes.

### 🔑 Quick Answer
**Systematic debugging (in order):**
1. **Correlation**: What changed overnight? Deployments? Config changes? Traffic spike?
2. **APM/Traces**: Distributed tracing (Zipkin/Jaeger) → which span went from 50ms to 1.8s?
3. **Database**: Slow query log → new query without index? Table grew past threshold?
4. **GC**: Full GC pauses from heap pressure → `-Xlog:gc` → significant STW pauses?
5. **External services**: Dependency health → did a downstream service degrade?
6. **Infrastructure**: New pod scheduling on weaker node? DNS resolution delays? Disk full?
7. **Lock contention**: Thread dump → threads BLOCKED on same monitor?

### 📖 How It Works (Detailed Explanation)

```
Debugging Timeline:
T+0  Alert: p99 > 2s threshold
T+5  Check: any deployment last 12h? → Yes: rolled back feature X 
     → Still slow? If yes, not the deploy. Continue investigation.
T+10 APM: Sort traces by duration DESC
     → Slowest span: "SELECT * FROM orders WHERE user_id = ?"
     → Was 5ms, now 800ms
T+15 DB: EXPLAIN ANALYZE the query
     → Sequential scan! Index was dropped by migration script
T+20 Fix: CREATE INDEX idx_orders_user_id ON orders(user_id)
T+25 Verify: p99 back to 80ms
T+30 Postmortem: Add index existence check to CI/CD pipeline
```

**If NOT deployment-related — investigate data growth:**
```sql
-- Check table growth
SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY n_live_tup DESC;
-- orders table grew from 1M to 50M rows → query plan changed from index scan to seq scan
```

### 🗣️ Answering Approach
"My debugging is systematic. First: what changed? I check deployment history and config changes from the last 12 hours. Second: I look at distributed traces — I sort by duration and find the slowest span. In my experience, 70% of latency spikes come from database queries that lost their index, or table growth causing query plan changes. I run EXPLAIN ANALYZE on the slow query. If it's not DB, I check GC logs for stop-the-world pauses and external service response times. For the fix, I address the root cause and add a monitoring check to catch it earlier next time."

### ⚡ Remember
- 70% of latency spikes = DB related (missing index, data growth, lock contention)
- Always check: what changed? (deploy, data, traffic, infra)
- **EXPLAIN ANALYZE** is your best friend for DB debugging
- Cross-ref: [Performance improvement approach → hr-behavioral/02-techno-managerial-round.md Q3](../hr-behavioral/02-techno-managerial-round.md#q3)

---

<a id="q9"></a>
## Q9. Design a scalable notification system (Email + SMS + Push)

### 📝 One-Liner
Event-driven architecture: Notification Service → Message Queue (Kafka) → Channel-specific Workers (Email/SMS/Push) with retry, template engine, and preference management.

### 🔑 Quick Answer

```
User Action → Notification Service → Kafka Topic (notification-events)
                                         ├── Email Worker → SMTP/SES → Email
                                         ├── SMS Worker → Twilio/SNS → SMS  
                                         └── Push Worker → FCM/APNs → Push

Key components:
1. Template Engine — parameterized templates per channel
2. User Preferences — which channels enabled, quiet hours
3. Rate Limiter — max 5 notifications/user/hour
4. Retry Queue — failed sends retried with backoff
5. Deduplication — idempotency key prevents double-send
```

**Database schema:**
```sql
notifications (id, user_id, type, channel, template_id, params_json, 
               status, retry_count, created_at, sent_at)
user_preferences (user_id, email_enabled, sms_enabled, push_enabled, quiet_start, quiet_end)
templates (id, channel, subject_template, body_template, locale)
```

### 🗣️ Answering Approach
"I'd design this as an event-driven system. The notification service receives events and publishes them to Kafka topics partitioned by channel. Separate consumer workers for Email, SMS, and Push process messages independently. This decouples sending from the main application flow. I'd add: a template engine for personalized messages, user preference management (opt-out, quiet hours), rate limiting per user, idempotency keys to prevent duplicate sends, and a retry mechanism with exponential backoff for transient failures. For scale, Kafka handles millions of events and each channel worker scales independently."

---

<a id="q10"></a>
## Q10. Design a real-time chat application

### 📝 One-Liner
WebSocket for real-time messaging, Redis Pub/Sub for cross-instance delivery, persistent storage for message history, with presence tracking and read receipts.

### 🔑 Quick Answer

```
Architecture:
  Client ←WebSocket→ Chat Server ←Redis Pub/Sub→ Other Chat Servers
                          ↓
                     Message Store (Cassandra/MongoDB)
                          ↓
                    Media Store (S3)

Flow:
  1. User connects via WebSocket → server maps userId → WebSocket session
  2. User sends message → server publishes to Redis channel (room:{roomId})
  3. All servers subscribed to that room → push message to connected clients
  4. Message persisted asynchronously to DB
  5. Offline users → push notification via FCM/APNs
```

**Key design decisions:**
| Decision | Choice | Why |
|----------|--------|-----|
| Protocol | WebSocket | Full-duplex, low latency |
| Cross-server delivery | Redis Pub/Sub | Fast, handles server-to-server routing |
| Message storage | Cassandra | Write-heavy, time-series data, horizontal scaling |
| Media storage | S3 + CDN | Cost-effective, scalable blob storage |
| Presence | Redis SET with TTL | Heartbeat-based online/offline tracking |

### 🗣️ Answering Approach
"For real-time chat, I'd use WebSockets for bidirectional communication. Each chat server maintains a map of userId to WebSocket sessions. When a user sends a message, the server publishes it to a Redis Pub/Sub channel for the chat room. All servers subscribed to that room push the message to their connected clients. Messages are persisted asynchronously to Cassandra for history. For offline users, I queue a push notification. Presence tracking uses Redis with TTL-based heartbeats. Media shares go through S3 with pre-signed URLs."

---

<a id="q11"></a>
## Q11. Design a file upload system for large files

### 📝 One-Liner
Multipart/chunked upload to object storage (S3) with pre-signed URLs, progress tracking, resumable uploads, and virus scanning before access.

### 🔑 Quick Answer

```
Flow:
  1. Client requests upload → Server returns pre-signed S3 URL (or multipart upload ID)
  2. Client uploads directly to S3 (bypasses server — saves bandwidth)
  3. S3 triggers Lambda/webhook → Virus scan → Metadata storage
  4. File available after scan passes

For very large files (>100MB) — Multipart Upload:
  1. Client: InitiateMultipartUpload → get uploadId
  2. Client: Upload parts (5MB each) in parallel
  3. Client: CompleteMultipartUpload
  4. If network fails → resume from last successful part
```

**Key patterns:**
| Concern | Solution |
|---------|----------|
| Large files (GB+) | S3 Multipart Upload (parts in parallel) |
| Bandwidth | Pre-signed URLs (client → S3 directly, not through server) |
| Resumable | Track uploaded parts; client resumes from last part |
| Security | Virus scan before making file accessible |
| Progress | Client tracks bytes sent per chunk |
| Storage cost | Lifecycle rules: move to S3 Glacier after 30 days |

### 🗣️ Answering Approach
"I wouldn't route file data through the application server — that wastes bandwidth and memory. Instead, the server generates a pre-signed S3 URL, and the client uploads directly to S3. For large files, I use S3 Multipart Upload — the file is split into 5MB chunks uploaded in parallel, which is both faster and resumable. If the upload fails after chunk 50, the client resumes from chunk 51 instead of restarting. After upload completes, an S3 event triggers virus scanning. The file is only made accessible after the scan passes."

---

<a id="q12"></a>
## Q12. How would you implement logging and monitoring for a production system?

### 📝 One-Liner
Structured logging (JSON) → centralized aggregation (ELK/Loki) → metrics (Prometheus/Micrometer) → dashboards (Grafana) → alerts (PagerDuty/Slack).

### 🔑 Quick Answer

```
Logging Stack:
  App → SLF4J + Logback → JSON format → Filebeat/Fluent Bit → Elasticsearch → Kibana

Metrics Stack:
  App → Micrometer → Prometheus → Grafana → Alerts (Slack/PagerDuty)

Tracing Stack:
  App → Spring Sleuth → Zipkin/Jaeger → Trace visualization
```

**Structured logging (not plain text):**
```json
{
  "timestamp": "2026-04-06T10:30:00Z",
  "level": "ERROR",
  "service": "order-service",
  "traceId": "abc123",
  "userId": "user456",
  "message": "Payment failed",
  "errorCode": "PAY_001",
  "latency_ms": 2500
}
```

**Key metrics to track:**
| Category | Metrics | Alert Threshold |
|----------|---------|-----------------|
| Latency | p50, p95, p99 | p99 > 2s |
| Errors | 5xx rate, error ratio | > 1% |
| Saturation | CPU, memory, DB pool, thread pool | > 80% |
| Business | Order success rate, payment failures | < 98% |

### 🗣️ Answering Approach
"I implement the three pillars of observability: logs, metrics, and traces. For logging, I use structured JSON logs with SLF4J and ship them to ELK via Filebeat — structured logs enable powerful querying. For metrics, Micrometer with Prometheus and Grafana dashboards tracking the four golden signals: latency, traffic, errors, and saturation. For traces, Spring Sleuth with Zipkin for distributed tracing across microservices. Every log entry includes a traceId for correlation. Alerts are tiered: p99 > 2s goes to Slack, 5xx > 5% goes to PagerDuty."

### ⚡ Remember
- Cross-ref: [System health metrics → hr-behavioral/02-techno-managerial-round.md Q12](../hr-behavioral/02-techno-managerial-round.md#q12)

---

<a id="q13"></a>
## Q13. How do you handle millions of requests per second?

### 📝 One-Liner
Layer defenses: CDN → load balancer → auto-scaling app tier → caching (Redis) → read replicas → async processing (Kafka) → database sharding.

### 🔑 Quick Answer

```
Traffic: 1M req/sec

Layer 1: CDN (CloudFront/Cloudflare)
  → Serve static assets, cache API responses → Absorbs 60% of traffic

Layer 2: Load Balancer (ALB/NLB)
  → Distributes across app instances → Health checks, auto-scaling

Layer 3: App Tier (100+ instances, auto-scaled)
  → Stateless, horizontally scaled → Rate limiting per user

Layer 4: Cache (Redis Cluster)
  → Cache hot data → 90% cache hit rate → Only 10% reaches DB

Layer 5: Database (Read replicas + Sharding)
  → Reads: 5 read replicas
  → Writes: Sharded by user_id

Layer 6: Async Processing (Kafka)
  → Non-critical work (analytics, notifications) processed async
```

### 🗣️ Answering Approach
"At millions of requests per second, every layer must be optimized. CDN absorbs static content and cacheable API responses — eliminating 60% of traffic from reaching my servers. Behind the load balancer, I auto-scale stateless application instances. A Redis cache cluster handles hot data with 90%+ hit rate, so only 10% of reads reach the database. For the database, I use read replicas for reads and shard by user_id for writes. Non-critical work — analytics, notifications, audit logs — goes through Kafka for async processing. Rate limiting at the API gateway prevents any single user from overwhelming the system."

---

<a id="q14"></a>
## Q14. How do you do zero-downtime deployment for a Spring Boot service in Kubernetes?

### 📝 One-Liner
Rolling deployment with readiness/liveness probes, graceful shutdown, connection draining, and database migration compatibility (zero-downtime schema changes).

### 🔑 Quick Answer

```yaml
# ✅ Kubernetes rolling update strategy
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod during rollout
      maxUnavailable: 0   # never reduce below desired count

# ✅ Health probes
  readinessProbe:
    httpGet:
      path: /actuator/health/readiness
      port: 8080
    initialDelaySeconds: 20
    periodSeconds: 5
  livenessProbe:
    httpGet:
      path: /actuator/health/liveness
      port: 8080
    initialDelaySeconds: 30
    periodSeconds: 10
```

**Spring Boot configuration:**
```yaml
# ✅ Graceful shutdown — finish in-flight requests
server:
  shutdown: graceful
spring:
  lifecycle:
    timeout-per-shutdown-phase: 30s

# ✅ Actuator probes for K8s
management:
  endpoint:
    health:
      probes:
        enabled: true
  health:
    readinessstate:
      enabled: true
    livenessstate:
      enabled: true
```

**Database migration compatibility:**
```
Rule: Old code and new code must both work with the database schema

✅ Safe: Add column with DEFAULT → old code ignores new column
✅ Safe: Add new table → old code doesn't query it
❌ Unsafe: Rename column → old code breaks
❌ Unsafe: Drop column → old code breaks

For column rename: 
  Deploy 1: Add new column, write to both
  Deploy 2: Read from new column, stop writing old
  Deploy 3: Drop old column
```

### 🗣️ Answering Approach
"Zero-downtime deployment has three parts. First, Kubernetes rolling update with maxUnavailable=0 — new pods start before old ones terminate. Second, Spring Boot's graceful shutdown with readiness probes — when a pod receives SIGTERM, it stops accepting new requests via readiness probe turning unhealthy, finishes in-flight requests within 30 seconds, then terminates. Third, database compatibility — both old and new code versions must work with the same schema. For column changes, I use an expand-migrate-contract pattern: add the new column first, deploy code that writes to both, then remove the old column in a subsequent deployment."
