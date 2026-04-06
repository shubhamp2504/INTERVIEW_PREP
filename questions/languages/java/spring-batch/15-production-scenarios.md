# 🔴 Real Production Scenarios — Q117 to Q125

---

## Q117. How would you process a 10GB file using Spring Batch?

### 📝 One-Liner
Split the file into smaller chunks (Tasklet) → partition them for parallel processing → FlatFileItemReader streams each chunk with constant memory.

### 🔑 Quick Answer
Two-step architecture: **(Step 1)** Tasklet splits 10GB file into 10 × 1GB files. **(Step 2)** Master step partitions the 10 files to 10 parallel workers, each using FlatFileItemReader (streaming, constant memory ~50MB per partition) + JdbcBatchItemWriter. Total time: ~10-20 minutes with 10 partitions. Each partition restartable independently. FlatFileItemReader streams line-by-line — never loads entire file into memory. *(Pehle file todo, phir har piece ko parallel mein process karo — memory constant rehti hai)*

### 📖 How It Works
```
Architecture:

Step 1: File Split (Tasklet)
  10GB file → Split into 10 files of ~1GB each
  /data/input_001.csv (1GB)
  /data/input_002.csv (1GB)
  ...
  /data/input_010.csv (1GB)

Step 2: Parallel Processing (Partitioned)
  Master Step
  ├── Partition 1: input_001.csv → FlatFileItemReader → process → write
  ├── Partition 2: input_002.csv → FlatFileItemReader → process → write
  ├── ...
  └── Partition 10: input_010.csv → FlatFileItemReader → process → write

Memory Usage:
  Each partition: ~50MB (reader buffer + chunk in memory)
  Total: 10 × 50MB = 500MB (NOT 10GB!)

Performance:
  Single thread: ~60 minutes
  10 partitions: ~10 minutes (6× faster, limited by DB write speed)
```

### 🗣️ Answering Approach
"For a 10GB file, I use a two-step approach. First, a Tasklet step splits the file into 10 equal chunks of approximately 1GB each. Second, a master step uses MultiResourcePartitioner to assign each file chunk to a parallel worker. Each worker uses FlatFileItemReader which streams line-by-line with constant memory — about 50MB per partition regardless of file size. Combined with JdbcBatchItemWriter for batch inserts, 10 partitions process the file in about 10 minutes. Each partition is independently restartable — if partition 7 fails, restart only re-reads that file chunk."

### 💻 Code
```java
// Step 1: File Split Tasklet
@Component
@StepScope
public class FileSplitTasklet implements Tasklet {
    @Value("#{jobParameters['inputFile']}")
    private String inputFile;

    @Override
    public RepeatStatus execute(StepContribution contribution, ChunkContext context) throws Exception {
        Path source = Path.of(inputFile);
        long totalLines = Files.lines(source).count();
        long linesPerFile = totalLines / 10;
        
        try (BufferedReader reader = Files.newBufferedReader(source)) {
            for (int i = 0; i < 10; i++) {
                Path output = Path.of("/data/split/chunk_" + i + ".csv");
                try (BufferedWriter writer = Files.newBufferedWriter(output)) {
                    for (long j = 0; j < linesPerFile && reader.ready(); j++) {
                        writer.write(reader.readLine());
                        writer.newLine();
                    }
                }
            }
        }
        return RepeatStatus.FINISHED;
    }
}

// Step 2: Partitioned processing
@Bean
public Step masterStep(JobRepository repo, Step workerStep) {
    MultiResourcePartitioner partitioner = new MultiResourcePartitioner();
    partitioner.setResources(resourcePatternResolver.getResources("file:/data/split/chunk_*.csv"));
    
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", partitioner)
            .step(workerStep)
            .gridSize(10)
            .taskExecutor(taskExecutor())
            .build();
}

@Bean
@StepScope
public FlatFileItemReader<Transaction> fileReader(
        @Value("#{stepExecutionContext['fileName']}") Resource file) {
    return new FlatFileItemReaderBuilder<Transaction>()
            .name("chunkReader")
            .resource(file)
            .delimited().delimiter(",")
            .names("id", "amount", "date", "status")
            .targetType(Transaction.class)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- FlatFileItemReader is NOT thread-safe — must use @StepScope for per-partition instances *(FlatFileItemReader thread-safe nahi hai — StepScope zaruri)*
- File split must handle header rows (don't split headers into multiple files)
- Disk I/O can be bottleneck — use SSD for temp files
- 10GB file + 10 split files = 20GB disk needed temporarily

### ⚡ Remember
- **Split first → then partition** *(pehle todo, phir parallel process karo)*
- FlatFileItemReader = streaming, constant memory (~50MB)
- MultiResourcePartitioner assigns files to workers
- 10 partitions = 10× faster (limited by DB)
- Each partition independently restartable

### 🔗 Follow-ups
- [Q118 → Process 100M database records](#q118)
- [Q86 → Partitioning details](#q86)
- [Q92 → Processing millions efficiently](#q92)

---

## Q118. How would you process 100 million database records?

### 📝 One-Liner
ID-range partitioning (20 partitions) + JdbcPagingItemReader + JdbcBatchItemWriter + chunk 500 = ~30-60 minutes.

### 🔑 Quick Answer
**ID-range partitioning**: Partitioner queries MIN/MAX ID, divides into 20 non-overlapping ranges. Each partition gets its own JdbcPagingItemReader scoped to its ID range via @StepScope. 20 partitions × 500 records/sec per partition = 10,000 records/sec total. 100M ÷ 10K/sec ≈ 10,000 sec ≈ ~3 hours. With batch inserts + tuning: ~30-60 minutes. Critical: index on ID column, connection pool ≥ partitions + 2. *(100 million records ko 20 ID ranges mein baanto — har range parallel process karo)*

### 📖 How It Works
```
Architecture:

