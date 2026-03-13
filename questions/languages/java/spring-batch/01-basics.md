# 🟢 Spring Batch — Basics (Q1–Q20)

> **How to use this guide:**  
> Each question follows this pattern — first understand it yourself, then learn how to explain it.  
> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>

## Q1. What is Spring Batch?

### 🔑 Quick Answer (say this first)

> Spring Batch is a **lightweight framework** for processing large amounts of data in **batches** — it reads data from a source, processes it, and writes it to a destination, all in a reliable, restartable way.

### 📖 Step-by-Step Explanation

**Step 1 — Understand the problem it solves:**  
Imagine you have 10 million customer records in a CSV file and you need to validate each one, transform it, and insert into a database. You can't do this one-by-one with a REST API. You need **batch processing** — handle bulk data in chunks.

**Step 2 — What Spring Batch gives you:**  
Instead of writing all the boilerplate yourself (reading files, managing transactions, handling failures), Spring Batch provides a ready-made framework:

```
INPUT (CSV/DB/API) → [Read] → [Process] → [Write] → OUTPUT (DB/File/API)
                         ↑          ↑          ↑
                   Spring Batch handles all of this
```

**Step 3 — Key capabilities it provides:**

| Capability | What it means |
|-----------|---------------|
| Chunk processing | Reads N records, processes them, writes them as a batch |
| Transaction management | If chunk fails, only that chunk rolls back |
| Restartability | If job crashes at record 5000, it resumes from 5001 |
| Skip & Retry | Skip bad records or retry on transient errors |
| Scalability | Parallel processing, partitioning across threads |

**Step 4 — What it does NOT do:**  
Spring Batch does NOT schedule jobs. It only executes them. For scheduling, you use Quartz, `@Scheduled`, or Kubernetes CronJob.

### 🗣️ How to Explain in Interview

> *"Spring Batch is a framework for batch processing — when you need to process millions of records reliably. It follows a Read → Process → Write pattern in chunks. So if I have a 10 million record CSV file, Spring Batch reads 500 records at a time, processes each one, writes all 500 to the database in one transaction, then moves to the next 500. If something fails mid-way, it can restart from where it left off. It also gives you skip logic for bad records, retry for transient failures, and parallel processing for performance. One important thing — it doesn't do scheduling, it only does execution."*

### 💻 Code Example

```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Job myJob(JobRepository jobRepository, Step step1) {
        return new JobBuilder("myJob", jobRepository)
                .start(step1)    // Start with step1
                .build();        // Build the job
    }

    @Bean
    public Step step1(JobRepository jobRepository,
                      PlatformTransactionManager txManager) {
        return new StepBuilder("step1", jobRepository)
                .<String, String>chunk(100, txManager)  // Process 100 records per chunk
                .reader(reader())       // WHERE to read from
                .processor(processor()) // HOW to transform
                .writer(writer())       // WHERE to write to
                .build();
    }
}
```

**Line-by-line walkthrough:**
- `@EnableBatchProcessing` → Activates Spring Batch infrastructure (creates JobRepository, JobLauncher, etc.)
- `JobBuilder("myJob", jobRepository)` → Creates a Job named "myJob" — JobRepository stores execution metadata
- `.start(step1)` → This job has one step. Jobs can have multiple steps chained with `.next()`
- `StepBuilder("step1", jobRepository)` → Creates a Step named "step1"
- `.<String, String>chunk(100, txManager)` → Input type is String, Output type is String, process 100 items per transaction
- `.reader()` / `.processor()` / `.writer()` → The three core components plugged in

### ⚡ Key Points to Remember

1. Spring Batch = **Read → Process → Write** in chunks
2. Provides **transaction management** at chunk level
3. Supports **restart from failure point** (not from beginning)
4. Does **NOT** do scheduling — only execution
5. Built on top of Spring Framework

### 🔗 Follow-up Questions Interviewer May Ask
- *"What are the core components of Spring Batch?"* → See Q4
- *"What is chunk processing?"* → See Q21
- *"How does restart work?"* → See Q61

---

<a id="q2"></a>

## Q2. What are the main use cases of Spring Batch?

### 🔑 Quick Answer

> Whenever you need to **process large volumes of data** in a **reliable, repeatable way** — ETL pipelines, data migration, report generation, bulk notifications, and data cleanup.

### 📖 Step-by-Step Explanation

**Step 1 — Think about what operations involve "bulk data":**  
Any operation where you're dealing with thousands to millions of records, not individual requests.

**Step 2 — Real-world use cases with examples:**

| # | Use Case | Real Example | Why Batch? |
|---|----------|-------------|------------|
| 1 | **ETL** | Read 5M records from CSV → validate → insert into PostgreSQL | Too many records for REST API |
| 2 | **Data Migration** | Move customer data from Oracle to MySQL | Need transaction safety + restart |
| 3 | **Report Generation** | Generate monthly billing report for 2M customers | Heavy computation, runs overnight |
| 4 | **File Processing** | Bank processes 50,000 cheque images daily | Each file needs validation + transformation |
| 5 | **Database Cleanup** | Archive orders older than 2 years | Delete/move millions of rows safely |
| 6 | **Billing** | Calculate monthly bills for telecom customers | Must process ALL customers, no skipping |
| 7 | **Bulk Notifications** | Send 1M promotional emails | Rate-limited, needs tracking |
| 8 | **Data Sync** | Sync inventory between warehouse and e-commerce DB | Runs every hour, must be idempotent |
| 9 | **Bulk Updates** | Update product prices from supplier feed | 100K products, need audit trail |
| 10 | **Compliance** | Process regulatory reports for banking | Must be accurate, auditable, restartable |

**Step 3 — Pattern: When to choose Spring Batch vs other approaches:**

```
Is it ONE record at a time?          → Use REST API / Messaging
Is it THOUSANDS of records?          → Consider Spring Batch
Is it MILLIONS of records?           → Definitely Spring Batch
Does it need to be RELIABLE?         → Spring Batch (restart, skip, retry)
Does it need REAL-TIME processing?   → Use Kafka Streams / Spring WebFlux
Does it run on a SCHEDULE?           → Spring Batch + Scheduler
```

### 🗣️ How to Explain in Interview

> *"In my experience, Spring Batch is ideal for any high-volume data processing task. For example, in our project we used it for an ETL pipeline — reading millions of records from a CSV file, validating and transforming each one, and writing to the database. We also used it for monthly report generation where we needed to process 2 million customer records. The key advantage over just writing a simple loop is that Spring Batch gives you transaction safety, restart capability if it fails midway, and skip logic for bad records — which are critical in production."*

### ⚡ Key Points to Remember

1. **High volume** = Spring Batch's sweet spot (thousands to millions of records)
2. Most common: **ETL, migration, reports, file processing, cleanup**
3. Not for real-time — use Kafka/WebFlux for that
4. Not for single-record operations — use REST API for that

### 🔗 Follow-up Questions
- *"Can you explain a batch job you built in your project?"*
- *"Why not just use a simple for loop?"* → Transaction safety, restart, monitoring
- *"How do you schedule these jobs?"* → See Q99

---

<a id="q3"></a>

## Q3. What problems does Spring Batch solve?

### 🔑 Quick Answer

> It solves the **hard parts of batch processing** — transaction management, failure recovery, restartability, parallel processing, and monitoring — so you focus only on business logic.

### 📖 Step-by-Step Explanation

**Step 1 — Imagine building batch processing without Spring Batch:**

