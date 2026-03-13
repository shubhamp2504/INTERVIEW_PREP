
# 🔴 Spring Batch — Error Handling Questions (70-78)

[![Questions](https://img.shields.io/badge/Questions-9-blue.svg)](#)
[![Difficulty](https://img.shields.io/badge/Level-Hard-red.svg)](#)


---

<a id="q1"></a>

## Q70. ❓ What is skip logic?

🔖 **Tags:** `#spring-batch` `#skip` `#error-handling` `#must-know`  
📊 **Difficulty:** 🔴 Hard  
🔥 **Frequency:** ⭐⭐⭐⭐⭐

### ✅ Answer

**Skip logic** allows Spring Batch to **skip bad records** and continue processing instead of failing the entire job.

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()                           // Enable fault tolerance
            .skip(FlatFileParseException.class)        // Skip parse errors
            .skip(ValidationException.class)           // Skip validation errors
            .skipLimit(100)                            // Max 100 skips total
            .noSkip(DatabaseException.class)           // NEVER skip DB errors
            .build();
}
```

### Skip Behavior Per Phase:

| Phase | On Skip |
|-------|---------|
| **Reader** | Bad item skipped, next item read. ReadSkipCount++ |
| **Processor** | Item excluded from chunk. ProcessSkipCount++ |
| **Writer** | Chunk re-processed one-by-one (scan mode). BadItem skipped. WriteSkipCount++ |

---

<a id="q2"></a>

## Q71. ❓ What is retry logic?

🔖 **Tags:** `#spring-batch` `#retry` `#error-handling`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

**Retry logic** re-attempts a failed operation before giving up. Used for **transient errors** (network timeout, deadlock).

```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .retry(DeadlockLoserDataAccessException.class)  // Retry on deadlock
            .retry(TransientDataAccessException.class)      // Retry on transient
            .retryLimit(3)                                  // Max 3 retries
            .build();
}
```

```
Item processing:
  Attempt 1 → DeadlockException ❌ (retry)
  Attempt 2 → DeadlockException ❌ (retry)
  Attempt 3 → SUCCESS ✅

If all 3 retries fail → check skip logic or fail step
```

### Skip + Retry Together:
```java
.faultTolerant()
.retry(TransientException.class).retryLimit(3)     // Retry 3 times
.skip(TransientException.class).skipLimit(10)      // If retry fails, skip (up to 10)
// Flow: Try → Retry 3x → If still fails → Skip
```

---

<a id="q3"></a>

## Q72. ❓ What is skip limit?

🔖 **Tags:** `#spring-batch` `#skip-limit`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**skipLimit** = maximum number of items that can be skipped before the step fails.

```java
.skipLimit(100)  // After 100 skips, step FAILS

// Tracking:
// skipCount = readSkipCount + processSkipCount + writeSkipCount
// If skipCount > skipLimit → StepFailedException
```

| skipLimit | Behavior |
|-----------|----------|
| `0` | No skipping (default) |
| `100` | Skip up to 100 bad records |
| `Integer.MAX_VALUE` | Skip unlimited (not recommended!) |

### 📌 Best Practice:
> 💡 Always set a reasonable skipLimit. If too many records fail, something is systemically wrong — you want to know about it.

---

<a id="q4"></a>

## Q73. ❓ What is retry limit?

🔖 **Tags:** `#spring-batch` `#retry-limit`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

**retryLimit** = how many times to retry a failed item before giving up.

```java
.retryLimit(3)  // Try original + 2 retries = 3 total attempts
```

| retryLimit | Attempts |
|-----------|---------|
| `1` | Original only (no retry) |
| `3` | Original + 2 retries |
| `5` | Original + 4 retries |

### With Backoff:
```java
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader()).writer(writer())
            .faultTolerant()
            .retry(TransientException.class)
            .retryLimit(3)
            .backOffPolicy(new ExponentialBackOffPolicy())  // Wait between retries
            .build();
}
```

---

<a id="q5"></a>

## Q74. ❓ What is SkipPolicy?

🔖 **Tags:** `#spring-batch` `#skip-policy` `#custom`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

`SkipPolicy` allows **custom skip decisions** beyond simple exception type + limit.

```java
public interface SkipPolicy {
    boolean shouldSkip(Throwable t, long skipCount) throws SkipLimitExceededException;
}

// Custom: Skip validation errors but fail on system errors
public class CustomSkipPolicy implements SkipPolicy {
    @Override
    public boolean shouldSkip(Throwable t, long skipCount) {
        if (t instanceof ValidationException) {
            return skipCount < 200;  // Skip up to 200 validation errors
        }
        if (t instanceof FileParseException) {
            return skipCount < 50;   // Skip up to 50 parse errors
        }
        return false;  // Don't skip anything else → fail
    }
}

// Usage:
.faultTolerant()
.skipPolicy(new CustomSkipPolicy())
```

---

<a id="q6"></a>

## Q75. ❓ What is RetryPolicy?

🔖 **Tags:** `#spring-batch` `#retry-policy`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

`RetryPolicy` controls **when and how many times** to retry.

```java
// Simple: Fixed retry count
SimpleRetryPolicy policy = new SimpleRetryPolicy();
policy.setMaxAttempts(3);

// Exception-specific:
Map<Class<? extends Throwable>, Boolean> retryable = new HashMap<>();
retryable.put(DeadlockException.class, true);     // Retry this
retryable.put(TimeoutException.class, true);       // Retry this
retryable.put(ValidationException.class, false);   // Don't retry
SimpleRetryPolicy policy = new SimpleRetryPolicy(3, retryable);

// With backoff:
RetryTemplate retryTemplate = RetryTemplate.builder()
    .maxAttempts(3)
    .exponentialBackoff(1000, 2.0, 10000)  // 1s, 2s, 4s...max 10s
    .retryOn(TransientException.class)
    .build();
```

---

<a id="q7"></a>

## Q76. ❓ How do you handle bad records in a batch job?

🔖 **Tags:** `#spring-batch` `#bad-records` `#production`  
📊 **Difficulty:** 🔴 Hard

### ✅ Answer

| Strategy | How |
|----------|-----|
| **Skip + Log** | Skip bad records, log to file/DB |
| **Dead Letter Queue** | Put failed items in separate queue/table |
| **Error File** | Write bad records to an error file |
| **RetryTemplate** | Retry transient errors with backoff |
| **Custom SkipListener** | Custom handling on skip event |

```java
// SkipListener to log/store bad records
public class BadRecordListener implements SkipListener<Employee, Employee> {
    
    @Override
    public void onSkipInRead(Throwable t) {
        log.error("Skip in read: {}", t.getMessage());
    }
    
    @Override
    public void onSkipInProcess(Employee item, Throwable t) {
        log.error("Skip in process: {} - {}", item.getId(), t.getMessage());
        badRecordRepository.save(item, t.getMessage());  // Store for review
    }
    
    @Override
    public void onSkipInWrite(Employee item, Throwable t) {
        log.error("Skip in write: {} - {}", item.getId(), t.getMessage());
        errorFileWriter.write(item);  // Write to error file
    }
}
```

---

<a id="q8"></a>

## Q77. ❓ How do you log failed records?

🔖 **Tags:** `#spring-batch` `#logging` `#error-handling`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

```java
// Method 1: SkipListener (shown above in Q76)

// Method 2: ItemReadListener + ItemWriteListener
public class ErrorLoggingListener implements ItemReadListener<Employee>, 
                                             ItemWriteListener<Employee> {
    @Override
    public void onReadError(Exception ex) {
        log.error("Read error: {}", ex.getMessage());
    }
    
    @Override
    public void onWriteError(Exception ex, Chunk<? extends Employee> items) {
        log.error("Write error for {} items: {}", items.size(), ex.getMessage());
        items.forEach(item -> log.error("  Failed item: {}", item));
    }
}

// Method 3: Write to error table
@Bean
public JdbcBatchItemWriter<FailedRecord> errorWriter(DataSource ds) {
    return new JdbcBatchItemWriterBuilder<FailedRecord>()
            .sql("INSERT INTO failed_records (item_data, error_msg, timestamp) VALUES (?,?,?)")
            .dataSource(ds)
            .build();
}
```

---

<a id="q9"></a>

## Q78. ❓ How do you store rejected records?

🔖 **Tags:** `#spring-batch` `#rejected-records`  
📊 **Difficulty:** 🟡 Medium

### ✅ Answer

| Approach | Storage | Best For |
|----------|---------|----------|
| **Error DB table** | `failed_records` table | Production, queryable |
| **Error file** | `rejected-YYYY-MM-DD.csv` | File-based jobs |
| **Dead letter queue** | Kafka DLQ / JMS DLQ | Event-driven systems |
| **Log file** | Application logs | Simple cases |

```java
// Complete setup: Error table + SkipListener
@Bean
public Step step(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("step", repo)
            .<I, O>chunk(100, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(Exception.class).skipLimit(500)
            .listener(new SkipListener<I, O>() {
                @Autowired JdbcTemplate jdbc;
                
                @Override
                public void onSkipInProcess(I item, Throwable t) {
                    jdbc.update(
                        "INSERT INTO rejected_records(data, error, created_at) VALUES(?,?,?)",
                        item.toString(), t.getMessage(), LocalDateTime.now()
                    );
                }
            })
            .build();
}
```

---

[← Back to Spring Batch Index](./README.md) | [← Prev: Transactions](./07-transactions-restart.md) | [Next: Tasklet →](./09-tasklet.md)
]]>
