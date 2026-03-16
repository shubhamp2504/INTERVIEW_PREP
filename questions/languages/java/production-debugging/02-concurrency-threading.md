# 🔥 Production Debugging — Concurrency & Threading (Q6–Q11)

> **"Production just broke. Let's see how you think."**
> Symptoms → Hypothesis → Diagnosis → Fix

---

<a id="q6"></a>
## Q6. A HashMap in your service sometimes loses data unexpectedly. What coding mistake could lead to this?

### 📝 One-Liner
**Using `HashMap` from multiple threads without synchronization** — `HashMap` is NOT thread-safe. Concurrent puts can corrupt the internal array, lose entries, or cause infinite loops (Java 7 treeification in Java 8 eliminated infinite loops but data loss remains).

### 🔑 Quick Answer
`HashMap` has **no internal synchronization**. When two threads call `put()` simultaneously: **(1)** both compute same bucket index → one write overwrites the other → **entry lost**. **(2)** During resize (rehash), concurrent modification can corrupt the linked list/tree structure → **entries silently disappear**. **(3)** One thread iterating while another modifies → `ConcurrentModificationException` (fail-fast iterator). The insidious part: it works fine under low concurrency and only fails under load — making it hard to reproduce in testing. *(HashMap multiple threads se use karte ho toh data gayab ho sakta hai — ConcurrentHashMap use karo)*

### 📖 How to Diagnose
```
Symptoms:
  → Data present in DB but missing from in-memory map
  → Intermittent — works most of the time, fails under load
  → Occasional ConcurrentModificationException in logs
  → Map.size() returns wrong count

Diagnosis:
  1. SEARCH CODEBASE for non-concurrent Map usage with multi-threaded access:
     → static HashMap shared across request threads ❌
     → HashMap field in @Service/@Component (singleton = shared!) ❌
     → HashMap in Runnable/Callable shared between threads ❌

  2. CHECK: Is the map a Spring bean / singleton field?
     Spring beans are SINGLETON by default → ALL request threads share it
     → @Service class with `HashMap<String, Data> cache` ← DANGER

  3. REPRODUCE: Load test with concurrent writes → data loss appears

Why it happens (two threads put to same bucket):
  Thread A: put("key1", val1) → bucket 5 → writing to node...
  Thread B: put("key2", val2) → bucket 5 → overwrites Thread A's node
  → val1 lost forever (no error, no exception, just gone)
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: HashMap in a singleton bean (shared by all request threads)
@Service
public class RateLimiter {
    private final Map<String, Integer> requestCounts = new HashMap<>(); // NOT thread-safe!

    public boolean allowRequest(String clientId) {
        int count = requestCounts.getOrDefault(clientId, 0);
        requestCounts.put(clientId, count + 1); // race condition: check-then-act
        return count < 100;
    }
}

// ✅ FIX: ConcurrentHashMap + atomic operations
@Service
public class RateLimiter {
    private final ConcurrentHashMap<String, AtomicInteger> requestCounts =
        new ConcurrentHashMap<>();

    public boolean allowRequest(String clientId) {
        AtomicInteger count = requestCounts.computeIfAbsent(
            clientId, k -> new AtomicInteger(0));
        return count.incrementAndGet() <= 100; // atomic — no race condition
    }
}

// ❌ BUG: "I'll just synchronize the whole thing" (works but slow)
private final Map<String, Data> cache = Collections.synchronizedMap(new HashMap<>());
// Problem: entire map locked for EVERY read AND write → contention bottleneck

// ✅ FIX: ConcurrentHashMap — concurrent reads, per-bucket writes
private final ConcurrentHashMap<String, Data> cache = new ConcurrentHashMap<>();
```

### ⚠️ Related Pitfalls
- **`computeIfAbsent` on regular HashMap** — also not thread-safe (two threads may both compute)
- **check-then-act** on ConcurrentHashMap is still unsafe: `if (!map.containsKey(k)) map.put(k, v)` → use `putIfAbsent` or `computeIfAbsent`
- **Collections.synchronizedMap** works but locks the entire map → use ConcurrentHashMap for better throughput

