# 🟡 Spring Batch — Chunk Processing (Q21–Q30)

> **How to use this guide:**  
> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q21"></a>

## Q21. What is chunk processing in Spring Batch?

### 🔑 Quick Answer

> Chunk processing is Spring Batch's core model — it reads items **one at a time**, processes each one, then writes the whole batch (chunk) **together in one transaction**. Read N → Process N → Write N → Commit. Repeat.

### 📖 Step-by-Step Explanation

**Step 1 — Visualize one chunk (chunk size = 3):**

```
── Chunk 1 ─────────────────────────────────────
  reader.read()  → item1  → processor.process(item1)  ─┐
  reader.read()  → item2  → processor.process(item2)  ─┤→ writer.write([item1, item2, item3])
  reader.read()  → item3  → processor.process(item3)  ─┘
                                                            → COMMIT ✅

── Chunk 2 ─────────────────────────────────────
  reader.read()  → item4  → processor.process(item4)  ─┐
  reader.read()  → item5  → processor.process(item5)  ─┤→ writer.write([item4, item5, item6])
  reader.read()  → item6  → processor.process(item6)  ─┘
                                                            → COMMIT ✅

── Chunk 3 ─────────────────────────────────────
  reader.read()  → item7  → processor.process(item7)  ─┐
  reader.read()  → null   ← END OF DATA                │→ writer.write([item7])
                                                            → COMMIT ✅ → STEP DONE
```

**Step 2 — Key insight: Reader is one-at-a-time, Writer is batch:**

| Component | How many items at once? |
|-----------|----------------------|
| Reader | ONE at a time (returns single item) |
| Processor | ONE at a time (transforms single item) |
| Writer | ALL at once (receives the whole chunk) |

**Step 3 — Why chunks? Why not process all at once?**

```
10 million records ALL at once:
  ❌ Out of memory (10M objects in RAM)
  ❌ One giant transaction (if fails, ALL rolls back)
  ❌ No checkpoint (restart from beginning)

10 million records in CHUNKS of 500:
  ✅ Only 500 in memory at a time
  ✅ Small transactions (if fails, only 500 lost)
  ✅ Checkpoint every 500 (restart from last committed chunk)
```

**Step 4 — One chunk = One transaction:**

```
BEGIN TRANSACTION
  Read 500 items
  Process 500 items
  Write 500 items
COMMIT                  ← This is the checkpoint

If anything fails BEFORE commit → ROLLBACK only this chunk
Previously committed chunks are SAFE
```

### 🗣️ How to Explain in Interview

> *"Chunk processing is the heart of Spring Batch. Instead of processing all records at once, it works in chunks — say 500 records at a time. It reads one item at a time from the source, processes each one, and when it has 500 processed items, it writes them all to the destination in one batch and commits the transaction. Then it moves to the next 500. This gives three big advantages: memory efficiency because only 500 items are in memory at once; transaction safety because if something fails, only the current chunk rolls back, not the previous ones; and restartability because after each commit, the position is checkpointed."*

### 💻 Code Example

```java
@Bean
public Step processOrdersStep(JobRepository repo,
                               PlatformTransactionManager tx) {
    return new StepBuilder("processOrders", repo)
            .<Order, ProcessedOrder>chunk(500, tx)  // Chunk size = 500
            .reader(orderReader())                   // Read from CSV
            .processor(orderProcessor())             // Validate + transform
            .writer(orderWriter())                   // Write to database
            .build();
}
```

**What happens at runtime:**
1. `reader.read()` called 500 times → returns 500 Order objects (one at a time)
2. `processor.process()` called 500 times → returns 500 ProcessedOrder objects
3. `writer.write()` called ONCE → receives list of 500 ProcessedOrder objects → batch INSERT
4. Transaction COMMIT → checkpoint saved to database
5. Repeat steps 1-4 until reader returns `null`

### ⚡ Key Points to Remember

1. **Read one-at-a-time**, **Write all-at-once** per chunk
2. **One chunk = One transaction = One commit**
3. Failure → only current chunk rolls back
4. Checkpoint saved after every commit → enables restart
5. Chunk size is configurable → impacts memory, speed, and transaction size

