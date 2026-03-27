# 🔒 Synchronization & Locks (Q18–Q28)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q18"></a>
## Q18. What is synchronization in Java?

### 📝 One-Liner
> Mechanism to allow only one thread at a time to access shared resource — prevents race conditions.

### 🔑 Quick Answer
> Synchronization ensures **mutual exclusion** — only one thread enters the critical section at a time. It also provides **visibility** — changes by one thread are visible to others. *(Ek time pe sirf ek thread andar — baaki line mein khade rahe)*

### 📖 How It Works
```
Without sync:
  Thread-1: read(count=0) → add → write(1)
  Thread-2: read(count=0) → add → write(1)  ← LOST UPDATE!
  *(Dono ne 0 padha, dono ne 1 likha — ek increment gaya)*

With sync:
  Thread-1: [LOCK] read(0) → add → write(1) [UNLOCK]
  Thread-2: [WAIT...] → [LOCK] read(1) → add → write(2) [UNLOCK] ✅
  *(Ek andar gaya — doosra bahar wait karta hai — sahi result)*
```

### 🗣️ How to Say in Interview
> *"Synchronization in Java provides two guarantees: mutual exclusion — only one thread can execute the synchronized block at a time — and memory visibility — changes made inside a synchronized block are flushed to main memory when the lock is released, so other threads see the latest values. In production, I prefer higher-level abstractions like ReentrantLock or concurrent collections, but synchronized is still the go-to for simple cases."*

### ⚠️ Pitfalls / Gotchas
- Synchronization has **cost** — lock acquisition, context switching *(free nahi hai)*
- Over-synchronization → **poor performance** *(har jagah lock lagaoge toh single-threaded jaisa ho jayega)*
- Under-synchronization → **race conditions** *(kam lagaoge toh data corrupt)*

### ⚡ Remember
1. **Mutual exclusion** — one thread at a time
2. **Visibility** — changes flushed to main memory
3. Cost: lock overhead + potential contention
4. Balance: not too much, not too little

