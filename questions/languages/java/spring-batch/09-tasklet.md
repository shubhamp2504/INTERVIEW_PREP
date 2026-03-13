# 🟢 Spring Batch — Tasklet (Q79–Q83)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q79"></a>

## Q79. What is Tasklet in Spring Batch?

### 🔑 Quick Answer

> A Tasklet is a **simple, single-task step** — it runs a piece of code once (or repeatedly) without the read-process-write pattern. Use it for setup, cleanup, notifications, or any task that doesn't involve processing data records.

### 📖 Step-by-Step Explanation

**Step 1 — Tasklet vs Chunk (the two step types):**

```
Job: "monthlyPayroll"
│
├── Step 1: "validateInputFiles"      ← TASKLET (check files exist)
├── Step 2: "processEmployees"        ← CHUNK (read → process → write)
├── Step 3: "generateReport"          ← CHUNK (read → process → write)
├── Step 4: "archiveFiles"            ← TASKLET (move processed files)
└── Step 5: "sendNotification"        ← TASKLET (send email)
```

**Step 2 — The interface:**

```java
public interface Tasklet {
    RepeatStatus execute(StepContribution contribution, 
                         ChunkContext chunkContext) throws Exception;
}

// Return values:
// RepeatStatus.FINISHED    → task done, move to next step
// RepeatStatus.CONTINUABLE → run this tasklet again (loop)
```

**Step 3 — Common use cases:**

| Use Case | What the Tasklet does |
|----------|----------------------|
| File validation | Check if input file exists before processing |
| Cleanup | Delete temp files after processing |
| Database setup | Run DDL scripts, create indexes |
| Notification | Send email/SMS after job completes |
| External API | Call REST API (e.g., trigger downstream system) |
| Archive | Move processed files to archive directory |
| Count/summary | Count records, store in ExecutionContext |

### 🗣️ How to Explain in Interview

> *"A Tasklet is the simpler of the two step types in Spring Batch. It's for tasks that don't follow the read-process-write pattern — like checking if an input file exists, deleting temporary files, sending a notification, or calling an external API. You implement the execute() method which runs your logic, and return RepeatStatus.FINISHED when done. Tasklets are typically used as setup steps before chunk processing or cleanup steps afterward."*

### 💻 Code Example

```java
// Simple tasklet: cleanup temporary files
@Bean
public Step cleanupStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("cleanup", repo)
            .tasklet((contribution, chunkContext) -> {
                
                Path tempDir = Path.of("/data/temp/batch");
                
                if (Files.exists(tempDir)) {
                    Files.walk(tempDir)
                         .sorted(Comparator.reverseOrder())
                         .forEach(path -> {
                             try { Files.deleteIfExists(path); }
                             catch (IOException e) { /* log warning */ }
                         });
                }
                
                return RepeatStatus.FINISHED;  // Done — move to next step
                
            }, tx)
            .build();
}
```

### ⚡ Key Points to Remember

1. **No read-process-write** — just execute a task
2. Return `FINISHED` → done; `CONTINUABLE` → run again
3. Use for **setup, cleanup, notifications, API calls**
4. Each execution = **one transaction**
5. Can access **JobParameters** and **ExecutionContext**

---

<a id="q80"></a>

## Q80. What is the difference between Tasklet and Chunk processing?

### 🔑 Quick Answer

> **Tasklet** = single task execution (no data records). **Chunk** = read-process-write loop for data records. Use Tasklet for setup/cleanup, Chunk for processing data.

### 📖 Step-by-Step Explanation

| Feature | Tasklet | Chunk-Based |
|---------|---------|-------------|
| **Pattern** | Execute single task | Read → Process → Write loop |
| **Data processing** | ❌ Not designed for it | ✅ Designed for it |
| **Transaction** | One per execute() call | One per chunk |
| **Components** | Just the Tasklet | Reader + Processor + Writer |
| **Repeat** | Optional (via CONTINUABLE) | Repeats until Reader returns null |
| **Skip/Retry** | Not built-in | ✅ Built-in skip/retry |
| **Restartability** | Re-runs from beginning | Resumes from last chunk |
| **Use case** | Cleanup, notifications, setup | Processing data records |

