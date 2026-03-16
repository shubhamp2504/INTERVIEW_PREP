# 🔄 Transactions & Restart — Q63 to Q69

---

## Q63. How does Spring Batch handle transactions?

### 📝 One-Liner
Each chunk runs in its own transaction — commit on success, rollback on failure — with metadata updates always persisted in a separate transaction.

### 🔑 Quick Answer
Spring Batch wraps each chunk in a transaction using `PlatformTransactionManager`. Read → Process → Write all happen in ONE transaction per chunk. On success → commit. On failure → rollback (only current chunk). Critical design: metadata updates (StepExecution, ExecutionContext) run in a SEPARATE transaction that always commits — even if the chunk fails. This is how Spring Batch knows where to restart. *(Har chunk ka apna transaction — fail hua toh sirf woh rollback, metadata hamesha save)*

### 📖 How It Works
```
Per-Chunk Transaction:

┌─ CHUNK TRANSACTION (data) ──────────────────┐
│  read 500 items → process 500 → write 500   │
│  SUCCESS → COMMIT ✅                         │
│  FAILURE → ROLLBACK ❌                       │
└─────────────────────────────────────────────┘
         ↓
┌─ METADATA TRANSACTION (always commits) ─────┐
│  UPDATE BATCH_STEP_EXECUTION                 │
│  UPDATE BATCH_STEP_EXECUTION_CONTEXT         │
│  COMMIT ✅ (even if chunk failed!)           │
└─────────────────────────────────────────────┘

Why separate transactions?
→ If chunk fails, metadata still records WHERE it failed
→ On restart, Spring Batch reads metadata → knows where to resume
```

### 🗣️ How to Say in Interview
"Spring Batch handles transactions at the chunk level. Each chunk runs within its own transaction — successful chunks commit, failed chunks roll back. The critical design insight is that metadata updates happen in a separate transaction that always commits, even when the data chunk fails. This separation is what makes restartability possible — the metadata preserves the exact failure point. In my project, we verified this by looking at BATCH_STEP_EXECUTION_CONTEXT after a failure — the read count and commit count were preserved, allowing the restart to resume from the exact chunk that failed."

### 💻 Code
```java
@Bean
public Step transactionalStep(JobRepository jobRepository,
                               PlatformTransactionManager tx) {
    return new StepBuilder("transactionalStep", jobRepository)
            .<Order, Order>chunk(500, tx)  // each chunk = one tx
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .build();
}

// Custom transaction attributes
@Bean
public Step customTxStep(JobRepository jobRepository,
                         PlatformTransactionManager tx) {
    DefaultTransactionAttribute txAttr = new DefaultTransactionAttribute();
    txAttr.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    txAttr.setTimeout(300);  // 5 min per chunk transaction
    txAttr.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);

    return new StepBuilder("customTxStep", jobRepository)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .transactionAttribute(txAttr)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Transaction timeout is per CHUNK, not per step or job *(timeout har chunk ke liye hai, poore job ke liye nahi)*
- Don't use `@Transactional` on batch components — Spring Batch manages transactions itself
- External API calls in writer are NOT transactional — rollback won't undo them
- File writes are NOT transactional — written data stays even after rollback

### 🎯 Tricky Interview Qs

**Q: Why doesn't Spring Batch use one big transaction for the entire step?**
Because a single transaction for millions of records would: (1) hold DB locks too long, (2) consume massive memory for undo logs, (3) risk timeout, (4) lose ALL work on single failure. Per-chunk transactions limit blast radius.

### ⚡ Remember
- One chunk = one transaction (commit or rollback)
- Metadata = SEPARATE transaction (always commits) *(metadata hamesha save — restart ke liye zaruri)*
- DataSourceTransactionManager for JDBC, JpaTransactionManager for JPA
- Don't add @Transactional on batch beans
- Non-transactional resources (files, APIs) won't rollback

### 🔗 Follow-ups
- [Q25 → Transaction details in chunk processing](#q25)
- [Q64 → What rollback means in batch](#q64)
- [Q66 → Restartability mechanism](#q66)

---

## Q64. What is rollback in Spring Batch?

### 📝 One-Liner
Rollback undoes all database changes from the current failed chunk — but file writes and external API calls are NOT rolled back.

### 🔑 Quick Answer
When a chunk fails, the transaction manager rolls back all database changes made by that chunk's write operation. All INSERTs, UPDATEs, DELETEs from that chunk are undone. But non-transactional operations are NOT rolled back: file writes stay on disk, API calls already sent can't be unsent, messages already published to Kafka remain. The `rollbackCount` metric in StepExecution tracks how many chunk rollbacks occurred. *(Database changes wapas ho jaate hain, lekin file write aur API calls wapas nahi hote)*

### 📖 How It Works
```
Rollback Scope:

