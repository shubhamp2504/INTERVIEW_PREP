# 🟡 Database & Metadata Tables — Q110 to Q116

---

## Q110. What tables does Spring Batch create?

### 📝 One-Liner
6 metadata tables + 3 sequence tables = 9 total. Core hierarchy: JOB_INSTANCE → JOB_EXECUTION → STEP_EXECUTION, plus params and context tables.

### 🔑 Quick Answer
**6 tables**: **(1) BATCH_JOB_INSTANCE** — unique job identity (name + params hash). **(2) BATCH_JOB_EXECUTION** — each attempt to run an instance. **(3) BATCH_JOB_EXECUTION_PARAMS** — parameters per execution. **(4) BATCH_JOB_EXECUTION_CONTEXT** — job-level shared state (across steps). **(5) BATCH_STEP_EXECUTION** — per-step detailed stats (read/write/skip counts). **(6) BATCH_STEP_EXECUTION_CONTEXT** — step-level state (reader position for restart). **3 sequences**: for generating IDs. These tables enable restart, monitoring, and duplicate prevention. *(6 tables + 3 sequences = 9 total — ye sab automatically create hote hain)*

### 📖 How It Works
```
Schema Hierarchy:

BATCH_JOB_INSTANCE (unique identity)
  │ JOB_NAME + JOB_KEY (hash of identifying params)
  │
  ├── BATCH_JOB_EXECUTION (each attempt)
  │   │ STATUS, START_TIME, END_TIME, EXIT_CODE, EXIT_MESSAGE
  │   │
  │   ├── BATCH_JOB_EXECUTION_PARAMS (parameters)
  │   │   PARAMETER_NAME, PARAMETER_TYPE, PARAMETER_VALUE, IDENTIFYING
  │   │
  │   ├── BATCH_JOB_EXECUTION_CONTEXT (job-level shared state)
  │   │   SHORT_CONTEXT (JSON: data shared between steps)
  │   │
  │   └── BATCH_STEP_EXECUTION (per-step stats)
  │       │ STEP_NAME, STATUS, READ_COUNT, WRITE_COUNT, SKIP_COUNT,
  │       │ FILTER_COUNT, COMMIT_COUNT, ROLLBACK_COUNT
  │       │
  │       └── BATCH_STEP_EXECUTION_CONTEXT (step-level state)
  │           SHORT_CONTEXT (JSON: reader position for restart)

Sequences:
  BATCH_JOB_SEQ, BATCH_JOB_EXECUTION_SEQ, BATCH_STEP_EXECUTION_SEQ

Table Purposes:
  JOB_INSTANCE    → "What job?" (identity + uniqueness)
  JOB_EXECUTION   → "How did it go?" (status, timing)
  STEP_EXECUTION  → "What happened in detail?" (counts)
  *_CONTEXT       → "Where were we?" (restart state)
  *_PARAMS        → "With what inputs?" (parameters)
```

### 🗣️ How to Say in Interview
"Spring Batch creates 6 metadata tables and 3 sequence tables. The hierarchy flows from BATCH_JOB_INSTANCE — which stores unique job identities based on job name and parameter hash — to BATCH_JOB_EXECUTION which records each attempt to run that instance. Each execution links to its parameters, a job-level context for sharing data between steps, and BATCH_STEP_EXECUTION which stores detailed per-step statistics like read, write, skip, and rollback counts. The step execution context stores the reader position enabling restart from where it left off. These tables are the backbone of Spring Batch's restartability, monitoring, and duplicate prevention."

