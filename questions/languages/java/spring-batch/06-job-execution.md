# 🟡 Spring Batch — Job Execution & Identity (Q55–Q62)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q55"></a>

## Q55. What is JobInstance?

### 🔑 Quick Answer

> A JobInstance is the **logical identity** of a job run — defined by **Job name + identifying JobParameters**. It represents "this specific batch run" and can have multiple JobExecutions (if it failed and was restarted).

### 📖 Step-by-Step Explanation

**Step 1 — Think of it as a unique "ticket":**

```
Job: "monthlyBilling"
Parameters: { month: "2024-01" }
→ Creates JobInstance: "monthlyBilling for January 2024"

This is ONE logical run. Whether it succeeds first time or takes 5 attempts,
it's still the SAME JobInstance.
```

**Step 2 — JobInstance vs JobExecution:**

```
JobInstance: "monthlyBilling" + month=2024-01 (THE IDENTITY)
│
├── JobExecution #1:  Started 2am, FAILED at Step 2
├── JobExecution #2:  Started 3am, FAILED at Step 2 again
└── JobExecution #3:  Started 4am, COMPLETED ✅

All 3 executions belong to the SAME JobInstance
because they have the SAME job name and parameters.
```

**Step 3 — Rule: Completed JobInstance cannot re-run:**

```
JobInstance (month=2024-01) → COMPLETED
Try to run again with month=2024-01 → ❌ JobInstanceAlreadyCompleteException

JobInstance (month=2024-02) → NEW JobInstance → ✅ Runs normally
```

### 🗣️ How to Explain in Interview

> *"A JobInstance is the logical identity of a job run. It's defined by the job name plus its identifying parameters. For example, 'monthlyBilling' with month=January is one JobInstance. If it fails, I restart with the same parameters, and Spring Batch creates a new JobExecution under the same JobInstance — it knows it's a retry, not a new run. Once a JobInstance completes successfully, you can't re-run it with the same parameters. This prevents accidental duplicate processing — like billing customers twice for the same month."*

### ⚡ Key Points to Remember

1. **JobInstance = Job name + identifying parameters**
2. One JobInstance → **multiple JobExecutions** (on failure/restart)
3. **Completed JobInstance cannot re-run** with same parameters
4. Stored in **BATCH_JOB_INSTANCE** table
5. Prevents **duplicate processing**

---

<a id="q56"></a>

## Q56. What is the difference between JobInstance and JobExecution?

### 🔑 Quick Answer

> **JobInstance** = the logical definition ("what" to run with "which" parameters). **JobExecution** = the physical attempt ("when" it ran and what happened). One JobInstance can have multiple JobExecutions.

### 📖 Step-by-Step Explanation

| Aspect | JobInstance | JobExecution |
|--------|-----------|-------------|
| **What it is** | Logical identity | Physical attempt |
| **How many** | One per unique params | Many per instance (on restart) |
| **Identified by** | Job name + parameters | Auto-generated ID |
| **Tracks** | Just identity | Status, times, exceptions |
| **Re-runnable** | Not if COMPLETED | New one created on restart |
| **Analogy** | An exam (Math Final 2024) | Each attempt to take that exam |

```
Think of it like an exam:

JobInstance: "Math Final Exam 2024"
├── JobExecution #1: Student took it on Dec 10 → FAILED (score 45%)
├── JobExecution #2: Student retook it on Dec 15 → FAILED (score 55%)
└── JobExecution #3: Student retook it on Dec 20 → COMPLETED (score 75%)

The exam is the same (JobInstance).
Each attempt is a different JobExecution.
Once passed (COMPLETED), can't retake.
```

### 🗣️ How to Explain in Interview

> *"JobInstance is the 'what' — it defines which job with which parameters. JobExecution is the 'when' — it records each attempt to run that instance. Think of it like an exam: the Math Final is the JobInstance, and each time the student takes it is a JobExecution. If the first attempt fails, a new JobExecution is created for the retry, but it's still the same JobInstance. Once the instance completes successfully, no more executions can be created — the job is done."*

### ⚡ Key Points to Remember

1. **Instance** = identity (what + parameters)
2. **Execution** = attempt (status, times, exceptions)
3. **1 Instance → N Executions** (on restart)
4. Instance COMPLETED → no new executions allowed
5. Instance FAILED → new execution created on restart

---

<a id="q57"></a>

## Q57. What is the difference between JobExecution and StepExecution?

### 🔑 Quick Answer

> **JobExecution** tracks the overall job status. **StepExecution** tracks each individual step with detailed metrics — read count, write count, skip count, commit count, rollback count.

### 📖 Step-by-Step Explanation