ROLLED BACK (transactional):
├── Database INSERTs from this chunk → undone
├── Database UPDATEs from this chunk → reverted
└── Database DELETEs from this chunk → restored

NOT ROLLED BACK (non-transactional):
├── File already written → stays on disk ❌
├── API calls already made → can't unsend ❌
├── Kafka messages published → stay in topic ❌
├── Emails sent → can't unsend ❌
└── Cache updates → still in cache ❌

StepExecution tracks:
  rollbackCount: 3  → 3 chunks were rolled back during this step
```

### 🗣️ How to Say in Interview
"Rollback in Spring Batch undoes all database changes from the failed chunk — all INSERTs, UPDATEs, and DELETEs for that chunk revert. However, non-transactional operations like file writes and external API calls cannot be rolled back. In my project, we had a step that wrote to both a database and a file. When a chunk failed, the database data rolled back but the file retained the partially written data. We solved this by writing to a temporary file first and renaming it only after the step completed successfully — a common pattern for transactional-like file handling."

### 💻 Code
```java
// Monitoring rollbacks
@Bean
public StepExecutionListener rollbackMonitor() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution se) {
            if (se.getRollbackCount() > 0) {
                log.warn("Step '{}' had {} rollbacks out of {} chunks",
                    se.getStepName(), se.getRollbackCount(), se.getCommitCount());
            }
            return se.getExitStatus();
        }
    };
}

