# 🔴 Spring Batch — Real Production Scenarios (Q117–Q125)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q117"></a>

## Q117. How would you process a 10GB file using Spring Batch?

### 🔑 Quick Answer

> **Split the file into chunks** (Tasklet) → **Partition** each chunk file to parallel workers → each worker uses **FlatFileItemReader** (streaming, constant memory) + **JdbcBatchItemWriter** (batch inserts). Total: ~10-20 minutes with 10 partitions.

### 📖 Step-by-Step Explanation

**Step 1 — The architecture:**

```
Job: process10GBFile
  │
  ├── Step 1: fileSplitTasklet  (split 10GB → 10 × 1GB files)
  │     input.csv → split_aa, split_ab, ... split_aj
  │
  └── Step 2: masterStep (partitioned)
        ├── Partition 1: FlatFileReader(split_aa) → Writer
        ├── Partition 2: FlatFileReader(split_ab) → Writer
        ├── ...
        └── Partition 10: FlatFileReader(split_aj) → Writer
        All 10 run in PARALLEL → 10× faster
```

**Step 2 — Why this works:**

```
Memory: FlatFileItemReader streams line by line
  → Only current chunk (500 lines) in memory
  → 10GB file needs only ~50MB per partition

Speed: 10 partitions = 10× throughput
  Single thread: ~2 hours
  10 partitions:  ~12 minutes

Reliability: Each partition = separate StepExecution
  → Partition 7 fails? Only partition 7 restarts
  → Other 9 keep their committed data
```

### 💻 Code Example

```java
// Step 1: Split file into manageable chunks
@Bean
public Tasklet fileSplitTasklet() {
    return (contribution, chunkContext) -> {
        // Split using Java (cross-platform)
        Path input = Path.of("/data/input.csv");
        long totalLines = Files.lines(input).count();
        long linesPerFile = totalLines / 10;
        
        // Split into 10 files: split_00.csv through split_09.csv
        // (implementation with BufferedReader/BufferedWriter)
        return RepeatStatus.FINISHED;
    };
}

// Step 2: Partition across split files
@Bean
public MultiResourcePartitioner partitioner() {
    MultiResourcePartitioner partitioner = new MultiResourcePartitioner();
    partitioner.setResources(
            new PathMatchingResourcePatternResolver()
                    .getResources("file:/data/split_*.csv"));
    partitioner.setKeyName("file");
    return partitioner;
}

// Worker reader: Each partition reads its own file
@Bean
@StepScope
public FlatFileItemReader<Record> reader(
        @Value("#{stepExecutionContext['file']}") Resource file) {
    return new FlatFileItemReaderBuilder<Record>()
            .name("partitionReader")
            .resource(file)
            .delimited()
            .names("col1", "col2", "col3", "col4")
            .targetType(Record.class)
            .build();
}

// Master step: Orchestrate 10 partitions
@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("master", repo)
            .partitioner("worker", partitioner())
            .step(workerStep(repo, tx))
            .gridSize(10)
            .taskExecutor(taskExecutor())
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"For a 10GB file, I use a two-step approach. First, a Tasklet splits the file into 10 smaller files — about 1GB each. Second, a partitioned step assigns one file to each of 10 parallel workers. Each worker uses FlatFileItemReader which streams line by line — so only the current chunk of 500 records is in memory, not the entire file. Combined with JdbcBatchItemWriter for batch inserts, this processes 10GB in about 10-20 minutes. Each partition is independently restartable — if one fails, only that partition needs to re-run."*

### ⚡ Key Points to Remember

1. **Split file first** (Tasklet) → then partition
2. **FlatFileItemReader** = streaming (constant memory)
3. **10 partitions** = ~10× faster
4. Each partition **independently restartable**
5. Memory: ~50MB per partition (not 10GB!)

---

<a id="q118"></a>

## Q118. How would you process 100 million database records?

### 🔑 Quick Answer

> **Partition by ID range** (20 partitions) + **JdbcPagingItemReader** (memory-safe) + **JdbcBatchItemWriter** (batch inserts) + chunk size 500. Estimated: 30-60 minutes.

### 📖 Step-by-Step Explanation

**Step 1 — The math:**

```
100,000,000 records ÷ 20 partitions = 5,000,000 per partition
5,000,000 ÷ 500 (chunk size) = 10,000 chunks per partition
20 partitions × 500 records/sec/partition = 10,000 records/sec total
100,000,000 ÷ 10,000 = 10,000 seconds ≈ ~2.8 hours (safe estimate)