### 🔗 Follow-up Questions
- *"How does it work internally?"* → See Q22
- *"What's the optimal chunk size?"* → See Q26
- *"What if a chunk fails?"* → See Q24

---

<a id="q22"></a>

## Q22. How does chunk processing work internally?

### 🔑 Quick Answer

> Internally, Spring Batch runs a **repeat loop**: open transaction → call `reader.read()` N times to fill a chunk → call `processor.process()` on each → pass the list to `writer.write()` → commit → repeat until reader returns null.

### 📖 Step-by-Step Explanation

**Step 1 — Internal pseudocode (this is what Spring Batch does):**

```
Step starts executing
│
└── REPEAT until reader returns null:
    │
    ├── 1. BEGIN TRANSACTION
    │
    ├── 2. Create empty List<OutputType> processedItems = []
    │
    ├── 3. REPEAT (chunkSize times):
    │   │
    │   ├── item = reader.read()
    │   │
    │   ├── if item == null → BREAK (no more data)
    │   │
    │   ├── processedItem = processor.process(item)
    │   │
    │   ├── if processedItem != null → processedItems.add(processedItem)
    │   │   (if null → item is FILTERED, not added)
    │   │
    │   └── continue loop
    │
    ├── 4. writer.write(processedItems)    ← Whole chunk at once!
    │
    ├── 5. COMMIT TRANSACTION
    │
    ├── 6. Update StepExecution counts (readCount, writeCount, etc.)
    │
    ├── 7. Save ExecutionContext to DB (checkpoint!)
    │
    └── NEXT CHUNK
```

**Step 2 — Important detail: processor.process() returns null means FILTER, not error:**

```
Chunk processing with filtering (chunk size = 3):

  read() → Employee{id=1, active=true}   → process() → Employee{id=1}  ← KEPT
  read() → Employee{id=2, active=false}  → process() → null            ← FILTERED
  read() → Employee{id=3, active=true}   → process() → Employee{id=3}  ← KEPT

  writer.write([Employee{1}, Employee{3}])  ← Only 2 items written!
  
  StepExecution: readCount=3, writeCount=2, filterCount=1
```

**Step 3 — What happens when reader returns null mid-chunk:**

```
Chunk size = 500, but only 350 records left:

  read() → item 1  (read #501)
  read() → item 2  (read #502)
  ...
  read() → item 350 (read #850)
  read() → null     ← End of data!

  writer.write([350 items])  ← Partial chunk, that's okay
  COMMIT
  Step COMPLETED
```

### 🗣️ How to Explain in Interview

> *"Internally, Spring Batch opens a transaction, then calls reader.read() repeatedly up to the chunk size — say 500 times. Each item returned is passed to processor.process(), and if the processor returns a non-null result, it's added to a list. If it returns null, the item is filtered out. Once we have 500 items (or the reader returns null for end-of-data), the entire list is passed to writer.write() in one call. Then the transaction commits, execution counts are updated, and the checkpoint is saved. The last chunk can be a partial chunk if there aren't enough records to fill it — that's normal."*

### ⚡ Key Points to Remember

1. **read()** → one item per call; null = end of data
2. **process()** → null = filter item; not an error
3. **write()** → receives the full chunk at once
4. **Last chunk** can be smaller than chunk size
5. **Checkpoint saved after every commit** (not after every read)

---

<a id="q23"></a>

## Q23. What is the difference between chunk size and commit interval?

### 🔑 Quick Answer

> They are the **same thing**. Chunk size = commit interval. If chunk size is 500, that means 500 items are processed per transaction, and the transaction commits after every 500 items.

### 📖 Step-by-Step Explanation

**Step 1 — They're the same concept:**

```java
.<Input, Output>chunk(500, txManager)
                 ↑
            This 500 means BOTH:
            - Process 500 items per chunk (chunk size)
            - Commit transaction every 500 items (commit interval)
```

**Step 2 — Why people get confused:**

In older Spring Batch versions (before 5.0), there was a `commit-interval` XML attribute. In modern Java config, it's just the chunk constructor parameter. Same concept, different name.