Partitioner:
  SELECT MIN(id), MAX(id) FROM orders → 1, 100000000
  Range per partition: 100M / 20 = 5M records each
  
  Partition 1:  IDs 1 - 5,000,000
  Partition 2:  IDs 5,000,001 - 10,000,000
  ...
  Partition 20: IDs 95,000,001 - 100,000,000

Each Partition:
  JdbcPagingItemReader (WHERE id BETWEEN :min AND :max)
  → Processor
  → JdbcBatchItemWriter (batch INSERT)
  
  500 records/sec × 20 partitions = 10,000 total/sec

Math:
  100,000,000 records ÷ 10,000/sec = 10,000 sec ≈ 2.8 hours (baseline)
  With optimized inserts + tuning: ~30-60 minutes

Infrastructure:
  Thread pool: 20 threads
  Connection pool: 22+ (20 partitions + master + monitoring)
  Heap: 4-8 GB
  Index: ON orders(id) — CRITICAL
```

### 🗣️ Answering Approach
"For 100 million records, I use ID-range partitioning with 20 partitions. The Partitioner queries MIN and MAX IDs, divides into 20 non-overlapping ranges of 5 million each. Each partition uses a StepScope JdbcPagingItemReader with WHERE id BETWEEN :minId AND :maxId, combined with JdbcBatchItemWriter for batch inserts. With 20 partitions running at 500 records per second each, we get 10,000 records per second total. The critical infrastructure requirements are an index on the ID column, a connection pool of at least 22, and 4-8 GB heap. In my project, this approach processes 100M payment records in about 45 minutes."

### 💻 Code
```java
@Component
public class IdRangePartitioner implements Partitioner {
    @Autowired private JdbcTemplate jdbc;

    @Override
    public Map<String, ExecutionContext> partition(int gridSize) {
        Long min = jdbc.queryForObject("SELECT MIN(id) FROM orders", Long.class);
        Long max = jdbc.queryForObject("SELECT MAX(id) FROM orders", Long.class);
        long range = (max - min) / gridSize + 1;

        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (i * range));
            ctx.putLong("maxId", Math.min(min + ((i + 1) * range) - 1, max));
            partitions.put("partition" + i, ctx);
        }
        return partitions;
    }
}

@Bean
public Step masterStep(JobRepository repo, Step workerStep) {
    return new StepBuilder("masterStep", repo)
            .partitioner("workerStep", idRangePartitioner)
            .step(workerStep)
            .gridSize(20)
            .taskExecutor(taskExecutor())  // 20 threads
            .build();
}

