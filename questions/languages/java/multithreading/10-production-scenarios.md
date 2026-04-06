# 🏭 Real Production Scenarios (Q87–Q96)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q87"></a>
## Q87. How to process millions of records using multithreading?

### 📝 One-Liner
> Partitioned step in Spring Batch — split by ID range, each worker has own reader, batch writes for throughput.

### 🔑 Quick Answer
> Use **partitioning** — split data into ranges (by ID/date), each thread gets its own reader. In Spring Batch: **partitioned step** + `ColumnRangePartitioner` + `ThreadPoolTaskExecutor`. Combine with **JDBC batch writes** for max throughput. *(Data ko hisson mein baato — har thread apna hissa padhe — parallel process karo)*

### 📖 How It Works
```
Processing 10M records:

Single-threaded:
  Thread-1: Read 10M → Process → Write = ~60 minutes

Partitioned (10 workers):
  Worker-1:  IDs 1-1M      → Read → Process → Batch Write
  Worker-2:  IDs 1M-2M     → Read → Process → Batch Write
  ...
  Worker-10: IDs 9M-10M    → Read → Process → Batch Write
  = ~6 minutes (linear speedup!) ⭐
  *(10 hisse, 10 thread — 10x fast)*
```

### 💻 Code
```java
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", columnRangePartitioner())
        .step(workerStep())
        .gridSize(20)                    // 20 partitions
        .taskExecutor(batchExecutor())   // 20 threads
        .build();
}

@Bean @StepScope  // new reader per partition
public JdbcPagingItemReader<Record> reader(
        @Value("#{stepExecutionContext['minValue']}") Long minId,
        @Value("#{stepExecutionContext['maxValue']}") Long maxId) {
    JdbcPagingItemReader<Record> reader = new JdbcPagingItemReader<>();
    reader.setDataSource(dataSource);
    reader.setPageSize(5000);  // read 5000 at a time
    return reader;
}

@Bean
public JdbcBatchItemWriter<Record> writer() {
    JdbcBatchItemWriter<Record> writer = new JdbcBatchItemWriter<>();
    writer.setDataSource(dataSource);
    writer.setSql("INSERT INTO processed VALUES (:id, :result)");
    return writer;
}
```

### 🗣️ Answering Approach
> *"For millions of records, I use Spring Batch partitioned steps. The Partitioner queries min/max IDs and divides into equal ranges. Each worker gets its own JdbcPagingItemReader scoped to its partition via @StepScope. Workers run in a ThreadPoolTaskExecutor. Writers use JDBC batch inserts — batching 5000 records per write. Partitions are independent — no shared state, no locking, linear scaling. I've processed 50 million records this way in under 30 minutes."*

### ⚡ Remember
1. **Partition by ID range** — independent workers
2. **@StepScope** reader — new per partition ⭐
3. **JDBC batch writes** — batch 1000-5000
4. **gridSize** = number of partitions
5. **Linear scaling** — double workers ≈ half time

