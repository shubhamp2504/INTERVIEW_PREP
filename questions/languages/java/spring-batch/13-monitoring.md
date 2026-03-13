# 🟡 Spring Batch — Monitoring & Debugging (Q104–Q109)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q104"></a>

## Q104. How do you monitor Spring Batch jobs?

### 🔑 Quick Answer

> Three levels: **Metadata tables** (SQL queries on BATCH_* tables), **Listeners** (JobExecutionListener logs start/end/counts), **Metrics** (Micrometer + Prometheus + Grafana dashboards).

### 📖 Step-by-Step Explanation

**Step 1 — Monitoring layers:**

```
Level 1: METADATA TABLES (built-in)
  Query BATCH_JOB_EXECUTION → status, start/end time
  Query BATCH_STEP_EXECUTION → read/write/skip counts
  ✅ Always available, zero setup
  ❌ Need manual SQL queries

Level 2: LISTENERS (application-level)
  JobExecutionListener → log job start/end, duration, status
  StepExecutionListener → log per-step metrics
  ChunkListener → log per-chunk timing
  ✅ Custom logging, alerting
  ❌ Need to implement

Level 3: METRICS (production-grade)
  Micrometer → expose metrics
  Prometheus → collect metrics
  Grafana → dashboards, alerts
  ✅ Real-time dashboards, historical data
  ❌ Infrastructure setup
```

**Step 2 — What to monitor:**

```
Must Monitor             | Where to Find
─────────────────────────|──────────────────────
Job status (COMPLETED?)  | BATCH_JOB_EXECUTION.STATUS
Job duration             | START_TIME → END_TIME
Read/Write counts        | BATCH_STEP_EXECUTION
Skip count (data quality)| BATCH_STEP_EXECUTION.SKIP_COUNT
Rollback count           | Should be 0 or very low
Memory usage             | JVM metrics / Micrometer
Throughput (records/sec) | write_count / duration
```

### 💻 Code Example

```java
// Production monitoring listener — logs everything you need
@Component
public class JobMonitoringListener implements JobExecutionListener {
    
    private static final Logger log = LoggerFactory.getLogger(JobMonitoringListener.class);
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("===== JOB STARTED: {} =====",
                jobExecution.getJobInstance().getJobName());
        log.info("Parameters: {}", jobExecution.getJobParameters());
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        long duration = Duration.between(
                jobExecution.getStartTime(),
                jobExecution.getEndTime()).toSeconds();
        
        log.info("===== JOB FINISHED =====");
        log.info("Status: {}", jobExecution.getStatus());
        log.info("Duration: {} seconds", duration);
        
        // Per-step summary
        jobExecution.getStepExecutions().forEach(step -> {
            log.info("  Step: {} | Status: {} | Read: {} | Written: {} | Skipped: {}",
                    step.getStepName(),
                    step.getStatus(),
                    step.getReadCount(),
                    step.getWriteCount(),
                    step.getSkipCount());
        });
        
        // Log failures
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            jobExecution.getAllFailureExceptions().forEach(ex ->
                    log.error("  Failure: {}", ex.getMessage()));
        }
    }
}
```

### 🗣️ How to Explain in Interview

> *"I monitor batch jobs at three levels. First, metadata tables — Spring Batch automatically records every execution in BATCH_JOB_EXECUTION and BATCH_STEP_EXECUTION tables with status, timestamps, and read/write counts. I query these for historical data. Second, I implement a JobExecutionListener that logs job start, end, duration, status, and per-step read/write/skip counts after every run. This gives instant visibility in application logs. Third, for production dashboards, I integrate Micrometer with Prometheus and Grafana — I expose job duration, throughput, and skip counts as metrics and set up alerts when skip count exceeds a threshold."*

### ⚡ Key Points to Remember

1. **Metadata tables** = built-in, always available
2. **JobExecutionListener** = custom logging per job
3. **Micrometer + Prometheus + Grafana** = production dashboards
4. Key metrics: **status, duration, read/write/skip counts**
5. Alert on **skip count spikes** (data quality issue)

---

<a id="q105"></a>

## Q105. How do you check job status?

### 🔑 Quick Answer

> Three ways: **SQL query** on BATCH_JOB_EXECUTION table, **JobExplorer** API (programmatic), or **REST endpoint** exposing status via JobExplorer.

### 📖 Step-by-Step Explanation