```
JobExecution: "monthlyBilling" run #3
├── status: COMPLETED
├── startTime: 2024-01-15 02:00
├── endTime: 2024-01-15 03:30
│
├── StepExecution: "readCustomers"
│   ├── status: COMPLETED
│   ├── readCount: 2,000,000
│   ├── writeCount: 1,950,000
│   ├── filterCount: 50,000
│   ├── commitCount: 4,000
│   └── duration: 45 minutes
│
├── StepExecution: "calculateBills"
│   ├── status: COMPLETED
│   ├── readCount: 1,950,000
│   ├── writeCount: 1,948,000
│   ├── skipCount: 2,000
│   ├── commitCount: 3,900
│   └── duration: 30 minutes
│
└── StepExecution: "sendNotifications"
    ├── status: COMPLETED
    └── duration: 15 minutes (Tasklet)
```

| Feature | JobExecution | StepExecution |
|---------|-------------|---------------|
| **Scope** | Entire job | One step |
| **Metrics** | Overall status, times | Detailed: read/write/skip/commit counts |
| **Multiple** | One per job attempt | One per step per attempt |
| **State** | Job-level ExecutionContext | Step-level ExecutionContext |

### 🗣️ How to Explain in Interview

> *"JobExecution gives the big picture — did the job complete or fail, when did it start and end. StepExecution gives the detail — for each step, how many records were read, written, filtered, and skipped, how many chunks committed successfully, how many rolled back. When debugging a failed job, I first check JobExecution for which step failed, then look at StepExecution for the specific counts and error messages. The detailed metrics in StepExecution are invaluable for production monitoring."*

### ⚡ Key Points to Remember

1. **JobExecution** = overall status (big picture)
2. **StepExecution** = per-step metrics (detail)
3. StepExecution has **readCount, writeCount, skipCount, commitCount**
4. Debug: check **JobExecution first** → then **StepExecution** for details
5. One JobExecution contains **multiple StepExecutions**

---

<a id="q58"></a>

## Q58. How does Spring Batch identify a unique JobInstance?

### 🔑 Quick Answer

> **Job name + identifying parameters** = unique JobInstance. Spring Batch computes a hash (JOB_KEY) from the identifying parameters. Same hash = same JobInstance.

### 📖 Step-by-Step Explanation

**Step 1 — The identity formula:**

```
JobInstance identity = JobName + hash(identifying parameters)

Example:
  Job: "monthlyBilling"
  Params: { month: "2024-01" (identifying), triggeredBy: "admin" (non-identifying) }
  
  JOB_KEY = hash("month=2024-01")  ← only identifying params used!
  
  "triggeredBy" is non-identifying → IGNORED for identity
```

**Step 2 — Identifying vs Non-identifying parameters:**

```java
JobParameters params = new JobParametersBuilder()
    .addString("month", "2024-01")                  // Identifying (default)
    .addString("triggeredBy", "admin", false)        // Non-identifying (3rd param = false)
    .addLong("timestamp", System.currentTimeMillis()) // Identifying (default)
    .toJobParameters();

Identity = "monthlyBilling" + hash(month=2024-01, timestamp=...)
"triggeredBy" is excluded from identity calculation
```

**Step 3 — Stored in BATCH_JOB_INSTANCE table:**

```
JOB_INSTANCE_ID | JOB_NAME        | JOB_KEY
1               | monthlyBilling  | a1b2c3d4e5  (hash of identifying params)
2               | monthlyBilling  | f6g7h8i9j0  (different month = different hash)
```

### 🗣️ How to Explain in Interview

> *"Spring Batch identifies a JobInstance by combining the job name with a hash of the identifying parameters. Only parameters marked as 'identifying' contribute to this hash. So if I have month as identifying and triggeredBy as non-identifying, two runs with the same month are the same JobInstance regardless of who triggered them. This is stored as JOB_KEY in the BATCH_JOB_INSTANCE table. This mechanism prevents duplicate processing — you can't accidentally run the same billing for the same month twice."*

### ⚡ Key Points to Remember

1. **Identity = Job name + hash(identifying params)**
2. **Non-identifying params** are ignored for identity
3. Default: all params are **identifying** (pass `false` to make non-identifying)
4. Stored as **JOB_KEY** hash in BATCH_JOB_INSTANCE table
5. Same identity + COMPLETED → **can't re-run**

---

<a id="q59"></a>

## Q59. What happens if a job is executed with the same parameters?

### 🔑 Quick Answer

> It depends on the previous status: **COMPLETED** → throws `JobInstanceAlreadyCompleteException`. **FAILED** → creates new JobExecution (restart). **STARTED** → throws `JobExecutionAlreadyRunningException`.

### 📖 Step-by-Step Explanation

**Step 1 — Three scenarios:**

```
Scenario 1: Previous run COMPLETED
  Run again with same params → ❌ JobInstanceAlreadyCompleteException
  "This job already finished successfully. Nothing to do."

Scenario 2: Previous run FAILED
  Run again with same params → ✅ Creates new JobExecution (restart!)
  "Previous run failed. Let me try again from where it left off."

Scenario 3: Previous run still STARTED/STARTING
  Run again with same params → ❌ JobExecutionAlreadyRunningException
  "This job is still running!"
```

