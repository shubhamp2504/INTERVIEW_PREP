# 🟡 Spring Batch — Database & Metadata Tables (Q110–Q116)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q110"></a>

## Q110. What tables does Spring Batch create?

### 🔑 Quick Answer

> **6 metadata tables** + **3 sequence tables**. The core tables are: BATCH_JOB_INSTANCE (unique jobs), BATCH_JOB_EXECUTION (each run attempt), BATCH_STEP_EXECUTION (step-level stats), BATCH_JOB_EXECUTION_PARAMS (parameters), and two EXECUTION_CONTEXT tables (state for restart).

### 📖 Step-by-Step Explanation

**Step 1 — The complete schema:**

```
BATCH_JOB_INSTANCE ──────────── "What job was defined?"
  │                               One row per unique Job + Params combo
  │
  └──→ BATCH_JOB_EXECUTION ──── "Each attempt to run that job"
        │                         COMPLETED, FAILED, or STOPPED
        │
        ├──→ BATCH_JOB_EXECUTION_PARAMS ── "What parameters were passed?"
        │                                    file=/data/march.csv, date=2026-03-15
        │
        ├──→ BATCH_JOB_EXECUTION_CONTEXT ─ "Job-level shared state"
        │                                    Data shared between steps
        │
        └──→ BATCH_STEP_EXECUTION ──────── "Each step's detailed stats"
              │                              Read: 50000, Write: 49500, Skip: 500
              │
              └──→ BATCH_STEP_EXECUTION_CONTEXT ── "Step-level state for restart"
                                                     FlatFileReader.read.count=25000
```

**Step 2 — What each table stores:**

| Table | Purpose | Key Data |
|-------|---------|----------|
| BATCH_JOB_INSTANCE | Unique job identity | job_name + key (hash of params) |
| BATCH_JOB_EXECUTION | Each run attempt | status, start/end time, exit_code |
| BATCH_JOB_EXECUTION_PARAMS | Parameters per run | param name, type, value, identifying? |
| BATCH_JOB_EXECUTION_CONTEXT | Job-level state | shared data between steps (JSON) |
| BATCH_STEP_EXECUTION | Step-level stats | read/write/skip/commit/rollback counts |
| BATCH_STEP_EXECUTION_CONTEXT | Step-level state | reader position, progress (JSON) |
| BATCH_JOB_SEQ | Sequence | next job instance ID |
| BATCH_JOB_EXECUTION_SEQ | Sequence | next job execution ID |
| BATCH_STEP_EXECUTION_SEQ | Sequence | next step execution ID |

### 🗣️ How to Explain in Interview

> *"Spring Batch creates 6 metadata tables and 3 sequence tables. At the top level, BATCH_JOB_INSTANCE stores unique jobs — one row per unique job name plus identifying parameters. Below that, BATCH_JOB_EXECUTION stores each run attempt — if a job fails and restarts, that's two execution rows for one instance. Each execution has PARAMS for the input parameters, and CONTEXT for job-level shared state. Then BATCH_STEP_EXECUTION stores detailed per-step statistics — read count, write count, skip count, commit count, rollback count. And STEP_EXECUTION_CONTEXT stores the step's internal state like the reader position — this is what enables restart from the failure point."*

### ⚡ Key Points to Remember

1. **6 tables** + 3 sequences = 9 total
2. **JOB_INSTANCE** = unique identity (name + params)
3. **JOB_EXECUTION** = each attempt (1 instance → N executions)
4. **STEP_EXECUTION** = detailed counts (read/write/skip/commit)
5. **EXECUTION_CONTEXT** = state for restart (reader position)

---

<a id="q111"></a>

## Q111. What is BATCH_JOB_INSTANCE?

### 🔑 Quick Answer

> Stores **unique job identities**. Each row = one unique combination of job name + identifying parameters. If you run the same job with the same params, it's the **same instance** (not a new row).

### 📖 Step-by-Step Explanation

**Step 1 — How uniqueness works:**

```
Run 1: jobName="dailyReport", params={date:"2026-03-15"}
  → Creates INSTANCE ID=1, JOB_KEY=hash("date=2026-03-15")

Run 2: jobName="dailyReport", params={date:"2026-03-16"}
  → Creates INSTANCE ID=2, JOB_KEY=hash("date=2026-03-16")
  → Different params → NEW instance

Run 3: jobName="dailyReport", params={date:"2026-03-15"}
  → Same params as Run 1 → SAME INSTANCE ID=1
  → If Run 1 COMPLETED → error (can't re-run completed instance)
  → If Run 1 FAILED → creates new execution under same instance (restart)
```

**Step 2 — Table columns:**