### 💻 Code
```java
// Auto-create tables (Spring Boot)
// spring.batch.jdbc.initialize-schema=always  (dev)
// spring.batch.jdbc.initialize-schema=never   (production — use Flyway/Liquibase)

// Manual creation — use scripts from Spring Batch:
// org/springframework/batch/core/schema-*.sql
// e.g., schema-mysql.sql, schema-postgresql.sql, schema-h2.sql

// Query all tables
// SELECT table_name FROM information_schema.tables
// WHERE table_name LIKE 'BATCH_%';
// → BATCH_JOB_INSTANCE
// → BATCH_JOB_EXECUTION
// → BATCH_JOB_EXECUTION_PARAMS
// → BATCH_JOB_EXECUTION_CONTEXT
// → BATCH_STEP_EXECUTION
// → BATCH_STEP_EXECUTION_CONTEXT
// → BATCH_JOB_SEQ
// → BATCH_JOB_EXECUTION_SEQ
// → BATCH_STEP_EXECUTION_SEQ
```

### ⚠️ Pitfalls / Gotchas
- `initialize-schema=always` in production can DROP and recreate tables — use `never` with Flyway *(production mein always mat use karo — data ud jayega)*
- Tables grow indefinitely — implement periodic cleanup
- For remote partitioning, all JVMs must share the SAME metadata database
- H2 (in-memory) loses all metadata on restart — use real DB for production

### ⚡ Remember
- **6 tables + 3 sequences** *(6 tables + 3 sequences = 9 total)*
- Hierarchy: INSTANCE → EXECUTION → STEP_EXECUTION
- INSTANCE = identity, EXECUTION = attempt, STEP = details
- CONTEXT tables = restart state
- Production: `initialize-schema=never` + Flyway

