# 🏭 Real Production Scenarios (Q87–Q96)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q87"></a>

## Q87. How to process millions of records using multithreading?

### 🔑 Quick Answer

> Use **partitioning** — split data into ranges (by ID or date), assign each range to a thread with its own reader. In Spring Batch, use **partitioned steps** with `ColumnRangePartitioner` and a `ThreadPoolTaskExecutor`. Combine with **batch writes** (JDBC batch insert/update) for maximum throughput.

### 📖 Step-by-Step Explanation

```
Processing 10 million records:

Single-threaded:
  Thread-1: Read 10M → Process 10M → Write 10M
  Time: ~60 minutes

Partitioned (10 workers):
  Worker-1:  IDs 1-1M       → Read → Process → Batch Write
  Worker-2:  IDs 1M-2M      → Read → Process → Batch Write
  Worker-3:  IDs 2M-3M      → Read → Process → Batch Write
  ...
  Worker-10: IDs 9M-10M     → Read → Process → Batch Write
  Time: ~6 minutes (roughly linear speedup!)

Key architecture:
  ┌─────────────┐
  │ Partitioner  │ → Splits by ID range
  └──────┬──────┘
         │ ExecutionContext (minId, maxId)
    ┌────┴────┬────────┬────────┐
    ▼         ▼        ▼        ▼
  Worker-1  Worker-2 Worker-3 Worker-N
  (own Reader, Processor, Writer per partition)
```

### 💻 Code Example

```java
// Partitioned step for 10M records
@Bean
public Step managerStep() {
    return stepBuilderFactory.get("managerStep")
        .partitioner("workerStep", columnRangePartitioner())
        .step(workerStep())
        .gridSize(20)                    // 20 partitions
        .taskExecutor(batchExecutor())   // 20 threads
        .build();
}

@Bean
public ColumnRangePartitioner columnRangePartitioner() {
    ColumnRangePartitioner partitioner = new ColumnRangePartitioner();
    partitioner.setDataSource(dataSource);
    partitioner.setTable("records");
    partitioner.setColumn("id");
    return partitioner;
}

@Bean
@StepScope
public JdbcPagingItemReader<Record> reader(
        @Value("#{stepExecutionContext['minValue']}") Long minId,
        @Value("#{stepExecutionContext['maxValue']}") Long maxId) {
    // Each worker reads ONLY its partition
    JdbcPagingItemReader<Record> reader = new JdbcPagingItemReader<>();
    reader.setDataSource(dataSource);
    reader.setPageSize(5000);  // Read 5000 at a time
    // WHERE id >= :minId AND id <= :maxId
    return reader;
}

@Bean
public JdbcBatchItemWriter<Record> writer() {
    // Batch write for performance
    JdbcBatchItemWriter<Record> writer = new JdbcBatchItemWriter<>();
    writer.setDataSource(dataSource);
    writer.setSql("INSERT INTO processed_records VALUES (:id, :result)");
    return writer;
}
```

### 🗣️ How to Explain in Interview

> *"For millions of records, I use Spring Batch with partitioned steps. The Partitioner queries the min and max IDs from the database and divides the range into equal partitions — say 20 partitions for 10 million records, so 500K each. Each worker gets its own JdbcPagingItemReader scoped to its partition via @StepScope. Workers run in a ThreadPoolTaskExecutor. The writer uses JDBC batch inserts for throughput — batching 1000-5000 records per write. The key is that partitions are independent — no shared state, no locking, linear scaling. I've processed 50 million records this way in under 30 minutes."*

### ⚡ Key Points to Remember

1. **Partition by ID range** — each worker reads independently
2. **@StepScope** reader — new instance per partition
3. **JDBC batch writes** — batch size 1000-5000
4. **gridSize** = number of partitions/workers
5. **Linear scaling** — double workers ≈ half the time

---

<a id="q88"></a>

## Q88. How to ensure thread safety for a shared resource?

### 🔑 Quick Answer

> Choose the right strategy based on the access pattern: **immutability** (best — no synchronization needed), **atomic variables** (single value updates), **synchronized/ReentrantLock** (compound operations), **concurrent collections** (thread-safe data structures), or **thread-local** (per-thread copies).

### 📖 Step-by-Step Explanation