With tuned JdbcBatch + indexes: 20,000+ records/sec
100,000,000 ÷ 20,000 ≈ ~83 minutes

With 50 partitions: ~30-40 minutes
```

**Step 2 — Architecture:**

```
Partitioner:
  SELECT MIN(id), MAX(id) FROM records
  → min=1, max=100,000,000
  → 20 partitions:
     partition0:  ID 1 - 5,000,000
     partition1:  ID 5,000,001 - 10,000,000
     ...
     partition19: ID 95,000,001 - 100,000,000

Each Worker:
  JdbcPagingItemReader (WHERE id BETWEEN :min AND :max)
  → Page size 500, chunk size 500
  → Own DB connection from pool
  → JdbcBatchItemWriter (batch INSERT)
```

### 💻 Code Example

```java
@Bean
public Partitioner idRangePartitioner(DataSource ds) {
    return gridSize -> {
        JdbcTemplate jdbc = new JdbcTemplate(ds);
        Long min = jdbc.queryForObject("SELECT MIN(id) FROM records", Long.class);
        Long max = jdbc.queryForObject("SELECT MAX(id) FROM records", Long.class);
        long range = (max - min) / gridSize + 1;
        
        Map<String, ExecutionContext> partitions = new HashMap<>();
        long start = min;
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", start);
            ctx.putLong("maxId", Math.min(start + range - 1, max));
            partitions.put("partition" + i, ctx);
            start += range;
        }
        return partitions;
    };
}

@Bean
@StepScope
public JdbcPagingItemReader<Record> reader(
        DataSource ds,
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<Record>()
            .dataSource(ds)
            .selectClause("SELECT *")
            .fromClause("FROM records")
            .whereClause("WHERE id >= :minId AND id <= :maxId")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .rowMapper(new BeanPropertyRowMapper<>(Record.class))
            .build();
}

@Bean
public Step masterStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("master", repo)
            .partitioner("worker", idRangePartitioner(dataSource))
            .step(workerStep(repo, tx))
            .gridSize(20)                        // 20 parallel partitions
            .taskExecutor(taskExecutor())
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"For 100 million records, I use ID-range partitioning. The Partitioner queries MIN and MAX ID, divides the range into 20 partitions of 5 million each. Each partition has its own JdbcPagingItemReader that reads only its ID range — page by page, so memory stays constant. JdbcBatchItemWriter sends 500 inserts as one batch call. With 20 partitions running in parallel, I get about 10,000-20,000 records per second depending on processing complexity. Each partition is independently restartable — if partition 15 fails, only that 5 million range needs to re-process."*

### ⚡ Key Points to Remember

1. **ID-range partitioning** = divide by MIN/MAX ID
2. **20+ partitions** for 100M+ records
3. **JdbcPagingItemReader** = constant memory
4. Connection pool size ≥ partition count + 2
5. **Index on ID column** (critical for WHERE clause)

---

<a id="q119"></a>

## Q119. How would you handle multiple batch jobs running simultaneously?

### 🔑 Quick Answer

> **Separate thread pools** per job (resource isolation), **limit DB connections** across all jobs (prevent pool exhaustion), ensure **jobs process different data** (no conflicts), use **async JobLauncher** for non-blocking execution.

### 📖 Step-by-Step Explanation

**Step 1 — The four concerns:**

```
Concern 1: RESOURCE CONTENTION
  Problem: Both jobs fight for CPU/memory/DB connections
  Fix: Separate thread pools with limits
  
  importJob:  ThreadPool(8 threads)  → max 8 DB connections
  reportJob:  ThreadPool(4 threads)  → max 4 DB connections
  Total: 12 connections → pool size = 15 (12 + headroom)

Concern 2: DB CONNECTION EXHAUSTION
  Problem: 50 spring.datasource.hikari.maximum-pool-size but 
           importJob(16) + reportJob(8) + billingJob(8) = 32 connections
  Fix: pool size = sum of all job threads + 5 overhead

Concern 3: DATA CONFLICTS
  Problem: Job A writes to orders table while Job B reads from it
  Fix: Jobs should process DIFFERENT data or use isolation

Concern 4: PRIORITY
  Problem: Urgent job needs resources but long-running job has them
  Fix: Job queue with priority, or dedicated executor per priority
```

