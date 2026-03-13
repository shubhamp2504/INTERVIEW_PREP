# 🔴 Spring Batch — Error Handling (Q70–Q78)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q70"></a>

## Q70. What is skip logic in Spring Batch?

### 🔑 Quick Answer

> Skip logic lets Spring Batch **ignore bad records** and continue processing instead of failing the entire job. You configure which exceptions to skip and a maximum skip limit.

### 📖 Step-by-Step Explanation

**Step 1 — Without skip (default):**

```
Processing 10,000 records...
Record #4,567 has bad data → 💥 Exception
→ Chunk rolls back → Step FAILS → Job FAILS
→ 10,000 records = 0 processed. All because of 1 bad record.
```

**Step 2 — With skip:**

```
Processing 10,000 records...
Record #4,567 has bad data → 💥 Exception → SKIPPED
→ Continue with record #4,568
→ 9,999 records processed successfully!
→ 1 record skipped (logged for investigation)
```

**Step 3 — Skip works at all three phases:**

```
READ skip:
  Reader hits a bad CSV line → skip it, read next line
  Tracked in: readSkipCount

PROCESS skip:
  Processor throws ValidationException → skip this item
  Tracked in: processSkipCount

WRITE skip:
  Writer throws DataIntegrityViolation → scan mode → find and skip bad item
  Tracked in: writeSkipCount
```

**Step 4 — Configuration:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()                                  // Enable fault tolerance
            .skip(FlatFileParseException.class)               // Skip bad CSV lines
            .skip(ValidationException.class)                  // Skip invalid data
            .skip(DataIntegrityViolationException.class)      // Skip constraint violations
            .noSkip(FileNotFoundException.class)              // NEVER skip this (critical)
            .skipLimit(100)                                   // Max 100 bad records
            .build();
}
```

### 🗣️ How to Explain in Interview

> *"Skip logic allows batch jobs to handle bad records gracefully instead of failing the entire job. You enable it with faultTolerant(), specify which exceptions are safe to skip — like parse errors or validation failures — and set a maximum skip limit. If a record causes a skippable exception, Spring Batch excludes that record and continues with the rest. I always set a reasonable skip limit because if 1000 out of 10,000 records fail, that's a systemic problem, not a few bad records. And I always use a SkipListener to log every skipped record so we can fix the data."*

### ⚡ Key Points to Remember

1. `faultTolerant()` → `skip()` → `skipLimit()` — three-step configuration
2. Works at **read, process, and write** phases
3. **skipLimit** = safety net (too many errors = fail the job)
4. `noSkip()` = exceptions that should **always** fail the job
5. Always use **SkipListener** to track what was skipped

---

<a id="q71"></a>

## Q71. What is retry logic in Spring Batch?

### 🔑 Quick Answer

> Retry logic makes Spring Batch **re-attempt** a failed operation before giving up. It's for **transient errors** — database deadlocks, network timeouts, service unavailability — where the second try might succeed.

### 📖 Step-by-Step Explanation

**Step 1 — When retry makes sense:**

```
TRANSIENT errors (retry CAN help):
  - Database deadlock → retry after other transaction releases lock
  - Service timeout → retry when service recovers
  - Network glitch → retry when connection re-established
  - Optimistic lock exception → retry with latest version

PERMANENT errors (retry is USELESS):
  - Invalid data format → will fail every time
  - Constraint violation → data doesn't match schema
  - Null pointer → code bug, won't self-fix
```

**Step 2 — Configuration:**

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .retry(DeadlockLoserDataAccessException.class)    // Retry deadlocks
            .retry(ServiceUnavailableException.class)         // Retry timeouts
            .retryLimit(3)                                    // Original + 2 retries = 3 total
            .build();
}
```

**Step 3 — What happens with retry:**

```
Processing item #247:
  Attempt 1: process(item) → 💥 DeadlockLoserDataAccessException
  Attempt 2: process(item) → 💥 DeadlockLoserDataAccessException
  Attempt 3: process(item) → ✅ Success! Continue normally.

If all 3 attempts fail:
  → Check if exception is skippable → skip or fail
```

**Step 4 — Retry with backoff (wait between retries):**

```java
.faultTolerant()
.retry(ServiceTimeoutException.class)
.retryLimit(3)
.backOffPolicy(new ExponentialBackOffPolicy())  // Wait longer each retry
// Retry 1: wait 100ms
// Retry 2: wait 200ms (doubled)
// Retry 3: wait 400ms (doubled again)
```

