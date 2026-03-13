<![CDATA[<div align="center">

# 🔴 Spring Batch — Transaction & Restart Questions (63-69)

[![Questions](https://img.shields.io/badge/Questions-7-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Hard-red.svg)](#)

</div>

---

<a id="q1"></a>
## Q63. ❓ How does Spring Batch handle transactions?

🔖 **Tags:** `#spring-batch` `#transactions` `#must-know` `#frequently-asked`  
📊 **Difficulty:** 🔴 Hard  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

Spring Batch manages transactions at the **chunk level** — each chunk is an atomic transaction unit.

```
┌─── Transaction Boundary ───────────────┐
│                                         │
│  Read item1 → Process item1            │
│  Read item2 → Process item2            │
│  Read item3 → Process item3            │
│  Write [item1, item2, item3]           │
│                                         │
│  ✅ COMMIT (all or nothing)             │
└─────────────────────────────────────────┘
```

### Transaction Rules:
| Rule | Detail |
|------|--------|
| **Granularity** | 1 chunk = 1 transaction |
| **Manager** | Uses `PlatformTransactionManager` |
| **Read** | Reads happen INSIDE the transaction |
| **Write** | Writes happen INSIDE the transaction |
| **Metadata** | JobRepository updates happen in SEPARATE transaction |
| **Failure** | Only current chunk rolls back |
| **Committed chunks** | Safe, never rolled back |

### Custom Transaction Configuration:
```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    DefaultTransactionAttribute txAttr = new DefaultTransactionAttribute();
    txAttr.setIsolationLevel(TransactionDefinition.ISOLATION_READ_COMMITTED);
    txAttr.setPropagationBehavior(TransactionDefinition.PROPAGATION_REQUIRED);
    txAttr.setTimeout(30);  // 30 seconds timeout per chunk

    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .writer(writer())
            .transactionAttribute(txAttr)
            .build();
}
```

---

<a id="q2"></a>
## Q64. ❓ What is rollback in Spring Batch?

🔖 **Tags:** `#spring-batch` `#rollback` `#transactions`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

When an exception occurs during chunk processing, the **current chunk's transaction is rolled back**.

```
Chunk Processing (chunk size = 3):

Chunk 1: Read→Process→Write [A,B,C] → COMMIT ✅
Chunk 2: Read→Process→Write [D,E,F] → ERROR on E → ROLLBACK ❌
                                        D,E,F NOT written

After rollback:
- Database state: Only A,B,C exist (from chunk 1)
- StepExecution: readCount=6, writeCount=3, rollbackCount=1
```

### What Gets Rolled Back:
| Rolls Back | Does NOT Roll Back |
|-----------|-------------------|
| Current chunk's DB writes | Previously committed chunks |
| Current chunk's JMS messages | File writes (need manual cleanup!) |
| Current JPA/Hibernate changes | External API calls already made |

### ⚠️ Important:
> File writes (`FlatFileItemWriter`) are NOT transactional. If chunk fails after writing to file, those lines still exist in the file. Use `shouldDeleteIfEmpty` or cleanup logic.

---

<a id="q3"></a>
## Q65. ❓ What happens when a chunk fails?

🔖 **Tags:** `#spring-batch` `#chunk-failure` `#error-handling`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
Chunk fails → What happens?

1. Transaction ROLLBACK for current chunk
2. Check: Is fault tolerance configured?
   │
   ├── NO → Step FAILS → Job FAILS
   │
   └── YES → Check exception type:
       │
       ├── Skippable? → Enter scan mode:
       │   Write items one-by-one to find bad item
       │   Skip bad item, commit good items
       │   Continue with next chunk
       │
       ├── Retryable? → Retry entire chunk N times
       │   If still fails → check skip
       │   If not skippable → step FAILS
       │
       └── No-skip/no-retry? → Step FAILS → Job FAILS
```

---

<a id="q4"></a>
## Q66. ❓ How does Spring Batch support restartability?

🔖 **Tags:** `#spring-batch` `#restartability` `#must-know`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

Restartability is achieved through **3 mechanisms**:

### 1️⃣ Step Status Tracking
```
On restart: Skip steps with status = COMPLETED
            Re-execute steps with status = FAILED from beginning of step
```

### 2️⃣ Chunk Commit Tracking
```
Step processes 1000 items in chunks of 100:
  Chunks 1-7 committed (700 items)
  Chunk 8 FAILED

On restart: Reader resumes from item 701 (chunk 8)
  - FlatFileItemReader: tracks line number in ExecutionContext
  - JdbcPagingItemReader: tracks page number in ExecutionContext
```

### 3️⃣ ExecutionContext Persistence
```java
// Custom reader saving state
@Override
public void update(ExecutionContext ctx) {
    ctx.putInt("currentIndex", currentIndex);  // Saved to DB
}

@Override
public void open(ExecutionContext ctx) {
    currentIndex = ctx.getInt("currentIndex", 0);  // Restored on restart
}
```

### ⚠️ Not All Readers Are Restartable:
| Reader | Restartable? |
|--------|-------------|
| `FlatFileItemReader` | ✅ Yes (tracks line number) |
| `JdbcPagingItemReader` | ✅ Yes (tracks page) |
| `JdbcCursorItemReader` | ✅ Yes (tracks row count) |
| `KafkaItemReader` | ✅ Yes (tracks offset) |
| Custom reader | Only if you implement `ItemStream` |

---

<a id="q5"></a>
## Q67. ❓ How does ExecutionContext work?

🔖 **Tags:** `#spring-batch` `#execution-context` `#internals`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

ExecutionContext is a **serialized Map** stored in the database that survives JVM restarts.

```
After each chunk commit:
  1. Chunk transaction commits ✅
  2. Separate transaction: Update BATCH_STEP_EXECUTION_CONTEXT
     { "currentItemCount": 500, "currentPage": 5 }
  3. If JVM crashes now → state is safe in DB

On restart:
  1. Read BATCH_STEP_EXECUTION_CONTEXT
  2. Restore: currentItemCount=500, currentPage=5
  3. Resume reading from item 501
```

### Two Contexts:

```java
// Step ExecutionContext — scoped to one step
StepExecution stepExec = chunkContext.getStepContext().getStepExecution();
stepExec.getExecutionContext().putString("tempFile", "/tmp/data.csv");

// Job ExecutionContext — shared across all steps
stepExec.getJobExecution().getExecutionContext().putInt("totalRecords", 50000);
```

---

<a id="q6"></a>
## Q68. ❓ Where is ExecutionContext stored?

🔖 **Tags:** `#spring-batch` `#execution-context` `#storage`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Context | Stored In | Serialized As |
|---------|----------|--------------|
| **Job ExecutionContext** | `BATCH_JOB_EXECUTION_CONTEXT` table | JSON (Spring Batch 5+) or Java serialization |
| **Step ExecutionContext** | `BATCH_STEP_EXECUTION_CONTEXT` table | JSON (Spring Batch 5+) or Java serialization |

```sql
-- Example data in BATCH_STEP_EXECUTION_CONTEXT
SELECT SHORT_CONTEXT FROM BATCH_STEP_EXECUTION_CONTEXT 
WHERE STEP_EXECUTION_ID = 123;

-- Result:
-- {"@class":"java.util.HashMap","currentItemCount":500,"FlatFileItemReader.read.count":500}
```

### ⚠️ Size Limits:
- `SHORT_CONTEXT` column: VARCHAR(2500) — for small contexts
- `SERIALIZED_CONTEXT` column: TEXT/CLOB — for large contexts

> 💡 Don't store large objects in ExecutionContext! Only store state needed for restart (counters, IDs, positions).

---

<a id="q7"></a>
## Q69. ❓ How does Spring Batch resume from a failure point?

🔖 **Tags:** `#spring-batch` `#resume` `#restart` `#must-know`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

```
First Run — Processing 50,000 records, chunk size 1000:

  Chunk  1: items     1-1000  → COMMIT ✅ → EC: {count:1000}
  Chunk  2: items  1001-2000  → COMMIT ✅ → EC: {count:2000}
  ...
  Chunk 25: items 24001-25000 → COMMIT ✅ → EC: {count:25000}
  Chunk 26: items 25001-25050 → FAILED ❌ (DB connection lost)

  State in DB:
    StepExecution: status=FAILED, readCount=25050, writeCount=25000
    ExecutionContext: {"FlatFileItemReader.read.count": 25050}

───────────────────────────────────────────────

Restart — Same job parameters:

  1. Find JobInstance → exists, status=FAILED
  2. Create new JobExecution
  3. For Step "processData":
     ├── Previous status: FAILED
     ├── Load ExecutionContext: {read.count: 25050}
     └── Reader.open(executionContext):
         └── Sets current position to 25001 (last committed)
  
  Chunk 26: items 25001-26000 → COMMIT ✅ → EC: {count:26000}
  Chunk 27: items 26001-27000 → COMMIT ✅
  ...
  Chunk 50: items 49001-50000 → COMMIT ✅
  
  StepExecution: COMPLETED ✅
  JobExecution: COMPLETED ✅
```

### 📌 Key Takeaway
> 💡 Resume = Read ExecutionContext from DB → Set reader position → Continue from last committed chunk. No data is reprocessed!

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Job Execution](./06-job-execution.md) | [Next: Error Handling →](./08-error-handling.md)
]]>