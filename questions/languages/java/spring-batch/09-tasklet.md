# 📋 Tasklet — Q79 to Q83

---

## Q79. What is Tasklet in Spring Batch?

### 📝 One-Liner
Tasklet is a simple, single-operation step that executes one `execute()` method — used for setup, cleanup, file operations, and one-shot tasks.

### 🔑 Quick Answer
A `Tasklet` is a step that runs a single task — no read-process-write cycle. It implements one method: `execute(StepContribution, ChunkContext)` which returns `RepeatStatus.FINISHED` (done) or `RepeatStatus.CONTINUABLE` (run again). Common uses: delete temp files, create directories, run DDL scripts, send notifications, call an API. Each `execute()` call runs in its own transaction. *(Tasklet = ek kaam karo aur khatam — read-process-write nahi hai)*

### 📖 How It Works
```
Tasklet vs Chunk:

Chunk Step:                    Tasklet Step:
┌──────────────────────┐      ┌──────────────────────┐
│ Read → Process → Write│      │ execute() {          │
│ Read → Process → Write│      │   deleteOldFiles();  │
│ Read → Process → Write│      │   return FINISHED;   │
│ ...repeat...         │      │ }                    │
└──────────────────────┘      └──────────────────────┘
 For: processing records       For: single operations

Tasklet Lifecycle:
  1. Spring Batch calls execute()
  2. Returns FINISHED → step ends
  3. Returns CONTINUABLE → execute() called again (new tx)
  4. Throws exception → step FAILED
```

### 🗣️ How to Say in Interview
"A Tasklet is a step for single operations that don't fit the read-process-write pattern. It has one execute method that performs the task and returns a status. In my project, we used tasklets for setup and cleanup: a pre-processing tasklet downloaded the CSV file from SFTP, the main chunk step processed the data, and a post-processing tasklet moved the processed file to an archive directory and sent a completion notification email. Tasklets are simple and keep the job flow clean by separating infrastructure tasks from business logic."

### 💻 Code
```java
// Simple tasklet: cleanup old files
@Component
public class CleanupTasklet implements Tasklet {

    @Override
    public RepeatStatus execute(StepContribution contribution, 
                                ChunkContext chunkContext) throws Exception {
        // Access job parameters
        String directory = chunkContext.getStepContext()
                .getJobParameters().get("outputDir").toString();

        // Delete files older than 30 days
        Path dir = Path.of(directory);
        try (Stream<Path> files = Files.list(dir)) {
            files.filter(f -> isOlderThan30Days(f))
                 .forEach(f -> {
                     try { Files.delete(f); } 
                     catch (IOException e) { log.warn("Failed to delete: {}", f); }
                 });
        }

        return RepeatStatus.FINISHED;  // done — step ends
    }
}

// Register tasklet in a step
@Bean
public Step cleanupStep(JobRepository repo, PlatformTransactionManager tx,
                        CleanupTasklet tasklet) {
    return new StepBuilder("cleanupStep", repo)
            .tasklet(tasklet, tx)
            .build();
}

// Typical job: setup tasklet → chunk step → cleanup tasklet
@Bean
public Job dailyJob(JobRepository repo, Step setupStep, Step processStep, Step cleanupStep) {
    return new JobBuilder("dailyJob", repo)
            .start(setupStep)       // Tasklet: download file, create dirs
            .next(processStep)      // Chunk: read → process → write
            .next(cleanupStep)      // Tasklet: cleanup, notify
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Tasklet has NO built-in skip/retry — must handle errors yourself *(tasklet mein skip/retry khud handle karo)*
- Each `execute()` call runs in its own transaction
- Don't use tasklet for processing collections of records — that's what chunk steps are for
- Returning `CONTINUABLE` without a termination condition = infinite loop
- Can access JobParameters and ExecutionContext via ChunkContext

### 🆚 vs. Comparison
| Aspect | Tasklet | Chunk Processing |
|--------|---------|-----------------|
| Model | Single execute() | Read → Process → Write loop |
| Transaction | Per execute() call | Per chunk |
| Use case | One-shot tasks | Data record processing |
| Skip/Retry | Manual | Built-in |
| Restart | Manual (use ExecutionContext) | Automatic (checkpoint) |
| Examples | Delete files, send email, DDL | CSV processing, DB migration |

### 🎯 Tricky Interview Qs

**Q: Can a tasklet access job parameters?**
Yes. Via `chunkContext.getStepContext().getJobParameters()` or inject `@Value("#{jobParameters['key']}")` with `@StepScope`.

**Q: Is tasklet transactional?**
Yes. Each execute() call runs inside a transaction. If it throws, the transaction rolls back.

### ⚡ Remember
- Single `execute()` method — no read/write cycle
- Returns FINISHED (done) or CONTINUABLE (run again) *(FINISHED = khatam, CONTINUABLE = phir se chalao)*
- No built-in skip/retry (handle manually)
- Great for: file ops, DDL, notifications, cleanup
- Typical job: Tasklet → Chunk → Tasklet

### 🔗 Follow-ups
- [Q80 → Tasklet vs Chunk comparison](#q80)
- [Q81 → When to use which](#q81)
- [Q82 → Running tasklet multiple times](#q82)

---

## Q80. What is the difference between Tasklet and Chunk processing?

### 📝 One-Liner
Tasklet runs a single operation per step; Chunk processes collections of records in a read-process-write loop with built-in fault tolerance and restart.

### 🔑 Quick Answer
**Tasklet**: executes `execute()` method once (or repeatedly with CONTINUABLE). No reader/writer. No built-in skip/retry. Manual restart handling. Use for one-shot tasks. **Chunk**: reads items one-at-a-time, processes each, writes in batches. Built-in fault tolerance (skip, retry), checkpointing, and restart. Use for processing collections of records. *(Tasklet = ek kaam, Chunk = bahut saare records process karo read-write loop mein)*

### 📖 How It Works
```
Tasklet:
  Step → execute() → FINISHED
  ├── One operation per call
  ├── One transaction per call
  └── No read/process/write pattern