**Step 1 — The three approaches:**

```
1. SQL QUERY (quickest for debugging)
   SELECT * FROM BATCH_JOB_EXECUTION ORDER BY CREATE_TIME DESC

2. JobExplorer API (in application code)
   jobExplorer.getJobInstances("jobName", 0, 5)
   → For each instance: jobExplorer.getJobExecutions(instance)

3. REST ENDPOINT (for dashboards/external systems)
   GET /api/jobs/{jobName}/status → Returns latest status
```

### 💻 Code Examples

```sql
-- SQL: Quick status check
SELECT je.JOB_EXECUTION_ID, ji.JOB_NAME, je.STATUS, je.EXIT_CODE,
       je.START_TIME, je.END_TIME
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
ORDER BY je.CREATE_TIME DESC
LIMIT 10;
```

```java
// Programmatic: Using JobExplorer
@Service
public class JobStatusService {
    
    @Autowired private JobExplorer jobExplorer;
    
    public String getLatestStatus(String jobName) {
        List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 1);
        if (instances.isEmpty()) return "NEVER_RUN";
        
        List<JobExecution> executions = jobExplorer.getJobExecutions(instances.get(0));
        return executions.get(0).getStatus().toString();
    }
}

// REST endpoint for external monitoring
@RestController
@RequestMapping("/api/jobs")
public class JobStatusController {
    
    @Autowired private JobExplorer jobExplorer;
    
    @GetMapping("/{jobName}/status")
    public ResponseEntity<Map<String, Object>> getStatus(@PathVariable String jobName) {
        List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 1);
        if (instances.isEmpty()) return ResponseEntity.notFound().build();
        
        JobExecution latest = jobExplorer.getJobExecutions(instances.get(0)).get(0);
        Map<String, Object> result = new HashMap<>();
        result.put("status", latest.getStatus().toString());
        result.put("startTime", latest.getStartTime());
        result.put("endTime", latest.getEndTime());
        result.put("exitCode", latest.getExitStatus().getExitCode());
        return ResponseEntity.ok(result);
    }
}
```

### 🗣️ How to Explain in Interview

> *"I check job status three ways depending on context. For quick debugging, I query the BATCH_JOB_EXECUTION table directly — it has status, start/end times, and exit codes. In application code, I use JobExplorer — it's Spring Batch's read-only API for querying job metadata. For external monitoring systems, I expose a REST endpoint that uses JobExplorer to return the latest job status. The possible statuses are STARTING, STARTED, COMPLETED, FAILED, STOPPED, and ABANDONED."*

### ⚡ Key Points to Remember

1. **SQL query** = quickest for debugging
2. **JobExplorer** = programmatic read-only API
3. **REST endpoint** = for dashboards and external systems
4. Statuses: STARTING → STARTED → **COMPLETED** / **FAILED** / STOPPED
5. Always query **latest execution** (not instance) for current status

---

<a id="q106"></a>

## Q106. How do you debug a failed batch job?

### 🔑 Quick Answer

> Five-step debugging: (1) Check **job status** in BATCH_JOB_EXECUTION, (2) Find **which step failed**, (3) Check **read/write/skip counts**, (4) Read **EXIT_MESSAGE** for exception, (5) Check **ExecutionContext** for state at failure.

### 📖 Step-by-Step Explanation

**Step 1 — The debugging flow:**

```
Step 1: WHAT FAILED?
  SELECT STATUS, EXIT_CODE, EXIT_MESSAGE 
  FROM BATCH_JOB_EXECUTION 
  WHERE STATUS = 'FAILED' 
  ORDER BY CREATE_TIME DESC LIMIT 1;
  
  → Result: "Job 'importJob' FAILED"

Step 2: WHICH STEP FAILED?
  SELECT STEP_NAME, STATUS, EXIT_CODE
  FROM BATCH_STEP_EXECUTION
  WHERE JOB_EXECUTION_ID = 42 AND STATUS = 'FAILED';
  
  → Result: "processOrdersStep FAILED"

Step 3: HOW FAR DID IT GET?
  SELECT READ_COUNT, WRITE_COUNT, SKIP_COUNT, ROLLBACK_COUNT, COMMIT_COUNT
  FROM BATCH_STEP_EXECUTION
  WHERE STEP_EXECUTION_ID = 105;
  
  → Result: read=45000, write=44500, skip=500, rollback=3
  → Insight: failed around record 45,000, had 500 skips + 3 rollbacks

Step 4: WHAT WAS THE EXCEPTION?
  SELECT EXIT_MESSAGE FROM BATCH_STEP_EXECUTION
  WHERE STEP_EXECUTION_ID = 105;
  
  → Result: "org.springframework.dao.DataIntegrityViolation: Duplicate key..."

Step 5: WHAT WAS THE STATE?
  SELECT SHORT_CONTEXT FROM BATCH_STEP_EXECUTION_CONTEXT
  WHERE STEP_EXECUTION_ID = 105;
  
  → Result: { "currentPage": 90, "lastCommittedId": 44500 }
  → Insight: was on page 90, last successful write was ID 44500
```