### 💻 Code Example

```java
// Separate thread pools per job
@Bean("importExecutor")
public TaskExecutor importExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(8);
    exec.setMaxPoolSize(8);
    exec.setThreadNamePrefix("import-");
    return exec;
}

@Bean("reportExecutor")
public TaskExecutor reportExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(4);
    exec.setMaxPoolSize(4);
    exec.setThreadNamePrefix("report-");
    return exec;
}

// Async job launcher — non-blocking
@Bean
public JobLauncher asyncJobLauncher(JobRepository repo) throws Exception {
    TaskExecutorJobLauncher launcher = new TaskExecutorJobLauncher();
    launcher.setJobRepository(repo);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());
    launcher.afterPropertiesSet();
    return launcher;
}
```

### 🗣️ How to Explain in Interview

> *"For concurrent batch jobs, I manage four things. First, separate thread pools — the import job gets 8 threads and the report job gets 4, so they don't steal threads from each other. Second, connection pool sizing — I set HikariCP's maximum pool to the sum of all job threads plus overhead. Third, data isolation — concurrent jobs should process different tables or different data ranges to avoid conflicts. Fourth, I use an async JobLauncher so launching one job doesn't block the launching of another."*

### ⚡ Key Points to Remember

1. **Separate TaskExecutor** per job (thread isolation)
2. **Connection pool** ≥ sum of all job threads + overhead
3. **Different data** per concurrent job (avoid conflicts)
4. **Async JobLauncher** for non-blocking launches
5. Monitor **total resource usage** across all jobs

---

<a id="q120"></a>

## Q120. How would you restart a job after a JVM crash?

### 🔑 Quick Answer

> On crash, the current chunk rolls back (uncommitted). The JobExecution stays in **STARTED** status (stale). On app restart, mark it as **FAILED** via SQL or programmatically, then re-launch with the **same parameters** — Spring Batch resumes from the last committed chunk.

### 📖 Step-by-Step Explanation

**Step 1 — What happens during a crash:**

```
Normal execution:
  Chunk 1: read → process → write → COMMIT ✅ (saved in DB)
  Chunk 2: read → process → write → COMMIT ✅ (saved in DB)
  Chunk 3: read → process → wr—— → 💥 JVM CRASH (not committed)

State after crash:
  BATCH_JOB_EXECUTION:  STATUS = 'STARTED' (stale — JVM is dead)
  BATCH_STEP_EXECUTION: READ_COUNT = 1500, WRITE_COUNT = 1000
                        (only chunks 1-2 committed, chunk 3 lost)
  EXECUTION_CONTEXT:    read.count = 1000 (last committed position)
```

**Step 2 — Recovery process:**

```
1. App restarts
2. Find stale STARTED execution:
   SELECT * FROM BATCH_JOB_EXECUTION WHERE STATUS = 'STARTED'
3. Mark as FAILED:
   UPDATE BATCH_JOB_EXECUTION SET STATUS='FAILED', EXIT_CODE='FAILED',
          END_TIME=NOW() WHERE STATUS='STARTED'
4. Re-launch with same parameters:
   jobLauncher.run(job, sameParams)
5. Spring Batch:
   → Finds FAILED execution for this JobInstance
   → Creates new JobExecution
   → Reads EXECUTION_CONTEXT: read.count = 1000
   → Reader skips first 1000 records
   → Resumes from record 1001 ✅
```

### 💻 Code Example

```java
// Automatic crash recovery on startup
@Component
public class CrashRecoveryRunner implements ApplicationRunner {
    
    @Autowired private JobExplorer jobExplorer;
    @Autowired private JobRepository jobRepository;
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job myJob;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Find stale STARTED executions (crashed jobs)
        Set<JobExecution> staleExecutions = 
                jobExplorer.findRunningJobExecutions("myJob");
        
        for (JobExecution exec : staleExecutions) {
            // Mark as FAILED so it can be restarted
            exec.setStatus(BatchStatus.FAILED);
            exec.setEndTime(LocalDateTime.now());
            exec.setExitStatus(new ExitStatus("FAILED", "JVM crash recovery"));
            jobRepository.update(exec);
            
            // Restart with same parameters → resumes from last checkpoint
            jobLauncher.run(myJob, exec.getJobParameters());
        }
    }
}
```

### 🗣️ How to Explain in Interview