**Decision guide:**

```
Do you need to process DATA RECORDS (CSV, DB rows, messages)?
  → Chunk-based (Reader + Processor + Writer)

Do you need to do ONE TASK (delete file, send email, run SQL)?
  → Tasklet

Can you frame it as "read N items, do something, write N items"?
  → Chunk-based

Is it a single operation that doesn't loop over records?
  → Tasklet
```

### 🗣️ How to Explain in Interview

> *"Tasklet and Chunk are the two step types. The key difference: Chunk is for processing data records — it reads one item at a time, processes it, and writes in batches with built-in skip, retry, and restart from checkpoint. Tasklet is for single tasks that don't process records — like validating files exist, cleaning up temp files, or sending notifications. Chunk gives you all the batch processing features automatically. Tasklet is just 'run this code, return FINISHED.' I use Tasklets for steps before and after the main chunk processing."*

### ⚡ Key Points to Remember

1. **Chunk** = data records; **Tasklet** = single task
2. Chunk has **skip/retry/restart**; Tasklet doesn't
3. Chunk loop = automatic; Tasklet = you control the loop
4. Most jobs = **Tasklet setup + Chunk processing + Tasklet cleanup**

---

<a id="q81"></a>

## Q81. When should you use Tasklet vs Chunk?

### 🔑 Quick Answer

> Use **Tasklet** when there are NO data records to loop over: file operations, DDL scripts, notifications, API calls. Use **Chunk** when you process a collection of records: CSV rows, DB records, messages.

### 📖 Step-by-Step Explanation

**Use Tasklet for:**

| Task | Why Tasklet |
|------|------------|
| ✅ Check if input file exists | One check, boolean result |
| ✅ Delete temporary files | Single operation |
| ✅ Run `ALTER TABLE` or `CREATE INDEX` | DDL operations |
| ✅ Send completion email | One email |
| ✅ Call REST API | Trigger downstream, get status |
| ✅ Count records, set ExecutionContext | Pre-processing |
| ✅ Move files to archive | File operations |

**Use Chunk for:**

| Task | Why Chunk |
|------|----------|
| ✅ Process CSV file → insert to DB | Many records, need chunk transactions |
| ✅ Read DB → transform → write report | Data pipeline |
| ✅ Validate 1M records | Each needs validation logic |
| ✅ Send individual emails per customer | Loop over customer records |
| ✅ Sync data between systems | Record-by-record comparison |

### 🗣️ How to Explain in Interview

> *"The rule is simple: if you're processing a collection of data records, use Chunk. If you're doing a single operation, use Tasklet. Chunk gives you automatic chunking, transactions, skip/retry, and restart. Tasklet is just 'run this code.' In practice, most of my jobs have 1-2 Tasklet steps for setup and cleanup, and 1-3 Chunk steps for the actual data processing in between."*

### ⚡ Key Points to Remember

1. **Records to loop over** → Chunk
2. **Single operation** → Tasklet
3. Typical job: Tasklet → Chunk → Chunk → Tasklet
4. When in doubt, ask: "Am I processing a LIST of things?" → Chunk

---

<a id="q82"></a>

## Q82. Can a Tasklet run multiple times?

### 🔑 Quick Answer

> Yes! Return `RepeatStatus.CONTINUABLE` and Spring Batch will call the Tasklet again. Each call is a **separate transaction**. Return `FINISHED` to stop.

### 📖 Step-by-Step Explanation

**Step 1 — CONTINUABLE = run again:**

```
execute() call 1 → return CONTINUABLE → runs again
execute() call 2 → return CONTINUABLE → runs again
execute() call 3 → return FINISHED    → step done!
```