### 💻 Code Example

```java
// Automated failure diagnosis listener
@Component
public class FailureDiagnosisListener implements JobExecutionListener {
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            log.error("=== FAILURE DIAGNOSIS ===");
            log.error("Job: {}", jobExecution.getJobInstance().getJobName());
            log.error("Params: {}", jobExecution.getJobParameters());
            
            jobExecution.getStepExecutions().stream()
                    .filter(step -> step.getStatus() == BatchStatus.FAILED)
                    .forEach(step -> {
                        log.error("Failed Step: {}", step.getStepName());
                        log.error("  Read: {}, Written: {}, Skipped: {}",
                                step.getReadCount(), step.getWriteCount(), step.getSkipCount());
                        log.error("  Rollbacks: {}, Commits: {}",
                                step.getRollbackCount(), step.getCommitCount());
                        log.error("  Exit: {}", step.getExitStatus().getExitDescription());
                    });
            
            jobExecution.getAllFailureExceptions()
                    .forEach(ex -> log.error("Exception: ", ex));
        }
    }
}
```

### 🗣️ How to Explain in Interview

> *"I debug failed jobs in five steps using metadata tables. First, I check BATCH_JOB_EXECUTION for the failed job. Second, I find which step failed in BATCH_STEP_EXECUTION. Third, I look at read/write/skip counts to understand progress — if it read 45,000 and wrote 44,500 with 500 skips, I know it failed around record 45,000. Fourth, I check the EXIT_MESSAGE column for the actual exception — usually constraint violations or parsing errors. Fifth, I check BATCH_STEP_EXECUTION_CONTEXT for the state at failure — like current page number — which tells me exactly where to look in the source data. I also implement a FailureDiagnosisListener that logs all this automatically."*

### ⚡ Key Points to Remember

1. **BATCH_JOB_EXECUTION** → was the job FAILED?
2. **BATCH_STEP_EXECUTION** → which step failed? + counts
3. **EXIT_MESSAGE** → the actual exception
4. **EXECUTION_CONTEXT** → state at failure point
5. Implement **FailureDiagnosisListener** for automatic logging

---

<a id="q107"></a>

## Q107. What logs should be monitored?

### 🔑 Quick Answer

> Monitor **skip count spikes** (data quality), **rollback count** (should be ~0), **duration trends** (performance regression), **memory usage** (leak detection), and **connection pool** (exhaustion).

### 📖 Step-by-Step Explanation

```
CRITICAL ALERTS (immediate action):
  🔴 Job FAILED → check exception, fix, restart
  🔴 OOM error → reduce chunk size, add memory
  🔴 Connection pool exhausted → increase pool or reduce partitions

WARNING ALERTS (investigate):
  🟡 Skip count > threshold → data quality degraded
  🟡 Duration > 2× normal → performance regression
  🟡 Rollback count > 0 → transient failures occurring
  🟡 Memory steadily increasing → possible leak

INFO TRACKING (trend analysis):
  🟢 Job duration over time → detect gradual slowdowns
  🟢 Data volume trends → plan capacity
  🟢 Skip count trends → data quality over time
  🟢 Throughput (records/sec) → benchmark for optimization
```

### 💻 Code Example

```yaml
# Production logging configuration
logging:
  level:
    org.springframework.batch.core: INFO          # Job/Step lifecycle
    org.springframework.batch.core.step.item: WARN # Reduce chunk noise
    org.springframework.batch.repeat: WARN         # Reduce repeat noise
    com.zaxxer.hikari: WARN                        # Connection pool alerts
    your.package.batch: INFO                       # Your batch code
```