```java
// WITHOUT Spring Batch — you write ALL of this yourself:
Connection conn = dataSource.getConnection();
BufferedReader reader = new BufferedReader(new FileReader("data.csv"));
String line;
int count = 0;
int failCount = 0;

try {
    conn.setAutoCommit(false);
    while ((line = reader.readLine()) != null) {
        try {
            // Parse, validate, transform
            Record record = parse(line);
            
            // Insert to DB
            insertToDb(conn, record);
            count++;
            
            // Commit every 500 records
            if (count % 500 == 0) {
                conn.commit();
                saveCheckpoint(count);  // For restart!
            }
        } catch (Exception e) {
            failCount++;
            logFailedRecord(line, e);   // Skip logic!
            if (failCount > 100) {
                throw new TooManyFailuresException();
            }
        }
    }
    conn.commit();
} catch (Exception e) {
    conn.rollback();
    // How do you restart from where you left off?
    // How do you handle partial failures?
    // How do you monitor progress?
}
```

**Step 2 — Problems with DIY approach:**

| Problem | What goes wrong |
|---------|----------------|
| Transaction management | Commit every N records? What if crash between commits? |
| Restart | How to resume from record #50,001 after crash? |
| Error handling | Skip bad record? Retry? How many times? |
| Monitoring | How to know job processed 5M of 10M records? |
| Parallel processing | How to split work across 8 threads safely? |
| State management | How to share data between steps? |
| Scaling | Works for 10K records, but what about 100M? |

**Step 3 — How Spring Batch solves each:**

| Problem | Spring Batch Solution |
|---------|----------------------|
| Transaction management | 1 chunk = 1 transaction (automatic) |
| Restart from failure | Stores checkpoint in DB, resumes from last committed chunk |
| Error handling | Built-in `skip()` and `retry()` with configurable limits |
| Monitoring | Metadata tables store read/write/skip counts per step |
| Parallel processing | Multi-threading, Partitioning, Remote Chunking |
| State management | ExecutionContext (persisted key-value store) |
| Scaling | Partitioning splits data across threads/machines |

### 🗣️ How to Explain in Interview

> *"Without Spring Batch, you'd write all the plumbing yourself — transaction management, checkpoint saving for restart, error handling with skip logic, progress monitoring. Spring Batch solves all of this out of the box. For example, if my job crashes after processing 50,000 records, Spring Batch knows exactly where it stopped and resumes from record 50,001. If a record is malformed, I can configure skip logic to skip up to 100 bad records. All this state — read counts, write counts, skip counts — is stored in metadata tables automatically."*

### ⚡ Key Points to Remember

1. **Transaction management** → Automatic per chunk
2. **Restartability** → Resumes from last committed checkpoint
3. **Skip/Retry** → Configurable error handling
4. **Monitoring** → Metadata tables track everything
5. **Scalability** → Built-in parallel processing options

### 🔗 Follow-up Questions
- *"How does the restart actually work internally?"* → See Q61, Q66
- *"How does it handle transactions?"* → See Q63

---

<a id="q4"></a>

## Q4. What are the core components of Spring Batch?

### 🔑 Quick Answer

> The core components are: **Job** (the entire batch process), **Step** (a phase within the job), **ItemReader/Processor/Writer** (the actual work), **JobRepository** (stores metadata), and **JobLauncher** (triggers execution).

### 📖 Step-by-Step Explanation

**Step 1 — Start with the big picture:**

```
You (or Scheduler)
      │
      ▼
 JobLauncher ───── "Hey, run this job!"
      │
      ▼
    Job ─────────── "I am the whole batch process"
      │
      ├── Step 1 ── "I do phase 1"
      │     ├── ItemReader     ── "I read data"
      │     ├── ItemProcessor  ── "I transform data"
      │     └── ItemWriter     ── "I write data"
      │
      ├── Step 2 ── "I do phase 2"
      │     └── Tasklet        ── "I do a single task"
      │
      └── Step 3 ── "I do phase 3"
            └── ...

 JobRepository ──── "I store everything that happened"
                    (metadata, status, counts, checkpoints)
```

**Step 2 — Understand each component one by one:**

| Component | What It Does | Analogy |
|-----------|-------------|---------|
| **JobLauncher** | Triggers a Job with parameters | The "Start" button |
| **Job** | Contains the entire batch process (all steps) | A recipe book |
| **Step** | One independent phase of the Job | One recipe |
| **ItemReader** | Reads data from a source, one item at a time | Eyes reading ingredients |
| **ItemProcessor** | Transforms/validates each item | Chef preparing each ingredient |
| **ItemWriter** | Writes a batch of items to destination | Serving all dishes at once |
| **JobRepository** | Stores execution metadata in database | The log book |
| **JobParameters** | Input parameters that identify a unique run | Recipe customizations |
| **ExecutionContext** | Key-value store for sharing state between steps | Chef's notepad |

**Step 3 — How they connect at runtime:**

```
1. JobLauncher receives Job + JobParameters
2. JobLauncher asks JobRepository: "Has this job already completed with these params?"
3. If new → creates JobInstance + JobExecution
4. Job starts Step 1:
     → Reader.read() called repeatedly (returns 1 item each time)
     → Processor.process(item) transforms each item
     → Writer.write(chunk) writes all items in one batch
     → Transaction committed
     → Repeat until Reader returns null
5. Step 1 complete → moves to Step 2
6. All steps done → Job status = COMPLETED
7. Everything recorded in JobRepository
```

### 🗣️ How to Explain in Interview

> *"Spring Batch has a clear component hierarchy. At the top is the Job, which represents the entire batch process. A Job contains one or more Steps — each Step is an independent phase. Inside a Step, you have an ItemReader that reads data one item at a time, an ItemProcessor that transforms each item, and an ItemWriter that writes the whole chunk at once. The JobLauncher triggers the job, and the JobRepository stores all execution metadata in the database — status, counts, timestamps — which enables restartability and monitoring."*

### 💻 Code Example

```java
@Configuration
@EnableBatchProcessing
public class OrderBatchConfig {

    // 1. JOB — the whole process
    @Bean
    public Job orderProcessingJob(JobRepository jobRepository,
                                   Step readStep, Step reportStep) {
        return new JobBuilder("orderProcessingJob", jobRepository)
                .start(readStep)      // First do this
                .next(reportStep)     // Then do this
                .build();
    }

    // 2. STEP — one phase (chunk-based: read → process → write)
    @Bean
    public Step readStep(JobRepository repo, PlatformTransactionManager tx) {
        return new StepBuilder("readStep", repo)
                .<Order, Order>chunk(500, tx)
                .reader(orderReader())        // 3. READER
                .processor(orderProcessor())  // 4. PROCESSOR
                .writer(orderWriter())        // 5. WRITER
                .build();
    }

    // 3. READER — reads from CSV file
    @Bean
    public FlatFileItemReader<Order> orderReader() {
        return new FlatFileItemReaderBuilder<Order>()
                .name("orderReader")
                .resource(new ClassPathResource("orders.csv"))
                .delimited()
                .names("orderId", "product", "amount")
                .targetType(Order.class)
                .build();
    }

    // 4. PROCESSOR — validates each order
    @Bean
    public ItemProcessor<Order, Order> orderProcessor() {
        return order -> {
            if (order.getAmount() <= 0) return null;  // Filter invalid
            order.setTax(order.getAmount() * 0.18);   // Calculate tax
            return order;
        };
    }

    // 5. WRITER — writes to database
    @Bean
    public JdbcBatchItemWriter<Order> orderWriter() {
        return new JdbcBatchItemWriterBuilder<Order>()
                .sql("INSERT INTO orders (id, product, amount, tax) VALUES (:orderId, :product, :amount, :tax)")
                .dataSource(dataSource)
                .beanMapped()
                .build();
    }
}
```