### 🔗 Follow-ups
→ [Q19. synchronized keyword](#q19) → [Q20. Method vs block](#q20)

---

<a id="q19"></a>
## Q19. How does the synchronized keyword work?

### 📝 One-Liner
> Acquires intrinsic lock (monitor) on an object — only one thread can hold it at a time.

### 🔑 Quick Answer
> `synchronized` acquires the **intrinsic lock (monitor)** on the specified object. Only one thread can hold this lock. Others block until it's released. *(Ek chabi hai — jo pehle le gaya woh andar, baaki darwaze pe khade)*

### 📖 How It Works
```
synchronized method → locks on 'this' (instance) or Class (static)
synchronized block  → locks on specified object

  Thread-1: synchronized(obj) { ... }  → acquires obj's monitor
  Thread-2: synchronized(obj) { ... }  → BLOCKED (same monitor)
  Thread-3: synchronized(other) { ... } → RUNS (different monitor!) ✅
  
  *(obj ka lock Thread-1 ke paas — Thread-2 wait karega,
    Thread-3 alag object ka lock hai toh chal jayega)*
```

### 💻 Code
```java
// Synchronized method — locks on 'this'
public synchronized void increment() {
    count++;  // only one thread at a time
}

// Synchronized block — locks on specific object (finer control)
public void increment() {
    synchronized (this) {  // same as synchronized method
        count++;
    }
}

// Static synchronized — locks on Class object
public static synchronized void staticMethod() {
    // locks on MyClass.class — ek hi class-level lock
}
```

### ⚠️ Pitfalls / Gotchas
- `synchronized` on different objects → **no protection** *(alag alag chabi se alag alag darwaza — koi fayda nahi)*
- `synchronized(null)` → `NullPointerException`
- String interning: `synchronized("lock")` — same String object across classes → unexpected sharing *(String pool ki wajah se galat lock share ho sakta hai)*

### 🎯 Tricky Interview Qs
**Q: If thread-1 is in synchronized method-A, can thread-2 enter synchronized method-B of same object?**
> No — both lock on `this`. Thread-2 blocks. *(Dono methods ek hi object pe lock lete hain — ek andar toh doosra bahar)*

**Q: Can a thread enter a synchronized method while another thread is in a non-synchronized method?**
> Yes — non-synchronized methods don't need the lock. *(Bina-lock wala method kisi ka wait nahi karta)*

### ⚡ Remember
1. Locks on **object's monitor** (intrinsic lock)
2. Instance method → locks `this`; Static method → locks `Class`
3. Same lock needed for protection *(alag lock = no protection)*
4. Non-synchronized methods run freely

### 🔗 Follow-ups
→ [Q20. Method vs block sync](#q20) → [Q21. Intrinsic locking](#q21)

---

<a id="q20"></a>
## Q20. What is the difference between synchronized method and synchronized block?

### 📝 One-Liner
> Method locks entire method on 'this'; block locks specific code on any object — block is preferred for finer control.

### 🆚 vs. Comparison
| Feature | Synchronized Method | Synchronized Block |
|---------|-------------------|-------------------|
| Lock object | `this` (instance) or `Class` (static) | **Any object you choose** ✅ |
| Scope | Entire method *(pura method lock)* | Specific lines *(sirf jaruri code lock)* |
| Granularity | Coarse ❌ | Fine ✅ |
| **Preferred** | Simple cases | **Production** ⭐ |

### 💻 Code
```java
// ❌ Synchronized method — locks too much
public synchronized void process() {
    prepareData();         // no need to lock this! (bekaar mein lock)
    updateSharedState();   // only this needs lock
    logResult();           // no need to lock this!
}

// ✅ Synchronized block — lock only what's needed
public void process() {
    prepareData();         // runs freely
    synchronized (this) {
        updateSharedState();  // only critical section locked
    }
    logResult();           // runs freely
}
```

### 🗣️ How to Say in Interview
> *"I always prefer synchronized blocks because they give finer control over what's locked and which object is the lock. Synchronized methods implicitly lock on 'this' for the entire method duration — often locking more code than necessary. With blocks, I lock only the critical section and can use a dedicated private lock object to prevent external code from accidentally using the same lock."*

### ⚠️ Pitfalls / Gotchas
- Synchronized method on `this` → external code can also sync on your object → potential **deadlock**
- Best practice: use **private final Object lock = new Object()** as dedicated lock

### ⚡ Remember
1. Block = **finer control** *(sirf jaruri code lock karo)* ⭐
2. Method = locks entire method *(zyada lock = slow)*
3. Use **private lock object** for safety
4. Less lock time = **better performance**

### 🔗 Follow-ups
→ [Q21. Intrinsic locking](#q21)

---

<a id="q21"></a>
## Q21. What is intrinsic locking / monitor lock?

### 📝 One-Liner
> Every Java object has a built-in lock (monitor) — synchronized uses this hidden lock.

### 🔑 Quick Answer
> Every Java object has an **intrinsic lock** (monitor) stored in its **object header**. `synchronized` uses this lock. When a thread enters synchronized, it **acquires** the lock; when it exits, it **releases** it. *(Har Java object ke andar ek chhupi hui tala hoti hai — synchronized us tala ko use karta hai)*

### 📖 How It Works
```
Object Header (Mark Word — 64 bits):
  ┌──────────────────────────────────────┐
  │ hash code | age | lock state bits    │
  └──────────────────────────────────────┘
  
  Lock states (escalation):
  1. NO LOCK      → object not synchronized
  2. BIASED LOCK  → single thread repeatedly enters (fastest)
                    *(Ek hi thread baar baar aata hai — uske liye fast)*
  3. LIGHTWEIGHT   → CAS-based spinlock (two threads, short waits)
                    *(Do threads, thoda wait — busy spin karta hai)*
  4. HEAVYWEIGHT   → OS mutex (high contention, threads sleep)
                    *(Bahut threads — OS ko bolo manage karo, slow)*

  Escalation: Biased → Lightweight → Heavyweight (never goes back!)
```

### 🗣️ How to Say in Interview
> *"Every Java object has an intrinsic lock in its object header mark word. The JVM optimizes lock acquisition through lock escalation: biased locking for single-thread repeated access which is nearly zero-cost, lightweight locking using CAS for low contention between few threads, and heavyweight locking using OS mutexes for high contention. This escalation is one-directional and happens automatically."*

### ⚠️ Pitfalls / Gotchas
- Lock escalation is **one-way** — heavyweight never goes back to lightweight *(ek baar heavy ho gaya toh heavy hi rahega)*
- Biased locking removed in Java 15+ (JEP 374) — wasn't worth the complexity

### ⚡ Remember
1. Every object has **intrinsic lock** in header *(chhupa hua tala)*
2. Escalation: biased → lightweight → heavyweight
3. **One-way** — never de-escalates
4. Biased = fast for single thread, heavyweight = OS mutex

### 🔗 Follow-ups
→ [Q22. Reentrant locking](#q22)

---

<a id="q22"></a>
## Q22. What is reentrant locking?

### 📝 One-Liner
> Same thread can acquire the same lock multiple times without deadlocking itself.

### 🔑 Quick Answer
> A reentrant lock allows a thread that **already holds the lock** to acquire it again without blocking. It uses a **hold count** — increments on each acquire, decrements on each release. *(Jis thread ke paas chabi hai woh dobara andar ja sakta hai — apne aap se lock nahi hoga)*

### 📖 How It Works
```
Thread-1 calls methodA():
  synchronized(lock)          // hold count = 1
    → calls methodB()
      synchronized(lock)      // hold count = 2 (same thread — allowed!)
      exit                    // hold count = 1
    exit                      // hold count = 0 → lock released

Without reentrancy:
  methodA locks → calls methodB → methodB tries to lock 
  → DEADLOCK! (waiting for itself) 💀
  *(Apne hi lock pe atak jaata — reentrant nahi hota toh problem)*
```

### 🗣️ How to Say in Interview
> *"Both synchronized and ReentrantLock are reentrant — a thread holding the lock can re-acquire it without blocking itself. Internally, a hold count tracks nested acquisitions. This is essential because methods often call other synchronized methods on the same object. Without reentrancy, a thread would deadlock waiting for a lock it already holds."*

### ⚡ Remember
1. Same thread can **re-acquire same lock** *(apne aap se deadlock nahi hoga)*
2. **Hold count** tracks nested entries
3. Both `synchronized` and `ReentrantLock` are reentrant
4. Without it → self-deadlock on nested calls

### 🔗 Follow-ups
→ [Q23. ReentrantLock class](#q23)

---

<a id="q23"></a>
## Q23. What is ReentrantLock?

### 📝 One-Liner
> Explicit lock from java.util.concurrent with tryLock, fairness, timeout, and multiple conditions — more flexible than synchronized.

### 🔑 Quick Answer
> `ReentrantLock` is an explicit lock that offers features `synchronized` doesn't: **tryLock()** *(lock mil raha hai toh lo, nahi toh chodo)*, **fairness** *(FIFO order)*, **interruptible locking**, and **multiple conditions** *(alag alag wait queues)*. Must manually unlock in `finally` block.

### 📖 How It Works
```
synchronized:
  Implicit lock → auto release → no tryLock → no fairness
  *(Simple hai par limited features)*

ReentrantLock:
  Explicit lock → manual unlock → tryLock ✅ → fairness ✅ → conditions ✅
  *(Zyada features — par khud unlock karna padta hai)*
```

### 💻 Code
```java
ReentrantLock lock = new ReentrantLock(true); // true = fair lock

public void criticalSection() {
    lock.lock();  // acquire
    try {
        // critical section
        updateSharedState();
    } finally {
        lock.unlock();  // MUST unlock in finally! (warna deadlock)
    }
}

// tryLock — non-blocking (mil gaya toh karo, nahi toh chhodo)
public boolean tryUpdate() {
    if (lock.tryLock(2, TimeUnit.SECONDS)) {
        try {
            update();
            return true;
        } finally {
            lock.unlock();
        }
    }
    return false;  // couldn't get lock in 2 seconds
}
```

### ⚠️ Pitfalls / Gotchas
- **Forgetting unlock** → permanent deadlock *(finally mein unlock bhool gaye toh forever lock)*
- **Always unlock in finally** — even if exception occurs ⭐
- More verbose than `synchronized` — only use when you need extra features

### 🎯 Tricky Interview Qs
**Q: When would you use ReentrantLock over synchronized?**
> When I need tryLock (avoid waiting), fairness, lockInterruptibly, or multiple conditions. Otherwise synchronized is simpler.

**Q: What happens if you call unlock() without lock()?**
> `IllegalMonitorStateException` *(bina lock kiye unlock nahi kar sakte)*

### ⚡ Remember
1. **tryLock** = don't wait forever *(timeout ke saath try karo)*
2. **fairness** = FIFO order (prevents starvation)
3. **Always unlock in finally** ⭐ *(bhool gaye = deadlock)*
4. Use when synchronized features aren't enough

### 🔗 Follow-ups
→ [Q24. ReentrantLock vs synchronized](#q24)

---

<a id="q24"></a>
## Q24. What is the difference between ReentrantLock and synchronized?

### 📝 One-Liner
> synchronized = simpler, auto-release; ReentrantLock = tryLock, fairness, conditions, manual unlock.

### 🆚 vs. Comparison
| Feature | synchronized | ReentrantLock |
|---------|-------------|--------------|
| Lock/unlock | Auto ✅ | Manual (finally!) *(khud karna padta)* |
| tryLock | ❌ | ✅ *(mil raha hai toh lo)* |
| Timeout | ❌ | ✅ `tryLock(time)` |
| Fairness | ❌ | ✅ `new ReentrantLock(true)` |
| Interruptible | ❌ | ✅ `lockInterruptibly()` |
| Conditions | 1 (wait/notify) | Multiple ✅ *(alag alag wait queues)* |
| Complexity | Simple | More verbose |
| **Default choice** | ✅ ⭐ | When features needed |

### 🗣️ How to Say in Interview
> *"I default to synchronized for simple mutual exclusion — it's cleaner and auto-releases. I switch to ReentrantLock when I need tryLock with timeout to avoid deadlocks, fair ordering for preventing starvation, lockInterruptibly to handle thread cancellation, or multiple conditions for complex producer-consumer patterns. The key risk with ReentrantLock is forgetting to unlock in finally."*

### ⚠️ Pitfalls / Gotchas
- Using ReentrantLock when synchronized is enough → **unnecessary complexity**
- Forgetting finally { unlock() } → builds lock → never released *(galti se bhool = deadlock)*

### ⚡ Remember
1. **Default** → synchronized *(simple aur safe)*
2. **Upgrade** to ReentrantLock when features needed
3. ReentrantLock = **always unlock in finally** ⭐
4. Fair lock: 10-20% slower *(fairness ka cost)*

### 🔗 Follow-ups
→ [Q25. Fair lock](#q25) → [Q26. Read-write lock](#q26)

---

<a id="q25"></a>
## Q25. What is a fair lock?

### 📝 One-Liner
> FIFO ordering — longest-waiting thread gets the lock next; prevents starvation but 10-20% slower.

### 🔑 Quick Answer
> Fair lock grants access in **FIFO order** — whichever thread has been waiting longest gets the lock first. Prevents **starvation** but costs ~10-20% throughput. *(Pehle aaya woh pehle — line tod ke nahi jaa sakte)*

### 💻 Code
```java
ReentrantLock fairLock = new ReentrantLock(true);  // true = fair
ReentrantLock unfairLock = new ReentrantLock();     // default = unfair (faster)
```

### 🆚 vs. Comparison
| | Fair | Unfair (default) |
|-|------|---------|
| Order | FIFO *(line mein khade raho)* | Any *(koi bhi le sakta hai)* |
| Starvation | Prevented ✅ | Possible ⚠️ |
| Throughput | ~10-20% less | Higher ✅ |

### 🗣️ How to Say in Interview
> *"A fair lock uses FIFO ordering — the longest-waiting thread gets it next. I use it when starvation is a concern — like multiple consumers competing for a shared resource where no thread should wait indefinitely. The tradeoff is about 10-20% lower throughput."*

### ⚡ Remember
1. `new ReentrantLock(true)` = fair *(FIFO)*
2. Prevents **starvation** *(bhuka koi nahi rahega)*
3. **10-20% slower** than unfair
4. Default is unfair (faster)

### 🔗 Follow-ups
→ [Q12. Starvation](#q12)

---

<a id="q26"></a>
## Q26. What is a read-write lock?

### 📝 One-Liner
> Multiple threads can read simultaneously, but only one can write — and writing blocks all readers.

### 🔑 Quick Answer
> `ReadWriteLock` allows: **Multiple readers** simultaneously ✅ (no conflict), but **only one writer** AND writer blocks all readers. Perfect for **read-heavy** data structures. *(Padhne wale sab ek saath padho — likhne wala akela likhega, tab koi nahi padhega)*

### 📖 How It Works
```
ReadWriteLock:
  Readers: T-1(read) + T-2(read) + T-3(read) → ALL at once ✅
  Writer:  T-4(write) → ALONE, all readers blocked
  
  *(Sab padh sakte hain ek saath — par likhne wala akela hoga)*

  Read-Read:  SHARED ✅ (koi conflict nahi)
  Read-Write: EXCLUSIVE ❌ (writer wait karega OR reader wait)
  Write-Write: EXCLUSIVE ❌ (ek ek karke)
```

### 💻 Code
```java
ReadWriteLock rwLock = new ReentrantReadWriteLock();

// Multiple threads can read simultaneously
public Data readData() {
    rwLock.readLock().lock();
    try {
        return cache.get(key);  // 100 threads reading = fine! ✅
    } finally {
        rwLock.readLock().unlock();
    }
}

// Only one thread can write, blocks all readers
public void writeData(Data data) {
    rwLock.writeLock().lock();
    try {
        cache.put(key, data);  // exclusive — no readers allowed
    } finally {
        rwLock.writeLock().unlock();
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Write lock **blocks ALL readers** — if writes are frequent, readers starve
- **Lock downgrade** OK: write → read (hold write, acquire read, release write) ✅
- **Lock upgrade** NOT OK: read → write ❌ (deadlock risk) *(padhte padhte likhna nahi — atak jaoge)*

### 🗣️ How to Say in Interview
> *"ReadWriteLock is ideal for read-heavy, write-rare scenarios — like a configuration cache. Multiple reader threads acquire the read lock concurrently with no contention. The write lock is exclusive — it waits for all readers to release, then blocks new readers until writing completes. I've used this for in-memory caches where reads happen thousands of times per second but writes happen only on config refresh."*

### ⚡ Remember
1. **Read-Read** = concurrent ✅ *(sab padho)*
2. **Write** = exclusive *(akela likhega)*
3. Read-heavy → **big performance win**
4. **Downgrade OK**, upgrade NOT OK *(write to read = fine, read to write = danger)*

### 🔗 Follow-ups
→ [Q27. ReentrantReadWriteLock](#q27)

---

<a id="q27"></a>
## Q27. How does ReentrantReadWriteLock work?

### 📝 One-Liner
> Separate read and write locks — readers share, writer is exclusive, supports fairness and reentrance.

### 🔑 Quick Answer
> `ReentrantReadWriteLock` implements `ReadWriteLock` with **separate readLock() and writeLock()** objects. Both are reentrant. Supports fairness mode. Write lock holder can also acquire read lock (downgrade). *(Alag alag tala — ek padhne ke liye, ek likhne ke liye)*

### 💻 Code
```java
ReentrantReadWriteLock rwLock = new ReentrantReadWriteLock(true); // fair

// Lock downgrade pattern: write → read
rwLock.writeLock().lock();
try {
    updateData();
    rwLock.readLock().lock();  // acquire read WHILE holding write ✅
} finally {
    rwLock.writeLock().unlock();  // release write, keep read
}
try {
    return readData();  // still holding read lock
} finally {
    rwLock.readLock().unlock();
}
```

### ⚡ Remember
1. `readLock()` and `writeLock()` are separate objects
2. Both **reentrant** — same thread can re-acquire
3. **Downgrade** (write → read) = safe ✅
4. **Upgrade** (read → write) = deadlock risk ❌

### 🔗 Follow-ups
→ [Q26. Read-write lock concept](#q26) → [Q28. Lock contention](#q28)

---

<a id="q28"></a>
## Q28. What is lock contention?

### 📝 One-Liner
> Multiple threads competing for the same lock — causes waiting, context switching, and poor performance.

### 🔑 Quick Answer
> Lock contention occurs when multiple threads **try to acquire the same lock simultaneously** — most block and wait. High contention = **poor throughput**. *(Sab ek hi darwaze se gusarna chahte hain — bheed lag gayi, slow ho gaya)*

### 📖 How It Works
```
Low contention:
  T-1: [lock][work 100ms][unlock]
  T-2:                          [lock][work][unlock]
  → Little overlap → both run fast ✅

High contention:
  T-1: [lock][────── work 100ms ──────][unlock]
  T-2:       [BLOCKED 100ms...                  ][lock][work][unlock]
  T-3:       [BLOCKED 200ms.....................................][lock]
  → Threads spending more time WAITING than WORKING ❌
  *(Kaam kam, wait zyada — paisa waste)*
```

### 🗣️ How to Say in Interview
> *"Lock contention is when many threads compete for the same lock, spending more time waiting than working. I reduce contention by minimizing synchronized scope — lock only the critical section, not the entire method. I use ConcurrentHashMap instead of synchronized HashMap, LongAdder instead of AtomicLong for high-write counters. If one lock protects too much, I use lock striping — splitting into multiple locks for independent data segments."*

### ⚠️ Pitfalls / Gotchas
- High contention can make multi-threaded code **slower than single-threaded** *(sab wait kar rahe hain — ek thread hi kaam kar raha hai practically)*
- Bigger synchronized blocks = more contention
- Common mistake: synchronizing entire methods unnecessarily

### ⚡ Remember
1. Many threads + same lock = **contention** *(bheed)*
2. Fix: **minimize lock scope** *(kam code lock karo)*
3. Fix: **lock-free** structures (ConcurrentHashMap, LongAdder)
4. Fix: **lock striping** (multiple locks for different segments)
5. High contention = worse than single-threaded *(sab ruk gaye)*

### 🔗 Follow-ups
→ [Q76. Lock striping](#q76) → [Q74. Performance improvement](#q74)

---

<a id="q29"></a>
## Q29. Can we synchronize static methods in Java?

### 📝 One-Liner
Yes — synchronizing a static method acquires the **class-level lock** (`ClassName.class`) instead of an object-level lock.

### 🔑 Quick Answer
Every class has a unique lock associated with the `Class` object. `synchronized static void method()` acquires this class lock. While a thread holds the class lock, no other thread can execute any other `synchronized static` method of that class. But they CAN execute: (1) Normal static methods. (2) Normal instance methods. (3) Synchronized instance methods (different lock). *(Static synchronized = class ka lock | Object lock alag, class lock alag)*

### 💻 Code Example

```java
class Counter {
    static int count = 0;
    static synchronized void increment() {  // ⭐ class-level lock
        count++;
    }
    // Equivalent to:
    static void incrementV2() {
        synchronized (Counter.class) {  // explicit class lock
            count++;
        }
    }
}
```

### ⚡ Remember
`synchronized static = class lock (ClassName.class) | Object lock is separate | Both can coexist`

---

<a id="q30"></a>
## Q30. Can we use synchronized block for primitives?

### 📝 One-Liner
No — `synchronized` requires an **object reference** as the lock. Primitives (`int`, `char`, etc.) cause a compile error.

### 🔑 Quick Answer
`synchronized(primitiveVar)` → compile error. Synchronized blocks work only with object references. If you need to synchronize on an int-like value, use `Integer` wrapper or a dedicated `Object lock = new Object()`. *(synchronized block sirf objects pe kaam karta hai, primitives pe compile error aayega)*

### ⚡ Remember
`synchronized(primitive) = compile error | Only object references | Use wrapper or dedicated lock object`

---

> **🎯 Navigation:** [← Basics (Q1-17)](01-basics.md) | [Next → Thread Communication (Q29-36)](03-thread-communication.md) | [📋 All Sections](README.md)