```sql
SELECT * FROM BATCH_JOB_INSTANCE;

| JOB_INSTANCE_ID | VERSION | JOB_NAME       | JOB_KEY                          |
|-----------------|---------|----------------|----------------------------------|
| 1               | 0       | dailyReport    | d41d8cd98f00b204e9800998ecf8427e |
| 2               | 0       | dailyReport    | 5ab2c8d7e1f3a9b4c6d8e0f2a4b6c8d0 |
| 3               | 0       | monthlyBilling | a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6 |

JOB_KEY = MD5 hash of identifying parameters
Same JOB_NAME + same JOB_KEY = same instance (rejected if already COMPLETED)
```

### 🗣️ How to Explain in Interview

> *"BATCH_JOB_INSTANCE stores unique job identities. Uniqueness is determined by job name plus the hash of identifying parameters. If I run a daily report with date=2026-03-15, it creates instance 1. Running with date=2026-03-16 creates instance 2 — different params, new instance. But running again with date=2026-03-15 references the same instance 1. If that instance already completed, Spring Batch rejects it — you can't re-run a completed instance with the same params. This is why I always include a timestamp parameter when I need to re-run."*

### ⚡ Key Points to Remember

1. **Unique identity** = job name + hash of identifying params
2. **Same params** = same instance (not a new run)
3. **COMPLETED instance** can't be re-run with same params
4. **FAILED instance** can be restarted (new execution, same instance)
5. Add **timestamp param** if you need to re-run anytime

---

<a id="q112"></a>

## Q112. What is BATCH_JOB_EXECUTION?

### 🔑 Quick Answer

> Stores **each attempt to run** a job instance. One instance can have multiple executions (first try fails, second try succeeds = 2 execution rows).

### 📖 Step-by-Step Explanation

**Step 1 — One instance, multiple executions:**

```
JobInstance ID=1 (dailyReport, date=2026-03-15):

  Execution 1: STARTED → FAILED (disk full at 2:05 AM)
  Execution 2: STARTED → FAILED (code bug at 3:00 AM)
  Execution 3: STARTED → COMPLETED (fix deployed, ran at 4:00 AM) ✅

  = 3 rows in BATCH_JOB_EXECUTION for 1 row in BATCH_JOB_INSTANCE
```

**Step 2 — Key columns:**

```sql
SELECT JOB_EXECUTION_ID, JOB_INSTANCE_ID, STATUS, EXIT_CODE,
       EXIT_MESSAGE, START_TIME, END_TIME, CREATE_TIME
FROM BATCH_JOB_EXECUTION
WHERE JOB_INSTANCE_ID = 1
ORDER BY CREATE_TIME;

| ID | INSTANCE | STATUS    | EXIT_CODE | EXIT_MESSAGE         | START           | END             |
|----|----------|-----------|-----------|----------------------|-----------------|-----------------|
| 1  | 1        | FAILED    | FAILED    | Disk full            | 2026-03-15 02:00| 2026-03-15 02:05|
| 2  | 1        | FAILED    | FAILED    | NullPointerException | 2026-03-15 03:00| 2026-03-15 03:02|
| 3  | 1        | COMPLETED | COMPLETED | All steps completed  | 2026-03-15 04:00| 2026-03-15 04:10|
```

### 🗣️ How to Explain in Interview

> *"BATCH_JOB_EXECUTION stores each attempt to run a job instance. If a job fails and I restart it, that's a new execution under the same instance. So one instance can have multiple executions — first attempt failed, second attempt succeeded means 2 rows here. The important columns are STATUS (COMPLETED, FAILED, STOPPED), EXIT_MESSAGE which contains the exception message for failures, and START/END times for duration tracking. This is the first table I query when debugging — I look for FAILED status and read the EXIT_MESSAGE."*

### ⚡ Key Points to Remember

1. **1 instance → N executions** (failed attempts + final success)
2. **STATUS**: STARTING, STARTED, COMPLETED, FAILED, STOPPED, ABANDONED
3. **EXIT_MESSAGE** = exception details for failures
4. **First table to check** when debugging failed jobs
5. Only one execution can be STARTED at a time per instance

---

<a id="q113"></a>

## Q113. What is BATCH_STEP_EXECUTION?

### 🔑 Quick Answer

> Stores **detailed execution statistics** for each step: read count, write count, skip count, commit count, rollback count, filter count. This is the **most useful table for diagnosing issues**.

### 📖 Step-by-Step Explanation

**Step 1 — What the counts mean:**

```sql
SELECT STEP_NAME, STATUS, READ_COUNT, WRITE_COUNT, COMMIT_COUNT,
       ROLLBACK_COUNT, READ_SKIP_COUNT, WRITE_SKIP_COUNT, FILTER_COUNT
FROM BATCH_STEP_EXECUTION
WHERE JOB_EXECUTION_ID = 3;

| STEP       | STATUS    | READ  | WRITE | COMMITS | ROLLBACKS | R_SKIP | W_SKIP | FILTER |
|-----------|-----------|-------|-------|---------|-----------|--------|--------|--------|
| readData   | COMPLETED | 50000 | 50000 | 100     | 0         | 0      | 0      | 0      |
| processData| COMPLETED | 50000 | 48500 | 97      | 3         | 5      | 10     | 1485   |
| sendReport | COMPLETED | 1     | 1     | 1       | 0         | 0      | 0      | 0      |
```