```
Strategy selection guide:

  Shared resource type         →  Strategy
  ─────────────────────────── → ──────────────────────
  Read-only configuration     →  Immutable object ✅
  Simple counter/flag         →  AtomicInteger/volatile ✅
  Shared map                  →  ConcurrentHashMap ✅
  Read-heavy list             →  CopyOnWriteArrayList ✅
  Check-then-act on map       →  ConcurrentHashMap.compute() ✅
  Complex business logic      →  synchronized block ✅
  Need fairness/timeout       →  ReentrantLock ✅
  Per-thread formatting       →  ThreadLocal ✅
  No sharing needed           →  Local variable ✅ (best)
```

### 💻 Code Example

```java
@Service
public class OrderService {
    
    // 1. Immutable config — thread-safe by nature
    private final AppConfig config;
    
    // 2. Atomic counter — lock-free
    private final AtomicLong orderCount = new AtomicLong(0);
    
    // 3. Concurrent map — thread-safe operations
    private final ConcurrentHashMap<String, Order> orderCache = new ConcurrentHashMap<>();
    
    public Order processOrder(OrderRequest request) {
        // 4. Local variable — thread-confined, always safe
        Order order = createOrder(request);
        
        // Atomic increment — no lock needed
        long count = orderCount.incrementAndGet();
        
        // Atomic map operation — no race condition
        orderCache.compute(order.getId(), (key, existing) -> {
            if (existing != null) throw new DuplicateOrderException(key);
            return order;
        });
        
        return order;
    }
    
    // 5. Synchronized for complex business logic
    public synchronized void reconcile() {
        // Multiple reads and writes that must be atomic together
        BigDecimal total = calculateTotal();
        validateAgainstLedger(total);
        updateLedger(total);
    }
}
```

### 🗣️ How to Explain in Interview

> *"My approach is to minimize sharing first. If I can make a resource immutable or thread-local, I don't need synchronization at all. For single-value updates, I use AtomicInteger or AtomicLong — CAS-based, lock-free. For shared maps, ConcurrentHashMap with its atomic compute/merge operations. For complex multi-step operations that must be atomic, I use synchronized blocks with the smallest possible scope. In Spring applications, most services should be stateless — shared state lives in the database, and I use database transactions for consistency. The key principle is: prefer no sharing > immutability > atomic variables > locks."*

### ⚡ Key Points to Remember

1. **Best**: no sharing or immutability
2. **Good**: atomic variables (AtomicXxx)
3. **OK**: concurrent collections (ConcurrentHashMap)
4. **Last resort**: synchronized/locks (minimize scope)
5. Spring services should be **stateless** ⭐

---

<a id="q89"></a>

## Q89. How to debug a deadlock in production?

### 🔑 Quick Answer

> 1. **Detect** — thread dump via `jstack <pid>` (shows "Found Java-level deadlock")
> 2. **Analyze** — identify the lock cycle and stack traces
> 3. **Root cause** — inconsistent lock ordering or nested lock acquisition
> 4. **Fix** — enforce consistent lock ordering or use tryLock with timeout
> 5. **Prevent** — add deadlock monitoring with `ThreadMXBean`

### 📖 Step-by-Step Explanation

```
Step-by-step production deadlock debugging:

1. SYMPTOM: Application hangs, no errors, requests timeout
   → CPU is low but threads are stuck

2. GET THREAD DUMP:
   $ jstack <pid> > dump1.txt
   $ sleep 5
   $ jstack <pid> > dump2.txt
   $ sleep 5
   $ jstack <pid> > dump3.txt

3. LOOK FOR DEADLOCK:
   Found one Java-level deadlock:
   =============================
   "http-nio-8080-exec-1":
     waiting to lock 0x00000076ab34c70 (Account@123)
     which is held by "http-nio-8080-exec-2"
   "http-nio-8080-exec-2":
     waiting to lock 0x00000076ab34c60 (Account@456)
     which is held by "http-nio-8080-exec-1"

4. READ STACK TRACES:
   → Find the exact lines of code where locks are acquired
   → Understand the lock ordering that caused the cycle

5. FIX:
   → Enforce consistent lock ordering (e.g., by account ID)
   → Or replace synchronized with tryLock(timeout)
```

### 💻 Code Example

