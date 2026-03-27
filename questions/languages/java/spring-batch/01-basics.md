# 🟢 Spring Batch — Basics (Q1–Q20)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. What is Spring Batch?

### 📝 One-Liner
> Framework for reliable bulk data processing — Read → Process → Write in chunks with restart, skip, retry.

### 🔑 Quick Answer
> Spring Batch is a **lightweight framework** for processing large data in **chunks** — reads from source, processes, writes to destination. Provides **transaction management**, **restart from failure point**, **skip/retry** for bad records, and **parallel processing**. Does NOT schedule jobs — only executes. *(Bulk data ka framework — 10 lakh records safely process karo, beeche mein fail hua toh wahin se restart)*

### 📖 How It Works
```
INPUT (CSV/DB/API)  → [ItemReader] → [ItemProcessor] → [ItemWriter] → OUTPUT (DB/File)
                          ↑               ↑                ↑
                    Spring Batch handles: transactions, restart, skip, retry

Example: 10M customer records in CSV
  → Read 500 at a time (chunk)
  → Validate & transform each
  → Batch insert 500 to DB (1 transaction)
  → Repeat until done
  *(500-500 karke padhega, process karega, DB mein daalega — agar fail hua toh wahin se dobara)*
```

### 💻 Code
```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {
    @Bean
    public Job myJob(JobRepository repo, Step step1) {
        return new JobBuilder("myJob", repo)
                .start(step1)
                .build();
    }

    @Bean
    public Step step1(JobRepository repo, PlatformTransactionManager tx) {
        return new StepBuilder("step1", repo)
                .<Order, Order>chunk(500, tx)  // 500 per transaction
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .build();
    }
}
```

### 🗣️ How to Say in Interview
> *"Spring Batch is a framework for batch processing — when you need to process millions of records reliably. It follows Read → Process → Write in chunks. If I have 10 million records, it reads 500 at a time, processes each, writes all 500 in one transaction. If it fails midway, it restarts from last committed chunk. It also has skip logic for bad records and retry for transient errors. In my project, we used it for daily ETL — reading CSV feeds and loading into the database."*

### ⚡ Remember
1. **Read → Process → Write** in chunks
2. **Transaction per chunk** (auto-managed)
3. **Restart from failure point** *(wahin se dobara)*
4. **Does NOT schedule** — only executes
5. Built on Spring Framework