### 🔗 Follow-ups
→ [Q81. Partitioning](09-spring-multithreading.md#q81)

---

<a id="q88"></a>
## Q88. How to ensure thread safety for a shared resource?

### 📝 One-Liner
> Prefer: no sharing > immutability > atomic variables > concurrent collections > synchronized (minimize scope).

### 🔑 Quick Answer
> Choose by access pattern: **immutability** (best — no sync needed), **atomic variables** (single value updates), **ConcurrentHashMap** (thread-safe map ops), **synchronized** (compound ops, minimize scope). Spring services should be **stateless**. *(Share mat karo > badal mat do > atomic use karo > lock kam karo)*

### 📖 How It Works
```
Strategy selection:
  Read-only config       → Immutable object ✅ (koi sync nahi)
  Simple counter/flag    → AtomicInteger/volatile ✅
  Shared map             → ConcurrentHashMap ✅
  Read-heavy list        → CopyOnWriteArrayList ✅
  Complex business logic → synchronized block ✅ (scope chhota)
  Per-thread resource    → ThreadLocal ✅ (sharing hi nahi)
  No sharing needed      → Local variable ✅ (BEST)
```

### 💻 Code
```java
@Service
public class OrderService {
    private final AppConfig config;               // immutable — safe
    private final AtomicLong count = new AtomicLong(0);  // atomic — safe
    private final ConcurrentHashMap<String, Order> cache = new ConcurrentHashMap<>();

    public Order process(OrderRequest req) {
        Order order = createOrder(req);           // local var — safe
        count.incrementAndGet();                  // atomic — no lock
        cache.compute(order.getId(), (k, v) -> {  // atomic map op
            if (v != null) throw new DuplicateException(k);
            return order;
        });
        return order;
    }
}
```

### 🗣️ Answering Approach
> *"My approach is to minimize sharing first. If immutable or thread-local, no synchronization needed. For single values, AtomicInteger — lock-free CAS. For maps, ConcurrentHashMap with compute/merge for atomic compound operations. synchronized blocks as last resort with smallest scope. In Spring, services are singletons — I keep them stateless, with state in the database."*

### ⚡ Remember
1. **Best**: no sharing or immutability *(badal nahi sakta = safe)*
2. **Good**: AtomicXxx (lock-free)
3. **OK**: ConcurrentHashMap
4. **Last resort**: synchronized (minimize scope)
5. Spring services = **stateless** ⭐

### 🔗 Follow-ups
→ [Q68. Thread safety](07-deadlock-problems.md#q68)

---

<a id="q89"></a>
## Q89. How to debug a deadlock in production?

### 📝 One-Liner
> Thread dump (jstack) → find lock cycle → fix lock ordering or use tryLock with timeout → add ThreadMXBean monitoring.

### 🔑 Quick Answer
> **Detect**: `jstack <pid>` — shows "Found Java-level deadlock". **Analyze**: identify lock cycle and stack traces. **Fix**: enforce consistent lock ordering or tryLock + timeout. **Prevent**: add `ThreadMXBean` scheduled check. *(jstack lo — deadlock apne aap dikh jaayega — lock order fix karo)*

### 📖 How It Works
```
Step-by-step:
  1. SYMPTOM: App hangs, no errors, requests timeout
     → CPU low but threads stuck
  
  2. GET DUMPS:
     $ jstack <pid> > dump1.txt
     $ sleep 5; jstack <pid> > dump2.txt
     $ sleep 5; jstack <pid> > dump3.txt
  
  3. FIND DEADLOCK:
     "Found one Java-level deadlock"
     Thread-1: waiting to lock 0x76ab34c70 (held by Thread-2)
     Thread-2: waiting to lock 0x76ab34c60 (held by Thread-1)
  
  4. FIX: Lock ordering or tryLock(timeout)
```

### 💻 Code
```java
// Production deadlock detector ⭐
@Component
public class DeadlockDetector {
    private final ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();

    @Scheduled(fixedRate = 10000)  // every 10 sec
    public void detect() {
        long[] ids = mxBean.findDeadlockedThreads();
        if (ids != null) {
            ThreadInfo[] infos = mxBean.getThreadInfo(ids, true, true);
            for (ThreadInfo info : infos) {
                log.error("DEADLOCK: {} waiting for {} held by {}",
                    info.getThreadName(), info.getLockName(), info.getLockOwnerName());
            }
            alertService.sendCritical("Deadlock detected!");
        }
    }
}
```

### 🗣️ Answering Approach
> *"When I suspect deadlock — hanging requests, low CPU — I take 3 thread dumps with jstack, 5 seconds apart. The JVM auto-detects deadlocks in the dump. I check which locks are involved and find the code with inconsistent lock ordering. The fix is either enforcing a global lock ordering or switching to tryLock with timeout. For prevention, I add a scheduled DeadlockDetector using ThreadMXBean."*

### ⚡ Remember
1. **jstack** = take thread dump ⭐
2. **3+ dumps**, 5 sec apart
3. Fix: **lock ordering** or **tryLock(timeout)**
4. **ThreadMXBean** for automated detection
5. Deadlocks are **silent** — proactive monitoring key

### 🔗 Follow-ups
→ [Q65. Prevention](07-deadlock-problems.md#q65)

---

<a id="q90"></a>
## Q90. How to limit concurrent threads accessing a resource?

### 📝 One-Liner
> Semaphore (permit-based limiting), bounded ThreadPoolExecutor (pool-size limiting), or Resilience4j Bulkhead.

### 🔑 Quick Answer
> **Semaphore** = most flexible — acquire permit before access, release after. **ThreadPoolTaskExecutor** with fixed size = natural limiter. **Resilience4j Bulkhead** = circuit-breaker style. Always release in finally, use tryAcquire with timeout. *(Semaphore = N pass — jiske paas pass woh andar)*

### 💻 Code
```java
// Semaphore — limit to 5 concurrent
private final Semaphore limiter = new Semaphore(5);

public Result query(String sql) throws InterruptedException {
    limiter.acquire();  // wait for permit
    try {
        return executeQuery(sql);
    } finally {
        limiter.release();  // ALWAYS in finally! ⭐
    }
}

// With timeout
if (!limiter.tryAcquire(5, TimeUnit.SECONDS)) {
    throw new ServiceUnavailableException("Too many concurrent requests");
}
```

### ⚠️ Pitfalls / Gotchas
- **Always release in finally** *(bhul gaye toh permit kho gaya — aur koi nahi jaa payega)*
- Release without acquire = permit count grows beyond initial *(galti se zyada release — limit badh jaayegi)*

### 🗣️ Answering Approach
> *"I use Semaphore to limit concurrent access — like limiting connections to an external API that allows max 5 concurrent calls. acquire() takes a permit, release() returns it — always in finally. For timeout-based limiting, tryAcquire with timeout avoids indefinite waiting. In Spring, a fixed-size ThreadPoolTaskExecutor also serves as a natural concurrency limiter."*

### ⚡ Remember
1. **Semaphore** = most flexible limiter
2. **Always release in finally** ⭐
3. **tryAcquire(timeout)** — don't wait forever
4. **ThreadPoolTaskExecutor** = natural limiter
5. **Resilience4j Bulkhead** = circuit-breaker style

### 🔗 Follow-ups
→ [Q48. Semaphore details](04-concurrency-utilities.md#q48)

---

<a id="q91"></a>
## Q91. What happens when a thread pool is exhausted?

### 📝 One-Liner
> Core → Queue → Max → Rejection policy: AbortPolicy (exception), CallerRunsPolicy (caller runs it — backpressure), DiscardPolicy (drops silently).

### 🔑 Quick Answer
> When **all threads busy AND queue full**: rejection policy activates. **CallerRunsPolicy** (⭐ best — caller thread runs task, natural backpressure), **AbortPolicy** (default — throws RejectedExecutionException), **DiscardPolicy** (drops silently — dangerous). *(Sab busy, queue bhari — ab kya karna hai yeh rejection policy decide karti hai)*

### 📖 How It Works
```
Pool: core=5, max=10, queue=100

1. 5 tasks → 5 core threads ✅
2. 100 more → queued ✅
3. 5 more → 5 extra threads (up to max=10) ✅
4. Task 111 → ALL FULL! → REJECTION POLICY

CallerRunsPolicy effect:
  HTTP thread → submit task → pool full → HTTP thread RUNS it
  → HTTP thread busy → stops accepting requests
  → Natural backpressure! Server slows instead of crashing ✅
  *(Caller khud kaam karega — server slow hoga par crash nahi)*
```

### 🆚 vs. Comparison
| Policy | Behavior | Production? |
|--------|----------|------------|
| **CallerRunsPolicy** | Caller thread runs task ⭐ | ✅ Best |
| AbortPolicy | Throws exception (default) | ⚠️ Fail-fast |
| DiscardPolicy | Silently drops ❌ | ❌ Data loss |
| DiscardOldestPolicy | Drops oldest queued | ❌ Data loss |

### 🗣️ Answering Approach
> *"When everything is maxed out, the rejection policy handles overflow. I always use CallerRunsPolicy — the submitting thread runs the task itself, creating natural backpressure. The caller slows down, preventing the server from being overwhelmed. AbortPolicy throws an exception — suitable for fail-fast. DiscardPolicy silently drops — dangerous because work is lost. Monitoring queue size and rejections is critical."*

### ⚡ Remember
1. **Core → Queue → Max → Reject** (exhaustion flow)
2. **CallerRunsPolicy** = best for production ⭐
3. **AbortPolicy** = default (throws exception)
4. **DiscardPolicy** = dangerous (silent loss)
5. **Monitor** queue size and rejection count

### 🔗 Follow-ups
→ [Q92. Monitor performance](#q92)

---

<a id="q92"></a>
## Q92. How to monitor thread performance?

### 📝 One-Liner
> Micrometer metrics (active count, queue size, rejections), Spring Actuator endpoints, Grafana dashboards, periodic logging.

### 🔑 Quick Answer
> **Micrometer gauges** for real-time metrics → **Grafana** dashboards. Monitor: **active thread count**, **queue size**, **queue utilization**, **rejection count**, **completed tasks**. Alert when queue > 80% or any rejection. *(Queue kitni bhari, kitne threads busy, kitne reject — sab monitor karo)*

### 💻 Code
```java
@Component
public class ThreadPoolMonitor {
    private final MeterRegistry registry;
    private final ThreadPoolTaskExecutor executor;

    public ThreadPoolMonitor(MeterRegistry registry,
            @Qualifier("taskExecutor") ThreadPoolTaskExecutor executor) {
        this.registry = registry;
        this.executor = executor;
        ThreadPoolExecutor pool = executor.getThreadPoolExecutor();

        Gauge.builder("pool.active", pool, ThreadPoolExecutor::getActiveCount)
             .register(registry);
        Gauge.builder("pool.queue.size", pool, e -> e.getQueue().size())
             .register(registry);
        Gauge.builder("pool.queue.util", pool,
             e -> (double) e.getQueue().size() / 
                  (e.getQueue().size() + e.getQueue().remainingCapacity()))
             .register(registry);  // alert when > 0.8 ⚠️
    }

    @Scheduled(fixedRate = 30000)
    public void log() {
        ThreadPoolExecutor pool = executor.getThreadPoolExecutor();
        log.info("Pool: active={}, queue={}, completed={}",
            pool.getActiveCount(), pool.getQueue().size(), pool.getCompletedTaskCount());
    }
}
```

### 🗣️ Answering Approach
> *"I monitor pools at three levels. Micrometer gauges for real-time metrics — active threads, queue size, utilization. These feed into Grafana for dashboards and alerting. I alert when queue utilization exceeds 80% or any rejections occur. Periodic logging as backup. Spring Actuator exposes /metrics and /threaddump endpoints. Thread dumps via Actuator provide snapshots when needed."*

### ⚡ Remember
1. **Micrometer + Grafana** for dashboards
2. Monitor: **active, queue size, rejections** ⭐
3. Alert on **queue > 80%** and **any rejections**
4. **Actuator**: /metrics, /threaddump
5. **Periodic logging** as backup

### 🔗 Follow-ups
→ [Q70. Pool tuning](08-performance.md#q70)

---

<a id="q93"></a>
## Q93. How to cancel a running thread?

### 📝 One-Liner
> Cooperative cancellation: Thread.interrupt() sets flag, thread checks it and stops — never use Thread.stop().

### 🔑 Quick Answer
> **Cooperative** — thread must check and stop itself. `Thread.interrupt()` sets the flag, thread checks `isInterrupted()` or catches `InterruptedException`. For executor tasks, `future.cancel(true)`. **Never** `Thread.stop()` (deprecated, unsafe). *(Thread se request karo "band karo" — forcefully nahi maar sakte)*

### 💻 Code
```java
public class DataProcessor implements Runnable {
    private volatile boolean cancelled = false;

    @Override
    public void run() {
        try {
            while (!cancelled && !Thread.currentThread().isInterrupted()) {
                Record r = readNext();
                if (r == null) break;
                process(r);
                Thread.sleep(10);  // throws InterruptedException if interrupted
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();  // RESTORE flag! ⭐
            log.info("Interrupted — shutting down");
        } finally {
            cleanup();  // always cleanup
        }
    }

    public void cancel() { cancelled = true; }
}

// Future-based cancellation
Future<?> future = executor.submit(processor);
future.cancel(true);  // sends interrupt
```

### ⚠️ Pitfalls / Gotchas
- **Never swallow InterruptedException** — always restore flag with `Thread.currentThread().interrupt()` *(flag restore karo — warna caller ko pata nahi chalega)*
- **Thread.stop()** releases all locks instantly → corrupted state *(deprecated — use mat karo)*

### 🗣️ Answering Approach
> *"Java uses cooperative cancellation. Thread.interrupt() sets the interrupt flag — the thread must check isInterrupted() or handle InterruptedException from blocking operations. For executor tasks, Future.cancel(true) sends an interrupt. The critical rule: when catching InterruptedException, always restore the interrupt flag. Never use Thread.stop() — it's deprecated and unsafe."*

### ⚡ Remember
1. **Cooperative** — thread must check and stop
2. **interrupt()** sets flag, thread checks it
3. **Restore interrupt flag** when catching IE ⭐
4. **future.cancel(true)** for executor tasks
5. **Never** Thread.stop() *(deprecated, unsafe)*

### 🔗 Follow-ups
→ [Q94. Safe shutdown](#q94)

---

<a id="q94"></a>
## Q94. How to stop a long-running thread safely?

### 📝 One-Liner
> Signal (set flag + shutdown()) → Wait (awaitTermination) → Force (shutdownNow) — three-phase graceful shutdown.

### 🔑 Quick Answer
> **Phase 1: Signal** — set volatile flag + `executor.shutdown()`. **Phase 2: Wait** — `awaitTermination(timeout)`. **Phase 3: Force** — `shutdownNow()` if timeout exceeded. Thread uses **timeout-based blocking** operations so it wakes periodically to check flag. *(Pehle bolo "band karo", phir wait karo, phir force karo)*

### 💻 Code
```java
public class Processor {
    private final ExecutorService executor = Executors.newFixedThreadPool(5);
    private volatile boolean shutdownRequested = false;

    public void start() {
        executor.submit(() -> {
            try {
                while (!shutdownRequested && !Thread.currentThread().isInterrupted()) {
                    Message msg = queue.poll(1, TimeUnit.SECONDS);  // timeout!
                    if (msg != null) process(msg);
                    // poll returns null → re-checks flag
                }
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            } finally {
                cleanup();
            }
        });
    }

    public void shutdown() {
        shutdownRequested = true;       // Phase 1: Signal
        executor.shutdown();            // stop accepting new tasks
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {  // Phase 2: Wait
                executor.shutdownNow(); // Phase 3: Force
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
    }
}
```

### 🗣️ Answering Approach
> *"Three-phase shutdown. Phase one: signal intent — set a volatile flag and call executor.shutdown(). Phase two: wait — awaitTermination gives tasks a grace period. Tasks check the shutdown flag and use timeout-based blocking to wake periodically. Phase three: force — shutdownNow() sends interrupts if tasks don't finish. In Spring, I configure waitForTasksToCompleteOnShutdown on the TaskExecutor."*

### ⚡ Remember
1. **Signal → Wait → Force** (3-phase) ⭐
2. Use **timeout-based blocking** (poll with timeout)
3. Check **both** volatile flag AND interrupted
4. **finally** block for cleanup
5. Spring: **waitForTasksToCompleteOnShutdown** = true

### 🔗 Follow-ups
→ [Q93. Cancel thread](#q93)

---

<a id="q95"></a>
## Q95. How to design a high-performance concurrent system?

### 📝 One-Liner
> Separate pools per stage (I/O, CPU, write), bounded queues for backpressure, stateless services, batch I/O, monitor everything.

### 🔑 Quick Answer
> Design as **pipeline**: separate pools for I/O (50 threads), CPU (= cores), write (batch). **Bounded queues** between stages for backpressure. **Stateless** services (state in DB). **Lock-free** structures. **Batch I/O**. **Monitor** every stage. *(Har stage ka alag pool, sab bounded, state DB mein, monitor karo)*

### 📖 How It Works
```
Pipeline Architecture:
  Requests → Rate Limiter
           ↓
  I/O Pool (50 threads) → read from DB, call APIs
           ↓ bounded queue
  CPU Pool (8 threads = cores) → transform, validate
           ↓ bounded queue
  Write Pool (20 threads) → batch write to DB
  
  *(Har stage alag — I/O ka alag, CPU ka alag, write ka alag)*
```

### 💻 Code
```java
@Bean("ioExecutor")
public ThreadPoolTaskExecutor ioExecutor() {
    var e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(50);  // many for I/O waiting
    e.setQueueCapacity(1000);
    e.setThreadNamePrefix("io-");
    return e;
}

@Bean("cpuExecutor")
public ThreadPoolTaskExecutor cpuExecutor() {
    int cores = Runtime.getRuntime().availableProcessors();
    var e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(cores);     // = CPU cores
    e.setMaxPoolSize(cores);      // don't exceed!
    e.setThreadNamePrefix("cpu-");
    return e;
}

@Bean("writerExecutor")
public ThreadPoolTaskExecutor writerExecutor() {
    var e = new ThreadPoolTaskExecutor();
    e.setCorePoolSize(20);
    e.setQueueCapacity(200);
    e.setThreadNamePrefix("writer-");
    return e;
}
```

### 🗣️ Answering Approach
> *"I design concurrent systems as a pipeline with separate pools per stage. I/O stage has many threads — they mostly wait. CPU stage = CPU cores — more just wastes on context switching. Write stage batches operations for throughput. Bounded queues between stages provide backpressure. Services are stateless — shared state in the database. I use ConcurrentHashMap for caches, LongAdder for counters, and monitor every stage with Micrometer."*

### ⚡ Remember
1. **Separate pools** per stage (I/O, CPU, write)
2. **Bounded queues** = backpressure ⭐
3. **Stateless** services — state in DB/cache
4. **Batch I/O** — batch writes, batch API calls
5. **Monitor** every stage (queue size, latency, errors)

### 🔗 Follow-ups
→ [Q70. Pool tuning](08-performance.md#q70)

---

<a id="q96"></a>
## Q96. How to maintain database consistency with multiple threads?

### 📝 One-Liner
> Optimistic locking (@Version + retry), pessimistic locking (SELECT FOR UPDATE), or atomic SQL updates — choose by contention level.

### 🔑 Quick Answer
> **Optimistic lock** (@Version): read + update, retry on conflict — best for **low contention**. **Pessimistic lock** (SELECT FOR UPDATE): locks row until commit — best for **high contention**. **Atomic SQL** (UPDATE SET x = x - 1): single statement, simplest. *(Kam ladai = optimistic; zyada ladai = pessimistic; simple = atomic SQL)*

### 📖 How It Works
```
Without protection:
  T1: SELECT balance → 1000
  T2: SELECT balance → 1000
  T1: UPDATE = 1000 - 100 = 900 ✅
  T2: UPDATE = 1000 - 200 = 800 ← WRONG! Should be 700 💀

Optimistic Locking (@Version):
  T1: SELECT (1000, v1)
  T2: SELECT (1000, v1)
  T1: UPDATE WHERE v=v1 → SUCCESS (v2)
  T2: UPDATE WHERE v=v1 → FAIL! → RETRY → (900, v2) → 700 ✅
  *(Version match nahi hua — retry karo)*

Pessimistic Locking (FOR UPDATE):
  T1: SELECT FOR UPDATE → 1000 (locks row)
  T2: SELECT FOR UPDATE → BLOCKED...
  T1: UPDATE 900 → COMMIT → unlock
  T2: unblocked → 900 → UPDATE 700 → COMMIT ✅
  *(Row lock kar di — doosra wait kare)*
```

### 💻 Code
```java
// Optimistic: @Version + @Retryable
@Entity
public class Account {
    @Id private Long id;
    private BigDecimal balance;
    @Version private Long version;  // auto-managed by JPA
}

@Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
@Transactional
public void debit(Long id, BigDecimal amount) {
    Account acc = repo.findById(id).orElseThrow();
    acc.setBalance(acc.getBalance().subtract(amount));
    repo.save(acc);  // version mismatch → retry
}

// Pessimistic: SELECT FOR UPDATE
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
Account findForUpdate(@Param("id") Long id);

// Atomic SQL (simplest!) ⭐
@Modifying
@Query("UPDATE Account SET balance = balance - :amt WHERE id = :id AND balance >= :amt")
int debitAtomic(@Param("id") Long id, @Param("amt") BigDecimal amt);
// returns 0 = insufficient, 1 = success — fully atomic!
```

### 🆚 vs. Comparison
| Strategy | Contention | Performance | Complexity |
|----------|-----------|-------------|-----------|
| **Optimistic** (@Version) | Low ✅ | High (no lock wait) | Medium (retry) |
| **Pessimistic** (FOR UPDATE) | High ✅ | Lower (blocks) | Low |
| **Atomic SQL** | Any ✅ | Highest ⭐ | Lowest ⭐ |

### 🗣️ Answering Approach
> *"For database consistency with concurrent threads, I choose between three approaches. Low contention — optimistic locking with @Version and @Retryable handles retry on conflict. High contention — pessimistic locking with SELECT FOR UPDATE blocks other threads until commit. For simple operations — atomic SQL like 'UPDATE SET balance = balance - amount WHERE balance >= amount' is a single atomic statement, simplest and fastest. I choose based on contention level and complexity."*

### ⚡ Remember
1. **Optimistic** (@Version) = low contention, retry on conflict
2. **Pessimistic** (FOR UPDATE) = high contention, blocks
3. **Atomic SQL** = simplest for inc/dec operations ⭐
4. **@Retryable** + optimistic = production pattern
5. Choose by **contention level** and **complexity**

### 🔗 Follow-ups
→ [Q67. Race conditions](07-deadlock-problems.md#q67)

---

> **🎯 Navigation:** [← Spring Multithreading (Q77-86)](09-spring-multithreading.md) | [📋 All Sections](README.md)