> *"When the JVM crashes, the current chunk's transaction rolls back automatically — that's the database guarantee. But the JobExecution stays in STARTED status because Spring Batch never got to mark it FAILED. On restart, I have an ApplicationRunner that finds stale STARTED executions, marks them FAILED, then re-launches with the same parameters. Spring Batch creates a new execution under the same instance, reads the ExecutionContext from the metadata tables — which has the reader position at the last committed chunk — and resumes from there. So if it crashed during chunk 50 out of 200, it resumes from chunk 49's position."*

### ⚡ Key Points to Remember

1. **Crash** → current chunk rolls back, status stays STARTED
2. **Mark FAILED** → manually via SQL or programmatic ApplicationRunner
3. **Re-launch** with same params → resumes from last checkpoint
4. **ExecutionContext** stored per commit → position is always safe
5. Implement **automatic crash recovery** in ApplicationRunner

---

<a id="q121"></a>

## Q121. How would you process files uploaded by multiple users?

### 🔑 Quick Answer

> Each file = **unique JobInstance** (userId + fileName as identifying parameters). Use **async JobLauncher** so uploads don't block. Each user's job is independent and separately trackable.

### 📖 Step-by-Step Explanation

```
User A uploads orders.csv  → Job(userId=A, file=orders.csv)  → Instance 1
User B uploads returns.csv → Job(userId=B, file=returns.csv) → Instance 2
User A uploads orders2.csv → Job(userId=A, file=orders2.csv) → Instance 3

Each is a SEPARATE JobInstance because params differ.
All can run SIMULTANEOUSLY with async launcher.
```

### 💻 Code Example

```java
@RestController
@RequestMapping("/api/upload")
public class FileUploadController {
    
    @Autowired private JobLauncher asyncJobLauncher;
    @Autowired private Job fileProcessJob;
    
    @PostMapping
    public ResponseEntity<Map<String, Object>> handleUpload(
            @RequestParam MultipartFile file,
            @RequestParam String userId) throws Exception {
        
        // Save file to user-specific directory
        Path userDir = Path.of("/uploads", userId);
        Files.createDirectories(userDir);
        Path filePath = userDir.resolve(file.getOriginalFilename());
        file.transferTo(filePath.toFile());
        
        // Launch job with unique params per user+file
        JobParameters params = new JobParametersBuilder()
                .addString("userId", userId)
                .addString("filePath", filePath.toString())
                .addLong("uploadTime", System.currentTimeMillis())
                .toJobParameters();
        
        JobExecution exec = asyncJobLauncher.run(fileProcessJob, params);
        
        return ResponseEntity.ok(Map.of(
                "jobId", exec.getId(),
                "status", "PROCESSING"));
    }
    
    @GetMapping("/status/{jobId}")
    public ResponseEntity<Map<String, Object>> checkStatus(
            @PathVariable Long jobId) {
        JobExecution exec = jobExplorer.getJobExecution(jobId);
        return ResponseEntity.ok(Map.of(
                "status", exec.getStatus().toString(),
                "readCount", exec.getStepExecutions().stream()
                        .mapToLong(StepExecution::getReadCount).sum()));
    }
}
```

### 🗣️ How to Explain in Interview

> *"Each user upload becomes a unique batch job. I use userId, fileName, and uploadTime as identifying parameters — so each upload is a separate JobInstance. The upload endpoint saves the file and launches the batch job asynchronously so the API responds immediately with a job ID. Users can poll the status endpoint to check progress. Jobs run independently — User A's import doesn't affect User B's. For resource control, I limit the async executor to prevent too many simultaneous imports from overwhelming the system."*

### ⚡ Key Points to Remember

1. **Unique params** per user+file → separate JobInstance
2. **Async launcher** → API responds immediately
3. **Status endpoint** → users can check progress
4. **Limit concurrent jobs** → prevent resource exhaustion
5. User-specific directories for file isolation

---

<a id="q122"></a>

## Q122. How would you avoid duplicate processing?

### 🔑 Quick Answer

> Three strategies: (1) **JobParameters uniqueness** — same params can't re-run a completed instance, (2) **Idempotent writer** — UPSERT instead of INSERT, (3) **Processed flag** — mark records as processed in the source table.

### 📖 Step-by-Step Explanation

**Step 1 — Three defense layers:**