### ⚡ Key Points to Remember

1. **Job** = entire process, **Step** = one phase
2. **Reader** reads one-at-a-time, **Writer** writes whole chunk at once
3. **Processor** is optional — can skip it
4. **JobRepository** = metadata storage (mandatory)
5. **JobLauncher** = entry point to trigger job

### 🔗 Follow-up Questions
- *"Explain Job, Step, JobInstance, JobExecution in detail"* → See Q5-Q7, Q55-Q57
- *"What are the metadata tables?"* → See Q18, Q110

---

<a id="q5"></a>

## Q5. What is a Job in Spring Batch?

### 🔑 Quick Answer

> A Job is the **top-level container** that represents the **entire batch process**. It contains one or more Steps and is uniquely identified by its name + parameters.

### 📖 Step-by-Step Explanation

**Step 1 — Job = The whole process:**

A Job is like a **recipe** — it defines what steps to execute and in what order.

**Step 2 — Job identity (important for interviews):**

```
Job Name + Job Parameters = JobInstance (unique identity)

Example:
  Job: "monthlyBilling"
  Parameters: { month: "2024-01" }
  
  This creates ONE JobInstance.
  
  If you run it again with { month: "2024-01" } → SAME JobInstance (won't re-run if completed)
  If you run it with { month: "2024-02" } → NEW JobInstance
```

**Step 3 — Job → JobInstance → JobExecution:**

```
Job: "monthlyBilling"
│
├── JobInstance (month=2024-01)
│   ├── JobExecution #1 → FAILED (crashed at step 2)
│   └── JobExecution #2 → COMPLETED (restarted, finished)
│
├── JobInstance (month=2024-02)
│   └── JobExecution #1 → COMPLETED (ran successfully first time)
│
└── JobInstance (month=2024-03)
    └── JobExecution #1 → STARTED (currently running)
```

- **Job** = The definition (like a class)
- **JobInstance** = One logical run identified by unique parameters (like an object)
- **JobExecution** = One physical attempt (can have many if restarts)

### 🗣️ How to Explain in Interview

> *"A Job is the top-level container for a batch process — it defines the steps and their order. What's important to understand is the identity model: a Job is uniquely identified by its name plus parameters. So if I have a job called 'monthlyBilling' and I run it with parameter month=January, that creates a JobInstance. If it fails and I restart with the same parameters, it creates a new JobExecution under the same JobInstance — so Spring Batch knows it's a retry, not a new run. If I run with month=February, it creates a completely new JobInstance."*

### 💻 Code Example

```java
@Bean
public Job monthlyBillingJob(JobRepository jobRepository,
                              Step readCustomersStep,
                              Step calculateBillsStep,
                              Step sendNotificationsStep) {
    return new JobBuilder("monthlyBilling", jobRepository)
            .start(readCustomersStep)          // Step 1: Read all customers
            .next(calculateBillsStep)          // Step 2: Calculate bills
            .next(sendNotificationsStep)       // Step 3: Send emails
            .build();
}
```

**What happens when this runs:**
1. JobLauncher receives this Job + parameters `{month: "2024-01"}`
2. JobRepository checks: does JobInstance for "monthlyBilling" + month=2024-01 exist?
   - No → Create new JobInstance + JobExecution
   - Yes, COMPLETED → Throw `JobInstanceAlreadyCompleteException`
   - Yes, FAILED → Create new JobExecution (restart)
3. Steps execute in order: readCustomers → calculateBills → sendNotifications

### ⚡ Key Points to Remember

1. Job = **entire batch process** (container for steps)
2. **Job + Parameters = JobInstance** (unique identity)
3. **JobInstance can have multiple JobExecutions** (on failure + restart)
4. A **COMPLETED** JobInstance cannot re-run with same parameters
5. Steps execute in the order defined by `.start()` and `.next()`

### 🔗 Follow-up Questions
- *"What is the difference between JobInstance and JobExecution?"* → See Q56
- *"What happens if I run the same job with same parameters?"* → See Q59

---

<a id="q6"></a>

## Q6. What is a Step?

### 🔑 Quick Answer

> A Step is an **independent, sequential phase** within a Job. It comes in two types: **Chunk-based** (read-process-write loop) and **Tasklet** (single task execution).

### 📖 Step-by-Step Explanation

**Step 1 — Step = One phase of the batch process:**

Think of a Job as a cooking recipe. Each Step is one task in that recipe:

```
Job: "processMonthlyPayroll"
├── Step 1: Read all employees         (Chunk-based)
├── Step 2: Calculate salaries         (Chunk-based)
├── Step 3: Generate salary slips      (Chunk-based)
└── Step 4: Send email notifications   (Tasklet — single task)
```

**Step 2 — Two types of Steps:**

| Type | How it works | When to use |
|------|-------------|-------------|
| **Chunk-based** | Read N items → Process each → Write all N → Repeat | Processing data records |
| **Tasklet** | Execute a single task → Done | Cleanup, delete files, send notification |

**Step 3 — Chunk-based Step (most common):**

```
[Read item 1] → [Process item 1]  ─┐
[Read item 2] → [Process item 2]  ─┤→ [Write all 3 at once] → COMMIT
[Read item 3] → [Process item 3]  ─┘
                                       ↑
                                  This is ONE chunk
                                  Repeat until no more data
```

**Step 4 — Tasklet Step (simple tasks):**

```
Execute once → Return FINISHED → Step done
```

### 🗣️ How to Explain in Interview

> *"A Step is one independent phase of a Job. There are two types. Most common is Chunk-based — it reads data one item at a time, processes each item, and then writes the whole batch together. For example, read 500 employee records, calculate salary for each, then write all 500 to the database in one transaction. The second type is Tasklet — for single tasks like deleting temporary files or sending a notification email. You don't need read-process-write for that."*

### 💻 Code Example

```java
// CHUNK-BASED STEP — for processing data records
@Bean
public Step calculateSalariesStep(JobRepository repo,
                                   PlatformTransactionManager tx) {
    return new StepBuilder("calculateSalaries", repo)
            .<Employee, PaySlip>chunk(500, tx)   // Read Employee, Output PaySlip
            .reader(employeeReader())             // Read from DB
            .processor(salaryCalculator())        // Calculate salary
            .writer(paySlipWriter())              // Write pay slips
            .build();
}

// TASKLET STEP — for a single task
@Bean
public Step sendNotificationStep(JobRepository repo,
                                  PlatformTransactionManager tx) {
    return new StepBuilder("sendNotification", repo)
            .tasklet((contribution, chunkContext) -> {
                emailService.send("Payroll processing complete!");
                return RepeatStatus.FINISHED;  // Done, don't repeat
            }, tx)
            .build();
}
```

**Walkthrough:**
- `.<Employee, PaySlip>chunk(500, tx)` → Input=Employee, Output=PaySlip, 500 per chunk
- `RepeatStatus.FINISHED` → Tasklet runs once and stops
- `RepeatStatus.CONTINUABLE` → Tasklet runs again (for batch deletions, etc.)

### ⚡ Key Points to Remember

1. **Chunk-based** = Read → Process → Write loop (for data)
2. **Tasklet** = Single task execution (for cleanup, notifications)
3. Each Step has its own **StepExecution** tracking (read/write/skip counts)
4. Steps run in **sequence** (unless you configure parallel flows)
5. Each chunk = **1 transaction**