```java
// Production deadlock detector — add to your Spring app
@Component
@Slf4j
public class DeadlockDetector {
    
    private final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    
    @Scheduled(fixedRate = 10000)  // Check every 10 seconds
    public void detectDeadlocks() {
        long[] deadlockedIds = threadMXBean.findDeadlockedThreads();
        
        if (deadlockedIds == null) return;
        
        log.error("🚨 DEADLOCK DETECTED! {} threads deadlocked", deadlockedIds.length);
        
        ThreadInfo[] infos = threadMXBean.getThreadInfo(deadlockedIds, true, true);
        for (ThreadInfo info : infos) {
            log.error("Deadlocked thread: {} (state: {})", 
                info.getThreadName(), info.getThreadState());
            log.error("  Waiting for lock: {} held by: {}", 
                info.getLockName(), info.getLockOwnerName());
            
            // Print stack trace
            for (StackTraceElement element : info.getStackTrace()) {
                log.error("    at {}", element);
            }
        }
        
        // Trigger alert (PagerDuty, Slack, etc.)
        alertingService.sendCriticalAlert("Deadlock detected in production!");
    }
}
```

### 🗣️ How to Explain in Interview

> *"When I suspect a deadlock in production — typically from hanging requests and low CPU — I immediately take 3 thread dumps with jstack, 5 seconds apart. The JVM automatically detects deadlocks and reports them in the dump. I check which locks are involved and the stack traces to find the exact code. Usually, it's inconsistent lock ordering — Thread-1 locks A then B, Thread-2 locks B then A. The fix is enforcing a global lock ordering. For prevention, I add a scheduled DeadlockDetector using ThreadMXBean.findDeadlockedThreads() that checks every 10 seconds and alerts the team immediately."*

### ⚡ Key Points to Remember

1. Take **3+ thread dumps**, 5 seconds apart
2. **jstack** auto-detects deadlocks
3. Look for **lock cycle** in dump output
4. Fix: **consistent lock ordering** or **tryLock(timeout)**
5. Add **automated detection** with ThreadMXBean

---

<a id="q90"></a>

## Q90. How to limit concurrent threads accessing a resource?

### 🔑 Quick Answer

> Use **Semaphore** to limit concurrent access (permits model), **ThreadPoolExecutor** with bounded pool size, or **rate limiters** (like Resilience4j). Semaphore is the most flexible — acquire a permit before access, release after.

### 📖 Step-by-Step Explanation

```
Semaphore-based limiting:

  Semaphore(3)  →  3 permits available

  Thread-1: acquire() → permit 1 → [ACCESS RESOURCE] → release()
  Thread-2: acquire() → permit 2 → [ACCESS RESOURCE] → release()
  Thread-3: acquire() → permit 3 → [ACCESS RESOURCE] → release()
  Thread-4: acquire() → BLOCKED (no permits) → waits...
  Thread-5: acquire() → BLOCKED → waits...
  
  Thread-1: release() → permit returned
  Thread-4: unblocked → permit 1 → [ACCESS RESOURCE] → release()
```

### 💻 Code Example

```java
// Limit to 5 concurrent database connections
@Service
public class DatabaseService {
    
    private final Semaphore connectionLimiter = new Semaphore(5);
    
    public Result query(String sql) throws InterruptedException {
        connectionLimiter.acquire();  // Wait for permit
        try {
            return executeQuery(sql);
        } finally {
            connectionLimiter.release();  // Always release!
        }
    }
    
    // With timeout — don't wait forever
    public Result queryWithTimeout(String sql) throws InterruptedException {
        if (!connectionLimiter.tryAcquire(5, TimeUnit.SECONDS)) {
            throw new ServiceUnavailableException("Too many concurrent requests");
        }
        try {
            return executeQuery(sql);
        } finally {
            connectionLimiter.release();
        }
    }
}

// Alternative: bounded thread pool
@Bean
public ThreadPoolTaskExecutor limitedExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(5);
    executor.setMaxPoolSize(5);         // Hard limit: 5 concurrent
    executor.setQueueCapacity(100);     // Queue overflow
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    return executor;
}
```

### 🗣️ How to Explain in Interview