### 🗣️ How to Explain in Interview

> *"Retry logic is for transient errors where the second or third attempt might succeed — like database deadlocks, network timeouts, or service unavailability. You configure which exceptions to retry and a retry limit. Spring Batch will re-attempt the failed item up to N times before giving up. I usually combine retry with exponential backoff — wait 100ms before first retry, 200ms before second, 400ms before third. This gives the external system time to recover. If all retries fail, the exception falls through to skip logic or fails the job."*

### ⚡ Key Points to Remember

1. **Retry** = for transient errors (deadlocks, timeouts)
2. `retry()` + `retryLimit()` — configure exception + max attempts
3. **Backoff** = wait between retries (exponential recommended)
4. If all retries fail → falls to **skip logic** (if configured)
5. Don't retry **permanent** errors (validation, constraint violations)

---

<a id="q72"></a>

## Q72. What is skip limit?

### 🔑 Quick Answer

> Skip limit is the **maximum number of records** that can be skipped before the step fails. It's a safety net — if too many records fail, something is systemically wrong and the job should stop.

### 📖 Step-by-Step Explanation

**Step 1 — Why skip limit matters:**

```
skipLimit(100):
  Record #1 fails → skip (total: 1/100)
  Record #500 fails → skip (total: 2/100)
  ...
  Record #8,000 fails → skip (total: 100/100)
  Record #8,050 fails → 💥 SKIP LIMIT REACHED → Step FAILS

This prevents situations like:
  "50% of records are failing — clearly something is WRONG
   with the data source or the processing logic."
```

**Step 2 — Guideline for setting skip limit:**

| Scenario | Recommended Limit | Reasoning |
|----------|------------------|-----------|
| Clean data expected | 0-10 | Very few errors expected |
| Some data quality issues | 50-100 | Handles known edge cases |
| External data (messy) | 100-500 | Third-party data can be unreliable |
| Huge dataset (100M+) | 1000-5000 | Proportional to data size |

**Step 3 — Skip limit applies to ALL skip types combined:**

```
skipLimit(100) means:
  readSkipCount + processSkipCount + writeSkipCount ≤ 100

NOT 100 per phase, but 100 TOTAL.
```

### 🗣️ How to Explain in Interview

> *"Skip limit is the maximum total records that can be skipped before the step fails. It's a safety net. If I'm processing 10,000 records and my skip limit is 100, the job tolerates up to 100 bad records. If the 101st bad record appears, the job fails — because that many failures suggest a systemic problem, not a few bad records. The limit applies to the total across all phases — read, process, and write skips combined. I typically set it proportional to the data size — 100 for 10K records, 1000 for 10M records."*

### ⚡ Key Points to Remember

1. **Maximum total skips** before step fails
2. Applies to **all phases combined** (read + process + write)
3. Set **proportional to data size**
4. Too high → hides real problems; too low → job fails on minor issues
5. Always monitor skip rate: `skipCount / readCount × 100` = skip percentage

---

<a id="q73"></a>

## Q73. What is retry limit?

### 🔑 Quick Answer

> Retry limit is the **maximum number of attempts** to process an item before giving up. If retryLimit=3, Spring Batch tries the original + 2 retries = 3 total attempts.

### 📖 Step-by-Step Explanation

**Step 1 — Understanding the count:**

```
retryLimit(3) means:
  Attempt 1: Original try     → FAILED
  Attempt 2: First retry      → FAILED
  Attempt 3: Second retry     → FAILED or SUCCESS

Total attempts: 3 (NOT 1 original + 3 retries)
```

**Step 2 — What happens after retries exhausted:**

```
Item #247, retryLimit=3, all attempts failed:

  If skip is configured for this exception → SKIP the item
  If skip is NOT configured → Step FAILS → Job FAILS
```

**Step 3 — Combining with backoff:**

```
retryLimit=3 with exponential backoff:

  Attempt 1: immediate               → FAILED
  wait 1 second
  Attempt 2: after 1s                → FAILED
  wait 2 seconds (doubled)
  Attempt 3: after 3s total          → FAILED
  → Give up → skip or fail

Total wall time for this item: ~3 seconds
```

### 🗣️ How to Explain in Interview

> *"Retry limit is the total number of attempts for an item. If I set retryLimit to 3, Spring Batch tries the item 3 times total. If all 3 fail, it either skips the item if skip is configured, or fails the job. I usually combine this with exponential backoff — wait 1 second, then 2, then 4 — giving the external system time to recover. The retry limit should be small — typically 3 to 5 — because if an operation fails 5 times in a row, it's unlikely to succeed on the 6th try."*

