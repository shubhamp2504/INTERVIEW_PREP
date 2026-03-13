# 🔒 Deadlock & Concurrency Problems (Q63–Q69)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q63"></a>

## Q63. What is deadlock?

### 🔑 Quick Answer

> Deadlock occurs when **two or more threads are permanently blocked**, each waiting for a lock held by the other. Neither can proceed — the application hangs forever.

### 📖 Step-by-Step Explanation

```
Classic deadlock scenario:

  Thread-1:                         Thread-2:
  lock(A) ✅                        lock(B) ✅
  lock(B) → WAITING...             lock(A) → WAITING...
         ↑                                ↑
         └────────────── DEADLOCK ─────────┘
         
  Thread-1 holds A, wants B
  Thread-2 holds B, wants A
  → Neither can proceed → PERMANENT HANG
```

### 💻 Code Example

```java
// DEADLOCK example
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {             // Step 1: Lock A
        Thread.sleep(100);             // Simulate work
        synchronized (lockB) {         // Step 2: Try Lock B → BLOCKED!
            System.out.println("T1");
        }
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {             // Step 1: Lock B
        Thread.sleep(100);             // Simulate work
        synchronized (lockA) {         // Step 2: Try Lock A → BLOCKED!
            System.out.println("T2");
        }
    }
});

t1.start(); t2.start();
// BOTH threads hang forever → DEADLOCK
```

### 🗣️ How to Explain in Interview

> *"Deadlock is when two or more threads are permanently waiting for each other. Thread-1 holds Lock A and waits for Lock B, while Thread-2 holds Lock B and waits for Lock A. Neither can proceed. In a real production scenario, I saw this happen with database connections — two transactions each held a row lock and tried to acquire the other's row lock. The application hung with no errors, no logs — just silence. Deadlocks are particularly dangerous because there's no timeout by default with synchronized."*

### ⚡ Key Points to Remember

1. **Two+ threads** waiting for each other's locks
2. **Permanent** — won't resolve on its own
3. **No error thrown** — application silently hangs
4. Most common: **nested locks in different order**
5. In production: appears as **hung threads, no CPU usage**

---

<a id="q64"></a>

## Q64. What are the four conditions for deadlock?

### 🔑 Quick Answer

> All four must be true simultaneously for deadlock (**Coffman conditions**):
> 1. **Mutual Exclusion** — resource held exclusively
> 2. **Hold and Wait** — holding one resource, waiting for another
> 3. **No Preemption** — can't force release of a lock
> 4. **Circular Wait** — circular chain of threads waiting

### 📖 Step-by-Step Explanation

```
All 4 conditions (ALL must be true for deadlock):

1. MUTUAL EXCLUSION:
   Lock A can only be held by ONE thread at a time ✅

2. HOLD AND WAIT:
   Thread-1 HOLDS Lock A while WAITING for Lock B ✅

3. NO PREEMPTION:
   Nobody can FORCE Thread-1 to release Lock A ✅

4. CIRCULAR WAIT:
   Thread-1 → waits for → Thread-2 → waits for → Thread-1 ✅
   (circular dependency)

Break ANY ONE condition → Deadlock IMPOSSIBLE!
```

| Condition | How to Break It |
|-----------|----------------|
| Mutual Exclusion | Use concurrent collections (shared access) |
| Hold and Wait | Lock ALL resources at once (or none) |
| No Preemption | Use tryLock() with timeout |
| **Circular Wait** | **Lock ordering** (always lock in same order) ⭐ |

### 🗣️ How to Explain in Interview

> *"Deadlock requires four conditions simultaneously — the Coffman conditions. Mutual exclusion: the resource is exclusive. Hold and wait: a thread holds one lock while waiting for another. No preemption: you can't force a thread to release its lock. Circular wait: there's a cycle in the wait graph. To prevent deadlock, break any one condition. The most practical approach is breaking circular wait by enforcing a global lock ordering — always acquire Lock A before Lock B, regardless of which thread. Another approach is using tryLock with a timeout to break hold-and-wait."*

### ⚡ Key Points to Remember

1. **All 4** conditions must hold simultaneously
2. Break **any one** → deadlock impossible
3. **Lock ordering** = most practical prevention ⭐
4. **tryLock(timeout)** = breaks hold-and-wait
5. Interview tip: know the names + how to break each

---

<a id="q65"></a>

## Q65. How to prevent deadlock?

### 🔑 Quick Answer