| Term | Meaning | Set Where |
|------|---------|-----------|
| Chunk size | How many items per chunk | `chunk(500, tx)` |
| Commit interval | How often to commit | Same — `chunk(500, tx)` |
| Page size | How many records reader fetches per DB query | Reader config (separate!) |

**Step 3 — chunk size vs page size (this IS different!):**

```
Chunk size = 500 → Process 500 items, then commit
Page size  = 100 → Reader fetches 100 rows per SQL query

For 500 items:
  Reader makes 5 SQL queries (5 × 100 = 500)
  Processor processes 500 items
  Writer writes 500 items
  COMMIT

They can be different! Page size ≤ Chunk size is recommended.
```

### 🗣️ How to Explain in Interview

> *"Chunk size and commit interval are the same thing in Spring Batch. When you set chunk(500), it means two things: process 500 items per chunk AND commit the transaction after those 500 items. What IS different is page size — that's a reader-level setting for how many rows to fetch per database query. So with chunk size 500 and page size 100, the reader makes 5 database calls to fill one chunk, then everything commits. They should not be confused."*

### ⚡ Key Points to Remember

1. **Chunk size = Commit interval** (same thing)
2. **Page size** is separate — it's how many rows the reader fetches per query
3. Recommended: **page size ≤ chunk size**
4. Chunk size affects: **memory, transaction size, commit frequency**

---

<a id="q24"></a>

## Q24. What happens if a failure occurs in the middle of a chunk?

### 🔑 Quick Answer

> The **current chunk rolls back entirely** — nothing from that chunk is written. But all **previously committed chunks are safe**. The job can restart from the failed chunk.

### 📖 Step-by-Step Explanation

**Step 1 — Scenario: processing 2000 records with chunk size 500:**

```
Chunk 1 (records 1-500):
  Read 500 → Process 500 → Write 500 → COMMIT ✅
  Saved to DB permanently. SAFE.

Chunk 2 (records 501-1000):
  Read 500 → Process 500 → Write 500 → COMMIT ✅
  Saved to DB permanently. SAFE.

Chunk 3 (records 1001-1500):
  Read 500 → Process 300 → 💥 EXCEPTION at record 1301!
  → ROLLBACK ❌ (all 500 records in this chunk lost)
  → StepExecution status = FAILED
  → JobExecution status = FAILED

Records 1-1000: ✅ Safely in database
Records 1001-1500: ❌ Not written (rolled back)
Records 1501-2000: ❌ Never processed
```

**Step 2 — On restart:**

```
Spring Batch reads from JobRepository:
  "Step was FAILED, last committed position = 1000"

Restart:
  Chunk 3 (records 1001-1500): Processes again from the beginning of THIS chunk
  Chunk 4 (records 1501-2000): Normal processing
  COMPLETED ✅
```

**Step 3 — What if you want to SKIP the bad record instead of failing?**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()                            // Enable fault tolerance
            .skip(ValidationException.class)            // Skip this type
            .skipLimit(100)                             // Allow up to 100 skips
            .build();
}
```

With skip enabled, the bad record is skipped, and the chunk continues.

### 🗣️ How to Explain in Interview

> *"If a failure occurs mid-chunk, the entire current chunk is rolled back — none of those records are written. But importantly, all previously committed chunks are safe because each chunk is its own transaction. On restart, Spring Batch reads the last committed position from the database and resumes from the start of the failed chunk. If you don't want the job to fail on a bad record, you can configure skip logic — tell Spring Batch to skip up to N bad records of a specific exception type, and continue processing the rest."*

### ⚡ Key Points to Remember

1. **Current chunk** = ROLLBACK (all or nothing)
2. **Previous chunks** = SAFE (already committed)
3. **Restart** = resumes from the failed chunk
4. **Skip logic** = alternative to failing the whole job
5. At most you lose **one chunk's worth** of work on crash

---

<a id="q25"></a>

## Q25. How does Spring Batch manage transactions in chunk processing?

### 🔑 Quick Answer

> Each chunk runs in its **own transaction**. Spring Batch opens a transaction before reading/processing, commits after writing, and rolls back if anything fails. Metadata updates run in a **separate transaction**.

### 📖 Step-by-Step Explanation

**Step 1 — Transaction boundaries per chunk:**

```
┌──── Chunk Transaction ─────────────────────────┐
│                                                 │
│  BEGIN TRANSACTION                              │
│  ├── reader.read() × N                         │
│  ├── processor.process() × N                   │
│  └── writer.write(chunk)                        │
│                                                 │
│  COMMIT or ROLLBACK                             │
│                                                 │
└─────────────────────────────────────────────────┘
          │
          ▼
