# 🟡 Monitoring & Debugging — Q104 to Q109

---

## Q104. How do you monitor Spring Batch jobs?

### 📝 One-Liner
Three levels: metadata tables (SQL queries), listeners (JobExecutionListener), and metrics (Micrometer + Prometheus + Grafana).

### 🔑 Quick Answer
**(1) Metadata tables**: query BATCH_JOB_EXECUTION and BATCH_STEP_EXECUTION for status, counts, duration. Built-in, always available. **(2) Listeners**: JobExecutionListener/StepExecutionListener for custom logging — log per-step summary (read, written, skipped) and failure details. **(3) Metrics**: Micrometer exposes metrics → Prometheus collects → Grafana dashboards + alerts. Key metrics: job status, duration, read/write/skip counts, throughput, memory. *(Teen levels — DB tables, listeners, metrics — production mein teeno chahiye)*

### 📖 How It Works
```
Three Monitoring Layers:

Layer 1: Metadata Tables (built-in)
  SELECT status, start_time, end_time FROM BATCH_JOB_EXECUTION
  → Quick health check, debugging

Layer 2: Listeners (custom logging)
  JobExecutionListener.afterJob() →
    "Job 'paymentJob' COMPLETED in 8 min"
    "  Step processPayments: read=10M, written=9.9M, skipped=500"
    "  Step generateReport: read=1, written=1, skipped=0"
  → Structured logging for ELK/Splunk

Layer 3: Metrics (dashboards + alerts)
  Micrometer → spring.batch.job.active (gauge)
             → spring.batch.job (timer)
             → spring.batch.step (timer)
             → spring.batch.item.read (counter)
  Prometheus scrapes → Grafana displays
  Alert: job duration > 2× average → PagerDuty

What to Monitor:
  ☐ Job status (COMPLETED/FAILED)
  ☐ Duration (trending up = regression)
  ☐ Read/Write counts (match expected)
  ☐ Skip count (spike = data quality issue)
  ☐ Memory usage (growth = leak)
  ☐ Throughput (records/second)
```

### 🗣️ Answering Approach
"I implement three monitoring layers. First, metadata tables — Spring Batch automatically logs every execution to BATCH_JOB_EXECUTION and BATCH_STEP_EXECUTION, which I query for quick debugging. Second, I implement JobExecutionListener that logs a structured summary after each job — step-by-step read, written, and skipped counts plus any exception details. This feeds into our ELK stack for searchable logs. Third, Micrometer metrics flow into Prometheus and Grafana for real-time dashboards. We set alerts for job failures, duration exceeding 2× average, and skip count spikes — which signal data quality issues."

### 💻 Code
```java
// JobExecutionListener — structured monitoring
@Component
public class JobMonitoringListener implements JobExecutionListener {

    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("Job '{}' STARTING with params: {}",
                jobExecution.getJobInstance().getJobName(),
                jobExecution.getJobParameters());
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        log.info("Job '{}' {} in {} seconds",
                jobExecution.getJobInstance().getJobName(),
                jobExecution.getStatus(),
                Duration.between(jobExecution.getStartTime(), 
                        jobExecution.getEndTime()).getSeconds());

        // Per-step summary
        for (StepExecution step : jobExecution.getStepExecutions()) {
            log.info("  Step '{}': status={}, read={}, written={}, skipped={}, " +
                     "filtered={}, commits={}, rollbacks={}",
                    step.getStepName(), step.getStatus(),
                    step.getReadCount(), step.getWriteCount(),
                    step.getSkipCount(), step.getFilterCount(),
                    step.getCommitCount(), step.getRollbackCount());
        }

        // Log failures
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            jobExecution.getAllFailureExceptions().forEach(ex ->
                    log.error("  Failure: {}", ex.getMessage()));
        }
    }
}

// Micrometer + Prometheus (auto-configured in Spring Boot)
// pom.xml: spring-boot-starter-actuator + micrometer-registry-prometheus
// application.yml:
// management:
//   endpoints:
//     web:
//       exposure:
//         include: prometheus, health, metrics
//   metrics:
//     tags:
//       application: payment-batch
```

### ⚠️ Pitfalls / Gotchas
- Metadata tables grow indefinitely — implement cleanup job for old executions *(metadata tables badhte rehte hain — purane records clean karo)*
- Micrometer metrics need actuator + prometheus dependency to expose
- Skip count = 0 doesn't mean no issues — check filter count too
- Duration trending up slowly = performance regression — set alert threshold

