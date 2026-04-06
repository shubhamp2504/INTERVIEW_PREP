# 🔴 Performance Optimization — Q92 to Q98

---

## Q92. How do you process millions of records efficiently?

### 📝 One-Liner
Partitioning + JdbcPagingItemReader + JdbcBatchItemWriter + chunk ~500 = handles 10M+ records.

### 🔑 Quick Answer
5-layer strategy: **(1) Partitioning** — split data into ranges, process in parallel (biggest gain: 10-16×). **(2) JdbcPagingItemReader** — memory-safe, reads page by page with constant memory. **(3) JdbcBatchItemWriter** — batch inserts (500 inserts → 1 DB call). **(4) Optimal chunk size ~500** — balance throughput vs memory. **(5) Proper infrastructure** — thread pool, connection pool, indexes. Real numbers: single thread 4 hours → partitioned+tuned 8 minutes for 10M records. *(Partitioning sabse bada gain deta hai — baaki sab uske upar optimize karte hain)*

### 📖 How It Works
```
5-Layer Strategy Stack:

Layer 1: Parallelism (BIGGEST WIN — 10-16× gain)
  ├── Partitioning ⭐ (recommended)
  │   10M records ÷ 16 partitions = 625K per partition
  │   16 threads process simultaneously
  └── Each partition = independent StepExecution (restartable)

Layer 2: Efficient Reading (constant memory)
  ├── JdbcPagingItemReader (page-by-page, thread-safe)
  ├── pageSize = chunkSize = 500
  └── Never loads all records into memory

Layer 3: Batch Writing (500→1 round trip)
  ├── JdbcBatchItemWriter (addBatch+executeBatch)
  ├── 500 individual inserts → 1 batch call
  └── rewriteBatchedStatements=true (MySQL)

Layer 4: Chunk Size Tuning
  ├── Start with 500
  ├── Benchmark: 100, 500, 1000, 2000
  └── Sweet spot: where increasing size doesn't help anymore

Layer 5: Infrastructure
  ├── Thread pool ≥ partition count
  ├── DB connection pool ≥ threads + 2
  └── Indexes on partition/sort/filter columns

Result: 10M records
  Before: 4 hours (single thread, individual inserts)
  After:  8 minutes (16 partitions, batch inserts, tuned)
```

### 🗣️ Answering Approach
"For processing millions of records, I use a 5-layer strategy. Partitioning gives the biggest performance gain — we split 10M records into 16 ID-range partitions processed by 16 threads simultaneously. Each partition uses JdbcPagingItemReader for constant memory usage and JdbcBatchItemWriter for batch inserts — converting 500 individual inserts into 1 database call. Chunk size is tuned to 500 after benchmarking. In my project, this reduced processing time for 10M payment records from 4 hours to 8 minutes. The key insight is that partitioning gives 10-16× improvement while all other optimizations combined give 2-5×."

### 💻 Code
```java
// Production setup for 10M+ records
@Bean
public Job millionRecordJob(JobRepository repo, Step masterStep) {
    return new JobBuilder("paymentProcessingJob", repo)
            .start(masterStep)
            .build();
}

@Bean
public Step masterStep(JobRepository repo, Step workerStep) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", idRangePartitioner())
            .step(workerStep)
            .gridSize(16)                          // 16 partitions
            .taskExecutor(batchTaskExecutor())
            .build();
}

@Bean
public Step workerStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("workerStep", repo)
            .<Payment, ProcessedPayment>chunk(500, tx)
            .reader(partitionedReader(null, null))
            .processor(paymentProcessor())
            .writer(batchWriter())
            .build();
}

@Bean
@StepScope
public JdbcPagingItemReader<Payment> partitionedReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<Payment>()
            .name("paymentReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM payments")
            .whereClause("WHERE id BETWEEN :minId AND :maxId")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Payment.class))
            .build();
}

@Bean
public JdbcBatchItemWriter<ProcessedPayment> batchWriter() {
    return new JdbcBatchItemWriterBuilder<ProcessedPayment>()
            .sql("INSERT INTO processed_payments (id, amount, status) VALUES (:id, :amount, :status)")
            .dataSource(dataSource)
            .beanMapped()
            .build();
}

@Bean
public TaskExecutor batchTaskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(16);
    executor.setMaxPoolSize(16);
    executor.initialize();
    return executor;
}
```