### ⚡ Key Points to Remember

1. **retryLimit(3)** = 3 total attempts (not 1 + 3)
2. After retries exhausted → **skip** (if configured) or **fail**
3. Use **exponential backoff** between retries
4. Keep limit small: **3-5** for most cases
5. Only for **transient** errors

---

<a id="q74"></a>

## Q74. What is SkipPolicy?

### 🔑 Quick Answer

> SkipPolicy is a **custom interface** that gives you full control over skip decisions — beyond simple exception type + limit. You implement `shouldSkip(Throwable, int skipCount)` and return true/false based on your own logic.

### 📖 Step-by-Step Explanation

**Step 1 — When default skip isn't enough:**

```
Default skip: "Skip ValidationException up to 100 times"
  → What if you want: "Skip validation errors always, but database errors only 10 times"
  → What if you want: "Skip errors from file A but never from file B"
  → What if you want: "Skip only if error rate < 5%"

Custom SkipPolicy solves all of these.
```

**Step 2 — The interface:**

```java
public interface SkipPolicy {
    boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException;
    // t = the exception that occurred
    // skipCount = how many items have been skipped so far
    // return true = skip this item
    // return false = fail the step
}
```

**Step 3 — Custom SkipPolicy example:**

```java
public class SmartSkipPolicy implements SkipPolicy {
    
    private static final int MAX_DATA_ERRORS = 100;
    private static final int MAX_SYSTEM_ERRORS = 5;
    
    private int dataErrorCount = 0;
    private int systemErrorCount = 0;
    
    @Override
    public boolean shouldSkip(Throwable t, long skipCount) {
        
        // Data errors (bad input) — tolerate up to 100
        if (t instanceof ValidationException || 
            t instanceof FlatFileParseException) {
            return ++dataErrorCount <= MAX_DATA_ERRORS;
        }
        
        // System errors (DB timeout) — tolerate only 5
        if (t instanceof DataAccessException) {
            return ++systemErrorCount <= MAX_SYSTEM_ERRORS;
        }
        
        // Unknown errors — never skip
        return false;
    }
}

// Usage:
.faultTolerant()
.skipPolicy(new SmartSkipPolicy())
```

### 🗣️ How to Explain in Interview

> *"SkipPolicy is a custom interface for advanced skip logic. The default skip configuration lets you specify exception types and a global skip limit, which works for most cases. But sometimes you need finer control — like allowing 100 data validation errors but only 5 database errors, because 5 database errors suggest infrastructure problems. You implement the shouldSkip method that receives the exception and current skip count, and return true to skip or false to fail. I've used this in production where different error types had different tolerance levels."*

### ⚡ Key Points to Remember

1. **Custom interface** for advanced skip decisions
2. Receives **exception** and **skipCount**
3. Returns **true** = skip, **false** = fail
4. Replaces both `skip()` and `skipLimit()` when used
5. Use when you need **different limits per exception type**

---

<a id="q75"></a>

## Q75. What is RetryPolicy?

### 🔑 Quick Answer

> RetryPolicy controls **when and how many times** to retry. The default `SimpleRetryPolicy` retries by exception type + limit, but you can create custom policies with exception maps and backoff strategies.

### 📖 Step-by-Step Explanation

**Step 1 — Built-in retry policies:**

| Policy | Behavior |
|--------|----------|
| `SimpleRetryPolicy` | Retry specific exceptions up to N times (default) |
| `TimeoutRetryPolicy` | Retry until a timeout period expires |
| `NeverRetryPolicy` | Never retry (fail immediately) |
| `AlwaysRetryPolicy` | Always retry (careful — infinite loop!) |

**Step 2 — SimpleRetryPolicy with exception map:**

```java
// Different retry limits per exception type
Map<Class<? extends Throwable>, Boolean> retryableExceptions = new HashMap<>();
retryableExceptions.put(DeadlockLoserDataAccessException.class, true);  // Retry
retryableExceptions.put(ServiceTimeoutException.class, true);           // Retry
retryableExceptions.put(NullPointerException.class, false);             // Don't retry

SimpleRetryPolicy policy = new SimpleRetryPolicy(3, retryableExceptions);
// 3 = max attempts for all retryable exceptions
```

**Step 3 — Combining with BackOffPolicy:**