```java
// Alert on skip count threshold
@Component
public class SkipAlertListener implements StepExecutionListener {
    
    private static final int SKIP_THRESHOLD = 100;
    
    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        if (stepExecution.getSkipCount() > SKIP_THRESHOLD) {
            log.warn("⚠️ HIGH SKIP COUNT: Step={}, Skipped={} (threshold={})",
                    stepExecution.getStepName(),
                    stepExecution.getSkipCount(),
                    SKIP_THRESHOLD);
            // Send alert: email, Slack, PagerDuty
        }
        return stepExecution.getExitStatus();
    }
}
```

### 🗣️ How to Explain in Interview

> *"I monitor five key signals. First, skip count — a sudden spike means data quality issues in the source. I set a threshold alert, say 100 skips, that triggers a Slack notification. Second, rollback count — should be near zero; if it's climbing, something is causing transaction failures. Third, duration trends — if a job that normally takes 10 minutes starts taking 30, there's a performance regression. Fourth, memory — steady growth across chunks indicates a leak, usually JPA persistence context. Fifth, connection pool — if jobs start failing with 'connection timeout', the pool is exhausted."*

### ⚡ Key Points to Remember

1. **Skip count spike** = data quality issue → alert
2. **Rollback count > 0** = transient failures → investigate
3. **Duration trend** = catch performance regressions
4. **Memory growth** = detect JPA leaks
5. Set **batch logging to INFO** in production (not DEBUG)

---

<a id="q108"></a>

## Q108. How do you track job performance?

### 🔑 Quick Answer

> Use **Micrometer** to expose metrics (duration, throughput, counts), **Prometheus** to collect them, and **Grafana** for dashboards. Key metrics: job duration, records/second, skip rate.

### 📖 Step-by-Step Explanation

**Step 1 — Metrics to track:**

```
THROUGHPUT METRICS:
  batch.job.duration     → How long the job took
  batch.step.read_count  → How many records read
  batch.step.write_count → How many records written
  batch.step.throughput  → records/second
  
QUALITY METRICS:
  batch.step.skip_count  → Data quality indicator
  batch.step.skip_rate   → skip_count / read_count (should be < 1%)
  batch.job.failure_count → Number of failed executions

RESOURCE METRICS:
  jvm.memory.used        → Memory consumption
  hikari.connections.active → DB connections in use
```

### 💻 Code Example

```java
// Micrometer integration for Prometheus/Grafana
@Component
public class BatchMetricsListener implements JobExecutionListener {
    
    private final MeterRegistry meterRegistry;
    
    public BatchMetricsListener(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public void afterJob(JobExecution exec) {
        String jobName = exec.getJobInstance().getJobName();
        long durationMs = Duration.between(
                exec.getStartTime(), exec.getEndTime()).toMillis();
        
        // Job duration
        meterRegistry.timer("batch.job.duration", "job", jobName)
                .record(durationMs, TimeUnit.MILLISECONDS);
        
        // Job status counter
        meterRegistry.counter("batch.job.completions",
                "job", jobName,
                "status", exec.getStatus().toString()).increment();
        
        // Per-step metrics
        exec.getStepExecutions().forEach(step -> {
            String stepName = step.getStepName();
            long stepDuration = Duration.between(
                    step.getStartTime(), step.getEndTime()).toSeconds();
            
            meterRegistry.gauge("batch.step.read_count",
                    Tags.of("step", stepName), step.getReadCount());
            meterRegistry.gauge("batch.step.write_count",
                    Tags.of("step", stepName), step.getWriteCount());
            
            // Throughput: records per second
            if (stepDuration > 0) {
                double throughput = (double) step.getWriteCount() / stepDuration;
                meterRegistry.gauge("batch.step.throughput",
                        Tags.of("step", stepName), throughput);
            }
        });
    }
}
```

### 🗣️ How to Explain in Interview

> *"I track batch performance with Micrometer, Prometheus, and Grafana. In a JobExecutionListener, I record job duration, read/write counts, and calculate throughput as records per second. These metrics go to Prometheus via Micrometer, and I build Grafana dashboards showing duration trends over time, throughput comparisons, and skip rate. This lets me spot performance regressions immediately — if yesterday the job took 10 minutes and today it took 25, the dashboard shows the spike. I also set Grafana alerts for duration exceeding 2× the average."*

### ⚡ Key Points to Remember

