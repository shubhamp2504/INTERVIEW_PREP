# 🔒 Synchronization & Locks (Q18–Q28)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q18"></a>

## Q18. What is synchronization in Java?

### 🔑 Quick Answer

> Synchronization is a mechanism to **control access to shared resources** so that only **one thread at a time** can execute a critical section. It prevents thread interference and ensures data consistency.

### 📖 Step-by-Step Explanation

```
WITHOUT synchronization:
  Thread-1:  read balance(1000) → subtract 800 → write(200)
  Thread-2:  read balance(1000) → subtract 600 → write(400)
  
  Both read 1000 before either writes!
  Expected final: 1000 - 800 - 600 = -400 (should reject one)
  Actual: last write wins → 200 or 400 (data corruption!)

WITH synchronization:
  Thread-1:  [LOCK] → read(1000) → subtract(800) → write(200) → [UNLOCK]
  Thread-2:  [WAIT...] → [LOCK] → read(200) → reject (insufficient) → [UNLOCK]
  
  Thread-2 sees the CORRECT balance after Thread-1's update ✅
```

### 🗣️ How to Explain in Interview

> *"Synchronization ensures that only one thread can execute a block of code at a time when accessing shared data. In Java, every object has an intrinsic lock — also called a monitor lock. When a thread enters a synchronized block, it acquires the lock. Other threads trying to enter a synchronized block on the same object are blocked until the lock is released. This gives us mutual exclusion — guaranteeing that compound operations like read-modify-write happen atomically."*

### ⚡ Key Points to Remember

1. **One thread at a time** in critical section
2. Uses **intrinsic lock** (monitor) on an object
3. Prevents **race conditions** and **data corruption**
4. Trade-off: **correctness vs performance** (contention)
5. Keep synchronized blocks **as short as possible**

---

<a id="q19"></a>

## Q19. What is the purpose of the synchronized keyword?

### 🔑 Quick Answer

> It provides **mutual exclusion** (only one thread in the block) and **visibility** (changes made in a synchronized block are visible to other threads). Two forms: synchronized **method** and synchronized **block**.

### 💻 Code Example

```java
// Synchronized method — locks on 'this'
public synchronized void deposit(int amount) {
    this.balance += amount;  // Only one thread at a time
}

// Synchronized block — locks on specific object
public void withdraw(int amount) {
    synchronized (this) {    // Explicit lock object
        if (balance >= amount) {
            balance -= amount;
        }
    }
    // Code outside sync block runs concurrently
}

// Static synchronized — locks on Class object
public static synchronized void updateGlobalCounter() {
    globalCount++;  // Locks on MyClass.class
}
```

### 🗣️ How to Explain in Interview

> *"The synchronized keyword serves two purposes: mutual exclusion and memory visibility. Mutual exclusion means only one thread can execute the synchronized code at a time. Memory visibility means when a thread exits a synchronized block, all its changes are flushed to main memory, and when another thread enters, it reads fresh values. I can apply it to methods — which locks on 'this' for instance methods or the Class object for static methods — or use synchronized blocks for finer-grained locking on a specific object."*

### ⚡ Key Points to Remember

1. **Mutual exclusion** + **memory visibility**
2. Instance method → locks on **this**
3. Static method → locks on **Class object**
4. Block → locks on **specified object**
5. Automatically releases lock on exit (even on exception)

---

<a id="q20"></a>

## Q20. What is the difference between synchronized method and synchronized block?

### 🔑 Quick Answer

> Synchronized **method** locks the entire method on `this` (or Class for static). Synchronized **block** locks only a specific section on any chosen object. Block is **more flexible and performant** — smaller critical section.

### 📖 Step-by-Step Explanation

```java
// Synchronized METHOD: Entire method is locked
public synchronized void transfer(Account to, int amount) {
    // ALL of this is locked (even non-critical code)
    log.info("Starting transfer");         // Doesn't need lock!
    this.balance -= amount;                // Needs lock
    to.balance += amount;                  // Needs lock
    log.info("Transfer complete");         // Doesn't need lock!
}

// Synchronized BLOCK: Only critical section is locked ⭐
public void transfer(Account to, int amount) {
    log.info("Starting transfer");         // Runs freely (no lock)
    synchronized (this) {                  // Lock only what's needed
        this.balance -= amount;
        to.balance += amount;
    }
    log.info("Transfer complete");         // Runs freely (no lock)
}
```