┌──── Metadata Transaction (separate!) ───────────┐
│  UPDATE StepExecution (readCount, writeCount)    │
│  UPDATE ExecutionContext (checkpoint)             │
│  COMMIT                                          │
└──────────────────────────────────────────────────┘
```

**Step 2 — Why separate transactions for metadata?**

If the chunk transaction rolls back, you still want to record that the failure happened. The metadata transaction runs independently so that:
- Execution counts are always up-to-date
- The failed chunk is tracked for restart

**Step 3 — Transaction uses PlatformTransactionManager:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)  // tx = Transaction Manager
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .build();
}

// The PlatformTransactionManager handles:
// - DataSourceTransactionManager → for JDBC
// - JpaTransactionManager → for JPA/Hibernate
```

### 🗣️ How to Explain in Interview

> *"Each chunk runs in its own transaction. Spring Batch opens a transaction before the chunk starts, reads and processes items within that transaction, writes the chunk, and then commits. If anything fails — whether in reading, processing, or writing — the transaction rolls back and only that chunk is lost. What's interesting is that metadata updates — like updating the read count and saving the checkpoint — happen in a separate transaction. That way, even if the business transaction fails, Spring Batch still records what happened, which is essential for restartability."*

### ⚡ Key Points to Remember

1. **One chunk = One transaction** (begin → read/process/write → commit)
2. **Metadata = separate transaction** (always persisted, even on failure)
3. ROLLBACK affects only **current chunk data**, not metadata
4. Uses **PlatformTransactionManager** (DataSource or JPA)
5. Previous chunks are **permanently committed** — never rolled back

---

<a id="q26"></a>

## Q26. What is the optimal chunk size?

### 🔑 Quick Answer

> There's no universal optimal chunk size — it depends on your data. But as a guideline: **100–500 for typical jobs**, **500–1000 for simple insert-heavy jobs**, **50–100 for jobs with external API calls**. Always benchmark.

### 📖 Step-by-Step Explanation

**Step 1 — Understand the tradeoff:**

```
Small chunk (10):
  ✅ Low memory usage
  ✅ Quick rollback (only 10 records lost)
  ❌ Too many commits → overhead (10M records = 1M commits!)
  ❌ Too many DB transactions → slow

Large chunk (10000):
  ✅ Fewer commits → more efficient
  ✅ Better writer batch performance
  ❌ High memory (10K objects in RAM)
  ❌ Large rollback on failure (10K records to reprocess)
  ❌ Long transactions (risk of lock timeout)

Sweet spot (100-500):
  ✅ Balanced memory usage
  ✅ Reasonable commit frequency
  ✅ Acceptable rollback scope
```

**Step 2 — Guidelines by use case:**

| Scenario | Recommended Chunk Size | Why |
|----------|----------------------|-----|
| Simple CSV → DB insert | 500–1000 | Fast writes, low processing |
| Complex transformations | 100–300 | More memory per item |
| External API calls per item | 50–100 | API latency dominates |
| Large objects (images, PDFs) | 10–50 | High memory per item |
| Heavy DB operations (joins) | 100–200 | DB connection/lock pressure |

**Step 3 — How to find YOUR optimal chunk size:**

