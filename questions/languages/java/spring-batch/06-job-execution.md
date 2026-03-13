<![CDATA[<div align="center">

# 🟡 Spring Batch — Job Execution Questions (55-62)

[![Questions](https://img.shields.io/badge/Questions-8-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)

</div>

---

<a id="q1"></a>
## Q55. ❓ What is JobInstance?

🔖 **Tags:** `#spring-batch` `#job-instance` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

A **JobInstance** = **Job Name + Job Parameters**. It represents a **logical run** of a job.

```
JobInstance = "dailyReport" + {date: "2026-03-13"}

This is ONE unique JobInstance.
It can have MULTIPLE JobExecutions (if it fails and restarts):

JobInstance("dailyReport", date=2026-03-13)
├── JobExecution #1: FAILED    (first attempt)
├── JobExecution #2: FAILED    (second attempt)
└── JobExecution #3: COMPLETED (third attempt — success!)
```

### 🎯 Key Rule:
> A **COMPLETED** JobInstance can **NEVER** run again with the same parameters. Spring Batch prevents it.

---

<a id="q2"></a>
## Q56. ❓ What is the difference between JobInstance and JobExecution?

🔖 **Tags:** `#spring-batch` `#comparison` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🟡 Medium  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

| Feature | JobInstance | JobExecution |
|---------|-----------|-------------|
| **What** | Logical definition of a job run | Physical attempt to run a JobInstance |
| **Identity** | JobName + JobParameters | Unique execution ID |
| **Count** | One per unique params | Multiple per JobInstance (on failure+restart) |
| **Stored in** | `BATCH_JOB_INSTANCE` | `BATCH_JOB_EXECUTION` |
| **Analogy** | 📋 A task on your to-do list | ✏️ Each attempt to complete that task |

```
Job: "monthlyBilling"

March 2026: JobInstance(monthlyBilling, month=March)
  ├── Execution #1 → FAILED (DB down)
  └── Execution #2 → COMPLETED ✅

April 2026: JobInstance(monthlyBilling, month=April)   ← Different instance!
  └── Execution #1 → COMPLETED ✅
```

---

<a id="q3"></a>
## Q57. ❓ What is the difference between JobExecution and StepExecution?

🔖 **Tags:** `#spring-batch` `#comparison`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Feature | JobExecution | StepExecution |
|---------|-------------|--------------|
| **Scope** | Entire job | Single step |
| **Contains** | Multiple StepExecutions | Read/write counts, skip counts |
| **Status** | Overall job status | Individual step status |
| **Context** | Job-level ExecutionContext | Step-level ExecutionContext |
| **Stored in** | `BATCH_JOB_EXECUTION` | `BATCH_STEP_EXECUTION` |

```
JobExecution (FAILED)
├── StepExecution: "readData" → COMPLETED ✅
├── StepExecution: "processData" → FAILED ❌ (job stopped here)
└── StepExecution: "writeReport" → NOT STARTED
```

---

<a id="q4"></a>
## Q58. ❓ How does Spring Batch identify a unique JobInstance?

🔖 **Tags:** `#spring-batch` `#job-instance` `#identity`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**JobInstance = Job Name + Identifying Job Parameters**

```java
// These create the SAME JobInstance (will fail on second run):
JobParameters params1 = new JobParametersBuilder()
    .addString("file", "data.csv")
    .toJobParameters();

// These create DIFFERENT JobInstances:
JobParameters params2 = new JobParametersBuilder()
    .addString("file", "data.csv")
    .addLong("runId", 1L)
    .toJobParameters();

JobParameters params3 = new JobParametersBuilder()
    .addString("file", "data.csv")
    .addLong("runId", 2L)          // Different value → different instance
    .toJobParameters();
```

### Non-Identifying Parameters:
```java
// Non-identifying params don't affect uniqueness
JobParameters params = new JobParametersBuilder()
    .addString("file", "data.csv", true)       // identifying = true (default)
    .addString("description", "test", false)    // identifying = false
    .toJobParameters();
```

---

<a id="q5"></a>
## Q59. ❓ What happens if a job is executed with the same parameters?

🔖 **Tags:** `#spring-batch` `#duplicate` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Previous Status | Same Params Again | Result |
|----------------|-------------------|--------|
| **COMPLETED** | Run again | ❌ `JobInstanceAlreadyCompleteException` |
| **FAILED** | Run again | ✅ Creates new JobExecution (restarts) |
| **STOPPED** | Run again | ✅ Creates new JobExecution (restarts) |
| **STARTED** (running) | Run again | ❌ `JobExecutionAlreadyRunningException` |

### Solution — Make Each Run Unique:
```java
JobParameters params = new JobParametersBuilder()
    .addString("file", "data.csv")
    .addLong("timestamp", System.currentTimeMillis())  // Unique every time
    .toJobParameters();
```

Or use `RunIdIncrementer`:
```java
@Bean
public Job job(JobRepository repo, Step step) {
    return new JobBuilder("myJob", repo)
            .incrementer(new RunIdIncrementer())  // Auto-increments run.id
            .start(step)
            .build();
}
```

---

<a id="q6"></a>
## Q60. ❓ How can you prevent duplicate job execution?

🔖 **Tags:** `#spring-batch` `#duplicate-prevention`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Method | How |
|--------|-----|
| **Default behavior** | Same params = same instance = won't re-run if COMPLETED |
| **Use identifying params** | Date, file name = natural uniqueness |
| **JobInstanceAlreadyCompleteException** | Framework throws this automatically |
| **Custom JobParametersValidator** | Validate before execution |
| **Distributed lock** | Use DB lock / Redis lock in clustered environments |

```java
// Custom validator to prevent duplicates
public class NoDuplicateValidator implements JobParametersValidator {
    @Override
    public void validate(JobParameters params) throws JobParametersInvalidException {
        if (params.getString("file") == null) {
            throw new JobParametersInvalidException("file parameter required!");
        }
    }
}
```

---

<a id="q7"></a>
## Q61. ❓ How do you restart a failed job?

🔖 **Tags:** `#spring-batch` `#restart` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Simply re-run the job with the **same parameters**:

```java
// First run → FAILED
JobExecution exec1 = jobLauncher.run(job, params);  // FAILED at step 3

// Restart → pass SAME params
JobExecution exec2 = jobLauncher.run(job, params);  // Resumes from step 3!
```

### What Happens on Restart:
```
First Run:
  Step 1: "readData"      → COMPLETED ✅
  Step 2: "processData"   → COMPLETED ✅
  Step 3: "writeReport"   → FAILED ❌ (at chunk 50 of 100)

Restart (same params):
  Step 1: "readData"      → SKIPPED (already complete)
  Step 2: "processData"   → SKIPPED (already complete)
  Step 3: "writeReport"   → RESUMES from chunk 50 ✅
```

### Requirements for Restart:
1. Job must be `restartable` (default = true)
2. Same JobParameters
3. Previous execution must be FAILED or STOPPED (not COMPLETED)

```java
// Disable restart
@Bean
public Job job(JobRepository repo, Step step) {
    return new JobBuilder("oneTimeJob", repo)
            .preventRestart()          // Cannot be restarted
            .start(step)
            .build();
}
```

---

<a id="q8"></a>
## Q62. ❓ What happens when a job restart occurs?

🔖 **Tags:** `#spring-batch` `#restart` `#internals`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
Restart Flow:
1. JobLauncher receives same params
2. Finds existing JobInstance (same name + params)
3. Finds last JobExecution → status = FAILED
4. Creates NEW JobExecution for same JobInstance
5. For each Step:
   ├── Check previous StepExecution status
   ├── If COMPLETED → SKIP this step
   ├── If FAILED → Resume from last committed chunk
   │   └── Uses ExecutionContext to find restart position
   └── If NOT STARTED → Execute from beginning
```

### 📌 Key Takeaway
> 💡 Spring Batch tracks state in metadata tables. On restart, it skips completed steps and resumes failed steps from the last committed chunk.

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Processors](./05-processors.md) | [Next: Transactions & Restart →](./07-transactions-restart.md)
]]>