# 🏃 Job Execution & Instance — Q55 to Q62

---

## Q55. What is JobInstance?

### 📝 One-Liner
JobInstance is the logical identity of a job run — defined by the combination of job name + identifying job parameters.

### 🔑 Quick Answer
A `JobInstance` represents one logical run of a job. It's identified by `job name + identifying JobParameters`. For example, "dailyPaymentJob" with parameter `date=2024-01-15` creates one unique JobInstance. If it fails and you restart with the same parameters, Spring Batch finds the SAME JobInstance and creates a new `JobExecution` under it. A completed JobInstance cannot be re-run with the same parameters. *(JobInstance = job ka naam + parameters — same combination ek hi JobInstance hai)*

### 📖 How It Works
```
JobInstance Identity:

Job Name: "dailyPaymentJob"
Parameters: date=2024-01-15

JobInstance #1 (dailyPaymentJob + date=2024-01-15)
├── JobExecution #1 → FAILED  (first attempt)
├── JobExecution #2 → FAILED  (retry)
└── JobExecution #3 → COMPLETED (success!)

JobInstance #2 (dailyPaymentJob + date=2024-01-16) ← different date = different instance
└── JobExecution #4 → COMPLETED

Stored in: BATCH_JOB_INSTANCE table
  JOB_INSTANCE_ID | JOB_NAME           | JOB_KEY (hash of params)
  1               | dailyPaymentJob    | abc123...
  2               | dailyPaymentJob    | def456...
```

### 🗣️ Answering Approach
"A JobInstance is the logical identity of a job run, defined by the job name plus identifying job parameters. Multiple JobExecutions can belong to the same JobInstance — this happens when a failed job is restarted with the same parameters. A completed JobInstance cannot be re-run with the same parameters, which prevents duplicate processing. In my project, we used the processing date as a job parameter, so each day automatically created a new JobInstance. If the job failed at night, the morning restart found the same JobInstance and resumed from where it left off."

### 💻 Code
```java
// Each unique parameter combination = new JobInstance
@Bean
public Job dailyJob(JobRepository repo, Step step) {
    return new JobBuilder("dailyPaymentJob", repo)
            .start(step)
            .build();
}

// Launch with parameters — creates JobInstance
JobParameters params = new JobParametersBuilder()
        .addLocalDate("date", LocalDate.of(2024, 1, 15))  // identifying param
        .addString("env", "prod", false)  // false = non-identifying (ignored for identity)
        .toJobParameters();

jobLauncher.run(dailyJob, params);
// → Creates JobInstance(dailyPaymentJob, date=2024-01-15)
// → Ignores "env" param for identity (non-identifying)
```

### ⚠️ Pitfalls / Gotchas
- Re-running completed job with same params → `JobInstanceAlreadyCompleteException` *(completed job same params se dobara nahi chal sakta)*
- Non-identifying parameters (created with `false` flag) are NOT used for JobInstance identity
- JOB_KEY in DB is a hash of identifying parameters — not the params themselves
- Deleting BATCH_JOB_INSTANCE breaks restart tracking

### 🎯 Tricky Interview Qs

**Q: Can two different jobs have the same JobInstance?**
No. JobInstance identity includes the job name. "jobA" with date=2024-01-15 and "jobB" with date=2024-01-15 are different JobInstances.

**Q: What if you WANT to re-run a completed job with the same date?**
Use `RunIdIncrementer` — it adds an auto-incrementing `run.id` parameter, making each run a unique JobInstance even with the same business parameters.

### ⚡ Remember
- JobInstance = job name + identifying parameters *(naam + parameters = identity)*
- 1 JobInstance → N JobExecutions (on failure/restart)
- Completed instance → can't re-run with same params
- RunIdIncrementer bypasses this restriction
- Stored in BATCH_JOB_INSTANCE table

