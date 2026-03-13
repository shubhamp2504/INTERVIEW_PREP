# 🔴 Spring Batch — Performance Optimization (Q92–Q98)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q92"></a>

## Q92. How do you process millions of records efficiently?

### 🔑 Quick Answer

> Use **partitioning** (split data into ranges processed in parallel) + **JdbcPagingItemReader** (memory-safe) + **JdbcBatchItemWriter** (batch inserts) + **optimal chunk size (~500)**. This combination handles 10M+ records.

### 📖 Step-by-Step Explanation

**Step 1 — The strategy stack (ordered by impact):**

```
Layer 1: PARALLELISM (biggest win)
  Partitioning → 16 partitions = 16× throughput
  ┌── Partition 1:  ID 1-625K      ──┐
  ├── Partition 2:  ID 625K-1.25M   ─┤  All run simultaneously
  ├── ...                             │  Each has own reader/writer
  └── Partition 16: ID 9.375M-10M  ──┘

Layer 2: EFFICIENT READING
  JdbcPagingItemReader → constant memory, reads in pages
  NOT cursor (holds DB connection open)
  NOT JPA (entity overhead)

Layer 3: BATCH WRITING
  JdbcBatchItemWriter → 500 inserts = 1 batch call
  NOT individual inserts (500 round trips!)

Layer 4: CHUNK SIZE TUNING
  500 items per chunk → sweet spot for most jobs
  Balance between: commit frequency vs memory vs throughput

Layer 5: INFRASTRUCTURE
  HikariCP connection pool → pool size ≥ partition count
  DB indexes on WHERE/ORDER columns
  JVM: -Xmx4g (adjust based on partitions × chunk size × object size)
```

**Step 2 — Real numbers comparison:**

```
10 Million Records:

Strategy                         | Time      | Throughput
─────────────────────────────────|───────────|────────────
Single thread, chunk=100         | 4 hours   | 700/sec
Single thread, chunk=500         | 2 hours   | 1,400/sec
Multi-threaded (8 threads)       | 25 min    | 6,700/sec
Partitioned (16 partitions)      | 15 min    | 11,000/sec  ⭐
Partitioned + tuned inserts      | 8 min     | 20,000/sec  ⭐⭐
```

### 🗣️ How to Explain in Interview

> *"For millions of records, I use a layered approach. First, partitioning — I split data by ID ranges into 16 partitions running in parallel, which gives me 16× throughput improvement. Second, I use JdbcPagingItemReader because it processes data page by page with constant memory. Third, JdbcBatchItemWriter with batch inserts — 500 items go as one batch call instead of 500 individual inserts. Fourth, I tune chunk size to 500 through benchmarking. For a 10 million record job, this combination brings processing from 4 hours down to about 8-15 minutes."*

### 💻 Code Example

```java
// Complete production setup for 10M+ records
@Bean
public Job millionRecordJob(JobRepository repo, PlatformTransactionManager tx) {
    return new JobBuilder("millionRecordJob", repo)
            .start(masterStep(repo, tx))
            .build();
}

@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("master", repo)
            .partitioner("worker", rangePartitioner())
            .step(workerStep(repo, tx))
            .gridSize(16)                         // 16 parallel partitions
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("worker", repo)
            .<Employee, Employee>chunk(500, tx)    // Tuned chunk size
            .reader(pagingReader(null, null))       // Memory-safe reader
            .writer(batchWriter())                  // Batch inserts
            .build();
}

@Bean
public TaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(16);                      // Match gridSize
    exec.setMaxPoolSize(16);
    exec.setQueueCapacity(0);
    return exec;
}
```

### ⚡ Key Points to Remember

1. **Partitioning** = biggest performance win (parallel execution)
2. **JdbcPagingItemReader** = memory-safe for millions
3. **JdbcBatchItemWriter** = batch inserts (500→1 call)
4. **Chunk size ~500** = sweet spot (benchmark to confirm)
5. Connection pool size ≥ partition count

