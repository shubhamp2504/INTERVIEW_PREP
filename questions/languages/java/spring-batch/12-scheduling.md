<![CDATA[<div align="center">

# 🟡 Spring Batch — Scheduling Questions (99-103)

[![Questions](https://img.shields.io/badge/Questions-5-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)

</div>

---

<a id="q1"></a>
## Q99. ❓ How do you schedule Spring Batch jobs?

🔖 **Tags:** `#spring-batch` `#scheduling` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Spring Batch does **NOT** have built-in scheduling. Use external schedulers:

| Scheduler | How |
|-----------|-----|
| **Spring @Scheduled** | Built-in, simplest |
| **Quartz** | Enterprise-grade, persistent, clustered |
| **Cron (OS)** | Linux crontab / Windows Task Scheduler |
| **Kubernetes CronJob** | Cloud-native |
| **Spring Cloud Data Flow** | Full batch orchestration platform |

### Spring @Scheduled (Simplest):
```java
@Configuration
@EnableScheduling
public class BatchScheduler {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job dailyReportJob;
    
    @Scheduled(cron = "0 0 2 * * ?")  // Every day at 2 AM
    public void runDailyReport() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLong("timestamp", System.currentTimeMillis())
                .toJobParameters();
        jobLauncher.run(dailyReportJob, params);
    }
}
```

---

<a id="q2"></a>
## Q100. ❓ How do you trigger jobs automatically?

🔖 **Tags:** `#spring-batch` `#trigger` `#automation`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Trigger | Use Case | How |
|---------|----------|-----|
| **Time-based** | Daily/hourly reports | @Scheduled / Cron |
| **Event-based** | File uploaded, message received | Listener/Watcher |
| **API-based** | External system trigger | REST endpoint |
| **On startup** | Run once when app starts | CommandLineRunner |

### REST API Trigger:
```java
@RestController
@RequestMapping("/api/jobs")
public class JobController {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job importJob;
    
    @PostMapping("/import")
    public ResponseEntity<String> triggerImport(@RequestParam String fileName) 
            throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addString("file", fileName)
                .addLong("time", System.currentTimeMillis())
                .toJobParameters();
        
        JobExecution exec = jobLauncher.run(importJob, params);
        return ResponseEntity.ok("Job started: " + exec.getId());
    }
}
```

### File Watcher Trigger:
```java
@Component
public class FileWatcher {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job fileProcessJob;
    
    @Scheduled(fixedDelay = 10000)  // Check every 10 seconds
    public void watchForFiles() throws Exception {
        File dir = new File("/incoming/");
        File[] files = dir.listFiles((d, name) -> name.endsWith(".csv"));
        
        if (files != null) {
            for (File file : files) {
                JobParameters params = new JobParametersBuilder()
                        .addString("file", file.getAbsolutePath())
                        .toJobParameters();
                jobLauncher.run(fileProcessJob, params);
            }
        }
    }
}
```

---

<a id="q3"></a>
## Q101. ❓ Can Spring Batch run with Quartz?

🔖 **Tags:** `#spring-batch` `#quartz` `#scheduling`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** Quartz is recommended for production scheduling — it supports clustering, persistence, and complex schedules.

```java
// Quartz Job that triggers Spring Batch Job
public class BatchQuartzJob extends QuartzJobBean {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job springBatchJob;
    
    @Override
    protected void executeInternal(JobExecutionContext context) {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLong("timestamp", System.currentTimeMillis())
                    .toJobParameters();
            jobLauncher.run(springBatchJob, params);
        } catch (Exception e) {
            throw new RuntimeException("Batch job failed", e);
        }
    }
}

// Quartz Configuration
@Bean
public JobDetail batchJobDetail() {
    return JobBuilder.newJob(BatchQuartzJob.class)
            .withIdentity("dailyBatchJob")
            .storeDurably()
            .build();
}

@Bean
public Trigger batchTrigger() {
    return TriggerBuilder.newTrigger()
            .forJob(batchJobDetail())
            .withSchedule(CronScheduleBuilder.cronSchedule("0 0 2 * * ?"))
            .build();
}
```

---

<a id="q4"></a>
## Q102. ❓ Can Spring Batch run with Cron scheduling?

🔖 **Tags:** `#spring-batch` `#cron`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**Yes!** Two approaches:

### Approach 1: Spring @Scheduled with Cron Expression
```java
@Scheduled(cron = "0 0 2 * * MON-FRI")  // Weekdays at 2 AM
public void runWeekdayJob() throws Exception {
    jobLauncher.run(job, params);
}
```

### Common Cron Expressions:
| Expression | Schedule |
|-----------|---------|
| `0 0 * * * ?` | Every hour |
| `0 0 2 * * ?` | Daily at 2 AM |
| `0 0 2 * * MON-FRI` | Weekdays at 2 AM |
| `0 0 0 1 * ?` | First day of month |
| `0 */30 * * * ?` | Every 30 minutes |

### Approach 2: OS Level Cron (run as JAR)
```bash
# Linux crontab
0 2 * * * java -jar batch-app.jar --spring.batch.job.name=dailyReport

# Windows Task Scheduler
schtasks /create /tn "DailyBatch" /tr "java -jar batch-app.jar" /sc daily /st 02:00
```

---

<a id="q5"></a>
## Q103. ❓ How do you run jobs on application startup?

🔖 **Tags:** `#spring-batch` `#startup`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

### Method 1: Spring Boot Auto-Run (Default)
```yaml
# application.yml — runs ALL jobs on startup (default behavior)
spring:
  batch:
    job:
      enabled: true           # default: true
      name: specificJobName   # run only this job
```

### Method 2: CommandLineRunner
```java
@Component
public class JobRunner implements CommandLineRunner {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job myJob;
    
    @Override
    public void run(String... args) throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLong("startupTime", System.currentTimeMillis())
                .toJobParameters();
        jobLauncher.run(myJob, params);
    }
}
```

### Method 3: Disable Auto-Run (Trigger Manually)
```yaml
spring:
  batch:
    job:
      enabled: false   # Don't run on startup — trigger via REST/scheduler
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Performance](./11-performance.md) | [Next: Monitoring →](./13-monitoring.md)
]]>