### ⚡ Remember
- **HashMap + multiple threads = data loss** (silent, no exception)
- Spring `@Service` is singleton → its fields are shared by ALL threads
- Fix: **ConcurrentHashMap** + atomic operations (`computeIfAbsent`, `merge`)
- Never do **check-then-act** in two separate calls — use single atomic method

### 🔗 Cross-References
- multithreading/05 Q52 → ConcurrentHashMap internals (bucket-level CAS)
- multithreading/02 → Thread safety and race conditions

---

<a id="q7"></a>
## Q7. Your application becomes unstable when too many threads are created dynamically. What is the underlying problem?

### 📝 One-Liner
**Thread leak** — creating `new Thread()` per task without a bounded pool. Each thread consumes ~1MB of native memory (stack) + OS resources → with thousands of threads, you hit OS limits (`OutOfMemoryError: unable to create native thread`) and excessive context switching kills performance.

### 🔑 Quick Answer
**(1) Memory exhaustion**: each thread's stack defaults to ~1MB. 2,000 threads = 2GB just for stacks (separate from heap). **(2) OS thread limit**: Linux default `ulimit -u` is ~30K processes/threads per user. Docker containers may have lower limits. **(3) Context switch overhead**: with 5,000 threads and 8 CPU cores, OS constantly switches threads → CPU spends more time switching than executing. **(4) GC impact**: GC needs to scan all thread stacks as GC roots → more threads = longer GC pauses. **Root cause**: using `new Thread(() -> ...).start()` instead of a thread pool, or using `Executors.newCachedThreadPool()` which creates unlimited threads. *(Har task ke liye new Thread = system unstable — bounded thread pool use karo)*

### 📖 How to Diagnose
```
Symptoms:
  → OutOfMemoryError: unable to create native thread
  → Application becomes sluggish even with low CPU usage
  → OS reports very high thread count: jcmd <pid> Thread.print | grep -c "tid="
  → GC pauses increase

Diagnosis:
  1. CHECK thread count:
     $ jcmd <pid> Thread.print | grep -c "tid="    # total thread count
     → 2000+ threads = likely leak
     
  2. THREAD DUMP — find who's creating threads:
     $ jstack <pid> > dump.txt
     → Search for thread names → pattern like "Thread-1234" = unnamed = raw new Thread()
     → Named threads (pool-1-thread-3) = from Executor

  3. MONITOR over time:
     Metric: jvm_threads_live (Micrometer)
     → If count keeps growing and never shrinks → thread leak
     → If stabilizes at pool max → pool is correctly bounded

  4. GREP CODEBASE for:
     new Thread(                    → should be rare, use pool instead
     Executors.newCachedThreadPool  → unbounded! can create unlimited threads
     .start()                       → direct thread start without pool
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: New thread per request
@PostMapping("/process")
public void process(@RequestBody Request request) {
    new Thread(() -> heavyProcessing(request)).start(); // unbounded thread creation!
    // Under 10K requests → 10K threads → system dies
}

// ❌ BUG: CachedThreadPool = unbounded
ExecutorService executor = Executors.newCachedThreadPool(); // creates threads as needed, no limit!

// ✅ FIX: Bounded thread pool with rejection policy
ExecutorService executor = new ThreadPoolExecutor(
    10,                                    // core pool
    50,                                    // max pool
    60L, TimeUnit.SECONDS,                // idle timeout
    new ArrayBlockingQueue<>(200),         // bounded queue
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure: caller runs if pool full
);

// ✅ FIX: Spring's @Async with configured pool
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("taskExecutor")
    public Executor taskExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(10);
        executor.setMaxPoolSize(50);
        executor.setQueueCapacity(200);
        executor.setThreadNamePrefix("async-task-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class ProcessingService {
    @Async("taskExecutor")
    public CompletableFuture<Result> process(Request request) {
        return CompletableFuture.completedFuture(heavyProcessing(request));
    }
}
```

