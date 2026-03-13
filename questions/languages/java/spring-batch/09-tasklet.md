<![CDATA[<div align="center">

# 🟢 Spring Batch — Tasklet Questions (79-83)

[![Questions](https://img.shields.io/badge/Questions-5-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Easy-green.svg)](#)

</div>

---

<a id="q1"></a>
## Q79. ❓ What is Tasklet in Spring Batch?

🔖 **Tags:** `#spring-batch` `#tasklet` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

A **Tasklet** is a simple step that executes a **single task** (not chunk-based read/process/write).

```java
public interface Tasklet {
    RepeatStatus execute(StepContribution contribution, 
                         ChunkContext chunkContext) throws Exception;
}
```

```java
@Bean
public Step cleanupStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("cleanupStep", repo)
            .tasklet((contribution, chunkContext) -> {
                // Delete temp files
                Files.deleteIfExists(Path.of("/tmp/batch-temp.csv"));
                log.info("Cleanup completed!");
                return RepeatStatus.FINISHED;
            }, tx)
            .build();
}
```

### Common Tasklet Use Cases:
| Use Case | Example |
|----------|---------|
| File cleanup | Delete temp files after processing |
| Send notification | Email/SMS after job completes |
| Create directories | Setup output folders |
| Run SQL script | Truncate staging table |
| Call REST API | Trigger external system |
| Archive files | Move processed files to archive |

---

<a id="q2"></a>
## Q80. ❓ What is the difference between Tasklet and Chunk processing?

🔖 **Tags:** `#spring-batch` `#tasklet-vs-chunk` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🟢 Easy  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

| Feature | Tasklet | Chunk Processing |
|---------|---------|-----------------|
| **Pattern** | Single task execution | Read → Process → Write loop |
| **Data** | No streaming data | Processes items in chunks |
| **Transaction** | 1 transaction per execute() | 1 transaction per chunk |
| **Use case** | Simple one-off tasks | Processing large datasets |
| **Components** | Just Tasklet interface | Reader + Processor + Writer |
| **Restartability** | Runs from beginning | Resumes from last chunk |
| **Example** | Delete files, send email | Process 1M DB records |

```
Job: "dailyETL"
├── Step 1: Tasklet      → Create output directory
├── Step 2: Chunk-based  → Read CSV → Transform → Write to DB (main work!)
├── Step 3: Tasklet      → Generate summary report
└── Step 4: Tasklet      → Send email notification
```

### 📌 Key Takeaway
> 💡 Use **Chunk** for processing data. Use **Tasklet** for everything else (setup, cleanup, notifications).

---

<a id="q3"></a>
## Q81. ❓ When should you use Tasklet?

🔖 **Tags:** `#spring-batch` `#tasklet` `#when-to-use`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

Use Tasklet when you **DON'T** need read-process-write pattern:

| ✅ Use Tasklet | ❌ Don't Use Tasklet |
|---------------|---------------------|
| Delete/move files | Process CSV records |
| Run SQL DDL/DML | Transform DB data |
| Send notifications | Bulk read + write |
| Check preconditions | Anything with large datasets |
| Call external APIs | Streaming data processing |
| Generate static reports | ETL operations |

---

<a id="q4"></a>
## Q82. ❓ Can Tasklet run multiple times?

🔖 **Tags:** `#spring-batch` `#tasklet` `#repeat`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**Yes!** Based on the `RepeatStatus` return value:

```java
public enum RepeatStatus {
    CONTINUABLE,  // Run execute() again
    FINISHED      // Stop, step is done
}
```

```java
// Runs ONCE:
.tasklet((contribution, chunkContext) -> {
    doSomething();
    return RepeatStatus.FINISHED;  // Done, won't run again
}, tx)

// Runs MULTIPLE times:
int[] counter = {0};
.tasklet((contribution, chunkContext) -> {
    processPage(counter[0]++);
    if (counter[0] >= totalPages) {
        return RepeatStatus.FINISHED;     // Done after all pages
    }
    return RepeatStatus.CONTINUABLE;      // Run again for next page
}, tx)
```

> Each `CONTINUABLE` execution is a **separate transaction**.

---

<a id="q5"></a>
## Q83. ❓ What is RepeatStatus?

🔖 **Tags:** `#spring-batch` `#repeat-status` `#tasklet`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

| Value | Meaning | Behavior |
|-------|---------|----------|
| `RepeatStatus.FINISHED` | Task is complete | Step moves to next step |
| `RepeatStatus.CONTINUABLE` | Task needs to run again | execute() called again in new transaction |

```java
// Practical example: Process in batches using Tasklet
@Bean
public Tasklet batchDeleteTasklet(JdbcTemplate jdbc) {
    return (contribution, chunkContext) -> {
        int deleted = jdbc.update(
            "DELETE FROM expired_records WHERE created_at < ? LIMIT 1000",
            LocalDate.now().minusDays(90)
        );
        
        if (deleted == 0) {
            return RepeatStatus.FINISHED;     // No more records to delete
        }
        return RepeatStatus.CONTINUABLE;      // More records exist, run again
    };
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Error Handling](./08-error-handling.md) | [Next: Parallel Processing →](./10-parallel-processing.md)
]]>