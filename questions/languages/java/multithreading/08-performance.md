# ⚡ Performance & Optimization (Q70–Q76)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q70"></a>
## Q70. How to tune a thread pool?

### 📝 One-Liner
> CPU-bound = threads ≈ cores; I/O-bound = cores × (1 + wait/compute ratio); always bounded queues; always CallerRunsPolicy.

### 🔑 Quick Answer
> Set pool size by workload: **CPU-bound** = number of CPU cores, **I/O-bound** = cores × (1 + wait/compute ratio). Always use **bounded queues** and **CallerRunsPolicy** for rejection. Monitor queue size and active threads. *(CPU kaam = core jitne thread; I/O kaam = zyada threads kyunki wait karte hain)*

### 📖 How It Works
```
Thread Pool Sizing:

CPU-bound (computation heavy):
  Threads = CPU cores
  8 cores → 8 threads (more = just context switching waste)
  *(CPU 100% busy — zyada threads se context switch ka loss)*

I/O-bound (DB calls, HTTP, file I/O):
  Threads = Cores × (1 + Wait Time / Compute Time)
  8 cores, 80ms wait, 20ms compute
  = 8 × (1 + 80/20) = 8 × 5 = 40 threads
  *(Wait karte hain — idle CPU ko doosra thread use kare)*

Key parameters:
  ThreadPoolExecutor(
    corePoolSize,    // always alive (even idle)
    maxPoolSize,     // max under burst load
    keepAliveTime,   // idle thread dies after this
    workQueue,       // ALWAYS bounded!
    rejectionPolicy  // CallerRunsPolicy = backpressure ⭐
  )
```

### 💻 Code
```java
// Production-grade configuration
ThreadPoolExecutor executor = new ThreadPoolExecutor(
    10,                                      // core: 10 always ready
    50,                                      // max: burst to 50
    60, TimeUnit.SECONDS,                    // idle die after 60s
    new LinkedBlockingQueue<>(1000),         // bounded queue!
    new ThreadPoolExecutor.CallerRunsPolicy() // backpressure ⭐
);

// Spring Boot
@Bean
public ThreadPoolTaskExecutor taskExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(50);
    executor.setQueueCapacity(1000);
    executor.setThreadNamePrefix("app-worker-");  // debugging!
    executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
    return executor;
}
```

### 🗣️ How to Say in Interview
> *"Thread pool tuning depends on workload type. CPU-bound — I match pool size to CPU cores because more threads just add context switching waste. I/O-bound — I use the formula cores × (1 + wait/compute ratio). The queue must always be bounded — an unbounded queue causes OOM under load. I use CallerRunsPolicy for rejection — it provides natural backpressure by making the submitting thread run the task itself."*

### ⚠️ Pitfalls / Gotchas
- **Unbounded queue** → OOM under load *(queue bharti jaayegi — memory khatam)*
- **Too many threads** for CPU-bound = performance drops (context switching) *(CPU kaam mein zyada threads = slow)*
- **Too few threads** for I/O-bound = CPU sitting idle *(I/O kaam mein kam threads = CPU khaali baitha)*

### ⚡ Remember
1. **CPU-bound** = threads ≈ cores *(zyada se context switch)*
2. **I/O-bound** = cores × (1 + wait/compute) *(wait mein doosra chale)*
3. **Always bounded queue** ⭐
4. **CallerRunsPolicy** = best rejection (backpressure)
5. **Monitor**: queue size, active threads, rejections

