# 🔴 Spring Batch — Transactions & Restartability (Q63–Q69)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q63"></a>

## Q63. How does Spring Batch handle transactions?

### 🔑 Quick Answer

> Each **chunk** runs in its own transaction. Spring Batch begins a transaction before reading, commits after writing, and rolls back the entire chunk if anything fails. **Metadata updates happen in a separate transaction** so they persist even on failure.

### 📖 Step-by-Step Explanation

**Step 1 — One chunk = One transaction:**

```
Chunk 1 (records 1-500):
  BEGIN TRANSACTION
    reader.read() × 500
    processor.process() × 500
    writer.write(500 items)
  COMMIT ✅

Chunk 2 (records 501-1000):
  BEGIN TRANSACTION
    reader.read() × 500
    processor.process() × 500
    writer.write(500 items)
  COMMIT ✅

Chunk 3 (records 1001-1500):
  BEGIN TRANSACTION
    reader.read() × 500
    processor.process() × 300
    💥 Exception!
  ROLLBACK ❌  ← Only this chunk's data is lost

Records 1-1000: ✅ Permanently in database (already committed)
Records 1001-1500: ❌ Rolled back (none written)
```

**Step 2 — Metadata is in a SEPARATE transaction:**

```
BUSINESS TRANSACTION (chunk data):
  BEGIN → Read/Process/Write → COMMIT or ROLLBACK

METADATA TRANSACTION (independent):
  BEGIN → Update StepExecution counts → Save ExecutionContext → COMMIT

Why separate?
  Even if business transaction rolls back,
  the metadata still records that the failure happened.
  This is ESSENTIAL for restart — without it, Spring Batch
  wouldn't know where the job failed.
```

**Step 3 — Transaction manager:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)  // tx manages the chunk transaction
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .build();
}

// PlatformTransactionManager can be:
// - DataSourceTransactionManager (JDBC)
// - JpaTransactionManager (JPA/Hibernate)
// - JtaTransactionManager (distributed transactions)
```

### 🗣️ How to Explain in Interview

> *"Spring Batch manages transactions at the chunk level. Each chunk — say 500 records — runs in its own transaction. If the chunk succeeds, it commits and those records are permanent. If it fails, only that chunk rolls back — previous chunks are safe. The clever part is that metadata updates happen in a separate transaction. So even when a chunk fails and rolls back, the metadata — read counts, the last successful position — is still saved. This is what makes restart possible: the metadata knows exactly where the job left off."*

### ⚡ Key Points to Remember

1. **One chunk = One transaction**
2. Failure → **only current chunk** rolls back
3. Previous chunks are **permanently committed**
4. **Metadata** in separate transaction (persists even on failure)
5. Transaction manager is injected in Step configuration

---

<a id="q64"></a>

## Q64. What is rollback in Spring Batch?

### 🔑 Quick Answer

> Rollback means the **current chunk's data changes are undone** — nothing from that chunk is written to the database. It happens when an exception occurs during read, process, or write. Previous chunks remain committed.

### 📖 Step-by-Step Explanation

**Step 1 — What gets rolled back and what doesn't:**

```
ROLLED BACK (undone):
  ✗ Database INSERTs/UPDATEs from writer
  ✗ JPA entity changes
  ✗ JMS messages (if transactional)