### 🔗 Follow-up Questions

- [Q86: How does partitioning work?](10-parallel-processing.md#q86)
- [Q96: How to optimize batch inserts?](#q96)
- [Q97: How to tune chunk size?](#q97)

---

<a id="q93"></a>

## Q93. How do you improve Spring Batch performance?

### 🔑 Quick Answer

> Optimize all four layers: **Reader** (paging, indexes, projections), **Processor** (cache lookups, avoid per-item API calls), **Writer** (JDBC batch, bulk operations), **Infrastructure** (chunk size, parallelism, connection pool).

### 📖 Step-by-Step Explanation

**Step 1 — Performance checklist by layer:**

```
📖 READER OPTIMIZATION:
  ✅ JdbcPagingItemReader (not cursor for large data)
  ✅ pageSize = chunk size (avoid mismatch)
  ✅ SELECT only needed columns (not SELECT *)
  ✅ DB indexes on WHERE and ORDER BY columns
  ✅ Avoid JPA for batch — use plain JDBC

⚙️ PROCESSOR OPTIMIZATION:
  ✅ Cache reference data (load once with @BeforeStep)
  ✅ Batch external API calls (not per-item)
  ✅ Remove processor if no transformation needed
  ✅ Keep logic simple — offload heavy work

✍️ WRITER OPTIMIZATION:
  ✅ JdbcBatchItemWriter (uses JDBC batch API)
  ✅ hibernate.jdbc.batch_size = chunk size
  ✅ rewriteBatchedStatements=true (MySQL)
  ✅ COPY command for PostgreSQL bulk loads
  ✅ assertUpdates(false) for pure inserts

🏗️ INFRASTRUCTURE:
  ✅ Partitioning for parallelism
  ✅ HikariCP pool size = partition count + 2
  ✅ JVM memory: -Xmx based on data volume
  ✅ Logging at WARN level in production
  ✅ Benchmark chunk size (100, 500, 1000)
```

**Step 2 — Impact rating:**

```
Change                          | Performance Gain
────────────────────────────────|──────────────────
Add partitioning (16 threads)   | 10-16×  ⭐⭐⭐⭐⭐
Switch cursor → paging reader   | 2-3×    ⭐⭐⭐⭐
JDBC batch insert               | 5-10×   ⭐⭐⭐⭐
Tune chunk size                 | 1.5-3×  ⭐⭐⭐
Cache reference data            | 2-5×    ⭐⭐⭐
Add DB indexes                  | 2-10×   ⭐⭐⭐
Reduce logging                  | 1.1-1.3× ⭐
```

### 🗣️ How to Explain in Interview

> *"I optimize Spring Batch at four layers. For reading, I use JdbcPagingItemReader — not cursor — with proper DB indexes and SELECT only needed columns. For processing, I cache all reference data using @BeforeStep so I'm not hitting the database per-item. For writing, I use JdbcBatchItemWriter which sends 500 inserts as one batch call. For infrastructure, I use partitioning for parallelism, tune chunk size through benchmarking, and size the connection pool to match partition count. The biggest single win is always partitioning — it can give 10-16× improvement."*

### ⚡ Key Points to Remember

1. Four layers: **Reader → Processor → Writer → Infrastructure**
2. Biggest win = **Partitioning** (10-16×)
3. Cache lookups with **@BeforeStep** (not per-item DB calls)
4. **JdbcBatchItemWriter** > individual inserts (5-10× faster)
5. Always **benchmark** — don't assume, measure

---

<a id="q94"></a>

## Q94. How do you reduce database calls in Spring Batch?

### 🔑 Quick Answer

> Three main techniques: **batch writes** (500 inserts → 1 call), **cache reference data** (load once with @BeforeStep), **tune chunk/page size** (fewer commits = fewer round trips).

### 📖 Step-by-Step Explanation

**Step 1 — Where DB calls happen and how to reduce them:**

```
READING (SELECT):
  Problem: Small page size → many SELECT queries
  Fix: page size = chunk size (500)
  Before: 200 SELECTs for 100K records (page=500)
  After:  200 SELECTs (same, but fewer commits between them)

PROCESSING (LOOKUP):
  Problem: Query reference data per-item → N queries per chunk
  Fix: Cache with @BeforeStep → 1 query total
  Before: 100K lookups = 100K queries  💀
  After:  1 query to load cache         ✅

WRITING (INSERT/UPDATE):
  Problem: Individual inserts → N round trips per chunk
  Fix: Batch inserts → 1 round trip per chunk
  Before: 500 inserts = 500 round trips
  After:  500 inserts = 1 batch round trip

COMMITTING:
  Problem: Small chunk → many commits
  Fix: Larger chunk size
  Before: chunk=100, 100K records → 1000 commits
  After:  chunk=500, 100K records → 200 commits
```

### 💻 Code Example

```java
// Cache reference data — eliminates per-item DB lookups
@Component
@StepScope
public class OrderEnrichmentProcessor implements ItemProcessor<Order, Order> {
    
    private Map<Long, Customer> customerCache;
    
    @Autowired
    private CustomerRepository customerRepo;
    
    @BeforeStep
    public void loadCache(StepExecution stepExecution) {
        // ONE query loads ALL customers into memory
        customerCache = customerRepo.findAll().stream()
                .collect(Collectors.toMap(Customer::getId, Function.identity()));
    }
    
    @Override
    public Order process(Order order) {
        // In-memory lookup — ZERO database calls
        Customer customer = customerCache.get(order.getCustomerId());
        order.setCustomerName(customer.getName());
        order.setCustomerTier(customer.getTier());
        return order;
    }
}
```

### 🗣️ How to Explain in Interview

> *"I reduce DB calls at three levels. First, writing: I use JdbcBatchItemWriter which converts 500 individual inserts into one batch call — that's 500 round trips reduced to 1. Second, processing: I cache all reference data using @BeforeStep. Instead of querying the customer table per order — 100K queries for 100K orders — I load the entire customer table once into a HashMap. Third, I tune chunk and page size to reduce the number of commits and SELECT queries. These three changes can take a job from millions of DB calls down to a few hundred."*

### ⚡ Key Points to Remember

1. **Batch writes** = N inserts → 1 batch call
2. **@BeforeStep cache** = eliminate per-item lookups
3. **page size = chunk size** = aligned reads
4. **Larger chunks** = fewer commits
5. For huge reference data → use **LRU cache** instead of loading all

---

<a id="q95"></a>

## Q95. What is the best reader for large datasets?

### 🔑 Quick Answer

> **JdbcPagingItemReader** — reads data page by page with constant memory usage. Combined with **partitioning** for very large datasets.

### 📖 Step-by-Step Explanation

**Step 1 — Reader comparison for large data:**

```
JdbcPagingItemReader ⭐⭐⭐⭐⭐
  ✅ Constant memory (processes one page at a time)
  ✅ Thread-safe (works with partitioning)
  ✅ Restartable (tracks page number)
  ✅ No connection held open
  ❌ Slightly slower than cursor (multiple queries)

JdbcCursorItemReader ⭐⭐⭐
  ✅ Fastest single query
  ❌ Holds DB connection open for entire step
  ❌ NOT thread-safe
  ❌ Connection timeout on very long jobs
  ❌ Memory risk (cursor buffer)

JpaPagingItemReader ⭐⭐
  ✅ Works with JPA entities
  ❌ Entity overhead (dirty checking, caching)
  ❌ Slower than JDBC
  ❌ Memory leak risk (persistence context)

FlatFileItemReader ⭐⭐⭐⭐
  ✅ Streaming (constant memory)
  ✅ No DB connection needed
  ❌ Not thread-safe
  ❌ Not inherently partitionable (need MultiResource)
```

**Step 2 — Decision by data size:**

```
Data Size     | Recommended Reader              | Approach
──────────────|─────────────────────────────────|──────────
< 100K        | Any reader works                | Single step
100K - 1M     | JdbcPagingItemReader            | Single step
1M - 10M      | JdbcPagingItemReader            | Partitioned (8 partitions)
10M - 100M    | JdbcPagingItemReader            | Partitioned (16-32 partitions)
100M+         | JdbcPagingItemReader            | Remote partitioning (50+ workers)
File-based    | FlatFileItemReader              | MultiResourceItemReader
```

### 🗣️ How to Explain in Interview

> *"For large datasets, JdbcPagingItemReader is my default choice. It reads data page by page — say 500 rows per query — so memory stays constant regardless of total data size. Unlike cursor reader, it doesn't hold the DB connection open for the entire step, which avoids timeout issues on long-running jobs. It's thread-safe, so it works with partitioning. For 10 million records, I'd combine it with partitioning — 16 partitions each with their own paging reader. The only case I'd use cursor is for smaller datasets where single-query performance matters and the job completes within the connection timeout."*

### ⚡ Key Points to Remember

1. **JdbcPagingItemReader** = default for large data ⭐
2. Key advantage: **constant memory** + **no connection hold**
3. **Thread-safe** → works with partitioning
4. For 10M+ → **partitioned JdbcPagingItemReader**
5. Avoid JPA for batch — too much entity overhead

---

<a id="q96"></a>

## Q96. How do you optimize batch inserts?

### 🔑 Quick Answer

> Use **JdbcBatchItemWriter** (not JPA), match **batch_size = chunk size**, enable **rewriteBatchedStatements** (MySQL), and use **COPY command** for PostgreSQL. This can give 5-10× improvement.

### 📖 Step-by-Step Explanation

**Step 1 — What makes batch inserts fast:**

```
SLOW (500 individual inserts = 500 round trips):
  INSERT INTO emp VALUES (1, 'Alice', 50000)    → DB → OK
  INSERT INTO emp VALUES (2, 'Bob', 55000)      → DB → OK
  INSERT INTO emp VALUES (3, 'Carol', 60000)    → DB → OK
  ... × 500 times
  Time: 2 seconds

FAST (1 batch call = 1 round trip):
  addBatch(INSERT INTO emp VALUES (1, 'Alice', 50000))
  addBatch(INSERT INTO emp VALUES (2, 'Bob', 55000))
  addBatch(INSERT INTO emp VALUES (3, 'Carol', 60000))
  ... × 500 times
  executeBatch()   → DB → OK (all 500 at once)
  Time: 0.1 seconds  (20× faster)
```

**Step 2 — Configuration layers:**

```
Layer 1: Use JdbcBatchItemWriter (not JPA writer)
Layer 2: batch_size = chunk size (e.g., both 500)
Layer 3: MySQL: rewriteBatchedStatements=true
         (rewrites individual INSERTs into multi-value INSERT)
Layer 4: PostgreSQL: COPY command (fastest possible)
Layer 5: assertUpdates(false) for pure inserts (skip verification)
```

### 💻 Code Example

```java
// Fastest writer configuration
@Bean
public JdbcBatchItemWriter<Employee> writer(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .sql("INSERT INTO employees(name, salary, dept) VALUES(:name, :salary, :dept)")
            .dataSource(ds)
            .beanMapped()
            .assertUpdates(false)       // Skip row count check (faster for inserts)
            .build();
}
```

```yaml
# application.yml
spring:
  datasource:
    hikari:
      maximum-pool-size: 20
      # MySQL: Enable batch rewrite for massive speedup
      jdbc-url: jdbc:mysql://host/db?rewriteBatchedStatements=true
  jpa:
    properties:
      hibernate:
        jdbc:
          batch_size: 500             # Match chunk size
        order_inserts: true           # Group inserts by entity type
        order_updates: true           # Group updates by entity type
```

### 🗣️ How to Explain in Interview

> *"I optimize batch inserts at multiple levels. First, I use JdbcBatchItemWriter instead of JPA writer — JPA has entity management overhead like dirty checking. JdbcBatchItemWriter uses JDBC's addBatch/executeBatch API, which sends 500 inserts as one network call. Second, I ensure batch_size matches chunk size — if chunk is 500, batch is 500. Third, for MySQL I enable rewriteBatchedStatements=true in the JDBC URL, which rewrites individual INSERT statements into a single multi-value INSERT. For pure inserts, I set assertUpdates to false to skip row count verification. These changes together give 5-10× improvement over naive inserts."*

### ⚡ Key Points to Remember

1. **JdbcBatchItemWriter** > JPA writer for performance
2. **batch_size = chunk size** (must match)
3. **rewriteBatchedStatements=true** for MySQL (major speedup)
4. **assertUpdates(false)** for pure inserts
5. **COPY** command for PostgreSQL (fastest possible)

---

<a id="q97"></a>

## Q97. How do you tune chunk size?

### 🔑 Quick Answer

> Start with **500**, benchmark with **100, 500, 1000, 2000**, pick the one that balances **throughput vs memory**. The sweet spot is where increasing chunk size no longer significantly improves speed.

### 📖 Step-by-Step Explanation

**Step 1 — What chunk size affects:**

```
Small Chunk (100):
  ✅ Low memory usage
  ✅ Less data lost on failure (100 items re-processed)
  ❌ More commits → more overhead
  ❌ More DB round trips

Large Chunk (5000):
  ✅ Fewer commits → less overhead
  ✅ Better throughput
  ❌ High memory (5000 objects in memory)
  ❌ More data lost on failure (5000 items re-processed)
  ❌ Long transactions → DB lock contention
```

**Step 2 — Benchmarking approach:**

```
Test | Chunk Size | Time (100K) | Memory  | Commits | Verdict
─────|────────────|─────────────|─────────|─────────|────────
1    | 50         | 180 sec     | 128 MB  | 2000    | Too slow
2    | 100        | 120 sec     | 256 MB  | 1000    | OK
3    | 500        | 80 sec      | 512 MB  | 200     | ⭐ Sweet spot
4    | 1000       | 75 sec      | 1 GB    | 100     | Diminishing returns
5    | 5000       | 73 sec      | 3 GB    | 20      | Wasting memory
6    | 10000      | OOM         | 💀      | —       | Too large

Sweet spot: chunk=500 (biggest time drop with acceptable memory)
```

**Step 3 — Rules of thumb:**

```
Simple read/write, small objects  → 500-1000
Complex processing per item       → 100-300
External API calls in processor   → 50-100
Very large objects (images, XML)  → 50-100
Pure inserts, no processing       → 1000-5000
```

### 🗣️ How to Explain in Interview

> *"I tune chunk size through benchmarking. I start with 500 — it's a good default — then test with 100, 500, 1000, and 2000. I measure three things: throughput, memory usage, and failure impact. The sweet spot is where increasing chunk size stops giving significant throughput improvement. Usually that's 500 for most jobs. I also consider the use case — for complex processing or external API calls, I use smaller chunks like 100. For simple inserts, I might go to 1000. I never go above 5000 because the memory risk outweighs the marginal performance gain."*

### ⚡ Key Points to Remember

1. **Default: 500** (good starting point for most jobs)
2. **Benchmark** with 100, 500, 1000 — pick the sweet spot
3. Chunk size trade-off: **throughput vs memory vs failure impact**
4. Complex processing → **smaller chunks** (100-300)
5. **Never** > 5000 (OOM risk, diminishing returns)

---

<a id="q98"></a>

## Q98. How do you avoid memory issues?

### 🔑 Quick Answer

> Use **paging reader** (not cursor), **clear JPA cache per chunk**, **reduce chunk size**, and **monitor heap usage**. The most common cause is accumulating objects in the persistence context or growing state in ExecutionContext.

### 📖 Step-by-Step Explanation

**Step 1 — Common memory issues and fixes:**

```
Issue 1: JPA persistence context grows endlessly
  Cause: EntityManager caches all read entities
  Fix: Clear EntityManager after each chunk
  
Issue 2: Large chunk size
  Cause: 5000 objects in memory per chunk
  Fix: Reduce chunk size (500 or less)

Issue 3: ExecutionContext growing
  Cause: Storing too much state in ExecutionContext
  Fix: Store only essential data (counts, positions)

Issue 4: Cursor reader buffering
  Cause: JdbcCursorItemReader may buffer large result sets
  Fix: Switch to JdbcPagingItemReader

Issue 5: Reference data cache too large
  Cause: Loading entire reference table into HashMap
  Fix: Use LRU cache (e.g., Caffeine) with size limit

Issue 6: Too many concurrent partitions
  Cause: 50 partitions × 500 chunk = 25K objects
  Fix: Reduce gridSize or chunk size
```

**Step 2 — JPA memory leak (most common):**

```
Chunk 1: Read 500 entities → EntityManager holds 500
Chunk 2: Read 500 more    → EntityManager holds 1000
Chunk 3: Read 500 more    → EntityManager holds 1500
...
Chunk 200: Read 500 more  → EntityManager holds 100,000 → OOM! 💀

FIX: Clear after each chunk:
Chunk 1: Read 500 → process → write → clear() → 0 in memory ✅
Chunk 2: Read 500 → process → write → clear() → 0 in memory ✅
```

### 💻 Code Example

```java
// Fix JPA memory leak: Clear persistence context after each chunk
@Bean
public Step safeStep(JobRepository repo, PlatformTransactionManager tx,
                     EntityManagerFactory emf) {
    return new StepBuilder("safeStep", repo)
            .<Employee, Employee>chunk(500, tx)
            .reader(pagingReader())                // Paging, NOT cursor
            .processor(processor())
            .writer(writer())
            .listener(new ChunkListener() {
                @Override
                public void afterChunk(ChunkContext context) {
                    // Clear JPA cache after each chunk → constant memory
                    EntityManager em = emf.createEntityManager();
                    em.clear();
                    em.close();
                }
            })
            .build();
}
```

```
JVM flags for large batch jobs:
  -Xms2g -Xmx4g              # Adequate heap
  -XX:+UseG1GC                # G1 for large heaps
  -XX:+HeapDumpOnOutOfMemoryError  # Debug OOM
```

### 🗣️ How to Explain in Interview

> *"The most common memory issue I've seen is JPA persistence context growth. EntityManager caches every entity it reads, so by chunk 200, you have 100K entities in memory causing OOM. The fix is clearing the EntityManager after each chunk using a ChunkListener. Other fixes: use JdbcPagingItemReader instead of cursor — paging has constant memory while cursor can buffer. Keep chunk size reasonable — 500, not 5000. And be careful with ExecutionContext — store only positions and counts, not actual data. For monitoring, I add heap usage logging and enable HeapDumpOnOutOfMemoryError to diagnose issues."*

### ⚡ Key Points to Remember

1. **#1 cause**: JPA persistence context growth → clear per chunk
2. **Paging reader** = constant memory (cursor can buffer)
3. **Chunk size 500** = reasonable memory footprint
4. **Don't store data** in ExecutionContext (only metadata)
5. **Monitor heap** with -XX:+HeapDumpOnOutOfMemoryError

---

> **🎯 Navigation:** [← Parallel Processing (Q84-91)](10-parallel-processing.md) | [Next → Scheduling (Q99-103)](12-scheduling.md) | [📋 All Sections](README.md)