**Step 2 — Reading the processData step:**

```
READ_COUNT = 50,000      → Reader returned 50,000 items
FILTER_COUNT = 1,485     → Processor returned null for 1,485 items (filtered)
WRITE_COUNT = 48,500     → Writer received 48,500 items
  (50,000 - 1,485 filtered - 15 skipped = 48,500)
READ_SKIP_COUNT = 5      → 5 items caused read errors, skipped
WRITE_SKIP_COUNT = 10    → 10 items caused write errors, skipped
COMMIT_COUNT = 97        → 97 chunks committed successfully
ROLLBACK_COUNT = 3       → 3 chunks rolled back (then chunk-level retry)
```

### 🗣️ How to Explain in Interview

> *"BATCH_STEP_EXECUTION is the most diagnostic table. It stores read count, write count, filter count, skip counts, commit count, and rollback count for each step. With these numbers I can reconstruct exactly what happened: if read is 50,000 and write is 48,500 with 1,485 filtered and 15 skipped, I know the processor intentionally filtered 1,485 records by returning null, and 15 records had errors that were skipped. The rollback count tells me how many chunks had failures requiring a rollback. And the commit count tells me processing progress."*

### ⚡ Key Points to Remember

1. **read - filter - skip = write** (the math must add up)
2. **FILTER_COUNT** = processor returned null (intentional)
3. **SKIP_COUNT** = errors that were tolerated
4. **ROLLBACK_COUNT** = chunks that failed (should be low)
5. **Most useful table** for debugging production issues

---

<a id="q114"></a>

## Q114. What is BATCH_JOB_EXECUTION_PARAMS?

### 🔑 Quick Answer

> Stores the **parameters passed to each job execution**: name, type, value, and whether the parameter is **identifying** (used to determine job instance uniqueness).

### 📖 Step-by-Step Explanation

**Step 1 — Identifying vs non-identifying params:**

```
IDENTIFYING (Y):
  → Used to compute JOB_KEY hash in BATCH_JOB_INSTANCE
  → Different value = different JobInstance
  → Example: date, fileName, customerId

NON-IDENTIFYING (N):
  → Metadata only, NOT used for uniqueness
  → Same value or different value = same instance
  → Example: description, triggeredBy, logLevel
```

```sql
SELECT JOB_EXECUTION_ID, PARAMETER_NAME, PARAMETER_TYPE,
       PARAMETER_VALUE, IDENTIFYING
FROM BATCH_JOB_EXECUTION_PARAMS
WHERE JOB_EXECUTION_ID = 1;

| EXEC_ID | NAME        | TYPE             | VALUE              | IDENTIFYING |
|---------|-------------|------------------|--------------------|-------------|
| 1       | file        | java.lang.String | /data/march.csv    | Y           |
| 1       | timestamp   | java.lang.Long   | 1710288000000      | Y           |
| 1       | description | java.lang.String | Monthly processing | N           |
```

### 🗣️ How to Explain in Interview

> *"BATCH_JOB_EXECUTION_PARAMS stores the parameters for each execution. Each parameter has a name, type, value, and an identifying flag. Identifying parameters determine job uniqueness — if I run with file=/data/march.csv, that's a different instance than file=/data/april.csv. Non-identifying parameters like description are metadata — they don't affect uniqueness. This is why I always add a timestamp as an identifying parameter when I need to allow re-runs with otherwise identical parameters."*

### ⚡ Key Points to Remember

1. **IDENTIFYING=Y** → affects instance uniqueness (JOB_KEY hash)
2. **IDENTIFYING=N** → metadata only
3. Types: String, Long, Double, Date
4. **timestamp** is common identifying param for re-runs
5. All params viewable via `JobExplorer.getJobParameters()`

---

<a id="q115"></a>

## Q115. What is BATCH_STEP_EXECUTION_CONTEXT?

### 🔑 Quick Answer

> Stores the **serialized state of each step** as JSON. This is **how restart works** — the reader's position (e.g., line 25000) is saved here, and on restart the reader skips to that position.

### 📖 Step-by-Step Explanation

**Step 1 — What's stored:**

