# 🟡 Spring Batch — Scheduling (Q99–Q103)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q99"></a>

## Q99. How do you schedule Spring Batch jobs?

### 🔑 Quick Answer

> Spring Batch has **no built-in scheduler**. Use external schedulers: **@Scheduled** (simplest), **Quartz** (production), **Kubernetes CronJob** (cloud), or **OS cron** (traditional).

### 📖 Step-by-Step Explanation

**Step 1 — Why no built-in scheduler:**

```
Spring Batch = Processing Framework (read → process → write)
It does NOT decide WHEN to run.

You need something external to say: "Run this job NOW"

Options:
  Spring @Scheduled  → Simple, in-app, no persistence
  Quartz             → Enterprise, clustered, persistent ⭐
  Kubernetes CronJob → Cloud-native, container based
  OS Cron            → Traditional, runs JAR as command
  Spring Cloud DF    → Full orchestration platform
```

**Step 2 — Comparison:**

| Scheduler | Clustering | Persistence | Setup | Best For |
|-----------|-----------|------------|-------|----------|
| @Scheduled | ❌ | ❌ | 2 lines | Dev, simple apps |
| Quartz | ✅ | ✅ | Medium | Production ⭐ |
| K8s CronJob | ✅ | ✅ (K8s) | Medium | Cloud apps |
| OS Cron | ❌ | ❌ | 1 line | Legacy, simple |

### 💻 Code Example

```java
// Simplest: @Scheduled
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

### 🗣️ How to Explain in Interview

> *"Spring Batch intentionally doesn't include scheduling — it's a processing framework, not a scheduler. For scheduling, I use @Scheduled for simple apps — just add EnableScheduling and a cron expression. For production, I use Quartz because it supports clustering — if you have 3 instances, only one runs the job, and if it goes down, another takes over. Quartz also persists schedule state to the database, so it survives restarts. In cloud environments, Kubernetes CronJob is the standard approach."*

### ⚡ Key Points to Remember

1. Spring Batch = **no built-in scheduler**
2. **@Scheduled** = simplest (but no clustering/persistence)
3. **Quartz** = production standard (clustered, persistent)
4. **Kubernetes CronJob** = cloud-native approach
5. Always pass **unique JobParameters** (timestamp) to avoid duplicate detection

---

<a id="q100"></a>

## Q100. How do you trigger jobs automatically?

### 🔑 Quick Answer

> Four trigger types: **Time-based** (@Scheduled/Cron), **Event-based** (file watcher, message listener), **API-based** (REST endpoint), **Startup** (CommandLineRunner).

### 📖 Step-by-Step Explanation

**Step 1 — Trigger types:**

```
1. TIME-BASED (most common)
   "Run every day at 2 AM"
   → @Scheduled, Quartz, Cron

2. EVENT-BASED
   "Run when a file arrives in /incoming/"
   → File watcher, Kafka listener, JMS listener

3. API-BASED
   "Run when someone calls POST /api/jobs/import"
   → REST controller, actuator endpoint

4. STARTUP
   "Run once when application starts"
   → CommandLineRunner, spring.batch.job.enabled=true
```

### 💻 Code Examples

```java
// 1. REST API trigger
@RestController
@RequestMapping("/api/jobs")
public class JobController {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job importJob;
    
    @PostMapping("/import")
    public ResponseEntity<String> trigger(@RequestParam String fileName)
            throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addString("file", fileName)
                .addLong("time", System.currentTimeMillis())
                .toJobParameters();
        
        JobExecution exec = jobLauncher.run(importJob, params);
        return ResponseEntity.ok("Job started: " + exec.getId());
    }
}

// 2. File watcher trigger
@Component
public class FileWatcher {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job fileJob;
    
    @Scheduled(fixedDelay = 10000)  // Check every 10 seconds
    public void watchForFiles() throws Exception {
        Path dir = Paths.get("/incoming/");
        try (DirectoryStream<Path> stream = 
                Files.newDirectoryStream(dir, "*.csv")) {
            for (Path file : stream) {
                JobParameters params = new JobParametersBuilder()
                        .addString("file", file.toString())
                        .toJobParameters();
                jobLauncher.run(fileJob, params);
            }
        }
    }
}

// 3. Startup trigger
@Component
public class StartupJobRunner implements CommandLineRunner {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job migrationJob;
    