### 🔗 Follow-ups
→ [Q71. CPU vs I/O bound](#q71)

---

<a id="q71"></a>
## Q71. CPU-bound vs I/O-bound tasks?

### 📝 One-Liner
> CPU-bound = bottleneck is computation (threads = cores); I/O-bound = bottleneck is waiting (many more threads OK).

### 🆚 vs. Comparison
| Feature | CPU-bound | I/O-bound |
|---------|-----------|-----------|
| **Bottleneck** | CPU computation | Waiting for I/O |
| **Examples** | Sorting, encryption | DB, HTTP, file |
| **Optimal threads** | = CPU cores | cores × (1 + W/C) |
| **CPU usage** | ~100% | Low (mostly waiting) |
| **More threads help?** | No ❌ | Yes ✅ |

### 📖 How It Works
```
CPU-bound (4 cores):
  Thread-1: [COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-2: [COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-3: [COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-4: [COMPUTE COMPUTE COMPUTE] → CPU 100%
  Thread-5: [waiting for CPU...] → WASTE! Just context switching
  *(CPU already 100% busy — naya thread kya karega)*

I/O-bound (4 cores):
  Thread-1:  [compute][---WAIT DB---][compute]
  Thread-2:  [compute][---WAIT HTTP---][compute]
  Thread-40: [compute][---WAIT DB---][compute]
  → 40 threads but CPU only 4 busy at a time
  → While one waits, CPU runs another ✅
  *(Wait mein hai toh CPU doosre ko de do)*
```

### 🗣️ How to Say in Interview
> *"CPU-bound tasks are computation heavy — sorting, encryption, transformation. Adding more threads than cores just adds context switching overhead. I/O-bound tasks spend most time waiting — for DB, HTTP, file I/O. Here more threads help because idle threads don't use CPU. Spring apps are typically I/O-bound — calling databases and REST APIs. I always separate CPU and I/O workloads into different pools to prevent starvation."*

### ⚡ Remember
1. **CPU-bound**: computation heavy → threads ≈ cores
2. **I/O-bound**: waiting heavy → many more threads OK
3. **Spring apps** are typically I/O-bound
4. **Separate pools** for CPU vs I/O workloads ⭐
5. **Virtual threads** (Java 21) = ideal for I/O-bound

### 🔗 Follow-ups
→ [Q72. When to use multithreading](#q72)

---

<a id="q72"></a>
## Q72. When to use multithreading?

### 📝 One-Liner
> Parallel I/O (biggest win), background processing, batch/bulk operations, producer-consumer pipelines.

### 🔑 Quick Answer
> Best for: **parallel I/O calls** (call 3 APIs in parallel = 3x faster), **background processing** (return immediately, process later), **batch operations** (split 1M records across threads), **producer-consumer** (Kafka consumer → process in parallel). Tasks must be **independent**. *(Alag alag kaam ek saath karo — time bachega)*

### 📖 How It Works
```
Biggest win — Parallel I/O:
  Sequential:               Parallel:
  API-A → 200ms            API-A ──╮
  API-B → 300ms            API-B ──┼──→ 300ms total!
  API-C → 150ms            API-C ──╯
  Total: 650ms             (60% faster)
  *(Teeno API ek saath call karo — sabse slow API jitna time lagega)*
```

### 🗣️ How to Say in Interview
> *"I use multithreading primarily for parallel I/O — calling multiple APIs or databases simultaneously cuts total latency. In a project, parallelizing three downstream service calls reduced latency from 650ms to 300ms. Second use is background processing — accept a request, return immediately, process async. For batch operations, Spring Batch with partitioned steps processes millions of records efficiently."*

### ⚡ Remember
1. **Parallel I/O** = biggest practical win ⭐
2. **Background processing** = async UX
3. **Batch operations** = parallel chunks
4. Tasks must be **independent**
5. **Don't use** if tasks are sequential or heavily shared

### 🔗 Follow-ups
→ [Q73. When to avoid](#q73)

---

<a id="q73"></a>
## Q73. When to avoid multithreading?

### 📝 One-Liner
> Sequential dependencies, fast tasks (overhead > benefit), heavy shared state (lock contention), simple apps where single-thread suffices.

### 🔑 Quick Answer
> Avoid when: tasks are **sequential** (step B needs A's result), task is **fast** (threading overhead > benefit), **heavy shared state** (all threads contend on same lock → worse than single thread), or single-threaded solution **meets performance needs**. *(Agar kaam ek ke baad ek karna hai — parallel karne se koi fayda nahi)*

### ⚠️ Pitfalls / Gotchas
- Thread creation ~1ms overhead — for 5ms task, overhead = 20% *(chhota kaam hai toh thread banana zyada mehnga)*
- Heavy lock contention = **worse** than single thread *(sab threads lock ka wait kar rahe — ek se bhi slow)*
- Concurrent bugs are **10x harder to debug** *(multi-thread bug dhundhna bahut mushkil)*

### 🗣️ How to Say in Interview
> *"I avoid multithreading when tasks are inherently sequential. Also when the task is fast — threading overhead exceeds the benefit. Heavy shared state is a red flag — if all threads synchronize on the same lock, performance is worse than single-threaded due to contention. Concurrent code is 10x harder to debug, so if single-threaded meets performance requirements, I prefer that."*

### ⚡ Remember
1. Sequential dependencies → **no benefit**
2. Fast tasks → **overhead > benefit**
3. Heavy shared state → **contention kills perf**
4. **Single-threaded first** → add threads only when proven necessary
5. Concurrent bugs = **10x harder** to find

### 🔗 Follow-ups
→ [Q74. Improve performance](#q74)

---

<a id="q74"></a>
## Q74. How to improve multithreading performance?

### 📝 One-Liner
> Reduce lock scope, use lock-free structures, minimize shared state, batch I/O, use ThreadLocal, use LongAdder over AtomicLong.

### 🔑 Quick Answer
> **#1: Reduce lock scope** — lock only what's necessary. **#2: Lock-free structures** — ConcurrentHashMap, LongAdder. **#3: Minimize sharing** — ThreadLocal copies. **#4: Batch I/O** — fewer calls, less context switching. *(Lock chhota karo, share kam karo, batch mein karo)*

### 📖 How It Works
```
1. REDUCE LOCK SCOPE:
  ❌ synchronized(lock) { prepare(); update(); log(); }  // too long!
  ✅ prepare(); synchronized(lock) { update(); } log();  // lock only update

2. LOCK-FREE:
  ❌ synchronized + HashMap      ✅ ConcurrentHashMap
  ❌ synchronized + counter      ✅ LongAdder (high contention)

3. MINIMIZE SHARING:
  ❌ Shared SimpleDateFormat     ✅ ThreadLocal<DateFormat>
  ❌ Global counter              ✅ Per-thread counters, merge at end

4. BATCH I/O:
  ❌ DB insert per record        ✅ Batch insert 1000 records
  *(Kam lock, kam sharing, batch mein kaam — fast)*
```

### 💻 Code
```java
// LongAdder — much faster than AtomicLong under contention
LongAdder counter = new LongAdder();
counter.increment();     // minimal contention (per-CPU cell)
long total = counter.sum();  // aggregate

// ThreadLocal — per-thread copy, no sync needed
private static final ThreadLocal<SimpleDateFormat> df =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
String date = df.get().format(new Date());  // each thread gets own instance
```

### 🗣️ How to Say in Interview
> *"Biggest gain is reducing lock contention. Minimize synchronized scope — lock only the mutation, not preparation or logging. Use lock-free structures — ConcurrentHashMap over synchronized HashMap, LongAdder over AtomicLong for high-contention counters. ThreadLocal for per-thread non-thread-safe objects. And batch I/O — instead of per-record DB inserts, batch 1000 at a time."*

### 🎯 Tricky Interview Qs
**Q: LongAdder vs AtomicLong — when to use which?**
> LongAdder: high-contention writes, occasional reads (counters). AtomicLong: low contention or need exact real-time value. *(Bahut threads likh rahe = LongAdder; kam threads = AtomicLong)*

### ⚡ Remember
1. **Reduce lock scope** = lock only what's needed
2. **Lock-free** = ConcurrentHashMap, LongAdder ⭐
3. **ThreadLocal** = per-thread copies, no sharing
4. **Batch I/O** = fewer calls, better throughput
5. **LongAdder > AtomicLong** for high write contention

### 🔗 Follow-ups
→ [Q75. False sharing](#q75)

---

<a id="q75"></a>
## Q75. What is false sharing?

### 📝 One-Liner
> Two threads modify different variables on the same CPU cache line — causes constant cache invalidation, 10-50x slower.

### 🔑 Quick Answer
> CPU cache works in **64-byte cache lines**. If two variables sit on the **same cache line**, modifying one **invalidates the entire line** for the other core — ping-pong effect. Even though variables are **logically independent**, they share hardware cache sab, causing **10-50x slowdown**. *(Do alag variables ek hi cache line mein — ek badlo toh doosre ka bhi cache kharab)*

### 📖 How It Works
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
→ Constant ping-pong between cores → 10-50x SLOWER! 💀
*(Variables alag hain par cache line ek hai — hardware problem)*

Fix with padding:
  volatile long counter1;
  long p1, p2, p3, p4, p5, p6, p7;  // 56 bytes padding
  volatile long counter2;  // now on different cache line! ✅

Java 8+: @Contended annotation (JVM handles padding)
```

### 🗣️ How to Say in Interview
> *"False sharing is a hardware-level problem. CPUs transfer data in 64-byte cache lines. If two threads on different cores modify different variables that happen to be on the same cache line, each write invalidates the entire line for the other core. The fix is padding — add dummy fields so they land on different cache lines. Java 8's @Contended annotation handles this automatically. LongAdder internally uses this technique in Striped64."*

### ⚡ Remember
1. **Cache line** = 64 bytes, unit of transfer
2. **Same cache line** = false sharing → ping-pong *(slow)*
3. Fix: **padding** or **@Contended** annotation
4. Can cause **10-50x** slowdown
5. **LongAdder** uses this internally (Striped64)

### 🔗 Follow-ups
→ [Q76. Lock striping](#q76)

---

<a id="q76"></a>
## Q76. What is lock striping?

### 📝 One-Liner
> Split one big lock into many small locks per segment — threads on different segments don't contend.

### 🔑 Quick Answer
> Instead of **one lock** for the entire data structure, split into **N segment locks**. Threads touching different segments run **concurrently**. **ConcurrentHashMap** is the classic example (Java 7: 16 segments, Java 8: per-bucket). *(Ek bada taala todke bahut chhote taale bana do — alag segment pe alag thread chale)*

### 📖 How It Works
```
Single lock (poor concurrency):
  ┌──────────────────────────────┐
  │ [bucket0][bucket1]...[bucketN] │
  │     ALL locked by ONE lock     │
  └──────────────────────────────┘
  Thread-1: put(key1) → LOCK ALL → unlock
  Thread-2: put(key2) → WAIT...

Lock striping (high concurrency):
  ┌───────────┐ ┌───────────┐ ┌───────────┐
  │ Segment 0 │ │ Segment 1 │ │ Segment 2 │
  │ Lock-0    │ │ Lock-1    │ │ Lock-2    │
  └───────────┘ └───────────┘ └───────────┘
  Thread-1: put(key1) → Lock-0 only
  Thread-2: put(key2) → Lock-2 only → CONCURRENT! ✅
  *(Alag segment — alag lock — parallel chalega)*
```

### 💻 Code
```java
// Custom lock striping
public class StripedMap<K, V> {
    private static final int STRIPES = 16;
    private final Object[] locks = new Object[STRIPES];
    private final Map<K, V>[] buckets = new HashMap[STRIPES];

    public StripedMap() {
        for (int i = 0; i < STRIPES; i++) {
            locks[i] = new Object();
            buckets[i] = new HashMap<>();
        }
    }

    public V get(K key) {
        int stripe = Math.abs(key.hashCode() % STRIPES);
        synchronized (locks[stripe]) { return buckets[stripe].get(key); }
    }

    public void put(K key, V value) {
        int stripe = Math.abs(key.hashCode() % STRIPES);
        synchronized (locks[stripe]) { buckets[stripe].put(key, value); }
    }
}
```

### 🗣️ How to Say in Interview
> *"Lock striping splits a single lock into multiple segment locks. ConcurrentHashMap in Java 7 used 16 segments — up to 16 threads could write simultaneously. Java 8 went further with per-bucket CAS and synchronized on the first node. I apply this concept whenever I have a large data structure with independent regions."*

### ⚡ Remember
1. **One big lock → many small locks** *(ek taala → bahut taale)*
2. **ConcurrentHashMap** = classic example
3. Different stripes → **no contention** → parallel
4. **More stripes** = less contention, more throughput
5. Trade-off: more locks = more memory

### 🔗 Follow-ups
→ [Q52. ConcurrentHashMap](05-concurrent-collections.md#q52)

---

> **🎯 Navigation:** [← Deadlock & Problems (Q63-69)](07-deadlock-problems.md) | [Next → Spring Multithreading (Q77-86)](09-spring-multithreading.md) | [📋 All Sections](README.md)