```
Layer 1: JOB-LEVEL (Spring Batch built-in)
  Same job + same identifying params = same instance
  If instance already COMPLETED → rejects new launch
  → Prevents re-running the same job

Layer 2: WRITER-LEVEL (idempotent operations)
  UPSERT instead of INSERT
  INSERT ... ON DUPLICATE KEY UPDATE ...
  → Running twice produces same result

Layer 3: SOURCE-LEVEL (processed flag)
  Add 'processed' column to source table
  Reader: WHERE processed = false
  Writer: UPDATE source SET processed = true
  → Records only processed once
```

### 💻 Code Example

```java
// Layer 2: Idempotent writer (UPSERT)
@Bean
public JdbcBatchItemWriter<Employee> idempotentWriter(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .sql("""
                INSERT INTO employees (id, name, salary, updated_at)
                VALUES (:id, :name, :salary, NOW())
                ON DUPLICATE KEY UPDATE 
                    name = :name, salary = :salary, updated_at = NOW()
            """)
            .dataSource(ds)
            .beanMapped()
            .build();
}

// Layer 3: Processed flag in reader + composite writer
@Bean
@StepScope
public JdbcPagingItemReader<Record> reader(DataSource ds) {
    return new JdbcPagingItemReaderBuilder<Record>()
            .selectClause("SELECT *")
            .fromClause("FROM source_records")
            .whereClause("WHERE processed = false")  // Only unprocessed
            .sortKeys(Map.of("id", Order.ASCENDING))
            .build();
}

@Bean
public CompositeItemWriter<Record> compositeWriter() {
    CompositeItemWriter<Record> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(
            targetWriter(),         // Write to destination
            markProcessedWriter()   // Mark source as processed
    ));
    return writer;
}
```

### 🗣️ How to Explain in Interview

> *"I prevent duplicates at three levels. First, Spring Batch's built-in uniqueness — same job parameters can't re-run a completed instance. Second, idempotent writes — I use UPSERT (INSERT ON DUPLICATE KEY UPDATE) so if a record is processed twice, the result is the same. Third, a processed flag on the source table — the reader queries WHERE processed=false, and after writing, a composite writer marks the source record as processed=true. These three layers ensure that even in failure and restart scenarios, no record gets duplicated."*

### ⚡ Key Points to Remember

1. **JobParameters uniqueness** = built-in duplicate prevention
2. **UPSERT** = idempotent writes (same result if run twice)
3. **Processed flag** = source-level tracking
4. **CompositeItemWriter** = write to target + mark source
5. All three layers together = bulletproof deduplication

---

<a id="q123"></a>

## Q123. How would you track failed records?

### 🔑 Quick Answer

> Implement a **SkipListener** that logs each skipped record to a **failed_records** error table with the phase (read/process/write), the record data, the exception, and a timestamp.

### 📖 Step-by-Step Explanation

**Step 1 — What to capture:**

```
For each failed record, store:
  - PHASE: where it failed (READ, PROCESS, WRITE)
  - RECORD_ID: identifier of the failed record
  - RECORD_DATA: the actual data (for reprocessing)
  - ERROR_MESSAGE: what went wrong
  - JOB_EXECUTION_ID: which job run
  - TIMESTAMP: when it failed

This creates an "error audit trail" for:
  1. Root cause analysis
  2. Manual reprocessing
  3. Data quality reporting
  4. Compliance/audit
```

**Step 2 — Error table schema:**

```sql
CREATE TABLE failed_records (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_execution_id BIGINT,
    step_name VARCHAR(100),
    phase VARCHAR(10),        -- READ, PROCESS, WRITE
    record_id VARCHAR(100),
    record_data TEXT,
    error_message TEXT,
    error_class VARCHAR(200),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 💻 Code Example

```java
// SkipListener — captures every skipped record
@Component
public class FailedRecordTracker implements SkipListener<Record, Record> {
    
    @Autowired private JdbcTemplate jdbc;
    
    @Override
    public void onSkipInRead(Throwable t) {
        jdbc.update(
            "INSERT INTO failed_records(phase, error_message, error_class, created_at) "
            + "VALUES (?, ?, ?, ?)",
            "READ", t.getMessage(), t.getClass().getName(), LocalDateTime.now());
    }
    