> *"I use Semaphore to limit concurrent access — for example, limiting connections to a downstream API that allows only 5 concurrent calls. Semaphore has acquire() which blocks until a permit is available, and release() which returns the permit. I always release in a finally block. For more robust limiting, I use tryAcquire with a timeout to avoid indefinite waiting. In Spring applications, I also use ThreadPoolTaskExecutor with a fixed pool size as a natural concurrency limiter — only N threads can execute simultaneously, and the queue handles overflow."*

### ⚡ Key Points to Remember

1. **Semaphore** = most flexible concurrency limiter
2. **Always release** in finally block ⭐
3. **tryAcquire(timeout)** — don't wait forever
4. **ThreadPoolTaskExecutor** = natural limiter via pool size
5. **Resilience4j BulkHead** = circuit-breaker-style limiting

---

<a id="q91"></a>

## Q91. What happens when a thread pool is exhausted?

### 🔑 Quick Answer

> When all threads are busy AND the queue is full, the **rejection policy** activates. Four built-in policies: **AbortPolicy** (throws exception — default), **CallerRunsPolicy** (caller thread executes — best for back-pressure), **DiscardPolicy** (silently drops), **DiscardOldestPolicy** (drops oldest queued task).

### 📖 Step-by-Step Explanation

```
Thread pool exhaustion flow:

  Pool: core=5, max=10, queue=100

  1. 5 tasks → 5 core threads created ✅
  2. 100 more tasks → queued (100 slots) ✅
  3. 5 more tasks → 5 extra threads (up to max=10) ✅
  4. Task 111 → ALL FULL! → REJECTION POLICY kicks in

Rejection Policies:
  ┌─────────────────────┬──────────────────────────────────┐
  │ Policy              │ Behavior                          │
  ├─────────────────────┼──────────────────────────────────┤
  │ AbortPolicy         │ Throws RejectedExecutionException│
  │ CallerRunsPolicy ⭐ │ Caller thread runs the task       │
  │ DiscardPolicy       │ Silently drops the task           │
  │ DiscardOldestPolicy │ Drops oldest queue task, adds new │
  └─────────────────────┴──────────────────────────────────┘

CallerRunsPolicy effect:
  HTTP thread → submit task → pool full → HTTP thread RUNS the task itself
  → HTTP thread is now busy → stops accepting new requests
  → Natural back-pressure! Server slows down instead of failing ✅
```

### 🗣️ How to Explain in Interview

> *"When all core threads are busy, new tasks go to the queue. When the queue is full, extra threads are created up to maxPoolSize. When everything is exhausted, the rejection policy activates. I always use CallerRunsPolicy in production — it makes the submitting thread run the task itself, creating natural back-pressure. The caller slows down, which prevents the server from being overwhelmed. AbortPolicy throws an exception — suitable when you want to fail fast. DiscardPolicy silently drops tasks — dangerous because work is lost without notification. Monitoring queue size and rejection count is critical."*

### ⚡ Key Points to Remember

1. **Core → Queue → Max → Reject** (exhaustion flow)
2. **CallerRunsPolicy** = best for production (back-pressure) ⭐
3. **AbortPolicy** = default (throws exception)
4. **DiscardPolicy** = dangerous (silent data loss)
5. **Monitor** queue size and rejection count

---

<a id="q92"></a>

## Q92. How to monitor thread performance?

### 🔑 Quick Answer

> Use **Micrometer metrics** (thread pool active count, queue size, completed tasks), **JMX/MBeans**, **thread dumps** for snapshots, **Spring Boot Actuator** for endpoints, and **APM tools** (Datadog, New Relic, Grafana) for dashboards and alerting.

### 💻 Code Example

