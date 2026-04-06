# 🛠️ Backend Scenario-Based Questions — Production Debugging

> Real-world scenario questions asked at PhonePe, Uber, and senior Java backend interviews (30+ LPA roles).

---

## Q1. Works in Staging, Fails in Production

### 📝 One-Liner
Application works perfectly in staging but randomly fails under production traffic — diagnose and fix.

### 🔑 Quick Answer
Likely causes: thread pool exhaustion, connection pool saturation, GC pauses, network timeouts, or load balancer misconfiguration. Systematically check resource limits and concurrency behavior under load. *(staging mein load kam hai isliye chalta hai, production mein resources exhaust ho jaate hain)*

### 📖 How It Works

**Diagnosis Checklist**:
1. **Thread Pool Exhaustion**: All threads busy → new requests queue → timeouts
   - Check: `jstack <pid>` → look for WAITING/BLOCKED threads
   - Fix: Increase pool size, add async processing, fix slow downstream calls

2. **Connection Pool Saturation**: DB/HTTP connection pool drained
   - Check: HikariCP metrics (`active`, `pending`, `idle` connections)
   - Fix: Increase pool, add connection timeout, fix leaks *(connection return nahi ho raha — leak)*

3. **GC Pauses**: Long GC pauses freeze application
   - Check: GC logs (`-Xlog:gc*`), look for long STW pauses
   - Fix: Tune heap size, switch to G1/ZGC, reduce object allocation

4. **Network Timeouts**: Downstream service slow under production load
   - Check: Response time percentiles (p50, p95, p99)
   - Fix: Circuit breaker, timeouts, async calls, bulkhead pattern

5. **Load Balancer**: Sticky sessions, health check misconfiguration
   - Check: Request distribution across instances
   - Fix: Proper health endpoints, session handling

### 🗣️ Answering Approach
"I'd approach this systematically. First, check if it's resource-related: thread dumps to see blocked threads, connection pool metrics for saturation, GC logs for pauses. Then check external dependencies: are downstream services slower under load? I'd look at p99 latency — if it's spiking, it's likely a saturation issue. In my experience, the most common cause is connection pool exhaustion because staging has lower concurrency."

### 💻 Code
```java
// Health check revealing resource status
@GetMapping("/health/details")
public Map<String, Object> health() {
    Map<String, Object> status = new HashMap<>();
    // Thread pool status
    ThreadPoolTaskExecutor exec = (ThreadPoolTaskExecutor) taskExecutor;
    status.put("activeThreads", exec.getActiveCount());
    status.put("poolSize", exec.getPoolSize());
    status.put("queueSize", exec.getThreadPoolExecutor().getQueue().size());
    
    // Connection pool (HikariCP)
    HikariPoolMXBean pool = dataSource.getHikariPoolMXBean();
    status.put("dbActiveConnections", pool.getActiveConnections());
    status.put("dbIdleConnections", pool.getIdleConnections());
    status.put("dbPendingThreads", pool.getThreadsAwaitingConnection());
    return status;
}
```

### ⚡ Remember
- **Thread dump** first → reveals blocked/waiting threads
- **GC logs** → reveals STW pauses
- **Connection metrics** → reveals pool saturation
- Circuit breakers + timeouts = production must-haves
- Load test staging with production-like traffic to catch these early

---

## Q2. Database is Slow After Deployment

### 📝 One-Liner
After deploying a new feature, DB CPU hits 95% and the entire system slows down.

### 🔑 Quick Answer
Check: new queries without indexes, N+1 queries from new ORM code, lock contention from new transaction patterns, isolation level issues. Use `EXPLAIN ANALYZE` and slow query log. *(naye feature mein ya toh index missing hai, ya N+1 query hai, ya lock lag raha hai)*

### 📖 How It Works

**Diagnosis Steps**:
1. **Slow Query Log**: Enable and check what's new
   - `SET GLOBAL slow_query_log = 1; SET GLOBAL long_query_time = 1;`
   
2. **EXPLAIN ANALYZE**: Check execution plan of slow queries
   - Full table scan? → Missing index *(poori table scan ho rahi hai — index lagao)*
   - Nested loops? → N+1 query pattern
   
3. **N+1 Queries**: New JPA `@OneToMany` without `JOIN FETCH`
   - 1 query for parent + N queries for children = N+1
   - Fix: `@EntityGraph` or `JOIN FETCH` in JPQL

4. **Lock Contention**: Long transactions holding row locks
   - Check: `SELECT * FROM information_schema.INNODB_TRX`
   - Fix: Shorter transactions, optimistic locking

5. **Isolation Level**: Serializable causes excessive locking
   - Check current isolation: `SELECT @@transaction_isolation`
   - Fix: Use READ_COMMITTED for most operations

### 🗣️ Answering Approach
"I'd start with the slow query log — compare before and after deployment. Then run EXPLAIN ANALYZE on new queries to check for missing indexes or full table scans. I'd also check for N+1 queries using Hibernate statistics. In most cases I've seen, it's either a missing index on a new column being filtered, or N+1 queries from a new entity relationship."

