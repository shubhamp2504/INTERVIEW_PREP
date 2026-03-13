# ⚡ Performance & Optimization (Q70–Q76)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q70"></a>

## Q70. How to tune a thread pool?

### 🔑 Quick Answer

> Set pool size based on workload type: **CPU-bound** = number of cores, **I/O-bound** = cores × (1 + wait/compute ratio). Use **bounded queues** to prevent memory issues. Monitor **queue size**, **rejection count**, and **active threads** in production.

### 📖 Step-by-Step Explanation

```
Thread Pool Sizing Formulas:

CPU-bound tasks (computation heavy):
  Threads = Number of CPU cores
  Example: 8 cores → 8 threads
  More threads = context switching waste (CPU already 100%)

I/O-bound tasks (waiting for DB, HTTP, file):
  Threads = Cores × (1 + Wait Time / Compute Time)
  Example: 8 cores, 80ms waiting, 20ms computing
  Threads = 8 × (1 + 80/20) = 8 × 5 = 40 threads

Mixed workloads:
  Separate pools! CPU-bound pool + I/O-bound pool
  Never share a single pool for both → CPU tasks starve
```

**Key parameters:**

```
ThreadPoolExecutor(
    corePoolSize,      // Threads always alive (even if idle)
    maxPoolSize,       // Maximum threads under load
    keepAliveTime,     // How long excess threads wait before dying
    workQueue,         // Queue for pending tasks
    rejectionPolicy    // What to do when queue is full + max threads reached
)

Production settings:
  ┌────────────────┬──────────────────────────────────┐
  │ Parameter      │ Recommendation                   │
  ├────────────────┼──────────────────────────────────┤
  │ corePoolSize   │ Based on formula above            │
  │ maxPoolSize    │ 2-3x core for burst handling      │
  │ Queue          │ ALWAYS bounded (e.g., 1000)       │
  │ Queue type     │ LinkedBlockingQueue or ArrayBQ    │
  │ Rejection      │ CallerRunsPolicy (back-pressure)  │
  │ KeepAlive      │ 60 seconds (for burst threads)    │
  └────────────────┴──────────────────────────────────┘
```

### 💻 Code Example

```java
// Production-grade thread pool configuration
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                                      // core: 10 threads always ready
    50,                                      // max: burst up to 50
    60, TimeUnit.SECONDS,                    // idle threads die after 60s
    new LinkedBlockingQueue<>(1000),         // bounded queue (1000 tasks max)
    new ThreadPoolExecutor.CallerRunsPolicy() // back-pressure: caller runs task
);

// Spring Boot configuration
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(1000);
    executor.setKeepAliveSeconds(60);
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    executor.setThreadNamePrefix("app-worker-");  // Important for debugging!
    executor.initialize();
    return executor;
}
```

### 🗣️ How to Explain in Interview

> *"Thread pool tuning depends on the workload. For CPU-bound tasks — like data transformation or computation — I set the pool size equal to the number of CPU cores, because more threads just add context switching overhead. For I/O-bound tasks — like database calls or HTTP requests — I use the formula: cores times (1 + wait/compute ratio). If a task waits 80ms for a database and computes for 20ms on 8 cores, I'd set 40 threads. The queue must always be bounded — an unbounded queue can cause OutOfMemoryError under load. I use CallerRunsPolicy for rejection — it provides natural back-pressure."*

### ⚡ Key Points to Remember

1. **CPU-bound** = threads ≈ cores
2. **I/O-bound** = cores × (1 + wait/compute)
3. **Always** use bounded queues ⭐
4. **CallerRunsPolicy** = best rejection policy for back-pressure
5. **Monitor**: queue size, active threads, rejection count

---

<a id="q71"></a>

## Q71. CPU-bound vs I/O-bound tasks?

### 🔑 Quick Answer