### ⚡ Remember
- **Never `new Thread()` in production** — always use bounded pools
- `Executors.newCachedThreadPool()` = **unbounded** → avoid *(CachedThreadPool = infinite thread — kabhi use mat karo)*
- Each thread = ~1MB native memory + OS overhead + GC scanning cost
- Monitor `jvm_threads_live` metric — should be stable, not growing
- **CallerRunsPolicy** = natural backpressure when pool is full

### 🔗 Cross-References
- multithreading/04 → ExecutorService, thread pool configuration
- core/01 → JVM Memory (thread stacks, OOM types)

---

<a id="q8"></a>
## Q8. Your system occasionally returns partially updated data during concurrent requests. What concurrency issue might exist?

### 📝 One-Liner
**Race condition with non-atomic compound operations** — read-modify-write or check-then-act across multiple fields/entities without proper synchronization → one thread sees half-written state.

### 🔑 Quick Answer
**(1) Non-atomic compound operation**: Thread A reads balance=100, Thread B reads balance=100, both deduct 50, both write 50 → only one deduction (lost update). **(2) Multi-field update without transaction**: updating `order.status = SHIPPED` and `order.shippingDate = now()` in two separate statements → another thread reads between the two updates → sees SHIPPED + null date (inconsistent). **(3) DB-level**: two requests update the same row without optimistic/pessimistic locking → last-write-wins → first update lost. **(4) In-memory state**: object fields updated without synchronization → happens-before not established → Thread B sees stale values for some fields (memory visibility). *(Partially updated data = ek thread ne aadha likha, doosre ne aadha padha — atomicity missing hai)*

### 📖 How to Diagnose
```
Symptoms:
  → User sees "Order: SHIPPED" but shippingDate is null
  → Account balance incorrect after concurrent transfers
  → Inventory shows wrong count after concurrent purchases
  → Works in single-user testing, fails under load (classic race condition)

Diagnosis:
  1. IDENTIFY the shared mutable state:
     → Database row updated by multiple requests
     → In-memory object modified by multiple threads
     → Multiple fields that must be updated atomically
  
  2. CHECK transaction boundaries:
     → Is @Transactional on the method? If not → operations aren't atomic
     → Is it a READ + WRITE without locking? → TOCTOU (time-of-check-time-of-use)
  
  3. TEST with concurrent requests:
     → Use JMeter/k6: 50 threads updating same entity simultaneously
     → Check: expected vs actual state after all threads complete

  Race Condition (lost update):
    Thread A: READ balance = 100
    Thread B: READ balance = 100    ← reads STALE (A hasn't written yet)
    Thread A: WRITE balance = 50    (deducted 50)
    Thread B: WRITE balance = 50    (deducted 50, overwriting A's write!)
    → Balance should be 0, but is 50 → $50 lost update
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: Non-atomic read-modify-write (database level)
@Transactional
public void deductBalance(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findById(accountId).orElseThrow();
    account.setBalance(account.getBalance().subtract(amount)); // race condition!
    accountRepository.save(account);
}

// ✅ FIX Option 1: Optimistic Locking (@Version)
@Entity
public class Account {
    @Id private Long id;
    @Version private Long version;  // auto-incremented on save
    private BigDecimal balance;
}
// If two threads load same version, second save throws OptimisticLockException

// ✅ FIX Option 2: Pessimistic Locking (SELECT FOR UPDATE)
@Query("SELECT a FROM Account a WHERE a.id = :id")
@Lock(LockModeType.PESSIMISTIC_WRITE)
Account findByIdForUpdate(@Param("id") Long id); // locks row until transaction commits

// ✅ FIX Option 3: Atomic SQL update (best for simple operations)
@Modifying
@Query("UPDATE Account a SET a.balance = a.balance - :amount WHERE a.id = :id AND a.balance >= :amount")
int deductBalance(@Param("id") Long id, @Param("amount") BigDecimal amount);
// Returns 0 if insufficient balance → check and retry

// ❌ BUG: Multi-field in-memory update without synchronization
public class OrderState {
    private String status;
    private Instant lastUpdated;

    public void updateStatus(String newStatus) {     // Thread A and B call simultaneously
        this.status = newStatus;                       // Thread B may see new status...
        this.lastUpdated = Instant.now();             // ...but old lastUpdated (stale)
    }
}

// ✅ FIX: Immutable state update (replace entire object atomically)
public class OrderState {
    private volatile OrderSnapshot snapshot;  // single volatile reference

    public void updateStatus(String newStatus) {
        this.snapshot = new OrderSnapshot(newStatus, Instant.now()); // atomic swap
    }

    record OrderSnapshot(String status, Instant lastUpdated) {}
}
```

