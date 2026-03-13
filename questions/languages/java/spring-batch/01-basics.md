
# 🟢 Spring Batch — Basic Questions (1-20)

[![Questions](https://img.shields.io/badge/Questions-20-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Easy-green.svg)](#)


---

<a id="q1"></a>

## Q1. ❓ What is Spring Batch?

🔖 **Tags:** `#spring-batch` `#basics` `#must-know` `#easy`  
📊 **Difficulty:** 🟢 Easy  
🔥 **Frequency:** ⭐⭐⭐⭐⭐ (Always Asked)

### ✅ Answer

**Spring Batch** is a lightweight, comprehensive **batch processing framework** built on top of the Spring Framework. It is designed to enable the development of robust batch applications that are vital for daily operations of enterprise systems.

### 🎯 Key Points:
- It provides **reusable functions** for processing large volumes of records (read, process, write)
- Handles **transaction management**, **job processing statistics**, **job restart**, **skip**, and **resource management**
- It does **NOT** do scheduling — it works with schedulers (Quartz, Cron, Spring Scheduler)
- Follows the **chunk-oriented processing** model

### 🏗️ Spring Batch = Spring Framework + Batch Processing Patterns

```
┌──────────────────────────────────┐
│          Spring Batch            │
│  ┌───────────┐  ┌────────────┐  │
│  │  Reading   │→│ Processing │→│ Writing │
│  │ (Source)   │  │ (Transform)│  │ (Dest) │
│  └───────────┘  └────────────┘  └────────┘
│                                            │
│  + Transaction Management                  │
│  + Error Handling (Skip/Retry)             │
│  + Job Restart & Recovery                  │
│  + Parallel Processing                     │
│  + Metadata & Monitoring                   │
└──────────────────────────────────┘
```

### 💻 Minimal Example:

```java
@Configuration
@EnableBatchProcessing
public class BatchConfig {

    @Bean
    public Job myJob(JobRepository jobRepository, Step step1) {
        return new JobBuilder("myJob", jobRepository)
                .start(step1)
                .build();
    }

    @Bean
    public Step step1(JobRepository jobRepository, 
                      PlatformTransactionManager txManager) {
        return new StepBuilder("step1", jobRepository)
                .<String, String>chunk(100, txManager)
                .reader(reader())
                .processor(processor())
                .writer(writer())
                .build();
    }
}
```

### 📌 Key Takeaway
> 💡 Spring Batch = **Read → Process → Write** in chunks with built-in error handling, restartability, and transaction management.

---

<a id="q2"></a>

## Q2. ❓ What are the main use cases of Spring Batch?

🔖 **Tags:** `#spring-batch` `#use-cases` `#must-know`  
📊 **Difficulty:** 🟢 Easy  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

| # | Use Case | Example |
|---|----------|---------|
| 1 | **ETL (Extract-Transform-Load)** | Read CSV → Transform → Write to DB |
| 2 | **Data Migration** | Migrate from legacy DB to new system |
| 3 | **Report Generation** | Generate daily/monthly reports |
| 4 | **File Processing** | Process uploaded CSV/XML/JSON files |
| 5 | **Database Cleanup** | Archive old records, purge expired data |
| 6 | **Billing & Invoicing** | Generate monthly bills for millions of customers |
| 7 | **Notification Processing** | Send bulk emails/SMS notifications |
| 8 | **Data Synchronization** | Sync data between multiple systems |
| 9 | **Bulk Updates** | Update prices, rates, statuses in bulk |
| 10 | **Compliance & Auditing** | Process regulatory data, generate audit reports |

### 📌 Key Takeaway
> 💡 If you need to process **large volumes of data** in a **reliable, repeatable way** — Spring Batch is the answer.

---

<a id="q3"></a>

## Q3. ❓ What problems does Spring Batch solve?

🔖 **Tags:** `#spring-batch` `#basics` `#why`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

| Problem | How Spring Batch Solves It |
|---------|---------------------------|
| **Processing millions of records** | Chunk-oriented processing with configurable chunk size |
| **Error handling** | Built-in Skip & Retry logic |
| **Job failure recovery** | Automatic restartability from failure point |
| **Transaction management** | Chunk-level transactions with rollback |
| **Monitoring** | Job/Step metadata stored in DB (status, time, count) |
| **Parallel processing** | Multi-threading, Partitioning, Remote chunking |
| **Code reusability** | Standard Reader/Processor/Writer pattern |
| **Scalability** | Horizontal scaling with partitioning |

---

<a id="q4"></a>

## Q4. ❓ What are the core components of Spring Batch?

🔖 **Tags:** `#spring-batch` `#architecture` `#must-know`  
📊 **Difficulty:** 🟢 Easy  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

```
┌─────────────────────────────────────────────┐
│              Core Components                 │
├─────────────────────────────────────────────┤
│                                             │
│  JobLauncher ──→ Job ──→ Step(s)            │
│                          │                  │
│                    ┌─────┴──────┐           │
│                    │            │           │
│               Chunk-Based   Tasklet         │
│               │                             │
│        ┌──────┼──────┐                      │
│     Reader  Processor  Writer               │
│                                             │
│  JobRepository (stores metadata)            │
│  JobParameters (identifies job instance)    │
│  ExecutionContext (shared state)             │
└─────────────────────────────────────────────┘
```

| Component | Role |
|-----------|------|
| **Job** | Represents the entire batch process |
| **Step** | Independent phase within a Job |
| **ItemReader** | Reads data from source |
| **ItemProcessor** | Transforms/validates data |
| **ItemWriter** | Writes data to destination |
| **JobRepository** | Stores job metadata in DB |
| **JobLauncher** | Triggers job execution |
| **JobParameters** | Input parameters to identify job instance |
| **ExecutionContext** | Key-value store for sharing state |

---

<a id="q5"></a>

## Q5. ❓ What is a Job in Spring Batch?

🔖 **Tags:** `#spring-batch` `#job` `#must-know`  
📊 **Difficulty:** 🟢 Easy  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

A **Job** is the **top-level container** that represents the entire batch process. It is composed of one or more **Steps**.

### 🎯 Key Points:
- A Job is a **sequence of Steps**
- Each Job has a unique **name**
- A Job is identified by `JobName + JobParameters` → creates a `JobInstance`
- Each time a Job runs, it creates a `JobExecution`

```java
@Bean
public Job orderProcessingJob(JobRepository jobRepository,
                               Step readOrdersStep,
                               Step processPaymentsStep,
                               Step generateReportStep) {
    return new JobBuilder("orderProcessingJob", jobRepository)
            .start(readOrdersStep)           // Step 1
            .next(processPaymentsStep)       // Step 2
            .next(generateReportStep)        // Step 3
            .build();
}
```

### 📊 Job Hierarchy:

```
Job: "orderProcessingJob"
├── JobInstance (job name + parameters = unique identity)
│   ├── JobExecution #1 (FAILED at Step 2)
│   └── JobExecution #2 (COMPLETED — restarted from Step 2)
└── Steps:
    ├── Step 1: readOrdersStep
    ├── Step 2: processPaymentsStep 
    └── Step 3: generateReportStep
```

---

<a id="q6"></a>

## Q6. ❓ What is a Step?

🔖 **Tags:** `#spring-batch` `#step` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

A **Step** is an **independent, sequential phase** of a Job. Each Step encapsulates a single unit of work.

### Two Types of Steps:

| Type | Description | Use Case |
|------|-------------|----------|
| **Chunk-Based** | Read → Process → Write in chunks | Processing records from DB/file |
| **Tasklet** | Execute a single task | Delete temp files, send notification |

```java
// Chunk-Based Step
@Bean
public Step processDataStep(JobRepository repo, 
                            PlatformTransactionManager tx) {
    return new StepBuilder("processDataStep", repo)
            .<InputData, OutputData>chunk(500, tx)
            .reader(itemReader())
            .processor(itemProcessor())
            .writer(itemWriter())
            .build();
}

// Tasklet Step
@Bean
public Step cleanupStep(JobRepository repo, 
                        PlatformTransactionManager tx) {
    return new StepBuilder("cleanupStep", repo)
            .tasklet((contribution, chunkContext) -> {
                // cleanup logic here
                Files.deleteIfExists(Path.of("/tmp/batch-temp.csv"));
                return RepeatStatus.FINISHED;
            }, tx)
            .build();
}
```

---

<a id="q7"></a>

## Q7. ❓ What is the difference between Job and Step?

🔖 **Tags:** `#spring-batch` `#comparison` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

| Feature | Job | Step |
|---------|-----|------|
| **What** | Entire batch process | Single phase within a Job |
| **Contains** | One or more Steps | Reader + Processor + Writer OR Tasklet |
| **Execution tracking** | JobExecution | StepExecution |
| **Parameters** | JobParameters | Gets parameters via StepExecution |
| **Restart** | Restarts from failed Step | Can restart from failed chunk |
| **Transaction** | No direct transaction | Each chunk = 1 transaction |
| **Analogy** | 📕 A Book | 📄 A Chapter |

```
Job ("monthlyBilling")
├── Step 1: "readCustomers"      ← Chunk-based (Read from DB)
├── Step 2: "calculateBills"     ← Chunk-based (Process + Write)
├── Step 3: "generateReport"     ← Tasklet (Create PDF)
└── Step 4: "sendNotifications"  ← Tasklet (Send emails)
```

---

<a id="q8"></a>

## Q8. ❓ What is ItemReader?

🔖 **Tags:** `#spring-batch` `#reader` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**ItemReader** is the component responsible for **reading data** from a source (file, database, queue, API, etc.), **one item at a time**.

```java
public interface ItemReader<T> {
    T read() throws Exception;
    // Returns null when there's no more data to read
}
```

### 🎯 Key Points:
- Reads **one item** per call
- Returns `null` when all data is read (signals end of input)
- Spring provides many built-in implementations

### Built-in Readers:

| Reader | Source |
|--------|--------|
| `FlatFileItemReader` | CSV, TXT, fixed-width files |
| `JdbcCursorItemReader` | DB via JDBC cursor |
| `JdbcPagingItemReader` | DB via JDBC paging |
| `JpaPagingItemReader` | DB via JPA paging |
| `JsonItemReader` | JSON files |
| `StaxEventItemReader` | XML files |
| `KafkaItemReader` | Kafka topics |
| `MongoItemReader` | MongoDB collections |

```java
@Bean
public FlatFileItemReader<Employee> reader() {
    return new FlatFileItemReaderBuilder<Employee>()
            .name("employeeReader")
            .resource(new ClassPathResource("employees.csv"))
            .delimited()
            .names("id", "name", "salary", "department")
            .targetType(Employee.class)
            .build();
}
```

---

<a id="q9"></a>

## Q9. ❓ What is ItemProcessor?

🔖 **Tags:** `#spring-batch` `#processor` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**ItemProcessor** is the component that **transforms, validates, or filters** data between reading and writing.

```java
public interface ItemProcessor<I, O> {
    O process(I item) throws Exception;
    // Return null to SKIP/FILTER this item
    // Return transformed item to pass to writer
}
```

### 🎯 Key Points:
- **Optional** — you can skip it if no transformation needed
- Input type (I) can be different from Output type (O)
- Returning `null` = **filter out** this item (won't be written)
- Can do: validation, transformation, enrichment, filtering

```java
@Bean
public ItemProcessor<RawEmployee, Employee> processor() {
    return rawEmployee -> {
        // Filter: skip inactive employees
        if (!rawEmployee.isActive()) {
            return null; // filtered out!
        }
        
        // Transform: create clean Employee object
        Employee emp = new Employee();
        emp.setName(rawEmployee.getName().toUpperCase());
        emp.setSalary(rawEmployee.getSalary() * 1.10); // 10% raise
        emp.setDepartment(rawEmployee.getDept().trim());
        return emp;
    };
}
```

---

<a id="q10"></a>

## Q10. ❓ What is ItemWriter?

🔖 **Tags:** `#spring-batch` `#writer` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**ItemWriter** is the component responsible for **writing data** to a destination. Unlike reader (one at a time), it writes a **whole chunk at once**.

```java
public interface ItemWriter<T> {
    void write(Chunk<? extends T> chunk) throws Exception;
    // Receives an entire chunk of items to write at once
}
```

### 🎯 Key Points:
- Receives a **list of items** (chunk), not one at a time
- This enables **batch inserts** for better performance
- One transaction per chunk write

### Built-in Writers:

| Writer | Destination |
|--------|------------|
| `FlatFileItemWriter` | CSV, TXT files |
| `JdbcBatchItemWriter` | DB via JDBC batch insert |
| `JpaItemWriter` | DB via JPA |
| `JsonFileItemWriter` | JSON files |
| `StaxEventItemWriter` | XML files |
| `KafkaItemWriter` | Kafka topics |
| `CompositeItemWriter` | Multiple writers |

```java
@Bean
public JdbcBatchItemWriter<Employee> writer(DataSource dataSource) {
    return new JdbcBatchItemWriterBuilder<Employee>()
            .sql("INSERT INTO employees (name, salary, dept) VALUES (:name, :salary, :department)")
            .dataSource(dataSource)
            .beanMapped()
            .build();
}
```

---

<a id="q11"></a>

## Q11. ❓ What is JobRepository?

🔖 **Tags:** `#spring-batch` `#job-repository` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**JobRepository** is the **persistence mechanism** that stores all batch metadata — job instances, executions, step executions, parameters, and execution contexts.

```
┌─────────────────────────────────────┐
│          JobRepository              │
│  (Stores in BATCH_* tables)         │
│                                     │
│  ├── BATCH_JOB_INSTANCE             │
│  ├── BATCH_JOB_EXECUTION            │
│  ├── BATCH_JOB_EXECUTION_PARAMS     │
│  ├── BATCH_JOB_EXECUTION_CONTEXT    │
│  ├── BATCH_STEP_EXECUTION           │
│  └── BATCH_STEP_EXECUTION_CONTEXT   │
└─────────────────────────────────────┘
```

### 🎯 Key Points:
- **Required** for Spring Batch to function
- Stores job status (STARTED, COMPLETED, FAILED)
- Enables **restartability** (knows where job failed)
- Can use **JDBC** (production) or **in-memory** (testing)

```java
// Production: Uses database (auto-configured with Spring Boot)
// Testing: In-memory
@Bean
public JobRepository jobRepository() throws Exception {
    JobRepositoryFactoryBean factory = new JobRepositoryFactoryBean();
    factory.setDataSource(dataSource);
    factory.setTransactionManager(transactionManager);
    factory.afterPropertiesSet();
    return factory.getObject();
}
```

---

<a id="q12"></a>

## Q12. ❓ What is JobLauncher?

🔖 **Tags:** `#spring-batch` `#job-launcher`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**JobLauncher** is the component that **starts/triggers** a Job with given JobParameters.

```java
public interface JobLauncher {
    JobExecution run(Job job, JobParameters params) throws ...;
}
```

### 🎯 Key Points:
- Validates that job hasn't already been completed with same parameters
- Creates a new `JobExecution`
- Can run **synchronously** (`SimpleJobLauncher`) or **asynchronously** (with `TaskExecutor`)

```java
@Autowired
private JobLauncher jobLauncher;

@Autowired
private Job myJob;

public void triggerJob() throws Exception {
    JobParameters params = new JobParametersBuilder()
            .addString("date", "2026-03-13")
            .addLong("time", System.currentTimeMillis())
            .toJobParameters();
    
    JobExecution execution = jobLauncher.run(myJob, params);
    System.out.println("Status: " + execution.getStatus());
}
```

---

<a id="q13"></a>

## Q13. ❓ What is JobParameters?

🔖 **Tags:** `#spring-batch` `#job-parameters`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**JobParameters** is a set of parameters passed to a Job that **uniquely identifies a JobInstance**.

### 🎯 Key Points:
- `JobName + JobParameters` = unique `JobInstance`
- Same job + same parameters = **same instance** (won't run again if completed)
- Supported types: `String`, `Long`, `Double`, `Date`
- Parameters can be **identifying** (default) or **non-identifying**

```java
JobParameters params = new JobParametersBuilder()
        .addString("inputFile", "data-march.csv")    // identifying
        .addLocalDate("date", LocalDate.now())        // identifying
        .addLong("time", System.currentTimeMillis())  // makes each run unique
        .toJobParameters();
```

### ⚠️ Common Pitfall:
> If you run the same job with the same parameters twice, Spring Batch will throw `JobInstanceAlreadyCompleteException`. Add a unique parameter (like timestamp) to create a new instance each time.

---

<a id="q14"></a>

## Q14. ❓ What is JobExecution?

🔖 **Tags:** `#spring-batch` `#job-execution`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**JobExecution** represents a **single attempt** to run a Job. A JobInstance can have multiple JobExecutions (if it fails and is restarted).

```
JobInstance: "dailyReport" + params(date=2026-03-13)
├── JobExecution #1: FAILED  (started 10:00, ended 10:05)
├── JobExecution #2: FAILED  (started 11:00, ended 11:02)
└── JobExecution #3: COMPLETED (started 12:00, ended 12:10)
```

### Key Properties:

| Property | Description |
|----------|-------------|
| `status` | STARTING, STARTED, STOPPING, STOPPED, FAILED, COMPLETED, ABANDONED |
| `startTime` | When execution started |
| `endTime` | When execution ended |
| `exitStatus` | COMPLETED, FAILED, or custom status |
| `executionContext` | Key-value data shared across steps |
| `failureExceptions` | List of exceptions that occurred |

---

<a id="q15"></a>

## Q15. ❓ What is StepExecution?

🔖 **Tags:** `#spring-batch` `#step-execution`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**StepExecution** represents a **single attempt** to execute a Step. It contains detailed statistics about the step's execution.

### Key Properties:

| Property | Description |
|----------|-------------|
| `readCount` | Number of items successfully read |
| `writeCount` | Number of items successfully written |
| `commitCount` | Number of transactions committed |
| `rollbackCount` | Number of transactions rolled back |
| `readSkipCount` | Number of items skipped during read |
| `writeSkipCount` | Number of items skipped during write |
| `processSkipCount` | Number of items skipped during process |
| `filterCount` | Number of items filtered by processor |
| `status` | COMPLETED, FAILED, etc. |

```java
// Access in a listener
@AfterStep
public ExitStatus afterStep(StepExecution stepExecution) {
    log.info("Read: {}", stepExecution.getReadCount());
    log.info("Written: {}", stepExecution.getWriteCount());
    log.info("Skipped: {}", stepExecution.getSkipCount());
    return stepExecution.getExitStatus();
}
```

---

<a id="q16"></a>

## Q16. ❓ What is ExecutionContext?

🔖 **Tags:** `#spring-batch` `#execution-context`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

**ExecutionContext** is a **key-value store** (like a `Map`) that allows you to persist state between steps or across restarts.

### Two Scopes:

| Scope | Shared Between | Stored In |
|-------|---------------|-----------|
| **Job ExecutionContext** | All steps in a job | `BATCH_JOB_EXECUTION_CONTEXT` |
| **Step ExecutionContext** | Within a single step (survives restart) | `BATCH_STEP_EXECUTION_CONTEXT` |

```java
// Save data in Step 1
@AfterStep
public ExitStatus afterStep(StepExecution stepExecution) {
    stepExecution.getJobExecution()
        .getExecutionContext()
        .putInt("totalRecords", 50000);
    return ExitStatus.COMPLETED;
}

// Read data in Step 2
@BeforeStep
public void beforeStep(StepExecution stepExecution) {
    int total = stepExecution.getJobExecution()
        .getExecutionContext()
        .getInt("totalRecords");
}
```

### 📌 Key Takeaway
> 💡 ExecutionContext is **serialized to DB** — this is how Spring Batch knows where to resume after a restart!

---

<a id="q17"></a>

## Q17. ❓ What is the lifecycle of a Spring Batch job?

🔖 **Tags:** `#spring-batch` `#lifecycle` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

```
1. JobLauncher.run(job, params)
   │
2. JobRepository creates JobInstance (if new params)
   │
3. JobRepository creates JobExecution
   │
4. Job starts executing Steps sequentially
   │
5. For each Step:
   │  ├── Create StepExecution
   │  ├── Open Reader
   │  │
   │  ├── LOOP (until reader returns null):
   │  │   ├── Read items (one at a time, up to chunk size)
   │  │   ├── Process each item
   │  │   ├── Write entire chunk
   │  │   └── Commit transaction
   │  │
   │  ├── Close Reader
   │  └── Update StepExecution status
   │
6. Update JobExecution status
   │
7. Job ends (COMPLETED / FAILED)
```

### Status Transitions:

```
STARTING → STARTED → COMPLETED ✅
                   → FAILED ❌ → (restart) → STARTED → COMPLETED ✅
                   → STOPPED ⏹ → (restart) → STARTED → COMPLETED ✅
```

---

<a id="q18"></a>

## Q18. ❓ What are the default tables created by Spring Batch?

🔖 **Tags:** `#spring-batch` `#metadata-tables` `#must-know`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

Spring Batch creates **6 metadata tables**:

| Table | Purpose |
|-------|---------|
| `BATCH_JOB_INSTANCE` | Unique job + parameters combination |
| `BATCH_JOB_EXECUTION` | Each run attempt of a job instance |
| `BATCH_JOB_EXECUTION_PARAMS` | Parameters passed to the job |
| `BATCH_JOB_EXECUTION_CONTEXT` | Job-level shared state (serialized) |
| `BATCH_STEP_EXECUTION` | Each run attempt of a step |
| `BATCH_STEP_EXECUTION_CONTEXT` | Step-level state (serialized) |

Plus a **sequence table**: `BATCH_JOB_SEQ`, `BATCH_JOB_EXECUTION_SEQ`, `BATCH_STEP_EXECUTION_SEQ`

```sql
-- Check job status
SELECT JOB_INSTANCE_ID, JOB_NAME, STATUS, START_TIME, END_TIME
FROM BATCH_JOB_EXECUTION
ORDER BY CREATE_TIME DESC;
```

---

<a id="q19"></a>

## Q19. ❓ What is the purpose of Spring Batch metadata tables?

🔖 **Tags:** `#spring-batch` `#metadata`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

| Purpose | How |
|---------|-----|
| **Restartability** | Knows which step/chunk failed → resume from there |
| **Prevent Duplicates** | Same job+params won't run twice if completed |
| **Monitoring** | Track status, read/write counts, duration |
| **Auditing** | When did job run, how many records processed |
| **State Persistence** | ExecutionContext survives JVM restarts |
| **Error Tracking** | Stores exception info for failed jobs |

> 💡 Without metadata tables, Spring Batch cannot restart jobs, prevent duplicates, or maintain state.

---

<a id="q20"></a>

## Q20. ❓ How does Spring Batch maintain job state?

🔖 **Tags:** `#spring-batch` `#state-management`  
📊 **Difficulty:** 🟢 Easy

### ✅ Answer

Spring Batch maintains state through **3 mechanisms**:

### 1️⃣ JobExecution Status
```
Stored in: BATCH_JOB_EXECUTION
Tracks: STARTED, COMPLETED, FAILED, STOPPED
```

### 2️⃣ StepExecution Counts
```
Stored in: BATCH_STEP_EXECUTION
Tracks: readCount, writeCount, commitCount, skipCount
→ On restart: knows exactly how many records were already processed
```

### 3️⃣ ExecutionContext
```
Stored in: BATCH_STEP_EXECUTION_CONTEXT
Tracks: Custom state like "current line number", "last processed ID"
→ On restart: reader can resume from exact position
```

### Example Flow:
```
First Run:
  Step1 → read 10,000 of 50,000 → FAILED (DB connection lost)
  State saved: { readCount: 10000, lastId: 10000 }

Restart:
  Step1 → resumes from record 10,001 → reads remaining 40,000
  → COMPLETED
```

### 📌 Key Takeaway
> 💡 Spring Batch persists ALL state to the database. Even if the JVM crashes, the job can resume from exactly where it left off.

---

[← Back to Spring Batch Index](./README.md) | [Next: Chunk Processing →](./02-chunk-processing.md)
]]>