    @Override
    public void onSkipInProcess(Record item, Throwable t) {
        jdbc.update(
            "INSERT INTO failed_records(phase, record_id, record_data, "
            + "error_message, error_class, created_at) VALUES (?, ?, ?, ?, ?, ?)",
            "PROCESS", String.valueOf(item.getId()), item.toString(),
            t.getMessage(), t.getClass().getName(), LocalDateTime.now());
    }
    
    @Override
    public void onSkipInWrite(Record item, Throwable t) {
        jdbc.update(
            "INSERT INTO failed_records(phase, record_id, record_data, "
            + "error_message, error_class, created_at) VALUES (?, ?, ?, ?, ?, ?)",
            "WRITE", String.valueOf(item.getId()), item.toString(),
            t.getMessage(), t.getClass().getName(), LocalDateTime.now());
    }
}

// Register in step
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Record, Record>chunk(500, tx)
            .reader(reader()).processor(processor()).writer(writer())
            .faultTolerant()
            .skip(Exception.class).skipLimit(1000)
            .listener(failedRecordTracker())          // Register tracker
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"I track failed records with a SkipListener that writes to an error table. The listener has three methods: onSkipInRead, onSkipInProcess, and onSkipInWrite. Each logs the phase, the record ID and data, the exception message, and a timestamp. After the job completes, I can query the error table to see exactly which records failed, during which phase, and why. This serves dual purpose — operations can investigate and fix specific records, and we can build a reprocessing job that reads from the error table and retries failed records."*

### ⚡ Key Points to Remember

1. **SkipListener** = three callbacks (read, process, write)
2. Log **everything**: phase, record, exception, timestamp
3. **Error table** = audit trail for failed records
4. Enable with `.faultTolerant().skip().listener()`
5. Build **reprocessing job** that reads from error table

---

<a id="q124"></a>

## Q124. How would you cancel a running batch job?

### 🔑 Quick Answer

> Use **JobOperator.stop(executionId)** — it sets a flag, and the job stops **after the current chunk completes**. Status goes STARTED → STOPPING → STOPPED. A stopped job can be **restarted** later.

### 📖 Step-by-Step Explanation

**Step 1 — How stopping works:**

```
                           stop() called here
                                 ↓
Chunk 1 ✅ → Chunk 2 ✅ → Chunk 3 [processing...] → Chunk 3 ✅ → STOPPED
                                 ↑                        ↑
                           Current chunk finishes    THEN job stops
                           (NOT killed mid-chunk)    (clean shutdown)

Status flow: STARTED → STOPPING → STOPPED
  STOPPING = "stop requested, waiting for current chunk"
  STOPPED  = "cleanly stopped, can restart later"
```

**Step 2 — Key behaviors:**

```
⚠️ stop() does NOT kill the job immediately!
  → Current chunk finishes (data integrity maintained)
  → Transactions are committed or rolled back cleanly
  → ExecutionContext is saved (for restart)

✅ STOPPED job CAN be restarted:
  → jobLauncher.run(job, sameParams)
  → Resumes from where it stopped

❌ If you need IMMEDIATE stop (rare):
  → Use setTerminateOnly() on StepExecution
  → Stops after current item (not chunk)
```

### 💻 Code Example

```java
// REST endpoint to stop a job
@RestController
@RequestMapping("/api/jobs")
public class JobControlController {
    
    @Autowired private JobOperator jobOperator;
    
    @PostMapping("/{executionId}/stop")
    public ResponseEntity<String> stopJob(@PathVariable Long executionId) 
            throws Exception {
        jobOperator.stop(executionId);
        return ResponseEntity.ok("Job stopping after current chunk completes...");
    }
    
    @PostMapping("/{executionId}/restart")
    public ResponseEntity<String> restartJob(@PathVariable Long executionId) 
            throws Exception {
        Long newExecId = jobOperator.restart(executionId);
        return ResponseEntity.ok("Job restarted with execution ID: " + newExecId);
    }
}

// Programmatic stop via ChunkListener (conditional)
@Component
public class ConditionalStopListener implements ChunkListener {
    
    @Override
    public void afterChunk(ChunkContext context) {
        // Check external condition (DB flag, config, time limit)
        if (shouldStop()) {
            context.getStepContext().getStepExecution()
                    .setTerminateOnly();  // Stop after this chunk
        }
    }
    
    private boolean shouldStop() {
        // Check DB flag, time limit, external signal, etc.
        return false;
    }
}
```

### 🗣️ How to Explain in Interview