Chunk:
  Step → [Read x N → Process x N → Write N] → COMMIT → repeat
  ├── Reads collection of records
  ├── One transaction per chunk
  ├── Built-in skip / retry / checkpoint
  └── Automatic restart from last commit
```

### 🗣️ How to Say in Interview
"The key difference is that chunk processing is designed for record-by-record data processing with built-in fault tolerance and restartability, while tasklets are for single operations. Chunk steps have readers, processors, and writers with skip/retry logic and automatic checkpointing. Tasklets have a single execute method with manual error handling. In my project, I used the rule: 'Am I processing a list of things?' — yes means chunk, no means tasklet. Our job had a tasklet to validate the input file exists, a chunk step to process 500K records, and another tasklet to archive the file and send email."

### 💻 Code
```java
// Job structure: tasklet + chunk + tasklet
@Bean
public Job completeJob(JobRepository repo, 
                       Step validateStep,    // Tasklet
                       Step processStep,     // Chunk
                       Step notifyStep) {    // Tasklet
    return new JobBuilder("completeJob", repo)
            .start(validateStep)    // Tasklet: check file exists
            .next(processStep)      // Chunk: read→process→write 500K records
            .next(notifyStep)       // Tasklet: send email notification
            .build();
}

// Tasklet step: single operation
@Bean
public Step validateStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("validateStep", repo)
            .tasklet((contribution, chunkContext) -> {
                String file = chunkContext.getStepContext()
                    .getJobParameters().get("inputFile").toString();
                if (!Files.exists(Path.of(file))) {
                    throw new FileNotFoundException("Input file not found: " + file);
                }
                return RepeatStatus.FINISHED;
            }, tx).build();
}

// Chunk step: process records
@Bean
public Step processStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(csvReader()).processor(orderProcessor()).writer(dbWriter())
            .faultTolerant().skip(ValidationException.class).skipLimit(100)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Don't use tasklet to process records one-by-one in a loop — use chunk step *(records process karne ke liye tasklet mat use karo — chunk use karo)*
- Tasklet has no concept of `readCount`, `writeCount`, `skipCount`
- Chunk step's restart/checkpoint only works with chunk (not tasklet)
- A tasklet step always shows as COMPLETED or FAILED (no partial progress)

### ⚡ Remember
- **Tasklet** = single operation (no R/P/W cycle) *(ek kaam — file delete, email bhejo)*
- **Chunk** = record processing (R → P → W loop)
- Chunk has skip/retry/checkpoint; Tasklet doesn't
- Rule: "Processing a LIST?" → Chunk; "Single task?" → Tasklet
- Typical job: Tasklet setup → Chunk process → Tasklet cleanup