| Feature | Synchronized Method | Synchronized Block |
|---------|-------------------|-------------------|
| **Lock scope** | Entire method | Specific code section |
| **Lock object** | `this` or `Class` | Any object you choose |
| **Performance** | Locks more than needed | Locks only what's needed ⭐ |
| **Flexibility** | Fixed to `this`/Class | Can lock on multiple objects |
| **Readability** | Simpler syntax | Explicit lock object |

### 🗣️ How to Explain in Interview

> *"Synchronized blocks are generally preferred over synchronized methods. A synchronized method locks the entire method body on 'this', which means even non-critical code holds the lock — increasing contention. A synchronized block lets me lock only the exact code that needs protection, on any object I choose. For example, if I have two independent resources, I can use two different lock objects — so threads accessing different resources don't block each other. The only advantage of synchronized methods is simplicity."*

### ⚡ Key Points to Remember

1. **Block > Method** for performance (smaller critical section)
2. Method locks on **this** (fixed), block locks on **any object**
3. Block → **finer-grained locking** → less contention
4. Two independent resources → **two different lock objects**
5. Method is simpler but **locks too broadly**

---

<a id="q21"></a>

## Q21. What is intrinsic locking (monitor lock)?

### 🔑 Quick Answer

> Every Java object has a built-in lock called an **intrinsic lock** or **monitor lock**. When a thread enters a `synchronized` block/method, it **acquires** the monitor. Other threads wait until it's **released**.

### 📖 Step-by-Step Explanation

```
Every Object in Java heap:
  ┌─────────────────────┐
  │ Object Header        │
  │  ├── Mark Word       │ ← Contains lock information
  │  │   (lock state,    │
  │  │    thread ID,     │
  │  │    hash code)     │
  │  └── Class Pointer   │
  │ Instance Data        │
  └─────────────────────┘

Lock states (Mark Word):
  Unlocked    → Biased     → Lightweight  → Heavyweight
  (no thread)   (1 thread)   (CAS-based)   (OS mutex)
  
  JVM optimizes: starts biased, escalates only if contention occurs
```

### 🗣️ How to Explain in Interview

> *"Every Java object has an intrinsic lock stored in its object header. When a thread enters a synchronized block, it acquires this lock. If the lock is already held, the thread blocks. The JVM optimizes this with lock escalation — it starts with biased locking for single-thread scenarios, moves to lightweight CAS-based locking for low contention, and escalates to heavyweight OS mutex only under high contention. This is why synchronized in modern JVMs is quite efficient for uncontested cases."*

### ⚡ Key Points to Remember

1. Every object has **one intrinsic lock** (in object header)
2. `synchronized` acquires the **object's monitor**
3. Lock escalation: **biased → lightweight → heavyweight**
4. Modern JVMs **optimize** uncontested synchronized
5. Only **one thread** can hold the intrinsic lock at a time

---

<a id="q22"></a>

## Q22. What is reentrant locking?

### 🔑 Quick Answer

> A thread that **already holds a lock can acquire the same lock again** without deadlocking. Java's `synchronized` and `ReentrantLock` are both reentrant. The lock maintains a **hold count** that increments on each acquisition.

### 📖 Step-by-Step Explanation

```java
// This would DEADLOCK without reentrant locking:
public synchronized void methodA() {
    // Thread holds lock on 'this'
    methodB();  // methodB also needs lock on 'this'!
}

public synchronized void methodB() {
    // Without reentrancy → deadlock (waiting for lock it already holds!)
    // With reentrancy → hold count goes from 1 to 2 ✅
}

// Internal mechanism:
// methodA(): acquire lock → holdCount = 1
//   methodB(): same thread → holdCount = 2
//   methodB() returns → holdCount = 1
// methodA() returns → holdCount = 0 → LOCK RELEASED
```

### 🗣️ How to Explain in Interview

