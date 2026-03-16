# 🏗️ Caching & Architecture Patterns (Q7–Q8)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q7"></a>
## Q7. What is Cache Hit and Cache Miss? How can caching with unique keys (e.g., Redis) help protect systems from heavy request traffic?

### 📝 One-Liner
**Cache Hit** = requested data found in cache (fast, no DB call); **Cache Miss** = data NOT in cache, must fetch from DB (slow, then store in cache for next time). Redis with unique keys absorbs heavy traffic by serving responses from memory instead of hitting the database.

### 🔑 Quick Answer
**Cache Hit**: client requests data → cache has it → returns immediately from memory (~1ms). No database call. **Cache Miss**: client requests data → cache doesn't have it → fetches from DB (~50-200ms) → stores result in cache → returns to client. Next request for same data = cache hit. **Pattern: Cache-Aside (Lazy Loading)**: check cache first → miss → load from DB → put in cache. With Redis using unique keys like `product:12345` or `user:session:abc`, the cache absorbs 90-95% of read traffic. During traffic spikes (flash sale, viral post), Redis handles **100K+ reads/sec** from memory while the database stays protected at its normal load. Without cache, 10K concurrent users hitting the DB directly would cause connection pool exhaustion and downtime. *(Cache Hit = data mil gaya cache mein — fast; Cache Miss = nahi mila — DB se laao aur cache mein daal do)*

### 📖 How It Works
```
Cache-Aside Pattern (most common):

  Request: GET /api/products/12345
  
  Step 1: Check Redis
    Redis.GET("product:12345")
    ├── HIT ✅ → return cached data (1ms) → DONE
    └── MISS ❌ → continue to Step 2

  Step 2: Query Database
    SELECT * FROM products WHERE id = 12345  (50ms)

  Step 3: Store in Cache
    Redis.SET("product:12345", jsonData, TTL=3600)  (1ms)

  Step 4: Return response

  Next request for product 12345 → Step 1 → HIT ✅ (1ms)

Traffic Protection Scenario:
  Flash Sale: 50,000 users requesting same product page

  WITHOUT Cache:
    50,000 requests → 50,000 DB queries → DB at 500% capacity → CRASH! 💥
  
  WITH Redis Cache:
    Request 1: MISS → DB query → cache product → 50ms
    Request 2-50,000: HIT → Redis serves from memory → 1ms each
    DB queries: 1 instead of 50,000! ✅
    
  Redis throughput: ~100,000-200,000 ops/sec (single instance)
  DB throughput: ~1,000-5,000 queries/sec
  → Redis absorbs 95%+ of load

Unique Key Strategy:
  product:12345           → product data by ID
  product:list:page:1     → paginated product list
  user:session:abc123     → user session data
  cart:user:456           → shopping cart
  rate_limit:ip:1.2.3.4  → rate limiting counter
```

### 🗣️ How to Say in Interview
"A cache hit means the requested data is found in the cache and returned immediately — typically under a millisecond from Redis. A cache miss means the data isn't cached — we fetch it from the database, store it in Redis with a TTL, and return it. The next identical request becomes a cache hit. By using unique keys like product:12345 or user:session:abc, Redis serves as a high-speed front layer absorbing the majority of read traffic. During a flash sale with 50,000 concurrent requests for the same product, only the first request hits the database. Redis handles the remaining 49,999 from memory at sub-millisecond latency, protecting the database from overload. In my project, adding Redis caching for product catalog queries reduced database load by 93% and brought p99 latency from 200ms to 3ms during peak traffic."