@Bean
@StepScope
public JdbcPagingItemReader<Order> orderReader(
        @Value("#{stepExecutionContext['minId']}") Long minId,
        @Value("#{stepExecutionContext['maxId']}") Long maxId) {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("orderReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, amount, status")
            .fromClause("FROM orders")
            .whereClause("WHERE id BETWEEN :minId AND :maxId")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .parameterValues(Map.of("minId", minId, "maxId", maxId))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- **Index on ID column is critical** — without it, each partition does full table scan *(index nahi hai toh full table scan — bahut slow)*
- Connection pool must be ≥ partitions + 2 (worker threads + master + monitoring)
- Data skew: if IDs are not evenly distributed, some partitions have more records
- Consider separate read and write datasources for load isolation

### ⚡ Remember
- ID-range partitioning with 20+ partitions *(ID ranges mein baanto — 20 partitions)*
- JdbcPagingItemReader (constant memory, thread-safe)
- Connection pool ≥ partitions + 2
- **Index on partition column is CRITICAL**
- ~30-60 minutes for 100M records (with tuning)

### 🔗 Follow-ups
- [Q117 → Process 10GB file](#q117)
- [Q86 → Partitioning details](#q86)
- [Q92 → Processing millions efficiently](#q92)

---

## Q119. How would you handle multiple batch jobs running simultaneously?

### 📝 One-Liner
Separate thread pools per job, ensure DB connection pool handles all threads, process different data per job, and use async JobLauncher.

### 🔑 Quick Answer
Four concerns: **(1) Resource contention** — give each job its own TaskExecutor (separate thread pools) so they don't steal threads from each other. **(2) DB connection pool** — total pool size ≥ sum of all job threads + overhead. **(3) Data isolation** — jobs should process different data (different tables or WHERE clauses) to avoid conflicts. **(4) Async JobLauncher** — use `SimpleAsyncTaskExecutor` on JobLauncher so launching one job doesn't block launching another. *(Har job ko apna thread pool do — connection pool sabke liye enough hona chahiye)*

### 📖 How It Works
```
Multiple Concurrent Jobs:

Job A (Import):       Thread Pool A (8 threads)  → 8 DB connections
Job B (Report):       Thread Pool B (4 threads)  → 4 DB connections
Job C (Notification): Thread Pool C (2 threads)  → 2 DB connections
                                                    ──────────────
                                           Total:    14 connections
                                           + master: 3  (1 per job)
                                           + buffer: 3
                                           ──────────────
                              Connection Pool Size:   20

Isolation:
  Job A: reads/writes orders table
  Job B: reads orders, writes reports table
  Job C: reads notifications table
  → No write conflicts

Async Launcher:
  POST /launch/importJob  → returns immediately (async)
  POST /launch/reportJob  → returns immediately (async)
  → Both jobs run in parallel
```

### 🗣️ Answering Approach
"For multiple concurrent batch jobs, I address four concerns. First, each job gets its own TaskExecutor with a dedicated thread pool, preventing one job from starving another. Second, the database connection pool must be large enough for all concurrent threads plus overhead — in our case, 20 connections for 14 worker threads plus buffers. Third, jobs process different data to avoid write conflicts. Fourth, I use an async JobLauncher so launching one job doesn't block launching others. In my project, we run three concurrent jobs — import, reporting, and notification — each with its own resources, totaling about 14 worker threads."

### 💻 Code
```java
// Separate thread pools per job
@Bean("importExecutor")
public TaskExecutor importExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(8);
    exec.setMaxPoolSize(8);
    exec.setThreadNamePrefix("import-");
    exec.initialize();
    return exec;
}

@Bean("reportExecutor")
public TaskExecutor reportExecutor() {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(4);
    exec.setMaxPoolSize(4);
    exec.setThreadNamePrefix("report-");
    exec.initialize();
    return exec;
}

// Async JobLauncher
@Bean
public JobLauncher asyncJobLauncher(JobRepository repo) throws Exception {
    TaskExecutorJobLauncher launcher = new TaskExecutorJobLauncher();
    launcher.setJobRepository(repo);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());
    launcher.afterPropertiesSet();
    return launcher;
}

// Connection pool — must handle all concurrent threads
// spring:
//   datasource:
//     hikari:
//       maximum-pool-size: 20   # 8 + 4 + 2 + 6 buffer
//       minimum-idle: 10
```

### ⚠️ Pitfalls / Gotchas
- Shared connection pool too small → threads wait → job hangs *(pool chhota hai toh threads wait karenge — job hang lagega)*
- Two jobs writing to same table → deadlocks or constraint violations
- Async launcher: can't get job result synchronously — need status polling
- Monitor total thread count — too many concurrent jobs can exhaust CPU

### ⚡ Remember
- **Separate TaskExecutor per job** *(har job ko apna thread pool)*
- Connection pool ≥ total threads + overhead
- Different data per concurrent job (avoid conflicts)
- Async JobLauncher for non-blocking launch
- Monitor total resource usage

### 🔗 Follow-ups
- [Q99 → Scheduling options](#q99)
- [Q84 → Parallel processing options](#q84)
- [Q124 → Cancel running job](#q124)

---

## Q120. How would you restart a job after a JVM crash?

### 📝 One-Liner
Crash → current chunk rolls back, status stays STARTED (stale). Recovery: detect stale STARTED executions → mark FAILED → re-launch with same params → resumes from last committed chunk.

### 🔑 Quick Answer
After JVM crash: **(1)** Committed chunks are safe in DB. **(2)** Current in-progress chunk rolls back (transaction). **(3)** Status stays STARTED (not FAILED — because afterJob() never ran). **(4)** On restart: detect stale STARTED executions (START_TIME old, no heartbeat), mark them FAILED via `jobOperator.abandon()` or direct UPDATE. **(5)** Re-launch with same parameters → framework reads ExecutionContext → resumes from last committed position. Implement automatic crash recovery in `ApplicationRunner`. *(Crash hone pe status STARTED rehta hai — FAILED mark karo phir restart karo)*

### 📖 How It Works
```
Crash Timeline:

Running:
  Chunk 1: read → process → write → COMMIT ✅ (safe in DB)
  Chunk 2: read → process → write → COMMIT ✅ (safe in DB)
  ...
  Chunk 50: read → process → write → COMMIT ✅ (safe in DB)
  Chunk 51: read → process → ☠️ JVM CRASH
  
  DB State:
    JOB_EXECUTION.STATUS = STARTED (stale — afterJob never ran)
    STEP_EXECUTION_CONTEXT = {read.count: 25000} (chunk 50's position)
    Current chunk 51 transaction = ROLLED BACK

Recovery:
  1. App restarts
  2. ApplicationRunner detects stale STARTED executions
     (START_TIME > 1 hour ago, still STARTED)
  3. Marks them FAILED
  4. Re-launches with same JobParameters
  5. Framework reads context: {read.count: 25000}
  6. Reader skips to record 25001
  7. Processing resumes from chunk 51 ✅
```

### 🗣️ Answering Approach
"After a JVM crash, committed chunks are safe but the current chunk's transaction rolls back, and the job status stays STARTED because the afterJob callback never executed. On restart, I have an ApplicationRunner that detects stale STARTED executions — those where the start time is more than an hour ago but status is still STARTED. It marks them as FAILED, then re-launches with the same parameters. The framework reads the last committed position from the step execution context and resumes from that point. In my project, a crash at record 25,000 of 100,000 lost only the current chunk of 500 records — restart continued from 25,001."

### 💻 Code
```java
// Automatic crash recovery on startup
@Component
public class CrashRecoveryRunner implements ApplicationRunner {
    @Autowired private JobExplorer jobExplorer;
    @Autowired private JobRepository jobRepository;
    @Autowired private JobLauncher jobLauncher;
    @Autowired private ApplicationContext context;

    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Find all job names
        for (String jobName : jobExplorer.getJobNames()) {
            // Find executions stuck in STARTED (stale after crash)
            Set<JobExecution> running = jobExplorer.findRunningJobExecutions(jobName);
            
            for (JobExecution execution : running) {
                // Check if truly stale (started > 1 hour ago)
                if (execution.getStartTime().isBefore(Instant.now().minus(Duration.ofHours(1)))) {
                    log.warn("Found stale execution: job={}, id={}, started={}",
                            jobName, execution.getId(), execution.getStartTime());

                    // Mark as FAILED so it can be restarted
                    execution.setStatus(BatchStatus.FAILED);
                    execution.setEndTime(Instant.now());
                    execution.setExitStatus(new ExitStatus("CRASHED", "JVM crash recovery"));
                    jobRepository.update(execution);

                    // Also update step executions
                    for (StepExecution step : execution.getStepExecutions()) {
                        if (step.getStatus() == BatchStatus.STARTED) {
                            step.setStatus(BatchStatus.FAILED);
                            step.setEndTime(Instant.now());
                            jobRepository.update(step);
                        }
                    }

                    // Re-launch with same params → resumes from last checkpoint
                    Job job = context.getBean(jobName, Job.class);
                    jobLauncher.run(job, execution.getJobParameters());
                }
            }
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Don't restart too quickly — the "stale" execution might still be running on another node *(jaldi restart mat karo — ho sakta hai doosre node pe chal raha ho)*
- Stale detection threshold: use reasonable time (1 hour) not too aggressive (1 minute)
- Files: if the job was writing to a file, partial file state must be handled
- Non-transactional side effects (emails sent, API calls made) cannot be rolled back

### 🎯 Tricky Interview Qs

**Q: What happens to data that was ALREADY committed before the crash?**
It's safe. Each committed chunk's data is in the destination database. The framework only re-processes from the last checkpoint position (chunk 51 onward), not from the beginning.

**Q: What if two instances of the app restart simultaneously?**
Second launch gets `JobExecutionAlreadyRunningException` because the first already set it to STARTED. Only one can process — this is handled by Spring Batch's metadata lock.

### ⚡ Remember
- Crash → status stays STARTED (stale) *(crash ke baad status STARTED rehta hai — FAILED nahi hota)*
- Committed chunks = safe (transaction committed)
- Current chunk = rolled back (transaction lost)
- Recovery: detect stale → mark FAILED → relaunch with same params
- Resumes from ExecutionContext checkpoint

### 🔗 Follow-ups
- [Q65 → Restartability concept](#q65)
- [Q115 → ExecutionContext (checkpoint storage)](#q115)
- [Q106 → Debug failed jobs](#q106)

---

## Q121. How would you process files uploaded by multiple users?

### 📝 One-Liner
Each file = unique JobInstance (userId + fileName as identifying parameters) + async JobLauncher so uploads don't block.

### 🔑 Quick Answer
Each user's file gets a **unique JobInstance** via identifying parameters: userId + filePath + uploadTime. **Async JobLauncher** ensures the upload API returns immediately while processing happens in background. Each file is processed independently — different users' jobs don't interfere. Status endpoint lets users check progress. Limit concurrent jobs to prevent resource exhaustion. *(Har user ki file = alag JobInstance — async mein process hota hai, API turant response deta hai)*

### 📖 How It Works
```
Flow:

User A uploads report.csv:
  POST /upload → save file → async launch job
    params: {userId: "A", file: "/data/users/A/report.csv", time: "..."}
    → JobInstance #1 → processing in background
    → API returns: {jobId: 101, status: "STARTED"}

User B uploads data.csv (simultaneously):
  POST /upload → save file → async launch job
    params: {userId: "B", file: "/data/users/B/data.csv", time: "..."}
    → JobInstance #2 → processing in background
    → API returns: {jobId: 102, status: "STARTED"}

Both process independently:
  Job 101 (User A): reading → processing → writing → COMPLETED ✅
  Job 102 (User B): reading → processing → writing → COMPLETED ✅

Status check:
  GET /status/101 → {status: "COMPLETED", records: 5000}
  GET /status/102 → {status: "STARTED", progress: "60%"}
```

### 🗣️ Answering Approach
"Each uploaded file becomes a unique JobInstance with userId, filePath, and upload timestamp as identifying parameters. I use an async JobLauncher so the upload API returns immediately with a job ID while processing runs in the background. Users can check progress via a status endpoint. Each job is independent — one user's failure doesn't affect another. We store files in user-specific directories and limit concurrent processing to prevent resource exhaustion. In my project, this handled 50+ concurrent user uploads with an 8-thread pool for batch processing."

### 💻 Code
```java
@RestController
@RequestMapping("/api/upload")
public class FileUploadController {
    @Autowired private JobLauncher asyncJobLauncher;
    @Autowired private Job fileProcessingJob;
    @Autowired private JobExplorer jobExplorer;

    @PostMapping
    public ResponseEntity<Map<String, Object>> uploadFile(
            @RequestParam("file") MultipartFile file,
            @RequestParam("userId") String userId) throws Exception {
        // Save file to user-specific directory
        Path userDir = Path.of("/data/uploads", userId);
        Files.createDirectories(userDir);
        Path filePath = userDir.resolve(file.getOriginalFilename());
        file.transferTo(filePath.toFile());

        // Launch job with unique params
        JobParameters params = new JobParametersBuilder()
                .addString("userId", userId)
                .addString("filePath", filePath.toString())
                .addLocalDateTime("uploadTime", LocalDateTime.now())
                .toJobParameters();
        JobExecution execution = asyncJobLauncher.run(fileProcessingJob, params);

        return ResponseEntity.accepted().body(Map.of(
                "jobId", execution.getId(),
                "status", execution.getStatus().toString()
        ));
    }

    @GetMapping("/status/{jobId}")
    public Map<String, Object> getStatus(@PathVariable Long jobId) {
        JobExecution exec = jobExplorer.getJobExecution(jobId);
        return Map.of(
                "status", exec.getStatus().toString(),
                "startTime", exec.getStartTime(),
                "readCount", exec.getStepExecutions().stream()
                        .mapToLong(StepExecution::getReadCount).sum()
        );
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Limit concurrent jobs — 100 users uploading simultaneously = 100 threads → resource exhaustion *(concurrent jobs limit karo — bahut saare ek saath chalenge toh system hang hoga)*
- File cleanup: delete uploaded files after processing
- User-specific directories prevent file name collisions
- Async launcher means exceptions are NOT returned to API caller — poll status instead

### ⚡ Remember
- Unique params: userId + filePath + timestamp *(har upload = unique JobInstance)*
- Async JobLauncher = non-blocking API
- User-specific file directories
- Status endpoint for progress tracking
- Limit concurrent job count

### 🔗 Follow-ups
- [Q100 → Trigger types](#q100)
- [Q119 → Multiple concurrent jobs](#q119)
- [Q117 → Processing large files](#q117)

---

## Q122. How would you avoid duplicate processing?

### 📝 One-Liner
Three layers: JobParameters uniqueness (built-in), idempotent writer (UPSERT), and processed flag on source table.

### 🔑 Quick Answer
Three-layer defense: **(1) JobParameters uniqueness** — Spring Batch rejects re-running a COMPLETED job with same params (built-in). **(2) Idempotent writer** — use UPSERT (INSERT ON DUPLICATE KEY UPDATE) so re-processing the same record produces the same result. **(3) Processed flag** — mark source records as processed after writing, read only unprocessed records (WHERE processed=false). All three together = bulletproof against duplicates. *(Teen layers — job uniqueness, UPSERT, processed flag — teeno milke duplicate nahi hone dete)*

### 📖 How It Works
```
Three-Layer Defense:

Layer 1: JobParameters Uniqueness (built-in)
  Job with {date=2024-01-15} COMPLETED
  → Same params again → JobInstanceAlreadyCompleteException
  → No re-execution! ✅

Layer 2: Idempotent Writer (UPSERT)
  INSERT INTO target (id, amount, status)
  VALUES (123, 500, 'DONE')
  ON DUPLICATE KEY UPDATE amount=500, status='DONE';
  → Same record processed twice → same result ✅

Layer 3: Processed Flag (source-level)
  Reader:  SELECT * FROM orders WHERE processed = false
  Writer:  UPDATE orders SET processed = true WHERE id = ?
  → Already processed records never re-read ✅

Combined:
  Layer 1 prevents: entire job re-run
  Layer 2 handles: chunk retry (same items reprocessed after rollback)
  Layer 3 prevents: items re-read on restart
```

### 🗣️ Answering Approach
"I implement three layers of duplicate prevention. First, Spring Batch's built-in JobParameters uniqueness prevents re-running a completed job with the same parameters. Second, I use idempotent writers with UPSERT — INSERT ON DUPLICATE KEY UPDATE — so if the same record is processed twice during a retry, the result is identical. Third, I mark source records as processed after writing, and the reader only queries unprocessed records. The CompositeItemWriter handles both writing to the target and marking the source as processed in one step. In my project, this three-layer approach gave us zero duplicate processing across 6 months of daily runs."

### 💻 Code
```java
// Layer 2: Idempotent writer (UPSERT)
@Bean
public JdbcBatchItemWriter<ProcessedOrder> upsertWriter() {
    return new JdbcBatchItemWriterBuilder<ProcessedOrder>()
            .sql("INSERT INTO processed_orders (id, amount, status) " +
                 "VALUES (:id, :amount, :status) " +
                 "ON DUPLICATE KEY UPDATE amount=:amount, status=:status")
            .dataSource(dataSource)
            .beanMapped()
            .build();
}

// Layer 3: Read only unprocessed + mark processed after write
@Bean
@StepScope
public JdbcPagingItemReader<Order> unprocessedReader() {
    return new JdbcPagingItemReaderBuilder<Order>()
            .name("unprocessedReader")
            .dataSource(dataSource)
            .selectClause("SELECT id, amount, status")
            .fromClause("FROM orders")
            .whereClause("WHERE processed = false")  // only unprocessed
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)
            .rowMapper(new BeanPropertyRowMapper<>(Order.class))
            .build();
}

@Bean
public JdbcBatchItemWriter<Order> markProcessedWriter() {
    return new JdbcBatchItemWriterBuilder<Order>()
            .sql("UPDATE orders SET processed = true WHERE id = :id")
            .dataSource(dataSource)
            .beanMapped()
            .build();
}

// Composite: write target + mark source
@Bean
public CompositeItemWriter<ProcessedOrder> compositeWriter() {
    CompositeItemWriter<ProcessedOrder> writer = new CompositeItemWriter<>();
    writer.setDelegates(List.of(upsertWriter(), markProcessedWriter()));
    return writer;
}
```

### ⚠️ Pitfalls / Gotchas
- UPSERT syntax varies by DB: MySQL = ON DUPLICATE KEY UPDATE, PostgreSQL = ON CONFLICT DO UPDATE *(har database ka UPSERT syntax alag hai)*
- Processed flag and target write must be in same transaction — otherwise crash can mark processed without writing
- CompositeItemWriter delegates execute in order — target first, then mark
- Read-modify-write on processed flag needs index on processed column

### ⚡ Remember
- **Three layers**: params uniqueness, UPSERT, processed flag *(teen layers = bulletproof)*
- JobParameters = built-in duplicate prevention
- UPSERT = idempotent writes
- processed flag = source-level tracking
- CompositeItemWriter = write + mark in one step

### 🔗 Follow-ups
- [Q55 → JobInstance uniqueness](#q55)
- [Q123 → Track failed records](#q123)
- [Q70 → Skip logic](#q70)

---

## Q123. How would you track failed records?

### 📝 One-Liner
Implement SkipListener to log each skipped record to an error table with phase, record data, exception, and timestamp for audit and reprocessing.

### 🔑 Quick Answer
**SkipListener** has three callbacks: `onSkipInRead()`, `onSkipInProcess()`, `onSkipInWrite()`. For each skipped item, log to a **failed_records error table**: phase (READ/PROCESS/WRITE), record data, error message, job execution ID, timestamp. This gives you a complete audit trail. Build a separate **reprocessing job** that reads from the error table. In production: skip → log → alert → investigate → reprocess. *(Har skip hone wale record ko error table mein save karo — baad mein reprocess kar sakte ho)*

### 📖 How It Works
```
Skip → Track → Reprocess Flow:

During Processing:
  Record 1: ✅ success → write to target
  Record 2: ❌ NumberFormatException → SKIP
    → SkipListener.onSkipInProcess(record2, exception)
    → INSERT into failed_records (phase=PROCESS, data=record2, error=NFE)
  Record 3: ✅ success → write to target
  ...

Error Table (failed_records):
  ID | JOB_EXEC_ID | PHASE   | RECORD_DATA       | ERROR_MESSAGE          | TIMESTAMP           | STATUS
  1  | 101         | PROCESS | {id:2, amount:xyz} | NumberFormatException  | 2024-01-15 02:05:00 | NEW
  2  | 101         | READ    | line 5045          | MalformedCSVException  | 2024-01-15 02:06:00 | NEW
  3  | 101         | WRITE   | {id:99, amount:500}| ConstraintViolation    | 2024-01-15 02:07:00 | NEW

Reprocessing:
  Fix root cause (upstream data, code bug)
  → Run reprocessing job that reads error table (STATUS=NEW)
  → Process and write → mark STATUS=RESOLVED
```

### 🗣️ Answering Approach
"I implement a SkipListener that inserts each skipped record into an error table with the phase — whether it failed during reading, processing, or writing — along with the record data, exception message, and timestamp. This gives us a complete audit trail. In production, we alert when skip count exceeds a threshold, investigate the root cause, and then run a separate reprocessing job that reads from the error table. In my project, we tracked about 50 skipped records per day from data quality issues in partner files. The reprocessing job ran weekly after data corrections."

### 💻 Code
```java
@Component
public class FailedRecordTracker implements SkipListener<Order, ProcessedOrder> {
    @Autowired private JdbcTemplate jdbc;

    @Override
    public void onSkipInRead(Throwable t) {
        jdbc.update("INSERT INTO failed_records (phase, error_message, timestamp, status) " +
                    "VALUES (?, ?, ?, ?)",
                "READ", t.getMessage(), Timestamp.from(Instant.now()), "NEW");
    }

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        jdbc.update("INSERT INTO failed_records (phase, record_id, record_data, error_message, " +
                    "timestamp, status) VALUES (?, ?, ?, ?, ?, ?)",
                "PROCESS", item.getId(), item.toString(), t.getMessage(),
                Timestamp.from(Instant.now()), "NEW");
    }

    @Override
    public void onSkipInWrite(ProcessedOrder item, Throwable t) {
        jdbc.update("INSERT INTO failed_records (phase, record_id, record_data, error_message, " +
                    "timestamp, status) VALUES (?, ?, ?, ?, ?, ?)",
                "WRITE", item.getId(), item.toString(), t.getMessage(),
                Timestamp.from(Instant.now()), "NEW");
    }
}

// Register in step
@Bean
public Step processStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(Exception.class)
            .skipLimit(1000)
            .listener(failedRecordTracker)  // track skipped records
            .build();
}

// Error table schema
// CREATE TABLE failed_records (
//     id BIGINT AUTO_INCREMENT PRIMARY KEY,
//     job_execution_id BIGINT,
//     phase VARCHAR(10),        -- READ, PROCESS, WRITE
//     record_id VARCHAR(50),
//     record_data TEXT,
//     error_message TEXT,
//     timestamp TIMESTAMP,
//     status VARCHAR(20)        -- NEW, INVESTIGATING, RESOLVED
// );
```

### ⚠️ Pitfalls / Gotchas
- SkipListener runs outside chunk transaction — error table insert won't roll back with chunk *(SkipListener chunk transaction ke bahar hai — error insert rollback nahi hoga)*
- onSkipInRead doesn't have the item (only throwable) — log whatever info you can
- Large record_data column — consider truncating or storing only key fields
- Error table grows — implement cleanup for old RESOLVED records

### ⚡ Remember
- **SkipListener**: onSkipInRead, onSkipInProcess, onSkipInWrite *(teen callbacks — teen phases)*
- Log: phase, record data, exception, timestamp
- Error table with status lifecycle (NEW → INVESTIGATING → RESOLVED)
- Build reprocessing job from error table
- Alert on high skip counts

### 🔗 Follow-ups
- [Q70 → Skip logic](#q70)
- [Q122 → Avoid duplicate processing](#q122)
- [Q78 → Store rejected records](#q78)

---

## Q124. How would you cancel a running batch job?

### 📝 One-Liner
JobOperator.stop(executionId) — job stops gracefully after current chunk completes. Status: STARTED → STOPPING → STOPPED. Stopped jobs can be restarted.

### 🔑 Quick Answer
**`JobOperator.stop(executionId)`** sends a stop signal. The job doesn't stop immediately — it finishes the current chunk, commits the transaction, saves the ExecutionContext, then stops. Status transitions: STARTED → STOPPING → STOPPED. **Stopped jobs can be restarted** with the same parameters — they resume from the last checkpoint. For immediate stop (within current chunk): use `ChunkListener` + `setTerminateOnly()` on StepExecution. *(stop() graceful hai — current chunk pura hone ke baad rukta hai, data safe rehta hai)*

### 📖 How It Works
```
Graceful Stop Flow:

  Chunk 100: read → process → write → COMMIT ✅
  Chunk 101: read → process → write → COMMIT ✅
  Chunk 102: read → pro...                          ← stop() called HERE
                   ...cess → write → COMMIT ✅       ← chunk finishes!
  → ExecutionContext saved with position after chunk 102
  → Status: STARTED → STOPPING → STOPPED

  Restart later:
  → Resumes from chunk 103 (after last committed position)

Immediate Stop (within chunk):
  StepExecution.setTerminateOnly()
  → Current chunk abandoned (rolled back)
  → Status: FAILED (not STOPPED)
  → More aggressive, loses current chunk

Stop API:
  JobOperator.stop(executionId)     ← graceful
  StepExecution.setTerminateOnly()   ← immediate (current chunk lost)
```

### 🗣️ Answering Approach
"To cancel a running job, I use JobOperator.stop() which sends a graceful stop signal. The job finishes the current chunk — completes the transaction and saves the ExecutionContext — then transitions to STOPPED status. This is important because no data is lost and the job can be restarted later from the checkpoint. For urgent situations where we can't wait for the current chunk, I use setTerminateOnly() on the StepExecution inside a ChunkListener, which abandons the current chunk. In my project, we exposed stop and restart endpoints for our operations team to manage long-running jobs."

### 💻 Code
```java
// REST endpoints for stop/restart
@RestController
@RequestMapping("/api/batch")
public class BatchControlController {
    @Autowired private JobOperator jobOperator;

    @PostMapping("/stop/{executionId}")
    public String stopJob(@PathVariable Long executionId) throws Exception {
        jobOperator.stop(executionId);  // graceful stop
        return "Stop signal sent. Job will stop after current chunk completes.";
    }

    @PostMapping("/restart/{executionId}")
    public Long restartJob(@PathVariable Long executionId) throws Exception {
        return jobOperator.restart(executionId);  // restart from checkpoint
    }
}

// Conditional stop using ChunkListener (immediate)
@Component
public class ConditionalStopListener implements ChunkListener {
    @Autowired private SomeConditionService conditionService;

    @Override
    public void afterChunk(ChunkContext context) {
        // Check if job should stop (e.g., business hours started, resource limit reached)
        if (conditionService.shouldStopProcessing()) {
            StepExecution stepExecution = context.getStepContext().getStepExecution();
            stepExecution.setTerminateOnly();  // immediate stop
            log.info("Terminating job: condition met");
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- stop() is NOT immediate — current chunk completes first (can take seconds to minutes) *(stop() turant nahi rokta — current chunk pura hota hai)*
- STOPPED ≠ FAILED: STOPPED can restart, FAILED restart depends on config
- setTerminateOnly() loses current chunk (rolls back)
- JobOperator needs the execution ID — get it from the launch result or query metadata tables

### 🆚 vs. Comparison
| Method | Speed | Data Safety | Status | Restart |
|--------|-------|-------------|--------|---------|
| JobOperator.stop() | Slow (waits for chunk) | ✅ Safe | STOPPED | ✅ Yes |
| setTerminateOnly() | Fast (within chunk) | ❌ Chunk lost | FAILED | ✅ Yes |
| kill -9 (JVM) | Immediate | ❌ Chunk lost | STARTED (stale) | Manual recovery |

### ⚡ Remember
- **stop() = graceful** (waits for chunk, data safe) *(stop graceful hai — data safe rehta hai)*
- **setTerminateOnly() = immediate** (chunk rolled back)
- STARTED → STOPPING → STOPPED
- Stopped jobs CAN restart
- Expose stop/restart via REST for operations team

### 🔗 Follow-ups
- [Q120 → Restart after crash](#q120)
- [Q105 → Check job status](#q105)
- [Q119 → Multiple concurrent jobs](#q119)

---

## Q125. How would you limit batch job execution time?

### 📝 One-Liner
Two levels: transaction timeout per chunk (prevents hanging) and job-level timeout (scheduled JobOperator.stop() after deadline).

### 🔑 Quick Answer
**(1) Chunk-level timeout**: `DefaultTransactionAttribute.setTimeout(60)` — if a single chunk takes longer than 60 seconds (DB deadlock, network hang), the transaction is rolled back. Prevents infinite hanging. **(2) Job-level timeout**: schedule `jobOperator.stop()` to fire after the SLA deadline. If the job hasn't finished by deadline, it stops gracefully. The stopped job can be investigated and restarted. Both together = defense against hanging chunks AND overrunning jobs. *(Chunk timeout = agar ek chunk hang ho gaya, Job timeout = agar poora job slow ho gaya)*

### 📖 How It Works
```
Two Timeout Levels:

Level 1: Chunk Transaction Timeout (60 seconds)
  Chunk 1: read → process → write → DONE (5 sec) ✅
  Chunk 2: read → process → ☠️ DB DEADLOCK → waiting...
    → 60 sec timeout → TransactionTimedOutException → rollback
    → Chunk retried or skipped based on error handling config

  Without timeout: chunk waits forever → job hangs indefinitely
  With timeout:    chunk fails fast → job continues or fails gracefully

Level 2: Job-Level Timeout (SLA: 2 hours)
  Job starts at 2:00 AM
  Schedule: stop job at 4:00 AM if still running
  
  Scenario A: Job completes at 3:30 AM → timer cancelled → no action
  Scenario B: Job still running at 4:00 AM → stop() called → graceful stop
    → STOPPED → investigate why it's slow → restart after fixing
```

### 🗣️ Answering Approach
"I implement two timeout levels. For chunk-level, I set a transaction timeout of 60 seconds on the step — if a single chunk hangs due to a database deadlock or network issue, the transaction fails fast instead of waiting indefinitely. For job-level, I schedule a JobOperator.stop() call for the SLA deadline — if our 2-hour SLA is breached, the job stops gracefully. The stopped job preserves its checkpoint for restart after investigation. In my project, the chunk timeout caught a database deadlock that would have hung the job indefinitely, and the job-level timeout caught a performance regression where data volume exceeded expectations."

### 💻 Code
```java
// Level 1: Chunk transaction timeout
@Bean
public Step timedStep(JobRepository repo, PlatformTransactionManager tx) {
    DefaultTransactionAttribute txAttr = new DefaultTransactionAttribute();
    txAttr.setTimeout(60);  // 60 seconds per chunk

    return new StepBuilder("timedStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .transactionAttribute(txAttr)  // timeout per chunk
            .build();
}

// Level 2: Job-level timeout
@Component
public class JobTimeoutManager {
    @Autowired private JobOperator jobOperator;
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);

    public void launchWithTimeout(JobLauncher launcher, Job job,
                                   JobParameters params, Duration timeout) throws Exception {
        JobExecution execution = launcher.run(job, params);

        // Schedule stop after timeout
        scheduler.schedule(() -> {
            try {
                if (execution.isRunning()) {
                    log.warn("Job {} exceeded timeout of {}. Stopping.",
                            execution.getId(), timeout);
                    jobOperator.stop(execution.getId());
                }
            } catch (Exception e) {
                log.error("Failed to stop timed-out job", e);
            }
        }, timeout.toMillis(), TimeUnit.MILLISECONDS);
    }
}

// Usage: launch with 2-hour SLA
// jobTimeoutManager.launchWithTimeout(launcher, paymentJob, params, Duration.ofHours(2));
```

### ⚠️ Pitfalls / Gotchas
- Chunk timeout rolls back the chunk — doesn't stop the job (only that chunk fails) *(chunk timeout sirf ek chunk rollback karta hai — job nahi rokta)*
- Job-level stop is graceful — still waits for current chunk
- ScheduledExecutorService needs cleanup on app shutdown
- Transaction timeout doesn't work with all drivers — test with your specific DB

### ⚡ Remember
- **Chunk timeout**: `DefaultTransactionAttribute.setTimeout(60)` *(chunk hang hua toh 60 sec mein rollback)*
- **Job timeout**: scheduled `JobOperator.stop()` at SLA deadline
- Chunk timeout → rollback 1 chunk (job continues)
- Job timeout → graceful stop (restart possible)
- Both together = complete timeout protection

### 🔗 Follow-ups
- [Q124 → Cancel running job](#q124)
- [Q27 → Transaction management](#q27)
- [Q92 → Performance optimization](#q92)