```sql
SELECT STEP_EXECUTION_ID, SHORT_CONTEXT
FROM BATCH_STEP_EXECUTION_CONTEXT;

| STEP_ID | SHORT_CONTEXT                                                       |
|---------|---------------------------------------------------------------------|
| 1       | {"FlatFileItemReader.read.count":25000}                              |
| 2       | {"JdbcPagingItemReader.read.count":45000, "currentPage":90}         |
| 3       | {"batch.taskletType":"org.springframework.batch.core.step.tasklet"} |
```

**Step 2 — How restart uses this:**

```
First Run (fails at record 25,000):
  Reader processes: 1, 2, 3, ... 24,999, 25,000 → ERROR!
  ExecutionContext saved: { "read.count": 25000 }
  Step status: FAILED

Restart:
  Reader opens → reads ExecutionContext: { "read.count": 25000 }
  Reader skips records 1-25,000
  Reader resumes from record 25,001
  Processing continues from where it left off ✅

Without this table → restart would re-process ALL records from the beginning
```

### 🗣️ How to Explain in Interview

> *"BATCH_STEP_EXECUTION_CONTEXT stores each step's internal state as serialized JSON. The most important use is enabling restart. When a FlatFileItemReader processes 25,000 records and then fails, its position — read.count=25000 — is saved in this table. On restart, the reader reads this context, skips the first 25,000 records, and resumes from 25,001. Without this table, every restart would start from the beginning. You can also store custom state here using the ExecutionContext API — like counters or intermediate results that need to survive a restart."*

### ⚡ Key Points to Remember

1. **Stores step state** as JSON (reader position, custom data)
2. **Enables restart** — reader knows where to resume
3. Saved **per chunk commit** (not just at end)
4. Custom data via `executionContext.put("key", value)`
5. Don't store large data here — only positions and counters

---

<a id="q116"></a>

## Q116. What is BATCH_JOB_EXECUTION_CONTEXT?

### 🔑 Quick Answer

> Stores **job-level shared state** — data that needs to be shared **between steps** within the same job execution. Different from step-level context which is per-step.

### 📖 Step-by-Step Explanation

**Step 1 — Step context vs Job context:**

```
STEP EXECUTION CONTEXT (per-step, private):
  Step 1 context: { "read.count": 25000 }    ← Only Step 1 sees this
  Step 2 context: { "read.count": 50000 }    ← Only Step 2 sees this

JOB EXECUTION CONTEXT (shared across all steps):
  Job context: { "totalRecords": 100000, "outputFile": "/reports/march.pdf" }
  ← ALL steps in this job can read/write this
```

**Step 2 — Common use case (passing data between steps):**

```
Step 1 (count records) → Job Context: { totalRecords: 50,000 }
Step 2 (process data)  → reads totalRecords from Job Context for progress %
Step 3 (generate report)→ reads totalRecords for report header
```

### 💻 Code Example

```java
// Step 1: Save data to Job ExecutionContext
@Component
public class CountingStepListener implements StepExecutionListener {
    
    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        // Write to JOB context (accessible by other steps)
        stepExecution.getJobExecution().getExecutionContext()
                .putInt("totalRecords", stepExecution.getReadCount());
        
        stepExecution.getJobExecution().getExecutionContext()
                .putString("processedDate", LocalDate.now().toString());
        
        return ExitStatus.COMPLETED;
    }
}

// Step 3: Read data from Job ExecutionContext
@Component
@StepScope
public class ReportProcessor implements ItemProcessor<Summary, Report> {
    
    @Value("#{jobExecutionContext['totalRecords']}")
    private int totalRecords;
    
    @Value("#{jobExecutionContext['processedDate']}")
    private String processedDate;
    
    @Override
    public Report process(Summary summary) {
        Report report = new Report();
        report.setTotalRecords(totalRecords);     // From Step 1
        report.setDate(processedDate);             // From Step 1
        report.setSummary(summary);
        return report;
    }
}
```

### 🗣️ How to Explain in Interview

> *"BATCH_JOB_EXECUTION_CONTEXT stores job-level shared state — data that needs to be passed between steps. For example, Step 1 counts records and saves totalRecords=50000 to the job ExecutionContext. Step 3 reads that value to generate a report header. This is different from step ExecutionContext which is step-private — used for restart state like reader position. Job ExecutionContext is shared across all steps in the same job. I access it through stepExecution.getJobExecution().getExecutionContext() in listeners, or through @Value with jobExecutionContext SpEL expression in @StepScope beans."*

### ⚡ Key Points to Remember

1. **Job context** = shared across all steps
2. **Step context** = private to one step (restart state)
3. Write: `jobExecution.getExecutionContext().put(key, value)`
4. Read: `@Value("#{jobExecutionContext['key']}")` in @StepScope beans
5. Store only **small data** (counts, names, paths) — not large objects

---

> **🎯 Navigation:** [← Monitoring (Q104-109)](13-monitoring.md) | [Next → Production Scenarios (Q117-125)](15-production-scenarios.md) | [📋 All Sections](README.md)