### 🔗 Follow-ups
→ [Q4. Core components](#q4) → [Q21. Chunk processing](02-chunk-processing.md#q21)

---

<a id="q2"></a>
## Q2. What are the main use cases of Spring Batch?

### 📝 One-Liner
> ETL pipelines, data migration, report generation, bulk file processing, database cleanup, billing systems.

### 🔑 Quick Answer
> Any **high-volume data processing**: **ETL** (CSV → DB), **data migration** (Oracle → MySQL), **report generation** (monthly billing for 2M customers), **file processing** (50K daily cheque images), **database cleanup** (archive old records). *(Lakho records ka kaam — CSV se DB, purana data archive, monthly report)*

### 🗣️ How to Say in Interview
> *"In my experience, Spring Batch is ideal for high-volume data processing. We used it for an ETL pipeline — reading millions of records from CSV, validating, transforming, and writing to the database. We also used it for monthly report generation with 2 million customer records. The advantage over a simple loop is transaction safety, restartability, and built-in monitoring."*

### ⚡ Remember
1. **ETL** = most common use case ⭐
2. **Data migration** between databases
3. **Report generation** (monthly/quarterly)
4. **File processing** (CSV, XML, fixed-width)
5. NOT for real-time — use Kafka/WebFlux for that

### 🔗 Follow-ups
→ [Q3. Problems it solves](#q3) → [Q99. Scheduling](12-scheduling.md#q99)

---

<a id="q3"></a>
## Q3. What problems does Spring Batch solve?

### 📝 One-Liner
> Transaction management, failure recovery, restart from checkpoint, skip/retry, parallel processing, monitoring — all out of the box.

### 🔑 Quick Answer
> Without Spring Batch, you'd build **all plumbing yourself** — transaction commits every N records, checkpoint saving, error handling, progress tracking. Spring Batch provides: **chunk transactions**, **restart from last committed point**, **skip/retry**, **metadata tables** for monitoring, **parallel processing**. *(Sab kuch khud likhna padta — Spring Batch sab de deta hai ready-made)*

### 🆚 vs. Comparison
| Problem | DIY Code | Spring Batch |
|---------|---------|-------------|
| Transaction | Manual commit every N | Automatic per chunk ✅ |
| Restart | Save checkpoint yourself | Auto from metadata ✅ |
| Error handling | Custom try-catch | skip()/retry() config ✅ |
| Monitoring | Custom logging | Metadata tables ✅ |
| Parallel | Manual thread mgmt | Partitioning built-in ✅ |

### ⚡ Remember
1. **Transaction** → automatic per chunk
2. **Restart** → resumes from last checkpoint *(wahin se)*
3. **Skip/Retry** → configurable error handling
4. **Monitoring** → metadata tables track everything
5. **Scaling** → built-in partitioning

### 🔗 Follow-ups
→ [Q63. Transaction handling](07-transactions-restart.md#q63) → [Q66. Restart](07-transactions-restart.md#q66)

---

<a id="q4"></a>
## Q4. What are the core components of Spring Batch?

### 📝 One-Liner
> Job (whole process) → Step (one phase) → Reader/Processor/Writer (actual work) + JobRepository (metadata) + JobLauncher (trigger).

### 🔑 Quick Answer
> **Job** = entire batch process. **Step** = one phase (chunk-based or Tasklet). **ItemReader/Processor/Writer** = read, transform, write. **JobRepository** = stores execution metadata in DB. **JobLauncher** = triggers the job. **JobParameters** = unique identifier for each run. *(Job ke andar Steps, Step ke andar Reader-Processor-Writer — sab ka record JobRepository mein)*

### 📖 How It Works
```
You / Scheduler
      │
      ▼
 JobLauncher ─── "Run this job!"
      │
      ▼
    Job ─── "I am the whole batch process"
      │
      ├── Step 1 (chunk-based)
      │     ├── ItemReader     → reads data
      │     ├── ItemProcessor  → transforms
      │     └── ItemWriter     → writes batch
      │
      ├── Step 2 (tasklet)
      │     └── Tasklet        → single task
      │
      └── Step 3 ...

 JobRepository → stores metadata (status, counts, checkpoints)
 *(Sab kuch record hota hai — kitne padhe, kitne likhe, kab fail hua)*
```

### 🗣️ How to Say in Interview
> *"Spring Batch has a clear hierarchy. Job is the entire batch process containing Steps. Each Step has an ItemReader, ItemProcessor, and ItemWriter. The JobLauncher triggers execution, and the JobRepository stores all metadata in the database — status, read/write counts, timestamps. This metadata enables restartability and monitoring."*

### ⚡ Remember
1. **Job** = entire process, **Step** = one phase
2. **Reader** reads one-at-a-time, **Writer** writes whole chunk
3. **Processor** is optional
4. **JobRepository** = metadata storage (mandatory)
5. **JobLauncher** = trigger

### 🔗 Follow-ups
→ [Q5. What is Job](#q5) → [Q6. What is Step](#q6)

---

<a id="q5"></a>
## Q5. What is a Job in Spring Batch?

### 📝 One-Liner
> Top-level container for the entire batch process — identified by name + parameters, contains ordered Steps.

### 🔑 Quick Answer
> A Job = **entire batch process** containing ordered Steps. **Job + Parameters = JobInstance** (unique identity). One JobInstance can have multiple **JobExecutions** (on failure + restart). A COMPLETED JobInstance can't re-run with same params. *(Job = poora kaam, Parameters se pehchaan — fail hua toh dobara try, complete hua toh same params se nahi chalega)*

### 📖 How It Works
```
Job: "monthlyBilling"
│
├── JobInstance (month=2024-01)
│   ├── JobExecution #1 → FAILED (crashed at step 2)
│   └── JobExecution #2 → COMPLETED (restarted, finished)
│
├── JobInstance (month=2024-02)
│   └── JobExecution #1 → COMPLETED
│
Job = definition (class)
JobInstance = logical run (params identify it)
JobExecution = physical attempt (can retry)
*(Ek Job, alag params = alag instance — fail hua toh naya execution)*
```

### 🗣️ How to Say in Interview
> *"A Job represents the entire batch process. It's identified by name plus parameters — so 'monthlyBilling' with month=January is one JobInstance. If it fails and I restart with the same parameters, Spring Batch creates a new JobExecution under the same JobInstance — it knows it's a retry. A completed JobInstance cannot re-run with same parameters."*

### ⚡ Remember
1. **Job + Params = JobInstance** (unique identity)
2. **JobInstance** can have multiple **JobExecutions**
3. **COMPLETED** can't re-run with same params *(dobara nahi chalega)*
4. Steps execute in defined order
5. JobRepository tracks everything

### 🔗 Follow-ups
→ [Q55. JobInstance vs JobExecution](06-job-execution.md#q55)

---

<a id="q6"></a>
## Q6. What is a Step?

### 📝 One-Liner
> One independent phase within a Job — either chunk-based (read-process-write) or Tasklet (single task).

### 🔑 Quick Answer
> Step = **one phase** of the Job. Two types: **Chunk-based** (read N items → process → write as batch) and **Tasklet** (execute a single task like cleanup, file move). Each step has its own **transaction**, **execution context**, and **status tracking**. *(Job ka ek kadam — ya toh chunk wala ya tasklet wala)*

### 🆚 vs. Comparison
| | Chunk-based | Tasklet |
|-|------------|---------|
| Pattern | Read → Process → Write loop | Single execute() call |
| Use for | Data processing (ETL) | Simple tasks (cleanup, file move) |
| Transaction | Per chunk | Per execution |
| Restart | From last committed chunk | Re-executes entirely |

### ⚡ Remember
1. **Chunk** = read-process-write loop *(data processing)*
2. **Tasklet** = single task execution *(cleanup, file ops)*
3. Each step has **own transaction** and **status**
4. Steps run **sequentially** by default
5. Can be **conditional** (if step1 fails → go to step3)

### 🔗 Follow-ups
→ [Q7. What is a chunk](#q7) → [Q79. What is Tasklet](09-tasklet.md#q79)

---

<a id="q7"></a>
## Q7. What is a chunk?

### 📝 One-Liner
> A group of N items read, processed, and written as a single transaction — if chunk fails, only that chunk rolls back.

### 🔑 Quick Answer
> A chunk = **N items processed in one transaction**. Reader reads N items one-by-one, processor transforms each, writer writes all N at once. If chunk fails → only that chunk rolls back, previous chunks are safe. *(500 items ek batch mein — fail hua toh sirf wo 500 rollback, pehle wale safe)*

### 📖 How It Works
```
chunk(500):
  Reader:  read() read() read() ... (500 times → 500 items)
  Processor: process(item1) process(item2) ... (500 transforms)
  Writer:  write([all 500 items])  ← ONE batch write
  → COMMIT transaction ✅
  
  Next chunk: read 500 more... repeat until reader returns null
  *(500-500 karke process — har 500 pe commit)*
```

### ⚡ Remember
1. **N items** = one transaction
2. Reader reads **one-by-one**, writer writes **all at once**
3. Fail → only **that chunk** rolls back
4. Previous committed chunks = **safe**
5. Reader returns null → step complete

### 🔗 Follow-ups
→ [Q21. Chunk processing deep dive](02-chunk-processing.md#q21)

---

<a id="q8"></a>
## Q8. How does Job → Step → Chunk hierarchy work?

### 📝 One-Liner
> Job contains Steps, each Step processes data in Chunks — Job is the recipe, Step is a task, Chunk is a batch of items.

### 📖 How It Works
```
Job: "processOrders"
│
├── Step 1: "readAndProcess" (chunk=500)
│   ├── Chunk 1: items 1-500 → COMMIT ✅
│   ├── Chunk 2: items 501-1000 → COMMIT ✅
│   ├── Chunk 3: items 1001-1500 → FAILED ❌ (rollback this chunk only)
│   └── Chunk 3 (retry): items 1001-1500 → COMMIT ✅
│
├── Step 2: "generateReport" (tasklet)
│   └── Single execution → COMMIT ✅
│
└── Job COMPLETED ✅

*(Job ke andar Steps, Step ke andar Chunks — har chunk ek transaction)*
```

### ⚡ Remember
1. **Job** → contains Steps (sequential)
2. **Step** → processes in Chunks (repeated)
3. **Chunk** → N items in one transaction
4. Fail at chunk level → rollback only that chunk
5. Previous chunks always safe ✅

### 🔗 Follow-ups
→ [Q21. Chunk processing](02-chunk-processing.md#q21)

---

<a id="q9"></a>
## Q9. What is ItemReader, ItemProcessor, ItemWriter?

### 📝 One-Liner
> Reader reads one item at a time, Processor transforms each item (optional), Writer writes the whole chunk at once.

### 🔑 Quick Answer
> **ItemReader** — reads one item per `read()` call, returns null when done. **ItemProcessor** — transforms/validates one item, return null to filter out. **ItemWriter** — receives the entire chunk and writes in batch. *(Reader ek ek padhta hai, Processor badalta hai, Writer sab ek saath likhta hai)*

### 💻 Code
```java
// Reader — one item at a time
public interface ItemReader<T> {
    T read() throws Exception;  // returns null when done
}

// Processor — transform one item (return null = filter out)
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
}

// Writer — write entire chunk at once
public interface ItemWriter<T> {
    void write(Chunk<? extends T> items) throws Exception;
}
```

### ⚡ Remember
1. **Reader**: one-at-a-time, null = done
2. **Processor**: transform/filter, null = skip item
3. **Writer**: whole chunk at once (batch write)
4. Processor is **optional**
5. Writer gets list of all processed items in chunk

### 🔗 Follow-ups
→ [Q31. Reader types](03-readers.md#q31) → [Q41. Writer types](04-writers.md#q41) → [Q49. Processor](05-processors.md#q49)

---

<a id="q10"></a>
## Q10. What is JobRepository?

### 📝 One-Liner
> Database-backed metadata store — tracks job/step status, read/write counts, execution context, enables restartability.

### 🔑 Quick Answer
> JobRepository stores **all execution metadata** in the database: job status, step status, read/write/skip counts, execution context (checkpoints). This is what enables **restartability** — on restart, Spring Batch reads the last checkpoint from JobRepository. *(Sab kuch DB mein record — kitne padhe, kitne likhe, kahan ruka — restart ke liye zaroori)*

### ⚠️ Pitfalls / Gotchas
- **Requires a database** — even for testing (use H2 in-memory) *(bina DB ke Spring Batch nahi chalega)*
- Don't delete metadata tables in production — restart won't work *(tables delete kiya toh restart nahi hoga)*

### ⚡ Remember
1. Stores in **database tables** (BATCH_JOB_INSTANCE, etc.)
2. Enables **restartability** (knows where to resume)
3. Tracks **read/write/skip counts** per step
4. **Mandatory** — can't run without it
5. Use H2 in-memory for testing

### 🔗 Follow-ups
→ [Q110. Metadata tables](14-database-metadata.md#q110) → [Q18. Metadata tables overview](#q18)

---

<a id="q11"></a>
## Q11. What is JobLauncher?

### 📝 One-Liner
> Entry point to trigger a Job with parameters — can run synchronously (waits) or asynchronously (returns immediately).

### 🔑 Quick Answer
> `JobLauncher` triggers a Job with `JobParameters`. Default `SimpleJobLauncher` runs **synchronously** (caller waits until job finishes). Configure with `TaskExecutor` for **async** (returns immediately, job runs in background). *(Job start karne ka button — sync = wait karo, async = turant return)*

### 💻 Code
```java
// Synchronous (default) — caller waits
jobLauncher.run(myJob, new JobParametersBuilder()
    .addString("date", "2024-01-15")
    .toJobParameters());

// Async — configure with TaskExecutor
SimpleJobLauncher launcher = new SimpleJobLauncher();
launcher.setJobRepository(jobRepository);
launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());  // async!
```

### ⚡ Remember
1. **Triggers** Job with JobParameters
2. Default = **synchronous** (waits for completion)
3. With TaskExecutor = **async** (returns immediately)
4. Returns **JobExecution** with status
5. Checks JobRepository before running (duplicate prevention)

### 🔗 Follow-ups
→ [Q12. JobParameters](#q12)

---

<a id="q12"></a>
## Q12. What is JobParameters?

### 📝 One-Liner
> Key-value pairs that uniquely identify a JobInstance — same Job + same params = same instance, can't re-run if completed.

### 🔑 Quick Answer
> JobParameters = **input parameters** passed when launching a job. **Job name + parameters = unique JobInstance**. Same job with same params won't re-run if already COMPLETED. Types: String, Long, Double, Date. *(Parameters se job ki pehchaan — same params = same run, complete hua toh dobara nahi chalega)*

### 💻 Code
```java
JobParameters params = new JobParametersBuilder()
    .addString("inputFile", "orders_2024.csv")
    .addLocalDate("date", LocalDate.now())
    .addLong("run.id", System.currentTimeMillis())  // unique each time!
    .toJobParameters();

jobLauncher.run(myJob, params);
```

### 🎯 Tricky Interview Qs
**Q: How to re-run a completed job with same parameters?**
> Add a unique parameter like `run.id = System.currentTimeMillis()` — makes each invocation a new JobInstance. *(Ek unique param daal do — har baar naya instance banega)*

### ⚡ Remember
1. **Job + Params = JobInstance** (unique identity)
2. COMPLETED can't re-run with same params
3. Add **run.id** for re-runnable jobs ⭐
4. Types: String, Long, Double, Date
5. Accessed via `@Value("#{jobParameters['key']}")`

### 🔗 Follow-ups
→ [Q15. Same params re-run](#q15) → [Q58. Unique JobInstance](06-job-execution.md#q58)

---

<a id="q13"></a>
## Q13. What is a repeatable job?

### 📝 One-Liner
> A job that can run multiple times with same logic but different parameters — each run = new JobInstance.

### 🔑 Quick Answer
> A repeatable job uses **different parameters** each time (date, file name, run.id) so each execution creates a **new JobInstance**. The job logic stays same but input changes. *(Har baar naye params — roz ka job roz alag instance)*

### ⚡ Remember
1. Same logic, **different parameters** each run
2. Each run = **new JobInstance**
3. Use date or run.id to make unique
4. Example: daily ETL with date param
5. vs. Restartable = re-running FAILED instance

### 🔗 Follow-ups
→ [Q14. Restartable job](#q14)

---

<a id="q14"></a>
## Q14. What is a restartable job?

### 📝 One-Liner
> A job that can resume from where it failed — Spring Batch saves checkpoint in metadata, restart picks up from last committed chunk.

### 🔑 Quick Answer
> If a job **fails** at record 5000, a restartable job **resumes from 5001** (not from start). Spring Batch saves the **checkpoint** in ExecutionContext via JobRepository. Default: jobs are restartable. Set `preventRestart()` to disable. *(Fail hua toh wahin se dobara — pehle se shuru nahi)*

### 📖 How It Works
```
Run 1: Chunk 1 ✅ → Chunk 2 ✅ → Chunk 3 ❌ FAILED
  → ExecutionContext saves: "lastProcessedId = 1000"

Run 2 (restart): Reads ExecutionContext → starts from 1001
  → Chunk 3 ✅ → Chunk 4 ✅ → COMPLETED
  *(Pehle wale chunks safe — sirf fail wala dobara)*
```

### ⚡ Remember
1. **Resumes from failure point** (not from start)
2. Checkpoint saved in **ExecutionContext** *(DB mein save)*
3. Default = restartable
4. `preventRestart()` disables it
5. Requires **same JobInstance** (same params)

### 🔗 Follow-ups
→ [Q66. Restartability details](07-transactions-restart.md#q66)

---

<a id="q15"></a>
## Q15. What happens if you run a job with same parameters?

### 📝 One-Liner
> If COMPLETED → throws JobInstanceAlreadyCompleteException. If FAILED → creates new execution (restart).

### 🔑 Quick Answer
> **COMPLETED** → exception (won't re-run). **FAILED** → new JobExecution under same JobInstance (restart). **RUNNING** → exception (already running). To force re-run: add unique param like `run.id`. *(Complete hua toh same params se nahi chalega — fail hua toh restart hoga)*

### 🎯 Tricky Interview Qs
**Q: How to force re-run a completed job?**
> Add `addLong("run.id", System.currentTimeMillis())` — creates new JobInstance. *(Unique param daal do)*

### ⚡ Remember
1. **COMPLETED** → exception *(dobara nahi)*
2. **FAILED** → restart (new execution)
3. **RUNNING** → exception (already running)
4. Add **run.id** to force re-run ⭐
5. Same params = same JobInstance

### 🔗 Follow-ups
→ [Q59. Duplicate execution](06-job-execution.md#q59)

---

<a id="q16"></a>
## Q16. What is a job execution listener?

### 📝 One-Liner
> Interceptor that runs before/after a Job — for logging, notification, cleanup, metrics.

### 🔑 Quick Answer
> `JobExecutionListener` has `beforeJob()` and `afterJob()` callbacks. Use for: **startup logging**, **sending completion notifications**, **cleanup**, **metrics recording**. Can be annotation-based (`@BeforeJob`, `@AfterJob`) or interface-based. *(Job shuru hone se pehle kuch karo, khatam hone ke baad kuch karo)*

### 💻 Code
```java
@Component
public class JobNotificationListener implements JobExecutionListener {
    @Override
    public void beforeJob(JobExecution exec) {
        log.info("Job {} starting", exec.getJobInstance().getJobName());
    }

    @Override
    public void afterJob(JobExecution exec) {
        if (exec.getStatus() == BatchStatus.COMPLETED) {
            sendSuccessNotification();
        } else {
            sendFailureAlert(exec.getAllFailureExceptions());
        }
    }
}
```

### ⚡ Remember
1. **beforeJob()** — setup, logging
2. **afterJob()** — notification, cleanup, metrics
3. Can check **status** in afterJob (COMPLETED/FAILED)
4. Register via `.listener(myListener)` on Job builder
5. Also: annotation-based `@BeforeJob`, `@AfterJob`

### 🔗 Follow-ups
→ [Q17. Step listener](#q17)

---

<a id="q17"></a>
## Q17. What is a step execution listener?

### 📝 One-Liner
> Interceptor for before/after a Step — for step-level logging, validation, and resource management.

### 🔑 Quick Answer
> `StepExecutionListener` has `beforeStep()` and `afterStep()`. Also: **ChunkListener** (before/after each chunk), **ItemReadListener**, **ItemWriteListener**, **SkipListener**. Each gives hooks at different granularity levels. *(Step level pe bhi before/after — aur finer levels pe bhi listeners hain)*

### ⚡ Remember
1. **StepListener** — before/after step
2. **ChunkListener** — before/after each chunk
3. **SkipListener** — when items are skipped
4. **ItemReadListener** — on read events
5. Multiple listener levels for different needs

### 🔗 Follow-ups
→ [Q30. Chunk listener](02-chunk-processing.md#q30)

---

<a id="q18"></a>
## Q18. What are Spring Batch metadata tables?

### 📝 One-Liner
> 6 database tables that store all execution metadata — job instances, executions, step details, parameters, context.

### 🔑 Quick Answer
> Spring Batch auto-creates **6 tables**: `BATCH_JOB_INSTANCE`, `BATCH_JOB_EXECUTION`, `BATCH_JOB_EXECUTION_PARAMS`, `BATCH_STEP_EXECUTION`, `BATCH_JOB_EXECUTION_CONTEXT`, `BATCH_STEP_EXECUTION_CONTEXT`. These enable **restartability**, **monitoring**, and **duplicate prevention**. *(6 tables mein sab record — kab chala, kahan ruka, kitne records)*

### 📖 How It Works
```
BATCH_JOB_INSTANCE     → unique job identity (name + params hash)
BATCH_JOB_EXECUTION    → each attempt (status, start/end time)
BATCH_JOB_EXEC_PARAMS  → job parameters
BATCH_STEP_EXECUTION   → each step's read/write/skip counts
BATCH_JOB_EXEC_CONTEXT → job-level shared state
BATCH_STEP_EXEC_CONTEXT → step-level checkpoint data

*(Sab kuch in 6 tables mein — monitoring aur restart ke liye)*
```

### ⚡ Remember
1. **6 tables** auto-created by Spring Batch
2. Enable **restart** (checkpoint in context tables)
3. Enable **monitoring** (counts in step execution)
4. Enable **duplicate prevention** (job instance uniqueness)
5. Schema scripts: `schema-*.sql` in spring-batch jar

### 🔗 Follow-ups
→ [Q110. Table details](14-database-metadata.md#q110)

---

<a id="q19"></a>
## Q19. How does Spring Batch ensure idempotency?

### 📝 One-Liner
> JobInstance uniqueness (same params = same instance) + ExecutionContext checkpoints + restart from last committed point.

### 🔑 Quick Answer
> **Idempotency** = running the same job multiple times produces same result. Spring Batch ensures this via: **JobInstance uniqueness** (same params = won't re-run if completed), **ExecutionContext checkpoints** (restart from where it left off, not from start), and **chunk transactions** (each chunk is all-or-nothing). *(Ek baar COMPLETE hua toh dobara nahi chalega — restart hua toh wahin se — duplicate nahi hoga)*

### ⚠️ Pitfalls / Gotchas
- **Your writer** must be idempotent too — use UPSERT or check-before-insert *(Spring Batch framework idempotent hai par tumhara code bhi hona chahiye)*
- Multi-threaded steps can break ordering — ensure writer handles duplicates

### ⚡ Remember
1. **Same params** = same JobInstance (no duplicate run)
2. **Checkpoint restart** = no duplicate processing
3. **Your code** must be idempotent too ⭐
4. Use UPSERT in writer for safety
5. Test idempotency by running job twice

### 🔗 Follow-ups
→ [Q122. Avoid duplicate processing](15-production-scenarios.md#q122)

---

<a id="q20"></a>
## Q20. Can you run multiple jobs simultaneously?

### 📝 One-Liner
> Yes — each job runs independently with its own JobExecution; use separate thread pools to avoid resource contention.

### 🔑 Quick Answer
> **Yes**. Each job gets its own `JobExecution` tracked independently. For parallel execution, configure `JobLauncher` with `TaskExecutor`. Use **separate thread pools** for different jobs to avoid contention. Watch for: **database connection pool exhaustion** and **shared resource conflicts**. *(Haan — alag alag pool mein chalao, DB connections ka dhyan rakhna)*

### ⚠️ Pitfalls / Gotchas
- **DB connection pool** may exhaust if too many jobs + steps *(connections khatam ho sakte hain)*
- Jobs sharing **same tables** may cause lock contention
- Use **separate schemas** or **time-based scheduling** to avoid conflicts

### ⚡ Remember
1. **Yes** — independent JobExecutions
2. Use **separate thread pools** per job
3. Watch **DB connection pool** limits
4. Avoid **shared table** contention
5. Better to **schedule at different times** if possible

### 🔗 Follow-ups
→ [Q119. Multiple jobs simultaneously](15-production-scenarios.md#q119)

---

> **🎯 Navigation:** [Next → Chunk Processing (Q21-30)](02-chunk-processing.md) | [📋 All Sections](README.md)
