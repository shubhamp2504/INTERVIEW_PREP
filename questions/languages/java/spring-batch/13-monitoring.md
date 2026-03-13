
# 🟡 Spring Batch — Monitoring & Debugging Questions (104-109)

[![Questions](https://img.shields.io/badge/Questions-6-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)


---

<a id="q1"></a>

## Q104. ❓ How do you monitor Spring Batch jobs?

🔖 **Tags:** `#spring-batch` `#monitoring` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Method | Tool | Detail |
|--------|------|--------|
| **Metadata Tables** | SQL queries | Query BATCH_* tables for status, counts |
| **Spring Boot Actuator** | `/actuator/health`, `/actuator/metrics` | Health checks, job metrics |
| **JobExecutionListener** | Custom listener | Log start/end/status of each job |
| **Spring Batch Admin** | Web UI (deprecated) | Dashboard for job monitoring |
| **Spring Cloud Data Flow** | Web UI | Modern orchestration + monitoring |
| **Micrometer + Prometheus** | Grafana dashboard | Production-grade metrics |
| **ELK Stack** | Kibana | Log aggregation and analysis |

### Custom Monitoring Listener:
```java
@Component
public class JobMonitoringListener implements JobExecutionListener {
    
    @Override
    public void beforeJob(JobExecution jobExecution) {
        log.info("===== JOB STARTED: {} =====", jobExecution.getJobInstance().getJobName());
        log.info("Parameters: {}", jobExecution.getJobParameters());
    }
    
    @Override
    public void afterJob(JobExecution jobExecution) {
        long duration = Duration.between(
            jobExecution.getStartTime(), 
            jobExecution.getEndTime()
        ).toSeconds();
        
        log.info("===== JOB FINISHED =====");
        log.info("Status: {}", jobExecution.getStatus());
        log.info("Duration: {} seconds", duration);
        
        jobExecution.getStepExecutions().forEach(step -> {
            log.info("  Step: {} | Status: {} | Read: {} | Written: {} | Skipped: {}",
                step.getStepName(),
                step.getStatus(),
                step.getReadCount(),
                step.getWriteCount(),
                step.getSkipCount()
            );
        });
        
        if (jobExecution.getStatus() == BatchStatus.FAILED) {
            jobExecution.getAllFailureExceptions().forEach(ex ->
                log.error("  Failure: {}", ex.getMessage())
            );
        }
    }
}
```

---

<a id="q2"></a>

## Q105. ❓ How do you check job status?

🔖 **Tags:** `#spring-batch` `#job-status`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

### Method 1: SQL Query
```sql
SELECT je.JOB_EXECUTION_ID, ji.JOB_NAME, je.STATUS, je.EXIT_CODE,
       je.START_TIME, je.END_TIME
FROM BATCH_JOB_EXECUTION je
JOIN BATCH_JOB_INSTANCE ji ON je.JOB_INSTANCE_ID = ji.JOB_INSTANCE_ID
ORDER BY je.CREATE_TIME DESC
LIMIT 10;
```

### Method 2: Programmatic
```java
@Autowired private JobExplorer jobExplorer;

public void checkJobStatus(String jobName) {
    List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 5);
    for (JobInstance instance : instances) {
        List<JobExecution> executions = jobExplorer.getJobExecutions(instance);
        executions.forEach(exec -> 
            log.info("Instance: {} | Execution: {} | Status: {}", 
                instance.getId(), exec.getId(), exec.getStatus())
        );
    }
}
```

### Method 3: REST Endpoint
```java
@GetMapping("/api/jobs/{jobName}/status")
public ResponseEntity<?> getJobStatus(@PathVariable String jobName) {
    List<JobInstance> instances = jobExplorer.getJobInstances(jobName, 0, 1);
    if (instances.isEmpty()) return ResponseEntity.notFound().build();
    
    JobExecution latest = jobExplorer.getJobExecutions(instances.get(0)).get(0);
    return ResponseEntity.ok(Map.of(
        "status", latest.getStatus().toString(),
        "startTime", latest.getStartTime(),
        "endTime", latest.getEndTime()
    ));
}
```

---

<a id="q3"></a>

## Q106. ❓ How do you debug a failed batch job?

🔖 **Tags:** `#spring-batch` `#debugging` `#troubleshooting`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

### Debugging Checklist:
```
1. Check Job Status:
   SELECT * FROM BATCH_JOB_EXECUTION WHERE STATUS = 'FAILED';

2. Check Which Step Failed:
   SELECT * FROM BATCH_STEP_EXECUTION 
   WHERE JOB_EXECUTION_ID = <failed_id> AND STATUS = 'FAILED';

3. Check Execution Context (state at failure):
   SELECT * FROM BATCH_STEP_EXECUTION_CONTEXT 
   WHERE STEP_EXECUTION_ID = <failed_step_id>;

4. Check Read/Write Counts:
   SELECT READ_COUNT, WRITE_COUNT, SKIP_COUNT, ROLLBACK_COUNT
   FROM BATCH_STEP_EXECUTION WHERE STEP_EXECUTION_ID = <id>;

5. Check Exception in Logs:
   Search logs for the failed step name

6. Check Exit Description:
   SELECT EXIT_MESSAGE FROM BATCH_JOB_EXECUTION WHERE JOB_EXECUTION_ID = <id>;
```

### Programmatic Debugging:
```java
@AfterJob
public void debugFailedJob(JobExecution jobExecution) {
    if (jobExecution.getStatus() == BatchStatus.FAILED) {
        log.error("=== FAILURE DIAGNOSIS ===");
        jobExecution.getStepExecutions().forEach(step -> {
            if (step.getStatus() == BatchStatus.FAILED) {
                log.error("Failed Step: {}", step.getStepName());
                log.error("Read: {}, Written: {}, Skipped: {}", 
                    step.getReadCount(), step.getWriteCount(), step.getSkipCount());
                log.error("Rollbacks: {}", step.getRollbackCount());
                log.error("Exit: {}", step.getExitStatus().getExitDescription());
            }
        });
        jobExecution.getAllFailureExceptions().forEach(ex -> 
            log.error("Exception: ", ex));
    }
}
```

---

<a id="q4"></a>

## Q107. ❓ What logs should be monitored?

🔖 **Tags:** `#spring-batch` `#logging` `#monitoring`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| What to Monitor | Why |
|----------------|-----|
| Job start/end time | Track duration trends |
| Read/Write counts | Detect data volume changes |
| Skip count | Sudden increase = data quality issue |
| Rollback count | Should be 0 or very low |
| Memory usage | Detect leaks |
| Database connection pool | Detect exhaustion |
| Chunk processing time | Find slow chunks |

```yaml
# Recommended logging config
logging:
  level:
    org.springframework.batch: INFO        # General batch info
    org.springframework.batch.core: INFO   # Job/Step lifecycle
    org.springframework.batch.item: DEBUG  # Reader/Writer details (dev only)
    org.springframework.jdbc: WARN         # Reduce JDBC noise
```

---

<a id="q5"></a>

## Q108. ❓ How do you track job performance?

🔖 **Tags:** `#spring-batch` `#performance` `#metrics`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```java
// Micrometer metrics for Prometheus/Grafana
@Component
public class BatchMetrics implements JobExecutionListener {
    
    private final MeterRegistry meterRegistry;
    
    @Override
    public void afterJob(JobExecution exec) {
        String jobName = exec.getJobInstance().getJobName();
        long duration = Duration.between(exec.getStartTime(), exec.getEndTime()).toMillis();
        
        // Record duration
        meterRegistry.timer("batch.job.duration", "job", jobName)
            .record(duration, TimeUnit.MILLISECONDS);
        
        // Record counts per step
        exec.getStepExecutions().forEach(step -> {
            meterRegistry.gauge("batch.step.read_count", 
                Tags.of("step", step.getStepName()), step.getReadCount());
            meterRegistry.gauge("batch.step.write_count",
                Tags.of("step", step.getStepName()), step.getWriteCount());
        });
        
        // Record status
        meterRegistry.counter("batch.job.status", 
            "job", jobName, "status", exec.getStatus().toString()).increment();
    }
}
```

---

<a id="q6"></a>

## Q109. ❓ How do you find slow steps?

🔖 **Tags:** `#spring-batch` `#performance` `#slow-steps`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```sql
-- Find slowest steps
SELECT STEP_NAME, 
       TIMESTAMPDIFF(SECOND, START_TIME, END_TIME) AS duration_seconds,
       READ_COUNT, WRITE_COUNT,
       WRITE_COUNT / NULLIF(TIMESTAMPDIFF(SECOND, START_TIME, END_TIME), 0) AS records_per_second
FROM BATCH_STEP_EXECUTION
WHERE JOB_EXECUTION_ID = <id>
ORDER BY duration_seconds DESC;
```

### ChunkListener for Chunk-Level Timing:
```java
@Component
public class ChunkTimingListener implements ChunkListener {
    private long chunkStart;
    
    @Override
    public void beforeChunk(ChunkContext context) {
        chunkStart = System.currentTimeMillis();
    }
    
    @Override
    public void afterChunk(ChunkContext context) {
        long duration = System.currentTimeMillis() - chunkStart;
        if (duration > 5000) {  // Alert if chunk takes > 5 seconds
            log.warn("SLOW CHUNK: Step={}, Duration={}ms", 
                context.getStepContext().getStepName(), duration);
        }
    }
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Scheduling](./12-scheduling.md) | [Next: DB & Metadata →](./14-database-metadata.md)
]]>