### 💻 Code
```java
// 1. Spring Boot + Redis Cache — @Cacheable (simplest approach)
@Service
@RequiredArgsConstructor
public class ProductService {
    private final ProductRepository productRepo;
    
    // CACHE-ASIDE: check cache → miss → DB → store in cache
    @Cacheable(value = "products", key = "#id")  // key = "products::12345"
    public ProductDTO getProduct(Long id) {
        // This method body ONLY executes on cache miss
        return productRepo.findById(id)
                .map(this::toDTO)
                .orElseThrow(() -> new ProductNotFoundException(id));
    }
    
    // CACHE EVICT: remove from cache when data changes
    @CacheEvict(value = "products", key = "#id")
    public ProductDTO updateProduct(Long id, UpdateProductRequest request) {
        Product product = productRepo.findById(id).orElseThrow();
        product.setName(request.name());
        product.setPrice(request.price());
        return toDTO(productRepo.save(product));
    }
    
    // CACHE PUT: always update cache with new value
    @CachePut(value = "products", key = "#result.id")
    public ProductDTO createProduct(CreateProductRequest request) {
        Product product = productRepo.save(toEntity(request));
        return toDTO(product);
    }
    
    // Evict entire product list cache
    @CacheEvict(value = "productList", allEntries = true)
    public void refreshProductCache() {}
}

// 2. Redis Configuration
@Configuration
@EnableCaching
public class RedisConfig {
    @Bean
    public RedisCacheManager cacheManager(RedisConnectionFactory factory) {
        RedisCacheConfiguration config = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))       // TTL = 1 hour
                .serializeValuesWith(
                    SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                .disableCachingNullValues();           // don't cache null results
        
        // Different TTL per cache
        Map<String, RedisCacheConfiguration> cacheConfigs = Map.of(
            "products", config.entryTtl(Duration.ofHours(2)),    // products: 2 hours
            "sessions", config.entryTtl(Duration.ofMinutes(30)),  // sessions: 30 min
            "productList", config.entryTtl(Duration.ofMinutes(10)) // lists: 10 min
        );
        
        return RedisCacheManager.builder(factory)
                .cacheDefaults(config)
                .withInitialCacheConfigurations(cacheConfigs)
                .build();
    }
}

// 3. Manual Redis operations (for complex caching logic)
@Service
@RequiredArgsConstructor
public class RateLimitService {
    private final StringRedisTemplate redisTemplate;
    
    public boolean isAllowed(String clientIp, int maxRequests, Duration window) {
        String key = "rate_limit:" + clientIp;
        Long count = redisTemplate.opsForValue().increment(key);
        if (count == 1) {
            redisTemplate.expire(key, window);  // set TTL on first request
        }
        return count <= maxRequests;
    }
}

// application.yml
// spring:
//   data:
//     redis:
//       host: localhost
//       port: 6379
//       timeout: 2000ms
//   cache:
//     type: redis
```

### ⚠️ Pitfalls / Gotchas
- **Cache Stampede** — cache expires, 1000 concurrent requests all miss and hit DB simultaneously. Fix: lock + single load, or stagger TTLs *(Cache expire hua aur 1000 requests ek saath DB pe aa gayi — lock lagao ek hi request DB jaaye)*
- **Stale data** — cached data can be outdated after DB update. Use @CacheEvict on writes or short TTL
- **Cache Penetration** — requesting IDs that don't exist → always miss → always hit DB. Fix: cache null/empty results with short TTL, or use Bloom filter
- **Serialization errors** — cached object class changes (add/remove fields) break deserialization. Add serialVersionUID or use JSON serialization
- **@Cacheable on self-invocation** — same AOP proxy issue as @Transactional. Internal method calls bypass cache
- **Memory limits** — Redis is in-memory; unbounded caching can cause OOM. Set maxmemory + eviction policy (allkeys-lru)

### 🆚 vs. Comparison

**Caching Strategies:**

| Strategy | Read | Write | Consistency | Use Case |
|----------|------|-------|------------|----------|
| Cache-Aside | Check cache → miss → DB → cache | Evict cache on write | Eventually consistent | Most apps ⭐ |
| Read-Through | Cache auto-loads from DB | — | Eventually consistent | Managed caches |
| Write-Through | — | Write DB + cache together | Strong | Critical data |
| Write-Behind | — | Write cache first, async DB | Weakest | High-write perf |

**Local Cache vs Redis:**

| Aspect | Local (Caffeine) | Redis (Distributed) |
|--------|------------------|-------------------|
| Speed | ~50ns (in-process) | ~1ms (network hop) |
| Shared | No (per instance) | Yes ⭐ (all instances) |
| Memory | JVM heap | Dedicated server |
| Scalability | Limited | High ⭐ |
| Use case | Rarely-changing config | Session, product data ⭐ |

### 🎯 Tricky Interview Qs

**Q: How do you handle cache warming?**
On application startup or deployment, pre-load frequently accessed data into cache. Either run a batch job that queries popular items and caches them, or use a "warm cache" endpoint triggered by CI/CD after deployment.

**Q: What is a Bloom filter and how does it prevent cache penetration?**
A probabilistic data structure that quickly tells you "definitely NOT in set" or "probably in set." Before checking cache/DB for a product ID, check the Bloom filter. If it says "not in set" → return 404 immediately without any cache/DB lookup. False positives are possible (check DB), but false negatives never happen. *(Bloom filter = definitely nahi hai ya shayad hai — DB pe faltu query nahi jaayegi)*

### ⚡ Remember
- **Cache Hit** = found in cache (fast, ~1ms) / **Cache Miss** = not found → DB → store → return
- **Cache-Aside** = check cache → miss → DB → cache *(pehle cache dekho, nahi mila toh DB se laao aur daal do)*
- **Redis** absorbs 90-95% of read traffic → protects DB from spikes
- Unique keys: `entity:id` pattern (`product:12345`, `user:session:abc`)
- Watch for: **stampede**, **penetration**, **stale data**
- @Cacheable + @CacheEvict = Spring's declarative caching (AOP-based)

