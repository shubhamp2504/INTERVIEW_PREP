# 🟡 Scheduling — Q99 to Q103

---

## Q99. How do you schedule Spring Batch jobs?

### 📝 One-Liner
Spring Batch has no built-in scheduler — use @Scheduled (simplest), Quartz (production), Kubernetes CronJob (cloud), or OS cron (traditional).

### 🔑 Quick Answer
Spring Batch is a **processing framework**, not a scheduling framework. You need an external scheduler. Options: **(1) @Scheduled** — simplest, good for single-instance apps, no clustering/persistence. **(2) Quartz** — production standard, supports clustering (only 1 instance runs), JDBC persistence, misfire handling. **(3) Kubernetes CronJob** — cloud-native, container-based. **(4) OS cron** — traditional, runs JAR with crontab. Always pass **unique JobParameters** (e.g., timestamp) to avoid "already completed" errors. *(Spring Batch ke paas apna scheduler nahi hai — bahar se lagana padta hai)*

### 📖 How It Works
```
Scheduler Options:

@Scheduled (simplest):
  ├── Built into Spring
  ├── Cron or fixed-rate
  ├── ❌ No clustering (all instances run)
  ├── ❌ No persistence of schedule state
  └── Good for: single-instance, non-critical

Quartz ⭐ (production):
  ├── Clustered (only 1 instance fires)
  ├── JDBC persistence (survives restarts)
  ├── Misfire handling
  ├── Rich API (pause, resume, reschedule)
  └── Good for: production, multi-instance

Kubernetes CronJob (cloud):
  ├── One pod per execution
  ├── Built-in retry, history
  ├── Cluster awareness via K8s
  └── Good for: containerized deployments

OS Cron (traditional):
  ├── Linux crontab / Windows Task Scheduler
  ├── Runs JAR externally
  ├── ❌ No clustering (needs wrapper)
  └── Good for: simple, legacy
```

### 🗣️ How to Say in Interview
"Spring Batch doesn't include a scheduler — it's purely a processing framework. For scheduling, I use Quartz in production because it supports clustering, ensuring only one instance in a cluster fires the trigger. It also persists schedule state in JDBC, so schedule information survives application restarts. For cloud deployments on Kubernetes, I've used CronJobs. The key with any scheduler is to always pass unique JobParameters — typically a timestamp — because Spring Batch rejects re-running a completed job with the same parameters."

### 💻 Code
```java
// @Scheduled — simplest approach
@Component
@EnableScheduling
public class BatchScheduler {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job dailyPaymentJob;

    @Scheduled(cron = "0 0 2 * * ?")  // 2 AM daily
    public void runDailyJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("runTime", LocalDateTime.now())  // unique each run
                .toJobParameters();
        jobLauncher.run(dailyPaymentJob, params);
    }
}

// Quartz — production approach (see Q101)
// Kubernetes CronJob — in deployment YAML:
// apiVersion: batch/v1
// kind: CronJob
// metadata:
//   name: daily-payment-job
// spec:
//   schedule: "0 2 * * *"
//   jobTemplate:
//     spec:
//       template:
//         spec:
//           containers:
//           - name: batch-job
//             image: myapp/batch:latest
//             command: ["java", "-jar", "batch.jar", "--job=dailyPayment"]
//           restartPolicy: OnFailure
```

### ⚠️ Pitfalls / Gotchas
- @Scheduled with no clustering → every instance in the cluster fires → duplicate executions *(clustering nahi hai toh sab instances job chalayenge)*
- Must pass unique JobParameters every run — same params = "already completed" error
- Quartz needs JDBC tables (10 tables for clustering support)
- K8s CronJob: set concurrencyPolicy: Forbid to prevent overlapping runs

### 🆚 vs. Comparison
| Feature | @Scheduled | Quartz ⭐ | K8s CronJob | OS Cron |
|---------|-----------|---------|-------------|---------|
| Clustering | ❌ | ✅ | ✅ (K8s) | ❌ |
| Persistence | ❌ | ✅ JDBC | ✅ (K8s) | ❌ |
| Misfire | ❌ | ✅ | ❌ | ❌ |
| Complexity | Low | Medium | Medium | Low |
| Best for | Dev/single | Production ⭐ | Cloud | Legacy |