### ⚡ Remember
- **Lost update** = two threads read same value, both write → one overwrite lost
- **Fix at DB level**: optimistic locking (`@Version`), pessimistic locking (`FOR UPDATE`), or atomic SQL
- **Fix in-memory**: `volatile` + immutable object swap, or `synchronized`
- **@Transactional alone doesn't prevent race conditions** — it ensures atomicity of YOUR writes, but not isolation from concurrent readers *(Sirf @Transactional se race condition nahi rukta — locking chahiye)*
- Best approach: **atomic SQL UPDATE** (single statement = inherently atomic at DB level)

### 🔗 Cross-References
- multithreading/02 → Race conditions, synchronized, volatile
- multithreading/06 → Java Memory Model (happens-before, visibility)

---

<a id="q9"></a>
## Q9. A synchronized block added for safety starts slowing down the entire application. Why?

### 📝 One-Liner
**Lock contention** — the synchronized block guards a hot code path, causing all threads to serialize (queue up one-by-one), turning a parallel system into a sequential bottleneck.

### 🔑 Quick Answer
`synchronized` acquires an **exclusive lock** — only one thread enters at a time, all others block. If the synchronized section is on a **hot path** (called thousands of times/second), and the **lock scope is too wide** (entire method vs. just the critical section), threads spend most of their time WAITING to acquire the lock instead of doing useful work. **CPU appears low** because threads are parked (not spinning). **Latency increases** linearly with concurrency. The fix is either reduce lock scope, use concurrent data structures, or eliminate the lock entirely. *(Synchronized zyada wide hai toh sab threads line mein lagte hain — ek ek karke jaate hain → slow)*

### 📖 How to Diagnose
```
Symptoms:
  → Latency increases proportionally to concurrent requests
  → CPU utilization is LOW (threads are waiting, not computing)
  → Thread dump shows many threads BLOCKED on same object monitor

Thread dump pattern:
  "http-nio-8080-exec-1" BLOCKED on monitor com.example.Cache@abc123
  "http-nio-8080-exec-2" BLOCKED on monitor com.example.Cache@abc123
  "http-nio-8080-exec-3" BLOCKED on monitor com.example.Cache@abc123
  ... (50+ threads blocked on same lock)
  "http-nio-8080-exec-4" RUNNABLE (the one thread holding the lock)

JFR analysis:
  $ jcmd <pid> JFR.start duration=30s filename=contention.jfr
  → Open in JMC → "Lock Instances" tab
  → Shows which lock, how many threads blocked, total block time
```