    @Override
    public void run(String... args) throws Exception {
        jobLauncher.run(migrationJob, new JobParametersBuilder()
                .addLong("time", System.currentTimeMillis())
                .toJobParameters());
    }
}
```

### 🗣️ How to Explain in Interview

> *"I trigger batch jobs four ways depending on the use case. Time-based with @Scheduled or Quartz for recurring jobs like daily reports. Event-based — I use a file watcher that checks an incoming directory every 10 seconds and triggers the import job when CSV files appear. API-based — a REST endpoint so external systems can trigger imports on demand. And startup — using CommandLineRunner for one-time migration jobs. The key in all cases is passing unique JobParameters — usually a timestamp — so Spring Batch doesn't reject the launch as a duplicate."*

### ⚡ Key Points to Remember

1. **Time**: @Scheduled, Quartz, Cron
2. **Event**: File watcher, message listener
3. **API**: REST controller (POST endpoint)
4. **Startup**: CommandLineRunner or spring.batch.job.enabled
5. Always include **unique parameter** (timestamp) in every launch

---

<a id="q101"></a>

## Q101. Can Spring Batch run with Quartz?

### 🔑 Quick Answer

> **Yes, and it's the production standard.** Create a QuartzJobBean that calls JobLauncher.run(). Quartz handles scheduling, clustering, and persistence — Spring Batch handles processing.

### 📖 Step-by-Step Explanation

**Step 1 — Why Quartz for production:**

```
@Scheduled problems in production:
  ❌ No clustering — all 3 instances run the job simultaneously
  ❌ No persistence — schedule lost on restart
  ❌ No retry — if app was down during scheduled time, job is skipped

Quartz solves all:
  ✅ Clustering — only 1 of 3 instances runs the job
  ✅ Persistence — schedule saved in DB (QRTZ_ tables)
  ✅ Misfire handling — runs missed jobs when app comes back
  ✅ Dynamic scheduling — change schedule without restart
```

**Step 2 — How it connects:**

```
Quartz                          Spring Batch
──────                          ────────────
QuartzJobBean                   
  ↓ (trigger fires)            
  executeInternal()             
    ↓                           
    jobLauncher.run(job, params)
                                ↓
                                JobExecution created
                                Steps execute
                                Results saved to BATCH_ tables
```

### 💻 Code Example

```java
// 1. Quartz Job wraps Spring Batch Job
public class BatchQuartzJob extends QuartzJobBean {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job dailyReportJob;
    
    @Override
    protected void executeInternal(JobExecutionContext context) {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLong("timestamp", System.currentTimeMillis())
                    .toJobParameters();
            jobLauncher.run(dailyReportJob, params);
        } catch (Exception e) {
            throw new RuntimeException("Batch job failed", e);
        }
    }
}

// 2. Quartz configuration
@Configuration
public class QuartzConfig {
    
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
                .withIdentity("dailyBatchTrigger")
                .withSchedule(CronScheduleBuilder
                        .cronSchedule("0 0 2 * * ?")  // Daily at 2 AM
                        .withMisfireHandlingInstructionFireAndProceed())
                .build();
    }
}
```

```yaml
# application.yml — Quartz persistence
spring:
  quartz:
    job-store-type: jdbc          # Store in DB (not memory)
    properties:
      org.quartz.jobStore.isClustered: true
      org.quartz.jobStore.clusterCheckinInterval: 15000
```

### 🗣️ How to Explain in Interview

> *"Yes, Quartz and Spring Batch work well together. I create a QuartzJobBean that calls jobLauncher.run() inside executeInternal(). Quartz handles the scheduling — when to run, clustering, misfire recovery — and Spring Batch handles the actual processing. In production, I configure Quartz with JDBC job store and clustering enabled, so if I have 3 application instances, only one runs the batch job. If the running instance goes down, another picks it up. The misfire policy handles cases where the app was down at the scheduled time."*

### ⚡ Key Points to Remember

1. QuartzJobBean calls **jobLauncher.run()** inside executeInternal()
2. Quartz = **clustering** (only 1 instance runs the job)
3. **JDBC job store** = schedule persists across restarts
4. **Misfire handling** = catches up on missed schedules
5. Quartz manages WHEN, Spring Batch manages WHAT

---

<a id="q102"></a>

## Q102. Can Spring Batch run with Cron scheduling?

### 🔑 Quick Answer

> **Yes, two approaches:** (1) Spring's **@Scheduled(cron = "...")** within the app, (2) **OS-level cron** (Linux crontab / Windows Task Scheduler) that runs the app as a JAR.

### 📖 Step-by-Step Explanation

```
Approach 1: IN-APP CRON (@Scheduled)
  App is long-running (web server), cron triggers job inside the app
  ✅ Simple, no external setup
  ❌ No clustering, no persistence