### 🔗 Follow-ups
- [Q8 → Master-Slave (Redis replication for HA)](#q8)
- Q17 → API performance bottleneck (architecture/01)
- Q6 → AOP (spring/02, @Cacheable uses AOP)

---

<a id="q8"></a>
## Q8. What is the difference between Master–Slave architecture and Producer–Consumer pattern?

### 📝 One-Liner
**Master-Slave** = one master distributes tasks/data to slaves who replicate or execute (centralized control, data replication); **Producer-Consumer** = producers generate work items into a queue, consumers pull independently (decoupled, async processing via message broker).

### 🔑 Quick Answer
**Master-Slave (Leader-Follower)**: one node (master) handles writes and distributes data to multiple slaves (replicas) for reads. Used for **database replication** (MySQL master → 3 read replicas), **Redis Sentinel** (master writes, slaves read). Master controls slaves, slaves are passive copies. If master dies → a slave is promoted. **Producer-Consumer**: completely different — producers push messages to a **queue/broker** (Kafka, RabbitMQ, SQS), consumers pull and process them independently. Producers don't know consumers. Used for **async processing** (order placed → email sent), **load buffering** (absorb traffic spikes), **decoupling** (services don't call each other directly). *(Master-Slave = ek boss, baaki copies; Producer-Consumer = kaam queue mein daalo, koi bhi utha le)*

### 📖 How It Works
```
MASTER-SLAVE (Database Replication):

  ┌─────────┐     replication     ┌─────────┐
  │ MASTER  │ ──── writes ──────> │ SLAVE 1 │ ← reads
  │ (write) │ ──── writes ──────> │ SLAVE 2 │ ← reads
  │         │ ──── writes ──────> │ SLAVE 3 │ ← reads
  └─────────┘                     └─────────┘
       ↑                               ↑
   App writes here              App reads from here
   (INSERT, UPDATE)             (SELECT queries)

  - Master = single source of truth (all writes)
  - Slaves = read-only copies (scale reads)
  - Replication: async (slight lag) or sync (consistent but slower)
  - Failover: slave promoted to master if master dies

PRODUCER-CONSUMER (Message Queue):

  ┌──────────┐                        ┌──────────────┐
  │ Producer │──publish──>  ┌──────┐  │ Consumer 1   │
  │ (Order   │              │Kafka │──│ (Email Svc)  │
  │  Service)│──publish──>  │Queue │  │              │
  └──────────┘              │      │──│ Consumer 2   │
  ┌──────────┐              │      │  │ (Inventory)  │
  │ Producer │──publish──>  │      │──│              │
  │ (Payment │              └──────┘  │ Consumer 3   │
  │  Service)│                        │ (Analytics)  │
  └──────────┘                        └──────────────┘

  - Producers put messages → don't know who consumes
  - Queue/Broker = buffer between producers and consumers
  - Consumers pull independently at their own pace
  - If consumer is slow/down → messages queue up (no data loss)
  - Multiple consumers can process in parallel (scale)

KEY DIFFERENCE:
  Master-Slave:      Master CONTROLS slaves → same data, replicated
  Producer-Consumer: Producer DOESN'T KNOW consumer → tasks, decoupled
```

### 🗣️ How to Say in Interview
"These are fundamentally different patterns solving different problems. Master-Slave is about data replication and read scaling — one master node handles all writes and replicates data to multiple slave nodes that serve read queries. It's used in database clusters where write volume is lower than read volume, like MySQL replication or Redis Sentinel. The master controls the slaves; slaves are passive copies. Producer-Consumer is about decoupled asynchronous processing — producers generate tasks or events and push them to a message queue like Kafka or RabbitMQ. Consumers independently pull and process these messages at their own pace. The producer has no knowledge of who consumes the message. This pattern is used for background processing, event-driven architectures, and load buffering during traffic spikes. In my project, we used MySQL Master-Slave for horizontal read scaling, and Kafka Producer-Consumer for async order processing — the order service produced events, and email, inventory, and analytics services consumed them independently."