### 💻 Code — Fix Strategies
```java
// ❌ PROBLEM: Synchronized entire method (too wide)
@Service
public class ProductService {
    private final Map<Long, Product> cache = new HashMap<>();

    public synchronized Product getProduct(Long id) {    // ENTIRE method locked
        Product product = cache.get(id);                  // fast: just a map read
        if (product == null) {
            product = database.findById(id);              // SLOW: 10ms DB call UNDER LOCK!
            cache.put(id, product);
        }
        return product;
    }
}
// → 200 threads: only 1 in getProduct at a time → serialized

// ✅ FIX 1: Replace with ConcurrentHashMap (no lock needed)
@Service
public class ProductService {
    private final ConcurrentHashMap<Long, Product> cache = new ConcurrentHashMap<>();

    public Product getProduct(Long id) {
        return cache.computeIfAbsent(id, database::findById); // lock-free reads, per-key lock for compute
    }
}

// ✅ FIX 2: ReadWriteLock (many readers, few writers)
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Map<Long, Product> cache = new HashMap<>();

public Product getProduct(Long id) {
    rwLock.readLock().lock();  // MULTIPLE readers concurrently ✅
    try {
        Product product = cache.get(id);
        if (product != null) return product;
    } finally {
        rwLock.readLock().unlock();
    }

    rwLock.writeLock().lock();  // exclusive write only when cache miss
    try {
        return cache.computeIfAbsent(id, database::findById);
    } finally {
        rwLock.writeLock().unlock();
    }
}

// ✅ FIX 3: Minimize lock scope
public Product getProduct(Long id) {
    Product product;
    synchronized (cache) {        // lock only the map access
        product = cache.get(id);
    }
    if (product == null) {
        product = database.findById(id);  // DB call OUTSIDE lock ✅
        synchronized (cache) {
            cache.putIfAbsent(id, product);
        }
    }
    return product;
}
```

### ⚡ Remember
- **Contention** = many threads compete for same lock → serialization *(Lock contention = sab line mein lag gaye — ek ek karke jaate hain)*
- **Diagnosis**: thread dump → BLOCKED on same monitor → JFR lock analysis
- **Fix priority**: (1) `ConcurrentHashMap` → (2) `ReadWriteLock` → (3) minimize lock scope → (4) lock striping
- **Never do I/O under a lock** — DB calls, HTTP calls outside the synchronized block
- CPU LOW + latency HIGH = almost always lock contention

### 🔗 Cross-References
- multithreading/02 → Lock contention, synchronized vs ReentrantLock
- multithreading/05 → ConcurrentHashMap (lock-free alternative)

---

<a id="q10"></a>
## Q10. A thread pool keeps growing but tasks are still delayed. What could be misconfigured?

### 📝 One-Liner
**Unbounded queue + small core pool** — `Executors.newFixedThreadPool(10)` uses `LinkedBlockingQueue` (unlimited) → only 10 threads ever created, tasks queue up infinitely. Or tasks are blocked on I/O, holding threads hostage.

### 🔑 Quick Answer
**ThreadPoolExecutor behavior**: (1) If threads < corePoolSize → create new thread. (2) If threads >= corePoolSize → **put task in queue**. (3) Only if **queue is full** → create thread up to maxPoolSize. (4) If max reached + queue full → reject.

**The trap**: `Executors.newFixedThreadPool(10)` creates `LinkedBlockingQueue` (Integer.MAX_VALUE capacity). Queue NEVER fills → max pool size NEVER reached → always exactly 10 threads. With 500 tasks/sec and each task taking 200ms, those 10 threads can handle ~50 tasks/sec → 450 tasks/sec accumulate in queue → delays grow.

**Other causes**: tasks doing blocking I/O (HTTP calls with no timeout) → threads stuck → fewer available for new tasks. Or tasks throwing exceptions silently (thread dies, pool replaces slowly). *(Queue unlimited hai toh max pool size kabhi trigger nahi hota — bounded queue use karo)*