Approach 2: OS CRON
  App starts, runs job, exits
  ✅ No long-running process
  ✅ Clean resource usage
  ❌ No Spring-level monitoring
  ❌ OS-dependent
```

### 💻 Code Examples

```java
// Approach 1: Spring @Scheduled
@Scheduled(cron = "0 0 2 * * MON-FRI")  // Weekdays at 2 AM
public void runWeekdayJob() throws Exception {
    jobLauncher.run(weekdayJob, new JobParametersBuilder()
            .addLong("time", System.currentTimeMillis())
            .toJobParameters());
}
```

```bash
# Approach 2: Linux crontab
0 2 * * * java -jar /opt/batch-app.jar --spring.batch.job.name=dailyReport
```

**Common cron expressions:**

| Expression | When |
|-----------|------|
| `0 0 2 * * ?` | Daily at 2 AM |
| `0 0 2 * * MON-FRI` | Weekdays at 2 AM |
| `0 0 * * * ?` | Every hour |
| `0 */30 * * * ?` | Every 30 minutes |
| `0 0 0 1 * ?` | First day of month, midnight |

### 🗣️ How to Explain in Interview

> *"There are two ways. First, Spring's @Scheduled with a cron expression — the app stays running and triggers the batch job at the scheduled time. This is simple but doesn't support clustering. Second, OS-level cron — Linux crontab or Windows Task Scheduler runs the batch app as a JAR command. The app starts, runs the job, and exits. This is cleaner for resource usage but loses in-app monitoring. For production, I prefer Quartz over both approaches because of clustering and persistence."*

### ⚡ Key Points to Remember

1. **@Scheduled(cron)** = in-app, simple, no clustering
2. **OS cron** = runs JAR as command, clean but manual
3. Spring cron: **6 fields** (sec min hour day month weekday)
4. Linux cron: **5 fields** (min hour day month weekday)
5. For production → **Quartz** over cron

---

<a id="q103"></a>

## Q103. How do you run jobs on application startup?

### 🔑 Quick Answer

> Three methods: (1) **Spring Boot default** — `spring.batch.job.enabled=true` auto-runs all jobs, (2) **CommandLineRunner** — programmatic control, (3) **Disable auto-run** with `enabled=false` and trigger via REST/scheduler.

### 📖 Step-by-Step Explanation

```
Method 1: AUTO-RUN (Spring Boot default)
  spring.batch.job.enabled=true (default)
  → ALL registered @Bean Job run on startup
  → Good for single-job apps

Method 2: COMMAND LINE RUNNER
  Implement CommandLineRunner
  → Full control over which job, which params
  → Good for migration scripts

Method 3: DISABLE + EXTERNAL TRIGGER
  spring.batch.job.enabled=false
  → No job runs on startup
  → Trigger via REST API, @Scheduled, or Quartz
  → Good for production apps with multiple jobs
```

### 💻 Code Examples

```yaml
# Method 1: Auto-run specific job
spring:
  batch:
    job:
      enabled: true
      name: migrationJob        # Only run this job (not all jobs)
```

```java
// Method 2: CommandLineRunner — full control
@Component
public class MigrationRunner implements CommandLineRunner {
    
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job migrationJob;
    
    @Override
    public void run(String... args) throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addString("version", "2.0")
                .addLong("time", System.currentTimeMillis())
                .toJobParameters();
        
        JobExecution execution = jobLauncher.run(migrationJob, params);
        System.out.println("Migration result: " + execution.getStatus());
    }
}
```

```yaml
# Method 3: Disable for production — trigger externally
spring:
  batch:
    job:
      enabled: false   # NO jobs run on startup
# Jobs triggered via REST API or Quartz scheduler
```

### 🗣️ How to Explain in Interview

> *"By default, Spring Boot auto-runs all registered batch jobs on startup. You can control which job runs with spring.batch.job.name. For more control, I use CommandLineRunner — it lets me set specific parameters and log the result. But in production, I disable auto-run with enabled=false and trigger jobs through Quartz or REST endpoints. This gives me control over when jobs run and prevents accidental execution on deployment."*

### ⚡ Key Points to Remember

1. **Default: enabled=true** → all jobs auto-run on startup
2. **spring.batch.job.name** = run specific job only
3. **CommandLineRunner** = programmatic startup control
4. **Production: enabled=false** + external trigger ⭐
5. Always pass unique params to avoid duplicate rejection

---

> **🎯 Navigation:** [← Performance (Q92-98)](11-performance.md) | [Next → Monitoring (Q104-109)](13-monitoring.md) | [📋 All Sections](README.md)