### 🔗 Follow-ups
- [Q56 → JobInstance vs JobExecution](#q56)
- [Q58 → How unique identity is computed](#q58)
- [Q59 → Same parameters behavior](#q59)

---

## Q56. What is the difference between JobInstance and JobExecution?

### 📝 One-Liner
JobInstance is the WHAT (logical run identity), JobExecution is the WHEN (one physical execution attempt with status, timestamps, and metrics).

### 🔑 Quick Answer
**JobInstance** = the logical "what" — identifies a job run by name + parameters. **JobExecution** = the physical "when" — one actual execution attempt with start time, end time, status (STARTED/COMPLETED/FAILED), and exit code. One JobInstance can have multiple JobExecutions (when restarted after failure). Think of it like an exam: the exam itself is the instance, each attempt is an execution. *(JobInstance = kya run karna hai, JobExecution = kab run hua aur result kya hua)*

### 📖 How It Works
```
Analogy — University Exam:
  Exam (JobInstance): "Math Final 2024"
  ├── Attempt 1 (JobExecution): Score 35 → FAILED
  ├── Attempt 2 (JobExecution): Score 40 → FAILED  
  └── Attempt 3 (JobExecution): Score 85 → COMPLETED ✅

JobInstance:                    JobExecution:
┌────────────────────┐         ┌──────────────────────────┐
│ JOB_INSTANCE_ID: 1 │         │ JOB_EXECUTION_ID: 1      │
│ JOB_NAME           │         │ JOB_INSTANCE_ID: 1       │
│ JOB_KEY (params)   │         │ STATUS: FAILED           │
│                    │←─┐      │ START_TIME, END_TIME     │
│ Identity only      │  │      │ EXIT_CODE, EXIT_MESSAGE  │
└────────────────────┘  │      │ CREATE_TIME              │
                        │      └──────────────────────────┘
                        │      ┌──────────────────────────┐
                        └──────│ JOB_EXECUTION_ID: 2      │
                               │ JOB_INSTANCE_ID: 1       │
                               │ STATUS: COMPLETED        │
                               └──────────────────────────┘
```

### 🗣️ Answering Approach
"JobInstance identifies what job was run — it's the logical identity based on job name and parameters. JobExecution represents one physical attempt at running that job, with actual start time, end time, and status. One JobInstance can have multiple JobExecutions — this happens when a failed job is restarted. In my project, I used this understanding to build a monitoring dashboard — showing the number of execution attempts per instance and the trend of failures before eventual success."

### 💻 Code
```java
// Accessing both in code
@Bean
public JobExecutionListener executionListener() {
    return new JobExecutionListener() {
        @Override
        public void afterJob(JobExecution execution) {
            // JobExecution properties
            log.info("Execution ID: {}", execution.getId());
            log.info("Status: {}", execution.getStatus());         // COMPLETED/FAILED
            log.info("Start: {}", execution.getStartTime());
            log.info("End: {}", execution.getEndTime());
            
            // JobInstance properties (accessed via execution)
            JobInstance instance = execution.getJobInstance();
            log.info("Instance ID: {}", instance.getInstanceId());
            log.info("Job Name: {}", instance.getJobName());
            
            // Parameters
            log.info("Params: {}", execution.getJobParameters());
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- Don't confuse them — interviewers specifically test this distinction *(interview mein zaroor puchte hain)*
- JobExecution has status/time; JobInstance has only identity
- 1 Instance → N Executions; N Instances → 1 Job (definition)
- Querying BATCH_JOB_EXECUTION without joining BATCH_JOB_INSTANCE gives incomplete picture

### 🆚 vs. Comparison
| Aspect | JobInstance | JobExecution |
|--------|------------|-------------|
| Represents | Logical identity | Physical attempt |
| Properties | name, key (params hash) | status, start/end time, exit code |
| Cardinality | 1 per unique params | N per instance (on restarts) |
| Table | BATCH_JOB_INSTANCE | BATCH_JOB_EXECUTION |
| Analogy | The exam | One attempt at the exam |

### ⚡ Remember
- Instance = WHAT (identity), Execution = WHEN (attempt + result)
- 1 Instance → N Executions *(ek instance ke kai attempts ho sakte hain)*
- Instance: no status; Execution: has status, times, metrics
- Exam analogy: instance = exam, execution = attempt

### 🔗 Follow-ups
- [Q55 → JobInstance details](#q55)
- [Q57 → JobExecution vs StepExecution](#q57)
- [Q61 → Restarting creates new execution](#q61)

---

## Q57. What is the difference between JobExecution and StepExecution?

### 📝 One-Liner
JobExecution tracks overall job status; StepExecution tracks each individual step with detailed metrics like readCount, writeCount, skipCount.

### 🔑 Quick Answer
**JobExecution** holds the job-level status, start/end time, and parameters. **StepExecution** holds step-level details including all the processing metrics: readCount, writeCount, filterCount, skipCount, commitCount, rollbackCount. One JobExecution contains multiple StepExecutions. The job's final status depends on all step statuses — if any step fails, the job fails. *(JobExecution = overall summary, StepExecution = har step ka detailed scorecard)*

### 📖 How It Works
```
JobExecution ("dailyBatch", COMPLETED)
├── StepExecution ("readStep")
│   ├── status: COMPLETED
│   ├── readCount: 10000
│   ├── writeCount: 9850
│   ├── filterCount: 100
│   ├── skipCount: 50
│   ├── commitCount: 20
│   └── rollbackCount: 2
├── StepExecution ("processStep")
│   ├── status: COMPLETED
│   ├── readCount: 9850
│   ├── writeCount: 9800
│   └── ...
└── StepExecution ("cleanupStep")
    └── status: COMPLETED

StepExecution Metrics:
| Metric | Meaning |
|--------|---------|
| readCount | Total items read |
| writeCount | Total items written |
| filterCount | Items filtered (processor returned null) |
| readSkipCount | Items skipped during read |
| processSkipCount | Items skipped during process |
| writeSkipCount | Items skipped during write |
| commitCount | Number of chunks committed |
| rollbackCount | Number of chunks rolled back |
```

### 🗣️ Answering Approach
"JobExecution gives the overall job status, while StepExecution provides granular step-level metrics. Each step tracks readCount, writeCount, filterCount, skipCount, commitCount, and rollbackCount. In my project, I built a monitoring system that queried StepExecution metrics after each job run — comparing readCount minus writeCount minus filterCount to detect data anomalies. If the skip rate exceeded 5%, we automatically triggered an alert to the operations team."

### 💻 Code
```java
@Bean
public StepExecutionListener metricsListener() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution se) {
            log.info("Step: {} | Status: {} | Read: {} | Written: {} | " +
                     "Filtered: {} | Skipped: {} | Commits: {} | Rollbacks: {}",
                se.getStepName(), se.getStatus(),
                se.getReadCount(), se.getWriteCount(),
                se.getFilterCount(), se.getSkipCount(),
                se.getCommitCount(), se.getRollbackCount());
            
            // Verify data integrity
            long expected = se.getReadCount() - se.getFilterCount() - se.getSkipCount();
            if (expected != se.getWriteCount()) {
                log.warn("DATA MISMATCH! Expected write: {}, Actual: {}", 
                    expected, se.getWriteCount());
            }
            return se.getExitStatus();
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- skipCount = readSkipCount + processSkipCount + writeSkipCount (it's a total) *(skipCount total hai — read + process + write ka sum)*
- commitCount includes the final commit (even for partial last chunk)
- rollbackCount counts chunk rollbacks, not individual item failures
- Job FAILED doesn't tell you WHICH step failed — check StepExecution

### ⚡ Remember
- JobExecution = big picture (overall status)
- StepExecution = detailed metrics per step *(har step ka detailed report StepExecution mein)*
- 1 JobExecution → N StepExecutions
- Key metrics: readCount, writeCount, filterCount, skipCount, commitCount
- `readCount - filterCount - skipCount = writeCount`

### 🔗 Follow-ups
- [Q56 → JobInstance vs JobExecution](#q56)
- [Q22 → filterCount from chunk processing](#q22)
- [Q104 → Monitoring step metrics](#q104)

---

## Q58. How does Spring Batch identify a unique JobInstance?

### 📝 One-Liner
Spring Batch computes a JOB_KEY hash from the job name and only the IDENTIFYING job parameters — non-identifying parameters are excluded.

### 🔑 Quick Answer
When you launch a job, Spring Batch takes the job name and all **identifying** parameters (those without the `false` flag), computes a hash (JOB_KEY), and checks BATCH_JOB_INSTANCE for an existing match. If found, it's the SAME JobInstance (restart). If not found, it creates a NEW JobInstance. Non-identifying parameters (created with the `false` flag) are stored but NOT used in the hash. *(Job name + sirf identifying parameters ka hash banata hai — non-identifying ignore hota hai)*

### 📖 How It Works
```
Identity Computation:

JobParameters:
  date = 2024-01-15     (identifying = true, default)
  env = "prod"           (identifying = false)
  requestId = "abc123"   (identifying = false)

JOB_KEY = hash(jobName + "date=2024-01-15")
                          ↑ only identifying params
                          env and requestId EXCLUDED from hash

BATCH_JOB_INSTANCE:
  JOB_INSTANCE_ID | JOB_NAME         | JOB_KEY
  1               | dailyPaymentJob  | <hash of "date=2024-01-15">
```

### 🗣️ Answering Approach
"Spring Batch identifies a unique JobInstance by computing a hash of the job name plus only the identifying job parameters. When you create a parameter with the boolean flag set to false, it becomes non-identifying — stored in the database but excluded from the identity hash. In my project, we used the business date as the identifying parameter and environment, requestId, and runId as non-identifying parameters. This way, the same business date always mapped to the same JobInstance for restart purposes, while non-identifying params provided context for logging and auditing."

### 💻 Code
```java
// Building parameters with identifying/non-identifying
JobParameters params = new JobParametersBuilder()
        .addLocalDate("date", LocalDate.of(2024, 1, 15))    // identifying (default)
        .addString("env", "prod", false)                      // NON-identifying
        .addString("requestId", UUID.randomUUID().toString(), false) // NON-identifying
        .toJobParameters();

// Both runs create SAME JobInstance (only "date" matters for identity):
// Run 1: date=2024-01-15, env=prod, requestId=abc
// Run 2: date=2024-01-15, env=staging, requestId=xyz
// → Same JOB_KEY because only date=2024-01-15 is used in hash
```

### ⚠️ Pitfalls / Gotchas
- Default is identifying (true) — pass `false` explicitly for non-identifying *(default mein sab parameters identifying hain)*
- Changing identifying params between runs = new JobInstance (not restart)
- JOB_KEY collision (different params, same hash) is theoretically possible but extremely rare
- Spring Batch 5 uses `addLocalDate`, `addLong`, etc. (Batch 4 used `addDate`, `addLong`)

### ⚡ Remember
- JOB_KEY = hash(job name + identifying params only)
- Non-identifying params: created with `false` flag *(false = non-identifying, hash mein nahi jaata)*
- Same hash = same JobInstance = restart scenario
- Different hash = new JobInstance = fresh run
- Default: all params are identifying

### 🔗 Follow-ups
- [Q55 → JobInstance concept](#q55)
- [Q59 → Same parameters behavior](#q59)
- [Q60 → Preventing duplicates](#q60)

---

## Q59. What happens if a job is executed with the same parameters?

### 📝 One-Liner
It depends on the last execution's status: COMPLETED → exception, FAILED → restart, STARTED → already-running exception.

### 🔑 Quick Answer
Spring Batch finds the existing JobInstance (same params) and checks the last JobExecution status: **COMPLETED** → throws `JobInstanceAlreadyCompleteException` (can't re-run). **FAILED** → creates a new JobExecution (restart — resumes from failure point). **STARTED/STARTING** → throws `JobExecutionAlreadyRunningException`. To force re-run a completed job, use `RunIdIncrementer` which adds an auto-incrementing `run.id` parameter. *(COMPLETED = dobara nahi chalega, FAILED = restart hoga, STARTED = already running error)*

### 📖 How It Works
```
Same Parameters Decision Tree:

Job launched with same params as existing JobInstance
  ↓
Last JobExecution status?
├── COMPLETED → JobInstanceAlreadyCompleteException ❌
│              "This job has already completed"
├── FAILED    → New JobExecution created (RESTART) ✅
│              Resumes from failure point
├── STOPPED   → New JobExecution created (RESTART) ✅
├── STARTED   → JobExecutionAlreadyRunningException ❌
│              "This job is currently running"
└── ABANDONED → New JobExecution created ✅

To bypass COMPLETED restriction:
  Use RunIdIncrementer → adds run.id=1, run.id=2, etc.
  → Each run becomes a NEW JobInstance
```

### 🗣️ Answering Approach
"When you launch a job with the same parameters, Spring Batch looks up the existing JobInstance and checks the last execution status. If it's completed, it throws JobInstanceAlreadyCompleteException — Spring Batch prevents duplicate processing by design. If it's failed, a new JobExecution is created and the job restarts from where it left off. In my project, we used RunIdIncrementer for jobs that needed to run multiple times per day with the same business date — each run got a unique run.id making it a new JobInstance."

### 💻 Code
```java
// RunIdIncrementer — allows re-running completed jobs
@Bean
public Job rerunableJob(JobRepository repo, Step step) {
    return new JobBuilder("dailyReport", repo)
            .incrementer(new RunIdIncrementer())  // adds run.id auto-increment
            .start(step)
            .build();
}

// First run:  params = {date: 2024-01-15, run.id: 1} → JobInstance #1
// Second run: params = {date: 2024-01-15, run.id: 2} → JobInstance #2 (new!)
// Third run:  params = {date: 2024-01-15, run.id: 3} → JobInstance #3 (new!)

// Without RunIdIncrementer — restart failed job
try {
    jobLauncher.run(job, sameParams);
} catch (JobInstanceAlreadyCompleteException e) {
    log.info("Job already completed for these params");
} catch (JobExecutionAlreadyRunningException e) {
    log.warn("Job is currently running with these params");
}
```

### ⚠️ Pitfalls / Gotchas
- `JobInstanceAlreadyCompleteException` is a FEATURE, not a bug — prevents duplicate processing *(ye protection hai, error nahi)*
- RunIdIncrementer makes every run unique — you lose restart capability for that run.id
- Don't catch and ignore these exceptions silently — log and handle appropriately
- ABANDONED status requires manual intervention (set via API)

### 🎯 Tricky Interview Qs

**Q: How do you re-run a completed job without RunIdIncrementer?**
Change an identifying parameter value (e.g., add a timestamp), or delete the JobInstance from BATCH_JOB_INSTANCE (not recommended in production).

**Q: What's the difference between STOPPED and FAILED?**
Both allow restart. STOPPED is intentional (operator stopped it via `JobOperator.stop()`). FAILED is due to an error. Restart behavior is the same.

### ⚡ Remember
- COMPLETED + same params = exception (protection) *(complete hua toh dobara nahi chalega)*
- FAILED + same params = restart (new execution)
- STARTED + same params = already running exception
- RunIdIncrementer = bypass duplicate protection
- FAILED restart resumes from failure point

### 🔗 Follow-ups
- [Q58 → How identity is computed](#q58)
- [Q60 → Preventing duplicate execution](#q60)
- [Q61 → Restart mechanism](#q61)

---

## Q60. How can you prevent duplicate job execution?

### 📝 One-Liner
Spring Batch has built-in duplicate prevention — completed jobs cannot re-run with same parameters; plus you can add validators and distributed locks.

### 🔑 Quick Answer
Four layers of protection: **(1) Built-in** — Spring Batch won't re-run a COMPLETED job with the same identifying parameters. **(2) JobParametersValidator** — validates required params are present and correct. **(3) Distributed locking** — in clustered environments, use database locks or Redis to prevent simultaneous runs. **(4) Database atomicity** — BATCH_JOB_INSTANCE's unique constraint on JOB_KEY prevents duplicate instances. *(Spring Batch mein built-in protection hai — completed job same params se nahi chalega)*

### 📖 How It Works
```
Duplicate Prevention Layers:

Layer 1: Built-in (framework level)
  └── COMPLETED + same params → JobInstanceAlreadyCompleteException

Layer 2: Parameter Validation
  └── JobParametersValidator checks required params before job starts

Layer 3: Distributed Lock (cluster level)
  └── Only one node can run the same job at a time
  └── Database lock or Redis lock

Layer 4: Database Constraint
  └── BATCH_JOB_INSTANCE(JOB_NAME, JOB_KEY) has unique constraint
```

### 🗣️ Answering Approach
"Spring Batch provides built-in duplicate prevention — a completed job cannot be re-run with the same identifying parameters. On top of that, I use a JobParametersValidator to ensure required parameters are present before the job starts. In my project's clustered environment with 3 nodes, we added a database-based distributed lock — before launching, the scheduler acquires a lock in a dedicated table, ensuring only one node runs the job. This prevents race conditions where two scheduler nodes might try to launch the same job simultaneously."

### 💻 Code
```java
// Built-in: validator checks params
@Bean
public Job validatedJob(JobRepository repo, Step step) {
    return new JobBuilder("validatedJob", repo)
            .validator(new DefaultJobParametersValidator(
                new String[]{"date"},        // required identifying params
                new String[]{"env", "debug"} // optional params
            ))
            .start(step)
            .build();
}

// Custom validator
public class BusinessDateValidator implements JobParametersValidator {
    @Override
    public void validate(JobParameters params) throws JobParametersInvalidException {
        LocalDate date = params.getLocalDate("date");
        if (date == null) {
            throw new JobParametersInvalidException("'date' parameter is required");
        }
        if (date.isAfter(LocalDate.now())) {
            throw new JobParametersInvalidException("Cannot process future date: " + date);
        }
    }
}

// Distributed lock for clustered environments
@Component
public class DistributedJobLauncher {
    @Autowired private LockRepository lockRepository;
    @Autowired private JobLauncher jobLauncher;

    public void launchWithLock(Job job, JobParameters params) {
        String lockKey = job.getName() + "_" + params.toString();
        if (lockRepository.tryAcquire(lockKey, Duration.ofHours(2))) {
            try {
                jobLauncher.run(job, params);
            } finally {
                lockRepository.release(lockKey);
            }
        } else {
            log.warn("Job already running on another node: {}", lockKey);
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Built-in prevention only works for COMPLETED status — FAILED allows re-run *(sirf completed ke liye protection hai — failed restart ho jaata hai)*
- In clusters, two nodes can launch simultaneously before DB check — use distributed lock
- RunIdIncrementer bypasses duplicate prevention by design
- Validator runs BEFORE job starts — fails fast on bad params

### ⚡ Remember
- Built-in: COMPLETED + same params = blocked
- Validator: check required params early *(pehle check karo params sahi hain)*
- Distributed lock: for cluster safety
- DB unique constraint: last line of defense
- Four layers: built-in → validator → lock → DB constraint

### 🔗 Follow-ups
- [Q59 → Same parameters behavior](#q59)
- [Q58 → How identity is computed](#q58)
- [Q110 → Metadata tables and constraints](#q110)

---

## Q61. How do you restart a failed job?

### 📝 One-Liner
Simply re-run with the same identifying parameters — Spring Batch automatically detects the restart and resumes from the failure point.

### 🔑 Quick Answer
To restart a failed job: launch it again with the SAME identifying parameters. Spring Batch finds the existing FAILED JobInstance, creates a new JobExecution, and resumes processing. Completed steps are skipped. Failed/stopped steps resume from the last committed chunk using the checkpoint saved in `ExecutionContext`. No special restart code needed — the framework handles everything. *(Same parameters se dobara run karo — Spring Batch automatically samajh jaata hai ki restart hai)*

### 📖 How It Works
```
Restart Flow:

First Run (FAILED):
  Step 1: "readCSV"    → COMPLETED ✅ (all chunks committed)
  Step 2: "processData"→ FAILED ❌ at chunk 15 of 100
  Step 3: "sendReport" → NOT STARTED

Restart (same params):
  Step 1: "readCSV"    → SKIPPED (already completed)
  Step 2: "processData"→ RESUMES from chunk 15 ← ExecutionContext has position
  Step 3: "sendReport" → EXECUTES after Step 2 completes

Restart Mechanism:
  1. JobLauncher finds existing FAILED JobInstance (same params)
  2. Creates NEW JobExecution under same JobInstance
  3. For each step: check last StepExecution status
     - COMPLETED → skip
     - FAILED/STOPPED → resume (load ExecutionContext)
     - NOT STARTED → execute fresh
```

### 🗣️ Answering Approach
"Restarting a failed job in Spring Batch is simple — I just run it again with the same identifying parameters. The framework automatically detects it's a restart and handles everything. Already completed steps are skipped. The failed step resumes from the last committed chunk using checkpoint data stored in the ExecutionContext. In my project, our scheduler automatically retried failed nightly jobs in the morning. The restart typically processed only the remaining 10-20% of data since the completed chunks from the previous night were already committed."

### 💻 Code
```java
// No special restart code needed — just launch with same params
@Scheduled(cron = "0 0 6 * * ?")  // 6 AM retry
public void retryFailedJobs() {
    List<JobExecution> failedJobs = jobExplorer.findRunningJobExecutions("dailyJob");
    // Also check for FAILED executions from last night
    
    // Just re-launch with same params — Spring Batch handles restart
    for (JobExecution failed : getFailedExecutions()) {
        try {
            jobLauncher.run(dailyJob, failed.getJobParameters());
            log.info("Restarted job: {}", failed.getId());
        } catch (JobInstanceAlreadyCompleteException e) {
            log.info("Job already completed, no restart needed");
        }
    }
}

// Programmatic restart via JobOperator
@Autowired private JobOperator jobOperator;

public void restart(long executionId) throws Exception {
    jobOperator.restart(executionId);  // restarts from last failure point
}
```

### ⚠️ Pitfalls / Gotchas
- Changing identifying params between runs = NEW job, not restart *(params badle toh naya job hai, restart nahi)*
- Multi-threaded steps lose restartability (read order not preserved)
- Reader must support restart — store position in ExecutionContext
- `allowStartIfComplete(true)` forces re-execution of completed steps (rarely needed)
- Custom readers must implement `ItemStreamReader` for restart support

### ⚡ Remember
- Same params = restart, different params = new job
- Completed steps skipped, failed step resumes *(pura step nahi, sirf baaki ka kaam)*
- ExecutionContext stores checkpoint position
- No special code needed — framework handles it
- Multi-threaded step = restart not reliable

### 🔗 Follow-ups
- [Q62 → Internal restart flow](#q62)
- [Q59 → Same parameters status check](#q59)
- [Q66 → Restartability mechanism](#q66)

---

## Q62. What happens when job restart occurs? (Internal flow)

### 📝 One-Liner
Spring Batch finds the previous FAILED execution, creates a new execution, then per-step decides: skip completed, resume failed (loading ExecutionContext), or execute new.

### 🔑 Quick Answer
Internally: **(1)** JobRepository finds the last JobExecution for the JobInstance (same params). **(2)** A new JobExecution is created under the same JobInstance. **(3)** For each step in the job flow: if the step was COMPLETED → skip it; if FAILED or STOPPED → load its ExecutionContext, initialize the reader at the saved position, and resume; if never started → execute fresh. **(4)** The reader's saved position (e.g., line number, page offset) is restored from ExecutionContext. *(Previous FAILED execution dhundhta hai, phir har step ka status check karke decide karta hai — skip, resume, ya fresh start)*

### 📖 How It Works
```
Internal Restart Flow:

1. JobRepository.getLastJobExecution(jobInstance) → FAILED execution #1

2. JobRepository.createJobExecution(jobInstance) → new execution #2

3. For each Step in job:
   ┌─────────────────────────────────────────────────────────────┐
   │ Step "readCSV" → last StepExecution: COMPLETED              │
   │   → SKIP (don't re-execute)                                 │
   ├─────────────────────────────────────────────────────────────┤
   │ Step "processData" → last StepExecution: FAILED             │
   │   → Load ExecutionContext from BATCH_STEP_EXECUTION_CONTEXT │
   │   → ExecutionContext: {reader.position: 7500, page: 15}     │
   │   → Initialize reader at position 7500                      │
   │   → Resume processing from chunk 16                         │
   ├─────────────────────────────────────────────────────────────┤
   │ Step "sendReport" → no previous StepExecution               │
   │   → Execute fresh                                           │
   └─────────────────────────────────────────────────────────────┘

4. Reader restores position:
   JdbcPagingItemReader → starts from page 15 (offset 7500)
   FlatFileItemReader → starts from line 7500
```

### 🗣️ Answering Approach
"Internally, Spring Batch first queries JobRepository to find the last execution for that JobInstance. It creates a new JobExecution under the same instance. Then for each step, it checks the previous StepExecution status — completed steps are skipped, failed steps are resumed by loading their ExecutionContext from the database, and never-started steps execute fresh. The ExecutionContext contains the reader's position, so the reader can initialize at the exact point where processing stopped. In my project, our JdbcPagingItemReader stored the last page offset in context, so restart skipped the already-processed pages."

### 💻 Code
```java
// Built-in readers save position in ExecutionContext automatically
// Custom reader — must implement ItemStreamReader for restart support
@Component
public class CustomApiReader implements ItemStreamReader<ApiRecord> {
    
    private int currentIndex = 0;
    private static final String INDEX_KEY = "current.index";

    @Override
    public void open(ExecutionContext context) {
        // On restart, restore position from saved context
        if (context.containsKey(INDEX_KEY)) {
            currentIndex = context.getInt(INDEX_KEY);
            log.info("Restarting from index: {}", currentIndex);
        }
    }

    @Override
    public ApiRecord read() {
        // Read next item
        if (currentIndex >= totalRecords) return null;
        return fetchRecord(currentIndex++);
    }

    @Override
    public void update(ExecutionContext context) {
        // Save current position after each chunk commit
        context.putInt(INDEX_KEY, currentIndex);
    }
    
    @Override
    public void close() { /* cleanup */ }
}

// Force re-execute a completed step on restart (rarely needed)
@Bean
public Step alwaysRunStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("alwaysRunStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .allowStartIfComplete(true)  // re-run even if completed
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Custom readers MUST implement `ItemStreamReader` for restart support *(custom reader mein open/update zaruri hai restart ke liye)*
- `update()` is called after every chunk commit — saves position
- `open()` is called at step start — restores position from context
- Multi-threaded steps break restart because threads read out of order
- `allowStartIfComplete(true)` is rare — use for steps that must always run (e.g., cleanup)

### 🎯 Tricky Interview Qs

**Q: What happens if the reader's data changed between original run and restart?**
Spring Batch doesn't guarantee consistency with changed data. If new rows were added, some might be skipped (cursor/paging based on position). Use keyset paging and immutable data for reliable restarts.

**Q: At most how much work is lost on a crash?**
One chunk. After each chunk commits, the ExecutionContext is updated. So on crash, only the in-progress (uncommitted) chunk is lost.

### ⚡ Remember
- Find FAILED execution → create new execution → per-step decision
- COMPLETED → skip, FAILED → resume, NOT_STARTED → fresh
- ExecutionContext holds reader position *(reader ka position context mein save hota hai)*
- Custom readers: implement `open()`, `update()`, `close()`
- At most one chunk of work lost on crash

### 🔗 Follow-ups
- [Q61 → How to restart](#q61)
- [Q67 → ExecutionContext details](#q67)
- [Q69 → Resume from failure point](#q69)
