# 🔄 Chunk Processing — Q21 to Q30

---

## Q21. What is chunk processing in Spring Batch?

### 📝 One-Liner
Chunk processing reads N items one-at-a-time, processes them, then writes all N together in one transaction commit.

### 🔑 Quick Answer
Spring Batch reads items one by one until it reaches chunk size (say 500), processes each item, then writes the entire chunk in a single batch write and commits the transaction. One chunk = one transaction = one checkpoint. If something fails, only the current chunk rolls back — previous chunks are already committed and safe. *(Ek chunk matlab ek transaction — fail hua toh sirf woh chunk rollback hoga)*

### 📖 How It Works
```
Chunk Processing Flow:
┌─────────────────────────────────────────────────┐
│  BEGIN TRANSACTION                               │
│  ┌───────────┐   ┌───────────┐   ┌───────────┐ │
│  │  READ x N  │ → │ PROCESS xN│ → │ WRITE ALL │ │
│  │ (one by one)│  │(one by one)│  │ (one batch)│ │
│  └───────────┘   └───────────┘   └───────────┘ │
│  COMMIT TRANSACTION                              │
└─────────────────────────────────────────────────┘
        ↓ repeat until no more data
```

Three key benefits:
1. **Memory efficient** — only chunk-size items in RAM at a time *(sirf utne items memory mein jitna chunk size hai)*
2. **Transaction safety** — failed chunk rolls back, previous chunks safe
3. **Restartable** — Spring Batch saves checkpoint after each commit, restart picks up from failed chunk

### 🗣️ How to Say in Interview
"Chunk processing is the core execution model of Spring Batch. It reads items one at a time until it reaches the configured chunk size, processes each item individually, and then writes the entire chunk as a single batch operation within one transaction. In my project, we used chunk size of 500 for processing daily payment files — reading CSV records, validating and transforming them, then batch-inserting into the database. Each committed chunk acts as a checkpoint, so if a failure occurs, only the current chunk rolls back and the job can restart from that point."

### 💻 Code
```java
@Bean
public Step paymentStep(JobRepository jobRepository,
                        PlatformTransactionManager tx) {
    return new StepBuilder("paymentStep", jobRepository)
            .<RawPayment, ProcessedPayment>chunk(500, tx)  // 500 items per chunk
            .reader(csvReader())
            .processor(paymentProcessor())
            .writer(jdbcWriter())
            .build();
}
// One chunk = read 500 → process 500 → write 500 → COMMIT
// Then next chunk starts
```

### ⚠️ Pitfalls / Gotchas
- Chunk size is NOT page size — page size is how many rows the reader fetches per DB query, chunk size is how many items per transaction *(page size aur chunk size alag cheezein hain)*
- Setting chunk size too large → high memory + big rollback on failure
- Setting chunk size too small → too many commits, slow performance
- Chunk transaction includes BOTH metadata update and data write

### 🆚 vs. Comparison
| Aspect | Chunk Processing | Tasklet Processing |
|--------|-----------------|-------------------|
| Model | Read → Process → Write loop | Single `execute()` method |
| Transaction | Per chunk | Per tasklet call |
| Use Case | Large data processing | One-shot tasks (delete, move file) |
| Built-in Restart | Checkpoint after each chunk | Must handle manually |
| Memory | Only chunk-size in RAM | Depends on your code |

### 🎯 Tricky Interview Qs

**Q: Can a chunk have fewer items than chunk size?**
Yes — the last chunk will have fewer items if total records aren't evenly divisible. If you have 1050 records with chunk size 500: chunk 1 = 500, chunk 2 = 500, chunk 3 = 50.

**Q: What happens to already-committed chunks if job fails midway?**
They stay committed. Spring Batch never rolls back previously committed chunks. That's the whole point of checkpointing. *(Pehle commit ho chuka data safe hai — woh kabhi rollback nahi hota)*

### ⚡ Remember
- One chunk = one transaction = one checkpoint
- Read one-by-one, write all-at-once *(ek ek karke padho, saath mein likho)*
- Failed chunk → only that chunk rolls back
- Chunk size ≠ page size
- Last chunk can be partial (smaller than chunk size)