### 📖 How to Diagnose
```
Symptoms:
  → Task submission succeeds (no RejectedExecutionException)
  → But tasks wait minutes in queue before executing
  → Pool size stays at core, never reaches max

Diagnosis:
  1. CHECK pool configuration:
     → What's the queue type? LinkedBlockingQueue = UNBOUNDED = trap
     → What's core vs max? If different, are you using bounded queue?

  2. MONITOR pool metrics (Micrometer):
     executor_pool_core_threads         = 10
     executor_pool_max_threads          = 50  ← never reached!
     executor_pool_size                 = 10  ← stuck at core
     executor_queue_remaining_capacity  = 2147483637  ← essentially infinite
     executor_queued_task_count         = 5000 ← tasks piling up!
     executor_active_threads            = 10  ← all busy

  3. THREAD DUMP — what are the 10 active threads doing?
     → WAITING on socket read?  → tasks doing blocking I/O with no timeout
     → RUNNABLE?                → tasks are CPU-heavy, need more threads
     → BLOCKED on lock?         → contention, same as Q9

ThreadPoolExecutor decision flow:
  task submitted
       │
       ▼
  threads < corePoolSize? ──YES──→ create thread, run task
       │ NO
       ▼
  queue has space? ──YES──→ add to queue (WAIT)   ← unbounded = always YES
       │ NO                                          → max pool NEVER used!
       ▼
  threads < maxPoolSize? ──YES──→ create thread
       │ NO
       ▼
  REJECT (RejectedExecutionHandler)
```

### 💻 Code — Bug → Fix
```java
// ❌ PROBLEM: Unbounded queue prevents scaling to max
ExecutorService pool = Executors.newFixedThreadPool(10);
// Internally: new ThreadPoolExecutor(10, 10, 0, SECONDS, new LinkedBlockingQueue<>())
// Queue = Integer.MAX_VALUE → max is meaningless → always 10 threads

// ❌ PROBLEM: max=50 but unbounded queue means max is never reached
ExecutorService pool = new ThreadPoolExecutor(
    10, 50,                                // core=10, max=50
    60, TimeUnit.SECONDS,
    new LinkedBlockingQueue<>()            // UNBOUNDED! max=50 is dead code
);

// ✅ FIX: Bounded queue forces pool to scale
ExecutorService pool = new ThreadPoolExecutor(
    10, 50,                                // core=10, max=50 (will actually reach 50!)
    60, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(100),         // BOUNDED queue (100 tasks max)
    new ThreadPoolExecutor.CallerRunsPolicy()  // backpressure when everything full
);

// ✅ FIX: Spring Boot task executor with monitoring
@Bean("appTaskExecutor")
public ThreadPoolTaskExecutor taskExecutor(MeterRegistry registry) {
    ThreadPoolTaskExecutor exec = new ThreadPoolTaskExecutor();
    exec.setCorePoolSize(10);
    exec.setMaxPoolSize(50);
    exec.setQueueCapacity(100);    // ← BOUNDED! max pool size will be used
    exec.setThreadNamePrefix("app-task-");
    exec.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    exec.initialize();
    
    // Monitor pool metrics
    new ExecutorServiceMetrics(exec.getThreadPoolExecutor(), "app-tasks", List.of())
        .bindTo(registry);
    return exec;
}
```

### ⚡ Remember
- **Unbounded queue = max pool size is useless** → always stuck at core threads
- Use `ArrayBlockingQueue(N)` to force pool to scale to max *(Bounded queue lagao — tabhi max pool size kaam karega)*
- `Executors.newFixedThreadPool` and `newSingleThreadExecutor` use unbounded queues → avoid in production
- **CallerRunsPolicy** = natural backpressure (caller thread runs the task itself)
- Monitor: `executor_pool_size`, `executor_queued_task_count`, `executor_active_threads`

### 🔗 Cross-References
- multithreading/04 → ThreadPoolExecutor internals, queue types, rejection policies

---

<a id="q11"></a>
## Q11. A long-running loop accidentally blocks an important worker thread. What kind of issue is this?

### 📝 One-Liner
**Thread starvation** — a CPU-intensive loop monopolizes a thread from a shared pool (Tomcat, event loop, scheduled executor), preventing other tasks that depend on that pool from executing.