### ⚡ Remember
- Spring Batch has NO scheduler *(apna scheduler nahi — bahar se lagao)*
- @Scheduled = simple, no clustering
- **Quartz = production standard** (clustering + persistence)
- K8s CronJob = cloud-native
- Always unique JobParameters (timestamp)

### 🔗 Follow-ups
- [Q101 → Quartz integration details](#q101)
- [Q100 → Job trigger types](#q100)
- [Q103 → Running on startup](#q103)

---

## Q100. How do you trigger jobs automatically?

### 📝 One-Liner
Four trigger types: time-based (@Scheduled/Quartz), event-based (file watcher/message listener), API-based (REST endpoint), and startup (CommandLineRunner).

### 🔑 Quick Answer
**(1) Time-based**: @Scheduled cron, Quartz trigger, K8s CronJob — run at fixed times. **(2) Event-based**: file arrives in directory → file watcher triggers job; message arrives in queue → listener triggers job. **(3) API-based**: REST POST endpoint calls JobLauncher — for on-demand or external system triggers. **(4) Startup**: CommandLineRunner or `spring.batch.job.enabled=true` — for one-time migrations. Always use unique parameters for every trigger. *(Char tarike — time, event, API, startup — sab mein unique params dena zaroori)*

### 📖 How It Works
```
Trigger Types:

TIME-BASED:
  @Scheduled(cron="0 0 2 * * ?") → 2 AM daily
  Quartz CronTrigger → clustered scheduling
  K8s CronJob → containerized scheduling

EVENT-BASED:
  File arrives in /incoming/ → WatchService detects → launch job
  Message in Kafka/RabbitMQ → Listener receives → launch job

API-BASED:
  POST /api/batch/launch?job=paymentJob → REST controller → launch job
  External system calls API → job starts

STARTUP:
  Application starts → CommandLineRunner → launch job
  spring.batch.job.enabled=true → auto-run all jobs
```

### 🗣️ How to Say in Interview
"I use four trigger types depending on the requirement. For daily reconciliation, it's time-based with Quartz. For file processing, we have a file watcher that triggers the job when a file arrives in the inbox directory. For on-demand processing, we expose a REST endpoint that operations teams can call. For one-time data migrations, we use CommandLineRunner that runs on application startup. The key across all triggers is unique JobParameters — I always include a timestamp so the same job can run again without the 'already completed' error."

### 💻 Code
```java
// EVENT-BASED: File watcher trigger
@Component
public class FileWatchTrigger {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job fileProcessingJob;

    @Scheduled(fixedDelay = 30000)  // check every 30 seconds
    public void watchInbox() throws Exception {
        Path inbox = Paths.get("/data/inbox");
        try (DirectoryStream<Path> files = Files.newDirectoryStream(inbox, "*.csv")) {
            for (Path file : files) {
                JobParameters params = new JobParametersBuilder()
                        .addString("filePath", file.toString())
                        .addLocalDateTime("triggerTime", LocalDateTime.now())
                        .toJobParameters();
                jobLauncher.run(fileProcessingJob, params);
                Files.move(file, Paths.get("/data/processing/" + file.getFileName()));
            }
        }
    }
}

// API-BASED: REST endpoint trigger
@RestController
@RequestMapping("/api/batch")
public class BatchController {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private ApplicationContext context;

    @PostMapping("/launch/{jobName}")
    public ResponseEntity<String> launchJob(@PathVariable String jobName) throws Exception {
        Job job = context.getBean(jobName, Job.class);
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("triggerTime", LocalDateTime.now())
                .toJobParameters();
        JobExecution execution = jobLauncher.run(job, params);
        return ResponseEntity.ok("Job started: " + execution.getId());
    }
}

// STARTUP: CommandLineRunner
@Component
public class MigrationRunner implements CommandLineRunner {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job migrationJob;

    @Override
    public void run(String... args) throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("migrationTime", LocalDateTime.now())
                .toJobParameters();
        jobLauncher.run(migrationJob, params);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- File watcher: move file to processing folder BEFORE launching to avoid re-trigger *(file ko pehle move karo — nahi toh dobara trigger hoga)*
- REST trigger: use async JobLauncher for long jobs (don't block HTTP request)
- Startup trigger: set `spring.batch.job.enabled=false` in production to prevent accidental auto-run
- Always unique params — without them second run fails

### ⚡ Remember
- Time: @Scheduled, Quartz, K8s CronJob
- Event: file watcher, message listener *(file aayi = job start)*
- API: REST POST endpoint
- Startup: CommandLineRunner / spring.batch.job.enabled
- Always unique JobParameters

### 🔗 Follow-ups
- [Q99 → Scheduling options](#q99)
- [Q101 → Quartz integration](#q101)
- [Q121 → File upload processing](#q121)

---

## Q101. Can Spring Batch run with Quartz?

### 📝 One-Liner
Yes — Quartz is the production standard. QuartzJobBean calls JobLauncher.run(). Quartz handles WHEN, Spring Batch handles WHAT.

### 🔑 Quick Answer
**Yes, and it's the recommended production setup.** `QuartzJobBean.executeInternal()` calls `jobLauncher.run()`. Quartz provides: **(1) Clustering** — only 1 instance in the cluster fires the trigger. **(2) JDBC persistence** — schedule survives restarts. **(3) Misfire handling** — handles missed triggers (app was down). **(4) Rich API** — pause, resume, reschedule programmatically. @Scheduled can't do any of this — it fires on every instance and loses state on restart. *(Quartz = WHEN job chalana hai, Spring Batch = WHAT karna hai — dono milke production ka standard hain)*

### 📖 How It Works
```
Quartz + Spring Batch Flow:

  Quartz Scheduler (clustered, JDBC-backed)
      ↓
  CronTrigger fires at 2 AM
      ↓
  QuartzJobBean.executeInternal()
      ↓
  jobLauncher.run(springBatchJob, params)
      ↓
  Spring Batch processes data

@Scheduled Problems (why Quartz is needed):
  ❌ No clustering → all instances fire → duplicate processing
  ❌ No persistence → restart loses schedule
  ❌ No misfire handling → missed at 2AM? → silently skipped
  ❌ No API → can't pause/resume dynamically

Quartz Fixes All:
  ✅ Clustering → only 1 instance fires (via DB lock)
  ✅ JDBC persistence → schedule survives restart
  ✅ Misfire → fires immediately when detected
  ✅ Rich API → pause/resume/reschedule
```

### 🗣️ How to Say in Interview
"Yes, Quartz with Spring Batch is the production standard. I create a QuartzJobBean that calls JobLauncher.run() in its executeInternal method. Quartz handles the scheduling concerns — clustering ensures only one instance fires the trigger via a database lock, JDBC persistence means the schedule survives application restarts, and misfire handling catches up on missed triggers. In my project, we had a 5-node cluster running our batch application, and Quartz ensured only one node executed each scheduled job. When we had a brief outage at 2 AM, the misfire policy triggered the job as soon as the app came back online."

### 💻 Code
```java
// Quartz Job Bean — calls Spring Batch
public class PaymentBatchJob extends QuartzJobBean {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job paymentJob;

    @Override
    protected void executeInternal(JobExecutionContext context) throws JobExecutionException {
        try {
            JobParameters params = new JobParametersBuilder()
                    .addLocalDateTime("scheduledTime", LocalDateTime.now())
                    .toJobParameters();
            jobLauncher.run(paymentJob, params);
        } catch (Exception e) {
            throw new JobExecutionException("Batch job failed", e);
        }
    }
}

// Quartz Configuration
@Configuration
public class QuartzConfig {

    @Bean
    public JobDetail paymentJobDetail() {
        return JobBuilder.newJob(PaymentBatchJob.class)
                .withIdentity("paymentJob", "batchGroup")
                .storeDurably()
                .build();
    }

    @Bean
    public CronTrigger paymentTrigger() {
        return TriggerBuilder.newTrigger()
                .forJob(paymentJobDetail())
                .withIdentity("paymentTrigger", "batchGroup")
                .withSchedule(CronScheduleBuilder
                        .cronSchedule("0 0 2 * * ?")          // 2 AM daily
                        .withMisfireHandlingInstructionFireAndProceed())  // catch missed
                .build();
    }
}

// application.yml — Quartz with JDBC store + clustering
// spring:
//   quartz:
//     job-store-type: jdbc                    # persist to DB
//     properties:
//       org.quartz.jobStore.isClustered: true # enable clustering
//       org.quartz.jobStore.clusterCheckinInterval: 20000
//       org.quartz.scheduler.instanceId: AUTO
```

### ⚠️ Pitfalls / Gotchas
- Quartz needs 10+ DB tables — auto-create with `spring.quartz.jdbc.initialize-schema=always` *(Quartz ke apne tables chahiye DB mein)*
- Don't confuse `org.quartz.Job` with `org.springframework.batch.core.Job` — different packages
- Quartz `JobStore` must be JDBC for clustering (not RAM)
- Misfire policy: `withMisfireHandlingInstructionFireAndProceed` = fire once immediately

### ⚡ Remember
- QuartzJobBean calls jobLauncher.run() *(Quartz = WHEN, Batch = WHAT)*
- **Clustering**: only 1 instance fires (DB lock)
- **JDBC persistence**: survives restart
- **Misfire handling**: catches missed triggers
- @Scheduled = no clustering, no persistence → dev only

### 🔗 Follow-ups
- [Q99 → Scheduling options](#q99)
- [Q102 → Cron scheduling](#q102)
- [Q119 → Multiple jobs running](#q119)

---

## Q102. Can Spring Batch run with Cron scheduling?

### 📝 One-Liner
Yes — two approaches: @Scheduled(cron) in-app or OS-level crontab running the JAR.

### 🔑 Quick Answer
**(1) In-app cron**: `@Scheduled(cron = "0 0 2 * * MON-FRI")` — job runs within the same JVM. Simple but no clustering. Spring cron = 6 fields (second minute hour day month weekday). **(2) OS cron**: Linux `crontab -e` → `0 2 * * * java -jar batch.jar`. OS starts a new JVM each execution. No long-running process needed but OS-dependent. For production, use Quartz (cron triggers with clustering). *(Cron do tarike se — app ke andar @Scheduled ya OS level crontab)*

### 📖 How It Works
```
Spring @Scheduled Cron vs OS Cron:

Spring Cron (6 fields):
  sec  min  hour  day  month  weekday
   0    0    2     *     *    MON-FRI   → Weekdays at 2 AM

Linux Cron (5 fields — no seconds):
  min  hour  day  month  weekday
   0    2     *     *       1-5        → Weekdays at 2 AM

Common Cron Expressions:
  "0 0 2 * * ?"        → Every day at 2 AM
  "0 0 2 * * MON-FRI"  → Weekdays at 2 AM
  "0 0 */4 * * ?"      → Every 4 hours
  "0 30 1 1 * ?"       → 1st of month at 1:30 AM
  "0 0 0 L * ?"        → Last day of month at midnight
```

### 🗣️ How to Say in Interview
"Yes, there are two approaches. The simpler one is @Scheduled with a cron expression inside the application — the job runs within the same JVM process. Spring cron has 6 fields including seconds. The other approach is OS-level crontab that starts the Java process externally. For production, I prefer Quartz with cron triggers because it adds clustering and persistence on top of cron scheduling. @Scheduled cron works well for single-instance development and non-critical jobs."

### 💻 Code
```java
// In-app cron scheduling
@Component
@EnableScheduling
public class CronScheduler {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job reconciliationJob;

    @Scheduled(cron = "0 0 2 * * MON-FRI")  // Weekdays 2 AM
    public void runWeekdayJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("scheduledAt", LocalDateTime.now())
                .toJobParameters();
        jobLauncher.run(reconciliationJob, params);
    }

    @Scheduled(cron = "0 30 1 1 * ?")  // 1st of month at 1:30 AM
    public void runMonthlyJob() throws Exception {
        JobParameters params = new JobParametersBuilder()
                .addLocalDateTime("scheduledAt", LocalDateTime.now())
                .toJobParameters();
        jobLauncher.run(reconciliationJob, params);
    }
}

// OS Cron (Linux crontab -e):
// 0 2 * * * /usr/bin/java -jar /opt/batch/batch-app.jar --spring.batch.job.name=paymentJob
// 30 1 1 * * /usr/bin/java -jar /opt/batch/batch-app.jar --spring.batch.job.name=monthlyReport
```

### ⚠️ Pitfalls / Gotchas
- Spring cron = 6 fields (with seconds), Linux cron = 5 fields (no seconds) *(Spring mein seconds hai, Linux mein nahi)*
- @Scheduled without clustering → multiple instances fire simultaneously
- OS cron starts new JVM each time → cold start overhead
- Don't forget unique parameters — without them job fails on second run

### ⚡ Remember
- @Scheduled cron = in-app, simple, no clustering
- OS cron = external, runs JAR
- Spring cron: 6 fields (sec min hr day mon weekday) *(Spring = 6 fields with seconds)*
- Linux cron: 5 fields (min hr day mon weekday)
- Production → Quartz with cron triggers

### 🔗 Follow-ups
- [Q99 → Scheduling options](#q99)
- [Q101 → Quartz (preferred for production)](#q101)
- [Q103 → Running on startup](#q103)

---

## Q103. How do you run jobs on application startup?

### 📝 One-Liner
Three methods: Spring Boot auto-runs all jobs (default), CommandLineRunner for programmatic control, or disable auto-run with spring.batch.job.enabled=false.

### 🔑 Quick Answer
**(1) Default behavior**: `spring.batch.job.enabled=true` (default) → Spring Boot auto-runs ALL registered jobs on startup. **(2) Specific job**: set `spring.batch.job.name=myJob` → only runs that job. **(3) Programmatic**: implement `CommandLineRunner` for custom logic (conditional runs, args parsing). **(4) Production**: set `enabled=false` → no auto-run → trigger externally (Quartz, REST, etc.). For one-time migrations, CommandLineRunner is ideal. *(Default mein sab jobs auto-run hote hain — production mein disable karke bahar se trigger karo)*

### 📖 How It Works
```
Startup Behavior:

spring.batch.job.enabled=true (default):
  App starts → finds ALL @Bean Job → runs ALL → exits/continues
  ⚠️ Dangerous in production — ALL jobs run!

spring.batch.job.name=myJob:
  App starts → finds only "myJob" → runs it → exits/continues
  Good for: specific job via command line

spring.batch.job.enabled=false:
  App starts → No jobs auto-run
  Jobs triggered externally (Quartz, REST, etc.)
  ✅ Production standard

CommandLineRunner:
  App starts → your run() method called → you decide what to run
  Good for: one-time migrations, conditional logic
```

### 🗣️ How to Say in Interview
"By default, Spring Boot auto-runs all registered batch jobs on startup, which is dangerous in production. I always set spring.batch.job.enabled=false and trigger jobs externally via Quartz or REST endpoints. For one-time data migrations, I use CommandLineRunner which gives me programmatic control — I can parse command-line arguments, check conditions, and decide whether to run. In my project, we had a database migration job that ran via CommandLineRunner on first deployment, then we removed it in the next release."

### 💻 Code
```java
// application.yml — production config
// spring:
//   batch:
//     job:
//       enabled: false  # ← never auto-run in production

// Run specific job via command line:
// java -jar batch.jar --spring.batch.job.name=paymentJob

// CommandLineRunner for one-time migration
@Component
public class MigrationRunner implements CommandLineRunner {
    @Autowired private JobLauncher jobLauncher;
    @Autowired private Job dataMigrationJob;

    @Override
    public void run(String... args) throws Exception {
        // Only run if migration hasn't completed yet
        if (shouldRunMigration()) {
            JobParameters params = new JobParametersBuilder()
                    .addString("migrationVersion", "v2.5")
                    .addLocalDateTime("startTime", LocalDateTime.now())
                    .toJobParameters();
            JobExecution execution = jobLauncher.run(dataMigrationJob, params);
            log.info("Migration completed with status: {}", execution.getStatus());
        }
    }
    
    private boolean shouldRunMigration() {
        // Check if migration already done (e.g., flag in DB)
        return !migrationRepository.isCompleted("v2.5");
    }
}

// Disable auto-run, trigger via REST
// spring.batch.job.enabled=false
// POST /api/batch/launch/paymentJob → launches on demand
```

### ⚠️ Pitfalls / Gotchas
- `enabled=true` (default) runs ALL jobs → dangerous in production *(production mein default true hai — sab jobs chal jayenge!)*
- Multiple jobs registered → all fire on startup unless filtered
- CommandLineRunner runs in startup thread — long job blocks application readiness
- Same params on restart → "already completed" error → always use unique params

### ⚡ Remember
- Default: `enabled=true` → ALL jobs auto-run *(default sab chalata hai — production mein band karo)*
- `spring.batch.job.name=X` → run specific job
- `enabled=false` → no auto-run (production standard)
- CommandLineRunner = programmatic startup logic
- Always unique JobParameters

### 🔗 Follow-ups
- [Q99 → Scheduling options](#q99)
- [Q100 → Trigger types](#q100)
- [Q101 → Quartz for production](#q101)