> **Lock ordering** (always acquire locks in the same global order), **tryLock with timeout** (back off if can't acquire), **lock-free algorithms** (CAS), and **reduce lock scope** (hold locks for shortest time).

### 📖 Step-by-Step Explanation

```
Strategy 1: LOCK ORDERING (best practice) ⭐
  BEFORE (deadlock-prone):
    Thread-1: lock(A) → lock(B)
    Thread-2: lock(B) → lock(A)  ← different order!

  AFTER (deadlock-free):
    Thread-1: lock(A) → lock(B)
    Thread-2: lock(A) → lock(B)  ← same order! ✅

Strategy 2: tryLock WITH TIMEOUT
  Thread-1: 
    lock(A);
    if (!lockB.tryLock(1, SECONDS)) {
      lockA.unlock();  // release and retry
    }
  → Breaks "hold and wait" condition ✅

Strategy 3: LOCK-FREE ALGORITHMS
  Use AtomicInteger, ConcurrentHashMap, etc.
  → No locks = no deadlock ✅

Strategy 4: SINGLE LOCK
  Instead of multiple locks, use ONE lock for all resources
  → No circular wait ✅ (but lower concurrency)
```

### 💻 Code Example

```java
// Fix 1: Lock ordering — always lock in consistent order
private void transfer(Account from, Account to, int amount) {
    // Determine lock order by account ID (consistent global order)
    Account first = from.id < to.id ? from : to;
    Account second = from.id < to.id ? to : from;
    
    synchronized (first) {
        synchronized (second) {
            from.balance -= amount;
            to.balance += amount;
        }
    }
}

// Fix 2: tryLock with timeout
private void transferSafe(Account from, Account to, int amount) {
    while (true) {
        if (from.lock.tryLock(1, TimeUnit.SECONDS)) {
            try {
                if (to.lock.tryLock(1, TimeUnit.SECONDS)) {
                    try {
                        from.balance -= amount;
                        to.balance += amount;
                        return;
                    } finally { to.lock.unlock(); }
                }
            } finally { from.lock.unlock(); }
        }
        Thread.sleep(random.nextInt(100));  // Back off and retry
    }
}
```

### 🗣️ How to Explain in Interview

> *"My go-to strategy for preventing deadlock is lock ordering — impose a global order on all locks, like by object ID or hash code, and always acquire them in that order. For the classic bank transfer problem, I lock the account with the lower ID first — this eliminates circular wait. When I can't control the lock order, I use ReentrantLock.tryLock() with a timeout — if I can't get the second lock, I release the first and retry with a random backoff. Beyond locks, I prefer lock-free alternatives like ConcurrentHashMap and AtomicInteger whenever possible — no locks means no deadlock."*

### ⚡ Key Points to Remember

1. **Lock ordering** = #1 strategy (consistent global order)
2. **tryLock(timeout)** = back off if can't acquire
3. **Lock-free** data structures eliminate deadlock risk
4. Minimize **lock scope** (hold lock for shortest time)
5. **Never** call unknown/external code while holding a lock

---

<a id="q66"></a>

## Q66. How to detect deadlock?

### 🔑 Quick Answer

> Use **thread dumps** (jstack, kill -3), **JConsole/VisualVM** (automatic detection), **ThreadMXBean** (programmatic detection), or **deadlock detection in logging**. Thread dumps are the most common production technique.

### 📖 Step-by-Step Explanation

```
Method 1: THREAD DUMP (most common in production)
  $ jstack <pid>
  
  Output:
  Found one Java-level deadlock:
  =============================
  "Thread-1":
    waiting to lock 0x000000076ab34c70 (Object B)
    which is held by "Thread-2"
  "Thread-2":
    waiting to lock 0x000000076ab34c60 (Object A)
    which is held by "Thread-1"

Method 2: JConsole / VisualVM
  Connect → Threads tab → "Detect Deadlock" button
  Shows exact deadlock chain with stack traces

Method 3: Programmatic detection
  ThreadMXBean → findDeadlockedThreads()
```

### 💻 Code Example

```java
// Programmatic deadlock detection
ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();
long[] deadlockedThreads = mxBean.findDeadlockedThreads();

if (deadlockedThreads != null) {
    ThreadInfo[] threadInfos = mxBean.getThreadInfo(deadlockedThreads, true, true);
    for (ThreadInfo info : threadInfos) {
        System.err.println("DEADLOCK: " + info.getThreadName());
        System.err.println("  State: " + info.getThreadState());
        System.err.println("  Waiting for: " + info.getLockName());
        System.err.println("  Held by: " + info.getLockOwnerName());
        for (StackTraceElement ste : info.getStackTrace()) {
            System.err.println("    at " + ste);
        }
    }
}

// Schedule periodic deadlock detection
ScheduledExecutorService scheduler = Executors.newSingleThreadScheduledExecutor();
scheduler.scheduleAtFixedRate(() -> {
    long[] ids = mxBean.findDeadlockedThreads();
    if (ids != null) {
        log.error("DEADLOCK DETECTED! Threads: {}", Arrays.toString(ids));
        // Alert, take thread dump, restart if needed
    }
}, 0, 30, TimeUnit.SECONDS);
```

### 🗣️ How to Explain in Interview

> *"In production, I use thread dumps — jstack with the process ID. The JVM automatically detects deadlocks and reports them at the bottom of the thread dump with the exact chain of locks. For monitoring, I use JConsole or VisualVM — they have a 'Detect Deadlock' button that shows the chain visually. For proactive detection, I schedule a periodic check using ThreadMXBean.findDeadlockedThreads() — this runs every 30 seconds and alerts if a deadlock is found. The key is detecting quickly — deadlocks don't produce errors, they just cause threads to hang silently."*

### ⚡ Key Points to Remember

1. **jstack** = thread dump, shows deadlock chain ⭐
2. **JConsole/VisualVM** = GUI deadlock detection
3. **ThreadMXBean** = programmatic detection
4. Deadlocks are **silent** — no errors, no exceptions
5. Set up **periodic monitoring** in production

---

<a id="q67"></a>

## Q67. What is a race condition?

### 🔑 Quick Answer

> A race condition occurs when **two or more threads access shared data concurrently**, and the **outcome depends on the timing/order of execution**. The result is **non-deterministic** and often incorrect.

### 📖 Step-by-Step Explanation

```
Classic race condition — counter increment:

  Shared: int count = 0;

  count++ is actually 3 operations:
    1. READ  count (0)
    2. ADD   0 + 1 = 1
    3. WRITE count = 1

  Thread-1:        Thread-2:
  READ 0           
                   READ 0           ← STALE! Should be 1
  ADD 0+1=1        
                   ADD 0+1=1        ← Wrong calculation
  WRITE 1          
                   WRITE 1          ← Expected 2, got 1!
```

**Types of race conditions:**

```
1. Check-then-act:
   if (map.get(key) == null) {     // Thread-1 checks → null
       map.put(key, value);        // Thread-2 also checks → null
   }                                // Both put → lost update!
   FIX: map.putIfAbsent(key, value);

2. Read-modify-write:
   count++;                         // read → increment → write
   FIX: AtomicInteger.incrementAndGet();

3. Compound operation:
   if (!list.contains(x)) {         // Check
       list.add(x);                 // Act — another thread may add between check & act
   }
   FIX: Use ConcurrentHashMap.putIfAbsent() or synchronized block
```

### 🗣️ How to Explain in Interview

> *"A race condition is when the correctness of a program depends on thread scheduling order. The classic example is count++ — it's three operations: read, increment, write. If two threads read the same value before either writes, one increment is lost. There are two main patterns: check-then-act — like checking if a key exists in a map then putting it — another thread may insert between the check and the put. And read-modify-write — like incrementing a counter. Fixes include atomic operations, synchronized blocks, or using concurrent collections with built-in atomic operations like putIfAbsent()."*

### ⚡ Key Points to Remember

1. **Non-deterministic** — depends on thread scheduling
2. **count++** = classic race condition (3 ops, not atomic)
3. **Check-then-act** = another common pattern
4. Fix with: **synchronized, AtomicXxx, concurrent collections**
5. Hard to **reproduce** — may pass tests, fail in production

---

<a id="q68"></a>

## Q68. What is thread safety?

### 🔑 Quick Answer

> A class is **thread-safe** if it behaves correctly when **accessed from multiple threads simultaneously**, with no additional synchronization from the caller. Correctness means the class invariants are always maintained.

### 📖 Step-by-Step Explanation

```
Levels of thread safety:

1. IMMUTABLE (safest)
   String, Integer, LocalDate
   → No mutable state → always thread-safe ✅

2. THREAD-SAFE (internally synchronized)
   ConcurrentHashMap, AtomicInteger, StringBuffer
   → Internal locking → safe without external sync ✅

3. CONDITIONALLY THREAD-SAFE
   Collections.synchronizedMap()
   → Individual operations are safe
   → Compound operations are NOT safe (need external sync) ⚠️

4. NOT THREAD-SAFE
   HashMap, ArrayList, SimpleDateFormat
   → Must synchronize externally ❌
```

**Making a class thread-safe:**

```java
// Approach 1: Immutability
public final class Money {
    private final int amount;
    private final String currency;
    public Money(int amount, String currency) {
        this.amount = amount;
        this.currency = currency;
    }
    // No setters — thread-safe by design ✅
}

// Approach 2: Synchronization
public class Counter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int getCount() { return count; }
}

// Approach 3: Atomic variables
public class Counter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); }
    public int getCount() { return count.get(); }
}
```

### 🗣️ How to Explain in Interview

> *"Thread safety means a class works correctly under concurrent access without requiring the caller to add synchronization. There are levels — immutable objects like String are inherently thread-safe because they can't be modified. ConcurrentHashMap is thread-safe through internal locking. My first choice is immutability — if an object can't change, there's no race condition. If I need mutation, I use atomic variables for single values or synchronized blocks for compound operations. In Spring, most beans are singletons, so I ensure they're either stateless or use thread-safe fields."*

### ⚡ Key Points to Remember

1. **Immutability** = best thread safety (no mutable state)
2. **Atomic variables** = thread-safe single values
3. **synchronized** = thread-safe compound operations
4. **Concurrent collections** = thread-safe data structures
5. Spring singletons must be **stateless** or use thread-safe fields

---

<a id="q69"></a>

## Q69. What tools are used for deadlock detection and thread analysis?

### 🔑 Quick Answer

> **jstack** (thread dumps), **jconsole** (GUI monitoring), **VisualVM** (profiling), **ThreadMXBean** (programmatic), **Arthas** (production diagnostics), and **async-profiler** (low-overhead profiling). In production, jstack and thread dumps are most common.

### 📖 Step-by-Step Explanation

```
Tool Comparison:

┌─────────────────┬─────────────────────────────┬────────────┐
│ Tool            │ Purpose                      │ Production │
├─────────────────┼─────────────────────────────┼────────────┤
│ jstack          │ Thread dump (snapshot)       │ ✅ Yes     │
│ jconsole        │ GUI monitor + deadlock detect│ ⚠️ Dev/UAT │
│ VisualVM        │ Thread profiling + timeline  │ ⚠️ Dev/UAT │
│ ThreadMXBean    │ Programmatic deadlock detect │ ✅ Yes     │
│ Arthas          │ Production diagnostics       │ ✅ Yes     │
│ async-profiler  │ Low-overhead thread profiling│ ✅ Yes     │
│ Thread dump     │ kill -3 / jstack             │ ✅ Yes     │
│ IntelliJ        │ IDE debugger + thread view   │ ❌ Dev only│
└─────────────────┴─────────────────────────────┴────────────┘
```

**Thread dump analysis workflow:**

```
1. Get dump:      jstack <pid> > threaddump.txt
2. Multiple dumps: Take 3-5 dumps, 5 seconds apart
3. Compare:       Threads stuck in SAME state across dumps = problem
4. Look for:
   - BLOCKED threads → contention
   - WAITING threads → possible deadlock
   - Same stack trace in multiple dumps → stuck thread
5. Tools:         fastthread.io (online analyzer), TDA (Thread Dump Analyzer)
```

### 💻 Code Example

```java
// Production monitoring setup with ThreadMXBean
@Component
public class DeadlockMonitor {
    
    private final ThreadMXBean threadMXBean = ManagementFactory.getThreadMXBean();
    
    @Scheduled(fixedRate = 30000)  // Every 30 seconds
    public void checkForDeadlocks() {
        long[] deadlockedThreads = threadMXBean.findDeadlockedThreads();
        
        if (deadlockedThreads != null) {
            log.error("🚨 DEADLOCK DETECTED! {} threads involved", deadlockedThreads.length);
            ThreadInfo[] infos = threadMXBean.getThreadInfo(deadlockedThreads, true, true);
            
            StringBuilder sb = new StringBuilder("Deadlock details:\n");
            for (ThreadInfo info : infos) {
                sb.append(String.format("Thread: %s, State: %s, Waiting for: %s, Held by: %s\n",
                    info.getThreadName(), info.getThreadState(),
                    info.getLockName(), info.getLockOwnerName()));
            }
            log.error(sb.toString());
            // Send alert to monitoring system
        }
    }
}
```

### 🗣️ How to Explain in Interview

> *"For thread analysis in production, my go-to is jstack — I take 3-5 thread dumps about 5 seconds apart and compare them. Threads that are in the same BLOCKED or WAITING state across all dumps indicate a deadlock or contention problem. The JVM automatically detects deadlocks in thread dumps and reports the exact chain. For automated monitoring, I set up a scheduled task using ThreadMXBean.findDeadlockedThreads() that checks every 30 seconds and alerts the team. In development, VisualVM and IntelliJ's thread debugger are great for visualizing thread states. For production profiling, async-profiler gives low-overhead thread analysis."*

### ⚡ Key Points to Remember

1. **jstack** = #1 production tool for thread analysis
2. Take **multiple dumps** (3-5) and compare
3. **ThreadMXBean** = programmatic + scheduled monitoring
4. **fastthread.io** = free online thread dump analyzer
5. Stuck in same state across dumps = **problem thread**

---

> **🎯 Navigation:** [← Memory Model (Q57-62)](06-memory-model.md) | [Next → Performance (Q70-76)](08-performance.md) | [📋 All Sections](README.md)