```java
// Custom metrics for thread pool monitoring
@Component
public class ThreadPoolMonitor {
    
    private final MeterRegistry meterRegistry;
    private final ThreadPoolTaskExecutor executor;
    
    public ThreadPoolMonitor(MeterRegistry meterRegistry,
                             @Qualifier("taskExecutor") ThreadPoolTaskExecutor executor) {
        this.meterRegistry = meterRegistry;
        this.executor = executor;
        registerMetrics();
    }
    
    private void registerMetrics() {
        ThreadPoolExecutor pool = executor.getThreadPoolExecutor();
        
        Gauge.builder("threadpool.active", pool, ThreadPoolExecutor::getActiveCount)
             .tag("pool", "taskExecutor")
             .register(meterRegistry);
             
        Gauge.builder("threadpool.queue.size", pool, e -> e.getQueue().size())
             .tag("pool", "taskExecutor")
             .register(meterRegistry);
             
        Gauge.builder("threadpool.pool.size", pool, ThreadPoolExecutor::getPoolSize)
             .tag("pool", "taskExecutor")
             .register(meterRegistry);
             
        // Alert when queue > 80% full
        Gauge.builder("threadpool.queue.utilization", pool, 
             e -> (double) e.getQueue().size() / (e.getQueue().size() + e.getQueue().remainingCapacity()))
             .tag("pool", "taskExecutor")
             .register(meterRegistry);
    }
    
    @Scheduled(fixedRate = 30000)
    public void logPoolStats() {
        ThreadPoolExecutor pool = executor.getThreadPoolExecutor();
        log.info("Thread Pool Stats - Active: {}, Pool Size: {}, Queue: {}, Completed: {}",
            pool.getActiveCount(), pool.getPoolSize(),
            pool.getQueue().size(), pool.getCompletedTaskCount());
    }
}
```

### 🗣️ How to Explain in Interview

> *"I monitor thread pools at three levels. First, Micrometer gauges for real-time metrics — active thread count, queue size, queue utilization, completed tasks. These feed into Grafana dashboards for visualization. Second, periodic logging every 30 seconds as a safety net. Third, alerting — when queue utilization exceeds 80%, we get a warning. When rejection count increases, we get a critical alert. In Spring Boot, the Actuator /metrics endpoint exposes these metrics out of the box with the right configuration. Thread dumps via Actuator's /threaddump endpoint provide snapshots when needed."*

### ⚡ Key Points to Remember

1. **Micrometer** + **Grafana** for real-time dashboards
2. Monitor: **active count, queue size, rejections** ⭐
3. **Actuator** endpoints: /metrics, /threaddump
4. Alert on **queue > 80%** and **any rejections**
5. Periodic **logging** as backup monitoring

---

<a id="q93"></a>

## Q93. How to cancel a running thread?

### 🔑 Quick Answer

> Use **cooperative cancellation** via `Thread.interrupt()` — the thread checks `Thread.interrupted()` or catches `InterruptedException` and stops gracefully. For `Future`, call `future.cancel(true)`. **Never use Thread.stop()** — it's deprecated and unsafe.

### 📖 Step-by-Step Explanation

```
Cancellation mechanisms:

1. INTERRUPT (cooperative):
   Outsider: thread.interrupt();  → Sets interrupt flag
   Thread:   
     if (Thread.interrupted()) {  → Checks flag, clears it
         cleanup();
         return;
     }
   
   If thread is in wait/sleep/blocking I/O:
     → InterruptedException thrown → thread can handle

2. VOLATILE FLAG:
   Outsider: running = false;     → Sets flag
   Thread:   while (running) { }  → Checks flag each loop

3. FUTURE.CANCEL():
   Future<?> future = executor.submit(task);
   future.cancel(true);  → Interrupts the thread
   future.cancel(false); → Doesn't interrupt (cancel only if not started)

⚠️ NEVER use:
   Thread.stop()  → Deprecated! Releases all locks instantly → corrupted state
   Thread.suspend() → Deprecated! Can cause deadlocks
```

### 💻 Code Example

```java
// Correct cancellation pattern
public class DataProcessor implements Runnable {
    private volatile boolean cancelled = false;
    
    @Override
    public void run() {
        try {
            while (!cancelled && !Thread.currentThread().isInterrupted()) {
                Record record = readNext();
                if (record == null) break;
                
                process(record);  // Respects interruption
                
                // If doing blocking operations:
                Thread.sleep(10);  // Throws InterruptedException if interrupted
            }
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();  // Restore interrupt flag!
            log.info("Processing interrupted — shutting down gracefully");
        } finally {
            cleanup();  // Always cleanup resources
        }
    }
    
    public void cancel() {
        cancelled = true;
    }
}

// Using Future for cancellation
ExecutorService executor = Executors.newFixedThreadPool(5);
Future<?> future = executor.submit(new DataProcessor());

// Cancel after 30 seconds
future.cancel(true);  // Sends interrupt to the running thread
```

### 🗣️ How to Explain in Interview

