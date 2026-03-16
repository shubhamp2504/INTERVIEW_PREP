# ⚠️ Error Handling — Q70 to Q78

---

## Q70. What is skip logic in Spring Batch?

### 📝 One-Liner
Skip logic lets you ignore bad records and continue processing — the skipped item is excluded and tracked, without failing the job.

### 🔑 Quick Answer
Skip logic tells Spring Batch: "if THIS type of exception occurs AND we haven't exceeded the skip limit, skip that item and continue." Three-step config: `.faultTolerant()` → `.skip(ExceptionClass)` → `.skipLimit(N)`. Works at all three phases: read (bad line), process (validation fail), write (constraint violation). Use `noSkip()` for exceptions that should ALWAYS fail. Always pair with `SkipListener` to log what was skipped. *(Kharab record ko chhod ke aage badho — lekin log zaroor karo)*

### 📖 How It Works
```
Skip Logic at Each Phase:

READ skip:
  reader.read() → FlatFileParseException → SKIP → next line
  (reader knows exactly which line failed)

PROCESS skip:
  processor.process(item) → ValidationException → SKIP → next item
  (processor knows exactly which item failed)

WRITE skip (SCAN MODE):
  writer.write([A,B,C,D,E]) → ConstraintViolationException
  → ROLLBACK entire chunk
  → Re-write one-by-one: A✅, B✅, C❌skip, D✅, E✅
  (writer received list, must find bad item individually)

Skip tracking:
  readSkipCount + processSkipCount + writeSkipCount = total skipCount
  skipLimit applies to total (not per-phase)
```

### 🗣️ How to Say in Interview
"Skip logic allows Spring Batch to ignore bad records and continue processing. I configure it with faultTolerant(), specify which exception types are skippable, and set a skipLimit as a safety net. It works differently at each phase — reads and processing skip immediately since the framework knows which item failed. Write skips trigger scan mode since the writer receives the entire chunk as a list. In my project, we skipped up to 100 validation exceptions for bad CSV records, always logging them via SkipListener to an error table for the ops team to review. We never exceeded 0.1% skip rate on normal data."

### 💻 Code
```java
@Bean
public Step skipStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("skipStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(csvReader())
            .processor(validator())
            .writer(dbWriter())
            .faultTolerant()
            // These exceptions can be skipped
            .skip(FlatFileParseException.class)        // bad CSV line
            .skip(ValidationException.class)            // business rule failure
            .skip(DuplicateKeyException.class)          // duplicate in DB
            .skipLimit(100)                              // max 100 skips total
            // These must ALWAYS fail the job
            .noSkip(DatabaseConnectionException.class)
            .noSkip(FileNotFoundException.class)
            // Log every skip
            .listener(skipLogger())
            .build();
}

@Bean
public SkipListener<Order, ProcessedOrder> skipLogger() {
    return new SkipListener<>() {
        @Override
        public void onSkipInRead(Throwable t) {
            log.error("SKIP in read: {}", t.getMessage());
        }
        @Override
        public void onSkipInProcess(Order item, Throwable t) {
            log.error("SKIP in process: id={}, error={}", item.getId(), t.getMessage());
        }
        @Override
        public void onSkipInWrite(ProcessedOrder item, Throwable t) {
            log.error("SKIP in write: id={}, error={}", item.getId(), t.getMessage());
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- Without `.faultTolerant()`, skip config is ignored *(faultTolerant lagana zaruri hai — bina uske skip kaam nahi karega)*
- skipLimit is TOTAL across all phases, not per phase
- Write skips trigger scan mode → slow (re-writes individually)
- `noSkip()` overrides `skip()` for specific exceptions
- `skipLimit(-1)` = unlimited skips (dangerous — can silently skip all data)

### 🎯 Tricky Interview Qs

**Q: What happens when skipLimit is reached?**
The step fails immediately with `SkipLimitExceededException`. This is a safety net — too many skips usually indicate a systemic problem, not individual bad records.

**Q: Does skip affect restart?**
Skipped items are recorded. On restart, previously skipped items are NOT re-processed.

### ⚡ Remember
- `.faultTolerant().skip(Exception).skipLimit(N)` *(teen step: faultTolerant, skip, skipLimit)*
- Works at read, process, and write phases
- Write skip → scan mode (slow)
- Always add SkipListener for audit
- noSkip() for fatal exceptions

### 🔗 Follow-ups
- [Q71 → Retry logic](#q71)
- [Q72 → Skip limit details](#q72)
- [Q74 → Custom SkipPolicy](#q74)

---

## Q71. What is retry logic in Spring Batch?

### 📝 One-Liner
Retry logic re-attempts failed operations for transient errors like deadlocks and timeouts — use for errors that might succeed on the next try.

### 🔑 Quick Answer
Retry is for TRANSIENT errors — errors that might succeed if you try again (database deadlocks, network timeouts, service temporarily unavailable). Configure with `.retry(Exception.class).retryLimit(N)`. DON'T retry permanent errors (validation failures, missing data). Combine with backoff policy to wait between retries. If all retries fail, the exception either triggers skip (if configured) or fails the step. *(Retry sirf transient errors ke liye — jo dobara try karne pe sahi ho sake)*

### 📖 How It Works
```
Retry Flow:

processor.process(item) → TimeoutException
  Attempt 1: FAILED → wait → retry
  Attempt 2: FAILED → wait → retry  
  Attempt 3: SUCCESS ✅ → continue processing

processor.process(item) → TimeoutException
  Attempt 1: FAILED → wait → retry
  Attempt 2: FAILED → wait → retry
  Attempt 3: FAILED → retryLimit exhausted
    → skip configured? → skip item
    → no skip? → STEP FAILED

Retry vs Skip:
  Retry: same operation tried again (transient errors)
  Skip: operation abandoned, move to next item (data errors)
  Best: retry first, then skip if retries exhausted
```

### 🗣️ How to Say in Interview
"Retry logic is for transient errors — situations where the same operation might succeed on the next attempt, like database deadlocks, network timeouts, or temporary service unavailability. I configure it with the exception types that are retryable and a retry limit. I always combine retry with skip — retry first for transient errors, then skip if all retries are exhausted. In my project, we retried 3 times for database deadlocks with exponential backoff, and if it still failed, the item was skipped and logged. Never retry permanent errors like validation failures — that wastes time."

### 💻 Code
```java
@Bean
public Step retryStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("retryStep", repo)
            .<Order, ProcessedOrder>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            // Retry transient errors
            .retry(DeadlockLoserDataAccessException.class)
            .retry(TransientDataAccessException.class)
            .retry(SocketTimeoutException.class)
            .retryLimit(3)
            // After retries exhausted → skip
            .skip(DeadlockLoserDataAccessException.class)
            .skip(SocketTimeoutException.class)
            .skipLimit(50)
            // Never retry permanent errors
            // (ValidationException NOT in retry list → fails/skips immediately)
            .skip(ValidationException.class)
            .listener(skipLogger())
            .build();
}