### 🔗 Follow-up Questions
- *"Difference between Job and Step?"* → See Q7
- *"When to use Tasklet vs Chunk?"* → See Q80
- *"What is chunk processing in detail?"* → See Q21

---

<a id="q7"></a>

## Q7. What is the difference between Job and Step?

### 🔑 Quick Answer

> **Job** = the entire batch process (like a book). **Step** = one independent phase within the Job (like a chapter). A Job has many Steps; each Step handles its own transactions.

### 📖 Step-by-Step Explanation

| Feature | Job | Step |
|---------|-----|------|
| **What is it** | The entire batch process | One phase of the process |
| **Contains** | One or more Steps | Reader + Processor + Writer (or Tasklet) |
| **Identified by** | Job name + parameters | Step name |
| **Tracking** | JobExecution (overall status) | StepExecution (read/write/skip counts) |
| **Transaction** | No direct transaction | Each chunk = 1 transaction |
| **Restart** | Restarts from the failed Step | Resumes from the last committed chunk |
| **Analogy** | 📕 A Book | 📄 A Chapter |

```
Job: "monthlyBilling"          ← This is the JOB
│
├── Step 1: readCustomers      ← This is STEP 1
│   ├── Reader: JdbcPagingItemReader
│   ├── Processor: ValidateCustomer
│   └── Writer: JdbcBatchItemWriter
│
├── Step 2: calculateBills     ← This is STEP 2
│   ├── Reader: JdbcCursorItemReader
│   ├── Processor: BillCalculator
│   └── Writer: JdbcBatchItemWriter
│
└── Step 3: sendEmails         ← This is STEP 3 (Tasklet)
    └── Tasklet: EmailSenderTasklet
```

### 🗣️ How to Explain in Interview

> *"A Job is the entire batch process — like 'process monthly billing.' A Step is one phase within that — like 'read customers,' 'calculate bills,' 'send emails.' The key difference is that transactions happen at the Step level, not the Job level. Each chunk inside a Step is one transaction. And when a Job restarts after failure, it doesn't start from the beginning — it skips completed Steps and resumes the failed Step from the last committed chunk."*

### ⚡ Key Points to Remember

1. **Job wraps Steps** — Job is parent, Steps are children
2. **Transactions live in Steps** — not in Job
3. **Restart granularity** — Job restarts from failed Step, Step restarts from failed chunk
4. Job has **one status**, each Step has **its own status**

---

<a id="q8"></a>

## Q8. What is ItemReader?

### 🔑 Quick Answer

> ItemReader is the component that **reads data from a source**, returning **one item at a time**. When there's no more data, it returns `null`.

### 📖 Step-by-Step Explanation

**Step 1 — The interface is simple:**

```java
public interface ItemReader<T> {
    T read() throws Exception;
    // Returns one item per call
    // Returns null = no more data (stop reading)
}
```

**Step 2 — How Spring Batch uses it:**

```
Spring Batch calls reader.read() repeatedly:

Call 1 → returns Employee{id=1, name="Amit"}
Call 2 → returns Employee{id=2, name="Priya"}
Call 3 → returns Employee{id=3, name="Rahul"}
...
Call N → returns null  ← "I'm done reading"
```