```java
// Exponential backoff: wait longer each retry
ExponentialBackOffPolicy backoff = new ExponentialBackOffPolicy();
backoff.setInitialInterval(1000);   // First wait: 1 second
backoff.setMultiplier(2.0);         // Double each time
backoff.setMaxInterval(30000);      // Never wait more than 30 seconds

// Retry 1: wait 1s
// Retry 2: wait 2s
// Retry 3: wait 4s
// Retry 4: wait 8s (if needed)
```

### 🗣️ How to Explain in Interview

> *"RetryPolicy controls the retry behavior. The default SimpleRetryPolicy retries based on exception types and a limit — like retry DeadlockException up to 3 times. You can customize with an exception map that specifies which exceptions should and shouldn't be retried. The real power comes from combining it with a BackOffPolicy — exponential backoff that waits longer between each retry, giving the external system time to recover. For database deadlocks, I use 3 retries with exponential backoff starting at 1 second."*

### ⚡ Key Points to Remember

1. **SimpleRetryPolicy** = most common (exception type + limit)
2. **Exception map** = different behavior per exception type
3. **ExponentialBackOffPolicy** = best for production
4. **TimeoutRetryPolicy** = retry until time expires (not by count)
5. Retry + Backoff = **best practice** for transient errors

---

<a id="q76"></a>

## Q76. How do you handle bad records in production?

### 🔑 Quick Answer

> Use **skip + SkipListener** to log bad records, write them to an **error table or error file**, and continue processing good records. In production, always track, log, and make bad records queryable for investigation.

### 📖 Step-by-Step Explanation

**Step 1 — Production error handling strategy:**

```
Good record → Normal processing → Write to target ✅
Bad record  → Skip → Log → Write to error table → Alert team ❌

Never silently skip — always track what was skipped and why!
```

**Step 2 — Complete production setup:**

```java
// Step configuration
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("processOrders", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(ValidationException.class)
            .skip(DataIntegrityViolationException.class)
            .skipLimit(100)
            .listener(new ErrorTrackingSkipListener(errorRepository))
            .build();
}
```

**Step 3 — SkipListener that writes to error table:**

```java
@Component
public class ErrorTrackingSkipListener implements SkipListener<Order, Order> {
    
    private final JdbcTemplate jdbc;
    
    @Override
    public void onSkipInRead(Throwable t) {
        jdbc.update(
            "INSERT INTO batch_errors (phase, error_msg, timestamp) VALUES (?, ?, ?)",
            "READ", t.getMessage(), LocalDateTime.now()
        );
    }
    
    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        jdbc.update(
            "INSERT INTO batch_errors (phase, item_id, error_msg, item_data, timestamp) " +
            "VALUES (?, ?, ?, ?, ?)",
            "PROCESS", item.getOrderId(), t.getMessage(),
            item.toString(), LocalDateTime.now()
        );
    }
    
    @Override
    public void onSkipInWrite(Order item, Throwable t) {
        jdbc.update(
            "INSERT INTO batch_errors (phase, item_id, error_msg, item_data, timestamp) " +
            "VALUES (?, ?, ?, ?, ?)",
            "WRITE", item.getOrderId(), t.getMessage(),
            item.toString(), LocalDateTime.now()
        );
    }
}
```

### 🗣️ How to Explain in Interview

> *"In production, I never just skip and ignore. My approach is: configure skip for known exception types, implement a SkipListener that captures every skipped record with the phase (read/process/write), the item ID, the error message, and the full item data — and writes it to an error table. After the job completes, we can query that table to see exactly what failed and why. We also set alerts on the skip rate — if more than 1% of records are skipped, the on-call team is notified. Some teams prefer an error file instead of a table — that works too, especially if the business team needs to review and re-submit the records."*

### ⚡ Key Points to Remember

1. **Never silently skip** — always log to error table or file
2. Track **phase + itemId + errorMessage + itemData + timestamp**
3. Set **alerts on skip rate** (> 1% = investigate)
4. Make errors **queryable** (error table with indexes)
5. Give business team a way to **review and fix** bad records

---

<a id="q77"></a>

## Q77. How do you log failed records?

### 🔑 Quick Answer

> Implement **SkipListener** (for skipped records), **ItemReadListener** / **ItemWriteListener** (for all records), or write to an **error database table**. The most production-ready approach is SkipListener + error table.

### 📖 Step-by-Step Explanation

**Step 1 — Three approaches:**