### ⚠️ Pitfalls / Gotchas
- Connection pool must be ≥ thread count + 2 (partitions + master + monitoring) *(connection pool kam hai toh threads wait karenge)*
- Partition columns must be indexed — without index, each partition does full table scan
- Data skew (uneven partition sizes) wastes resources — fastest thread idles
- Don't skip Layer 1 (partitioning) — it's 80% of the gain

### 🎯 Tricky Interview Qs

**Q: What if 10M records have 8M in one ID range and 2M in another?**
Data skew — some partitions process 5× more. Solution: partition by hash instead of range, or use more partitions than threads so idle threads pick up next partition.

**Q: JdbcPagingItemReader or JdbcCursorItemReader for 10M?**
Paging — constant memory, thread-safe. Cursor holds connection for entire read and isn't thread-safe.

### ⚡ Remember
- **Partitioning = biggest win** (10-16× improvement) *(partitioning se sabse zyada speed milti hai)*
- JdbcPagingItemReader = constant memory, thread-safe
- JdbcBatchItemWriter = 500 → 1 DB call
- Chunk size ~500, benchmark to confirm
- Pool sizes: threads ≤ connections - 2

### 🔗 Follow-ups
- [Q86 → Partitioning details](#q86)
- [Q96 → Optimize batch inserts](#q96)
- [Q97 → Tune chunk size](#q97)

---

## Q93. How do you improve Spring Batch performance?

### 📝 One-Liner
Optimize four layers: Reader (paging, indexes), Processor (cache lookups), Writer (JDBC batch), Infrastructure (partitioning, chunk size).

### 🔑 Quick Answer
**(1) Reader**: use JdbcPagingItemReader, add database indexes, project only needed columns. **(2) Processor**: cache reference data with @BeforeStep (load once, lookup in-memory), avoid per-item API calls. **(3) Writer**: JdbcBatchItemWriter > JPA (5-10× faster), enable rewriteBatchedStatements. **(4) Infrastructure**: partitioning (10-16×), chunk size ~500, connection pool ≥ threads + 2. Impact: Partitioning > Caching > Batch writing > Chunk tuning. *(Char layers — Reader, Processor, Writer, Infrastructure — sabko optimize karo)*

### 📖 How It Works
```
Performance Optimization Checklist:

READER (Layer 1):
  ☐ JdbcPagingItemReader (not Cursor) ............ 2× gain
  ☐ SELECT only needed columns ................... 1.5×
  ☐ Index on WHERE + ORDER BY columns ............ 5-10×
  ☐ pageSize = chunkSize ......................... consistency

PROCESSOR (Layer 2):
  ☐ @BeforeStep cache reference data ............. 2-5×
  ☐ Avoid per-item DB calls ...................... 10×
  ☐ Batch external API calls ..................... 3-5×
  ☐ Filter early (return null) ................... reduces write load

WRITER (Layer 3):
  ☐ JdbcBatchItemWriter (not JPA) ................ 5-10×
  ☐ rewriteBatchedStatements=true ................ 2×
  ☐ Batch size = chunk size ...................... consistency
  ☐ assertUpdates(false) for inserts ............. skip update check

INFRASTRUCTURE (Layer 4):
  ☐ Partitioning ⭐ ............................. 10-16×
  ☐ Chunk size ~500 ............................. tuned
  ☐ Connection pool ≥ threads + 2 ............... no starvation
  ☐ Thread pool = partition count ................ full utilization
```

### 🗣️ Answering Approach
"I optimize four layers. For the reader, JdbcPagingItemReader with proper indexes and column projection. For the processor, I cache reference data using @BeforeStep — loading all lookup data into a HashMap once instead of querying per item, which eliminated thousands of database calls. For the writer, JdbcBatchItemWriter is 5-10× faster than JPA because it uses native batch inserts. For infrastructure, partitioning is the biggest single optimization at 10-16×. In my project, the combination of all four layers reduced our daily payment job from 4 hours to 8 minutes."

### 💻 Code
```java
// Processor with @BeforeStep cache — key optimization
@Component
@StepScope
public class OrderEnrichmentProcessor implements ItemProcessor<Order, EnrichedOrder> {
    private Map<Long, Customer> customerCache;
    
    @Autowired private JdbcTemplate jdbc;

    @BeforeStep
    public void loadCache(StepExecution stepExecution) {
        // Load ALL reference data ONCE — not per item
        customerCache = jdbc.query("SELECT id, name, tier FROM customers",
            (rs, i) -> new Customer(rs.getLong("id"), rs.getString("name"), rs.getString("tier")))
            .stream()
            .collect(Collectors.toMap(Customer::getId, Function.identity()));
        // 50K customers loaded in 2 sec vs 50K individual queries
    }

    @Override
    public EnrichedOrder process(Order order) {
        Customer customer = customerCache.get(order.getCustomerId()); // O(1) HashMap lookup
        return new EnrichedOrder(order, customer.getName(), customer.getTier());
    }
}
```

### ⚠️ Pitfalls / Gotchas
- @BeforeStep cache only works for data that fits in memory — for huge reference data use LRU cache *(bahut bada reference data hai toh LRU cache use karo)*
- rewriteBatchedStatements is MySQL-specific — PostgreSQL uses COPY
- Don't optimize prematurely — benchmark first, optimize the real bottleneck
- JPA writer clears EntityManager per chunk — if not, OOM from persistence context growth

### ⚡ Remember
- Four layers: Reader → Processor → Writer → Infrastructure
- **Biggest win = Partitioning** (10-16×) *(partitioning = sabse bada gain)*
- Cache with @BeforeStep (2-5×)
- JdbcBatchItemWriter > JPA (5-10×)
- Always benchmark before and after

### 🔗 Follow-ups
- [Q92 → Processing millions of records](#q92)
- [Q94 → Reduce database calls](#q94)
- [Q96 → Optimize batch inserts](#q96)

---

## Q94. How do you reduce database calls in Spring Batch?

### 📝 One-Liner
Three techniques: batch writes (500→1 call), cache reference data (@BeforeStep HashMap), tune chunk/page size (fewer commits).

### 🔑 Quick Answer
Database calls happen in 4 places: **(1) Reading** — tune pageSize to reduce round trips. **(2) Processing/Lookup** — cache reference data with @BeforeStep HashMap (eliminate per-item queries). **(3) Writing** — JdbcBatchItemWriter converts N inserts → 1 batch call. **(4) Committing** — larger chunk size = fewer commits. The biggest saving is caching lookups — converting 50K individual queries into 1 bulk load. *(Har item ke liye DB query mat karo — @BeforeStep mein sab ek baar load karo)*

### 📖 How It Works
```
Where DB Calls Happen and How to Reduce:

READING (per page):
  Before: pageSize=10 → 5000 pages for 50K records
  After:  pageSize=500 → 100 pages for 50K records → 50× fewer reads

LOOKUP (per item):
  Before: 50K items × 1 query each = 50,000 DB calls (2+ minutes)
  After:  @BeforeStep loads all → 1 query + 50K HashMap lookups (2 sec)

WRITING (per chunk):
  Before: Individual inserts → 500 DB calls per chunk
  After:  Batch insert → 1 DB call per chunk → 500× fewer writes

COMMITTING (per chunk):
  Before: chunk=50 → 1000 commits for 50K records
  After:  chunk=500 → 100 commits for 50K records → 10× fewer commits

Total DB calls for 50K records:
  Before: 100 + 50,000 + 50,000 + 1,000 = 101,100 calls
  After:  100 + 1 + 100 + 100 = 301 calls → 335× reduction!
```

### 🗣️ Answering Approach
"Database calls are the biggest performance bottleneck in batch processing. I target four areas. First, I match pageSize to chunkSize — typically 500. Second, and most impactful, I cache reference data using @BeforeStep — loading all customers into a HashMap once instead of querying per item. This alone eliminated 50,000 database calls in our order enrichment job. Third, JdbcBatchItemWriter batches all writes into one call. Fourth, larger chunk size means fewer commits. The net effect was reducing total database calls from over 100,000 to about 300 for a 50K record batch."

### 💻 Code
```java
@Component
@StepScope
public class OrderProcessor implements ItemProcessor<Order, EnrichedOrder> {
    private Map<Long, Customer> customerCache;
    private Map<String, ProductCategory> categoryCache;

    @Autowired private JdbcTemplate jdbc;

    @BeforeStep
    public void loadCaches(StepExecution step) {
        // Load ALL reference data in bulk — 2 queries total
        customerCache = jdbc.query("SELECT id, name, tier FROM customers",
            (rs, i) -> new Customer(rs.getLong("id"), rs.getString("name"), rs.getString("tier")))
            .stream().collect(Collectors.toMap(Customer::getId, Function.identity()));

        categoryCache = jdbc.query("SELECT code, name, discount FROM product_categories",
            (rs, i) -> new ProductCategory(rs.getString("code"), rs.getString("name"), rs.getDouble("discount")))
            .stream().collect(Collectors.toMap(ProductCategory::getCode, Function.identity()));
    }

    @Override
    public EnrichedOrder process(Order order) {
        // O(1) HashMap lookups — ZERO database calls
        Customer customer = customerCache.get(order.getCustomerId());
        ProductCategory category = categoryCache.get(order.getCategoryCode());
        return new EnrichedOrder(order, customer, category);
    }
}

// application.yml — match page and batch sizes
// spring.datasource.hikari.maximum-pool-size: 20
// pageSize = chunkSize = 500
```

### ⚠️ Pitfalls / Gotchas
- If reference data is too large for memory (10M+ rows), use LRU/Caffeine cache instead of full load *(bahut bada data hai toh Caffeine cache use karo — poora memory mein mat load karo)*
- pageSize and chunkSize should match — different values cause unnecessary reads
- Batch inserts require rewriteBatchedStatements=true on MySQL for real batching
- Connection pool exhaustion: each thread holds a connection during chunk processing

### ⚡ Remember
- **@BeforeStep cache** = biggest DB call reducer *(ek baar load, baar baar use)*
- pageSize = chunkSize = 500
- JdbcBatchItemWriter = N → 1 call
- Larger chunk = fewer commits
- For huge ref data → LRU cache (not full load)

### 🔗 Follow-ups
- [Q93 → Performance optimization layers](#q93)
- [Q96 → Optimize batch inserts](#q96)
- [Q98 → Avoid memory issues](#q98)

---

## Q95. What is the best reader for large datasets?

### 📝 One-Liner
JdbcPagingItemReader — reads page by page with constant memory, thread-safe, works with partitioning.

### 🔑 Quick Answer
**JdbcPagingItemReader** is the default choice for large datasets. It reads page by page (SQL LIMIT/OFFSET or keyset pagination), uses constant memory regardless of dataset size, and is thread-safe for multi-threaded steps and partitioning. For very large datasets (10M+), combine with partitioning. JdbcCursorItemReader is slightly faster for small-medium data but holds a DB connection for the entire read and isn't thread-safe. JpaPagingItemReader adds ORM overhead — avoid for pure batch workloads. *(Bade dataset ke liye JdbcPagingItemReader — memory constant rehti hai, thread-safe hai)*

### 📖 How It Works
```
Reader Comparison for Large Data:

JdbcPagingItemReader ⭐⭐⭐⭐⭐ (recommended)
  ├── Reads page by page (500 rows per query)
  ├── Memory: constant (~page size)
  ├── Thread-safe: ✅ Yes
  ├── Connection: acquired per page (released between)
  ├── Restartable: ✅ Yes
  └── Use for: 100K+ records

JdbcCursorItemReader ⭐⭐⭐
  ├── Opens DB cursor, fetches row by row
  ├── Memory: low (one row at a time)
  ├── Thread-safe: ❌ No
  ├── Connection: held for ENTIRE read
  ├── Restartable: ✅ Yes (not with multi-thread)
  └── Use for: <100K records, single-threaded only

JpaPagingItemReader ⭐⭐
  ├── Uses JPQL, ORM mapping overhead
  ├── Memory: page + entity cache growth
  ├── Thread-safe: ✅ Yes
  ├── Overhead: entity lifecycle management
  └── Use for: when JPA entities are mandatory

FlatFileItemReader ⭐⭐⭐⭐
  ├── Streaming (line by line)
  ├── Memory: constant
  ├── Thread-safe: ❌ No
  └── Use for: CSV/fixed-width files

Decision Table:
  <100K rows   → JdbcCursorItemReader (simplest)
  100K-1M rows → JdbcPagingItemReader
  1M-10M rows  → JdbcPagingItemReader + partitioning
  10M-100M+    → JdbcPagingItemReader + partitioning (20+ partitions)
```

### 🗣️ Answering Approach
"For large datasets, JdbcPagingItemReader is my default choice. It reads page by page with constant memory — whether the table has 100K or 100M rows, memory usage stays the same. It's thread-safe, which means it works seamlessly with multi-threaded steps and partitioning. For our 10M-record payment job, we used JdbcPagingItemReader with 16-partition setup, and each partition's reader only consumed about 50MB regardless of partition size. I avoid JpaPagingItemReader for batch because ORM overhead and entity cache growth cause performance and memory issues at scale."

### 💻 Code
```java
// JdbcPagingItemReader — production standard for large data
@Bean
@StepScope
public JdbcPagingItemReader<Payment> paymentReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<Payment>()
            .name("paymentReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, amount, status, created_date")  // only needed columns
            .fromClause("FROM payments")
            .whereClause("WHERE id BETWEEN :minId AND :maxId AND status = 'PENDING'")
            .sortKeys(Map.of("id", Order.ASCENDING))  // mandatory for paging
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)  // match chunk size
            .rowMapper(new BeanPropertyRowMapper<>(Payment.class))
            .build();
}

// For files — FlatFileItemReader (streaming, constant memory)
@Bean
@StepScope
public FlatFileItemReader<Transaction> fileReader(
        @Value("#{stepExecutionContext['file']}") Resource file) {
    return new FlatFileItemReaderBuilder<Transaction>()
            .name("fileReader")
            .resource(file)
            .delimited().delimiter(",")
            .names("id", "amount", "date", "status")
            .targetType(Transaction.class)
            .build();
}
```

### 🆚 vs. Comparison
| Feature | JdbcPaging ⭐ | JdbcCursor | JpaPaging |
|---------|------------|-----------|-----------|
| Memory | Constant | Low (1 row) | Growing (entity cache) |
| Thread-safe | ✅ Yes | ❌ No | ✅ Yes |
| Connection | Per page | Entire read | Per page |
| Speed | Fast | Fastest | Slow |
| Partitioning | ✅ Works | ❌ Wrapper needed | ✅ Works |
| Best for | Production ⭐ | Small data | JPA required |

### ⚡ Remember
- **JdbcPagingItemReader** = default for large data *(bada data = PagingReader)*
- Constant memory, thread-safe, works with partitioning
- JdbcCursor = faster but holds connection, not thread-safe
- JPA = avoid for batch (ORM overhead + entity cache leak)
- 10M+ → Paging + Partitioning

### 🔗 Follow-ups
- [Q34 → JdbcPagingItemReader details](#q34)
- [Q33 → JdbcCursorItemReader details](#q33)
- [Q92 → Processing millions of records](#q92)

---

## Q96. How do you optimize batch inserts?

### 📝 One-Liner
JdbcBatchItemWriter + rewriteBatchedStatements=true + batch_size = chunk size = 5-10× faster than individual inserts.

### 🔑 Quick Answer
**JdbcBatchItemWriter** uses JDBC batch API (`addBatch()` + `executeBatch()`) — converts 500 individual inserts into 1 database call. Five config layers: **(1)** Use JdbcBatchItemWriter (not JPA). **(2)** batch_size = chunk size (e.g., 500). **(3)** `rewriteBatchedStatements=true` (MySQL — rewrites to multi-row INSERT). **(4)** `assertUpdates(false)` for pure inserts (skip update count check). **(5)** PostgreSQL → COPY command for maximum speed. Net result: 2 sec → 0.1 sec per chunk (20× faster). *(500 alag insert ki jagah 1 baar mein sab bhejo — 20 guna tez)*

### 📖 How It Works
```
Slow (Individual Inserts):
  INSERT INTO orders VALUES (1, ...);  → round trip 1
  INSERT INTO orders VALUES (2, ...);  → round trip 2
  ...
  INSERT INTO orders VALUES (500, ...); → round trip 500
  Total: 500 round trips → ~2 seconds

Fast (JDBC Batch):
  PreparedStatement.addBatch() × 500
  PreparedStatement.executeBatch()     → 1 round trip
  Total: 1 round trip → ~0.1 seconds → 20× faster!

Even Faster (MySQL rewrite):
  Without: 500 individual INSERT statements in batch
  With:    INSERT INTO orders VALUES (1,...),(2,...),(3,...),... (one multi-row)
  → MySQL processes as single operation → additional 2× gain

Configuration Stack:
  Layer 1: JdbcBatchItemWriter (not JPA) ........... 5-10×
  Layer 2: batch_size = chunk size ................. consistency
  Layer 3: rewriteBatchedStatements=true (MySQL) ... 2×
  Layer 4: assertUpdates(false) .................... skip check
  Layer 5: COPY (PostgreSQL) ...................... 10-50×
```

### 🗣️ Answering Approach
"For optimizing batch inserts, JdbcBatchItemWriter is essential — it uses JDBC batch API to convert 500 individual inserts into a single database call, reducing round trips from 500 to 1. On MySQL, I enable rewriteBatchedStatements which rewrites individual INSERT statements into a single multi-row INSERT, giving an additional 2× improvement. I also match batch\_size to chunk size for consistency. In my project, switching from JPA writer to JdbcBatchItemWriter with rewriteBatchedStatements reduced write time from 2 seconds to 0.1 seconds per chunk — a 20× improvement."

### 💻 Code
```java
@Bean
public JdbcBatchItemWriter<ProcessedOrder> batchWriter() {
    return new JdbcBatchItemWriterBuilder<ProcessedOrder>()
            .sql("INSERT INTO processed_orders (id, amount, status, processed_date) " +
                 "VALUES (:id, :amount, :status, :processedDate)")
            .dataSource(dataSource)
            .beanMapped()
            .assertUpdates(false)  // skip update count check for pure inserts
            .build();
}

// application.yml — critical MySQL optimization
// spring:
//   datasource:
//     url: jdbc:mysql://host/db?rewriteBatchedStatements=true
//     hikari:
//       maximum-pool-size: 20
//   jpa:
//     properties:
//       hibernate:
//         jdbc.batch_size: 500        # match chunk size
//         order_inserts: true          # group same-table inserts
//         order_updates: true          # group same-table updates

// PostgreSQL COPY for maximum speed (custom writer)
public class PostgresCopyWriter implements ItemWriter<Order> {
    @Override
    public void write(Chunk<? extends Order> items) throws Exception {
        CopyManager copy = new CopyManager(connection.unwrap(BaseConnection.class));
        StringWriter sw = new StringWriter();
        for (Order item : items) {
            sw.write(item.getId() + "\t" + item.getAmount() + "\t" + item.getStatus() + "\n");
        }
        copy.copyIn("COPY orders FROM STDIN", new StringReader(sw.toString()));
    }
}
```

### ⚠️ Pitfalls / Gotchas
- rewriteBatchedStatements is MySQL-only — doesn't exist in PostgreSQL *(ye sirf MySQL ke liye hai)*
- JPA writer flushes per item internally → loses batch benefit
- assertUpdates(true) throws exception if update count doesn't match — disable for INSERTs
- Connection pool exhaustion: each partition thread holds a connection during write
- batch_size ≠ chunk size → partial batches waste resources

### 🆚 vs. Comparison
| Method | Speed | Round Trips | Complexity |
|--------|-------|-------------|-----------|
| Individual INSERT | 1× (baseline) | N per chunk | Simple |
| JdbcBatchItemWriter | 5-10× | 1 per chunk | Low |
| + rewriteBatchedStatements | 10-20× | 1 (multi-row) | Low |
| PostgreSQL COPY | 20-50× | 1 | Medium |

### ⚡ Remember
- JdbcBatchItemWriter: 500 → 1 round trip *(ek baar mein sab insert — 20x tez)*
- rewriteBatchedStatements=true (MySQL)
- batch_size = chunk size = 500
- assertUpdates(false) for inserts
- PostgreSQL → COPY command for max speed

### 🔗 Follow-ups
- [Q42 → JdbcBatchItemWriter details](#q42)
- [Q97 → Tune chunk size](#q97)
- [Q92 → Processing millions of records](#q92)

---

## Q97. How do you tune chunk size?

### 📝 One-Liner
Start with 500, benchmark 100/500/1000/2000, pick the sweet spot where throughput plateaus and memory stays safe.

### 🔑 Quick Answer
Chunk size = number of items processed per transaction. **Too small** (50): too many commits, high overhead. **Too large** (5000): high memory, long rollback on failure. **Sweet spot**: usually 500.  Benchmark with 100, 500, 1000, 2000 — find where increasing chunk size no longer significantly improves throughput. Consider: simple processing → larger (500-1000), complex processing → smaller (100-300), memory-constrained → smaller. *(500 se shuru karo, benchmark karo — jahan speed badhna band ho wahi ruko)*

### 📖 How It Works
```
Chunk Size Tradeoffs:

Small chunk (50):
  ✅ Low memory (~50 items in memory)
  ✅ Small rollback on failure (50 items max)
  ❌ 200 commits for 10K records (high overhead)
  ❌ 200 DB connections used

Large chunk (5000):
  ❌ High memory (~5000 items in memory)
  ❌ Large rollback on failure (5000 items lost)
  ✅ 2 commits for 10K records (low overhead)

Benchmark Results (typical):
  chunk=100  → 10K records in 12 sec  (100 commits)
  chunk=500  → 10K records in 4 sec   (20 commits) ← sweet spot
  chunk=1000 → 10K records in 3.5 sec (10 commits)
  chunk=2000 → 10K records in 3.3 sec (5 commits) + OOM risk
  chunk=5000 → 10K records in 3.2 sec + memory spike

  500→1000: only 12% gain but 2× memory
  → 500 is the sweet spot!

Rules of Thumb:
  Simple transform:     500-1000
  Complex processing:   100-300
  Memory-constrained:   100-200
  JPA entities:         100-300 (entity cache growth)
  External API in proc: 50-100  (limit blast radius)
```

### 🗣️ Answering Approach
"I start with chunk size 500 and benchmark with 100, 500, 1000, and 2000. The key is finding where increasing chunk size no longer significantly improves throughput. In my project, going from 100 to 500 gave us 3× improvement, but 500 to 1000 only gave 12% while doubling memory usage — so 500 was our sweet spot. For steps with complex processing or external API calls, I use smaller chunks (100-200) to limit the blast radius of failures. I also consider memory — JPA entities grow the persistence context, so I keep chunks smaller (100-300) when using JPA."

### 💻 Code
```java
// Benchmark different chunk sizes
@Bean
public Step benchmarkStep(JobRepository repo, PlatformTransactionManager tx) {
    int chunkSize = 500;  // start here, then try 100, 1000, 2000
    return new StepBuilder("processStep", repo)
            .<Order, Order>chunk(chunkSize, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .listener(new ChunkTimingListener())   // measure per-chunk time
            .build();
}

// Chunk timing listener for benchmarking
public class ChunkTimingListener implements ChunkListener {
    private long startTime;
    private int chunkCount = 0;

    @Override
    public void beforeChunk(ChunkContext context) {
        startTime = System.currentTimeMillis();
    }

    @Override
    public void afterChunk(ChunkContext context) {
        long duration = System.currentTimeMillis() - startTime;
        chunkCount++;
        log.info("Chunk {} completed in {} ms", chunkCount, duration);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Never go above 5000 — memory spikes and rollback becomes expensive *(5000 se upar kabhi mat jao)*
- JPA persistence context grows per chunk — clear with EntityManager.clear() per chunk
- External API calls in processor: small chunks (50-100) to limit retry scope
- Chunk size affects both memory AND transaction duration

### ⚡ Remember
- Start with **500**, benchmark 100/500/1000/2000 *(500 se shuru, benchmark karo)*
- Sweet spot = where speed plateaus
- Simple processing → 500-1000
- Complex/JPA/API → 100-300
- Never > 5000

### 🔗 Follow-ups
- [Q21 → Chunk processing concept](#q21)
- [Q96 → Optimize batch inserts](#q96)
- [Q98 → Avoid memory issues](#q98)

---

## Q98. How do you avoid memory issues?

### 📝 One-Liner
Use paging reader (constant memory), clear JPA cache per chunk, reduce chunk size, and monitor heap — #1 cause is JPA persistence context growth.

### 🔑 Quick Answer
6 common memory issues: **(1) JPA persistence context** — EntityManager holds ALL entities read so far → grows to 100K+ entities → OOM. Fix: clear EntityManager per chunk. **(2) Large chunk size** — 5000 items in memory at once. Fix: reduce to 500. **(3) ExecutionContext bloat** — storing too much data. Fix: only store positions/counters. **(4) Cursor reader buffering** — large fetch size. Fix: use paging reader. **(5) Reference cache too large** — HashMap with millions of entries. Fix: LRU cache. **(6) Too many partitions** — 100+ concurrent threads. Fix: limit to core count. *(Sabse bada issue JPA ka persistence context hai — har chunk ke baad clear karo)*

### 📖 How It Works
```
#1 JPA Persistence Context Leak:

  Chunk 1: EntityManager holds 500 entities    → 10 MB
  Chunk 2: EntityManager holds 1000 entities   → 20 MB
  Chunk 10: EntityManager holds 5000 entities  → 100 MB
  Chunk 200: EntityManager holds 100000 entities → 2 GB → OOM!

  FIX: Clear EntityManager after each chunk
  Chunk 1: 500 entities → process → write → CLEAR → 0 entities
  Chunk 2: 500 entities → process → write → CLEAR → 0 entities
  Memory stays constant at ~10 MB ✅

Memory Checklist:
  ☐ Paging reader (not cursor) → constant memory
  ☐ Clear JPA EntityManager per chunk → no leak
  ☐ Chunk size ≤ 500 → predictable memory
  ☐ ExecutionContext: only positions/counters → small
  ☐ Reference cache: LRU with max size → bounded
  ☐ Partition count: ≤ CPU cores → bounded threads
  ☐ JVM: -Xmx4g, G1GC → appropriate limits
```

### 🗣️ Answering Approach
"The number one cause of memory issues in Spring Batch is JPA persistence context growth. The EntityManager holds a reference to every entity read — after 200 chunks of 500 items, that's 100K entities consuming gigabytes of memory. The fix is implementing a ChunkListener that calls EntityManager.clear() after each chunk, keeping memory at a constant ~10MB. Other common issues include large chunk sizes, ExecutionContext bloat from storing data instead of just positions, and unbounded reference caches. In my project, we also configure JVM with -Xmx4g, G1GC, and HeapDumpOnOutOfMemoryError for diagnosis."

### 💻 Code
```java
// JPA EntityManager cleanup per chunk — critical!
@Component
public class JpaCacheClearListener implements ChunkListener {
    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public void afterChunk(ChunkContext context) {
        entityManager.flush();
        entityManager.clear();  // release all entities from persistence context
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        entityManager.clear();
    }
}

// Register in step
@Bean
public Step safeJpaStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("safeStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(jpaPagingReader())
            .writer(jpaWriter())
            .listener(new JpaCacheClearListener())  // clear per chunk
            .build();
}

// JVM flags for batch applications
// java -Xms2g -Xmx4g
//      -XX:+UseG1GC
//      -XX:+HeapDumpOnOutOfMemoryError
//      -XX:HeapDumpPath=/logs/heapdump.hprof
//      -jar batch-app.jar
```

### ⚠️ Pitfalls / Gotchas
- JPA persistence context is the silent killer — no error until OOM *(JPA context chupke se badhta hai — OOM tab pata chalta hai)*
- ExecutionContext is serialized to DB each commit — large data slows every chunk
- Cursor readers with large fetchSize buffer many rows in memory
- Multiple partitions × chunk size = total memory needed
- Always enable HeapDumpOnOutOfMemoryError in production

### 🎯 Tricky Interview Qs

**Q: We use JPA entities — how do we prevent memory leak?**
ChunkListener that calls entityManager.clear() after each chunk. This releases all entities from the first-level cache. Also consider switching to JdbcBatchItemWriter for the write side while keeping JPA for reads.

**Q: Our ExecutionContext keeps growing — why?**
You're storing data (not just positions) in ExecutionContext. It's serialized into DB every chunk commit. Store only counters and reader positions — never store entities or large collections.

### ⚡ Remember
- **#1 cause = JPA persistence context** → clear per chunk *(har chunk ke baad EntityManager.clear())*
- Paging reader = constant memory
- Chunk size ≤ 500
- ExecutionContext: positions only, not data
- JVM: -Xmx4g, G1GC, HeapDumpOnOutOfMemoryError

### 🔗 Follow-ups
- [Q97 → Tune chunk size](#q97)
- [Q35 → JpaPagingItemReader details](#q35)
- [Q92 → Processing millions efficiently](#q92)