**Step 2 — Real use case: Batch delete (delete in pages):**

```java
@Bean
public Step archiveStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("archiveOldOrders", repo)
            .tasklet(new Tasklet() {
                private static final int BATCH_SIZE = 1000;
                
                @Override
                public RepeatStatus execute(StepContribution contribution, 
                                           ChunkContext chunkContext) {
                    // Delete 1000 old orders per call
                    int deleted = jdbcTemplate.update(
                        "DELETE FROM orders WHERE created_date < ? LIMIT ?",
                        LocalDate.now().minusYears(2), BATCH_SIZE
                    );
                    
                    if (deleted < BATCH_SIZE) {
                        return RepeatStatus.FINISHED;    // No more to delete
                    }
                    return RepeatStatus.CONTINUABLE;     // More records remain
                }
            }, tx)
            .build();
}
```

**What happens:**
```
Call 1: DELETE 1000 old orders → COMMIT → CONTINUABLE
Call 2: DELETE 1000 old orders → COMMIT → CONTINUABLE
Call 3: DELETE 1000 old orders → COMMIT → CONTINUABLE
...
Call 50: DELETE 700 old orders → COMMIT → FINISHED (only 700 < 1000)
Step done! Deleted 49,700 old orders in 50 transactions.
```

### 🗣️ How to Explain in Interview

> *"Yes, a Tasklet can run multiple times by returning RepeatStatus.CONTINUABLE. Each call is a separate transaction. This is useful for batch operations that shouldn't be done in one giant transaction — like deleting millions of old records. Instead of one DELETE that locks the table for minutes, I delete 1000 per transaction. When the delete returns fewer than 1000 rows, I know there's no more data and return FINISHED. This keeps transactions small and avoids lock contention."*

### ⚡ Key Points to Remember

1. `CONTINUABLE` = run again; `FINISHED` = stop
2. Each call = **separate transaction**
3. Great for **batch deletes** (avoid giant transactions)
4. Implement **termination condition** (otherwise infinite loop!)
5. Each execution is **not restartable** at sub-call level (restarts from beginning)

---

<a id="q83"></a>

## Q83. What is RepeatStatus?

### 🔑 Quick Answer

> RepeatStatus is an **enum** with two values: `FINISHED` (task is done, move to next step) and `CONTINUABLE` (run this tasklet again). It controls whether the Tasklet step repeats.

### 📖 Step-by-Step Explanation

```java
public enum RepeatStatus {
    CONTINUABLE,  // "I'm not done yet — call me again"
    FINISHED      // "I'm done — move to the next step"
}
```

| Value | What happens | When to use |
|-------|-------------|-------------|
| `FINISHED` | Step completes, moves to next step | Task is done (most common) |
| `CONTINUABLE` | execute() is called again | Batch delete, paginated API calls |

**Helper method:**

```java
// RepeatStatus.continueIf(condition)
return RepeatStatus.continueIf(moreRecordsExist);

// Equivalent to:
return moreRecordsExist ? RepeatStatus.CONTINUABLE : RepeatStatus.FINISHED;
```

### 🗣️ How to Explain in Interview

> *"RepeatStatus is a simple enum that controls Tasklet repetition. FINISHED means the task is complete, move to the next step. CONTINUABLE means call this Tasklet again — useful for operations you want to do in multiple transactions, like deleting records in batches. Most Tasklets return FINISHED because they're simple single-operation tasks."*

### ⚡ Key Points to Remember

1. `FINISHED` = done, next step (most common)
2. `CONTINUABLE` = run again (batch operations)
3. `RepeatStatus.continueIf(boolean)` = convenience method
4. **Always have a termination condition** for CONTINUABLE
5. 95% of Tasklets just return `FINISHED`

---

> **🎯 Navigation:** [← Error Handling (Q70-78)](08-error-handling.md) | [Next → Parallel Processing (Q84-91)](10-parallel-processing.md) | [📋 All Sections](README.md)