### 🔗 Follow-ups
- [Q79 → Tasklet basics](#q79)
- [Q81 → When to use which](#q81)
- [Q21 → Chunk processing basics](#q21)

---

## Q81. When should you use Tasklet vs Chunk?

### 📝 One-Liner
Use Tasklet for one-shot operations (file ops, notifications, DDL); use Chunk for processing collections of records.

### 🔑 Quick Answer
Ask: **"Am I processing a LIST of things?"** Yes → Chunk. No → Tasklet. **Tasklet** use cases: delete/move/download files, create directories, run SQL scripts, send notifications, call an API once, validate preconditions. **Chunk** use cases: process CSV records, migrate database rows, process messages, transform data sets. The decision is about the nature of the work, not complexity. *(LIST process karna hai? = Chunk. Single operation? = Tasklet)*

### 📖 How It Works
```
Decision Guide:

"Am I processing a LIST of things?"
├── YES → Chunk Processing
│   ├── CSV rows
│   ├── Database records
│   ├── Messages from queue
│   └── API records to process
│
└── NO → Tasklet
    ├── Delete temp files
    ├── Download file from SFTP
    ├── Create output directory
    ├── Send email notification
    ├── Execute DDL script
    ├── Call API once
    └── Validate file exists
```

### 🗣️ How to Say in Interview
"I follow a simple rule: if I'm processing a collection of records, I use a chunk step because I get built-in fault tolerance, checkpointing, and restart for free. If it's a single operation like moving a file or sending a notification, I use a tasklet. In my project, we had a job with five steps: tasklet to download the file from SFTP, chunk step to validate records, chunk step to process and write to database, tasklet to archive the file, and tasklet to send summary email. Each step type matched the nature of the operation."

### 💻 Code
```java
// Full job demonstrating when to use each
@Bean
public Job endToEndJob(JobRepository repo,
                       Step downloadStep,      // Tasklet: SFTP download
                       Step validateStep,       // Chunk: validate 500K records
                       Step processStep,        // Chunk: process records
                       Step archiveStep,        // Tasklet: move file
                       Step notifyStep) {       // Tasklet: email
    return new JobBuilder("endToEndJob", repo)
            .start(downloadStep)
            .next(validateStep)
            .next(processStep)
            .next(archiveStep)
            .next(notifyStep)
            .build();
}
```

### ⚡ Remember
- **"Am I processing a LIST?"** — the key question
- Chunk: records, rows, messages *(list of items = Chunk)*
- Tasklet: file ops, notifications, DDL, single API call
- Chunk gets skip/retry/restart for free
- Mix both in a job: Tasklet → Chunk → Tasklet

### 🔗 Follow-ups
- [Q79 → Tasklet details](#q79)
- [Q80 → Tasklet vs Chunk comparison](#q80)
- [Q21 → Chunk processing basics](#q21)

---

## Q82. Can a Tasklet run multiple times?

### 📝 One-Liner
Yes — return `RepeatStatus.CONTINUABLE` to run again; each call gets its own transaction; must have a termination condition to avoid infinite loop.

### 🔑 Quick Answer
When `execute()` returns `RepeatStatus.CONTINUABLE`, Spring Batch calls `execute()` again immediately in a new transaction. Returns `FINISHED` to stop. Use case: batch deletes (delete 1000 records per call to avoid giant transactions). Each call = separate transaction. MUST have a termination condition — otherwise infinite loop. `RepeatStatus.continueIf(boolean)` is a convenience method. *(CONTINUABLE return karo toh phir se chalega — FINISHED karo toh ruk jaayega)*

### 📖 How It Works
```
Tasklet Repeat Behavior:

Call 1: execute() → delete 1000 rows → return CONTINUABLE → new tx
Call 2: execute() → delete 1000 rows → return CONTINUABLE → new tx
Call 3: execute() → delete 500 rows  → return FINISHED → STOP

Each call = separate transaction
Total: 2500 rows deleted across 3 transactions
(vs. one giant DELETE 2500 in single tx → lock issues)
```

### 🗣️ How to Say in Interview
"Yes, a tasklet can run multiple times by returning RepeatStatus.CONTINUABLE. Each call runs in its own transaction. I used this pattern for batch cleanup — deleting old records 1000 at a time to avoid long-running transactions that would lock the table. Each call deleted 1000 rows and returned CONTINUABLE. When fewer than 1000 were deleted, it returned FINISHED. This gave us controlled, incremental cleanup without giant transactions."

### 💻 Code
```java
// Batch delete tasklet — runs multiple times
@Component
public class BatchDeleteTasklet implements Tasklet {

    @Autowired private JdbcTemplate jdbc;
    private static final int BATCH_SIZE = 1000;

    @Override
    public RepeatStatus execute(StepContribution contribution,
                                ChunkContext chunkContext) {
        int deleted = jdbc.update(
            "DELETE FROM old_records WHERE created_date < ? LIMIT ?",
            LocalDate.now().minusDays(90), BATCH_SIZE);

        log.info("Deleted {} old records", deleted);

        // CONTINUABLE if more to delete, FINISHED if done
        return RepeatStatus.continueIf(deleted == BATCH_SIZE);
    }
}

// Simple one-shot tasklet (most common — 95% of tasklets)
@Bean
public Step oneTimeStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("oneTimeStep", repo)
            .tasklet((contribution, context) -> {
                sendEmail("Job completed successfully");
                return RepeatStatus.FINISHED;  // run once only
            }, tx).build();
}
```

### ⚠️ Pitfalls / Gotchas
- No termination condition = INFINITE LOOP *(CONTINUABLE mein condition nahi diya toh infinite loop)*
- Each call is a separate transaction — not one big transaction
- No built-in checkpoint like chunk processing
- `RepeatStatus.continueIf(deleted >= BATCH_SIZE)` — convenience method
- Most tasklets (95%) just return FINISHED

### ⚡ Remember
- CONTINUABLE = run again, FINISHED = stop *(CONTINUABLE = phir chalao, FINISHED = band karo)*
- Each call = new transaction
- Great for: batch deletes, incremental cleanup
- MUST have termination condition
- `RepeatStatus.continueIf(boolean)` for cleaner code

### 🔗 Follow-ups
- [Q83 → RepeatStatus details](#q83)
- [Q79 → Tasklet basics](#q79)
- [Q80 → Tasklet vs Chunk](#q80)

---

## Q83. What is RepeatStatus?

### 📝 One-Liner
RepeatStatus is an enum with two values: `FINISHED` (step complete, move on) and `CONTINUABLE` (call execute again).

### 🔑 Quick Answer
`RepeatStatus` controls tasklet repetition: **FINISHED** = done, proceed to next step (used 95% of the time). **CONTINUABLE** = call execute() again in a new transaction (used for incremental operations). Convenience method: `RepeatStatus.continueIf(condition)` returns CONTINUABLE if condition is true, FINISHED if false. *(FINISHED = khatam agla step pe jao, CONTINUABLE = ek aur baar chalao)*

### 📖 How It Works
```
RepeatStatus Enum:

RepeatStatus.FINISHED:
  execute() → FINISHED → step COMPLETED → move to next step

RepeatStatus.CONTINUABLE:
  execute() → CONTINUABLE → execute() again (new tx) → CONTINUABLE → ...
  → eventually FINISHED → step COMPLETED

RepeatStatus.continueIf(boolean):
  continueIf(true)  → CONTINUABLE
  continueIf(false) → FINISHED
```

### 🗣️ How to Say in Interview
"RepeatStatus is an enum that controls whether a tasklet runs again or finishes. FINISHED means the step is done and the job moves to the next step — this is what 95% of tasklets return. CONTINUABLE tells Spring Batch to call execute again in a new transaction, useful for batch operations like deleting records in chunks. The continueIf convenience method makes conditional returns cleaner."

### 💻 Code
```java
// Most common: FINISHED (one-shot)
return RepeatStatus.FINISHED;

// Repeat: CONTINUABLE
return RepeatStatus.CONTINUABLE;

// Conditional: continueIf
int deleted = deleteOldRecords(1000);
return RepeatStatus.continueIf(deleted == 1000);  // continue if full batch deleted
```

### ⚡ Remember
- Two values: FINISHED and CONTINUABLE
- FINISHED = done (95% of tasklets) *(zyada tar FINISHED use hota hai)*
- CONTINUABLE = run again (batch operations)
- `continueIf(boolean)` = convenience method
- Each CONTINUABLE call = new transaction

### 🔗 Follow-ups
- [Q82 → Running tasklet multiple times](#q82)
- [Q79 → Tasklet basics](#q79)
- [Q80 → When to use tasklet](#q80)