### 🔗 Follow-ups
- [Q22 → Internal working of chunk processing](#q22)
- [Q23 → Chunk size vs commit interval](#q23)
- [Q26 → Optimal chunk size](#q26)
- [Q79 → Tasklet vs Chunk](#q79)

---

## Q22. How does chunk processing work internally?

### 📝 One-Liner
Internally, Spring Batch runs a repeat loop — read N items, process each, collect into list, write list, commit, repeat until reader returns null.

### 🔑 Quick Answer
The `ChunkOrientedTasklet` drives the loop: it calls `reader.read()` in a loop until chunk size is reached or reader returns null (no more data). Each read item goes through `processor.process()` — if processor returns null, item is filtered out. All processed items are collected into a list, then `writer.write(list)` is called once, and the transaction commits. *(Reader null return kare toh data khatam — loop ruk jaata hai)*

### 📖 How It Works
```
Internal Pseudocode:
┌──────────────────────────────────────────────────┐
│ while (true) {                                    │
│   BEGIN TRANSACTION                               │
│   List<O> items = new ArrayList<>();              │
│                                                   │
│   for (int i = 0; i < chunkSize; i++) {          │
│     I input = reader.read();  // returns null     │
│     if (input == null) break;  // = no more data  │
│                                                   │
│     O output = processor.process(input);          │
│     if (output != null) {    // null = filtered   │
│       items.add(output);                          │
│     }                                             │
│   }                                               │
│                                                   │
│   if (items.isEmpty()) break;  // all done        │
│                                                   │
│   writer.write(items);  // ONE batch call         │
│   COMMIT TRANSACTION                              │
│   update metadata (execution context)             │
│ }                                                 │
└──────────────────────────────────────────────────┘
```

Key internals:
- **`SimpleChunkProvider`** — handles the read loop, calls `reader.read()` repeatedly
- **`SimpleChunkProcessor`** — calls `processor.process()` for each item, then `writer.write()` for the chunk
- **`ChunkOrientedTasklet`** — orchestrates provider + processor, manages transaction
- **Processor returning null** = item is FILTERED (tracked as `filterCount` in `StepExecution`)

### 🗣️ How to Say in Interview
"Internally, ChunkOrientedTasklet drives the chunk loop. SimpleChunkProvider reads items one at a time by calling reader.read() until either chunk size is reached or null is returned, meaning no more data. Each item passes through the processor — if it returns null, the item is filtered. SimpleChunkProcessor then collects all non-null results and calls writer.write() with the full list. After writing, the transaction commits and metadata is updated. In my project, I debugged a chunk processing issue by looking at filterCount in StepExecution — some records were being silently filtered because the processor returned null for invalid records."

### 💻 Code
```java
// Spring Batch internal classes (simplified)
// You don't write this — but knowing it helps in interviews

// ChunkOrientedTasklet.execute():
Chunk<I> inputs = chunkProvider.provide(contribution);   // reads N items
Chunk<O> outputs = chunkProcessor.process(contribution, inputs); // process + write

// SimpleChunkProvider.provide():
for (int i = 0; i < chunkSize; i++) {
    I item = reader.read();   // one item at a time
    if (item == null) break;  // no more data
    inputs.add(item);
}

// SimpleChunkProcessor.process():
for (I item : inputs) {
    O output = processor.process(item);
    if (output != null) {     // null = filtered
        outputs.add(output);
    }
}
writer.write(outputs.getItems());  // single batch write
```

### ⚠️ Pitfalls / Gotchas
- Processor returning null is NOT an error — it's intentional filtering *(null return karna error nahi, filtering hai)*
- `filterCount` tracks filtered items — check this when write count doesn't match read count
- Metadata update happens in a SEPARATE transaction — even if chunk fails, metadata is stored
- Reader must return null to signal end of data — infinite loop if it never returns null

### 🎯 Tricky Interview Qs

**Q: If processor filters 3 items from a chunk of 500, how many items does writer receive?**
497. The 3 null returns are excluded. `readCount = 500`, `filterCount = 3`, `writeCount = 497`.

**Q: Where does Spring Batch save progress?**
In `BATCH_STEP_EXECUTION_CONTEXT` table. After each chunk commits, execution context is updated with read count, write count, and commit count.

### ⚡ Remember
- `reader.read()` returning null = end of data *(null matlab data khatam)*
- `processor.process()` returning null = filter that item
- Three internal classes: ChunkProvider → ChunkProcessor → ChunkOrientedTasklet
- filterCount explains readCount - writeCount gap
- Metadata saved in separate transaction (always persisted)

### 🔗 Follow-ups
- [Q21 → What is chunk processing](#q21)
- [Q24 → Failure in middle of chunk](#q24)
- [Q54 → Filtering with processor](#q54)

---

## Q23. What is the difference between chunk size and commit interval?

### 📝 One-Liner
They are the SAME thing — chunk size IS the commit interval in Spring Batch.

### 🔑 Quick Answer
In Spring Batch, `chunk(500)` means both: process 500 items per chunk AND commit every 500 items. There is no separate "commit interval" config. The confusion comes from comparing with page size, which is completely different — page size is how many rows the reader fetches from the database per SQL query. *(Chunk size aur commit interval ek hi cheez hai — alag nahi)*

### 📖 How It Works
```
chunk(500) means:
├── Read 500 items     ← chunk size = 500
├── Process 500 items
├── Write 500 items
└── COMMIT             ← commit interval = 500  (same!)

pageSize(1000) means:
└── Reader fetches 1000 rows per SQL query (separate config)
    (reader may need 1, 2, or more pages to fill one chunk)
```

Three separate concepts:
| Concept | What It Controls | Config |
|---------|-----------------|--------|
| **Chunk Size** | Items per transaction | `.chunk(500, tx)` |
| **Commit Interval** | Same as chunk size | Same config |
| **Page Size** | Rows per DB query | `.pageSize(1000)` on reader |

Best practice: set page size ≤ chunk size for predictable behavior.

### 🗣️ How to Say in Interview
"Chunk size and commit interval are the same thing in Spring Batch. When you configure chunk(500), it means 500 items are processed per transaction and the commit happens after every 500 items. The common confusion is with page size, which is a completely different concept — it controls how many rows the reader fetches from the database per SQL query. In my project, we set chunk size to 500 and page size to 500 as well, keeping them aligned for predictable memory usage and transaction boundaries."

### 💻 Code
```java
@Bean
public Step orderStep(JobRepository jobRepository,
                      PlatformTransactionManager tx) {
    return new StepBuilder("orderStep", jobRepository)
            .<Order, Order>chunk(500, tx)        // chunk size = commit interval = 500
            .reader(pagingReader())               // reader has its own page size
            .processor(orderProcessor())
            .writer(orderWriter())
            .build();
}

@Bean
public JdbcPagingItemReader<Order> pagingReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)    // page size = rows per DB query (separate from chunk!)
            .rowMapper(new OrderRowMapper())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Don't confuse chunk size with page size — interviewers specifically test this *(ye sabse common confusion hai)*
- If page size > chunk size, extra rows fetched but not processed in current chunk → wasted memory
- There's no separate `commitInterval()` method in modern Spring Batch (Spring Batch 5+)
- Old Spring Batch 3.x had a separate `commit-interval` XML attribute — same concept, different syntax

### 🎯 Tricky Interview Qs

**Q: Can you have different chunk size and commit interval?**
No. In Spring Batch, they are always the same. `chunk(N)` sets both.

**Q: If chunk size is 500 and page size is 1000, what happens?**
Reader fetches 1000 rows in one SQL query but only 500 are processed per chunk. The remaining 500 stay buffered and are used in the next chunk without another SQL query.

### ⚡ Remember
- Chunk size = Commit interval (SAME THING) *(dono ek hi cheez hai)*
- Page size is DIFFERENT — controls DB fetch, not transaction
- Best practice: page size ≤ chunk size
- No separate `commitInterval()` in Spring Batch 5

### 🔗 Follow-ups
- [Q21 → What is chunk processing](#q21)
- [Q26 → Optimal chunk size](#q26)
- [Q34 → JdbcPagingItemReader page size](#q34)

---

## Q24. What happens if a failure occurs in the middle of a chunk?

### 📝 One-Liner
The current chunk transaction rolls back entirely, but all previously committed chunks remain safe.

### 🔑 Quick Answer
If any exception occurs during read, process, or write of a chunk, the entire current chunk's transaction rolls back — all-or-nothing for that chunk. Previous chunks that already committed are safe and won't be touched. Without fault tolerance, the step and job fail immediately. With skip/retry configured, Spring Batch can skip the bad item or retry the operation. On restart, the job resumes from the failed chunk. *(Current chunk rollback hota hai, pehle wale safe — restart mein wahi se shuru)*

### 📖 How It Works
```
Processing 2500 records with chunk size 500:

Chunk 1 (items 1-500)    → ✅ COMMITTED (safe forever)
Chunk 2 (items 501-1000) → ✅ COMMITTED (safe forever)
Chunk 3 (items 1001-1500)→ ❌ EXCEPTION at item 1234
                            → ROLLBACK chunk 3 entirely
                            → Items 1001-1500 all rolled back
                            
On RESTART:
→ Resumes from item 1001 (chunk 3) — chunks 1 & 2 not re-processed
```

Without fault tolerance:
- Exception → rollback chunk → step FAILED → job FAILED

With fault tolerance (skip):
```java
// Skip up to 100 bad records
.faultTolerant()
.skip(ValidationException.class)
.skipLimit(100)
```
- Exception → skip that item → continue processing rest of chunk
- Tracked as `skipCount` in StepExecution

With fault tolerance (retry):
```java
// Retry transient errors up to 3 times
.faultTolerant()
.retry(TimeoutException.class)
.retryLimit(3)
```

### 🗣️ How to Say in Interview
"When a failure occurs mid-chunk, the entire chunk's transaction rolls back — it's all-or-nothing for that chunk. Previously committed chunks remain safe since they're already persisted. Without fault tolerance, the job fails immediately. In my project, we configured skip logic for validation exceptions with a skip limit of 100, so individual bad records were skipped and logged without failing the entire job. We also used SkipListener to write skipped records to an error file for manual review. On restart, Spring Batch automatically resumed from the last uncommitted chunk using checkpoint data from the metadata tables."

### 💻 Code
```java
@Bean
public Step resilientStep(JobRepository jobRepository,
                          PlatformTransactionManager tx) {
    return new StepBuilder("resilientStep", jobRepository)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(orderReader())
            .processor(orderProcessor())
            .writer(orderWriter())
            .faultTolerant()
            .skip(ValidationException.class)     // skip validation errors
            .skip(FlatFileParseException.class)   // skip bad CSV lines
            .skipLimit(100)                       // max 100 skips total
            .retry(DeadlockLoserDataAccessException.class) // retry deadlocks
            .retryLimit(3)
            .listener(skipListener())             // log skipped items
            .build();
}

@Bean
public SkipListener<Order, ProcessedOrder> skipListener() {
    return new SkipListener<>() {
        @Override
        public void onSkipInRead(Throwable t) {
            log.warn("Skipped on read: {}", t.getMessage());
        }
        @Override
        public void onSkipInProcess(Order item, Throwable t) {
            log.warn("Skipped on process: item={}, error={}", item.getId(), t.getMessage());
        }
        @Override
        public void onSkipInWrite(ProcessedOrder item, Throwable t) {
            log.warn("Skipped on write: item={}, error={}", item.getId(), t.getMessage());
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- Without `.faultTolerant()`, even one exception kills the job *(bina fault tolerance ke ek error se poora job fail)*
- Skip limit is TOTAL across all chunks, not per chunk
- When skip happens during WRITE, Spring Batch enters "scan mode" — re-processes items one-by-one to find the bad one
- `skipLimit(-1)` = unlimited skips (dangerous — could skip all data silently)
- Always add SkipListener to log what was skipped

### 🎯 Tricky Interview Qs

**Q: If chunk fails and job restarts, does it re-read already committed chunks?**
No. Spring Batch stores the read count in execution context. On restart, the reader skips ahead to the last checkpoint position.

**Q: What is "scan mode" in Spring Batch?**
When a write fails with skip enabled, Spring Batch can't know which item caused it (since write is batched). So it enters scan mode — rolls back, then re-writes items one-by-one to identify and skip the bad one. *(Write fail hone par ek ek item likhta hai taaki pata chale kaunsa kharab hai)*

### ⚡ Remember
- Current chunk rolls back, previous chunks safe
- No fault tolerance → one error kills job
- Skip = for bad data, Retry = for transient errors *(skip data ke liye, retry network issues ke liye)*
- Scan mode: write fail → re-write one-by-one to find bad item
- Always add SkipListener for audit trail

### 🔗 Follow-ups
- [Q25 → Transaction management in chunks](#q25)
- [Q70 → Error handling strategies](#q70)
- [Q22 → Internal chunk working](#q22)

---

## Q25. How does Spring Batch manage transactions in chunk processing?

### 📝 One-Liner
Each chunk runs in its own transaction — commit after success, rollback on failure — with metadata updates in a separate transaction.

### 🔑 Quick Answer
Spring Batch wraps each chunk in a transaction using `PlatformTransactionManager`. All reads, processing, and writes within a chunk happen in ONE transaction. On success → commit. On failure → rollback. The key insight: metadata updates (StepExecution, ExecutionContext) run in a SEPARATE transaction that always commits — even if the chunk fails. This ensures restart information is never lost. *(Metadata alag transaction mein save hota hai — chunk fail bhi ho toh restart info safe rehta hai)*

### 📖 How It Works
```
Transaction Architecture:

Chunk Transaction (DATA):
┌─────────────────────────────────┐
│  BEGIN TX-1 (data)              │
│  ├── read 500 items             │
│  ├── process 500 items          │
│  ├── write 500 items            │
│  └── COMMIT TX-1    ✅          │
└─────────────────────────────────┘
         ↓
Metadata Transaction (ALWAYS commits):
┌─────────────────────────────────┐
│  BEGIN TX-2 (metadata)          │
│  ├── update BATCH_STEP_EXECUTION│
│  ├── update readCount, writeCount│
│  ├── update EXECUTION_CONTEXT   │
│  └── COMMIT TX-2    ✅          │
└─────────────────────────────────┘

If chunk FAILS:
  TX-1 → ROLLBACK ❌ (data rolled back)
  TX-2 → COMMIT ✅  (metadata still saved!)
  → This is how restart knows where to resume
```

Transaction manager options:
- **`DataSourceTransactionManager`** — for pure JDBC (faster, recommended for batch)
- **`JpaTransactionManager`** — for JPA entities (adds ORM overhead but manages entity lifecycle)

### 🗣️ How to Say in Interview
"Spring Batch manages transactions at the chunk level. Each chunk runs within its own transaction — on success, it commits; on failure, it rolls back. What's important is that metadata updates happen in a separate transaction that always commits, even when the chunk fails. This is what makes restart possible — the job knows exactly where it failed because the metadata transaction preserved the execution state. In my project, we used DataSourceTransactionManager for batch steps since we were doing pure JDBC operations, and it was significantly faster than JpaTransactionManager since it avoided ORM overhead."

### 💻 Code
```java
@Configuration
public class BatchConfig {

    @Bean
    public PlatformTransactionManager transactionManager(DataSource dataSource) {
        // For JDBC-based batch — recommended for performance
        return new DataSourceTransactionManager(dataSource);
    }

    @Bean
    public Step transactionalStep(JobRepository jobRepository,
                                  PlatformTransactionManager tx) {
        return new StepBuilder("transactionalStep", jobRepository)
                .<Order, Order>chunk(500, tx)  // tx manages chunk transaction
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .transactionAttribute(txAttribute()) // optional: customize
                .build();
    }

    // Optional: customize transaction attributes
    private TransactionAttribute txAttribute() {
        DefaultTransactionAttribute attr = new DefaultTransactionAttribute();
        attr.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
        attr.setTimeout(300); // 5 min timeout per chunk
        return attr;
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Using JPA? Must use `JpaTransactionManager` — `DataSourceTransactionManager` won't flush JPA context *(JPA use kar rahe ho toh JpaTransactionManager zaruri hai)*
- Transaction timeout applies PER CHUNK, not per job
- If your writer calls an external API (non-transactional), rollback won't undo the API call
- Don't mix `@Transactional` on your reader/writer beans — Spring Batch manages transactions itself

### 🎯 Tricky Interview Qs

**Q: If chunk fails, how does Spring Batch know where to restart?**
Because metadata updates run in a separate transaction that always commits. Even when the chunk data rolls back, the execution context (with read count, commit count) is persisted.

**Q: Can you use two different transaction managers — one for data, one for metadata?**
Yes. Spring Batch 5 supports different transaction managers for JobRepository (metadata) and Step (data). Useful when metadata DB and business DB are different.

### ⚡ Remember
- One chunk = one transaction (commit or rollback)
- Metadata = SEPARATE transaction (always commits) *(metadata hamesha save hota hai)*
- DataSourceTransactionManager for JDBC, JpaTransactionManager for JPA
- Don't add `@Transactional` on batch components — framework handles it
- External API calls in writer = NOT rollback-safe

### 🔗 Follow-ups
- [Q24 → Failure in middle of chunk](#q24)
- [Q63 → Transaction isolation in batch](#q63)
- [Q28 → Job restart mechanism](#q28)

---

## Q26. What is the optimal chunk size?

### 📝 One-Liner
100–500 is the typical sweet spot, but always benchmark with your actual data.

### 🔑 Quick Answer
There's no universal "best" chunk size — it depends on item complexity, memory, DB capacity, and error tolerance. Small chunks (50-100) mean low memory and fast rollback but many commits. Large chunks (1000+) mean fewer commits but high memory and large rollbacks. For most projects, 100-500 works well. Always benchmark — increase chunk size until throughput plateaus or memory becomes an issue. *(Chhota chunk = zyada commits, bada chunk = zyada memory — balance dhundho benchmarking se)*

### 📖 How It Works
```
Chunk Size Tradeoff Table:

| Chunk Size | Memory  | Commits/1M rows | Rollback Risk | Best For              |
|------------|---------|-----------------|---------------|----------------------|
| 10         | Very Low| 100,000         | Minimal       | Testing only         |
| 50         | Low     | 20,000          | Small         | Complex processing   |
| 100        | Medium  | 10,000          | Medium        | API calls per item   |
| 500        | Medium  | 2,000           | Medium-High   | DB read-write (default)|
| 1000       | High    | 1,000           | High          | Simple transformations|
| 5000       | V.High  | 200             | Very High     | Simple inserts only  |
```

Factors to consider:
1. **Item size** — large objects (BLOBs) → smaller chunk
2. **Processing complexity** — API calls → smaller chunk (100)
3. **Write type** — batch INSERT → larger chunk, API write → smaller
4. **Error tolerance** — need to skip items → smaller (less scan-mode overhead)
5. **Memory available** — limited heap → smaller chunk

### 🗣️ How to Say in Interview
"There's no one-size-fits-all chunk size. I follow a benchmarking approach — start with 500 for typical database read-write operations, then test with 100, 250, 1000 and measure throughput and memory. In my project, we found 500 was optimal for our payment processing — going to 1000 only improved throughput by 3% but doubled memory usage. For steps that called external APIs per item, we used chunk size of 100 to keep transaction duration short and reduce retry scope on failures."

### 💻 Code
```java
// Benchmarking different chunk sizes
// Run same job with different sizes, measure time and memory

// For typical DB operations
.<Order, Order>chunk(500, tx)  // Good default

// For complex processing with API calls
.<Order, EnrichedOrder>chunk(100, tx)  // Smaller = faster rollback

// For simple data migration (insert-only)
.<Record, Record>chunk(1000, tx)  // Larger = fewer commits

// Monitor with StepExecution metrics
@Bean
public StepExecutionListener chunkMonitor() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution se) {
            log.info("Read: {}, Written: {}, Commits: {}, Duration: {}s",
                se.getReadCount(), se.getWriteCount(), 
                se.getCommitCount(), 
                Duration.between(se.getStartTime(), se.getEndTime()).getSeconds());
            return se.getExitStatus();
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- Don't guess — always benchmark with production-like data *(andaaze se mat karo, benchmark karo)*
- Large chunk + skip enabled = slow scan mode (re-processes entire chunk one-by-one on write failure)
- Large chunk + JPA = memory issues from first-level cache (flush/clear needed)
- Network latency matters: if writer does remote calls, smaller chunks reduce blast radius

### ⚡ Remember
- **Default start**: 500 for DB operations, 100 for API-heavy steps
- Small chunk = many commits, low memory, fast rollback
- Large chunk = few commits, high memory, big rollback *(bada chunk = bada risk)*
- Always BENCHMARK, never assume
- Monitor: readCount, writeCount, commitCount, duration

### 🔗 Follow-ups
- [Q27 → Memory issues with large chunks](#q27)
- [Q23 → Chunk size vs page size](#q23)
- [Q92 → Performance tuning](#q92)

---

## Q27. How do you handle memory issues with large chunks?

### 📝 One-Liner
Reduce chunk size, use paging readers, clear JPA cache, and consider partitioning for very large datasets.

### 🔑 Quick Answer
Memory issues with large chunks happen because all items in a chunk stay in memory until the write completes. Solutions: (1) reduce chunk size, (2) use paging reader instead of cursor reader, (3) clear JPA first-level cache after each chunk using `EntityManager.clear()`, (4) avoid loading entire file into memory, (5) consider partitioning to split data across multiple threads. *(Chunk ke saare items memory mein rehte hain jab tak write nahi ho jaata — chunk chhota karo ya paging reader use karo)*

### 📖 How It Works
```
Memory Usage per Chunk:
┌──────────────────────────────────┐
│ chunk(5000)                      │
│ ├── 5000 raw items (reader)      │ ← Input objects
│ ├── 5000 processed items         │ ← Output objects  
│ ├── JPA first-level cache        │ ← Entity cache (if JPA)
│ └── JDBC batch buffer            │ ← Write buffer
│ = ~4x items × object size in RAM │
└──────────────────────────────────┘

Solutions:
1. Reduce chunk size → chunk(500) instead of chunk(5000)
2. Paging reader → releases connection between pages
3. JPA cache clear → EntityManager.clear() in ChunkListener
4. Streaming → process without loading all into memory
5. Partitioning → split data across threads/processes
```

### 🗣️ How to Say in Interview
"Memory issues in chunk processing typically come from three sources: the items themselves being too large or too many per chunk, JPA first-level cache accumulating entities, and batch writer buffers. In my project, we initially used chunk size of 2000 with JPA and ran into OutOfMemoryErrors. We solved it by reducing chunk size to 500, switching from cursor to paging reader, and adding a ChunkListener that called EntityManager.clear() after each chunk to flush the JPA cache. For our largest dataset of 50 million records, we also used partitioning to split the work across 8 threads."

### 💻 Code
```java
// Solution 1: Clear JPA cache after each chunk
@Component
public class JpaCacheClearListener implements ChunkListener {
    
    @PersistenceContext
    private EntityManager entityManager;
    
    @Override
    public void afterChunk(ChunkContext context) {
        entityManager.clear();  // release all cached entities
    }
}

// Solution 2: Use paging reader (releases connection between pages)
@Bean
public JdbcPagingItemReader<Order> pagingReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            .selectClause("SELECT *")
            .fromClause("FROM orders WHERE status = 'PENDING'")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)    // fetch 500 rows per query
            .rowMapper(new OrderRowMapper())
            .build();
}

// Solution 3: Partitioning for very large datasets
@Bean
public Step partitionedStep(JobRepository jobRepository,
                            PlatformTransactionManager tx) {
    return new StepBuilder("partitionedStep", jobRepository)
            .partitioner("workerStep", rangePartitioner())
            .step(workerStep(jobRepository, tx))
            .gridSize(8)      // 8 parallel threads
            .taskExecutor(new SimpleAsyncTaskExecutor())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Cursor reader holds DB connection for entire step — can timeout + doesn't release memory between pages *(cursor reader ek hi connection pakad ke rakhta hai)*
- `EntityManager.clear()` detaches ALL entities — don't do this if other code expects managed entities
- Increasing JVM heap is a band-aid, not a fix — will just delay the OOM
- Partitioning adds complexity — use only for genuinely large datasets (10M+ records)

### ⚡ Remember
- Reduce chunk size first (simplest fix)
- Paging > Cursor for memory efficiency
- Clear JPA cache in ChunkListener *(JPA cache har chunk ke baad saaf karo)*
- Don't just increase heap — fix the root cause
- Partitioning for 10M+ records

### 🔗 Follow-ups
- [Q26 → Optimal chunk size](#q26)
- [Q36 → Cursor vs Paging reader](#q36)
- [Q84 → Partitioning in Spring Batch](#q84)

---

## Q28. What is the role of PlatformTransactionManager in chunk processing?

### 📝 One-Liner
PlatformTransactionManager starts, commits, and rolls back the transaction for each chunk.

### 🔑 Quick Answer
`PlatformTransactionManager` is the Spring interface that controls transaction boundaries for each chunk. Spring Batch calls `getTransaction()` before chunk starts, `commit()` after successful write, and `rollback()` on failure. You pass it to the `chunk()` builder. Two common implementations: `DataSourceTransactionManager` for pure JDBC (faster) and `JpaTransactionManager` for JPA (handles entity lifecycle). *(Har chunk ke liye transaction shuru karna, commit karna, rollback karna — yeh sab PlatformTransactionManager karta hai)*

### 📖 How It Works
```
PlatformTransactionManager Lifecycle per Chunk:

  tx.getTransaction()  → BEGIN TRANSACTION
       ↓
  Read → Process → Write
       ↓
  Success? → tx.commit()    → COMMIT
  Failure? → tx.rollback()  → ROLLBACK
```

Choosing the right one:
| If You Use... | Transaction Manager | Why |
|---------------|-------------------|-----|
| JDBC only | `DataSourceTransactionManager` | Lightweight, fast |
| JPA entities | `JpaTransactionManager` | Manages EntityManager flush/clear |
| Multiple datasources | `ChainedTransactionManager` or `JtaTransactionManager` | Coordinates multiple resources |

### 🗣️ How to Say in Interview
"PlatformTransactionManager is the Spring abstraction that manages transaction boundaries for each chunk. Spring Batch calls it to begin a transaction before processing starts, commit after successful write, and rollback on failure. In my project, we used DataSourceTransactionManager since our batch steps used plain JDBC operations — it's faster than JpaTransactionManager because it doesn't have to manage entity state. For a step that wrote to two different databases, we used JtaTransactionManager to coordinate distributed transactions."

### 💻 Code
```java
@Configuration
public class BatchTransactionConfig {

    // Option 1: JDBC — lightweight, fast (recommended for batch)
    @Bean
    public PlatformTransactionManager jdbcTxManager(DataSource dataSource) {
        return new DataSourceTransactionManager(dataSource);
    }

    // Option 2: JPA — manages entity lifecycle
    @Bean
    public PlatformTransactionManager jpaTxManager(EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }

    // Pass to chunk builder
    @Bean
    public Step step(JobRepository jobRepository,
                     PlatformTransactionManager tx) {
        return new StepBuilder("step", jobRepository)
                .<Order, Order>chunk(500, tx)  // tx used for every chunk
                .reader(reader())
                .writer(writer())
                .build();
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Using JPA writer with `DataSourceTransactionManager` → entities won't flush properly *(JPA ke saath DataSourceTxManager mat use karo)*
- Batch 5 requires passing TransactionManager explicitly to `chunk()` — Batch 4 auto-detected it
- Transaction timeout in tx manager applies per chunk, not per job
- Using `@Transactional` on batch components conflicts with Spring Batch's transaction management

### ⚡ Remember
- JDBC → DataSourceTransactionManager (fast, lightweight)
- JPA → JpaTransactionManager (handles flush/clear)
- Must pass explicitly in Spring Batch 5: `.chunk(500, tx)` *(Batch 5 mein tx zaruri hai chunk builder mein)*
- Don't mix @Transactional with batch steps
- One transaction per chunk, not per job

### 🔗 Follow-ups
- [Q25 → Transaction management details](#q25)
- [Q43 → JpaItemWriter needs JpaTransactionManager](#q43)
- [Q63 → Transaction isolation settings](#q63)

---

## Q29. Can you have different chunk sizes for different steps?

### 📝 One-Liner
Yes — each step has its own chunk size configured independently.

### 🔑 Quick Answer
Absolutely. Each step in a job is configured independently with its own chunk size, reader, processor, writer, and even transaction manager. You can have Step 1 with chunk(100) for API-heavy processing and Step 2 with chunk(1000) for simple data migration — all in the same job. *(Har step ka apna chunk size hota hai — ek job mein alag alag ho sakte hain)*

### 📖 How It Works
```
Job "dailyBatch":
├── Step 1: "validateOrders"    → chunk(100)   ← calls external API
├── Step 2: "processPayments"   → chunk(500)   ← normal DB operations  
├── Step 3: "archiveRecords"    → chunk(2000)  ← simple INSERT
└── Step 4: "sendNotifications" → chunk(50)    ← email per item
```

Each step is a self-contained unit with its own:
- Chunk size
- Reader/Processor/Writer
- Transaction manager (can be different)
- Fault tolerance config
- Skip/retry logic

### 🗣️ How to Say in Interview
"Yes, each step in a Spring Batch job has its own independent chunk size. In my project, we had a job with three steps — the first step validated records against an external API with chunk size 100 to limit concurrent API calls, the second step processed payments with chunk size 500 for optimal database throughput, and the third step archived records with chunk size 2000 since it was simple inserts with no processing. Each step's chunk size was tuned based on benchmarking for that specific operation."

### 💻 Code
```java
@Bean
public Job dailyBatchJob(JobRepository jobRepository,
                         Step validateStep, Step processStep, Step archiveStep) {
    return new JobBuilder("dailyBatchJob", jobRepository)
            .start(validateStep)
            .next(processStep)
            .next(archiveStep)
            .build();
}

@Bean
public Step validateStep(JobRepository jobRepository, PlatformTransactionManager tx) {
    return new StepBuilder("validateStep", jobRepository)
            .<Order, ValidatedOrder>chunk(100, tx)   // small: calls API
            .reader(orderReader())
            .processor(apiValidator())
            .writer(validatedOrderWriter())
            .build();
}

@Bean
public Step processStep(JobRepository jobRepository, PlatformTransactionManager tx) {
    return new StepBuilder("processStep", jobRepository)
            .<ValidatedOrder, ProcessedOrder>chunk(500, tx)  // medium: DB ops
            .reader(validatedReader())
            .processor(paymentProcessor())
            .writer(paymentWriter())
            .build();
}

@Bean
public Step archiveStep(JobRepository jobRepository, PlatformTransactionManager tx) {
    return new StepBuilder("archiveStep", jobRepository)
            .<ProcessedOrder, ArchivedOrder>chunk(2000, tx)  // large: simple insert
            .reader(processedReader())
            .writer(archiveWriter())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Don't use the same chunk size everywhere — tune per step based on workload *(har step ke liye alag chunk size benchmark karo)*
- Steps run sequentially by default — next step starts only after current completes
- Different chunk sizes don't affect step ordering or flow

### ⚡ Remember
- Each step = independent chunk size configuration
- Tune based on step's specific workload *(API calls → chhota, simple insert → bada)*
- Steps run sequentially in a job
- Can even have different transaction managers per step

### 🔗 Follow-ups
- [Q26 → Optimal chunk size guidelines](#q26)
- [Q84 → Parallel step execution](#q84)
- [Q21 → Chunk processing basics](#q21)

---

## Q30. What is ChunkListener and how is it used?

### 📝 One-Liner
ChunkListener provides callbacks before and after each chunk, and on chunk errors — useful for logging, metrics, and resource cleanup.

### 🔑 Quick Answer
`ChunkListener` is an interface with three callbacks: `beforeChunk()` (runs before each chunk starts), `afterChunk()` (runs after each chunk commits), and `afterChunkError()` (runs when chunk fails). Common uses: logging progress, clearing caches, sending metrics, and resource cleanup. You register it with `.listener()` on the step builder. *(Har chunk ke pehle, baad, aur error pe callback milta hai)*

### 📖 How It Works
```
Chunk Lifecycle with Listener:

  listener.beforeChunk()     ← Before chunk starts
       ↓
  BEGIN TRANSACTION
  ├── Read → Process → Write
  └── COMMIT
       ↓
  listener.afterChunk()      ← After successful commit

  On ERROR:
  ├── ROLLBACK
  └── listener.afterChunkError() ← After rollback
```

ChunkListener vs other listeners:
| Listener | Scope | Methods |
|----------|-------|---------|
| ChunkListener | Per chunk | beforeChunk, afterChunk, afterChunkError |
| StepExecutionListener | Per step | beforeStep, afterStep |
| JobExecutionListener | Per job | beforeJob, afterJob |
| SkipListener | Per skipped item | onSkipInRead/Process/Write |
| ItemReadListener | Per read | beforeRead, afterRead, onReadError |

### 🗣️ How to Say in Interview
"ChunkListener provides lifecycle callbacks for each chunk — beforeChunk, afterChunk, and afterChunkError. In my project, I used a ChunkListener for two purposes: first, to log progress after every chunk showing how many records were processed so far, and second, to clear the JPA EntityManager cache after each chunk to prevent memory issues. I also used afterChunkError to send an alert to our monitoring system when a chunk failed, so the ops team could investigate immediately."

### 💻 Code
```java
@Component
public class ProgressChunkListener implements ChunkListener {

    private static final Logger log = LoggerFactory.getLogger(ProgressChunkListener.class);

    @PersistenceContext
    private EntityManager entityManager;

    @Override
    public void beforeChunk(ChunkContext context) {
        log.debug("Starting chunk #{}", 
            context.getStepContext().getStepExecution().getCommitCount() + 1);
    }

    @Override
    public void afterChunk(ChunkContext context) {
        StepExecution se = context.getStepContext().getStepExecution();
        log.info("Chunk committed — Total read: {}, written: {}, filtered: {}, skipped: {}",
            se.getReadCount(), se.getWriteCount(), 
            se.getFilterCount(), se.getSkipCount());
        
        // Clear JPA cache to prevent memory issues
        entityManager.clear();
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        StepExecution se = context.getStepContext().getStepExecution();
        log.error("Chunk FAILED at read count: {} — will rollback", se.getReadCount());
        // Send alert to monitoring system
    }
}

// Register on step
@Bean
public Step monitoredStep(JobRepository jobRepository, PlatformTransactionManager tx,
                          ProgressChunkListener chunkListener) {
    return new StepBuilder("monitoredStep", jobRepository)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .listener(chunkListener)  // register ChunkListener
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- `afterChunk()` runs AFTER commit — if listener throws exception, chunk data is already committed *(afterChunk mein error aaye toh data toh commit ho chuka hota hai)*
- `beforeChunk()` runs INSIDE the chunk transaction
- `afterChunkError()` gets the exception from `ChunkContext.getAttribute(ChunkListener.ROLLBACK_EXCEPTION_KEY)`
- Don't do heavy work in listeners — they run for EVERY chunk

### 🎯 Tricky Interview Qs

**Q: Does afterChunk run inside or outside the transaction?**
Outside. The transaction commits first, then afterChunk runs. So if afterChunk throws, the chunk data is safe.

**Q: Can you use @Annotation-based listener instead of implementing interface?**
Yes. Use `@BeforeChunk`, `@AfterChunk`, `@AfterChunkError` on any POJO methods. Spring Batch detects them automatically when registered with `.listener()`.

### ⚡ Remember
- Three callbacks: beforeChunk, afterChunk, afterChunkError
- afterChunk runs AFTER commit (data safe even if listener fails)
- Great for: progress logging, cache clearing, metrics *(progress log, cache saaf, metrics bhejo)*
- Can use annotations: `@BeforeChunk`, `@AfterChunk`, `@AfterChunkError`
- Don't do heavy work — runs every chunk

### 🔗 Follow-ups
- [Q22 → Internal chunk lifecycle](#q22)
- [Q27 → Clear JPA cache in listener](#q27)
- [Q77 → Step listeners](#q77)
