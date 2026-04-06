# 🔒 Deadlock & Concurrency Problems (Q63–Q69)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q63"></a>
## Q63. What is deadlock?

### 📝 One-Liner
> Two or more threads permanently blocked — each waiting for a lock held by the other, neither can proceed.

### 🔑 Quick Answer
> Deadlock = **circular wait** where Thread-1 holds Lock A wanting Lock B, Thread-2 holds Lock B wanting Lock A. Neither can proceed — **permanent hang**, no errors, no logs, just silence. *(Do thread ek dusre ka lock maang rahe hain — dono ruk gaye, koi aage nahi badhega)*

### 📖 How It Works
```
Thread-1:                         Thread-2:
lock(A) ✅                        lock(B) ✅
lock(B) → WAITING...             lock(A) → WAITING...
       ↑                                ↑
       └────────── DEADLOCK ─────────────┘

Thread-1 holds A, wants B
Thread-2 holds B, wants A
→ Neither proceeds → PERMANENT HANG 💀
*(Dono ek dusre ka lock pakde hain — koi nahi chhodega)*
```

### 💻 Code
```java
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {           // holds A
        Thread.sleep(100);
        synchronized (lockB) { }     // wants B → BLOCKED!
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {           // holds B
        Thread.sleep(100);
        synchronized (lockA) { }     // wants A → BLOCKED!
    }
});
t1.start(); t2.start();  // DEADLOCK → both hang forever
```

### 🗣️ Answering Approach
> *"Deadlock is when two or more threads are permanently waiting for each other's locks. Thread-1 holds Lock A and waits for Lock B, while Thread-2 holds Lock B and waits for Lock A. In production, deadlocks are dangerous because there's no error thrown — the application silently hangs. I've seen this with database row locks where two transactions each held different rows and tried to acquire the other."*

### ⚠️ Pitfalls / Gotchas
- **No error thrown** — app silently hangs *(koi exception nahi — bas hang)*
- Deadlocks are **timing-dependent** — hard to reproduce in testing *(test mein nahi dikhega, production mein aayega)*
- `synchronized` has **no timeout** — blocked forever *(timeout nahi hai — hamesha wait karega)*

### ⚡ Remember
1. **Two+ threads** waiting for each other's locks
2. **Permanent** — won't resolve on its own
3. **No error** — silent hang *(sabse khatarnak)*
4. Most common: **nested locks in different order**
5. Production symptom: **hung threads, no CPU usage**