> *"Reentrant locking means a thread can acquire the same lock multiple times without deadlocking. The lock keeps a hold count — it increments when the same thread acquires it and decrements when it releases. The lock is only truly released when the count reaches zero. This is essential because synchronized methods often call other synchronized methods on the same object. Without reentrancy, a thread calling methodA() which calls methodB() — both synchronized on 'this' — would deadlock waiting for a lock it already holds."*

### ⚡ Key Points to Remember

1. **Same thread** can acquire **same lock** multiple times
2. **Hold count**: increments on acquire, decrements on release
3. Lock released when count → **0**
4. Both `synchronized` and `ReentrantLock` are reentrant
5. Without reentrancy → **self-deadlock** in nested synchronized calls

---

<a id="q23"></a>

## Q23. What is ReentrantLock?

### 🔑 Quick Answer

> A **java.util.concurrent.locks** lock that provides the same mutual exclusion as `synchronized` but with extra features: **tryLock()** (with timeout), **fairness**, **interruptible locking**, and **multiple condition variables**.

### 📖 Step-by-Step Explanation

```
synchronized:
  Simple, automatic lock/unlock, no timeout, not interruptible

ReentrantLock:
  ✅ tryLock(timeout)     — try to acquire, give up after timeout
  ✅ lockInterruptibly()  — can be interrupted while waiting
  ✅ Fairness option      — FIFO ordering (no starvation)
  ✅ Multiple Conditions   — multiple wait-sets per lock
  ❌ Manual unlock (must be in finally block!)
```

### 💻 Code Example

```java
private final ReentrantLock lock = new ReentrantLock();

public void transfer(Account to, int amount) {
    lock.lock();        // Acquire lock (blocks if held by another thread)
    try {
        this.balance -= amount;
        to.balance += amount;
    } finally {
        lock.unlock();  // ALWAYS unlock in finally! ⚠️
    }
}

// tryLock — avoid deadlock with timeout
public boolean tryTransfer(Account to, int amount) throws InterruptedException {
    if (lock.tryLock(2, TimeUnit.SECONDS)) {  // Wait up to 2 seconds
        try {
            this.balance -= amount;
            to.balance += amount;
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false;  // Couldn't get lock within timeout
}
```

### 🗣️ How to Explain in Interview

> *"ReentrantLock gives me everything synchronized does, plus extras. The biggest advantage is tryLock() with a timeout — if I can't acquire the lock within 2 seconds, I can give up and do something else, preventing deadlocks. It also supports fairness — new ReentrantLock(true) ensures threads acquire the lock in FIFO order, preventing starvation. And lockInterruptibly() lets me interrupt a thread that's waiting for a lock. The trade-off is responsibility — I must always unlock in a finally block, whereas synchronized releases automatically."*

### ⚡ Key Points to Remember

1. **tryLock(timeout)** = prevent deadlock ⭐
2. **fairness** = FIFO ordering (prevent starvation)
3. **lockInterruptibly()** = cancel waiting threads
4. **Always unlock in finally** (or use try-with-resources pattern)
5. Use when you need features beyond `synchronized`

---

<a id="q24"></a>

## Q24. Difference between ReentrantLock and synchronized?

### 🔑 Quick Answer

> `synchronized` is simpler (auto-release). `ReentrantLock` is more powerful (**tryLock, fairness, interruptible, multiple conditions**). Use `synchronized` for simple cases, `ReentrantLock` when you need advanced features.

### 📖 Step-by-Step Explanation

| Feature | synchronized | ReentrantLock |
|---------|-------------|--------------|
| **Lock/Unlock** | Automatic | Manual (lock/unlock) |
| **tryLock with timeout** | ❌ No | ✅ Yes ⭐ |
| **Fairness** | ❌ No (unfair) | ✅ Optional (true/false) |
| **Interruptible** | ❌ No | ✅ lockInterruptibly() |
| **Multiple conditions** | ❌ 1 wait-set | ✅ Multiple Conditions |
| **Performance** | Optimized by JVM | Similar |
| **Risk** | Safe (auto-release) | Risky (can forget unlock) |
| **Readability** | Simpler | More verbose |

```
When to use which:

synchronized:
  ✅ Simple mutual exclusion
  ✅ No timeout needed
  ✅ No fairness requirement
  ✅ Want simplicity

ReentrantLock:
  ✅ Need tryLock() with timeout (deadlock prevention)
  ✅ Need fairness (prevent starvation)
  ✅ Need interruptible lock acquisition
  ✅ Need multiple condition variables
```

