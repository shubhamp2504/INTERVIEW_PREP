<![CDATA[<div align="center">

# 🟡 Spring Batch — Database & Metadata Questions (110-116)

[![Questions](https://img.shields.io/badge/Questions-7-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Medium-yellow.svg)](#)

</div>

---

<a id="q1"></a>
## Q110. ❓ What tables does Spring Batch create?

🔖 **Tags:** `#spring-batch` `#metadata` `#tables` `#must-know`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```
┌─────────────────────────────────────────────────┐
│           Spring Batch Metadata Schema           │
│                                                   │
│  BATCH_JOB_INSTANCE                               │
│    │ JOB_INSTANCE_ID (PK)                        │
│    │ JOB_NAME                                     │
│    │ JOB_KEY (hash of identifying params)         │
│    │                                               │
│    └──→ BATCH_JOB_EXECUTION (1:N)                │
│          │ JOB_EXECUTION_ID (PK)                  │
│          │ STATUS, EXIT_CODE, EXIT_MESSAGE        │
│          │ START_TIME, END_TIME, CREATE_TIME       │
│          │                                         │
│          ├──→ BATCH_JOB_EXECUTION_PARAMS (1:N)   │
│          │     PARAMETER_NAME, PARAMETER_TYPE     │
│          │     PARAMETER_VALUE, IDENTIFYING       │
│          │                                         │
│          ├──→ BATCH_JOB_EXECUTION_CONTEXT (1:1)  │
│          │     SHORT_CONTEXT, SERIALIZED_CONTEXT  │
│          │                                         │
│          └──→ BATCH_STEP_EXECUTION (1:N)         │
│                │ STEP_EXECUTION_ID (PK)           │
│                │ STEP_NAME, STATUS                 │
│                │ READ_COUNT, WRITE_COUNT           │
│                │ COMMIT_COUNT, ROLLBACK_COUNT      │
│                │ READ_SKIP_COUNT, WRITE_SKIP_COUNT │
│                │ FILTER_COUNT                       │
│                │ START_TIME, END_TIME               │
│                │                                    │
│                └──→ BATCH_STEP_EXECUTION_CONTEXT   │
│                      SHORT_CONTEXT                  │
│                      SERIALIZED_CONTEXT             │
│                                                     │
│  + Sequence tables: BATCH_JOB_SEQ,                 │
│    BATCH_JOB_EXECUTION_SEQ, BATCH_STEP_EXECUTION_SEQ│
└─────────────────────────────────────────────────────┘
```

---

<a id="q2"></a>
## Q111. ❓ What is BATCH_JOB_INSTANCE?

🔖 **Tags:** `#spring-batch` `#metadata` `#job-instance`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **unique job instances** (JobName + identifying parameters hash).

```sql
SELECT * FROM BATCH_JOB_INSTANCE;

| JOB_INSTANCE_ID | VERSION | JOB_NAME       | JOB_KEY                          |
|-----------------|---------|----------------|----------------------------------|
| 1               | 0       | dailyReport    | d41d8cd98f00b204e9800998ecf8427e |
| 2               | 0       | dailyReport    | 5ab2c8d7e1f3a9b4c6d8e0f2a4b6c8d0 |
| 3               | 0       | monthlyBilling | a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6 |
```

- `JOB_KEY` = MD5 hash of identifying JobParameters
- Same `JOB_NAME` + same `JOB_KEY` = same instance (won't create new)

---

<a id="q3"></a>
## Q112. ❓ What is BATCH_JOB_EXECUTION?

🔖 **Tags:** `#spring-batch` `#metadata` `#job-execution`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **each execution attempt** of a job instance.

```sql
SELECT JOB_EXECUTION_ID, JOB_INSTANCE_ID, STATUS, EXIT_CODE, 
       START_TIME, END_TIME
FROM BATCH_JOB_EXECUTION;

| JOB_EXECUTION_ID | JOB_INSTANCE_ID | STATUS    | EXIT_CODE | START_TIME          | END_TIME            |
|------------------|-----------------|-----------|-----------|---------------------|---------------------|
| 1                | 1               | FAILED    | FAILED    | 2026-03-13 02:00:00 | 2026-03-13 02:05:30 |
| 2                | 1               | COMPLETED | COMPLETED | 2026-03-13 03:00:00 | 2026-03-13 03:10:15 |
| 3                | 2               | COMPLETED | COMPLETED | 2026-03-14 02:00:00 | 2026-03-14 02:08:45 |
```

---

<a id="q4"></a>
## Q113. ❓ What is BATCH_STEP_EXECUTION?

🔖 **Tags:** `#spring-batch` `#metadata` `#step-execution`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **detailed statistics** for each step execution.

```sql
SELECT STEP_NAME, STATUS, READ_COUNT, WRITE_COUNT, COMMIT_COUNT,
       ROLLBACK_COUNT, READ_SKIP_COUNT, WRITE_SKIP_COUNT, FILTER_COUNT
FROM BATCH_STEP_EXECUTION WHERE JOB_EXECUTION_ID = 2;

| STEP_NAME    | STATUS    | READ  | WRITE | COMMITS | ROLLBACKS | SKIP_R | SKIP_W | FILTER |
|-------------|-----------|-------|-------|---------|-----------|--------|--------|--------|
| readData     | COMPLETED | 50000 | 50000 | 100     | 0         | 0      | 0      | 0      |
| processData  | COMPLETED | 50000 | 48500 | 97      | 3         | 5      | 10     | 1485   |
| sendReport   | COMPLETED | 0     | 0     | 1       | 0         | 0      | 0      | 0      |
```

### Reading This Data:
- **50,000** records read
- **48,500** written (1,500 filtered by processor)
- **100** chunks committed
- **3** chunks had rollbacks
- **5** read skips + **10** write skips = **15** total skipped records

---

<a id="q5"></a>
## Q114. ❓ What is BATCH_JOB_PARAMS?

🔖 **Tags:** `#spring-batch` `#metadata` `#job-params`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **parameters** passed to each job execution.

```sql
-- Spring Batch 5 format
SELECT JOB_EXECUTION_ID, PARAMETER_NAME, PARAMETER_TYPE, 
       PARAMETER_VALUE, IDENTIFYING
FROM BATCH_JOB_EXECUTION_PARAMS;

| JOB_EXECUTION_ID | PARAMETER_NAME | PARAMETER_TYPE | PARAMETER_VALUE     | IDENTIFYING |
|------------------|---------------|----------------|---------------------|-------------|
| 1                | file          | java.lang.String | /data/march.csv    | Y           |
| 1                | timestamp     | java.lang.Long   | 1710288000000      | Y           |
| 1                | description   | java.lang.String | Monthly processing | N           |
```

- `IDENTIFYING = Y` → Used to identify unique JobInstance
- `IDENTIFYING = N` → Metadata only, doesn't affect instance uniqueness

---

<a id="q6"></a>
## Q115. ❓ What is BATCH_STEP_EXECUTION_CONTEXT?

🔖 **Tags:** `#spring-batch` `#metadata` `#execution-context`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **serialized state** of each step — this is how restart works!

```sql
SELECT STEP_EXECUTION_ID, SHORT_CONTEXT
FROM BATCH_STEP_EXECUTION_CONTEXT;

| STEP_EXECUTION_ID | SHORT_CONTEXT                                                    |
|-------------------|-----------------------------------------------------------------|
| 1                 | {"@class":"java.util.HashMap","batch.taskletType":"..."}         |
| 2                 | {"FlatFileItemReader.read.count":25000,"batch.taskletType":"..."}|
```

- `FlatFileItemReader.read.count: 25000` → Reader has read 25,000 lines
- On restart: Reader will skip first 25,000 lines and resume from 25,001

---

<a id="q7"></a>
## Q116. ❓ What is BATCH_JOB_EXECUTION_CONTEXT?

🔖 **Tags:** `#spring-batch` `#metadata` `#execution-context`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

Stores **job-level shared state** — data shared between steps.

```sql
SELECT JOB_EXECUTION_ID, SHORT_CONTEXT
FROM BATCH_JOB_EXECUTION_CONTEXT;

| JOB_EXECUTION_ID | SHORT_CONTEXT                                              |
|------------------|------------------------------------------------------------|
| 1                | {"totalRecords":50000,"outputFile":"/reports/march.pdf"}   |
```

### Use Case:
```java
// Step 1: Count records, save to Job ExecutionContext
@AfterStep
public ExitStatus afterStep(StepExecution stepExecution) {
    stepExecution.getJobExecution().getExecutionContext()
        .putInt("totalRecords", stepExecution.getReadCount());
    return ExitStatus.COMPLETED;
}

// Step 3: Read total from Job ExecutionContext
@BeforeStep
public void beforeStep(StepExecution stepExecution) {
    int total = stepExecution.getJobExecution().getExecutionContext()
        .getInt("totalRecords");
    // Use in report generation
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Monitoring](./13-monitoring.md) | [Next: Production Scenarios →](./15-production-scenarios.md)
]]>