> *"Java uses cooperative cancellation — you can't forcibly stop a thread safely. I use two mechanisms: First, Thread.interrupt() sets the interrupt flag. The thread must check this flag periodically via isInterrupted() or handle InterruptedException from blocking operations like sleep() or wait(). Second, a volatile boolean flag — the thread checks it in its main loop. For tasks submitted to an executor, Future.cancel(true) sends an interrupt. The critical rule: when catching InterruptedException, either re-throw it or restore the interrupt flag with Thread.currentThread().interrupt(). Never swallow it silently."*

### ⚡ Key Points to Remember

1. **Cooperative** cancellation — thread must check and stop
2. **Thread.interrupt()** sets flag — thread checks it
3. **volatile flag** for simple loop-based cancellation
4. **Restore interrupt flag** when catching InterruptedException ⭐
5. **Never** use Thread.stop() — deprecated and unsafe

---

<a id="q94"></a>

## Q94. How to stop a long-running thread safely?

### 🔑 Quick Answer

> Combine **interrupt + volatile flag** for the check, add **timeout** to blocking operations, implement **graceful shutdown** with cleanup in finally block, and use **ExecutorService.shutdownNow()** for pool-level shutdown.

### 💻 Code Example

```java
// Production-grade safe shutdown
public class LongRunningProcessor {
    private final ExecutorService executor;
    private volatile boolean shutdownRequested = false;
    
    public LongRunningProcessor() {
        this.executor = Executors.newFixedThreadPool(5);
    }
    
    public void startProcessing() {
        executor.submit(() -> {
            try {
                while (!shutdownRequested && !Thread.currentThread().isInterrupted()) {
                    try {
                        // Use timeout-based blocking operations
                        Message msg = queue.poll(1, TimeUnit.SECONDS);
                        if (msg != null) {
                            processMessage(msg);
                        }
                        // poll returns null on timeout → loop re-checks shutdown flag
                    } catch (InterruptedException e) {
                        Thread.currentThread().interrupt();
                        break;
                    }
                }
            } finally {
                cleanup();  // Close connections, flush buffers
                log.info("Processor shut down gracefully");
            }
        });
    }
    
    public void shutdown() {
        log.info("Shutdown requested...");
        shutdownRequested = true;
        
        executor.shutdown();  // No new tasks accepted
        try {
            if (!executor.awaitTermination(30, TimeUnit.SECONDS)) {
                log.warn("Forcing shutdown after 30s timeout");
                executor.shutdownNow();  // Interrupt running tasks
                
                if (!executor.awaitTermination(10, TimeUnit.SECONDS)) {
                    log.error("Executor did not terminate!");
                }
            }
        } catch (InterruptedException e) {
            executor.shutdownNow();
            Thread.currentThread().interrupt();
        }
        log.info("Shutdown complete");
    }
}

// Spring-managed graceful shutdown
@Component
public class GracefulShutdown implements DisposableBean {
    
    @Autowired
    private ThreadPoolTaskExecutor taskExecutor;
    
    @Override
    public void destroy() {
        taskExecutor.setWaitForTasksToCompleteOnShutdown(true);
        taskExecutor.setAwaitTerminationSeconds(30);
        taskExecutor.shutdown();
    }
}
```

### 🗣️ How to Explain in Interview

> *"For safe shutdown, I follow a three-phase approach. Phase one: signal intent — set a volatile shutdown flag and call executor.shutdown() to stop accepting new tasks. Phase two: wait — awaitTermination gives running tasks a grace period to finish. The tasks themselves check the shutdown flag and interrupt status in their main loops, and use timeout-based blocking operations so they wake up periodically to check. Phase three: force — if tasks don't complete within the timeout, shutdownNow() sends interrupts to all running threads. In Spring, I configure waitForTasksToCompleteOnShutdown on the TaskExecutor for automatic graceful shutdown."*

### ⚡ Key Points to Remember

1. **Signal → Wait → Force** (three-phase shutdown)
2. Use **timeout-based blocking** operations (poll with timeout)
3. Check **both** volatile flag AND Thread.interrupted()
4. **finally** block for cleanup (close resources)
5. Spring: **waitForTasksToCompleteOnShutdown** = true

---

<a id="q95"></a>

## Q95. How to design a high-performance concurrent system?

### 🔑 Quick Answer