**Step 2 — How to allow re-running completed jobs:**

```java
// Solution 1: Add a unique parameter (timestamp) each time
JobParameters params = new JobParametersBuilder()
        .addString("month", "2024-01")
        .addLong("run.id", System.currentTimeMillis())  // Makes each run unique
        .toJobParameters();

// Solution 2: Use RunIdIncrementer (auto-increments run.id)
@Bean
public Job myJob(JobRepository repo, Step step) {
    return new JobBuilder("myJob", repo)
            .incrementer(new RunIdIncrementer())  // Adds auto-incrementing run.id
            .start(step)
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"It depends on the previous status. If the job completed successfully, Spring Batch throws JobInstanceAlreadyCompleteException — it prevents accidental re-processing. If it failed, running with the same parameters creates a new JobExecution under the existing JobInstance — this is the restart mechanism. If it's still running, you get a JobExecutionAlreadyRunningException. To allow re-running a completed job, I either add a unique timestamp parameter or use RunIdIncrementer which automatically adds an incrementing ID. But I only do this when re-running is the intended behavior — the default protection against duplicates is usually what you want."*

### ⚡ Key Points to Remember

1. **COMPLETED** → Exception (can't re-run)
2. **FAILED** → Restart (new execution, same instance)
3. **STARTED** → Exception (already running)
4. **RunIdIncrementer** = allow re-running completed jobs
5. Default duplicate protection is usually **desirable**

---

<a id="q60"></a>

## Q60. How can you prevent duplicate job execution?

### 🔑 Quick Answer

> Spring Batch **prevents duplicates by default** — same job + same identifying parameters can't run again if completed. For extra protection: use a **JobParametersValidator**, check status before launching, or use a **distributed lock** in clustered environments.

### 📖 Step-by-Step Explanation

**Step 1 — Built-in protection (default):**

```
Run "monthlyBilling" with month=2024-01 → COMPLETED
Run "monthlyBilling" with month=2024-01 → ❌ REJECTED (already completed)

This is automatic. No extra code needed.
```

**Step 2 — Extra protection with validator:**

```java
@Bean
public Job billingJob(JobRepository repo, Step step) {
    return new JobBuilder("monthlyBilling", repo)
            .validator(new DefaultJobParametersValidator(
                new String[]{"month"},           // Required parameters
                new String[]{"triggeredBy"}      // Optional parameters
            ))
            .start(step)
            .build();
}
```

**Step 3 — Clustered environments (multiple servers):**

```
Server A: Launches "monthlyBilling" month=2024-01
Server B: Also launches "monthlyBilling" month=2024-01

Without lock: BOTH might start → duplicate processing!

Solution: Distributed lock
  Server A acquires lock → runs job
  Server B checks lock → waits or skips
```

### 🗣️ How to Explain in Interview

> *"Spring Batch has built-in duplicate prevention — once a job completes with specific parameters, it can't re-run with the same parameters. For additional safety, I use a JobParametersValidator to ensure required parameters like 'month' are always provided. In clustered environments where multiple instances might trigger the same job, I use a distributed lock — either through the database (SELECT FOR UPDATE on the job instance row) or using something like Redis/ZooKeeper locks. The database tables themselves act as a pessimistic lock since creating a JobInstance is an atomic database operation."*

### ⚡ Key Points to Remember

1. **Built-in**: completed job + same params = can't re-run
2. **Validator**: ensures required params are present
3. **Clustered**: use distributed locks for multi-server safety
4. **Database**: BATCH_JOB_INSTANCE creation is atomic (basic protection)
5. Don't disable this protection unless you really need to

---

<a id="q61"></a>

## Q61. How do you restart a failed job?

### 🔑 Quick Answer

> Simply re-run the job with the **same parameters**. Spring Batch automatically detects it's a restart — it skips completed Steps and resumes the failed Step from the last committed chunk.

### 📖 Step-by-Step Explanation

**Step 1 — What happens during restart:**

```
First run (FAILED):
  Step 1: readCustomers     → COMPLETED ✅ (all records processed)
  Step 2: calculateBills    → FAILED ❌ (crashed at chunk 50/100)
  Step 3: sendNotifications → NOT STARTED

Restart (same parameters):
  Step 1: readCustomers     → SKIPPED (already COMPLETED)
  Step 2: calculateBills    → RESUMED from chunk 51 (reads ExecutionContext)
  Step 3: sendNotifications → EXECUTES normally
  
Result: COMPLETED ✅
```

**Step 2 — How does it know where to resume?**

```
ExecutionContext saved after chunk 50:
  {
    "JdbcPagingItemReader.read.count": 25000,   // Reader was at row 25,000
    "JdbcPagingItemReader.start.after": {"id": 25000}
  }