```
Benchmark with different sizes and measure:

| Chunk Size | Total Time | Memory | Commits | Records/sec |
|-----------|------------|--------|---------|-------------|
| 50        | 120 min    | 200MB  | 200,000 | 1,389       |
| 100       | 80 min     | 300MB  | 100,000 | 2,083       |
| 500       | 45 min     | 500MB  | 20,000  | 3,703       | ← Sweet spot
| 1000      | 42 min     | 900MB  | 10,000  | 3,968       |
| 5000      | 43 min     | 2.5GB  | 2,000   | 3,876       |

In this example, 500 is the sweet spot — doubling to 1000 barely improves speed
but doubles memory. Going to 5000 actually gets SLOWER (GC pressure).
```

### 🗣️ How to Explain in Interview

> *"There's no one-size-fits-all chunk size. It's a balance between commit overhead and memory/rollback scope. Too small means too many database commits — that's overhead. Too large means high memory and if a chunk fails, you reprocess thousands of records. For typical ETL jobs, I start with 500 and benchmark. For jobs with external API calls, I go smaller like 50-100 because the API latency is the bottleneck, not the commit frequency. In production, I always benchmark at least 4-5 different sizes and compare total time and memory usage."*

### ⚡ Key Points to Remember

1. **Small chunk** = more commits, less memory, less rollback risk
2. **Large chunk** = fewer commits, more memory, bigger rollback
3. **Start with 500**, benchmark and adjust
4. Always **measure** — don't guess
5. **External API calls** → use smaller chunks (50-100)

---

<a id="q27"></a>

## Q27. How do you handle memory issues with large chunks?

### 🔑 Quick Answer

> Reduce chunk size, use **paging readers** instead of cursor readers, clear JPA cache after each chunk, avoid loading all data into memory, and consider partitioning for very large datasets.

### 📖 Step-by-Step Explanation

**Step 1 — Why memory issues happen:**

```
Chunk size = 5000 with complex objects:

  5000 × InputObject   = in memory during read
  5000 × OutputObject  = in memory during process
  + JPA first-level cache (if using JPA)
  + Reader buffer
  = POTENTIAL OutOfMemoryError
```

**Step 2 — Solutions ranked by impact:**

| # | Solution | Impact | How |
|---|----------|--------|-----|
| 1 | **Reduce chunk size** | ⭐⭐⭐⭐⭐ | `chunk(100, tx)` instead of `chunk(5000, tx)` |
| 2 | **Use Paging reader** | ⭐⭐⭐⭐ | JdbcPagingItemReader fetches pages, not whole result set |
| 3 | **Clear JPA cache** | ⭐⭐⭐ | Clear EntityManager after each chunk |
| 4 | **Use projections** | ⭐⭐⭐ | `SELECT id, name` instead of `SELECT *` |
| 5 | **Partitioning** | ⭐⭐⭐⭐⭐ | Split work across threads (each has own memory) |