### 🗣️ How to Explain in Interview

> *"I use synchronized when I need simple mutual exclusion — it's safer because the lock is automatically released even on exceptions. I switch to ReentrantLock when I need its specific features: tryLock with timeout for deadlock prevention, fairness for starvation prevention, or lockInterruptibly for thread cancellation. In modern Java, synchronized is well-optimized by the JVM, so the performance difference is negligible. My default is synchronized unless I need a specific ReentrantLock feature."*

### ⚡ Key Points to Remember

1. **Default to synchronized** (simpler, safer)
2. Switch to ReentrantLock for: **timeout, fairness, interruptible**
3. **Always** unlock in finally with ReentrantLock
4. Performance: **similar** in modern JVMs
5. ReentrantLock supports **multiple Condition objects**

---

<a id="q25"></a>

## Q25. What is a fair lock?

### 🔑 Quick Answer

> A fair lock grants access in **FIFO order** — the thread that has been waiting the longest gets the lock next. Prevents **starvation** but reduces **throughput** compared to unfair locks.

### 📖 Step-by-Step Explanation

```
UNFAIR LOCK (default):
  Thread-A waiting for 10ms
  Thread-B waiting for 5ms
  Thread-C just arrived

  Lock released → Thread-C may steal it! (even though A waited longest)
  → Higher throughput (less overhead)
  → Risk of starvation for long-waiting threads

FAIR LOCK:
  Thread-A waiting for 10ms
  Thread-B waiting for 5ms
  Thread-C just arrived

  Lock released → Thread-A gets it (waited longest) ← FIFO
  → No starvation
  → Lower throughput (queue management overhead)
```

```java
// Unfair lock (default — higher throughput)
ReentrantLock unfairLock = new ReentrantLock();

// Fair lock (FIFO — no starvation)
ReentrantLock fairLock = new ReentrantLock(true);
```

### 🗣️ How to Explain in Interview

> *"A fair lock uses FIFO ordering — the thread waiting the longest gets the lock next. This prevents starvation but costs throughput, roughly 10-20% slower, because the JVM has to maintain a queue. Unfair locks — which are the default — allow newly arriving threads to 'steal' the lock, which gives better throughput due to better cache locality. I use fair locks only when starvation is a real concern, like in systems where all threads must get fair access to a shared resource."*

### ⚡ Key Points to Remember

1. **Fair** = FIFO ordering, no starvation
2. **Unfair** = better throughput, risk of starvation
3. `new ReentrantLock(true)` = fair
4. Fair is **10-20% slower** than unfair
5. Default is **unfair** (for performance)

---

<a id="q26"></a>

## Q26. What is read-write lock?

### 🔑 Quick Answer

> A lock that allows **multiple concurrent readers** but only **one exclusive writer**. Read operations don't block each other, but writes block everything. Ideal for **read-heavy** data structures.

### 📖 Step-by-Step Explanation

```
Regular lock (synchronized):
  Reader-1: [LOCK] read... [UNLOCK]
  Reader-2: [WAIT...] [LOCK] read... [UNLOCK]  ← Readers block each other!
  Reader-3: [WAIT...] [WAIT...] [LOCK] read...  ← All sequential!

Read-Write lock:
  Reader-1: [READ-LOCK] read...  ─────→
  Reader-2: [READ-LOCK] read...  ─────→   ALL read simultaneously! ⭐
  Reader-3: [READ-LOCK] read...  ─────→
  Writer:   [WAIT...] [WRITE-LOCK] write... [UNLOCK]   ← Exclusive
```

| Scenario | Regular Lock | ReadWriteLock |
|----------|-------------|--------------|
| Multiple readers | Sequential ❌ | Concurrent ✅ |
| Reader + Writer | Blocking | Writer waits for readers |
| Multiple writers | Sequential | Sequential |
| Read-heavy workload | Slow | Fast ⭐ |

### 🗣️ How to Explain in Interview