### 💻 Code
```java
// Detect N+1: enable Hibernate statistics
spring.jpa.properties.hibernate.generate_statistics=true

// Fix N+1 with @EntityGraph
@EntityGraph(attributePaths = {"orders", "orders.items"})
List<Customer> findAllWithOrders();

// Or JPQL JOIN FETCH
@Query("SELECT c FROM Customer c JOIN FETCH c.orders WHERE c.id = :id")
Customer findWithOrders(@Param("id") Long id);
```

### ⚡ Remember
- **EXPLAIN ANALYZE** → first tool for slow queries
- **N+1** → most common ORM performance bug
- **Index** → check new WHERE/JOIN columns
- **Lock contention** → check `INNODB_TRX` and `INNODB_LOCKS`
- Rollback plan: always be ready to revert the deployment

---

## Q3. Kafka Consumers Processing Duplicate Events

### 📝 One-Liner
Kafka consumers are processing the same messages multiple times, causing business-level duplicates.

### 🔑 Quick Answer
Kafka guarantees at-least-once by default. Use idempotent consumers (dedup by message ID), manage offsets carefully, and consider transactional producers. *(Kafka mein same message dobara aa sakti hai — consumer ko idempotent banao)*

### 📖 How It Works

**Why Duplicates Happen**:
1. **Consumer Rebalance**: Consumer processes message, crashes before committing offset → reprocessed after rebalance
2. **Producer Retries**: Network timeout → producer retries → message sent twice
3. **Manual Offset Management**: Offset committed before/after processing (timing mismatch)

**Solutions**:
1. **Idempotent Consumer**: Store processed message IDs in DB/Redis
   ```
   IF messageId NOT IN processed_table → process + store messageId
   ```

2. **Transactional Producer**: `enable.idempotence=true` + `transactional.id`
   - Prevents duplicate sends from producer side

3. **Exactly-Once Semantics (EOS)**:
   - `isolation.level=read_committed` on consumer
   - Transactional producer + consumer offset commit in same transaction

4. **Dedup Table**: Business-level dedup key (orderId, paymentId)

### 💻 Code
```java
// Idempotent consumer with dedup table
@KafkaListener(topics = "payments")
@Transactional
public void consume(ConsumerRecord<String, PaymentEvent> record) {
    String messageId = record.key();
    
    // Check if already processed
    if (processedRepo.existsById(messageId)) {
        log.info("Duplicate message skipped: {}", messageId);
        return;
    }
    
    // Process the event
    paymentService.process(record.value());
    
    // Mark as processed (same transaction)
    processedRepo.save(new ProcessedMessage(messageId, Instant.now()));
}

// Producer config for idempotency
props.put(ProducerConfig.ENABLE_IDEMPOTENCE_CONFIG, true);
props.put(ProducerConfig.ACKS_CONFIG, "all");
props.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, "payment-producer-1");
```

### ⚡ Remember
- At-least-once = Kafka default → duplicates possible
- Idempotent consumer > trying to prevent duplicates at source
- Dedup key: use business ID (orderId), not Kafka offset
- Exactly-once: transactional producer + consumer in same DB txn
- DLQ: dead-letter queue for unprocessable messages

---

## Q4. Design URL Shortener + Failure Handling

### 📝 One-Liner
Design a URL shortener and explain how to handle Redis crash, DB replication lag, service downtime, and traffic spikes.

### 🔑 Quick Answer
Base design: ID generation → base62 encode → store in DB + cache in Redis. Failure handling: fallback to DB on Redis miss, async replication awareness, circuit breakers, and auto-scaling. *(basic design ke upar failure scenarios handle karo)*

### 📖 How It Works

**Base Design**: (see system-design/04 Q6 for full design)

**Failure Scenarios**:

1. **Redis Crashes**:
   - Fallback to DB for reads (higher latency but functional)
   - Redis Sentinel/Cluster for auto-failover
   - Warm cache on Redis restart from DB *(Redis down toh DB se padho, Redis restart pe cache warm karo)*

2. **DB Replication Lag**:
   - Write to primary → read from replica → newly shortened URL not found
   - Fix: Read-after-write consistency — read from primary for recent writes
   - Or: Cache the write in Redis immediately (serve from cache)

3. **Service Goes Down**:
   - Multiple instances behind load balancer
   - Health checks → LB routes away from unhealthy instance
   - Circuit breaker for downstream dependencies

4. **Traffic Spikes 10x**:
   - Auto-scaling (horizontal) for API servers
   - Redis handles read throughput easily (100K+ ops/sec)
   - Rate limiting per client
   - Pre-warm cache with popular URLs

### 🗣️ Answering Approach
"For Redis failure, I'd implement a fallback to DB reads — latency increases but the service stays functional. Redis Sentinel provides automatic failover. For replication lag, I'd use read-after-write consistency for the creating user, and rely on Redis cache for others. For traffic spikes, horizontal auto-scaling with rate limiting per client protects the system."

### ⚡ Remember
- Redis failover: Sentinel/Cluster for HA
- Replication lag: read-your-writes consistency pattern
- Circuit breaker: Resilience4j in Spring Boot
- Scale: reads >> writes → cache-heavy architecture
- Graceful degradation > hard failure

---