> Follow these principles: **minimize shared state** (stateless where possible), **partition data** (each thread works on independent subset), use **lock-free** structures (CAS, ConcurrentHashMap), **batch I/O** operations, separate **CPU-bound and I/O-bound** in different pools, and design for **backpressure** at every stage.

### 📖 Step-by-Step Explanation

```
High-Performance Concurrent Architecture:

      ┌──────────────────────────────────────────┐
      │ Incoming Requests (HTTP/Kafka/Queue)      │
      └─────────────────┬────────────────────────┘
                        │ Rate Limiter + Backpressure
                        ▼
      ┌──────────────────────────────────────────┐
      │ I/O Pool (50 threads)                     │
      │  Read from DB, call APIs, receive messages│
      └─────────────────┬────────────────────────┘
                        │ BlockingQueue (bounded!)
                        ▼
      ┌──────────────────────────────────────────┐
      │ CPU Pool (= cores, e.g., 8 threads)       │
      │  Transform, validate, compute             │
      └─────────────────┬────────────────────────┘
                        │ Batch queue
                        ▼
      ┌──────────────────────────────────────────┐
      │ Write Pool (20 threads)                   │
      │  Batch write to DB, publish events        │
      └──────────────────────────────────────────┘

Key design principles:
  1. Stateless services (state in DB/cache, not in memory)
  2. Per-stage thread pools (I/O, CPU, Write — isolated)
  3. Bounded queues between stages (backpressure)
  4. Lock-free data structures where possible
  5. Batch I/O (batch DB writes, batch API calls)
  6. Monitoring at every stage (queue size, latency, errors)
```

### 💻 Code Example

```java
// High-performance pipeline architecture
@Configuration
public class PipelineConfig {
    
    // Stage 1: I/O-bound readers
    @Bean("ioExecutor")
    public ThreadPoolTaskExecutor ioExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(50);  // Many threads for I/O waiting
        e.setMaxPoolSize(100);
        e.setQueueCapacity(1000);
        e.setThreadNamePrefix("io-");
        return e;
    }
    
    // Stage 2: CPU-bound processors
    @Bean("cpuExecutor")
    public ThreadPoolTaskExecutor cpuExecutor() {
        int cores = Runtime.getRuntime().availableProcessors();
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(cores);      // = CPU cores
        e.setMaxPoolSize(cores);       // Don't exceed!
        e.setQueueCapacity(500);
        e.setThreadNamePrefix("cpu-");
        return e;
    }
    
    // Stage 3: Batch writers
    @Bean("writerExecutor")
    public ThreadPoolTaskExecutor writerExecutor() {
        ThreadPoolTaskExecutor e = new ThreadPoolTaskExecutor();
        e.setCorePoolSize(20);
        e.setMaxPoolSize(20);
        e.setQueueCapacity(200);
        e.setThreadNamePrefix("writer-");
        return e;
    }
}

// Pipeline processing
@Service
public class DataPipeline {
    
    @Async("ioExecutor")
    public CompletableFuture<List<Record>> fetchData(String source) {
        return CompletableFuture.completedFuture(apiClient.fetchBatch(source));
    }
    
    @Async("cpuExecutor")  
    public CompletableFuture<List<Result>> processData(List<Record> records) {
        return CompletableFuture.completedFuture(
            records.stream().map(this::transform).collect(Collectors.toList())
        );
    }
    
    @Async("writerExecutor")
    public CompletableFuture<Void> writeBatch(List<Result> results) {
        jdbcTemplate.batchUpdate(SQL, results);  // Batch write!
        return CompletableFuture.completedFuture(null);
    }
}
```

### 🗣️ How to Explain in Interview

> *"I design concurrent systems as a pipeline with separate stages, each with its own thread pool. The I/O stage has many threads — 50-100 — because they spend most time waiting. The CPU stage has threads equal to CPU cores — more would just waste time on context switching. The write stage uses batch operations for throughput. Bounded queues between stages provide backpressure — if the CPU stage can't keep up, the I/O stage slows down naturally. Each service is stateless — shared state lives in the database. I use ConcurrentHashMap for in-memory caches, LongAdder for counters, and monitor every stage with Micrometer metrics."*

### ⚡ Key Points to Remember

1. **Separate pools** for I/O, CPU, and write stages
2. **Bounded queues** between stages (backpressure) ⭐
3. **Stateless** services — state in DB/cache
4. **Batch I/O** — batch writes, batch API calls
5. **Monitor** every stage (queue size, latency, errors)