// Safe file writing pattern (temp file + rename)
@Bean
public Step safeFileStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("safeFileStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(tempFileWriter())          // write to temp file
            .listener(new StepExecutionListener() {
                @Override
                public ExitStatus afterStep(StepExecution se) {
                    if (se.getStatus() == BatchStatus.COMPLETED) {
                        Files.move(tempFile, finalFile);  // rename on success
                    } else {
                        Files.delete(tempFile);           // delete on failure
                    }
                    return se.getExitStatus();
                }
            })
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- File operations = NOT transactional → must handle cleanup manually *(file ka rollback manual karna padta hai)*
- `rollbackCount` includes retry rollbacks (if retry is configured)
- JPA entities may still be in first-level cache after rollback → clear EntityManager
- High rollbackCount indicates data quality or configuration issues

### ⚡ Remember
- Rollback = undo DB changes for current chunk only
- Files, APIs, messages = NOT rolled back *(non-transactional cheezein rollback nahi hoti)*
- `rollbackCount` tracks number of rolled-back chunks
- Use temp file + rename for safe file writing
- Previous chunks (already committed) are safe

### 🔗 Follow-ups
- [Q63 → Transaction management](#q63)
- [Q65 → What happens when chunk fails](#q65)
- [Q24 → Chunk failure scenarios](#q24)

---

## Q65. What happens when a chunk fails?

### 📝 One-Liner
Current chunk rolls back; without fault tolerance the job fails; with skip/retry, bad items are handled and processing continues.

### 🔑 Quick Answer
When a chunk fails: **(1) No fault tolerance** → chunk rolls back → step FAILED → job FAILED (immediate). **(2) With retry** → retry the operation up to N times (for transient errors like deadlocks). **(3) With skip** → skip the bad item, continue with rest (for data errors). **(4) With both** → retry first, then skip if retries exhausted. Previous committed chunks are ALWAYS safe. The `skipLimit` acts as a safety net — too many skips means systemic problem, fails the step. *(Pehle retry karo, phir skip karo, zyada ho gaye toh poora band)*

### 📖 How It Works
```
Chunk Failure Decision Tree:

Chunk fails with exception
  ↓
faultTolerant() configured?
├── NO → ROLLBACK → STEP FAILED → JOB FAILED
└── YES
     ├── retry() configured for this exception?
     │   ├── YES → retry up to retryLimit
     │   │   ├── retry succeeds → continue ✅
     │   │   └── all retries fail → check skip
     │   └── NO → check skip
     └── skip() configured for this exception?
         ├── YES → skipCount < skipLimit?
         │   ├── YES → skip item → continue ✅
         │   └── NO → too many skips → STEP FAILED
         └── NO → ROLLBACK → STEP FAILED

Best Practice Config:
  .faultTolerant()
  .retry(DeadlockException.class).retryLimit(3)     // transient
  .skip(ValidationException.class).skipLimit(100)     // data errors
  .noSkip(FatalException.class)                       // must fail
  .listener(skipListener)                             // audit trail
```

### 🗣️ How to Say in Interview
"When a chunk fails, Spring Batch first checks the fault tolerance configuration. Without it, the step fails immediately. With retry configured, it retries the operation — useful for transient errors like deadlocks or timeouts. With skip, it skips the bad item and continues — useful for data errors like validation failures. I combine both: retry 3 times for transient errors, then skip if retries are exhausted. The skipLimit acts as a safety net — if we're skipping too many records, it's likely a systemic issue and the job should fail. In my project, we always logged skipped items through SkipListener for the operations team to review."

### 💻 Code
```java
@Bean
public Step resilientStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("resilientStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            // Retry transient errors
            .retry(DeadlockLoserDataAccessException.class)
            .retry(TransientDataAccessException.class)
            .retryLimit(3)
            // Skip data errors
            .skip(ValidationException.class)
            .skip(FlatFileParseException.class)
            .skipLimit(100)
            // These must ALWAYS fail (never skip)
            .noSkip(DatabaseConnectionException.class)
            .noSkip(OutOfMemoryError.class)
            // Audit trail
            .listener(skipListener())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Without `.faultTolerant()`, even one bad record kills the job *(bina fault tolerance ke ek error = job khatam)*
- skipLimit is TOTAL (read + process + write combined)
- Write failures trigger scan mode (slow — re-writes one-by-one)
- `noSkip()` overrides `skip()` — use for fatal errors that must stop the job
- Always add SkipListener — otherwise skipped items are invisible

### ⚡ Remember
- No config → one error kills job
- Retry → for transient errors (deadlock, timeout)
- Skip → for data errors (validation, parse) *(retry transient ke liye, skip data error ke liye)*
- skipLimit = safety net (too many = systemic problem)
- Always: retry + skip + noSkip + SkipListener

### 🔗 Follow-ups
- [Q70 → Skip logic details](#q70)
- [Q71 → Retry logic details](#q71)
- [Q24 → Chunk failure in chunk processing](#q24)

---

## Q66. How does Spring Batch support restartability?

### 📝 One-Liner
Three mechanisms: step-level status tracking (skip completed steps), chunk-level checkpointing (save position after each commit), and database persistence (survives JVM crashes).

### 🔑 Quick Answer
Restartability is built on three pillars: **(1) Step-level tracking** — completed steps are skipped on restart. **(2) Chunk-level checkpointing** — after each chunk commits, the reader's position is saved in `ExecutionContext`. On restart, the reader initializes at the last saved position. **(3) Database persistence** — all metadata (execution status, context) is stored in Spring Batch tables, surviving JVM crashes. At most ONE chunk of work is lost on failure. *(Teen cheezein: step status, chunk checkpoint, aur database mein save — zyada se zyada ek chunk ka kaam jaata hai)*

### 📖 How It Works
```
Restartability Architecture:

Step Level:
  Step 1 (COMPLETED) → SKIP on restart
  Step 2 (FAILED)    → RESUME on restart
  Step 3 (NOT RUN)   → EXECUTE on restart

Chunk Level (within failed step):
  Chunk 1 → COMMITTED ✅ (saved: position=500)
  Chunk 2 → COMMITTED ✅ (saved: position=1000)
  Chunk 3 → COMMITTED ✅ (saved: position=1500)
  Chunk 4 → IN-PROGRESS → 💥 CRASH
  
  On restart:
  → Load position=1500 from ExecutionContext
  → Reader starts at 1500 (chunks 1-3 skipped)
  → Chunk 4 re-processed from 1501

Database Persistence:
  BATCH_STEP_EXECUTION → step status
  BATCH_STEP_EXECUTION_CONTEXT → reader position, custom data
  → Survives JVM crash, server restart, deployment
```

### 🗣️ How to Say in Interview
"Spring Batch's restartability is built on three pillars. First, step-level tracking — completed steps are skipped on restart. Second, chunk-level checkpointing — after each chunk commits, the reader's position is saved in the ExecutionContext. On restart, the reader initializes at the last checkpoint, so we only re-process the failed chunk. Third, database persistence — all this metadata is stored in the batch tables, so even a JVM crash doesn't lose restart information. In my project, our nightly job processing 10 million records crashed at 3 AM. The 6 AM restart picked up from the 8-millionth record because 80% was already checkpointed."

### 💻 Code
```java
// Built-in readers auto-save position
// FlatFileItemReader → saves line number
// JdbcPagingItemReader → saves page/offset

// Custom reader with checkpoint support
@Component
public class CheckpointReader implements ItemStreamReader<Record> {
    private int position = 0;

    @Override
    public void open(ExecutionContext ctx) {
        if (ctx.containsKey("position")) {
            position = ctx.getInt("position");  // restore on restart
        }
    }

    @Override
    public Record read() {
        return hasMore() ? fetchAt(position++) : null;
    }

    @Override
    public void update(ExecutionContext ctx) {
        ctx.putInt("position", position);  // save after each chunk
    }
}

// Disable restartability for a step (rare)
@Bean
public Step nonRestartableStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("nonRestartable", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .allowStartIfComplete(false)  // default: don't re-run completed
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Multi-threaded steps LOSE restartability — threads read out of order, position tracking breaks *(multi-threaded step mein restart reliable nahi)*
- Custom readers MUST implement `ItemStreamReader` with open/update/close for restart
- Partitioned steps restart at partition level (failed partitions only)
- Database metadata tables must be on reliable storage (not in-memory DB in production)

### ⚡ Remember
- Three pillars: step status + chunk checkpoint + DB persistence
- At most one chunk lost on crash *(zyada se zyada ek chunk ka loss)*
- ExecutionContext = checkpoint storage
- Built-in readers auto-checkpoint; custom readers need ItemStreamReader
- Multi-threaded = no reliable restart

### 🔗 Follow-ups
- [Q62 → Internal restart flow](#q62)
- [Q67 → ExecutionContext details](#q67)
- [Q69 → Resume from failure point](#q69)

---

## Q67. How does ExecutionContext work?

### 📝 One-Liner
ExecutionContext is a serialized key-value map stored in the database — job-level context is shared across steps, step-level context is private to one step.

### 🔑 Quick Answer
ExecutionContext is a `Map<String, Object>` that persists between job/step executions. Two scopes: **Job ExecutionContext** — shared across all steps (store data to pass between steps). **Step ExecutionContext** — private to one step (store reader position, custom counters). It's saved to the database after every chunk commit, making it survive crashes. Access via `@BeforeStep` injection or SpEL expressions. *(Do scope: Job context = sab steps ke liye shared, Step context = ek step ka private)*

### 📖 How It Works
```
ExecutionContext Scopes:

Job ExecutionContext (shared):
┌─────────────────────────────────┐
│ key: "totalOrders" → 5000       │ ← shared across all steps
│ key: "fileDate" → "2024-01-15"  │
│ Stored in: BATCH_JOB_EXECUTION_CONTEXT
└─────────────────────────────────┘
  ↕ accessible from Step 1, Step 2, Step 3

Step ExecutionContext (private):
┌─────────────────────────────────┐
│ Step 1 context:                 │
│   reader.position → 7500        │ ← only Step 1 can see this
│   commit.count → 15             │
│ Stored in: BATCH_STEP_EXECUTION_CONTEXT
└─────────────────────────────────┘

Persistence: saved after EVERY chunk commit → survives crashes
```

### 🗣️ How to Say in Interview
"ExecutionContext is a persistent key-value map with two scopes. Job-level context is shared across all steps — useful for passing data like file names or summary counts between steps. Step-level context is private to one step — used by readers for checkpoint positions and custom counters. It's serialized and saved to the database after every chunk commit, which is what makes restart work. In my project, we stored the total record count in job context from the validation step, then used it in the report step to verify data integrity after processing."

### 💻 Code
```java
// Accessing Step ExecutionContext
@Component
public class ContextAwareProcessor implements ItemProcessor<Order, Order>, StepExecutionListener {
    
    private StepExecution stepExecution;

    @Override
    public void beforeStep(StepExecution se) {
        this.stepExecution = se;
    }

    @Override
    public Order process(Order order) {
        // Write to step context (private to this step)
        ExecutionContext stepCtx = stepExecution.getExecutionContext();
        stepCtx.putInt("processedCount", 
            stepCtx.getInt("processedCount", 0) + 1);

        // Write to job context (shared across steps)
        ExecutionContext jobCtx = stepExecution.getJobExecution().getExecutionContext();
        jobCtx.putString("lastProcessedId", order.getId().toString());

        return order;
    }
}

// Pass data between steps using Job ExecutionContext
@Bean
public Step step1(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step1", repo)
            .tasklet((contribution, chunkContext) -> {
                // Write to job context
                chunkContext.getStepContext().getStepExecution()
                    .getJobExecution().getExecutionContext()
                    .putInt("totalRecords", countRecords());
                return RepeatStatus.FINISHED;
            }, tx).build();
}

// Read from job context in step 2 using SpEL
@Bean
@StepScope
public ItemReader<Order> step2Reader(
        @Value("#{jobExecutionContext['totalRecords']}") int totalRecords) {
    log.info("Step 1 found {} records", totalRecords);
    return buildReader(totalRecords);
}
```

### ⚠️ Pitfalls / Gotchas
- Context is serialized — only Serializable values can be stored *(sirf Serializable objects store ho sakte hain)*
- SHORT_CONTEXT column has 2500 char limit — large data goes to SERIALIZED_CONTEXT
- Don't store huge objects — context is saved EVERY chunk commit
- Job context is loaded once at job start; step context is loaded at step start
- SpEL: `#{jobExecutionContext['key']}` for job-level, `#{stepExecutionContext['key']}` for step-level

### ⚡ Remember
- Two scopes: Job (shared) and Step (private) *(Job context shared, Step context private)*
- Saved after every chunk commit → survives crashes
- Only Serializable values allowed
- SpEL for late-binding: `#{jobExecutionContext['key']}`
- Don't store large data — saved frequently

### 🔗 Follow-ups
- [Q68 → Where context is stored in DB](#q68)
- [Q66 → Restartability using context](#q66)
- [Q62 → Context in restart flow](#q62)

---

## Q68. Where is ExecutionContext stored?

### 📝 One-Liner
Job context in BATCH_JOB_EXECUTION_CONTEXT table, step context in BATCH_STEP_EXECUTION_CONTEXT table — serialized as JSON in Spring Batch 5.

### 🔑 Quick Answer
Two database tables: **BATCH_JOB_EXECUTION_CONTEXT** stores job-level context (one row per JobExecution). **BATCH_STEP_EXECUTION_CONTEXT** stores step-level context (one row per StepExecution). Each table has `SHORT_CONTEXT` (VARCHAR 2500) for small data and `SERIALIZED_CONTEXT` (CLOB) for overflow. Spring Batch 5 uses JSON serialization (Batch 4 used Java serialization by default). *(Do tables — ek job ke liye, ek step ke liye — JSON format mein serialize hota hai)*

### 📖 How It Works
```
Database Storage:

BATCH_JOB_EXECUTION_CONTEXT:
┌──────────────────┬─────────────────────────────────────┐
│ JOB_EXECUTION_ID │ SHORT_CONTEXT                       │
│ 1                │ {"totalRecords": 5000, "file": ...} │
│                  │ SERIALIZED_CONTEXT (CLOB for large)  │
└──────────────────┴─────────────────────────────────────┘

BATCH_STEP_EXECUTION_CONTEXT:
┌───────────────────┬────────────────────────────────────┐
│ STEP_EXECUTION_ID │ SHORT_CONTEXT                      │
│ 1                 │ {"reader.position": 7500, ...}     │
│                   │ SERIALIZED_CONTEXT (CLOB if needed) │
└───────────────────┴────────────────────────────────────┘

Serialization:
  Spring Batch 5: JSON (human-readable, debuggable)
  Spring Batch 4: Java serialization (binary, fragile)
```

### 🗣️ How to Say in Interview
"ExecutionContext is stored in two database tables — BATCH_JOB_EXECUTION_CONTEXT for job-level data and BATCH_STEP_EXECUTION_CONTEXT for step-level data. Each has a SHORT_CONTEXT column for small data and a SERIALIZED_CONTEXT CLOB for larger objects. Spring Batch 5 uses JSON serialization, which makes debugging much easier compared to Batch 4's Java serialization. In my project, when troubleshooting restart issues, I directly queried BATCH_STEP_EXECUTION_CONTEXT to check the reader's saved position and verify it matched our expected checkpoint."

### 💻 Code
```sql
-- Query job context
SELECT jec.JOB_EXECUTION_ID, jec.SHORT_CONTEXT
FROM BATCH_JOB_EXECUTION_CONTEXT jec
JOIN BATCH_JOB_EXECUTION je ON je.JOB_EXECUTION_ID = jec.JOB_EXECUTION_ID
WHERE je.JOB_INSTANCE_ID = 123;

-- Query step context (check reader position)
SELECT se.STEP_NAME, sec.SHORT_CONTEXT
FROM BATCH_STEP_EXECUTION_CONTEXT sec
JOIN BATCH_STEP_EXECUTION se ON se.STEP_EXECUTION_ID = sec.STEP_EXECUTION_ID
WHERE se.JOB_EXECUTION_ID = 456;

-- Example SHORT_CONTEXT value (JSON in Batch 5):
-- {"@class":"java.util.HashMap","reader.position":7500,"commit.count":15}
```

### ⚠️ Pitfalls / Gotchas
- SHORT_CONTEXT has 2500 char limit — storing large objects silently moves to SERIALIZED_CONTEXT *(2500 character ke baad CLOB mein chala jaata hai)*
- Java serialization (Batch 4) breaks if class changes between deployments
- JSON serialization (Batch 5) is more resilient to class changes
- Don't query these tables in production without proper indexes

### ⚡ Remember
- Job context → BATCH_JOB_EXECUTION_CONTEXT
- Step context → BATCH_STEP_EXECUTION_CONTEXT *(do tables yaad rakho)*
- SHORT_CONTEXT (2500 chars) + SERIALIZED_CONTEXT (CLOB overflow)
- Batch 5 = JSON, Batch 4 = Java serialization
- Directly queryable for debugging

### 🔗 Follow-ups
- [Q67 → ExecutionContext usage](#q67)
- [Q110 → All metadata tables](#q110)
- [Q62 → Context used in restart](#q62)

---

## Q69. How does Spring Batch resume from the failure point?

### 📝 One-Liner
On restart, Spring Batch loads the reader's last saved position from ExecutionContext and initializes the reader at that position — skipping already-committed data.

### 🔑 Quick Answer
The resume mechanism: **(1)** Restart launches same JobInstance (same params). **(2)** For failed steps, Spring Batch loads the `ExecutionContext` from BATCH_STEP_EXECUTION_CONTEXT. **(3)** The context contains the reader's last saved position (line number, page offset, etc.). **(4)** Reader's `open(ExecutionContext)` method reads this position and initializes there. **(5)** Processing resumes from the next unprocessed chunk. At most ONE chunk of work is lost — the in-progress chunk at crash time. *(ExecutionContext se reader ka position load karta hai — wahi se shuru hota hai jahan chhoda tha)*

### 📖 How It Works
```
Resume Flow:

Original Run:
  Chunk 1 → COMMIT (save position=500)
  Chunk 2 → COMMIT (save position=1000)
  Chunk 3 → COMMIT (save position=1500)
  Chunk 4 → 💥 CRASH at item 1750
  
  ExecutionContext in DB: {position: 1500}  ← saved after last COMMIT

Restart:
  1. Load ExecutionContext → position: 1500
  2. reader.open(context) → initialize at position 1500
  3. Chunk 4 → READ from 1501 (items 1501-1750 re-read)
  4. Chunk 5 → continues from 2001
  5. ...until complete

Lost work: items 1501-1750 (one chunk) → re-processed on restart
```

### 🗣️ How to Say in Interview
"On restart, Spring Batch loads the ExecutionContext from the database, which contains the reader's last checkpointed position. The reader's open method receives this context and initializes at the saved position. This means all previously committed chunks are skipped and processing resumes from the last uncommitted chunk. At most one chunk of work is re-processed. In my project, our FlatFileItemReader saved line numbers and our JdbcPagingItemReader saved page offsets — both automatically handled by the framework. On a crash at 3 million out of 10 million records, the restart only re-processed 500 items (one chunk) plus the remaining 7 million."

### 💻 Code
```java
// Built-in readers handle this automatically:
// FlatFileItemReader → saves/restores line count
// JdbcPagingItemReader → saves/restores page number

// Verify resume point programmatically
@Bean
public StepExecutionListener resumeVerifier() {
    return new StepExecutionListener() {
        @Override
        public void beforeStep(StepExecution se) {
            ExecutionContext ctx = se.getExecutionContext();
            if (!ctx.isEmpty()) {
                log.info("RESUMING step '{}' from context: {}", 
                    se.getStepName(), ctx.entrySet());
            } else {
                log.info("FRESH START for step '{}'", se.getStepName());
            }
        }
    };
}

// Custom reader with resume support
@Component
public class ResumableApiReader implements ItemStreamReader<ApiRecord> {
    private int offset = 0;

    @Override
    public void open(ExecutionContext ctx) {
        if (ctx.containsKey("api.offset")) {
            this.offset = ctx.getInt("api.offset");
            log.info("Resuming API reader from offset: {}", offset);
        }
    }

    @Override
    public ApiRecord read() {
        return hasMore(offset) ? fetchAndIncrement() : null;
    }

    @Override
    public void update(ExecutionContext ctx) {
        ctx.putInt("api.offset", offset);  // checkpoint after each chunk
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Multi-threaded steps break resume — threads read items out of order *(multi-threaded mein resume kaam nahi karta)*
- If source data changed between runs (new inserts, deletes), offset-based resume may miss or duplicate data
- Custom readers without `ItemStreamReader` implementation → no resume support
- Partitioned steps resume at partition level — only failed partitions re-run

### ⚡ Remember
- ExecutionContext stores reader position → loaded on restart
- Reader's `open()` restores position, `update()` saves position *(open = restore, update = save)*
- At most one chunk lost on crash
- Built-in readers handle resume automatically
- Multi-threaded steps = resume not reliable

### 🔗 Follow-ups
- [Q66 → Restartability pillars](#q66)
- [Q62 → Full restart internal flow](#q62)
- [Q67 → ExecutionContext details](#q67)