### 💻 Code
```java
// MASTER-SLAVE: Read/Write routing in Spring Boot
// Route writes → master datasource, reads → slave datasource

// 1. Define master and slave DataSources
@Configuration
public class DataSourceConfig {
    @Bean
    @ConfigurationProperties("spring.datasource.master")
    public DataSource masterDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @ConfigurationProperties("spring.datasource.slave")
    public DataSource slaveDataSource() {
        return DataSourceBuilder.create().build();
    }
    
    @Bean
    @Primary
    public DataSource routingDataSource(
            @Qualifier("masterDataSource") DataSource master,
            @Qualifier("slaveDataSource") DataSource slave) {
        ReadWriteRoutingDataSource routing = new ReadWriteRoutingDataSource();
        routing.setTargetDataSources(Map.of(
            "master", master,
            "slave", slave));
        routing.setDefaultTargetDataSource(master);
        return routing;
    }
}

// Routing logic — @Transactional(readOnly=true) → slave
public class ReadWriteRoutingDataSource extends AbstractRoutingDataSource {
    @Override
    protected Object determineCurrentLookupKey() {
        return TransactionSynchronizationManager.isCurrentTransactionReadOnly()
                ? "slave" : "master";
    }
}

// Usage — Spring routes automatically
@Service
public class ProductService {
    @Transactional                        // → MASTER (write)
    public Product save(Product p) { return repo.save(p); }
    
    @Transactional(readOnly = true)       // → SLAVE (read)
    public List<Product> findAll() { return repo.findAll(); }
}

// PRODUCER-CONSUMER: Kafka example
// Producer — publishes order event
@Service
@RequiredArgsConstructor
public class OrderProducer {
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    public void publishOrderCreated(Order order) {
        OrderEvent event = new OrderEvent(order.getId(), order.getTotal(), Instant.now());
        kafkaTemplate.send("order-events", order.getId().toString(), event);
        // Fire and forget — doesn't know who consumes
    }
}

// Consumer 1 — Email notification
@Component
public class EmailConsumer {
    @KafkaListener(topics = "order-events", groupId = "email-service")
    public void onOrderCreated(OrderEvent event) {
        emailService.sendConfirmation(event.orderId());
    }
}

// Consumer 2 — Inventory update (independent, different group)
@Component
public class InventoryConsumer {
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void onOrderCreated(OrderEvent event) {
        inventoryService.deductStock(event.orderId());
    }
}

// Consumer 3 — Analytics (independent, different group)
@Component
public class AnalyticsConsumer {
    @KafkaListener(topics = "order-events", groupId = "analytics-service")
    public void onOrderCreated(OrderEvent event) {
        analyticsService.recordSale(event);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Master-Slave replication lag** — slave might return stale data (writes on master not yet replicated). For critical reads after writes, read from master *(Slave pe recent write dikhe ya na dikhe — replication lag ki wajah se)*
- **Master-Slave single point of failure** — if master dies and failover isn't configured, writes stop. Use Sentinel/Cluster for automatic failover
- **Producer-Consumer message ordering** — Kafka only guarantees order within a partition. Use same key for related events
- **Consumer lag** — if consumer is slower than producer, queue grows unbounded. Monitor consumer lag, scale consumers
- **At-least-once delivery** — consumer might process same message twice (Kafka default). Make processing **idempotent**
- Don't confuse **Pub-Sub** (all subscribers get all messages) with **Queue** (each message consumed by ONE consumer). Kafka supports both via consumer groups

### 🆚 vs. Comparison
| Aspect | Master-Slave | Producer-Consumer |
|--------|-------------|------------------|
| Purpose | Data replication + read scaling | Async task processing |
| Relationship | Master controls slaves | Producer doesn't know consumers |
| Data flow | Same data → all slaves | Task/event → one or more consumers |
| Coupling | Tight (same data model) | Loose (only message contract) ⭐ |
| Scaling | Scale reads (add slaves) | Scale processing (add consumers) |
| Failure handling | Promote slave to master | Messages queue up, retry later |
| Example | MySQL replication, Redis | Kafka, RabbitMQ, SQS |
| Communication | Synchronous replication | Asynchronous ⭐ |
| Data consistency | Near real-time | Eventually consistent |

### 🎯 Tricky Interview Qs

**Q: Can you combine both patterns?**
Absolutely — and most production systems do. A MySQL master-slave setup handles database replication for read scaling, while Kafka producer-consumer decouples microservices. The master DB writes trigger CDC (Change Data Capture) events into Kafka, which consumers use to build read-optimized views (CQRS pattern). *(Dono saath use hote hain — DB mein master-slave, services ke beech producer-consumer)*

**Q: What happens when the Kafka broker itself goes down?**
Kafka uses its own replication — each topic partition has a leader and replicas (followers). If the leader broker dies, a follower is promoted. This is actually a Master-Slave pattern within Kafka itself!

### ⚡ Remember
- **Master-Slave** = one writes, others copy → **data replication** (DB, Redis)
- **Producer-Consumer** = produce tasks → queue → consume independently → **decoupled async** (Kafka, RabbitMQ)
- Master-Slave: scale reads, single write point *(Ek master likhta hai, slaves padhte hain)*
- Producer-Consumer: decouple services, buffer traffic *(Producer kaam queue mein daalta hai, consumer apni speed se uthata hai)*
- Real systems use BOTH: master-slave for DB + producer-consumer for services

### 🔗 Follow-ups
- [Q7 → Caching with Redis (Redis master-slave for HA)](#q7)
- Q18 → REST vs Kafka (architecture/01)
- Q20 → Fault tolerance (architecture/01)