**Step 3 — Built-in readers (don't build from scratch):**

| Reader | Reads From | Thread-Safe? |
|--------|-----------|-------------|
| `FlatFileItemReader` | CSV, TXT, fixed-width files | ❌ |
| `JdbcCursorItemReader` | Database (keeps cursor open) | ❌ |
| `JdbcPagingItemReader` | Database (page by page) | ✅ |
| `JpaPagingItemReader` | Database via JPA entities | ✅ |
| `JsonItemReader` | JSON files | ❌ |
| `StaxEventItemReader` | XML files | ❌ |
| `KafkaItemReader` | Kafka topics | ✅ |
| `MongoItemReader` | MongoDB collections | ✅ |

### 🗣️ How to Explain in Interview

> *"ItemReader is the component that reads data from a source — one item per call. It returns null when there's no more data, which signals Spring Batch to stop reading. Spring provides many built-in readers — FlatFileItemReader for CSV files, JdbcPagingItemReader for database queries, JsonItemReader for JSON files, and so on. The most important thing when choosing a reader is thread safety — if you need multi-threaded processing, use a paging reader, not a cursor reader."*

### 💻 Code Example

```java
// Read employees from a CSV file
@Bean
public FlatFileItemReader<Employee> employeeReader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeReader")                         // Name for tracking
            .resource(new ClassPathResource("employees.csv")) // File location
            .delimited()                                    // CSV (comma-separated)
            .names("id", "name", "salary", "department")    // Column names
            .targetType(Employee.class)                     // Map to this class
            .build();
}
```

**What happens internally:**
1. Opens `employees.csv`
2. First `read()` call → reads line 1, maps to `Employee{id=1, name="Amit", salary=50000, department="IT"}`
3. Second `read()` call → reads line 2, maps to next Employee
4. Last line read → next `read()` returns `null` → Spring Batch stops reading

### ⚡ Key Points to Remember

1. Returns **one item per call**
2. Returns **null** = end of data
3. Choose **Paging readers** for thread safety
4. Choose **Cursor readers** for simple, sequential reads
5. **FlatFileItemReader** for files, **JdbcPagingItemReader** for databases

### 🔗 Follow-up Questions
- *"Cursor vs Paging reader?"* → See Q36
- *"Which reader for large datasets?"* → See Q37
- *"How to read multiple files?"* → See Q40

---

<a id="q9"></a>

## Q9. What is ItemProcessor?

### 🔑 Quick Answer

> ItemProcessor **transforms, validates, or filters** each item between reading and writing. It takes one input type and returns a different (or same) output type. **Returning null filters out the item**.

### 📖 Step-by-Step Explanation

**Step 1 — The interface:**

```java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
    // I = input type (what Reader gives)
    // O = output type (what Writer receives)
    // Return null = skip this item (filter)
}
```

**Step 2 — Three common use cases:**

```
1. TRANSFORM: Employee → PaySlip
   Input: Employee{name, salary}
   Output: PaySlip{name, netSalary, tax}

2. VALIDATE: Employee → Employee (or null)
   Input: Employee{name, salary=-5000}
   Output: null (filtered out — invalid salary)

3. ENRICH: Order → Order (with extra data)
   Input: Order{id, customerId}
   Output: Order{id, customerId, customerName, address}  ← added from DB lookup
```

**Step 3 — Processor is OPTIONAL:**

```java
// Without processor — directly read and write
.<Employee, Employee>chunk(100, tx)
    .reader(reader())
    .writer(writer())   // No .processor() — that's fine!
    .build();
```

### 🗣️ How to Explain in Interview

> *"ItemProcessor sits between the reader and writer. It receives each item from the reader and can do three things: transform it — like converting an Employee into a PaySlip; validate it — returning null to filter out invalid records; or enrich it — like looking up additional data from another service. It's optional — if your read and write types are the same and you don't need transformation, you can skip it. One important thing: if the processor returns null, the item is silently filtered, not treated as an error."*

### 💻 Code Example

```java
// Transform + Validate in one processor
@Bean
public ItemProcessor<Employee, PaySlip> salaryProcessor() {
    return employee -> {
        // VALIDATE — filter out inactive employees
        if (!employee.isActive()) {
            return null;  // This employee is SKIPPED (not written)
        }

        // TRANSFORM — convert Employee to PaySlip
        PaySlip paySlip = new PaySlip();
        paySlip.setEmployeeName(employee.getName());
        paySlip.setGrossSalary(employee.getSalary());
        paySlip.setTax(employee.getSalary() * 0.30);          // 30% tax
        paySlip.setNetSalary(employee.getSalary() * 0.70);    // After tax
        paySlip.setGeneratedDate(LocalDate.now());
        
        return paySlip;  // This goes to the Writer
    };
}
```

**Flow:**
```
Reader gives: Employee{name="Amit", salary=80000, active=true}
Processor returns: PaySlip{name="Amit", gross=80000, tax=24000, net=56000}
→ Goes to Writer ✅

Reader gives: Employee{name="Rahul", salary=60000, active=false}
Processor returns: null
→ Filtered out, NOT written ❌ (counted as filterCount, not error)
```

### ⚡ Key Points to Remember

1. **Optional** — you can skip it if not needed
2. **Return null** = item is filtered (not an error)
3. Can **change the type** — Input ≠ Output is fine
4. Runs **once per item** in the chunk
5. Filtered items tracked in `StepExecution.filterCount`

### 🔗 Follow-up Questions
- *"Can you chain multiple processors?"* → See Q51, Q52
- *"What if processor throws exception?"* → See Q53

---

<a id="q10"></a>

## Q10. What is ItemWriter?

### 🔑 Quick Answer

> ItemWriter **writes the entire chunk at once** to the destination (database, file, API). Unlike Reader which reads one-at-a-time, Writer receives **all processed items in one batch**.

### 📖 Step-by-Step Explanation

**Step 1 — The interface:**

```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> items) throws Exception;
    // Receives ALL items from the current chunk
    // Not one-at-a-time like Reader!
}
```

**Step 2 — Why write in batches?**

```
❌ Writing one-by-one (10,000 DB calls):
   INSERT record 1    → 1 network round trip
   INSERT record 2    → 1 network round trip
   INSERT record 3    → 1 network round trip
   ... × 10,000 = SLOW

✅ Writing as a batch (1 DB call):
   INSERT records 1-500  → 1 network round trip (500 rows at once!)
   INSERT records 501-1000 → 1 network round trip
   ... × 20 = FAST
```

**Step 3 — Built-in writers:**

| Writer | Writes To | Best For |
|--------|----------|----------|
| `JdbcBatchItemWriter` | Database via JDBC batch | Fastest for DB writes |
| `JpaItemWriter` | Database via JPA/Hibernate | When using entities |
| `FlatFileItemWriter` | CSV/TXT files | File output |
| `JsonFileItemWriter` | JSON files | API-style output |
| `KafkaItemWriter` | Kafka topics | Event publishing |
| `CompositeItemWriter` | Multiple destinations | Write to DB + file |

### 🗣️ How to Explain in Interview

> *"ItemWriter receives the entire chunk of processed items at once and writes them in a batch. This is different from the Reader which reads one item at a time. The reason is performance — instead of 500 individual INSERT statements, the writer does one batch insert with 500 rows, which is dramatically faster. Spring provides built-in writers like JdbcBatchItemWriter for database, FlatFileItemWriter for CSV output, and CompositeItemWriter if you need to write the same data to multiple destinations."*

### 💻 Code Example

```java
// Write to database using JDBC batch insert
@Bean
public JdbcBatchItemWriter<PaySlip> paySlipWriter(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<PaySlip>()
            .sql("INSERT INTO pay_slips (employee_name, gross_salary, tax, net_salary, date) " +
                 "VALUES (:employeeName, :grossSalary, :tax, :netSalary, :generatedDate)")
            .dataSource(dataSource)
            .beanMapped()   // Map PaySlip properties to :parameterNames
            .build();
}
```

**What happens at runtime (chunk size = 500):**
```
Spring Batch collects 500 processed PaySlip items
         │
         ▼
Writer receives: List<PaySlip> with 500 items
         │
         ▼
JdbcBatchItemWriter does:
  - PreparedStatement.addBatch() × 500 (add all to batch)
  - PreparedStatement.executeBatch()   (send ALL at once)
  - Transaction COMMIT
         │
         ▼
500 rows inserted in ONE database call
```

### ⚡ Key Points to Remember

1. Writer receives **entire chunk at once** (not one-by-one)
2. **Batch writes** = much faster than individual writes
3. `JdbcBatchItemWriter` = **fastest** for database
4. `JpaItemWriter` = slower (ORM overhead) but convenient with entities
5. Writer + Commit = end of one chunk transaction

### 🔗 Follow-up Questions
- *"How does batch insert work internally?"* → See Q46
- *"How to write to multiple destinations?"* → See Q45, Q47

---

<a id="q11"></a>

## Q11. What is JobRepository?

### 🔑 Quick Answer

> JobRepository is the **persistence mechanism** that stores all execution metadata — job status, step statistics, execution context — in database tables. It's what enables **restartability and monitoring**.

### 📖 Step-by-Step Explanation

**Step 1 — Why is it needed?**

Without JobRepository, Spring Batch has no memory. It wouldn't know:
- Did this job already complete? 
- Where did it fail last time?
- How many records were processed?

**Step 2 — What it stores:**

```
JobRepository manages 6 database tables:

BATCH_JOB_INSTANCE          → Unique job definitions
BATCH_JOB_EXECUTION         → Each run attempt (status, times)
BATCH_JOB_EXECUTION_PARAMS  → Parameters for each run
BATCH_JOB_EXECUTION_CONTEXT → Job-level shared state
BATCH_STEP_EXECUTION        → Per-step stats (read/write/skip counts)
BATCH_STEP_EXECUTION_CONTEXT → Step-level state (reader position)
```

**Step 3 — How it enables restartability:**

```
First Run:
  Job starts → JobRepository creates JobExecution (status=STARTED)
  Step 1 processes 5000 records → JobRepository saves: readCount=5000
  Step 2 crashes at record 3000 → JobRepository saves: status=FAILED, readCount=3000

Restart:
  Job restarts → JobRepository checks: "Step 1 was COMPLETED, skip it"
  Step 2 resumes → JobRepository reads: "last committed position was record 3000"
  Step 2 continues from record 3001
```

### 🗣️ How to Explain in Interview

> *"JobRepository is the heart of Spring Batch's reliability. It stores all execution metadata in 6 database tables — job status, step execution counts, execution context, and parameters. This is what makes restart possible — when a job fails at record 3000 and you re-run it, the JobRepository tells Spring Batch that Step 1 was completed and Step 2 failed at record 3000, so it resumes from 3001 instead of re-processing everything. It's mandatory — you can't run Spring Batch without it, though for testing you can use an in-memory version."*

### 💻 Code Example

```java
// Default: Spring Boot auto-configures JobRepository with your DataSource
// It creates BATCH_* tables automatically

// For testing: use in-memory (Map-based) repository
@Bean
public JobRepository jobRepository() throws Exception {
    JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
    factory.setDataSource(dataSource);
    factory.setTransactionManager(transactionManager);
    factory.setDatabaseType("POSTGRES");     // or MYSQL, H2, ORACLE
    factory.setTablePrefix("BATCH_");        // Default prefix
    factory.afterPropertiesSet();
    return factory.getObject();
}
```

### ⚡ Key Points to Remember

1. **Mandatory** for Spring Batch — stores all metadata
2. Uses **6 BATCH_* tables** in the database
3. Enables **restartability** (knows where job left off)
4. Enables **monitoring** (read/write/skip counts)
5. Prevents **duplicate execution** (same params can't run twice if completed)

### 🔗 Follow-up Questions
- *"What are the 6 tables?"* → See Q18, Q110
- *"How is ExecutionContext stored?"* → See Q68

---

<a id="q12"></a>

## Q12. What is JobLauncher?

### 🔑 Quick Answer

> JobLauncher is the **entry point** that **triggers** a Job with given parameters. It creates a JobExecution and starts the job.

### 📖 Step-by-Step Explanation

**Step 1 — Think of it as the "Start" button:**

```
YOU (or Scheduler or REST API)
      │
      ▼
  JobLauncher.run(job, parameters)
      │
      ├── Checks with JobRepository: "Has this already completed?"
      ├── Creates JobExecution
      ├── Starts the Job
      └── Returns JobExecution (with status, exit code)
```

**Step 2 — When does the JobLauncher get called?**

| Trigger | How |
|---------|-----|
| Application startup | Spring Boot auto-runs jobs by default |
| REST API | Controller calls `jobLauncher.run()` |
| Scheduler | `@Scheduled` method calls `jobLauncher.run()` |
| Command line | `CommandLineJobRunner` |

**Step 3 — Synchronous vs Asynchronous:**

```
Synchronous (default):
  jobLauncher.run(job, params)  → waits → returns when job completes

Asynchronous:
  jobLauncher.run(job, params)  → returns immediately → job runs in background
```

### 🗣️ How to Explain in Interview

> *"JobLauncher is the entry point to execute a batch job. You call jobLauncher.run() with the Job and JobParameters. It first checks with the JobRepository whether this job has already completed with these parameters — if yes, it throws an exception. If not, it creates a new JobExecution and starts the job. By default it's synchronous — it waits for the job to finish. But you can configure it as asynchronous with a TaskExecutor, which is useful when triggering jobs from a REST API."*

### 💻 Code Example

```java
// Triggering a job from a REST controller
@RestController
public class BatchController {

    private final JobLauncher jobLauncher;
    private final Job monthlyBillingJob;

    @PostMapping("/api/run-billing")
    public String runBilling(@RequestParam String month) {
        JobParameters params = new JobParametersBuilder()
                .addString("month", month)
                .addLong("timestamp", System.currentTimeMillis())  // Make unique
                .toJobParameters();

        JobExecution execution = jobLauncher.run(monthlyBillingJob, params);
        
        return "Job started! Status: " + execution.getStatus();
        // Returns: "Job started! Status: STARTED"
    }
}
```

### ⚡ Key Points to Remember

1. **Entry point** to start a job
2. Calls `jobLauncher.run(job, parameters)`
3. **Default = synchronous** (blocks until job completes)
4. Can be made **async** with `SimpleAsyncTaskExecutor`
5. Spring Boot **auto-launches** jobs on startup (disable with `spring.batch.job.enabled=false`)

---

<a id="q13"></a>

## Q13. What is JobParameters?

### 🔑 Quick Answer

> JobParameters are **input values** passed to a job that **uniquely identify** a JobInstance. Same job + same parameters = same JobInstance (can't re-run if completed).

### 📖 Step-by-Step Explanation

**Step 1 — Why parameters matter:**

```
Job: "monthlyBilling"
Parameters: { month: "2024-01" }
→ Creates JobInstance #1

Same job, same parameters: { month: "2024-01" }
→ SAME JobInstance #1 → CANNOT re-run (already completed)!

Different parameters: { month: "2024-02" }
→ Creates JobInstance #2 → Runs as new job
```

**Step 2 — Identifying vs Non-Identifying parameters:**

| Type | Affects Job Identity? | Example |
|------|----------------------|---------|
| **Identifying** (default) | ✅ Yes — changes JobInstance | month="2024-01" |
| **Non-Identifying** | ❌ No — just metadata | triggeredBy="John" |

```java
JobParameters params = new JobParametersBuilder()
    .addString("month", "2024-01")                          // Identifying (default)
    .addString("triggeredBy", "admin", false)                // Non-identifying
    .addLong("timestamp", System.currentTimeMillis())        // Identifying (makes each run unique)
    .toJobParameters();
```

**Step 3 — Accessing parameters in Reader/Processor/Writer:**

```java
@Bean
@StepScope  // Required for late binding!
public FlatFileItemReader<Employee> reader(
        @Value("#{jobParameters['fileName']}") String fileName) {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("reader")
            .resource(new FileSystemResource(fileName))
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"JobParameters are input values passed to a job — like month, fileName, or date. They serve two purposes: first, they provide runtime configuration to the job; second, and more importantly, they uniquely identify a JobInstance. So if I run 'monthlyBilling' with month=January, it creates one JobInstance. If I try to run it again with the same parameter, Spring Batch will refuse because it's already completed. To make each run unique, I usually add a timestamp parameter. There's also a concept of non-identifying parameters that don't affect uniqueness — useful for metadata like 'triggeredBy'."*

### ⚡ Key Points to Remember

1. **Job name + identifying params = unique JobInstance**
2. Completed job **cannot** re-run with same identifying params
3. Use `@StepScope` + `#{jobParameters['key']}` to access in beans
4. Add **timestamp** parameter to allow repeated runs
5. **Non-identifying** params = metadata only (don't affect uniqueness)

### 🔗 Follow-up Questions
- *"What happens if same parameters are used?"* → See Q59
- *"How to prevent duplicate execution?"* → See Q60

---

<a id="q14"></a>

## Q14. What is JobExecution?

### 🔑 Quick Answer

> JobExecution represents a **single attempt** to run a Job. A JobInstance can have **multiple JobExecutions** if the first attempt failed and was restarted.

### 📖 Step-by-Step Explanation

**Step 1 — The relationship:**

```
JobInstance (month=2024-01)        ← The logical run
├── JobExecution #1                ← First attempt
│   ├── Status: FAILED
│   ├── Start: 2024-01-15 02:00
│   ├── End: 2024-01-15 02:30
│   └── Exit: FAILED (Step 2 threw NullPointerException)
│
└── JobExecution #2                ← Second attempt (restart)
    ├── Status: COMPLETED
    ├── Start: 2024-01-15 03:00
    ├── End: 2024-01-15 03:45
    └── Exit: COMPLETED
```

**Step 2 — What JobExecution tracks:**

| Property | Description |
|----------|-------------|
| `status` | STARTING, STARTED, STOPPING, STOPPED, FAILED, COMPLETED, ABANDONED |
| `startTime` | When this execution began |
| `endTime` | When this execution finished |
| `exitStatus` | Exit code + description (can be custom) |
| `executionContext` | Key-value store for sharing state |
| `failureExceptions` | List of exceptions that caused failure |
| `jobParameters` | Parameters this execution was run with |

### 🗣️ How to Explain in Interview

> *"JobExecution is a single physical attempt to run a job. Think of it this way: if I have a monthly billing job for January, that's one JobInstance. If it fails the first time, that's JobExecution #1 with status FAILED. When I restart it, that creates JobExecution #2 under the same JobInstance. So a JobInstance is the 'what' — what job with what parameters — and JobExecution is the 'when' — each attempt to run it. The JobExecution tracks start time, end time, status, and the exceptions that occurred."*

### ⚡ Key Points to Remember

1. **One JobInstance → many JobExecutions** (on restart)
2. Tracks **status, times, exceptions, exit code**
3. COMPLETED execution → job won't re-run with same params
4. FAILED execution → can create new execution (restart)

---

<a id="q15"></a>

## Q15. What is StepExecution?

### 🔑 Quick Answer

> StepExecution represents a **single attempt to execute a Step** and tracks detailed statistics — how many records were read, written, skipped, filtered, and how many commits/rollbacks occurred.

### 📖 Step-by-Step Explanation

**Step 1 — StepExecution is WHERE the detail lives:**

JobExecution tells you: "Job FAILED."  
StepExecution tells you: "Step 2 FAILED after reading 48,500 records, writing 47,000, skipping 1,500, with 3 rollbacks."

**Step 2 — Key metrics tracked:**

```
StepExecution for "calculateSalaries":
├── status: COMPLETED
├── readCount: 50,000          ← Records read from source
├── writeCount: 48,500         ← Records written to destination
├── filterCount: 1,500         ← Filtered by Processor (returned null)
├── readSkipCount: 0           ← Skipped during read (bad CSV lines)
├── processSkipCount: 200      ← Skipped during process (validation fail)
├── writeSkipCount: 50         ← Skipped during write (constraint violation)
├── commitCount: 100           ← Number of committed chunks (50000/500)
├── rollbackCount: 3           ← Chunks that rolled back
├── startTime: 2024-01-15 02:00
└── endTime: 2024-01-15 02:30
```

**Step 3 — Why these metrics matter in production:**

```
readCount - writeCount - filterCount = skipCount (total skipped)
                                     = something went wrong with those records

commitCount × chunkSize ≈ readCount   (sanity check)

rollbackCount > 0 = some chunks failed (investigate!)
```

### 🗣️ How to Explain in Interview

> *"StepExecution has the real detail you need for monitoring and debugging. It tracks exactly how many records were read, written, skipped, and filtered for each step. For example, if a step read 50,000 records but only wrote 48,500, I can check: 1,500 were filtered by the processor, 200 were process-skipped, and 50 were write-skipped due to constraint violations. It also tracks commit count and rollback count, which helps with performance analysis — like how many chunks were processed versus how many failed."*

### ⚡ Key Points to Remember

1. **readCount, writeCount, filterCount, skipCount** — the key metrics
2. **commitCount** = number of successfully committed chunks
3. **rollbackCount** = number of chunks that failed
4. Essential for **debugging** failed jobs
5. Stored in `BATCH_STEP_EXECUTION` table

---

<a id="q16"></a>

## Q16. What is ExecutionContext?

### 🔑 Quick Answer

> ExecutionContext is a **persisted key-value store** (like a Map) that allows sharing state between steps and across restarts. It's saved to the database after every chunk commit.

### 📖 Step-by-Step Explanation

**Step 1 — Two scopes:**

```
Job ExecutionContext              Step ExecutionContext
├── Shared across ALL steps       ├── Private to ONE step
├── Stored in BATCH_JOB_          ├── Stored in BATCH_STEP_
│   EXECUTION_CONTEXT             │   EXECUTION_CONTEXT
└── Use for inter-step data       └── Use for checkpoint/position
```

**Step 2 — Real use case — sharing data between steps:**

```
Step 1: "countRecords"
  → Reads file, counts records
  → Saves to Job ExecutionContext: { "totalRecords": 50000 }

Step 2: "processRecords"
  → Reads from Job ExecutionContext: totalRecords = 50000
  → Uses it for progress logging: "Processing 500/50000 (1%)"
```

**Step 3 — Real use case — restart from last position:**

```
Step ExecutionContext after chunk 5:
  { "FlatFileItemReader.read.count": 2500 }   ← Reader is at line 2500

Job CRASHES here.

On RESTART:
  Spring Batch reads ExecutionContext from DB
  FlatFileItemReader sees: read.count = 2500
  Skips to line 2501 and continues reading!
```

### 🗣️ How to Explain in Interview

> *"ExecutionContext is a persisted Map that Spring Batch saves to the database. There are two scopes — Job-level and Step-level. Job-level context is shared across all steps, so if Step 1 counts total records and saves it, Step 2 can read that count. Step-level context is used internally for checkpointing — the FlatFileItemReader saves its current line position in the Step ExecutionContext after every chunk. So if the job crashes, on restart it reads the context from the database and knows to resume from line 2501 instead of line 1."*

### 💻 Code Example

```java
// Saving data in Step 1
@Bean
public Tasklet countRecordsTasklet() {
    return (contribution, chunkContext) -> {
        int count = countRecordsInFile("input.csv");
        
        // Save to JOB context (shared across steps)
        chunkContext.getStepContext()
                    .getStepExecution()
                    .getJobExecution()
                    .getExecutionContext()
                    .putInt("totalRecords", count);
        
        return RepeatStatus.FINISHED;
    };
}

// Reading data in Step 2
@Bean
@StepScope
public ItemProcessor<Employee, Employee> progressProcessor(
        @Value("#{jobExecutionContext['totalRecords']}") int total) {
    
    AtomicInteger current = new AtomicInteger(0);
    return employee -> {
        int processed = current.incrementAndGet();
        if (processed % 1000 == 0) {
            log.info("Progress: {}/{} ({}%)", processed, total, 
                     processed * 100 / total);
        }
        return employee;
    };
}
```

### ⚡ Key Points to Remember

1. **Job context** = shared between steps (inter-step communication)
2. **Step context** = private to step (checkpointing for restart)
3. **Persisted to DB** after every chunk commit
4. Survives **JVM restarts** — enables recovery
5. Access via `#{jobExecutionContext['key']}` or `#{stepExecutionContext['key']}`

---

<a id="q17"></a>

## Q17. What is the lifecycle of a Spring Batch job?

### 🔑 Quick Answer

> The lifecycle flows: **JobLauncher → creates JobExecution → executes Steps one by one → each Step runs chunk loop (read→process→write→commit) → updates status → COMPLETED or FAILED**.

### 📖 Step-by-Step Explanation

```
1. TRIGGER
   JobLauncher.run(job, params)
        │
2. VALIDATE
   ├── JobRepository checks: already completed with these params?
   │   ├── Yes → throw JobInstanceAlreadyCompleteException
   │   └── No  → continue
        │
3. CREATE
   ├── Create JobInstance (if new params) or reuse existing
   └── Create JobExecution (status = STARTED)
        │
4. EXECUTE STEPS (in order)
   ├── Step 1:
   │   ├── Create StepExecution (status = STARTED)
   │   ├── Open transaction
   │   ├── ┌─── CHUNK LOOP ────────────────────┐
   │   │   │ reader.read() × N items            │
   │   │   │ processor.process() × N items      │
   │   │   │ writer.write(chunk of N items)     │
   │   │   │ COMMIT transaction                 │
   │   │   │ Update StepExecution counts        │
   │   │   │ Save ExecutionContext to DB         │
   │   │   └───── Repeat until reader returns null ──┘
   │   └── StepExecution status = COMPLETED
   │
   ├── Step 2: (same pattern)
   └── Step 3: (same pattern)
        │
5. FINISH
   ├── All steps completed → JobExecution status = COMPLETED
   └── Any step failed → JobExecution status = FAILED
```

### 🗣️ How to Explain in Interview

> *"The lifecycle starts with the JobLauncher receiving a Job and parameters. First, it checks with the JobRepository whether this job already completed with these parameters. If not, it creates a JobExecution and starts executing steps in order. Each step runs a chunk loop: read N items, process each one, write the whole chunk, commit the transaction, save checkpoints to the database, and repeat until no more data. After each chunk commit, the execution state is saved — so if the JVM crashes, it can resume from the last committed chunk. When all steps complete, the job is marked COMPLETED."*

### ⚡ Key Points to Remember

1. **Validate first** — check for duplicate completions
2. **Steps run in order** — one at a time (unless parallel configured)
3. **Checkpoint saved after every chunk** — enables restart
4. **Status updated continuously** — STARTED → COMPLETED or FAILED
5. Entire lifecycle is **persisted in metadata tables**

---

<a id="q18"></a>

## Q18. What are the default tables created by Spring Batch?

### 🔑 Quick Answer

> Spring Batch creates **6 main tables** prefixed with `BATCH_` — storing job instances, executions, parameters, and execution context for both jobs and steps.

### 📖 Step-by-Step Explanation

```
BATCH_JOB_INSTANCE
├── One row per unique Job + Parameters combination
│
├── BATCH_JOB_EXECUTION
│   ├── One row per execution attempt (can be many per instance)
│   │
│   ├── BATCH_JOB_EXECUTION_PARAMS
│   │   └── Parameters for this execution
│   │
│   ├── BATCH_JOB_EXECUTION_CONTEXT
│   │   └── Job-level shared state (key-value pairs)
│   │
│   └── BATCH_STEP_EXECUTION
│       ├── One row per step attempt (read/write/skip counts)
│       │
│       └── BATCH_STEP_EXECUTION_CONTEXT
│           └── Step-level state (reader position for restart)
```

| Table | What It Stores | Key Columns |
|-------|---------------|-------------|
| `BATCH_JOB_INSTANCE` | Unique job definitions | JOB_NAME, JOB_KEY |
| `BATCH_JOB_EXECUTION` | Each run attempt | STATUS, START_TIME, END_TIME |
| `BATCH_JOB_EXECUTION_PARAMS` | Parameters per run | PARAMETER_NAME, PARAMETER_VALUE |
| `BATCH_JOB_EXECUTION_CONTEXT` | Job-level state (Map) | SHORT_CONTEXT (JSON) |
| `BATCH_STEP_EXECUTION` | Per-step statistics | READ_COUNT, WRITE_COUNT, SKIP_COUNT |
| `BATCH_STEP_EXECUTION_CONTEXT` | Step-level state | SHORT_CONTEXT (reader position) |

### 🗣️ How to Explain in Interview

> *"Spring Batch creates 6 metadata tables, all prefixed with BATCH_. BATCH_JOB_INSTANCE stores unique job definitions — one row per unique combination of job name and parameters. BATCH_JOB_EXECUTION stores each run attempt with status and timestamps. BATCH_JOB_EXECUTION_PARAMS stores the parameters for each run. Then there are two context tables — job-level and step-level — for persisting ExecutionContext as JSON. And BATCH_STEP_EXECUTION is the most detail-rich — it stores read count, write count, skip count, commit count, and rollback count for each step."*

### ⚡ Key Points to Remember

1. **6 tables** — all prefixed with `BATCH_`
2. `BATCH_STEP_EXECUTION` has the **most useful data** (counts)
3. Context tables store **JSON** (for restart and state sharing)
4. **Auto-created** by Spring Boot (or manually via schema SQL scripts)
5. Can change prefix with `spring.batch.jdbc.table-prefix`

---

<a id="q19"></a>

## Q19. What is the purpose of Spring Batch metadata tables?

### 🔑 Quick Answer

> They enable **restartability** (know where job stopped), **prevent duplicates** (completed jobs can't re-run), **monitoring** (track progress and statistics), and **auditing** (full execution history).

### 📖 Step-by-Step Explanation

| Purpose | How Tables Enable It |
|---------|---------------------|
| **Restartability** | Step ExecutionContext stores reader position → resume from last committed chunk |
| **Duplicate prevention** | Job Instance table tracks completed runs → same params can't re-run |
| **Monitoring** | Step Execution stores read/write/skip counts → know exactly what happened |
| **Auditing** | All executions stored with timestamps → full history |
| **State sharing** | Job ExecutionContext shared between steps → inter-step communication |
| **Error tracking** | Exit messages and exception details stored → debug failures |

### 🗣️ How to Explain in Interview

> *"The metadata tables serve four critical purposes. First, restartability — the step execution context stores the reader's position, so after a crash it resumes from where it stopped. Second, duplicate prevention — the job instance table ensures you can't re-run a completed job with the same parameters accidentally. Third, monitoring — the step execution table has exact read, write, and skip counts, so you know exactly how many records were processed. And fourth, auditing — every execution is stored with timestamps, so you have a complete history."*

### ⚡ Key Points to Remember

1. **Restart** = knows where to resume
2. **No duplicates** = completed won't re-run
3. **Monitoring** = read/write/skip counts
4. **Audit trail** = full execution history

---

<a id="q20"></a>

## Q20. How does Spring Batch maintain job state?

### 🔑 Quick Answer

> It persists **ExecutionContext** (checkpoint data) and **execution status** to the database after every chunk commit. On restart, it reads this state back to resume from the exact failure point.

### 📖 Step-by-Step Explanation

**Step 1 — State saved after EVERY chunk:**

```
Chunk 1 (records 1-500):
  Read 500, Process 500, Write 500, COMMIT
  → Save to DB: StepExecution.readCount=500
  → Save to DB: ExecutionContext.read.count=500

Chunk 2 (records 501-1000):
  Read 500, Process 500, Write 500, COMMIT
  → Save to DB: StepExecution.readCount=1000
  → Save to DB: ExecutionContext.read.count=1000

Chunk 3 (records 1001-1500):
  Read 300... JVM CRASHES! ← Not committed, so this is lost
```

**Step 2 — On restart, state is recovered:**

```
Spring Batch reads from DB:
  Step 1: COMPLETED → skip
  Step 2: FAILED, ExecutionContext.read.count=1000

Step 2 resumes:
  Reader opens file, skips to line 1001
  Continues processing from record 1001
  Records 1-1000 are NOT re-processed ✅
```

**Step 3 — Three types of state persisted:**

| State | Where Stored | Purpose |
|-------|-------------|---------|
| Job status | BATCH_JOB_EXECUTION | Know which steps to skip on restart |
| Step counts | BATCH_STEP_EXECUTION | Know exact progress |
| Checkpoint | BATCH_STEP_EXECUTION_CONTEXT | Know reader's position |

### 🗣️ How to Explain in Interview

> *"Spring Batch saves state after every chunk commit — both the execution counts and the execution context. The execution context stores things like the reader's current position. So if a job processes 10,000 records in chunks of 500, after each chunk it saves that the reader is at line 500, then 1000, then 1500. If the JVM crashes after processing 5000 records, on restart it reads the database and knows the reader was at line 5000, so it resumes from 5001. The key insight is that state is saved per CHUNK COMMIT, not per record — so you might lose at most one chunk's worth of work on crash."*

### ⚡ Key Points to Remember

1. State saved **after every chunk commit** (not per record)
2. On crash, you lose **at most one chunk** of work
3. Restart reads **ExecutionContext from DB** to find position
4. **Completed Steps are skipped** on restart
5. Failed Step **resumes from last committed chunk**

### 🔗 Follow-up Questions
- *"How does ExecutionContext work in detail?"* → See Q67
- *"How does restart work internally?"* → See Q61, Q69

---

> **🎯 Navigation:** [Next → Chunk Processing (Q21-30)](02-chunk-processing.md) | [📋 All Sections](README.md)