1. **Micrometer** → expose metrics from code
2. **Prometheus** → collect and store metrics
3. **Grafana** → dashboards and alerts
4. Key metrics: **duration**, **throughput (records/sec)**, **skip rate**
5. Alert on **duration > 2× average** and **skip rate > 1%**

---

<a id="q109"></a>

## Q109. How do you find slow steps?

### 🔑 Quick Answer

> Query **BATCH_STEP_EXECUTION** for duration per step, calculate **records/second** per step. For chunk-level detail, implement a **ChunkListener** that logs timing per chunk and alerts on slow chunks.

### 📖 Step-by-Step Explanation

**Step 1 — Step-level analysis:**

```sql
-- Find slowest steps in a job execution
SELECT STEP_NAME,
       TIMESTAMPDIFF(SECOND, START_TIME, END_TIME) AS duration_sec,
       READ_COUNT,
       WRITE_COUNT,
       WRITE_COUNT / NULLIF(TIMESTAMPDIFF(SECOND, START_TIME, END_TIME), 0) AS records_per_sec
FROM BATCH_STEP_EXECUTION
WHERE JOB_EXECUTION_ID = 42
ORDER BY duration_sec DESC;

-- Result:
-- processOrdersStep  | 600 sec | 500000 read | 499000 write | 831 rec/sec ← Slow!
-- enrichmentStep     |  30 sec |  10000 read |  10000 write | 333 rec/sec
-- reportStep         |   5 sec |      1 read |      1 write |   —
```

**Step 2 — Chunk-level analysis (find slow chunks):**

```
Step processes 100,000 records in 200 chunks (500 per chunk):

Chunk  1: 200ms ✅
Chunk  2: 180ms ✅
Chunk  3: 210ms ✅
...
Chunk 45: 15000ms ⚠️ SLOW! (75× normal)
...
Chunk 200: 190ms ✅

Chunk 45 took 15 seconds — why?
  → Network timeout on external API?
  → Garbage collection pause?
  → DB lock contention?
```

### 💻 Code Example

```java
// Chunk timing listener — detects slow chunks
@Component
public class ChunkTimingListener implements ChunkListener {
    
    private static final Logger log = LoggerFactory.getLogger(ChunkTimingListener.class);
    private long chunkStartTime;
    private int chunkCount = 0;
    private static final long SLOW_THRESHOLD_MS = 5000; // 5 seconds
    
    @Override
    public void beforeChunk(ChunkContext context) {
        chunkStartTime = System.currentTimeMillis();
        chunkCount++;
    }
    
    @Override
    public void afterChunk(ChunkContext context) {
        long duration = System.currentTimeMillis() - chunkStartTime;
        
        if (duration > SLOW_THRESHOLD_MS) {
            log.warn("⚠️ SLOW CHUNK #{}: Step={}, Duration={}ms (threshold={}ms)",
                    chunkCount,
                    context.getStepContext().getStepName(),
                    duration,
                    SLOW_THRESHOLD_MS);
        }
    }
    
    @Override
    public void afterChunkError(ChunkContext context) {
        long duration = System.currentTimeMillis() - chunkStartTime;
        log.error("❌ CHUNK ERROR #{}: Step={}, Duration={}ms",
                chunkCount,
                context.getStepContext().getStepName(),
                duration);
    }
}
```

### 🗣️ How to Explain in Interview

> *"I find slow steps at two levels. First, I query BATCH_STEP_EXECUTION and calculate records per second for each step — the step with the lowest throughput is the bottleneck. Second, for deeper analysis, I implement a ChunkListener that measures each chunk's processing time. If a chunk exceeds a threshold — say 5 seconds when normal chunks take 200ms — it logs a warning with the chunk number and step name. This pinpoints exactly which chunk was slow, which helps me trace back to specific data ranges or external service timeouts causing the slowdown."*

### ⚡ Key Points to Remember

1. **SQL on BATCH_STEP_EXECUTION** → find slowest step
2. Calculate **records/second** per step for comparison
3. **ChunkListener** → find slow individual chunks
4. Alert when chunk duration > **5× average**
5. Slow chunks usually caused by: **GC pauses, DB locks, network timeouts**

---

> **🎯 Navigation:** [← Scheduling (Q99-103)](12-scheduling.md) | [Next → Database & Metadata (Q110-116)](14-database-metadata.md) | [📋 All Sections](README.md)