// With backoff policy (wait between retries)
@Bean
public Step retryWithBackoff(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("retryWithBackoff", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .faultTolerant()
            .retry(TransientDataAccessException.class)
            .retryLimit(3)
            .backOffPolicy(new ExponentialBackOffPolicy() {{
                setInitialInterval(1000);   // 1 sec
                setMultiplier(2.0);          // 1s → 2s → 4s
                setMaxInterval(10000);       // max 10 sec
            }})
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Don't retry permanent errors — wastes time and resources *(permanent error retry karne ka koi fayda nahi)*
- retryLimit(3) means 3 TOTAL attempts (initial + 2 retries in Spring Retry, or 3 total in Spring Batch)
- Without backoff, retries happen immediately — can worsen the problem (thundering herd)
- Retry applies per ITEM, not per chunk
- Processor must be idempotent for retry safety

### 🆚 vs. Comparison
| Aspect | Retry | Skip |
|--------|-------|------|
| For | Transient errors | Data/permanent errors |
| Action | Try again | Abandon item |
| Example | Deadlock, timeout | Validation failure |
| Tracked as | (not separately tracked) | skipCount |
| Best with | Backoff policy | SkipListener |

### ⚡ Remember
- Retry = transient errors (might succeed next time) *(retry = shayad agla try kaam kare)*
- Don't retry permanent errors (validation, missing data)
- Combine retry + skip + backoff for production
- retryLimit = total attempts per item
- Always add backoff between retries

### 🔗 Follow-ups
- [Q70 → Skip logic](#q70)
- [Q73 → Retry limit details](#q73)
- [Q75 → Custom RetryPolicy](#q75)

---

## Q72. What is skip limit?

### 📝 One-Liner
Skip limit is the maximum total number of records that can be skipped before the step fails — it's a safety net against systemic data problems.

### 🔑 Quick Answer
`skipLimit(N)` sets the maximum number of items that can be skipped across ALL phases (read + process + write) in a step. Once exceeded, the step fails with `SkipLimitExceededException`. It's a safety net: a few bad records (5 out of 100K) = normal; thousands of bad records = systemic problem that should stop the job. Set proportional to data size. *(Kitni records skip ho sakti hain uski limit — zyada ho jaaye toh matlab kuch galat hai)*

### 📖 How It Works
```
Skip Limit Tracking:

skipLimit(100):
  readSkipCount: 20  ← bad CSV lines
  processSkipCount: 15 ← validation failures
  writeSkipCount: 5   ← constraint violations
  TOTAL: 40           ← under limit (100), continue ✅

  ... later ...
  TOTAL: 100          ← reached limit!
  Next skip attempt → SkipLimitExceededException → STEP FAILED ❌

Setting Guidelines:
| Data Size | Typical skipLimit | Reasoning |
|-----------|------------------|-----------|
| < 1K      | 5-10             | Few records, any skip is significant |
| 1K-100K   | 50-100           | Normal tolerance for data quality |
| 100K-1M   | 200-500          | Allow reasonable noise |
| > 1M      | 500-1000         | Large datasets have more noise |
```

### 🗣️ How to Say in Interview
"Skip limit is the safety net that prevents Spring Batch from silently skipping too many records, which would indicate a systemic data quality problem rather than individual bad records. I set it proportional to the data volume — typically 0.1% of expected records. In my project processing 500K daily payment records, we set skipLimit to 500. If we ever hit that limit, it meant the input file was fundamentally corrupt and needed investigation, not processing. We also monitored the skip rate after each run — a sudden increase in skip count triggered an alert."

### 💻 Code
```java
@Bean
public Step limitedSkipStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("limitedSkipStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .skip(ValidationException.class)
            .skip(FlatFileParseException.class)
            .skipLimit(100)   // max 100 total skips across all phases
            .listener(new StepExecutionListener() {
                @Override
                public ExitStatus afterStep(StepExecution se) {
                    double skipRate = (double) se.getSkipCount() / se.getReadCount() * 100;
                    if (skipRate > 1.0) {
                        log.warn("HIGH SKIP RATE: {:.1f}% — investigate data quality!", skipRate);
                    }
                    return se.getExitStatus();
                }
            })
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- skipLimit is TOTAL (read + process + write), not per phase *(total count hai — sab phases ka combined)*
- `skipLimit(-1)` = unlimited skips (dangerous, can skip ALL records)
- `skipLimit(0)` = no skips allowed (same as no fault tolerance)
- Exceeding limit throws `SkipLimitExceededException` and fails the step
- Monitor skip rate, not just absolute count

### ⚡ Remember
- Safety net: too many skips = systemic problem *(limit se zyada skip = kuch galat hai)*
- Total across all phases (read + process + write)
- Set proportional to data size (~0.1%)
- `skipLimit(-1)` = unlimited (avoid in production)
- Monitor skip rate after each run

### 🔗 Follow-ups
- [Q70 → Skip logic basics](#q70)
- [Q73 → Retry limit](#q73)
- [Q74 → Custom SkipPolicy for per-exception limits](#q74)

---

## Q73. What is retry limit?

### 📝 One-Liner
Retry limit is the maximum number of total attempts per item — `retryLimit(3)` means try at most 3 times before giving up.

### 🔑 Quick Answer
`retryLimit(3)` means Spring Batch will attempt the operation up to 3 times total for each failing item. After exhaustion: if skip is configured for that exception → item is skipped; otherwise → step fails. Keep retry limits small (3-5) because retries are expensive. Combine with exponential backoff to avoid hammering the failing service. *(3 baar try karega — phir bhi fail toh skip ya fail)*

### 📖 How It Works
```
retryLimit(3) behavior:

Attempt 1: processor.process(item) → TimeoutException → retry
Attempt 2: processor.process(item) → TimeoutException → retry
Attempt 3: processor.process(item) → TimeoutException → EXHAUSTED
  → skip configured? → skip item + continue
  → no skip? → STEP FAILED

With backoff:
  Attempt 1 → fail → wait 1s
  Attempt 2 → fail → wait 2s
  Attempt 3 → fail → wait 4s → EXHAUSTED
```

### 🗣️ How to Say in Interview
"Retry limit sets the maximum number of attempts for each failing item. I keep it at 3 for most transient errors — database deadlocks often resolve in 1-2 retries, and network timeouts in 2-3. I always combine it with exponential backoff to avoid hammering the service and with skip for cases where all retries are exhausted. In my project, we saw that 95% of deadlocks resolved within 2 retries, so retryLimit of 3 was sufficient. For external API timeouts, we used retryLimit of 3 with initial backoff of 2 seconds."

### 💻 Code
```java
@Bean
public Step retryLimitStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("retryLimitStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(apiProcessor())
            .writer(writer())
            .faultTolerant()
            .retry(DeadlockLoserDataAccessException.class)
            .retry(SocketTimeoutException.class)
            .retryLimit(3)    // 3 total attempts per item
            // If all retries fail → skip
            .skip(DeadlockLoserDataAccessException.class)
            .skip(SocketTimeoutException.class)
            .skipLimit(50)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- retryLimit applies per ITEM, not per chunk *(har item ke liye alag count hai)*
- High retryLimit without backoff → rapid-fire retries → worsen the problem
- Retrying in WRITE triggers chunk rollback + re-processing (expensive)
- Don't retry permanent errors — wastes time multiplied by retryLimit

### ⚡ Remember
- retryLimit(3) = 3 total attempts per item
- Keep small: 3-5 for most cases *(3-5 attempts kaafi hain)*
- Always add backoff between retries
- After exhaustion → skip or fail
- Per item, not per chunk

### 🔗 Follow-ups
- [Q71 → Retry logic basics](#q71)
- [Q75 → RetryPolicy for advanced control](#q75)
- [Q72 → Skip limit comparison](#q72)

---

## Q74. What is SkipPolicy?

### 📝 One-Liner
SkipPolicy is a custom interface for advanced skip decisions — different limits per exception type or conditional logic beyond simple type + count.

### 🔑 Quick Answer
`SkipPolicy` is the interface behind skip logic: it has one method `shouldSkip(Throwable, int skipCount)` that returns true (skip) or false (fail). The default `LimitCheckingItemSkipPolicy` implements the standard type + limit behavior. Create a custom SkipPolicy when you need: different limits per exception type, conditional skip based on item data, or custom logging before deciding. A custom SkipPolicy replaces BOTH `.skip()` and `.skipLimit()` config. *(Standard skip se zyada control chahiye toh custom SkipPolicy banao)*

### 📖 How It Works
```
SkipPolicy Interface:

boolean shouldSkip(Throwable exception, int skipCount)
  ├── return true  → skip the item, continue
  └── return false → fail the step

Default: LimitCheckingItemSkipPolicy
  → checks exception type in skip list
  → checks skipCount < skipLimit

Custom: different limits per exception
  → ValidationException: up to 100
  → ParseException: up to 50
  → DataIntegrityViolation: up to 10
```

### 🗣️ How to Say in Interview
"SkipPolicy is the interface behind Spring Batch's skip mechanism. The default implementation checks exception type and skip count. I create a custom SkipPolicy when I need different skip limits per exception type — for example, allowing 100 validation errors but only 10 constraint violations, since constraint violations are more likely to indicate a systemic issue. This gives finer control than the basic skip() configuration which shares a single limit across all exception types."

### 💻 Code
```java
// Custom SkipPolicy with per-exception limits
public class GranularSkipPolicy implements SkipPolicy {
    
    @Override
    public boolean shouldSkip(Throwable t, int skipCount) {
        if (t instanceof ValidationException) {
            return skipCount < 100;   // allow up to 100 validation errors
        }
        if (t instanceof FlatFileParseException) {
            return skipCount < 50;    // allow up to 50 parse errors
        }
        if (t instanceof DuplicateKeyException) {
            return skipCount < 10;    // only 10 duplicates tolerated
        }
        // All other exceptions → don't skip, fail
        return false;
    }
}

// Register custom policy
@Bean
public Step customSkipStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("customSkipStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .writer(writer())
            .faultTolerant()
            .skipPolicy(new GranularSkipPolicy())  // replaces skip() + skipLimit()
            .listener(skipLogger())
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Custom SkipPolicy replaces `.skip()` and `.skipLimit()` — don't mix them *(custom policy use karo toh skip() aur skipLimit() mat lagao)*
- `skipCount` parameter is the TOTAL skips so far (not per exception type) — track per type yourself
- Must return false for fatal exceptions (don't skip everything)
- Test thoroughly — wrong policy can silently skip all data

### ⚡ Remember
- Interface: `shouldSkip(Throwable, skipCount)` → true/false
- Use for: per-exception limits, conditional logic *(alag alag exception ke liye alag limit)*
- Replaces both skip() and skipLimit()
- Default: LimitCheckingItemSkipPolicy
- Test thoroughly — data safety depends on it

### 🔗 Follow-ups
- [Q70 → Basic skip logic](#q70)
- [Q75 → RetryPolicy (similar pattern)](#q75)
- [Q76 → Production bad record handling](#q76)

---

## Q75. What is RetryPolicy?

### 📝 One-Liner
RetryPolicy controls when and how many times to retry — SimpleRetryPolicy for basic count-based retry, with backoff policies for wait intervals.

### 🔑 Quick Answer
`RetryPolicy` determines if an operation should be retried. **SimpleRetryPolicy** (most common) retries based on exception type and max count. **ExponentialBackOffPolicy** adds increasing wait time between retries (1s → 2s → 4s). You can create a Map of exception → limit for per-exception retry counts. For production, always combine RetryPolicy with BackOffPolicy. *(SimpleRetryPolicy sabse common hai — exception type + count check karta hai)*

### 📖 How It Works
```
RetryPolicy + BackOffPolicy:

SimpleRetryPolicy:
  maxAttempts: 3
  retryableExceptions: {TimeoutException: true, DeadlockException: true}

ExponentialBackOffPolicy:
  initialInterval: 1000ms
  multiplier: 2.0
  maxInterval: 10000ms

Combined behavior:
  Attempt 1 → TimeoutException → wait 1s
  Attempt 2 → TimeoutException → wait 2s
  Attempt 3 → TimeoutException → EXHAUSTED (3 attempts done)
```

### 🗣️ How to Say in Interview
"RetryPolicy controls retry behavior in Spring Batch. SimpleRetryPolicy is the most common — it specifies retryable exception types and maximum attempt count. I always pair it with ExponentialBackOffPolicy in production to avoid rapid-fire retries that can worsen the problem. In my project, we had a per-exception retry map — 3 retries for deadlocks with 1-second backoff, and 5 retries for external API timeouts with 2-second initial backoff and exponential increase. This gave us fine-grained control based on the error type."

### 💻 Code
```java
// SimpleRetryPolicy with per-exception limits
@Bean
public Step advancedRetryStep(JobRepository repo, PlatformTransactionManager tx) {
    // Different retry limits per exception
    Map<Class<? extends Throwable>, Boolean> retryableExceptions = new HashMap<>();
    retryableExceptions.put(DeadlockLoserDataAccessException.class, true);  // retry
    retryableExceptions.put(SocketTimeoutException.class, true);             // retry
    retryableExceptions.put(ValidationException.class, false);               // DON'T retry

    SimpleRetryPolicy retryPolicy = new SimpleRetryPolicy(3, retryableExceptions);

    // Exponential backoff
    ExponentialBackOffPolicy backoff = new ExponentialBackOffPolicy();
    backoff.setInitialInterval(1000);   // 1 second
    backoff.setMultiplier(2.0);          // double each time
    backoff.setMaxInterval(10000);       // cap at 10 seconds

    return new StepBuilder("advancedRetryStep", repo)
            .<Order, Order>chunk(500, tx)
            .reader(reader())
            .processor(processor())
            .writer(writer())
            .faultTolerant()
            .retryPolicy(retryPolicy)
            .backOffPolicy(backoff)
            .skip(SocketTimeoutException.class)  // skip after retries exhausted
            .skipLimit(50)
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Without backoff, retries are immediate → can worsen congestion *(bina backoff ke turant retry = problem aur badh sakti hai)*
- SimpleRetryPolicy's maxAttempts includes the initial attempt
- Don't mix `.retryPolicy()` with `.retry().retryLimit()` — use one or the other
- Exponential backoff can cause long delays — set maxInterval to cap wait time

### ⚡ Remember
- SimpleRetryPolicy = exception type + count *(sabse common — type + count)*
- ExponentialBackOffPolicy = increasing wait (1s → 2s → 4s)
- Don't retry without backoff in production
- Per-exception map for fine-grained control
- maxInterval caps the wait time

### 🔗 Follow-ups
- [Q71 → Retry logic basics](#q71)
- [Q73 → Retry limit](#q73)
- [Q74 → SkipPolicy (similar custom pattern)](#q74)

---

## Q76. How do you handle bad records in production?

### 📝 One-Liner
Skip + SkipListener to log bad records to an error table or file — never silently skip; always track, alert, and make errors queryable.

### 🔑 Quick Answer
Production bad record handling strategy: **(1) Skip** the bad record (don't fail the whole job for one bad item). **(2) Log** every skip via SkipListener — write to a dedicated error table or error file. **(3) Alert** the operations team if skip rate exceeds threshold. **(4) Make errors queryable** — store in a table with timestamp, error details, original data for investigation. **(5) Reprocess** — after fixing root cause, reprocess skipped records from error table. Never silently skip. *(Skip karo, log karo, alert karo, queryable banao — kabhi chup-chaap skip mat karo)*

### 📖 How It Works
```
Production Error Handling Pipeline:

Bad Record → Skip (don't fail job)
    ↓
SkipListener → Write to ERROR_RECORDS table
    ↓
Post-Step Listener → Check skip rate
    ↓
skip rate > 1%? → ALERT ops team
    ↓
Ops team → Query ERROR_RECORDS → Fix root cause
    ↓
Reprocess → Run separate job on error records

ERROR_RECORDS table:
| ID | JOB_EXEC_ID | STEP | PHASE | ITEM_DATA | ERROR_MSG | TIMESTAMP |
| 1  | 456         | processStep | WRITE | {order:123} | Duplicate | 2024-01-15 |
```

### 🗣️ How to Say in Interview
"In production, I follow a five-step approach for bad records: skip, log, alert, query, reprocess. First, configure skip for expected error types so one bad record doesn't fail the entire job. Second, use SkipListener to write every skipped record to a dedicated ERROR_RECORDS table with the original item data, error message, phase, and timestamp. Third, monitor skip rate after each step — if it exceeds 1%, trigger an alert. Fourth, the ops team queries the error table to investigate. Fifth, after fixing the root cause, we run a separate reprocessing job on the error records. In my project, this approach reduced our incident response time from hours to minutes because errors were immediately visible and queryable."

### 💻 Code
```java
@Component
public class ProductionSkipListener implements SkipListener<Order, ProcessedOrder> {

    @Autowired private ErrorRecordRepository errorRepo;
    @Autowired private AlertService alertService;

    @Override
    public void onSkipInRead(Throwable t) {
        errorRepo.save(ErrorRecord.builder()
                .phase("READ")
                .errorMessage(t.getMessage())
                .errorType(t.getClass().getSimpleName())
                .timestamp(LocalDateTime.now())
                .build());
    }

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        errorRepo.save(ErrorRecord.builder()
                .phase("PROCESS")
                .itemData(toJson(item))
                .errorMessage(t.getMessage())
                .errorType(t.getClass().getSimpleName())
                .timestamp(LocalDateTime.now())
                .build());
    }

    @Override
    public void onSkipInWrite(ProcessedOrder item, Throwable t) {
        errorRepo.save(ErrorRecord.builder()
                .phase("WRITE")
                .itemData(toJson(item))
                .errorMessage(t.getMessage())
                .errorType(t.getClass().getSimpleName())
                .timestamp(LocalDateTime.now())
                .build());
    }
}

// Post-step alert
@Bean
public StepExecutionListener skipRateMonitor() {
    return new StepExecutionListener() {
        @Override
        public ExitStatus afterStep(StepExecution se) {
            if (se.getReadCount() > 0) {
                double skipRate = (double) se.getSkipCount() / se.getReadCount() * 100;
                if (skipRate > 1.0) {
                    alertService.sendAlert("HIGH SKIP RATE: " + skipRate + "% in " + 
                        se.getStepName());
                }
            }
            return se.getExitStatus();
        }
    };
}
```

### ⚠️ Pitfalls / Gotchas
- NEVER skip silently — always log and alert *(chup-chaap skip karna = data loss — kabhi mat karo)*
- Error table inserts should be in separate transaction (not the chunk transaction)
- Large error messages → truncate to fit DB column
- Reprocessing job should be idempotent (safe to run multiple times)

### ⚡ Remember
- Five steps: skip → log → alert → query → reprocess
- SkipListener writes to ERROR_RECORDS table *(har skip ka record rakho)*
- Monitor skip rate (> 1% = alert)
- Error table: queryable by phase, error type, timestamp
- Never skip silently in production

### 🔗 Follow-ups
- [Q77 → Logging failed records](#q77)
- [Q78 → Storing rejected records](#q78)
- [Q70 → Skip logic configuration](#q70)

---

## Q77. How do you log failed records?

### 📝 One-Liner
Use SkipListener to capture skipped items at each phase, then log to file, database error table, or monitoring system.

### 🔑 Quick Answer
Three logging destinations: **(1) Log file** — use SLF4J logger in SkipListener for immediate visibility. **(2) Error database table** — store structured error data for querying and reporting. **(3) Error CSV file** — write a parallel error file alongside main output for easy review. For production, use both log file (for real-time monitoring) AND error table (for querying and reprocessing). *(SkipListener se log file, error table, ya error CSV mein likho)*

### 📖 How It Works
```
SkipListener → Three Output Channels:

onSkipInRead(Throwable t):
  ├── log.error("Skip in read: {}", t.getMessage())    → log file
  ├── errorRepo.save(new ErrorRecord(...))              → error table
  └── errorWriter.write(rawLine + "," + t.getMessage()) → error CSV

onSkipInProcess(Order item, Throwable t):
  ├── log.error("Skip in process: id={}", item.getId()) → log file
  ├── errorRepo.save(new ErrorRecord(item, t))          → error table
  └── errorWriter.write(item.toCsv() + "," + error)    → error CSV

onSkipInWrite(ProcessedOrder item, Throwable t):
  ├── log.error("Skip in write: id={}", item.getId())  → log file
  ├── errorRepo.save(new ErrorRecord(item, t))          → error table
  └── errorWriter.write(item.toCsv() + "," + error)    → error CSV
```

### 🗣️ How to Say in Interview
"I use SkipListener to log failed records through multiple channels. For real-time visibility, I log to the application log file using SLF4J. For querying and analysis, I write to a dedicated error table with item data, error message, and timestamp. In my project, we also generated an error CSV file alongside the main output — our operations team could open it in Excel for quick review. The error table was the primary source for our reprocessing pipeline, while the log file triggered real-time alerts through our log aggregation system."

### 💻 Code
```java
@Component
public class MultiChannelSkipListener implements SkipListener<Order, ProcessedOrder> {

    private static final Logger log = LoggerFactory.getLogger(MultiChannelSkipListener.class);
    
    @Autowired private ErrorRecordRepository errorRepo;
    private FlatFileItemWriter<String> errorFileWriter;

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        // Channel 1: Log file (real-time monitoring)
        log.error("SKIP [PROCESS] id={}, error={}, type={}", 
            item.getId(), t.getMessage(), t.getClass().getSimpleName());

        // Channel 2: Error table (queryable)
        errorRepo.save(new ErrorRecord(
            "PROCESS", item.getId(), toJson(item), 
            t.getMessage(), LocalDateTime.now()));

        // Channel 3: Error file (easy review)
        try {
            errorFileWriter.write(new Chunk<>(
                item.getId() + "|" + t.getMessage() + "|" + item.toCsv()));
        } catch (Exception e) {
            log.error("Failed to write error record to file", e);
        }
    }
}

// Annotation-based listener (simpler)
@Component
public class SimpleSkipLogger {

    @OnSkipInRead
    public void onSkipRead(Throwable t) {
        log.error("Skipped in read: {}", t.getMessage());
    }

    @OnSkipInProcess
    public void onSkipProcess(Order item, Throwable t) {
        log.error("Skipped in process: id={} error={}", item.getId(), t.getMessage());
    }

    @OnSkipInWrite
    public void onSkipWrite(ProcessedOrder item, Throwable t) {
        log.error("Skipped in write: id={} error={}", item.getId(), t.getMessage());
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Error logging should NOT throw exceptions — catch and log separately *(error log mein exception aaye toh catch karo — job fail nahi hona chahiye)*
- `onSkipInRead` doesn't have the item (only exception) because read failed before creating object
- Use separate transaction for error table writes (don't participate in chunk transaction)
- Annotation listeners (`@OnSkipInRead`) are simpler but less flexible

### ⚡ Remember
- Three channels: log file + error table + error CSV
- `onSkipInRead` → only Throwable (no item) *(read skip mein item nahi milta)*
- `onSkipInProcess` → item + Throwable
- `onSkipInWrite` → processed item + Throwable
- Error logging must not fail the main job

### 🔗 Follow-ups
- [Q76 → Production bad record strategy](#q76)
- [Q78 → Storing rejected records for reprocessing](#q78)
- [Q70 → Skip logic setup](#q70)

---

## Q78. How do you store rejected records for later processing?

### 📝 One-Liner
Write rejected records to a dedicated error table with full item data, error details, and status — then run a reprocessing job on that table.

### 🔑 Quick Answer
Store rejected records in a queryable error table: item data (as JSON), error message, error type, phase, timestamp, and a `status` field (PENDING → REPROCESSED / IGNORED). After fixing the root cause, run a separate reprocessing job that reads from the error table (status=PENDING), processes the items, and updates status. For file-based flows, write an error CSV file alongside the main output. *(Error table mein rakho — status field se track karo — baad mein reprocess karo)*

### 📖 How It Works
```
Rejected Record Lifecycle:

Main Job → skip item → SkipListener → INSERT into ERROR_RECORDS (status=PENDING)
                                           ↓
Ops Team → query ERROR_RECORDS → investigate root cause → fix
                                           ↓
Reprocess Job → READ from ERROR_RECORDS (status=PENDING)
             → process item → write to main table
             → UPDATE ERROR_RECORDS set status=REPROCESSED

ERROR_RECORDS schema:
| Column | Type | Description |
|--------|------|-------------|
| id | BIGINT | Auto-increment PK |
| job_execution_id | BIGINT | Which job run |
| step_name | VARCHAR | Which step |
| phase | VARCHAR | READ/PROCESS/WRITE |
| item_data | TEXT (JSON) | Original item as JSON |
| error_message | VARCHAR | Exception message |
| error_type | VARCHAR | Exception class name |
| status | VARCHAR | PENDING/REPROCESSED/IGNORED |
| created_at | TIMESTAMP | When skipped |
| reprocessed_at | TIMESTAMP | When reprocessed (nullable) |
```

### 🗣️ How to Say in Interview
"I store rejected records in a dedicated ERROR_RECORDS table with the full item data serialized as JSON, error details, and a status field — initially PENDING. The ops team queries this table to investigate patterns. After fixing the root cause, we run a separate reprocessing job that reads PENDING records from the error table, processes them, writes to the main table, and updates the status to REPROCESSED. In my project, this pattern recovered 95% of rejected records — only 5% were genuinely invalid and marked as IGNORED after manual review."

### 💻 Code
```java
// Error record entity
@Entity
@Table(name = "error_records")
public class ErrorRecord {
    @Id @GeneratedValue
    private Long id;
    private Long jobExecutionId;
    private String stepName;
    private String phase;           // READ, PROCESS, WRITE
    @Column(columnDefinition = "TEXT")
    private String itemData;        // JSON-serialized item
    private String errorMessage;
    private String errorType;       // Exception class name
    private String status;          // PENDING, REPROCESSED, IGNORED
    private LocalDateTime createdAt;
    private LocalDateTime reprocessedAt;
}

// SkipListener stores rejected records
@Component
public class RejectedRecordLogger implements SkipListener<Order, ProcessedOrder> {
    @Autowired private ErrorRecordRepository repo;
    @Autowired private ObjectMapper mapper;

    @Override
    public void onSkipInProcess(Order item, Throwable t) {
        repo.save(ErrorRecord.builder()
            .phase("PROCESS")
            .itemData(mapper.writeValueAsString(item))
            .errorMessage(t.getMessage())
            .errorType(t.getClass().getSimpleName())
            .status("PENDING")
            .createdAt(LocalDateTime.now())
            .build());
    }
}

// Reprocessing job
@Bean
public Job reprocessJob(JobRepository repo, Step reprocessStep) {
    return new JobBuilder("reprocessFailedRecords", repo)
            .start(reprocessStep)
            .build();
}

@Bean
public Step reprocessStep(JobRepository repo, PlatformTransactionManager tx) {
    return new StepBuilder("reprocessStep", repo)
            .<ErrorRecord, Order>chunk(100, tx)
            .reader(errorTableReader())      // read PENDING records
            .processor(reprocessProcessor()) // deserialize + process
            .writer(mainTableWriter())       // write to main table
            .listener(reprocessStatusUpdater()) // update status
            .build();
}
```

### ⚠️ Pitfalls / Gotchas
- Reprocess job must be idempotent — safe to run multiple times *(reprocess job idempotent hona chahiye)*
- Store item as JSON, not Java serialization (survives class changes)
- Error table can grow large — add retention policy (archive/delete old records)
- Use separate datasource connection for error writes to avoid chunk transaction interference

### ⚡ Remember
- Error table: item_data (JSON) + error + status (PENDING/REPROCESSED) *(JSON mein save karo — class badle toh bhi kaam kare)*
- Reprocessing job reads PENDING → processes → updates status
- Always queryable by ops team
- Retention policy for table growth
- Reprocess job must be idempotent

### 🔗 Follow-ups
- [Q76 → Production error handling strategy](#q76)
- [Q77 → Logging failed records](#q77)
- [Q117 → Production scenarios](#q117)
