<![CDATA[<div align="center">

# 🔴 Spring Batch — Real Production Scenario Questions (117-125)

[![Questions](https://img.shields.io/badge/Questions-9-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Hard-red.svg)](#)
[![Type](https://img.shields.io/badge/Type-Scenario%20Based-purple.svg)](#)

</div>

---

<a id="q1"></a>
## Q117. ❓ How would you process a 10GB file using Spring Batch?

🔖 **Tags:** `#spring-batch` `#production` `#large-file` `#must-know`  
📊 **Difficulty:** 🔴 Hard  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

```
Strategy: FlatFileItemReader + Partitioning by File Splits

Option 1: Single-Threaded (Simple, Slower)
──────────────────────────────────────────
FlatFileItemReader → streams line by line (low memory!)
Chunk size: 500-1000
Writer: JdbcBatchItemWriter

Memory: ~100MB (only current chunk in memory)
Time: 1-3 hours depending on processing

Option 2: Multi-File Partitioning (Fast, Recommended)
──────────────────────────────────────────
Pre-step: Split 10GB file into 10 x 1GB files (Tasklet)
Master: FilePartitioner → assigns one file per partition
Slaves: 10 parallel FlatFileItemReaders

Time: ~10-20 minutes with 10 threads
```

```java
// Pre-processing Tasklet: Split large file
@Bean
public Tasklet fileSplitTasklet() {
    return (contribution, chunkContext) -> {
        // Split 10GB file into smaller files (using Linux split or Java)
        Runtime.getRuntime().exec("split -l 1000000 input.csv split_");
        return RepeatStatus.FINISHED;
    };
}

// Partitioner: One file per partition
@Bean
public MultiResourcePartitioner partitioner() {
    MultiResourcePartitioner partitioner = new MultiResourcePartitioner();
    partitioner.setResources(resources);  // split_aa, split_ab, split_ac...
    partitioner.setKeyName("file");
    return partitioner;
}

// Worker step: Each partition processes one file
@Bean
@StepScope
public FlatFileItemReader<Record> reader(
        @Value("#{stepExecutionContext['file']}") Resource file) {
    return new FlatFileItemReaderBuilder<Record>()
            .name("partitionReader")
            .resource(file)
            .delimited().names("col1", "col2", "col3")
            .targetType(Record.class)
            .build();
}

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

---

<a id="q2"></a>
## Q118. ❓ How would you process 100 million database records?

🔖 **Tags:** `#spring-batch` `#production` `#large-dataset`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
Strategy: Partitioning by ID Range + JdbcPagingItemReader

100M records ÷ 20 partitions = 5M per partition
Each partition: JdbcPagingItemReader (page size 500)
Chunk size: 500
Writer: JdbcBatchItemWriter
Threads: 20 (match partition count)

Estimated time: ~30-60 minutes
```

```java
@Bean
public Partitioner idRangePartitioner(JdbcTemplate jdbc) {
    return gridSize -> {
        Long min = jdbc.queryForObject("SELECT MIN(id) FROM records", Long.class);
        Long max = jdbc.queryForObject("SELECT MAX(id) FROM records", Long.class);
        long range = (max - min) / gridSize + 1;
        
        Map<String, ExecutionContext> partitions = new HashMap<>();
        for (int i = 0; i < gridSize; i++) {
            ExecutionContext ctx = new ExecutionContext();
            ctx.putLong("minId", min + (i * range));
            ctx.putLong("maxId", Math.min(min + ((i + 1) * range) - 1, max));
            partitions.put("partition" + i, ctx);
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
    
    Map<String, Object> params = new HashMap<>();
    params.put("minId", minId);
    params.put("maxId", maxId);
    
    return new JdbcPagingItemReaderBuilder<Record>()
            .dataSource(ds)
            .selectClause("SELECT *")
            .fromClause("FROM records")
            .whereClause("WHERE id BETWEEN :minId AND :maxId")
            .sortKeys(Map.of("id", Order.ASCENDING))
            .pageSize(500)
            .parameterValues(params)
            .rowMapper(new BeanPropertyRowMapper<>(Record.class))
            .build();
}
```

---

<a id="q3"></a>
## Q119. ❓ How would you handle multiple batch jobs running simultaneously?

🔖 **Tags:** `#spring-batch` `#concurrent-jobs` `#production`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Concern | Solution |
|---------|---------|
| **Resource contention** | Use different thread pools per job |
| **DB connection exhaustion** | Limit total connections across jobs |
| **Data conflicts** | Ensure jobs process different data sets |
| **Priority** | Use job queue with priority ordering |
| **Job dependency** | Use job orchestration (Step → conditional next job) |

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

// Run jobs with async launcher
@Bean
public JobLauncher asyncJobLauncher(JobRepository repo) {
    TaskExecutorJobLauncher launcher = new TaskExecutorJobLauncher();
    launcher.setJobRepository(repo);
    launcher.setTaskExecutor(new SimpleAsyncTaskExecutor());  // Non-blocking
    return launcher;
}
```

---

<a id="q4"></a>
## Q120. ❓ How would you restart a job after a crash?

🔖 **Tags:** `#spring-batch` `#restart` `#crash-recovery`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
Scenario: JVM crashed during processing

1. On crash: Current chunk rolls back (uncommitted)
   Metadata tables: JobExecution status = STARTED (stale)

2. On restart:
   a. Spring Batch finds stale STARTED execution
   b. Marks it as FAILED (abandoned)
   c. Creates new JobExecution
   d. Resumes from last committed chunk

   OR manually:
   UPDATE BATCH_JOB_EXECUTION SET STATUS='FAILED', EXIT_CODE='FAILED' 
   WHERE STATUS='STARTED';
```

```java
// Automatic handling with ApplicationRunner
@Component
public class CrashRecovery implements ApplicationRunner {
    @Autowired private JobExplorer jobExplorer;
    @Autowired private JobRepository jobRepository;
    @Autowired private JobLauncher jobLauncher;
    
    @Override
    public void run(ApplicationArguments args) throws Exception {
        // Find abandoned (stale STARTED) executions
        Set<JobExecution> running = jobExplorer.findRunningJobExecutions("myJob");
        for (JobExecution exec : running) {
            exec.setStatus(BatchStatus.FAILED);
            exec.setEndTime(LocalDateTime.now());
            jobRepository.update(exec);
            
            // Restart the failed job
            jobLauncher.run(exec.getJobInstance().getJob(), 
                           exec.getJobParameters());
        }
    }
}
```

---

<a id="q5"></a>
## Q121. ❓ How would you process files uploaded by multiple users?

🔖 **Tags:** `#spring-batch` `#multi-user` `#file-processing`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
Architecture:
1. User uploads file → stored in /uploads/{userId}/
2. File watcher OR REST API triggers batch job
3. Each file = separate Job execution (different params)
4. Results stored with userId association

Key: Each user's file = unique JobInstance (userId + fileName as params)
```

```java
@PostMapping("/upload")
public ResponseEntity<?> handleUpload(
        @RequestParam MultipartFile file, 
        @RequestParam String userId) throws Exception {
    
    // Save file
    String path = "/uploads/" + userId + "/" + file.getOriginalFilename();
    file.transferTo(new File(path));
    
    // Trigger batch job
    JobParameters params = new JobParametersBuilder()
            .addString("userId", userId)
            .addString("filePath", path)
            .addLong("uploadTime", System.currentTimeMillis())
            .toJobParameters();
    
    // Async launch — don't block the API
    JobExecution exec = asyncJobLauncher.run(fileProcessJob, params);
    
    return ResponseEntity.ok(Map.of("jobId", exec.getId(), "status", "PROCESSING"));
}
```

---

<a id="q6"></a>
## Q122. ❓ How would you avoid duplicate processing?

🔖 **Tags:** `#spring-batch` `#idempotency` `#duplicate`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Strategy | How |
|----------|-----|
| **JobParameters uniqueness** | Same params = same instance (won't re-run) |
| **Idempotent writer** | UPSERT instead of INSERT |
| **Processed flag** | Mark records as processed in source |
| **Deduplication in processor** | Check if record already exists |
| **File tracking** | Track processed files in a table |

```java
// Idempotent writer: UPSERT (INSERT ... ON DUPLICATE KEY UPDATE)
@Bean
public JdbcBatchItemWriter<Employee> idempotentWriter(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .sql("""
                INSERT INTO employees (id, name, salary) VALUES (:id, :name, :salary)
                ON DUPLICATE KEY UPDATE name = :name, salary = :salary
            """)
            .dataSource(ds)
            .beanMapped()
            .build();
}

// Mark source records as processed
@Bean
public JdbcBatchItemWriter<Record> markProcessedWriter(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<Record>()
            .sql("UPDATE source_records SET processed = true WHERE id = :id")
            .dataSource(ds)
            .beanMapped()
            .build();
}

// Reader: Only read unprocessed records
.whereClause("WHERE processed = false")
```

---

<a id="q7"></a>
## Q123. ❓ How would you track failed records?

🔖 **Tags:** `#spring-batch` `#failed-records` `#production`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader()).processor(processor()).writer(writer())
            .faultTolerant()
            .skip(Exception.class).skipLimit(1000)
            .listener(failedRecordTracker())
            .build();
}

@Bean
public SkipListener<Employee, Employee> failedRecordTracker() {
    return new SkipListener<>() {
        @Autowired JdbcTemplate jdbc;
        
        @Override
        public void onSkipInRead(Throwable t) {
            jdbc.update(
                "INSERT INTO failed_records(phase, error, timestamp) VALUES(?,?,?)",
                "READ", t.getMessage(), LocalDateTime.now()
            );
        }
        
        @Override
        public void onSkipInProcess(Employee item, Throwable t) {
            jdbc.update(
                "INSERT INTO failed_records(phase, record_id, data, error, timestamp) VALUES(?,?,?,?,?)",
                "PROCESS", item.getId(), item.toString(), t.getMessage(), LocalDateTime.now()
            );
        }
        
        @Override
        public void onSkipInWrite(Employee item, Throwable t) {
            jdbc.update(
                "INSERT INTO failed_records(phase, record_id, data, error, timestamp) VALUES(?,?,?,?,?)",
                "WRITE", item.getId(), item.toString(), t.getMessage(), LocalDateTime.now()
            );
        }
    };
}
```

---

<a id="q8"></a>
## Q124. ❓ How would you cancel a running batch job?

🔖 **Tags:** `#spring-batch` `#cancel` `#stop-job`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```java
// Method 1: JobOperator.stop()
@Autowired private JobOperator jobOperator;

public void stopJob(Long executionId) throws Exception {
    jobOperator.stop(executionId);
    // Sets status to STOPPING → after current chunk completes → STOPPED
}

// Method 2: REST endpoint
@PostMapping("/api/jobs/{executionId}/stop")
public ResponseEntity<?> stopJob(@PathVariable Long executionId) throws Exception {
    jobOperator.stop(executionId);
    return ResponseEntity.ok("Job stopping after current chunk...");
}
```

### ⚠️ Important:
- `stop()` does NOT immediately kill the job
- It sets a flag — the job stops **after the current chunk completes**
- Status goes: `STARTED → STOPPING → STOPPED`
- A STOPPED job can be **restarted** later

### Method 3: Custom StepExecution Listener (force stop):
```java
@Bean
public StepListener conditionalStopListener() {
    return new ChunkListener() {
        @Override
        public void afterChunk(ChunkContext context) {
            if (shouldStop()) {  // External flag, DB check, etc.
                context.getStepContext().getStepExecution()
                    .setTerminateOnly();  // Force stop after this chunk
            }
        }
    };
}
```

---

<a id="q9"></a>
## Q125. ❓ How would you limit batch job execution time?

🔖 **Tags:** `#spring-batch` `#timeout` `#execution-limit`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```java
// Method 1: Transaction timeout per chunk
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    DefaultTransactionAttribute attr = new DefaultTransactionAttribute();
    attr.setTimeout(60);  // 60 seconds per chunk
    
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader()).writer(writer())
            .transactionAttribute(attr)
            .build();
}

// Method 2: Job-level timeout with scheduled stop
@Component
public class JobTimeoutManager {
    @Autowired private JobOperator jobOperator;
    
    private final ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(1);
    
    public void startWithTimeout(JobLauncher launcher, Job job, 
                                  JobParameters params, Duration timeout) throws Exception {
        JobExecution exec = launcher.run(job, params);
        
        scheduler.schedule(() -> {
            try {
                if (exec.isRunning()) {
                    jobOperator.stop(exec.getId());
                    log.warn("Job {} timed out after {}", exec.getId(), timeout);
                }
            } catch (Exception e) {
                log.error("Failed to stop timed-out job", e);
            }
        }, timeout.toMillis(), TimeUnit.MILLISECONDS);
    }
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: DB & Metadata](./14-database-metadata.md)
]]>