### 🔑 Quick Answer
Thread starvation happens when a long/infinite computation hogs a thread from a **shared pool**: **(1) Tomcat thread pool** — request handler runs O(n²) loop on large input → ties up one of 200 Tomcat threads → under load, all 200 can be stuck → no threads for new requests. **(2) Event loop** (WebFlux, Netty) — blocking call or CPU loop on the event-loop thread → ALL requests on that event loop stall (single-threaded reactor pattern). **(3) `@Scheduled` task** — scheduled every 5 min but execution takes 10 min → overlapping runs, pool exhausted. **(4) ForkJoinPool.commonPool** — `parallelStream()` running a slow operation → blocks shared common pool → other `parallelStream`/`CompletableFuture` tasks starve. *(Ek thread pe heavy loop = baaki tasks ka turn nahi aata — thread starvation)*

### 📖 How to Diagnose
```
Symptoms:
  → Some requests hang indefinitely while others work fine
  → Scheduled task stops running on schedule
  → parallelStream operations slow down across unrelated parts of the app

Diagnosis:
  1. THREAD DUMP:
     $ jstack <pid> > dump.txt
     → Find the stuck thread → what code is it executing?
     → RUNNABLE in a loop (CPU work) or WAITING (I/O block)?
  
  2. IDENTIFY the pool:
     → Thread name "http-nio-8080-exec-5" → Tomcat pool
     → Thread name "reactor-http-nio-3"   → WebFlux event loop! (NEVER block this)
     → Thread name "scheduling-1"         → @Scheduled pool (default: 1 thread!)
     → Thread name "ForkJoinPool.commonPool-worker-3" → common pool

  3. @Scheduled starvation (very common!):
     Default pool size = 1 thread!
     If task A takes 10 min, task B (scheduled every 1 min) never runs.

Blocked event loop (worst case):
  WebFlux Event Loop Thread:
     [Request 1: for-loop processing 1M records]  ← STUCK
     [Request 2: waiting....]  [Request 3: waiting....]
     → ALL requests on this event loop frozen
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: Blocking the WebFlux event loop
@GetMapping("/report")
public Mono<Report> generateReport() {
    List<Order> orders = orderRepository.findAll();  // BLOCKING DB CALL on event loop!
    Report report = heavyComputation(orders);         // CPU loop on event loop!
    return Mono.just(report);
}

// ✅ FIX: Offload to bounded scheduler
@GetMapping("/report")
public Mono<Report> generateReport() {
    return Mono.fromCallable(() -> {
        List<Order> orders = orderRepository.findAll();
        return heavyComputation(orders);
    }).subscribeOn(Schedulers.boundedElastic());  // runs on separate pool
}

// ❌ BUG: @Scheduled default pool = 1 thread → starvation
@Scheduled(fixedRate = 60_000) // every 60s
public void taskA() { /* takes 10 minutes */ }

@Scheduled(fixedRate = 5_000)  // every 5s
public void taskB() { /* never runs while taskA is executing! */ }

// ✅ FIX: Configure scheduler pool size
@Configuration
public class SchedulerConfig implements SchedulingConfigurer {
    @Override
    public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
        taskRegistrar.setScheduler(Executors.newScheduledThreadPool(5));
    }
}

// ❌ BUG: parallelStream blocks common pool → starves other parallel operations
orders.parallelStream()
    .map(this::callExternalService)  // HTTP call holds common pool thread!
    .collect(toList());

// ✅ FIX: Use dedicated pool for I/O work
ExecutorService ioPool = Executors.newFixedThreadPool(20);
List<CompletableFuture<Result>> futures = orders.stream()
    .map(order -> CompletableFuture.supplyAsync(
        () -> callExternalService(order), ioPool))
    .toList();
```

### ⚡ Remember
- **Thread starvation** = important pool threads blocked by long-running work
- **WebFlux event loop**: NEVER block → use `subscribeOn(Schedulers.boundedElastic())`
- **@Scheduled default pool = 1 thread** → configure to 5+ *(Scheduled tasks ka default pool 1 thread ka hai — badhao!)*
- **ForkJoinPool.commonPool** = shared → don't use for I/O or slow work
- Always use **dedicated pools** for different workload types (CPU vs I/O)

### 🔗 Cross-References
- multithreading/04 → ScheduledExecutorService, pool sizing
- multithreading/09 → Virtual Threads, reactive programming patterns