> *"A read-write lock distinguishes between read and write operations. Multiple threads can hold the read lock simultaneously — reads don't conflict with each other. But the write lock is exclusive — no readers or writers can proceed while a write is happening. This is perfect for data structures that are read much more frequently than written, like a configuration cache. Using a regular synchronized lock for that would make all reads sequential, even though they don't interfere with each other."*

### ⚡ Key Points to Remember

1. **Multiple readers** simultaneously ✅
2. **One writer** exclusively (no readers allowed)
3. **Read-heavy** workloads benefit most
4. Write-heavy → no benefit (writes are still exclusive)
5. Java: `ReentrantReadWriteLock`

---

<a id="q27"></a>

## Q27. What is ReentrantReadWriteLock?

### 🔑 Quick Answer

> Java's implementation of read-write lock. Has `readLock()` and `writeLock()` methods. Supports **reentrant** acquisition, optional **fairness**, and **lock downgrading** (write → read).

### 💻 Code Example

```java
private final ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock();
private final Lock readLock = rwLock.readLock();
private final Lock writeLock = rwLock.writeLock();
private Map<String, String> cache = new HashMap<>();

// Multiple threads can read simultaneously
public String get(String key) {
    readLock.lock();
    try {
        return cache.get(key);
    } finally {
        readLock.unlock();
    }
}

// Only one thread can write at a time (exclusive)
public void put(String key, String value) {
    writeLock.lock();
    try {
        cache.put(key, value);
    } finally {
        writeLock.unlock();
    }
}
```

### 🗣️ How to Explain in Interview

> *"ReentrantReadWriteLock gives me a readLock and writeLock pair. In a cache implementation, I use readLock for get operations — multiple threads read concurrently. For put operations, I use writeLock — it's exclusive, blocking all readers and writers until the write completes. It supports lock downgrading — I can acquire writeLock, then readLock, then release writeLock while keeping readLock. But not upgrading — you can't go from read to write lock without releasing read first, because that could deadlock."*

### ⚡ Key Points to Remember

1. `readLock()` — shared (multiple readers)
2. `writeLock()` — exclusive (one writer)
3. **Downgrade** OK (write → read), **upgrade** NOT OK (read → write)
4. Supports **fairness** option
5. Great for **caches** and **configuration stores**

---

<a id="q28"></a>

## Q28. What is lock contention?

### 🔑 Quick Answer

> When multiple threads **compete for the same lock**, causing some threads to **block and wait**. High contention = threads spend more time waiting than working = poor performance.

### 📖 Step-by-Step Explanation

```
LOW CONTENTION (fast):
  T1: [lock] work [unlock]
  T2: [lock] work [unlock]     ← Rarely overlap, little waiting
  T3: [lock] work [unlock]

HIGH CONTENTION (slow):
  T1: [lock] work..........[unlock]
  T2: [WAIT..............] [lock] work [unlock]
  T3: [WAIT..................................] [lock] work [unlock]
  
  Threads spend 80% time WAITING, 20% WORKING → poor throughput

SOLUTIONS:
  1. Reduce lock scope (shorter critical section)
  2. Use concurrent data structures (ConcurrentHashMap)
  3. Lock striping (multiple locks for different segments)
  4. Read-write locks (readers don't block each other)
  5. Lock-free algorithms (CAS-based: AtomicInteger)
```

### 🗣️ How to Explain in Interview

> *"Lock contention happens when many threads compete for the same lock simultaneously. The fix depends on the root cause. If the critical section is too large, I shrink it — lock only the minimum necessary code. If many threads read the same data, I use ReadWriteLock or ConcurrentHashMap. If there's a single hot lock, I use lock striping — ConcurrentHashMap does this by locking individual buckets instead of the whole map. For simple counters, I use AtomicInteger which is completely lock-free using CAS operations."*

### ⚡ Key Points to Remember

1. **Contention** = threads fighting for the same lock
2. Fix: **smaller critical sections** (less time holding lock)
3. Fix: **ReadWriteLock** for read-heavy workloads
4. Fix: **lock striping** (multiple locks for segments)
5. Fix: **lock-free** CAS (AtomicInteger, ConcurrentHashMap)

---

> **🎯 Navigation:** [← Basics (Q1-17)](01-basics.md) | [Next → Thread Communication (Q29-36)](03-thread-communication.md) | [📋 All Sections](README.md)