| Approach | What it captures | Best for |
|----------|-----------------|----------|
| **SkipListener** | Only skipped records | Production error tracking |
| **ItemReadListener.onReadError()** | All read errors | Debugging read phase |
| **ItemWriteListener.onWriteError()** | All write errors | Debugging write phase |
| **Log files (SLF4J)** | Custom logging | Development/debugging |

**Step 2 — SkipListener (recommended for production):**

Already shown in Q76 — this is the most common and recommended approach.

**Step 3 — Adding SLF4J logging for development:**

```java
@Component
@Slf4j
public class LoggingSkipListener implements SkipListener<Order, Order> {
    
    @Override
    public void onSkipInRead(Throwable t) {
        log.error("SKIP IN READ: {}", t.getMessage());
    }
    
    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        log.error("SKIP IN PROCESS: OrderId={}, Error={}", 
                  item.getOrderId(), t.getMessage());
    }
    
    @Override
    public void onSkipInWrite(Order item, Throwable t) {
        log.error("SKIP IN WRITE: OrderId={}, Error={}", 
                  item.getOrderId(), t.getMessage());
    }
}
```

### 🗣️ How to Explain in Interview

> *"For logging failed records, I use a layered approach. SkipListener is the primary mechanism — it captures the actual item that was skipped along with the exception. In development, I log to SLF4J. In production, I write to an error database table with structured columns — item ID, phase, error message, timestamp — so the support team can query and investigate. I also set up structured logging with MDC (Mapped Diagnostic Context) to include the job ID and step name in every log entry, making it easy to correlate logs with specific job runs."*

### ⚡ Key Points to Remember

1. **SkipListener** = primary mechanism for skip tracking
2. **Error table** = production standard for queryable errors
3. **SLF4J logs** = supplementary for debugging
4. Include **item ID + phase + error + timestamp** in every log
5. Use **MDC** (jobId, stepName) for log correlation

---

<a id="q78"></a>

## Q78. How do you store rejected records for later processing?

### 🔑 Quick Answer

> Four approaches: **Error database table** (most common), **error file** (CSV/JSON), **dead letter queue** (messaging), or **staging table** with status flag. All use SkipListener to capture rejected items.

### 📖 Step-by-Step Explanation

**Step 1 — Approach comparison:**

| Approach | Best For | Retry Possible? |
|----------|---------|----------------|
| **Error DB table** | Most applications | ✅ Query and re-process |
| **Error CSV file** | Business review (Excel) | ✅ Re-import the file |
| **Dead letter queue** | Event-driven systems | ✅ Automatic retry |
| **Staging table + status** | Complex workflows | ✅ Update status and re-run |

**Step 2 — Error table approach (most common):**

```sql
CREATE TABLE batch_failed_records (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    job_execution_id BIGINT,
    step_name VARCHAR(100),
    phase VARCHAR(10),          -- READ, PROCESS, WRITE
    item_id VARCHAR(100),
    item_data TEXT,              -- Full serialized item
    error_message TEXT,
    error_class VARCHAR(255),
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- After job completes, query:
SELECT item_id, error_message, COUNT(*) 
FROM batch_failed_records 
WHERE job_execution_id = 123
GROUP BY error_message;
```

**Step 3 — Re-processing failed records:**

```
Option 1: Fix data, run a separate "cleanup" batch job that reads from error table
Option 2: Fix source data, re-run the whole job (skip already-processed records)
Option 3: Manual review by business team, update source, re-trigger
```

### 🗣️ How to Explain in Interview

> *"I store rejected records in an error database table with columns for the job execution ID, step name, phase, item ID, full item data, error message, and timestamp. After the job runs, I can query this table to see a summary of failures — which items failed, at what phase, and why. For re-processing, I either fix the source data and re-run the job — which is possible because Spring Batch only processes items it hasn't committed yet — or I run a separate cleanup job that reads from the error table, applies fixes, and writes to the target. In messaging systems, a dead letter queue is the natural choice for rejected items."*

### ⚡ Key Points to Remember

1. **Error table** = most versatile (query, analyze, re-process)
2. Include **full item data** for re-processing
3. Include **job_execution_id** for correlation
4. Build a **summary query** for quick analysis
5. Plan for **re-processing** from day one

---

> **🎯 Navigation:** [← Transactions & Restart (Q63-69)](07-transactions-restart.md) | [Next → Tasklet (Q79-83)](09-tasklet.md) | [📋 All Sections](README.md)