> *"I use JobOperator.stop() which sets a flag for graceful shutdown. The current chunk finishes processing — we don't kill it mid-chunk because that would leave data in an inconsistent state. After the chunk commits, the job transitions from STARTED to STOPPING to STOPPED. The job can be restarted later with the same parameters — it resumes from where it stopped. I expose this through a REST endpoint so operations teams can stop jobs on demand. For conditional stops — like time limits — I use a ChunkListener that checks a condition after each chunk and calls setTerminateOnly()."*

### ⚡ Key Points to Remember

1. **JobOperator.stop()** = graceful (waits for current chunk)
2. Status: STARTED → **STOPPING** → **STOPPED**
3. STOPPED jobs **can be restarted** (resumes from checkpoint)
4. **NOT immediate** — current chunk always completes
5. For conditional stops → **ChunkListener** + setTerminateOnly()

---

<a id="q125"></a>

## Q125. How would you limit batch job execution time?

### 🔑 Quick Answer

> Two levels: **Transaction timeout** per chunk (prevents individual chunk from hanging) and **Job-level timeout** (scheduled stop after total time limit using JobOperator).

### 📖 Step-by-Step Explanation

**Step 1 — Chunk-level timeout:**

```
Problem: One chunk hangs forever (DB lock, network timeout)
Fix: Transaction timeout per chunk

  Chunk 1: 2 sec ✅
  Chunk 2: 2 sec ✅
  Chunk 3: ...waiting for DB lock... → 60 sec → TIMEOUT! → Rollback + Retry/Fail
```

**Step 2 — Job-level timeout:**

```
Problem: Job must complete within SLA (e.g., 2 hours)
Fix: Scheduled stop after deadline

  Job starts: 02:00 AM
  Deadline: 04:00 AM (2 hours)
  If still running at 04:00 AM → JobOperator.stop()
  Job transitions to STOPPED → can restart in next window
```

### 💻 Code Example

```java
// Level 1: Transaction timeout per chunk (60 seconds)
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    DefaultTransactionAttribute txAttr = new DefaultTransactionAttribute();
    txAttr.setTimeout(60);  // 60 seconds per chunk transaction
    
    return new StepBuilder("step", repo)
            .<Record, Record>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .transactionAttribute(txAttr)    // Chunk timeout
            .build();
}

// Level 2: Job-level timeout
@Component
public class JobTimeoutManager {
    
    @Autowired private JobOperator jobOperator;
    
    private final ScheduledExecutorService scheduler = 
            Executors.newSingleThreadScheduledExecutor();
    
    public JobExecution launchWithTimeout(JobLauncher launcher, Job job,
                                          JobParameters params,
                                          Duration timeout) throws Exception {
        JobExecution exec = launcher.run(job, params);
        
        // Schedule stop if job exceeds timeout
        scheduler.schedule(() -> {
            try {
                if (exec.isRunning()) {
                    jobOperator.stop(exec.getId());
                    log.warn("Job {} timed out after {}",
                            exec.getJobInstance().getJobName(), timeout);
                }
            } catch (Exception e) {
                log.error("Failed to stop timed-out job", e);
            }
        }, timeout.toMillis(), TimeUnit.MILLISECONDS);
        
        return exec;
    }
}

// Usage
jobTimeoutManager.launchWithTimeout(
    jobLauncher, importJob, params, Duration.ofHours(2));
```

### 🗣️ How to Explain in Interview

> *"I set timeouts at two levels. First, per-chunk transaction timeout — I set transactionAttribute with a 60-second timeout on the step. If any single chunk hangs longer than 60 seconds — say waiting for a DB lock — the transaction times out and rolls back. Second, job-level timeout — I schedule a stop using JobOperator after the SLA deadline. If the job is still running after 2 hours, it gets stopped gracefully after the current chunk. It can be restarted in the next processing window. The combination ensures no chunk hangs forever and no job exceeds its time boundary."*

### ⚡ Key Points to Remember

1. **Chunk timeout** = DefaultTransactionAttribute.setTimeout(seconds)
2. **Job timeout** = scheduled JobOperator.stop() after deadline
3. Chunk timeout **rolls back** the hanging chunk
4. Job timeout **stops gracefully** (current chunk finishes)
5. Stopped jobs **can be restarted** later

---

> **🎯 Navigation:** [← Database & Metadata (Q110-116)](14-database-metadata.md) | [📋 All Sections](README.md)