### 🔗 Follow-ups
→ [Q64. Coffman conditions](#q64) → [Q65. Prevention](#q65)

---

<a id="q64"></a>
## Q64. What are the four conditions for deadlock?

### 📝 One-Liner
> Coffman conditions: mutual exclusion + hold-and-wait + no preemption + circular wait — break ANY one to prevent deadlock.

### 🔑 Quick Answer
> All **four must be true simultaneously** for deadlock (Coffman conditions). Break **any one** = deadlock impossible. Most practical: break **circular wait** with consistent lock ordering. *(Chaar sharti ek saath lagni chahiye — ek bhi todo toh deadlock nahi hoga)*

### 📖 How It Works
```
1. MUTUAL EXCLUSION:    Lock held by only ONE thread ✅
2. HOLD AND WAIT:       Holding one lock, waiting for another ✅
3. NO PREEMPTION:       Can't force release of a lock ✅
4. CIRCULAR WAIT:       T1→waits→T2→waits→T1 (cycle) ✅

Break ANY ONE → Deadlock IMPOSSIBLE!
```

| Condition | How to Break |
|-----------|-------------|
| Mutual Exclusion | Use concurrent collections (shared access) |
| Hold and Wait | Lock ALL at once or use tryLock + release |
| No Preemption | Use tryLock() with timeout |
| **Circular Wait** | **Lock ordering** (always same order) ⭐ |

### 🗣️ Answering Approach
> *"Deadlock requires the four Coffman conditions simultaneously. The most practical prevention is breaking circular wait by enforcing a global lock ordering — always acquire Lock A before Lock B, regardless of which thread. Another approach is tryLock with timeout to break hold-and-wait. In my projects, I sort locks by a deterministic order like object ID."*

### 🎯 Tricky Interview Qs
**Q: Which condition is easiest to break in practice?**
> Circular wait — by enforcing consistent lock ordering (e.g., always lock by ascending ID). *(Lock ka order fix karo — chhota ID pehle, bada baad mein)*

### ⚡ Remember
1. **All 4** conditions must hold simultaneously
2. Break **any one** → no deadlock
3. **Lock ordering** = most practical prevention ⭐
4. **tryLock(timeout)** = breaks hold-and-wait
5. Know **names + how to break each** for interview

### 🔗 Follow-ups
→ [Q65. Prevention](#q65)

---

<a id="q65"></a>
## Q65. How to prevent deadlock?

### 📝 One-Liner
> Lock ordering (same global order), tryLock with timeout (back off), lock-free structures (CAS), minimize lock scope.

### 🔑 Quick Answer
> **#1: Lock ordering** — always acquire locks in the same deterministic order (e.g., by ID). **#2: tryLock with timeout** — back off and retry if lock unavailable. **#3: Lock-free** — CAS-based structures. **#4: Minimize scope** — hold locks for shortest time. *(Sabse aasan — hamesha ek hi order mein lock karo)*

### 💻 Code
```java
// Fix 1: Lock ordering — consistent order by account ID ⭐
private void transfer(Account from, Account to, int amount) {
    Account first = from.id < to.id ? from : to;   // deterministic order
    Account second = from.id < to.id ? to : from;

    synchronized (first) {
        synchronized (second) {
            from.balance -= amount;
            to.balance += amount;
        }
    }
}

// Fix 2: tryLock with timeout — back off if can't acquire
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
        Thread.sleep(random.nextInt(100));  // random backoff
    }
}
```

### 🗣️ Answering Approach
> *"My go-to strategy is lock ordering — impose a global order on all locks, like by object ID, and always acquire in that order. For the bank transfer problem, I lock the account with the lower ID first. When I can't control lock order, I use tryLock with timeout — if I can't get the second lock, I release the first and retry with random backoff. Beyond locks, I prefer lock-free alternatives like ConcurrentHashMap — no locks means no deadlock."*

### ⚠️ Pitfalls / Gotchas
- **Never call external/unknown code** while holding a lock *(bahar ka code lock ke andar mat bulo — woh bhi lock le sakta hai)*
- tryLock retry loop needs **random backoff** to avoid livelock *(dono ek saath try karenge — dono fail, phir repeat)*

### ⚡ Remember
1. **Lock ordering** = #1 strategy ⭐ *(hamesha ek hi order)*
2. **tryLock(timeout)** = back off if unavailable
3. **Lock-free** structures eliminate risk
4. **Minimize** lock scope (hold briefly)
5. **Never** call external code holding a lock

### 🔗 Follow-ups
→ [Q66. Detection](#q66)

---

<a id="q66"></a>
## Q66. How to detect deadlock?

### 📝 One-Liner
> Thread dump (jstack), JConsole/VisualVM GUI, ThreadMXBean programmatic detection — thread dumps are the #1 production tool.

### 🔑 Quick Answer
> **jstack \<pid\>** = takes thread dump, JVM auto-detects deadlock chain. **JConsole/VisualVM** = GUI with "Detect Deadlock" button. **ThreadMXBean** = programmatic detection in code. *(jstack sabse common — thread dump le lo, deadlock apne aap dikh jaayega)*

### 💻 Code
```java
// Programmatic deadlock detector — add to Spring app ⭐
@Component
public class DeadlockMonitor {
    private final ThreadMXBean mxBean = ManagementFactory.getThreadMXBean();

    @Scheduled(fixedRate = 30000)  // every 30 sec
    public void check() {
        long[] ids = mxBean.findDeadlockedThreads();
        if (ids != null) {
            ThreadInfo[] infos = mxBean.getThreadInfo(ids, true, true);
            for (ThreadInfo info : infos) {
                log.error("DEADLOCK: {} waiting for {} held by {}",
                    info.getThreadName(), info.getLockName(), info.getLockOwnerName());
            }
            // alert team!
        }
    }
}

// Thread dump via terminal:
// $ jstack <pid> > dump.txt
// Look for: "Found one Java-level deadlock"
```

### 🗣️ Answering Approach
> *"In production, I take thread dumps with jstack — the JVM automatically detects deadlocks and reports the exact lock chain. I take 3 dumps 5 seconds apart — threads stuck in the same state across dumps indicate a problem. For proactive detection, I add a scheduled DeadlockDetector using ThreadMXBean that checks every 30 seconds and alerts the team."*

### ⚡ Remember
1. **jstack \<pid\>** = thread dump, shows deadlock chain ⭐
2. **JConsole/VisualVM** = GUI detection
3. **ThreadMXBean** = programmatic in-app detection
4. Take **3+ dumps**, 5 sec apart *(ek dump se confirm nahi hota)*
5. Deadlocks are **silent** — no errors, no exceptions

### 🔗 Follow-ups
→ [Q89. Debug deadlock in production](10-production-scenarios.md#q89)

---

<a id="q67"></a>
## Q67. What is a race condition?

### 📝 One-Liner
> Outcome depends on thread scheduling timing — two threads access shared data, result is non-deterministic and often wrong.

### 🔑 Quick Answer
> A race condition = correctness depends on **which thread runs first**. Classic: `count++` — two threads read same value, both increment, one update lost. **Non-deterministic** and **hard to reproduce**. *(Kaun pehle chalega uspe result depend karta hai — galat answer aata hai)*

### 📖 How It Works
```
count++ is actually 3 operations:
  1. READ  count (0)
  2. ADD   0 + 1 = 1
  3. WRITE count = 1

Thread-1:        Thread-2:
READ 0
                 READ 0        ← stale! should be 1
ADD 0+1=1
                 ADD 0+1=1     ← same wrong calc
WRITE 1
                 WRITE 1       ← expected 2, got 1! 💀

Types:
  check-then-act:  if(map.get(key)==null) map.put(key,val)  ← race!
  read-modify-write: count++                                 ← race!
  *(Beech mein doosra thread aa gaya — data galat)*
```

### ⚠️ Pitfalls / Gotchas
- **Hard to reproduce** — may pass 1M tests, fail in production *(test mein sahi, production mein galat)*
- Even **single-line** code can be a race (count++ = 3 bytecode ops)
- **Check-then-act** on ConcurrentHashMap still racy if using separate get+put *(get-then-put mat karo — compute/putIfAbsent use karo)*

### 🗣️ Answering Approach
> *"A race condition is when program correctness depends on thread scheduling order. The classic example is count++ — three operations that aren't atomic. Two patterns to watch: check-then-act, like testing if a map key exists then putting — use putIfAbsent instead. And read-modify-write like count++ — use AtomicInteger. Race conditions are especially dangerous because they're hard to reproduce in testing but appear under production load."*

### ⚡ Remember
1. **Non-deterministic** — depends on timing *(random galti)*
2. **count++** = classic race (3 ops, not atomic)
3. **Check-then-act** = another common pattern
4. Fix: **synchronized, AtomicXxx, concurrent collections**
5. **Hard to reproduce** — passes tests, fails in prod

### 🔗 Follow-ups
→ [Q68. Thread safety](#q68)

---

<a id="q68"></a>
## Q68. What is thread safety?

### 📝 One-Liner
> A class is thread-safe if it works correctly under concurrent access without the caller needing to add any synchronization.

### 🔑 Quick Answer
> Thread-safe = **correct behavior** under multi-threaded access with **no external sync** needed. Levels: **Immutable** (best — String, Integer), **Thread-safe** (ConcurrentHashMap), **Conditionally safe** (synchronizedMap — compound ops not safe), **Not safe** (HashMap, ArrayList). *(Thread-safe matlab — bina lock lagaye bhi sab sahi chalega)*

### 📖 How It Works
```
Levels of thread safety:

1. IMMUTABLE (safest):
   String, Integer, LocalDate
   → No mutable state → always safe ✅
   *(Badal hi nahi sakta — toh galat kaise hoga)*

2. THREAD-SAFE (internally synced):
   ConcurrentHashMap, AtomicInteger
   → Internal locking → safe ✅

3. CONDITIONALLY SAFE:
   Collections.synchronizedMap()
   → Individual ops safe, compound ops NOT ⚠️

4. NOT SAFE:
   HashMap, ArrayList, SimpleDateFormat
   → Must sync externally ❌
```

### 💻 Code
```java
// Approach 1: Immutability (BEST)
public final class Money {
    private final int amount;
    private final String currency;
    // No setters → thread-safe by design ✅
}

// Approach 2: Atomic variables
private final AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet();  // lock-free, atomic ✅

// Approach 3: Synchronized
public synchronized void increment() { count++; }  // one thread at a time ✅
```

### 🗣️ Answering Approach
> *"Thread safety means correct behavior under concurrent access without external synchronization. My first choice is immutability — if it can't change, there's no race condition. Second, atomic variables for single values. Third, concurrent collections for data structures. synchronized blocks as last resort, with smallest scope. In Spring, beans are singletons, so I ensure they're either stateless or use thread-safe fields."*

### ⚡ Remember
1. **Immutability** = best thread safety *(badal nahi sakta = safe)*
2. **Atomic variables** = lock-free single values
3. **Concurrent collections** = thread-safe structures
4. **synchronized** = last resort, minimize scope
5. Spring singletons must be **stateless** or thread-safe

### 🔗 Follow-ups
→ [Q88. Ensure thread safety for shared resource](10-production-scenarios.md#q88)

---

<a id="q69"></a>
## Q69. What tools are used for deadlock detection and thread analysis?

### 📝 One-Liner
> jstack (thread dump), JConsole/VisualVM (GUI), ThreadMXBean (programmatic), Arthas (production), async-profiler (low overhead).

### 🆚 vs. Comparison
| Tool | Purpose | Production? |
|------|---------|------------|
| **jstack** | Thread dump snapshot ⭐ | ✅ Yes |
| **JConsole** | GUI monitor + deadlock detect | ⚠️ Dev/UAT |
| **VisualVM** | Thread profiling + timeline | ⚠️ Dev/UAT |
| **ThreadMXBean** | Programmatic detection | ✅ Yes |
| **Arthas** | Production diagnostics | ✅ Yes |
| **async-profiler** | Low-overhead profiling | ✅ Yes |
| **fastthread.io** | Online dump analyzer | ✅ Yes |

### 📖 How It Works
```
Thread dump analysis workflow:
  1. Get dump:    jstack <pid> > dump.txt
  2. Multiple:    Take 3-5 dumps, 5 seconds apart
  3. Compare:     Threads stuck in SAME state = problem
  4. Look for:
     - BLOCKED → lock contention
     - WAITING → possible deadlock
     - Same stack in multiple dumps → stuck thread
  5. Analyze:     fastthread.io or TDA (Thread Dump Analyzer)
```

### 🗣️ Answering Approach
> *"My primary tool is jstack for thread dumps — it auto-detects deadlocks. I take 3 dumps 5 seconds apart and compare — threads stuck in the same state across dumps indicate a problem. For monitoring, I use JConsole or VisualVM in development. In production, I add a scheduled ThreadMXBean check that alerts on deadlock. For online analysis, fastthread.io parses the dump and visualizes thread states and lock contention. Arthas is great for live production diagnostics without restarting the app."*

### ⚡ Remember
1. **jstack** = #1 production tool ⭐
2. **3+ dumps**, 5 sec apart (compare for stuck threads)
3. **ThreadMXBean** = automated in-app monitoring
4. **fastthread.io** = online dump analyzer
5. Deadlocks are **silent** — proactive detection is key

### 🔗 Follow-ups
→ [Q89. Debug deadlock in production](10-production-scenarios.md#q89)

---

> **🎯 Navigation:** [← Memory Model (Q57-62)](06-memory-model.md) | [Next → Performance (Q70-76)](08-performance.md) | [📋 All Sections](README.md)