| Feature | CPU-bound | I/O-bound |
|---------|-----------|-----------|
| **Bottleneck** | CPU computation | Waiting for I/O |
| **Examples** | Sorting, encryption, calculations | DB queries, HTTP calls, file I/O |
| **Optimal threads** | = CPU cores | = cores × (1 + wait/compute) |
| **CPU usage** | ~100% | Low (threads mostly waiting) |
| **More threads help?** | No (just more context switching) | Yes (waiting threads don't use CPU) |

### 📖 Step-by-Step Explanation

```
CPU-bound (4 cores):
  Thread-1: [COMPUTE COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-2: [COMPUTE COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-3: [COMPUTE COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-4: [COMPUTE COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-5: [waiting for CPU...] → WASTED! Just adds context switching

I/O-bound (4 cores):
  Thread-1:  [compute][---WAIT DB---][compute]
  Thread-2:  [compute][---WAIT HTTP---][compute]
  Thread-3:  [compute][---WAIT FILE---][compute]
  ...
  Thread-40: [compute][---WAIT DB---][compute]
  → 40 threads but only 4 CPUs busy at any time
  → While one thread waits, CPU runs another → efficient!
```

### 🗣️ How to Explain in Interview

> *"CPU-bound tasks spend most of their time computing — like encryption, sorting, or data transformation. Adding more threads than CPU cores doesn't help because the CPU is already 100% utilized. I/O-bound tasks spend most time waiting — for database responses, HTTP calls, file reads. Here, adding more threads helps because while one thread waits for I/O, the CPU can work on another thread. In Spring applications, most tasks are I/O-bound — calling databases, REST APIs, messaging. That's why I typically set larger pool sizes for these. I also separate CPU-bound and I/O-bound into different pools to prevent I/O tasks from starving CPU tasks."*

### ⚡ Key Points to Remember

1. **CPU-bound**: computation heavy → threads ≈ cores
2. **I/O-bound**: waiting heavy → many more threads OK
3. **Spring apps** are typically I/O-bound
4. **Separate pools** for CPU vs I/O workloads
5. **Virtual threads** (Java 21) are ideal for I/O-bound tasks

---

<a id="q72"></a>

## Q72. When to use multithreading?

### 🔑 Quick Answer

> Use multithreading when you have **independent tasks that can run in parallel** (I/O-bound work like parallel API calls), **background processing** (async operations), **parallel computation** on large datasets, or **responsive UIs** that need non-blocking operations.

### 📖 Step-by-Step Explanation

```
Good use cases for multithreading:

1. PARALLEL I/O (biggest win):
   Sequential:                   Parallel:
   call API-A → 200ms           call API-A ──╮
   call API-B → 300ms           call API-B ──┼──→ 300ms total!
   call API-C → 150ms           call API-C ──╯
   Total: 650ms                 (60% faster)

2. BACKGROUND PROCESSING:
   User: "Process these 10,000 records"
   → Return "Processing started" immediately
   → Process in background thread pool

3. BATCH/BULK OPERATIONS:
   Process 1M records → split into chunks → process in parallel

4. PRODUCER-CONSUMER:
   Kafka consumer → process messages in parallel → write to DB

5. SCHEDULED TASKS:
   Cleanup jobs, monitoring, heartbeats — all run on separate threads
```

### 🗣️ How to Explain in Interview

> *"I use multithreading primarily for parallel I/O — when I need to call multiple APIs or databases independently, running them in parallel reduces total latency significantly. In a recent project, we called three downstream services sequentially in 650ms — parallelizing brought it down to 300ms. The second major use case is background processing — accepting a request, returning immediately, and processing asynchronously. For batch operations, Spring Batch with multi-threaded steps processes millions of records efficiently. The key is that tasks must be independent — if they share state heavily or have strong ordering requirements, threading adds complexity without benefit."*

### ⚡ Key Points to Remember

1. **Parallel I/O** = biggest practical win
2. **Background processing** = async user experience
3. **Batch operations** = parallel chunk processing
4. Tasks must be **independent** for safe parallelism
5. **Don't use** if tasks are sequential or heavily share state

---

<a id="q73"></a>

## Q73. When to avoid multithreading?

### 🔑 Quick Answer

> Avoid when tasks are **sequential/dependent**, for **simple/fast operations** (threading overhead > benefit), when dealing with **heavy shared state** (excessive locking kills performance), or when **single-thread** solution is simpler and fast enough.

### 📖 Step-by-Step Explanation

```
DON'T use multithreading when:

1. TASKS ARE SEQUENTIAL:
   Step 1 → Step 2 → Step 3 (each depends on previous)
   → Threading adds overhead, no benefit

2. TASK IS FAST:
   Processing 100 records in 5ms
   → Thread creation overhead (~1ms per thread) > benefit
   → Just use a simple loop

3. HEAVY SHARED STATE:
   All threads read/write same data structure
   → Excessive locking → threads mostly waiting for locks
   → Worse than single-threaded! (lock contention + context switching)

4. CORRECTNESS IS CRITICAL:
   Financial calculations, ledger updates
   → Threading bugs are hard to reproduce, hard to debug
   → Single-threaded = deterministic = safe

5. COMPLEXITY BUDGET:
   Small team, tight deadline
   → Concurrent code is 10x harder to debug
   → Ship correct single-threaded code > buggy concurrent code
```

### 🗣️ How to Explain in Interview

> *"I avoid multithreading when the tasks are inherently sequential — if step B depends on step A's result, threading doesn't help. Also when the task is fast — for processing 100 records in 5ms, the overhead of creating threads and context switching exceeds the benefit. Heavy shared state is another red flag — if all threads need to synchronize on the same lock, you get worse performance than single-threaded due to contention. And importantly, if the single-threaded solution is simple and meets performance requirements, I prefer that — concurrent code is significantly harder to debug and test."*

### ⚡ Key Points to Remember

1. Sequential dependencies → **no benefit** from threading
2. Fast tasks → threading **overhead exceeds benefit**
3. Heavy shared state → **lock contention kills performance**
4. Single-threaded first → add threading only when **proven necessary**
5. Concurrent bugs are **10x harder** to find and fix

---

<a id="q74"></a>

## Q74. How to improve multithreading performance?

### 🔑 Quick Answer

> **Reduce lock contention** (minimize synchronized scope, use lock-free data structures), **right-size thread pools**, **minimize shared state**, use **work-stealing** (ForkJoinPool), and **batch I/O operations** to reduce context switching.

### 📖 Step-by-Step Explanation

```
Top performance improvements:

1. REDUCE LOCK SCOPE:
   ❌ synchronized(lock) {          ✅ prepareData();
        prepareData();                  synchronized(lock) {
        updateSharedState();                updateSharedState();
        log();                          }
      }                                 log();
   // Locks for too long              // Lock only what's needed

2. LOCK-FREE DATA STRUCTURES:
   ❌ synchronized + HashMap         ✅ ConcurrentHashMap
   ❌ synchronized + counter         ✅ AtomicInteger / LongAdder

3. MINIMIZE SHARED STATE:
   ❌ Shared mutable object          ✅ Thread-local copies
   ❌ Global counter                 ✅ Per-thread counters, merge at end

4. BATCHING:
   ❌ DB insert per record           ✅ Batch insert 1000 records
   ❌ HTTP call per item             ✅ Batch API with list

5. THREAD-LOCAL STORAGE:
   ❌ Shared SimpleDateFormat        ✅ ThreadLocal<SimpleDateFormat>
   (or just use DateTimeFormatter which is thread-safe)
```

### 💻 Code Example

```java
// LongAdder for high-contention counters (Java 8+)
// Much faster than AtomicLong under heavy write contention
LongAdder counter = new LongAdder();

// Multiple threads increment concurrently
counter.increment();  // No CAS retry loop → minimal contention

// Read the total (less frequent)
long total = counter.sum();

// ThreadLocal for non-thread-safe objects
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// Each thread gets its own instance — no synchronization needed
String date = dateFormat.get().format(new Date());
```

### 🗣️ How to Explain in Interview

> *"The biggest performance gain is reducing lock contention. First, minimize synchronized scope — lock only the shared state mutation, not the preparation or logging. Second, use lock-free structures — ConcurrentHashMap over synchronized HashMap, LongAdder over AtomicLong for high-contention counters. Third, minimize shared state — use thread-local copies where possible. Fourth, batch operations — instead of one database insert per record, batch 1000 at a time. And right-size the pool — too few threads underutilize resources, too many cause excessive context switching."*

### ⚡ Key Points to Remember

1. **Reduce lock scope** = lock only what's necessary
2. **Lock-free** = ConcurrentHashMap, LongAdder, AtomicXxx
3. **ThreadLocal** = per-thread copies, no sharing
4. **Batch I/O** = fewer calls, less context switching
5. **LongAdder > AtomicLong** for high write contention

---

<a id="q75"></a>

## Q75. What is false sharing?

### 🔑 Quick Answer

> False sharing occurs when **two threads modify different variables** that happen to be on the **same CPU cache line** (typically 64 bytes). Each write invalidates the entire cache line for the other core, causing **severe performance degradation** even though the variables are logically independent.

### 📖 Step-by-Step Explanation

```
CPU Cache Line (64 bytes):
  ┌──────────────────────────────────────────────────┐
  │ counter1 (8 bytes) │ counter2 (8 bytes) │ ...    │
  └──────────────────────────────────────────────────┘
           ↑                      ↑
        Thread-1               Thread-2
        (Core 1)               (Core 2)

Thread-1 writes counter1 → invalidates ENTIRE cache line on Core 2
Thread-2 writes counter2 → invalidates ENTIRE cache line on Core 1
→ Constant cache invalidation → "ping-pong" between cores
→ Performance can be 10-50x WORSE than expected!

The variables are independent — they don't need synchronization!
But the hardware treats the whole cache line as one unit.
```

**Fix with padding:**

```java
// BEFORE: False sharing
class Counters {
    volatile long counter1;  // Same cache line!
    volatile long counter2;  // Same cache line!
}

// AFTER: Padding to separate cache lines
class Counters {
    volatile long counter1;
    long p1, p2, p3, p4, p5, p6, p7;  // 56 bytes padding
    volatile long counter2;  // Now on different cache line!
}

// Java 8+: @Contended annotation (JVM handles padding)
class Counters {
    @jdk.internal.vm.annotation.Contended
    volatile long counter1;
    @jdk.internal.vm.annotation.Contended
    volatile long counter2;
}
```

### 🗣️ How to Explain in Interview

> *"False sharing is a hardware-level performance problem. Modern CPUs transfer data in cache lines — typically 64 bytes. If two threads on different cores modify different variables that happen to be in the same cache line, each write invalidates the entire cache line for the other core. This causes constant cache-line bouncing between cores — even though the variables are logically independent. I've seen this cause 10-50x slowdowns in tight loops. The fix is padding — add dummy fields between the hot variables so they land on different cache lines. Java 8 added the @Contended annotation that handles this automatically. LongAdder and Striped64 internally use this technique."*

### ⚡ Key Points to Remember

1. **Cache line** = 64 bytes, transferred as a unit
2. Different variables, **same cache line** = false sharing
3. Causes **cache-line bouncing** between cores (ping-pong)
4. Fix: **padding** or **@Contended** annotation
5. **LongAdder** uses this internally (Striped64)

---

<a id="q76"></a>

## Q76. What is lock striping?

### 🔑 Quick Answer

> Lock striping divides a data structure into **segments**, each with its **own lock**. Instead of one lock for the entire structure, threads only contend on the lock for their specific segment. **ConcurrentHashMap** uses this concept.

### 📖 Step-by-Step Explanation

```
Single lock (poor concurrency):
  HashMap + 1 synchronized lock
  ┌────────────────────────────┐
  │ [bucket0][bucket1]...[bucketN] │
  │        ALL locked by ONE lock  │
  └────────────────────────────┘
  Thread-1: put(key1) → LOCK ALL → work → UNLOCK
  Thread-2: put(key2) → WAIT...

Lock striping (high concurrency):
  ConcurrentHashMap-style with segment locks
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ Segment 0 │ │ Segment 1 │ │ Segment 2 │
  │ Lock-0    │ │ Lock-1    │ │ Lock-2    │
  │[b0][b1]   │ │[b2][b3]   │ │[b4][b5]   │
  └───────────┘ └───────────┘ └───────────┘
  Thread-1: put(key1) → Lock-0 only
  Thread-2: put(key2) → Lock-2 only → CONCURRENT! ✅
```

### 💻 Code Example

```java
// Custom lock striping example
public class StripedMap<K, V> {
    private static final int NUM_STRIPES = 16;
    private final Object[] locks;
    private final Map<K, V>[] buckets;

    @SuppressWarnings("unchecked")
    public StripedMap() {
        locks = new Object[NUM_STRIPES];
        buckets = new HashMap[NUM_STRIPES];
        for (int i = 0; i < NUM_STRIPES; i++) {
            locks[i] = new Object();
            buckets[i] = new HashMap<>();
        }
    }

    private int stripeFor(K key) {
        return Math.abs(key.hashCode() % NUM_STRIPES);
    }

    public V get(K key) {
        int stripe = stripeFor(key);
        synchronized (locks[stripe]) {        // Lock only this stripe
            return buckets[stripe].get(key);
        }
    }

    public void put(K key, V value) {
        int stripe = stripeFor(key);
        synchronized (locks[stripe]) {        // Lock only this stripe
            buckets[stripe].put(key, value);
        }
    }
}
```

### 🗣️ How to Explain in Interview

> *"Lock striping is the technique of splitting a single lock into multiple locks — one per segment of the data structure. Instead of all threads contending on one lock, threads only contend on the lock for their specific segment. ConcurrentHashMap in Java 7 used 16 segments, each with its own lock — so up to 16 threads could write simultaneously. In Java 8+, it went further — using per-bucket CAS and synchronized on the first node. Lock striping is the generalization of this idea — I apply it whenever I have a large data structure with independent regions that different threads typically access."*

### ⚡ Key Points to Remember

1. **One big lock** → **many small locks** (per segment)
2. **ConcurrentHashMap** = the classic example
3. Threads on **different stripes** don't contend
4. **More stripes** = less contention = higher throughput
5. Trade-off: more locks = more memory, harder global operations

---

> **🎯 Navigation:** [← Deadlock & Problems (Q63-69)](07-deadlock-problems.md) | [Next → Spring Multithreading (Q77-86)](09-spring-multithreading.md) | [📋 All Sections](README.md)