---

<a id="q96"></a>

## Q96. How to maintain database consistency with multiple threads?

### 🔑 Quick Answer

> Use **database transactions** (ACID guarantees), **optimistic locking** (@Version — retry on conflict), **pessimistic locking** (SELECT FOR UPDATE — block other threads), **idempotent operations**, and **proper isolation levels**. Choose based on contention level and performance needs.

### 📖 Step-by-Step Explanation

```
Scenario: Two threads update the same account balance

Without protection:
  Thread-1: SELECT balance → 1000
  Thread-2: SELECT balance → 1000
  Thread-1: UPDATE balance = 1000 - 100 = 900 ✅
  Thread-2: UPDATE balance = 1000 - 200 = 800 ← WRONG! Should be 700

Solution 1 — Optimistic Locking:
  Thread-1: SELECT balance, version → (1000, v1)
  Thread-2: SELECT balance, version → (1000, v1)
  Thread-1: UPDATE WHERE version = v1 → SUCCESS (v2)
  Thread-2: UPDATE WHERE version = v1 → FAIL! (version changed)
  Thread-2: RETRY → SELECT (900, v2) → UPDATE (700, v3) ✅

Solution 2 — Pessimistic Locking:
  Thread-1: SELECT FOR UPDATE balance → 1000 (locks row)
  Thread-2: SELECT FOR UPDATE balance → BLOCKED...
  Thread-1: UPDATE balance = 900 → COMMIT → releases lock
  Thread-2: unblocked → SELECT → 900 → UPDATE 700 → COMMIT ✅
```

### 💻 Code Example

```java
// Approach 1: Optimistic Locking with @Version (JPA)
@Entity
public class Account {
    @Id
    private Long id;
    private BigDecimal balance;
    
    @Version  // Automatic optimistic locking!
    private Long version;
}

@Service
public class AccountService {
    
    @Retryable(value = OptimisticLockingFailureException.class, maxAttempts = 3)
    @Transactional
    public void debit(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findById(accountId)
            .orElseThrow(() -> new AccountNotFoundException(accountId));
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
        // If another thread updated between read and write → OptimisticLockException
        // @Retryable automatically retries up to 3 times
    }
}

// Approach 2: Pessimistic Locking (high contention)
public interface AccountRepository extends JpaRepository<Account, Long> {
    
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("SELECT a FROM Account a WHERE a.id = :id")
    Account findByIdForUpdate(@Param("id") Long id);
}

@Service
public class AccountService {
    
    @Transactional
    public void debit(Long accountId, BigDecimal amount) {
        Account account = accountRepository.findByIdForUpdate(accountId);  // Locks row!
        account.setBalance(account.getBalance().subtract(amount));
        accountRepository.save(account);
        // Lock released when transaction commits
    }
}

// Approach 3: Database-level atomic update (simplest!)
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance - :amount WHERE a.id = :id AND a.balance >= :amount")
int debitAtomic(@Param("id") Long id, @Param("amount") BigDecimal amount);
// Returns 0 if insufficient balance, 1 if success — fully atomic!
```

### 🗣️ How to Explain in Interview

> *"For database consistency with concurrent threads, I choose between three approaches. For low contention — like updating different user profiles — optimistic locking with @Version works great. JPA automatically checks the version on save and throws OptimisticLockException if it changed, then @Retryable handles the retry. For high contention — like a popular product inventory — I use pessimistic locking with SELECT FOR UPDATE, which blocks other threads until the transaction commits. For simple increments or decrements, I prefer atomic SQL updates — 'UPDATE SET balance = balance - amount WHERE id = ?' — this is a single atomic database operation, no application-level locking needed."*

### ⚡ Key Points to Remember

1. **Optimistic lock** (@Version) = low contention, retry on conflict
2. **Pessimistic lock** (SELECT FOR UPDATE) = high contention, blocks
3. **Atomic SQL** = simplest for increment/decrement operations ⭐
4. **@Retryable** + optimistic locking = production pattern
5. Choose based on **contention level** and **performance** needs

---

> **🎯 Navigation:** [← Spring Multithreading (Q77-86)](09-spring-multithreading.md) | [📋 All Sections](README.md)