### 🔗 Follow-ups
- [Q111 → BATCH_JOB_INSTANCE details](#q111)
- [Q112 → BATCH_JOB_EXECUTION details](#q112)
- [Q113 → BATCH_STEP_EXECUTION details](#q113)

---

## Q111. What is BATCH_JOB_INSTANCE?

### 📝 One-Liner
Stores unique job identities — each row = unique combination of job name + identifying parameters (JOB_KEY = MD5 hash of params).

### 🔑 Quick Answer
BATCH_JOB_INSTANCE stores unique job identities. Uniqueness is determined by **job name + JOB_KEY** (MD5 hash of identifying parameters). Same job name + same identifying params = same instance → cannot re-run if COMPLETED. Different params = new instance → always allowed. Key columns: JOB_INSTANCE_ID, VERSION, JOB_NAME, JOB_KEY. If a COMPLETED job is launched with same params → `JobInstanceAlreadyCompleteException`. If FAILED → restart allowed (same instance, new execution). *(Ek job name + same params = same instance — COMPLETED ho toh dobara nahi chalega)*

### 📖 How It Works
```
BATCH_JOB_INSTANCE Table:
  JOB_INSTANCE_ID | VERSION | JOB_NAME    | JOB_KEY
  1               | 0       | paymentJob  | abc123...  (hash of date=2024-01-01)
  2               | 0       | paymentJob  | def456...  (hash of date=2024-01-02)
  3               | 0       | reportJob   | abc123...  (hash of type=monthly)

Uniqueness via JOB_KEY (MD5 hash of identifying params):
  paymentJob + {date=2024-01-01} → JOB_KEY = abc123
  paymentJob + {date=2024-01-02} → JOB_KEY = def456 (different!)
  paymentJob + {date=2024-01-01} → JOB_KEY = abc123 (SAME → rejected if completed)

Scenarios:
  1. New params → new instance → ✅ always runs
  2. Same params + COMPLETED → ❌ JobInstanceAlreadyCompleteException
  3. Same params + FAILED → ✅ restart (new execution for same instance)
```

### 🗣️ How to Say in Interview
"BATCH_JOB_INSTANCE stores the unique identity of each job run, determined by job name plus a JOB_KEY which is an MD5 hash of the identifying parameters. If I launch a job with the same parameters and the previous instance completed successfully, Spring Batch throws JobInstanceAlreadyCompleteException — this prevents duplicate processing. If the previous instance failed, it allows a restart by creating a new execution under the same instance. In my project, we use the processing date as an identifying parameter, so each day creates a new instance while preventing accidental re-runs of the same day's data."

### 💻 Code
```java
// Parameters that create unique instances
JobParameters params = new JobParametersBuilder()
        .addString("date", "2024-01-15")              // IDENTIFYING (default)
        .addString("note", "test run", false)          // NON-IDENTIFYING (false)
        .toJobParameters();
// JOB_KEY = hash of "date=2024-01-15" only (non-identifying excluded)

// Re-running with same params after COMPLETED → exception
try {
    jobLauncher.run(paymentJob, sameParams);
} catch (JobInstanceAlreadyCompleteException e) {
    // "A job instance already exists and is complete for parameters={date=2024-01-15}"
}

// To always allow re-run → add unique param
JobParameters params = new JobParametersBuilder()
        .addString("date", "2024-01-15")
        .addLocalDateTime("runTime", LocalDateTime.now())  // unique each time
        .toJobParameters();
```

### ⚠️ Pitfalls / Gotchas
- Non-identifying params (false flag) do NOT affect JOB_KEY → same params despite different notes *(non-identifying params JOB_KEY mein nahi jaate)*
- JOB_KEY is MD5 hash → can't see original params from it
- Batch 5 changed default: all params are identifying unless explicitly non-identifying
- Table grows indefinitely — each day's run adds a row forever

### ⚡ Remember
- Unique = job name + JOB_KEY (MD5 of identifying params) *(naam + params ka hash = uniqueness)*
- COMPLETED + same params = rejected
- FAILED + same params = restart allowed
- Add timestamp for guaranteed uniqueness
- Non-identifying params excluded from hash

### 🔗 Follow-ups
- [Q112 → BATCH_JOB_EXECUTION](#q112)
- [Q55 → JobInstance concept](#q55)
- [Q114 → BATCH_JOB_EXECUTION_PARAMS](#q114)

---

## Q112. What is BATCH_JOB_EXECUTION?

### 📝 One-Liner
Stores each attempt to run a job instance — 1 instance can have N executions (first run FAILED, restart COMPLETED = 2 rows).

### 🔑 Quick Answer
BATCH_JOB_EXECUTION records every attempt to run a job instance. One instance can have **multiple executions** (fail + restart = 2 rows, fail + fail + success = 3 rows). Key columns: STATUS (STARTED, COMPLETED, FAILED, STOPPED), EXIT_CODE, EXIT_MESSAGE (exception details), START_TIME, END_TIME, JOB_INSTANCE_ID (FK to instance). This is the **first table to check** when debugging — tells you if the job succeeded or failed and why. *(Ek instance ke multiple executions ho sakte hain — fail + restart = 2 rows)*

### 📖 How It Works
```
Example: 3 executions for 1 instance

JOB_EXECUTION_ID | JOB_INSTANCE_ID | STATUS    | EXIT_MESSAGE
1                | 100             | FAILED    | "Disk full: cannot write..."
2                | 100             | FAILED    | "NullPointerException at..."
3                | 100             | COMPLETED | ""

Instance 100 was:
  Execution 1: FAILED (disk full) → fixed disk space
  Execution 2: FAILED (code bug) → fixed NullPointer
  Execution 3: COMPLETED ✅

Status Lifecycle:
  STARTING → STARTED → COMPLETING → COMPLETED ✅
                     → FAILED ❌
                     → STOPPING → STOPPED ⏸️

Key Columns:
  STATUS:       STARTED, COMPLETED, FAILED, STOPPED, ABANDONED
  EXIT_CODE:    COMPLETED, FAILED, UNKNOWN, NOOP, custom
  EXIT_MESSAGE: Exception stack trace (most useful for debugging)
  START_TIME:   When execution started
  END_TIME:     When execution ended (NULL if STARTED/crashed)
```

### 🗣️ How to Say in Interview
"BATCH_JOB_EXECUTION records every attempt to run a job instance. A single instance can have multiple executions — if the first run fails and we restart, that creates a second execution row under the same instance. The EXIT_MESSAGE column is the most valuable for debugging — it contains the actual exception that caused the failure. I always check this table first when investigating a batch issue. In my project, we had a job instance with 3 executions — first failed from a disk issue, second from a code bug, third completed successfully — all traceable through this table."

### 💻 Code
```java
// Query job executions
// SELECT JOB_EXECUTION_ID, STATUS, EXIT_CODE, EXIT_MESSAGE,
//        START_TIME, END_TIME
// FROM BATCH_JOB_EXECUTION
// WHERE JOB_INSTANCE_ID = 100
// ORDER BY JOB_EXECUTION_ID DESC;

// Programmatic access via JobExplorer
List<JobExecution> executions = jobExplorer.getJobExecutions(jobInstance);
for (JobExecution exec : executions) {
    log.info("Execution #{}: status={}, exit={}",
            exec.getId(), exec.getStatus(), exec.getExitStatus().getExitCode());
    if (exec.getStatus() == BatchStatus.FAILED) {
        log.error("  Failure: {}", exec.getExitStatus().getExitDescription());
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Only ONE execution can be STARTED at a time for an instance — concurrent launch throws exception *(ek instance ka ek hi execution STARTED ho sakta hai — concurrent nahi chalega)*
- After JVM crash, status stays STARTED (stale) — must manually mark FAILED before restart
- EXIT_MESSAGE can be very long (full stack traces) — check column size in your DB
- END_TIME is NULL for STARTED executions — useful for detecting stale/hung jobs

### ⚡ Remember
- 1 instance → N executions (fail + restart pattern) *(ek instance ke multiple attempts)*
- **First debugging table** — check STATUS and EXIT_MESSAGE
- StatusL: STARTING → STARTED → COMPLETED/FAILED/STOPPED
- EXIT_MESSAGE = actual exception
- Only 1 STARTED execution per instance at a time

### 🔗 Follow-ups
- [Q111 → BATCH_JOB_INSTANCE](#q111)
- [Q113 → BATCH_STEP_EXECUTION (detailed counts)](#q113)
- [Q106 → Debug failed jobs](#q106)

---

## Q113. What is BATCH_STEP_EXECUTION?

### 📝 One-Liner
Stores detailed execution statistics per step: read, write, skip, filter, commit, rollback counts — the most useful table for diagnosing batch issues.

### 🔑 Quick Answer
BATCH_STEP_EXECUTION is the **most useful debugging table**. It stores per-step statistics including READ_COUNT, WRITE_COUNT, SKIP_COUNT (read + process + write skips), FILTER_COUNT (processor returned null), COMMIT_COUNT, and ROLLBACK_COUNT. The math must work: **read - filter - skip = write**. Example: read=50000, filter=1485, skip=15, write=48500 → 50000 - 1485 - 15 = 48500 ✅. Rollback count should be near 0. *(Ye sabse useful table hai — read, write, skip, rollback sab milta hai)*

### 📖 How It Works
```
BATCH_STEP_EXECUTION (key columns):

STEP_EXECUTION_ID | STEP_NAME       | STATUS    | READ | WRITE | SKIP | FILTER | COMMIT | ROLLBACK
42                | processPayments | COMPLETED | 50000| 48500 | 15   | 1485   | 97     | 3

Math Check:
  READ - FILTER - SKIP = WRITE
  50000 - 1485 - 15 = 48500 ✅

What Each Count Tells You:
  READ_COUNT:     Total items read from source
  FILTER_COUNT:   Processor returned null (intentionally filtered)
  SKIP_COUNT:     Items skipped due to errors (read + process + write skips combined)
  WRITE_COUNT:    Items successfully written
  COMMIT_COUNT:   Successful chunk commits (≈ read / chunkSize)
  ROLLBACK_COUNT: Chunk failures rolled back (should be near 0)

Red Flags:
  ❌ SKIP_COUNT = 500 (high) → data quality issues
  ❌ ROLLBACK_COUNT = 10 (high) → transient DB/network failures
  ❌ WRITE_COUNT < expected → items lost or filtered
  ❌ COMMIT_COUNT much lower than READ/chunkSize → large chunks or failures
```

### 🗣️ How to Say in Interview
"BATCH_STEP_EXECUTION is the most useful table for diagnosing batch issues. It stores detailed per-step statistics — read, write, skip, filter, commit, and rollback counts. The key formula is read minus filter minus skip equals write — if this math doesn't add up, something is wrong. Skip count tells me about data quality issues, rollback count about transient failures, and filter count about intentional exclusions. In my project, when we saw 500 skips in a daily job that normally has 5, we immediately knew the upstream data had quality issues. The combination of counts tells the complete story of what happened during processing."

### 💻 Code
```java
// Query step execution details
// SELECT step_name, status, read_count, write_count, skip_count,
//        filter_count, commit_count, rollback_count,
//        exit_code, exit_message
// FROM BATCH_STEP_EXECUTION
// WHERE JOB_EXECUTION_ID = ?
// ORDER BY STEP_EXECUTION_ID;

// Programmatic access
for (StepExecution step : jobExecution.getStepExecutions()) {
    log.info("Step '{}': read={}, written={}, skipped={}, filtered={}, " +
             "commits={}, rollbacks={}",
            step.getStepName(),
            step.getReadCount(),
            step.getWriteCount(),
            step.getSkipCount(),
            step.getFilterCount(),
            step.getCommitCount(),
            step.getRollbackCount());
    
    // Verify math
    int expected = step.getReadCount() - step.getFilterCount() - step.getSkipCount();
    if (expected != step.getWriteCount()) {
        log.warn("Count mismatch! Expected write={}, actual={}", expected, step.getWriteCount());
    }
}
```

### ⚠️ Pitfalls / Gotchas
- SKIP_COUNT combines read, process, AND write skips into one number — use SkipListener for per-phase breakdown *(skip count teen phases ka combined hai — detail ke liye SkipListener lagao)*
- FILTER_COUNT > 0 is usually intentional (processor null = filter) — not an error
- ROLLBACK_COUNT > 0 means chunks were retried before succeeding or being skipped
- For partitioned steps, each partition has its OWN step execution row

### 🎯 Tricky Interview Qs

**Q: read=50000, write=48500, skip=15, filter=0 — what happened to 1485 records?**
Math doesn't add up: 50000 - 0 - 15 = 49985, not 48500. Either the counts are wrong or there's a bug. Most likely FILTER_COUNT wasn't updated (processor returned null but filter count didn't increment — happens with custom processors that don't follow conventions).

**Q: commit=97, chunk=500 — why not commit=100 for 50000 records?**
50000 / 500 = 100 expected commits. But 3 rollbacks happened (chunks failed and retried, some items skipped) → 97 successful commits + 3 rollback-retry cycles ≈ 100.

### ⚡ Remember
- **read - filter - skip = write** (math must add up) *(ye formula yaad rakhna — agar match nahi karta toh problem hai)*
- Most useful debugging table
- SKIP_COUNT = combined (read + process + write skips)
- ROLLBACK_COUNT should be near 0
- Partitioned step: 1 row per partition

### 🔗 Follow-ups
- [Q112 → BATCH_JOB_EXECUTION](#q112)
- [Q115 → BATCH_STEP_EXECUTION_CONTEXT](#q115)
- [Q106 → Debug failed jobs](#q106)

---

## Q114. What is BATCH_JOB_EXECUTION_PARAMS?

### 📝 One-Liner
Stores parameters per execution with an IDENTIFYING flag — identifying params affect instance uniqueness (JOB_KEY hash), non-identifying are metadata only.

### 🔑 Quick Answer
Stores JobParameters passed to each execution. Key column: **IDENTIFYING** flag. Identifying (Y) → included in JOB_KEY hash → different value = different job instance. Non-identifying (N) → metadata only, doesn't affect uniqueness. Example: `date=2024-01-15` (IDENTIFYING) determines uniqueness, `description="test run"` (NON-IDENTIFYING) is just a note. Viewable via JobExplorer for audit trail. *(IDENTIFYING params se uniqueness decide hoti hai — non-identifying sirf note hai)*

### 📖 How It Works
```
BATCH_JOB_EXECUTION_PARAMS:

JOB_EXECUTION_ID | PARAMETER_NAME | PARAMETER_TYPE | PARAMETER_VALUE | IDENTIFYING
1                | date           | STRING         | 2024-01-15      | Y
1                | filePath       | STRING         | /data/input.csv | Y
1                | description    | STRING         | Daily payment   | N
1                | runTime        | DATE           | 2024-01-15T02:00| Y

IDENTIFYING = Y → affects JOB_KEY hash → uniqueness
  date=2024-01-15 + filePath=/data/input.csv + runTime=...
  → JOB_KEY = MD5(these values)

IDENTIFYING = N → metadata only → NOT in hash
  description="Daily payment" → just a label

Consequence:
  Same date + filePath + runTime = same JOB_KEY = same instance
  Same date + filePath + runTime + different description = STILL same instance
```

### 🗣️ How to Say in Interview
"BATCH_JOB_EXECUTION_PARAMS stores all parameters passed to a job execution. The critical column is IDENTIFYING — when set to Y, that parameter is included in the JOB_KEY hash that determines instance uniqueness. When N, it's just metadata. In my project, we use the processing date and file path as identifying parameters, ensuring each day's file creates a unique instance. We add a description as non-identifying for audit purposes — it doesn't affect whether the job can re-run. Spring Batch 5 changed the default: all parameters are identifying unless explicitly marked non-identifying."

### 💻 Code
```java
// Creating parameters with identifying flag
JobParameters params = new JobParametersBuilder()
        .addString("date", "2024-01-15")              // identifying (default in Batch 5)
        .addString("filePath", "/data/input.csv")      // identifying
        .addString("description", "Daily run", false)   // NON-identifying (false)
        .addLocalDateTime("runTime", LocalDateTime.now()) // identifying
        .toJobParameters();

// Query params for an execution
// SELECT parameter_name, parameter_type, parameter_value, identifying
// FROM BATCH_JOB_EXECUTION_PARAMS
// WHERE JOB_EXECUTION_ID = ?;

// Programmatic access
JobParameters params = jobExecution.getJobParameters();
String date = params.getString("date");
String desc = params.getString("description");
```

### ⚠️ Pitfalls / Gotchas
- Spring Batch 5: ALL params are identifying by default — pass `false` explicitly for non-identifying *(Batch 5 mein sab params identifying hain by default — dhyan rakhna)*
- Spring Batch 4: used type-based methods (addString/addLong/addDate)
- Spring Batch 5: unified to String-based with type conversion
- Non-identifying params still stored — just not in JOB_KEY hash

### ⚡ Remember
- **IDENTIFYING=Y** → affects JOB_KEY → uniqueness *(identifying = uniqueness decide karta hai)*
- **IDENTIFYING=N** → metadata only
- Batch 5: all params identifying by default
- Add `false` for non-identifying: `addString("key", "val", false)`
- Viewable via JobExplorer for audit

### 🔗 Follow-ups
- [Q111 → BATCH_JOB_INSTANCE (where JOB_KEY is)](#q111)
- [Q55 → JobInstance uniqueness](#q55)
- [Q56 → JobInstance vs JobExecution](#q56)

---

## Q115. What is BATCH_STEP_EXECUTION_CONTEXT?

### 📝 One-Liner
Stores serialized step state as JSON — most importantly the reader position — enabling restart from the last committed chunk.

### 🔑 Quick Answer
BATCH_STEP_EXECUTION_CONTEXT stores step-level state as **JSON** (Spring Batch 5) or serialized Java (older versions). Primary use: **reader position for restart**. After each chunk commit, the reader position is saved. On restart, the reader reads this position and skips ahead to resume processing. Example: `{"FlatFileItemReader.read.count": 25000}` → on restart, skip 25000 records and continue from 25001. You can also store custom data via `executionContext.put()`. *(Reader ki position save hoti hai — restart pe wahi se shuru hota hai)*

### 📖 How It Works
```
How Restart Works via Context:

First Run (fails at record 25000):
  Chunk 1: read 1-500     → commit → save context: {read.count: 500}
  Chunk 2: read 501-1000  → commit → save context: {read.count: 1000}
  ...
  Chunk 50: read 24501-25000 → FAIL
  Context saved: {read.count: 25000}  ← last committed position

Restart:
  Framework reads context: {read.count: 25000}
  Reader skips records 1-25000
  Resumes from record 25001
  → No duplicate processing!

Context Examples by Reader Type:
  FlatFileItemReader:   {"FlatFileItemReader.read.count": 25000}
  JdbcPagingItemReader: {"JdbcPagingItemReader.start.after": {"id": 25000}}
                        {"JdbcPagingItemReader.read.count": 25000}
```

### 🗣️ How to Say in Interview
"BATCH_STEP_EXECUTION_CONTEXT stores the step's state as JSON, primarily the reader position. After every chunk commit, the current reader position is saved to this table. When a job restarts, the framework reads this context and tells the reader to skip ahead to the last committed position. This is how Spring Batch achieves restart without reprocessing — if a job fails at record 25,000, restart picks up from 25,001. In my project, I also use it to store custom state like running totals or error counters that I need to preserve across restarts."

### 💻 Code
```java
// Context is saved automatically after each chunk commit
// Reader position stored by Spring Batch framework

// Custom data in step context
@Component
@StepScope
public class StatefulProcessor implements ItemProcessor<Order, Order>, StepExecutionListener {
    private StepExecution stepExecution;
    private int errorCount = 0;

    @Override
    public void beforeStep(StepExecution stepExecution) {
        this.stepExecution = stepExecution;
        // Restore state from previous run (restart scenario)
        errorCount = stepExecution.getExecutionContext().getInt("errorCount", 0);
    }

    @Override
    public Order process(Order order) {
        try {
            return enrichOrder(order);
        } catch (Exception e) {
            errorCount++;
            // Save state — persisted on next chunk commit
            stepExecution.getExecutionContext().putInt("errorCount", errorCount);
            return null;  // filter
        }
    }
}

// Query context
// SELECT STEP_EXECUTION_ID, SHORT_CONTEXT
// FROM BATCH_STEP_EXECUTION_CONTEXT
// WHERE STEP_EXECUTION_ID = ?;
// → {"FlatFileItemReader.read.count":25000, "errorCount":42}
```

### ⚠️ Pitfalls / Gotchas
- Context is serialized to DB every chunk commit — don't store large objects! *(bada data mat dalo — har chunk mein DB mein jaata hai)*
- Store only positions, counters, small metadata — never entities or collections
- Large context slows every chunk commit
- Spring Batch 5 = JSON serialization, older = Java serialization (migration issue)

### ⚡ Remember
- Stores reader position → enables restart *(position save = restart possible)*
- Saved per chunk commit (automatically)
- JSON format in Spring Batch 5
- Store only small data (positions, counters)
- Custom data via `executionContext.put()`

### 🔗 Follow-ups
- [Q116 → BATCH_JOB_EXECUTION_CONTEXT](#q116)
- [Q66 → ExecutionContext concept](#q66)
- [Q68 → Resume from failure point](#q68)

---

## Q116. What is BATCH_JOB_EXECUTION_CONTEXT?

### 📝 One-Liner
Stores job-level shared state — data that needs to be shared between steps within the same job execution.

### 🔑 Quick Answer
BATCH_JOB_EXECUTION_CONTEXT stores **job-level shared state** as JSON. Unlike step context (private per step, for restart), job context is **shared across all steps** in the same execution. Use case: Step 1 counts total records, saves to job context → Step 3 reads it for a report. Write via `jobExecution.getExecutionContext().put()`, read via `@Value("#{jobExecutionContext['key']}")` in @StepScope beans. Only store small data — counters, flags, file paths. *(Job context = steps ke beech data share karne ke liye — step context = restart ke liye)*

### 📖 How It Works
```
Job Context vs Step Context:

Job Context (shared across steps):
  Step 1 writes: jobContext.put("totalRecords", 50000)
  Step 2 writes: jobContext.put("validRecords", 48500)
  Step 3 reads:  jobContext.get("totalRecords")  → 50000
                 jobContext.get("validRecords")   → 48500
  → Uses data from previous steps for report generation

Step Context (private per step, restart):
  Step 1 context: {"reader.position": 25000}  ← only Step 1 sees this
  Step 2 context: {"reader.position": 48500}  ← only Step 2 sees this
  → Each step's restart state is independent

Summary:
  Job Context  = SHARED across steps = inter-step communication
  Step Context = PRIVATE per step = restart state
```

### 🗣️ How to Say in Interview
"BATCH_JOB_EXECUTION_CONTEXT stores job-level shared state that persists across steps within the same execution. The key difference from step context: step context is private to each step and stores restart state, while job context is shared across all steps for inter-step communication. In my project, the first step counted total records and wrote it to job context, and the final report step read that count to include in the summary email. I write to it using jobExecution.getExecutionContext().put() in a StepExecutionListener, and read from it using @Value with the jobExecutionContext SpEL expression in @StepScope beans."

### 💻 Code
```java
// Step 1: Write to job context
@Component
public class CountingStepListener implements StepExecutionListener {
    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        long totalRecords = stepExecution.getReadCount();
        long validRecords = stepExecution.getWriteCount();
        
        // Write to JOB context (shared across steps)
        ExecutionContext jobContext = stepExecution.getJobExecution().getExecutionContext();
        jobContext.putLong("totalRecords", totalRecords);
        jobContext.putLong("validRecords", validRecords);
        
        return stepExecution.getExitStatus();
    }
}

// Step 3: Read from job context
@Component
@StepScope
public class ReportProcessor implements ItemProcessor<ReportRequest, Report> {
    @Value("#{jobExecutionContext['totalRecords']}")
    private Long totalRecords;

    @Value("#{jobExecutionContext['validRecords']}")
    private Long validRecords;

    @Override
    public Report process(ReportRequest request) {
        Report report = new Report();
        report.setTotalRecords(totalRecords);
        report.setValidRecords(validRecords);
        report.setSkippedRecords(totalRecords - validRecords);
        return report;
    }
}

// Query job context
// SELECT SHORT_CONTEXT FROM BATCH_JOB_EXECUTION_CONTEXT
// WHERE JOB_EXECUTION_ID = ?;
// → {"totalRecords": 50000, "validRecords": 48500}
```

### ⚠️ Pitfalls / Gotchas
- Job context write from step: use `stepExecution.getJobExecution().getExecutionContext()` *(job context access karne ke liye jobExecution se jaana padta hai)*
- Step context write: use `stepExecution.getExecutionContext()` — don't confuse the two
- @Value("#{jobExecutionContext[...]}") requires @StepScope
- Only store small data — job context is also serialized to DB per commit

### 🆚 vs. Comparison
| Aspect | Job Context | Step Context |
|--------|-----------|-------------|
| Scope | Shared across ALL steps | Private to ONE step |
| Purpose | Inter-step communication | Restart state |
| Who reads | Any step in the job | Only that step |
| Write via | jobExecution.getExecutionContext() | stepExecution.getExecutionContext() |
| Read via | #{jobExecutionContext['key']} | #{stepExecutionContext['key']} |

### ⚡ Remember
- **Job context = shared** across steps *(job context = steps ke beech share)*
- **Step context = private** per step (restart)
- Write: `stepExecution.getJobExecution().getExecutionContext().put()`
- Read: `@Value("#{jobExecutionContext['key']}")` + @StepScope
- Only store small data (counters, flags)

### 🔗 Follow-ups
- [Q115 → BATCH_STEP_EXECUTION_CONTEXT](#q115)
- [Q66 → ExecutionContext concept](#q66)
- [Q110 → All metadata tables](#q110)