### ⚡ Remember
- **Three layers**: tables, listeners, metrics *(teeno layers production mein chahiye)*
- Metadata tables = built-in debugging
- Listeners = structured logging (ELK/Splunk)
- Micrometer → Prometheus → Grafana = dashboards + alerts
- Alert on: failures, duration spikes, skip count spikes

### 🔗 Follow-ups
- [Q105 → Check job status](#q105)
- [Q106 → Debug failed jobs](#q106)
- [Q109 → Find slow steps](#q109)

---

## Q105. How do you check job status?

### 📝 One-Liner
Three ways: SQL query on BATCH_JOB_EXECUTION (quickest), JobExplorer API (programmatic), REST endpoint (dashboards).

### 🔑 Quick Answer
**(1) SQL query** — direct query on BATCH_JOB_EXECUTION table. Quickest for debugging. **(2) JobExplorer** — read-only Spring Batch API. `getJobInstances()`, `getJobExecutions()`, `getStepExecutions()`. Use in code. **(3) REST endpoint** — expose status via controller for dashboards and external monitoring. Status lifecycle: STARTING → STARTED → COMPLETED / FAILED / STOPPED. *(SQL sabse tez hai debugging ke liye — JobExplorer code mein use karo)*

### 📖 How It Works
```
Job Status Lifecycle:
  STARTING → STARTED → COMPLETING → COMPLETED ✅
                     → FAILED ❌
                     → STOPPING → STOPPED ⏸️
                     → ABANDONED 🗑️

Three Query Methods:
  SQL (quickest for debugging):
    SELECT status, start_time, end_time, exit_code
    FROM BATCH_JOB_EXECUTION e
    JOIN BATCH_JOB_INSTANCE i ON e.JOB_INSTANCE_ID = i.JOB_INSTANCE_ID
    WHERE i.JOB_NAME = 'paymentJob'
    ORDER BY e.START_TIME DESC

  JobExplorer (programmatic):
    jobExplorer.getJobInstances("paymentJob", 0, 10)
    jobExplorer.getJobExecutions(instance)
    execution.getStatus(), execution.getExitStatus()

  REST (dashboards):
    GET /api/batch/status/paymentJob
    → { "status": "COMPLETED", "duration": "8 min", "records": 10000000 }
```

### 🗣️ Answering Approach
"For quick debugging, I query BATCH_JOB_EXECUTION directly — joining with BATCH_JOB_INSTANCE to filter by job name and ordering by start time for the latest execution. Programmatically, I use JobExplorer — a read-only API that gives me job instances, executions, and step details without touching the database directly. For dashboards, I expose a REST endpoint that returns status, duration, and record counts. In my project, we had a Grafana dashboard pulling from our REST endpoint for real-time visibility across all batch jobs."

### 💻 Code
```java
// SQL — quickest debugging
// SELECT e.STATUS, e.START_TIME, e.END_TIME, e.EXIT_CODE, e.EXIT_MESSAGE
// FROM BATCH_JOB_EXECUTION e
// JOIN BATCH_JOB_INSTANCE i ON e.JOB_INSTANCE_ID = i.JOB_INSTANCE_ID
// WHERE i.JOB_NAME = 'paymentJob'
// ORDER BY e.START_TIME DESC
// FETCH FIRST 5 ROWS ONLY;

// JobExplorer — programmatic
@Service
public class JobStatusService {
    @Autowired private JobExplorer jobExplorer;

    public String getLatestStatus(String jobName) {
        List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 1);
        if (instances.isEmpty()) return "NEVER_RUN";
        
        List<JobExecution> executions = jobExplorer.getJobExecutions(instances.get(0));
        JobExecution latest = executions.get(0);  // most recent execution
        return latest.getStatus().toString();
    }
}

// REST endpoint
@RestController
@RequestMapping("/api/batch")
public class BatchStatusController {
    @Autowired private JobExplorer jobExplorer;

    @GetMapping("/status/{jobName}")
    public Map<String, Object> getStatus(@PathVariable String jobName) {
        List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 1);
        if (instances.isEmpty()) return Map.of("status", "NEVER_RUN");
        
        JobExecution exec = jobExplorer.getJobExecutions(instances.get(0)).get(0);
        return Map.of(
                "status", exec.getStatus().toString(),
                "startTime", exec.getStartTime(),
                "endTime", exec.getEndTime(),
                "exitCode", exec.getExitStatus().getExitCode()
        );
    }
}
```

### ⚠️ Pitfalls / Gotchas
- JobExplorer is read-only — cannot modify execution state (use JobOperator for that) *(JobExplorer sirf padhne ke liye hai — change nahi kar sakta)*
- BATCH_JOB_EXECUTION.STATUS stays STARTED after JVM crash — stale detection needed
- Always query latest execution (ORDER BY START_TIME DESC)
- ExitStatus vs BatchStatus: ExitStatus has custom exit codes, BatchStatus is the enum

### ⚡ Remember
- **SQL** = quickest debugging *(SQL se fastest check hota hai)*
- **JobExplorer** = programmatic read-only API
- **REST** = dashboards and external monitoring
- Status: STARTING → STARTED → COMPLETED/FAILED/STOPPED
- Always check latest execution (not oldest)

### 🔗 Follow-ups
- [Q106 → Debug failed jobs](#q106)
- [Q104 → Monitoring overview](#q104)
- [Q112 → BATCH_JOB_EXECUTION table](#q112)

---

## Q106. How do you debug a failed batch job?

### 📝 One-Liner
Five-step debugging: check job status → find failed step → examine read/write/skip counts → read EXIT_MESSAGE for exception → check ExecutionContext state.

### 🔑 Quick Answer
Systematic 5-step approach using metadata tables: **(1)** Query BATCH_JOB_EXECUTION — was the job FAILED? **(2)** Query BATCH_STEP_EXECUTION — which step failed? **(3)** Check counts — read, write, skip, rollback numbers tell the story. **(4)** Read EXIT_MESSAGE — contains the actual exception stack trace. **(5)** Check EXECUTION_CONTEXT — what was the state at failure (e.g., reader position, last processed ID). This lets you reconstruct exactly what happened: "failed around record 45,000, had 500 skips and 3 rollbacks." *(Paanch steps — status, step, counts, exception, context — poori kahani samajh aati hai)*

### 📖 How It Works
```
5-Step Debugging Flow:

Step 1: Was the job FAILED?
  SELECT status, exit_code, exit_message
  FROM BATCH_JOB_EXECUTION WHERE JOB_INSTANCE_ID = ?
  → STATUS=FAILED, EXIT_CODE=FAILED

Step 2: Which step failed?
  SELECT step_name, status, exit_message
  FROM BATCH_STEP_EXECUTION WHERE JOB_EXECUTION_ID = ?
  → "processPayments" STATUS=FAILED

Step 3: What do the counts tell us?
  SELECT read_count, write_count, skip_count,
         filter_count, rollback_count, commit_count
  FROM BATCH_STEP_EXECUTION WHERE STEP_EXECUTION_ID = ?
  → read=45000, write=44487, skip=500, rollback=3, commit=89

  Story: read 45K, skipped 500 bad records, had 3 chunk rollbacks,
         then hit an error that exceeded skip limit

Step 4: What was the exception?
  EXIT_MESSAGE contains: "Skip limit of 500 exceeded,
  caused by: NumberFormatException: bad amount '-$100'"

Step 5: What was the state at failure?
  SELECT short_context FROM BATCH_STEP_EXECUTION_CONTEXT
  → {"currentPage": 90, "lastProcessedId": 44987}
  → Failed around page 90, last successful ID was 44987
```

### 🗣️ Answering Approach
"I use a systematic 5-step debugging approach. First, I check BATCH_JOB_EXECUTION to confirm the job failed. Second, I find which step failed in BATCH_STEP_EXECUTION. Third, I analyze the counts — read, write, skip, and rollback numbers tell the full story. For example, 45,000 reads with 500 skips and 3 rollbacks means we had 500 bad records and 3 transient failures before the fatal error. Fourth, EXIT_MESSAGE gives me the actual exception. Fifth, EXECUTION_CONTEXT shows the state at failure — which page we were on, the last processed ID. In my project, this approach let us diagnose that a data file had corrupted records starting at line 44,000."

### 💻 Code
```java
// Automated failure diagnosis listener
@Component
public class FailureDiagnosisListener implements JobExecutionListener {

    @Override
    public void afterJob(JobExecution jobExecution) {
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            log.error("=== JOB FAILURE DIAGNOSIS ===");
            log.error("Job: {}", jobExecution.getJobInstance().getJobName());
            log.error("Params: {}", jobExecution.getJobParameters());

            for (StepExecution step : jobExecution.getStepExecutions()) {
                if (step.getStatus() == BatchStatus.FAILED) {
                    log.error("Failed Step: {}", step.getStepName());
                    log.error("  Read:      {}", step.getReadCount());
                    log.error("  Written:   {}", step.getWriteCount());
                    log.error("  Skipped:   {}", step.getSkipCount());
                    log.error("  Filtered:  {}", step.getFilterCount());
                    log.error("  Rollbacks: {}", step.getRollbackCount());
                    log.error("  Commits:   {}", step.getCommitCount());
                    log.error("  Exit:      {}", step.getExitStatus().getExitDescription());
                }
            }

            jobExecution.getAllFailureExceptions().forEach(ex ->
                    log.error("  Exception: {}", ex.getMessage(), ex));
        }
    }
}

// Debug queries — run manually
// -- Step 1: Job status
// SELECT * FROM BATCH_JOB_EXECUTION WHERE STATUS = 'FAILED'
//   ORDER BY START_TIME DESC FETCH FIRST 5 ROWS ONLY;
//
// -- Step 2: Failed step details
// SELECT * FROM BATCH_STEP_EXECUTION
//   WHERE JOB_EXECUTION_ID = ? AND STATUS = 'FAILED';
//
// -- Step 3-4: Counts + exception in EXIT_MESSAGE
//
// -- Step 5: State at failure
// SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT
//   WHERE STEP_EXECUTION_ID = ?;
```

### ⚠️ Pitfalls / Gotchas
- EXIT_MESSAGE can be truncated in some DBs (check column size) *(EXIT_MESSAGE truncate ho sakta hai — column size check karo)*
- Rollback count > 0 AND skip count at limit → skip limit was the cause
- After JVM crash, status stays STARTED (not FAILED) — need crash recovery
- ExecutionContext is JSON in Spring Batch 5, serialized Java in older versions

### 🎯 Tricky Interview Qs

**Q: How do you know exactly which record caused the failure?**
Check skip count (how many were tolerated) + EXIT_MESSAGE (the exception that exceeded skip limit) + reader position from ExecutionContext (which page/offset). Also implement SkipListener to log each skipped record with its ID and error.

**Q: Job shows STARTED but nothing is running — what happened?**
JVM crashed. Status stuck at STARTED (stale). Detect stale executions where START_TIME is old and no heartbeat. Mark FAILED, then restart.

### ⚡ Remember
- 5 steps: status → step → counts → exception → context *(paanch steps se poori kahani milti hai)*
- read - filter - skip = write (math must work)
- EXIT_MESSAGE = actual exception
- ExecutionContext = state at failure point
- Implement FailureDiagnosisListener for auto-logging

### 🔗 Follow-ups
- [Q105 → Check job status](#q105)
- [Q107 → What logs to monitor](#q107)
- [Q120 → Restart after crash](#q120)

---

## Q107. What logs should be monitored?

### 📝 One-Liner
Monitor skip count spikes (data quality), rollback count (transient failures), duration trends (regression), memory usage (leak), and connection pool (exhaustion).

### 🔑 Quick Answer
Three alert tiers: **CRITICAL** (immediate action): job FAILED, OutOfMemoryError, connection pool exhausted. **WARNING** (investigate): skip count spike, duration > 2× average, rollback count > 0, memory growth. **INFO** (capacity planning): throughput trends, record count changes, seasonal patterns. The most important early warning: **skip count spike** — means incoming data quality degraded. *(Skip count badha = data quality kharab — sabse pehle isko pakdo)*

### 📖 How It Works
```
Alert Tiers:

CRITICAL (immediate):
  🔴 Job FAILED → page on-call
  🔴 OutOfMemoryError → restart + investigate
  🔴 Connection pool exhausted → check pool size, leaks
  🔴 Job hung (STARTED > 2× expected) → kill or investigate

WARNING (investigate within hours):
  🟡 Skip count spike (normal: 5, today: 500) → data quality issue
  🟡 Duration > 2× average → performance regression
  🟡 Rollback count > 0 → transient DB/network failures
  🟡 Memory growth pattern → JPA cache leak
  🟡 Thread count spike → partition/thread leak

INFO (capacity planning):
  🟢 Throughput trending down → infrastructure scaling needed
  🟢 Record count growing → plan for more partitions
  🟢 Seasonal patterns → adjust scheduling windows
```

### 🗣️ Answering Approach
"I set up three alert tiers. Critical alerts page on-call for job failures, OOM errors, and stuck jobs. Warning alerts trigger investigation for skip count spikes — which is the most valuable early warning because it means data quality degraded — duration exceeding twice the average, and any rollback counts. Info-level dashboards track throughput and record count trends for capacity planning. In my project, a skip count alert caught a data corruption issue from an upstream system before it became a production incident — the skip count jumped from the normal 5 to 500, and we traced it to malformed records from a partner API change."

### 💻 Code
```java
// Skip count alert listener
@Component
public class SkipAlertListener implements StepExecutionListener {
    private static final int SKIP_THRESHOLD = 50;

    @Override
    public ExitStatus afterStep(StepExecution stepExecution) {
        int skipCount = stepExecution.getSkipCount();
        if (skipCount > SKIP_THRESHOLD) {
            log.warn("HIGH SKIP COUNT ALERT: Step '{}' skipped {} records (threshold: {})",
                    stepExecution.getStepName(), skipCount, SKIP_THRESHOLD);
            // Send alert to monitoring system
            alertService.warn("Batch skip count spike",
                    "Step %s skipped %d records".formatted(
                            stepExecution.getStepName(), skipCount));
        }
        return stepExecution.getExitStatus();
    }
}

// Logging configuration for batch
// logging:
//   level:
//     org.springframework.batch: INFO       # framework logs
//     com.myapp.batch: INFO                  # application batch logs
//     org.springframework.batch.core.step.item.ChunkOrientedTasklet: DEBUG  # chunk details
//     com.zaxxer.hikari: WARN               # connection pool warnings
```

### ⚠️ Pitfalls / Gotchas
- Skip count = 0 might hide issues if skip is disabled — check if faultTolerant() is configured *(skip disable hai toh count 0 dikhega lekin problem hai toh job fail hoga)*
- Duration comparison needs baseline — first establish "normal" metrics
- Memory growth is subtle — compare start vs end of job, not just peak
- Connection pool warnings often come too late — monitor active connections proactively

### ⚡ Remember
- **Skip count spike** = data quality issue (most important early warning) *(skip count badha = upstream data kharab)*
- Rollback > 0 = transient failures
- Duration trending up = regression
- Memory growth = JPA/cache leak
- Three tiers: CRITICAL, WARNING, INFO

### 🔗 Follow-ups
- [Q104 → Monitoring overview](#q104)
- [Q106 → Debug failed jobs](#q106)
- [Q108 → Track job performance](#q108)

---

## Q108. How do you track job performance?

### 📝 One-Liner
Micrometer exposes metrics → Prometheus collects → Grafana dashboards with alerts on duration, throughput, and skip rate.

### 🔑 Quick Answer
**Micrometer** (Spring Boot auto-configured) exposes batch metrics: job duration, record counts, throughput (records/sec). **Prometheus** scrapes these metrics. **Grafana** displays dashboards and fires alerts. Key metrics: **(1) Duration** (timer per job/step). **(2) Throughput** (records/second). **(3) Quality** (skip count, skip rate). **(4) Resources** (JVM memory, HikariCP connections). Alert when: duration > 2× average, skip rate > 1%, memory > 80% heap. *(Micrometer → Prometheus → Grafana — production ka standard monitoring stack)*

### 📖 How It Works
```
Metrics Pipeline:

Spring Batch + Micrometer (auto-configured):
  spring.batch.job       → timer (duration per job)
  spring.batch.step      → timer (duration per step)
  spring.batch.item.read → counter (total items read)
  spring.batch.chunk     → timer (per-chunk duration)

  → /actuator/prometheus endpoint

Prometheus (scrapes every 15s):
  → Stores time-series data

Grafana (dashboards + alerts):
  Dashboard 1: Job Overview
    - Job status (COMPLETED/FAILED pie chart)
    - Duration trend (line graph)
    - Throughput (records/second)

  Dashboard 2: Step Details
    - Per-step duration
    - Read/Write/Skip counts
    - Commit/Rollback counts

  Dashboard 3: Resources
    - JVM heap usage
    - HikariCP active connections
    - Thread count

  Alerts:
    - Duration > 2× average → WARNING
    - Skip rate > 1% → WARNING
    - Job FAILED → CRITICAL
```

### 🗣️ Answering Approach
"I use the Micrometer-Prometheus-Grafana stack for performance tracking. Spring Boot auto-configures Micrometer with Spring Batch, exposing job and step timers, item counters, and chunk metrics via the Prometheus endpoint. Prometheus scrapes these every 15 seconds, and Grafana displays dashboards for job duration trends, throughput, skip rates, and resource usage. We alert when job duration exceeds twice the average or when skip rate exceeds 1%. In my project, this setup helped us catch a gradual performance regression — our daily job's duration increased from 8 to 12 minutes over two weeks, which we traced to a missing index on a newly added column."

### 💻 Code
```java
// Custom metrics listener (beyond auto-configured metrics)
@Component
public class BatchMetricsListener implements JobExecutionListener {
    private final MeterRegistry registry;

    public BatchMetricsListener(MeterRegistry registry) {
        this.registry = registry;
    }

    @Override
    public void afterJob(JobExecution jobExecution) {
        String jobName = jobExecution.getJobInstance().getJobName();
        String status = jobExecution.getStatus().toString();

        // Job completion counter by status
        registry.counter("batch.job.completions",
                "job", jobName, "status", status).increment();

        // Per-step metrics
        for (StepExecution step : jobExecution.getStepExecutions()) {
            String stepName = step.getStepName();
            long duration = Duration.between(step.getStartTime(), step.getEndTime()).getSeconds();

            registry.gauge("batch.step.throughput",
                    Tags.of("job", jobName, "step", stepName),
                    (double) step.getWriteCount() / Math.max(duration, 1));

            registry.counter("batch.step.skips",
                    "job", jobName, "step", stepName)
                    .increment(step.getSkipCount());
        }
    }
}

// Dependencies (pom.xml):
// spring-boot-starter-actuator
// micrometer-registry-prometheus

// application.yml:
// management:
//   endpoints:
//     web:
//       exposure:
//         include: prometheus, health, metrics
//   metrics:
//     tags:
//       application: payment-batch
//       environment: production
```

### ⚠️ Pitfalls / Gotchas
- Auto-configured metrics only work with Spring Boot actuator dependency *(actuator dependency nahi hai toh metrics nahi aayenge)*
- Prometheus endpoint needs to be secured in production (don't expose publicly)
- Custom metrics: use Tags for dimensions, not metric name suffixes
- High-cardinality tags (like user IDs) can crash Prometheus

### ⚡ Remember
- **Micrometer → Prometheus → Grafana** = production standard *(ye teen milke monitoring banate hain)*
- Auto-configured: job/step timers, item counters
- Key alerts: duration > 2× average, skip > 1%, FAILED
- Custom metrics via MeterRegistry in listeners
- Secure the /actuator/prometheus endpoint

### 🔗 Follow-ups
- [Q104 → Monitoring overview](#q104)
- [Q109 → Find slow steps](#q109)
- [Q107 → What logs to monitor](#q107)

---

## Q109. How do you find slow steps?

### 📝 One-Liner
Query BATCH_STEP_EXECUTION for duration per step, calculate records/second, and use ChunkListener for chunk-level timing to find individual slow chunks.

### 🔑 Quick Answer
**Step-level**: query BATCH_STEP_EXECUTION — calculate duration and records/second per step. The step with lowest throughput is the bottleneck. **Chunk-level**: implement ChunkListener that times each chunk — find individual slow chunks (e.g., Chunk 45 took 15 sec when normal is 200ms). Causes of slow chunks: GC pauses, DB locks, network timeouts, data skew. *(Step level pe sabse slow step dhundho, phir chunk level pe exactly kahan slow hua — ChunkListener se pata chalta hai)*

### 📖 How It Works
```
Step-Level Analysis (SQL):
  Step Name         | Duration | Records | Records/sec
  ─────────────────┼──────────┼─────────┼───────────
  readOrders        | 30 sec   | 100K    | 3,333/sec
  processPayments   | 180 sec  | 100K    | 555/sec    ← BOTTLENECK
  generateReport    | 5 sec    | 1       | N/A

  → processPayments is 6× slower per record → investigate

Chunk-Level Analysis (ChunkListener):
  Chunk 1:  200ms ✅
  Chunk 2:  180ms ✅
  Chunk 44: 210ms ✅
  Chunk 45: 15000ms ⚠️ ← SLOW CHUNK!
  Chunk 46: 190ms ✅
  
  → Chunk 45 is 75× slower → investigate
  → Common causes: GC pause, DB lock wait, network timeout,
     data requiring expensive external API call
```

### 🗣️ Answering Approach
"I find slow steps at two levels. First, I query BATCH_STEP_EXECUTION and calculate records per second for each step — the step with the lowest throughput is the bottleneck. Second, I implement a ChunkListener that logs the duration of every chunk. This helps me find individual slow chunks — for example, in our payment job, most chunks processed in 200ms but every 50th chunk took 15 seconds. We traced it to garbage collection pauses caused by JPA persistence context growth. After adding EntityManager.clear() per chunk, all chunks processed consistently in 200ms."

### 💻 Code
```java
// ChunkListener for timing individual chunks
@Component
public class ChunkTimingListener implements ChunkListener {
    private static final long SLOW_THRESHOLD_MS = 5000;
    private long chunkStartTime;
    private int chunkNumber = 0;

    @Override
    public void beforeChunk(ChunkContext context) {
        chunkStartTime = System.currentTimeMillis();
        chunkNumber++;
    }

    @Override
    public void afterChunk(ChunkContext context) {
        long duration = System.currentTimeMillis() - chunkStartTime;
        String stepName = context.getStepContext().getStepName();
        
        if (duration > SLOW_THRESHOLD_MS) {
            log.warn("SLOW CHUNK: Step '{}' chunk {} took {} ms (threshold: {} ms)",
                    stepName, chunkNumber, duration, SLOW_THRESHOLD_MS);
        } else {
            log.debug("Step '{}' chunk {} took {} ms", stepName, chunkNumber, duration);
        }
    }

    @Override
    public void afterChunkError(ChunkContext context) {
        long duration = System.currentTimeMillis() - chunkStartTime;
        log.error("CHUNK ERROR: Step '{}' chunk {} failed after {} ms",
                context.getStepContext().getStepName(), chunkNumber, duration);
    }
}

// Step-level SQL analysis
// SELECT step_name,
//        TIMESTAMPDIFF(SECOND, START_TIME, END_TIME) AS duration_sec,
//        READ_COUNT AS records,
//        ROUND(READ_COUNT / GREATEST(TIMESTAMPDIFF(SECOND, START_TIME, END_TIME), 1)) AS rec_per_sec
// FROM BATCH_STEP_EXECUTION
// WHERE JOB_EXECUTION_ID = ?
// ORDER BY duration_sec DESC;
```

### ⚠️ Pitfalls / Gotchas
- Slow chunks might be GC pauses, not code issues — check GC logs *(slow chunk = GC pause bhi ho sakta hai — logs check karo)*
- DB lock waits don't throw exceptions — chunk just takes longer
- First chunk is always slower (JIT warmup, lazy initialization)
- Network timeouts in processor cause entire chunk to be slow

### 🎯 Tricky Interview Qs

**Q: Most chunks are 200ms but every 50th is 15 seconds — what's wrong?**
Likely GC pause caused by growing memory (JPA persistence context). Fix: clear EntityManager per chunk. Or check if every 50th chunk triggers a different code path (e.g., batch of records needing external API call).

**Q: How do you find which item in a chunk caused slowness?**
Add logging in the processor with System.currentTimeMillis() before/after each item. Or use Micrometer timer around the process() method to track per-item timing.

### ⚡ Remember
- Step level: SQL query → records/sec per step *(konsa step slow hai — SQL se pata karo)*
- Chunk level: ChunkListener → individual slow chunks
- Slow chunk causes: GC, DB lock, network, data skew
- Alert when chunk > 5× average duration
- First chunk always slower (warmup)

### 🔗 Follow-ups
- [Q104 → Monitoring overview](#q104)
- [Q108 → Track performance metrics](#q108)
- [Q98 → Memory issues](#q98)