**Step 3 — Clearing JPA cache (most missed by developers):**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx,
                 EntityManagerFactory emf) {
    return new StepBuilder("step", repo)
            .<Input, Output>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .listener(new ChunkListenerSupport() {
                @Override
                public void afterChunk(ChunkContext context) {
                    // Clear JPA first-level cache after each chunk
                    emf.createEntityManager().clear();
                }
            })
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"First, reduce the chunk size — that's the quickest fix. If you're using a cursor reader, switch to a paging reader which fetches data in small pages instead of holding the whole result set. If you're using JPA, clear the EntityManager cache after each chunk because JPA caches all entities in its first-level cache, which grows throughout the step. Also use SELECT projections to read only the columns you need. For very large datasets, partitioning is the real solution — split the data into ranges and process each range in a separate thread with its own memory space."*

### ⚡ Key Points to Remember

1. **Reduce chunk size** — simplest fix
2. **Paging reader** — don't hold full result set in memory
3. **Clear JPA cache** — most common hidden memory leak
4. **Partitioning** — best for massive datasets
5. Monitor with **-Xmx JVM flags** and heap dump analysis

---

<a id="q28"></a>

## Q28. What happens if ItemProcessor returns null?

### 🔑 Quick Answer

> The item is **filtered out** — it will NOT be passed to the writer. This is **not an error** — Spring Batch tracks it as `filterCount` in StepExecution. It's the standard way to exclude items.

### 📖 Step-by-Step Explanation

**Step 1 — Processor returning null = "skip this item":**

```java
@Bean
public ItemProcessor<Employee, Employee> activeOnlyProcessor() {
    return employee -> {
        if (!employee.isActive()) {
            return null;  // ← This employee will NOT be written
        }
        return employee;  // ← This employee WILL be written
    };
}
```

**Step 2 — What happens in the chunk (chunk size = 5):**

```
Read: Emp1(active) → Process → Emp1    ← Added to chunk
Read: Emp2(inactive) → Process → null  ← FILTERED (not added)
Read: Emp3(active) → Process → Emp3    ← Added to chunk
Read: Emp4(inactive) → Process → null  ← FILTERED (not added)
Read: Emp5(active) → Process → Emp5    ← Added to chunk

Writer receives: [Emp1, Emp3, Emp5]    ← Only 3 items!

StepExecution counts:
  readCount: 5        (5 items were read)
  writeCount: 3       (3 items were written)
  filterCount: 2      (2 items were filtered by processor returning null)
```

**Step 3 — Filter vs Skip — they are DIFFERENT:**

| Concept | What triggers it | Counts as |
|---------|-----------------|-----------|
| **Filter** | Processor returns `null` | filterCount (normal business logic) |
| **Skip** | Exception + skip configured | skipCount (error scenario) |

Filter = intentional business decision ("don't include inactive employees")  
Skip = error handling ("this record has bad data, skip it")

### 🗣️ How to Explain in Interview

> *"When the processor returns null, the item is filtered out and won't be sent to the writer. This is not an error — it's the standard way to exclude items based on business logic. Spring Batch tracks filtered items separately in the filterCount field. This is different from skip logic — a filter is an intentional business decision, like excluding inactive employees, while a skip is error handling for bad data. So if I read 10,000 records, the processor filters 500, and skip handles 50 errors, my counts would be: readCount=10,000, filterCount=500, writeCount=9,450, skipCount=50."*

### ⚡ Key Points to Remember

1. **Return null** from processor = item filtered
2. **Not an error** — tracked as `filterCount` in StepExecution
3. Writer receives **fewer items** than reader read
4. **Filter ≠ Skip**: filter is business logic, skip is error handling
5. `readCount - filterCount - skipCount = writeCount`

---

<a id="q29"></a>

## Q29. What happens if ItemWriter fails?

### 🔑 Quick Answer

> The **entire chunk transaction rolls back** — none of the items in that chunk are written. If skip is configured, Spring Batch enters **scan mode** — it re-processes items one-by-one to identify the bad record.

### 📖 Step-by-Step Explanation

**Step 1 — Without skip (default behavior):**

```
Chunk (500 items):
  Read 500 ✅
  Process 500 ✅
  Write 500 → 💥 DataIntegrityViolationException (1 bad record)
  
  ROLLBACK entire chunk (all 500 items lost)
  StepExecution: FAILED
  JobExecution: FAILED
```

**Step 2 — With skip configured — SCAN MODE (important!):**

```java
.faultTolerant()
.skip(DataIntegrityViolationException.class)
.skipLimit(10)
```

```
Chunk (500 items):
  Read 500 ✅
  Process 500 ✅
  Write 500 → 💥 DataIntegrityViolationException
  ROLLBACK

  → Enter SCAN MODE:
    Write [item1] alone → ✅ COMMIT
    Write [item2] alone → ✅ COMMIT
    Write [item3] alone → ✅ COMMIT
    ...
    Write [item247] alone → 💥 FAIL → SKIP this item
    ...
    Write [item500] alone → ✅ COMMIT

  Result: 499 items written, 1 skipped
  Continue to next chunk!
```

**Step 3 — Why scan mode is slow but necessary:**

The writer receives all 500 items at once. When it fails, Spring Batch doesn't know WHICH item caused the failure. So it has to re-try them one-by-one (scan mode) to identify the bad record. This is slower, but it's only triggered when a failure happens.

### 🗣️ How to Explain in Interview

> *"If the writer fails, the entire chunk rolls back. Without skip configuration, the job fails. But if you've configured skip for that exception type, Spring Batch does something clever — it enters scan mode. Since the writer received 500 items and doesn't know which one failed, it rollbacks the chunk and then re-processes each item individually — writes one item, commits, writes the next, commits. When it finds the problematic item, it skips it. The remaining 499 items still get written successfully. It's slower for that one chunk, but it saves the whole job from failing."*

### ⚡ Key Points to Remember

1. Writer failure → **whole chunk rolls back**
2. Without skip → **job fails**
3. With skip → **scan mode** (one-by-one) to find the bad item
4. Scan mode is **slow but only happens when failure occurs**
5. Previously committed chunks are **always safe**

---

<a id="q30"></a>

## Q30. Can we skip records in chunk processing?

### 🔑 Quick Answer

> Yes! Configure **fault tolerance** with `faultTolerant()`, specify which exceptions to `skip()`, and set a `skipLimit()`. You can skip during reading, processing, or writing.

### 📖 Step-by-Step Explanation

**Step 1 — Basic skip configuration:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)
            .reader(orderReader())
            .processor(orderProcessor())
            .writer(orderWriter())
            .faultTolerant()                                    // Enable fault tolerance
            .skip(ValidationException.class)                    // Skip these errors
            .skip(DataIntegrityViolationException.class)        // And these
            .noSkip(FileNotFoundException.class)                // NEVER skip this
            .skipLimit(100)                                     // Max 100 skips total
            .build();
}
```

**Step 2 — Skip can happen at each phase:**

| Phase | What Happens on Skip |
|-------|---------------------|
| **Reader** | Bad line in CSV → skip line, move to next |
| **Processor** | Validation fails → item excluded, move to next |
| **Writer** | Constraint violation → scan mode, skip bad item |

```
StepExecution counts after completion:
  readCount: 10,000
  readSkipCount: 5          ← 5 bad CSV lines
  processSkipCount: 20      ← 20 failed validation
  writeSkipCount: 3         ← 3 constraint violations
  writeCount: 9,972         ← 10,000 - 5 - 20 - 3
  skipCount: 28             ← Total: 5 + 20 + 3