NOT ROLLED BACK (still happens):
  ✓ File writes (files don't support transactions!)
  ✓ External API calls already made
  ✓ Emails/SMS already sent
  ✓ Metadata updates (separate transaction)
  ✓ Previously committed chunks
```

**Step 2 — Scenario: failure at different phases:**

```
Failure during READ:
  No data was processed or written → rollback is trivial
  Reader may retry or skip (if configured)

Failure during PROCESS:
  Some items read but nothing written → rollback discards processed items
  With skip: bad item excluded, chunk retried

Failure during WRITE:
  ALL items in chunk rolled back (even good ones)
  With skip: scan mode (one-by-one) to find bad item
```

**Step 3 — The rollbackCount metric:**

```
StepExecution after completion:
  commitCount: 97         ← 97 chunks committed successfully
  rollbackCount: 3        ← 3 chunks had to be rolled back

rollbackCount > 0 means something went wrong
Check logs and SkipListener to find what failed
```

### 🗣️ How to Explain in Interview

> *"Rollback in Spring Batch means the current chunk's database changes are completely undone. If the writer was inserting 500 records and it fails at record 300, all 500 are rolled back — none are written. This is standard database transaction behavior. What's important to know is that non-transactional operations like file writes or API calls can't be rolled back. If I wrote to a file AND a database in the same step and the database write fails, the file already has the data — that's a potential inconsistency. In such cases, I use separate steps or idempotent writes."*

### ⚡ Key Points to Remember

1. Rollback = **current chunk's DB changes undone**
2. **Previous chunks** stay committed
3. **File operations** can't be rolled back (not transactional)
4. Check **rollbackCount** in StepExecution for monitoring
5. Use **idempotent** operations for non-transactional resources

---

<a id="q65"></a>

## Q65. What happens when a chunk fails?

### 🔑 Quick Answer

> Transaction rolls back → if **no fault tolerance**: step FAILS, job FAILS. If **skip configured**: check exception type, skip bad item or fail. If **retry configured**: retry the operation before failing.

### 📖 Step-by-Step Explanation

**Step 1 — Decision tree after chunk failure:**

```
Chunk fails with Exception
│
├── Is faultTolerant() configured?
│   │
│   ├── NO → ROLLBACK → Step FAILED → Job FAILED
│   │
│   └── YES →
│       │
│       ├── Is this exception retryable?
│       │   ├── YES → Retry (up to retryLimit)
│       │   │         If still fails after retries → check skip
│       │   └── NO → check skip
│       │
│       └── Is this exception skippable?
│           ├── YES → Is skipLimit reached?
│           │   ├── NO → Skip item, continue processing
│           │   └── YES → Step FAILED (too many skips)
│           │
│           └── NO → Step FAILED → Job FAILED
```

**Step 2 — Complete example:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processOrders", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .retry(DeadlockLoserDataAccessException.class)     // Retry deadlocks
            .retryLimit(3)                                      // Up to 3 times
            .skip(DataIntegrityViolationException.class)        // Skip bad data
            .skip(ValidationException.class)                    // Skip invalid items
            .noSkip(FileNotFoundException.class)                // Never skip this!
            .skipLimit(100)                                     // Max 100 skips total
            .listener(new LoggingSkipListener())                // Log what was skipped
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"When a chunk fails, it depends on whether fault tolerance is configured. Without it, the step and job fail immediately. With fault tolerance, Spring Batch checks: is this a retryable exception like a database deadlock? If so, retry up to N times. If retries fail or the exception isn't retryable, check if it's skippable — like a data validation error. If yes and skip limit isn't reached, skip the bad record and continue. If the skip limit is reached, the job fails because too many errors indicate a systemic problem. I always configure both retry and skip in production with a SkipListener to log every skipped record."*

### ⚡ Key Points to Remember

1. No fault tolerance → **immediate failure**
2. Retry first → for **transient errors** (deadlocks, timeouts)
3. Skip next → for **data errors** (bad records)
4. **skipLimit** = safety net (too many errors = systemic problem)
5. Always **log skipped records** with SkipListener

---

<a id="q66"></a>

## Q66. How does Spring Batch support restartability?

### 🔑 Quick Answer

> Through three mechanisms: (1) **Step-level status tracking** — skip completed steps on restart. (2) **ExecutionContext checkpointing** — save reader position after each chunk commit. (3) **Database persistence** — all state is in BATCH_* tables, survives JVM crashes.

### 📖 Step-by-Step Explanation

**Step 1 — Mechanism 1: Step status tracking:**

```
First run:
  Step 1: COMPLETED ✅
  Step 2: FAILED ❌
  Step 3: Not started

Restart:
  Step 1: Status=COMPLETED → SKIP ✅ (don't re-run)
  Step 2: Status=FAILED → RESUME from checkpoint
  Step 3: Status=null → Execute from beginning
```

**Step 2 — Mechanism 2: Chunk-level checkpointing:**

```
Step 2 processes 10,000 records (chunk size 500):
  Chunk 1 (1-500):    COMMIT → Save: ExecutionContext{"read.count": 500}
  Chunk 2 (501-1000): COMMIT → Save: ExecutionContext{"read.count": 1000}
  ...
  Chunk 10 (4501-5000): COMMIT → Save: ExecutionContext{"read.count": 5000}
  Chunk 11 (5001-5500): 💥 CRASH

On restart:
  Load ExecutionContext from DB: {"read.count": 5000}
  Reader starts from record 5001
  Records 1-5000 are NOT re-processed ✅
```

**Step 3 — Mechanism 3: Database persistence:**

```
All of this works because:
  - BATCH_JOB_EXECUTION stores: job status
  - BATCH_STEP_EXECUTION stores: step status + counts
  - BATCH_STEP_EXECUTION_CONTEXT stores: reader position (JSON)

Even if the JVM crashes completely, the state is in the DATABASE.
A new JVM can read it and resume.
```

### 🗣️ How to Explain in Interview

> *"Spring Batch supports restartability through three mechanisms. First, step-level status tracking — completed steps are skipped on restart. Second, chunk-level checkpointing — after each chunk commits, the reader's current position is saved in the ExecutionContext, which is persisted to the database. So if the job crashes after processing 5000 records, on restart it reads the saved position and starts from record 5001. Third, all of this survives JVM crashes because it's stored in the database, not in memory. A completely new JVM instance can pick up exactly where the previous one left off."*

### ⚡ Key Points to Remember

1. **Step tracking**: completed steps skipped on restart
2. **Chunk checkpoint**: reader position saved after each commit
3. **Database**: all state persists across JVM crashes
4. **At most one chunk** of work is lost on crash
5. Built into framework — **no custom restart code needed**

---

<a id="q67"></a>

## Q67. How does ExecutionContext work?

### 🔑 Quick Answer

> ExecutionContext is a **serialized Map** (key-value pairs) that gets saved to the database after every chunk commit. Two scopes: **Job-level** (shared across steps) and **Step-level** (private to one step, used for checkpointing).

### 📖 Step-by-Step Explanation

**Step 1 — Two scopes, two tables:**

```
Job ExecutionContext:
  - Stored in: BATCH_JOB_EXECUTION_CONTEXT
  - Scope: Shared across ALL steps
  - Use: Inter-step communication
  - Example: {"totalRecords": 50000, "outputFile": "report.csv"}

Step ExecutionContext:
  - Stored in: BATCH_STEP_EXECUTION_CONTEXT
  - Scope: Private to ONE step
  - Use: Reader position (checkpointing)
  - Example: {"FlatFileItemReader.read.count": 25000}
```

**Step 2 — When is it saved?**

```
Every chunk commit:
  1. Read N items
  2. Process N items
  3. Write N items
  4. COMMIT business transaction
  5. Save ExecutionContext to DB ← happens here!
  6. Update StepExecution counts

It's saved AFTER each chunk, not after each record.
So if you crash between two chunk commits, the LAST saved state is used.
```

**Step 3 — How to use it in code:**

```java
// WRITE to Job ExecutionContext (Step 1)
@Bean
public Tasklet step1Tasklet() {
    return (contribution, chunkContext) -> {
        ExecutionContext jobContext = chunkContext.getStepContext()
                .getStepExecution()
                .getJobExecution()
                .getExecutionContext();
        
        jobContext.putInt("totalRecords", 50000);
        jobContext.putString("reportDate", "2024-01-15");
        
        return RepeatStatus.FINISHED;
    };
}

// READ from Job ExecutionContext (Step 2)
@Bean
@StepScope
public ItemProcessor<Record, Record> step2Processor(
        @Value("#{jobExecutionContext['totalRecords']}") int total) {
    
    return record -> {
        // Use the value from Step 1
        log.info("Processing record against total: {}", total);
        return record;
    };
}
```

### 🗣️ How to Explain in Interview

> *"ExecutionContext is a Map that Spring Batch serializes to the database. There are two scopes. Job ExecutionContext is shared across all steps — useful for passing data between steps, like a total count or output file name. Step ExecutionContext is private to one step — used mainly by readers to save their position for restart. The built-in FlatFileItemReader automatically saves 'read.count' in the Step ExecutionContext. The important thing is that it's saved after every chunk commit, so on restart, the last saved state is loaded and the reader picks up from there."*

### ⚡ Key Points to Remember

1. **Job context** = shared between steps
2. **Step context** = private, used for checkpointing
3. Saved **after every chunk commit**
4. Serialized as **JSON** in BATCH_*_EXECUTION_CONTEXT tables
5. Access: `#{jobExecutionContext['key']}` or `#{stepExecutionContext['key']}`

---

<a id="q68"></a>

## Q68. Where is ExecutionContext stored?

### 🔑 Quick Answer

> **Job ExecutionContext** → `BATCH_JOB_EXECUTION_CONTEXT` table. **Step ExecutionContext** → `BATCH_STEP_EXECUTION_CONTEXT` table. Both are serialized as **JSON** (Spring Batch 5+) and stored in the `SHORT_CONTEXT` column.

### 📖 Step-by-Step Explanation

**Step 1 — The tables:**

```sql
-- Job ExecutionContext
SELECT * FROM BATCH_JOB_EXECUTION_CONTEXT WHERE JOB_EXECUTION_ID = 12;

JOB_EXECUTION_ID | SHORT_CONTEXT
12               | {"totalRecords":50000,"reportDate":"2024-01-15"}

-- Step ExecutionContext
SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT WHERE STEP_EXECUTION_ID = 45;

STEP_EXECUTION_ID | SHORT_CONTEXT
45                | {"FlatFileItemReader.read.count":25000}
```

**Step 2 — Serialization format:**

| Spring Batch Version | Format |
|---------------------|--------|
| 4.x and earlier | Java serialization (binary) |
| 5.x and later | **JSON** (human-readable) |

**Step 3 — Size considerations:**

```
SHORT_CONTEXT column: VARCHAR(2500) by default
SERIALIZED_CONTEXT column: TEXT/CLOB (for larger contexts)

If your context fits in 2500 chars → goes to SHORT_CONTEXT
If larger → goes to SERIALIZED_CONTEXT

⚠️ Don't store large objects in ExecutionContext!
Store only: counts, positions, file paths, small metadata
```

### 🗣️ How to Explain in Interview

> *"ExecutionContext is stored in two tables: BATCH_JOB_EXECUTION_CONTEXT for job-level context and BATCH_STEP_EXECUTION_CONTEXT for step-level context. In Spring Batch 5, it's serialized as JSON in the SHORT_CONTEXT column — which has a 2500 character limit. For larger data, it overflows to the SERIALIZED_CONTEXT column which is a CLOB. The key thing is to keep the context small — just positions, counts, and paths. Don't store actual data records in it."*

### ⚡ Key Points to Remember

1. **Job context** → `BATCH_JOB_EXECUTION_CONTEXT` table
2. **Step context** → `BATCH_STEP_EXECUTION_CONTEXT` table
3. Format: **JSON** (Spring Batch 5+)
4. Keep context **small** — counts, positions, paths only
5. `SHORT_CONTEXT` (2500 chars) → `SERIALIZED_CONTEXT` (CLOB) for overflow

---

<a id="q69"></a>

## Q69. How does Spring Batch resume from the failure point?

### 🔑 Quick Answer

> It reads the **ExecutionContext from the database**, extracts the reader's last saved position, initializes the reader at that position, and continues reading from the next record. Only the last uncommitted chunk is lost.

### 📖 Step-by-Step Explanation

**Step 1 — Complete resume flow:**

```
ORIGINAL RUN:
  Chunk 1 → 500 records → COMMIT → Save EC: {"read.count": 500}
  Chunk 2 → 500 records → COMMIT → Save EC: {"read.count": 1000}
  Chunk 3 → 500 records → COMMIT → Save EC: {"read.count": 1500}
  Chunk 4 → reading record 1823 → 💥 JVM CRASH!
  
  Last saved EC in DB: {"read.count": 1500}
  Records 1-1500: ✅ In database (committed)
  Records 1501-1823: ❌ Lost (uncommitted chunk)

RESTART:
  1. Load StepExecution from DB → status=FAILED
  2. Load ExecutionContext from DB → {"read.count": 1500}
  3. Initialize reader with read.count=1500
  4. Reader: open file, skip to line 1501
  5. Process from record 1501 (chunk 4 restarts from beginning)
  
  Records 1501-1823 are re-processed (they were never committed)
  This is AT MOST one chunk's worth of duplicate processing.
```

**Step 2 — How the reader uses ExecutionContext:**

```java
// FlatFileItemReader internal implementation (simplified):
public void open(ExecutionContext executionContext) {
    this.resource = resource; // Open file
    
    // Check if this is a restart
    if (executionContext.containsKey("read.count")) {
        int savedPosition = executionContext.getInt("read.count");
        
        // Skip to saved position
        for (int i = 0; i < savedPosition; i++) {
            readLine(); // Skip already-processed lines
        }
    }
}

public void update(ExecutionContext executionContext) {
    // Called after each chunk commit
    executionContext.putInt("read.count", currentLineNumber);
}
```

**Step 3 — What about database readers?**

```
JdbcPagingItemReader resume:
  Saved EC: {"start.after": {"id": 1500}}
  
  On restart:
  Normal query: SELECT * FROM emp WHERE id > 0 ORDER BY id LIMIT 500
  Restart query: SELECT * FROM emp WHERE id > 1500 ORDER BY id LIMIT 500
  
  The reader automatically adjusts the WHERE clause!
```

### 🗣️ How to Explain in Interview

> *"The resume mechanism is elegant. After each chunk commits, the reader saves its current position in the ExecutionContext — for FlatFileItemReader it saves the line count, for JdbcPagingItemReader it saves the last processed ID. This is persisted to the database. On restart, the reader's open() method checks if the ExecutionContext has a saved position. If yes, it initializes to that position: the file reader skips ahead that many lines, the paging reader adjusts its WHERE clause. The key insight is that at most one chunk's worth of data might be re-processed — the uncommitted chunk. Everything before that was already committed."*

### ⚡ Key Points to Remember

1. **ExecutionContext** loaded from DB on restart
2. Reader **skips to saved position** (line count or ID)
3. **At most one chunk** of data re-processed
4. File readers skip lines, DB readers adjust WHERE clause
5. Built into framework — **readers handle this automatically**

---

> **🎯 Navigation:** [← Job Execution (Q55-62)](06-job-execution.md) | [Next → Error Handling (Q70-78)](08-error-handling.md) | [📋 All Sections](README.md)