On restart:
  Reader reads ExecutionContext: "I was at row 25,000"
  Starts query with: WHERE id > 25000
  Resumes from row 25,001
```

**Step 3 — Code — restart is the SAME code as first run:**

```java
// No special restart code needed!
// Just run with same parameters:
JobParameters params = new JobParametersBuilder()
        .addString("month", "2024-01")    // Same params as failed run
        .toJobParameters();

jobLauncher.run(monthlyBillingJob, params);

// Spring Batch automatically:
// 1. Finds existing JobInstance (month=2024-01)
// 2. Sees it FAILED
// 3. Creates new JobExecution
// 4. Skips COMPLETED steps
// 5. Resumes FAILED step from checkpoint
```

### 🗣️ How to Explain in Interview

> *"Restarting is beautifully simple — you just run the job again with the same parameters. Spring Batch does the rest. It finds the existing JobInstance, sees it failed, creates a new JobExecution, and then checks each step. Completed steps are skipped entirely. The failed step is resumed from the last committed chunk — because after each chunk commit, the reader's position is saved in the ExecutionContext in the database. So if the step failed at chunk 50, it reads the saved position and starts from chunk 51. No records are re-processed."*

### ⚡ Key Points to Remember

1. **Same parameters** → Spring Batch detects restart automatically
2. **Completed Steps** are skipped
3. **Failed Step** resumes from last committed chunk
4. **ExecutionContext** stores the reader's position
5. No special restart code needed — it's the same `jobLauncher.run()`

---

<a id="q62"></a>

## Q62. What happens when job restart occurs? (Internal flow)

### 🔑 Quick Answer

> Spring Batch finds the previous FAILED execution, creates a new execution, checks each step's status (skip if COMPLETED, resume if FAILED/STOPPED, execute if NEVER_STARTED), and reads ExecutionContext from DB to restore reader position.

### 📖 Step-by-Step Explanation

**Step 1 — Complete internal restart flow:**

```
jobLauncher.run(job, params)
│
├── 1. Find JobInstance for "billingJob" + params
│   └── Found: JobInstance #5
│
├── 2. Get last JobExecution for this instance
│   └── Found: JobExecution #12, status=FAILED
│
├── 3. Create new JobExecution #13, status=STARTED
│
├── 4. For each Step in the Job:
│   │
│   ├── Step "readCustomers":
│   │   └── Previous StepExecution: COMPLETED → SKIP ✅
│   │
│   ├── Step "calculateBills":
│   │   ├── Previous StepExecution: FAILED
│   │   ├── Load StepExecution's ExecutionContext from DB
│   │   │   └── { "reader.position": 25000 }
│   │   ├── Create new StepExecution, status=STARTED
│   │   ├── Initialize reader with saved position (starts at 25001)
│   │   └── EXECUTE from chunk 51 onwards → COMPLETED ✅
│   │
│   └── Step "sendNotifications":
│       ├── No previous StepExecution (never started)
│       ├── Create new StepExecution
│       └── EXECUTE normally → COMPLETED ✅
│
└── 5. All steps done → JobExecution #13 status=COMPLETED
```

**Step 2 — Key decision table for each step:**

| Previous Step Status | Restart Action |
|---------------------|----------------|
| COMPLETED | **Skip** — don't re-execute |
| FAILED | **Resume** from last checkpoint |
| STOPPED | **Resume** from last checkpoint |
| Never started | **Execute** from beginning |
| ABANDONED | **Skip** (manually marked, don't touch) |

**Step 3 — What if a Step should NOT be restartable?**

```java
@Bean
public Step nonRestartableStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("cleanup", repo)
            .tasklet(cleanupTasklet(), tx)
            .allowStartIfComplete(true)   // Re-execute even if completed (always run)
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"Internally, when a restart happens, Spring Batch loads the previous failed JobExecution and iterates through each step. For completed steps, it skips them entirely. For the failed step, it loads the ExecutionContext from the database — this contains the reader's saved position. It creates a new StepExecution and initializes the reader with that saved position. The reader then starts from where it left off. Steps that never started are executed normally. There's also an option to force re-execution of completed steps with allowStartIfComplete(true), which is useful for cleanup steps that should always run."*

### ⚡ Key Points to Remember

1. **COMPLETED steps** → skipped on restart
2. **FAILED steps** → resumed from checkpoint
3. **Never-started steps** → executed normally
4. **ExecutionContext** loaded from DB → restores reader position
5. `allowStartIfComplete(true)` → always re-execute (even if completed)

---

> **🎯 Navigation:** [← Processors (Q49-54)](05-processors.md) | [Next → Transactions & Restart (Q63-69)](07-transactions-restart.md) | [📋 All Sections](README.md)