```

**Step 3 — Track skipped records with SkipListener:**

```java
public class LoggingSkipListener implements SkipListener<Order, Order> {

    @Override
    public void onSkipInRead(Throwable t) {
        log.error("Skipped during READ: {}", t.getMessage());
    }

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        log.error("Skipped during PROCESS: Order={}, Error={}", 
                  item.getId(), t.getMessage());
    }

    @Override
    public void onSkipInWrite(Order item, Throwable t) {
        log.error("Skipped during WRITE: Order={}, Error={}", 
                  item.getId(), t.getMessage());
    }
}

// Register it:
.listener(new LoggingSkipListener())
```

### 🗣️ How to Explain in Interview

> *"Yes, Spring Batch supports skipping records. You enable it with faultTolerant(), specify which exception classes to skip and a maximum skip limit. Skipping works at all three phases — read, process, and write. During read, if a CSV line is malformed, it skips to the next line. During processing, if validation fails, the item is excluded. During writing, if there's a constraint violation, it enters scan mode to identify and skip the bad item. I always use a SkipListener to log skipped records so we can investigate later. And I set a reasonable skip limit — if more than 100 records fail, something is systemically wrong and the job should fail."*

### ⚡ Key Points to Remember

1. **`faultTolerant()` → `skip()` → `skipLimit()`** — three method chain
2. Skip works at **all three phases** (read, process, write)
3. Write skip triggers **scan mode** (one-by-one re-processing)
4. Always use **SkipListener** to log skipped records
5. **skipLimit** = safety net — too many bad records means systemic problem

### 🔗 Follow-up Questions
- *"What is retry logic?"* → See Q71
- *"How to handle bad records in production?"* → See Q76
- *"What is SkipPolicy?"* → See Q74

---

> **🎯 Navigation:** [← Basics (Q1-20)](01-basics.md) | [Next → Readers (Q31-40)](03-readers.md) | [📋 All Sections](README.md)
