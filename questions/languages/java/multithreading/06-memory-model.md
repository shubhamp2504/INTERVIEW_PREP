# 🧠 Memory Model & Visibility (Q57–Q62)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q57"></a>

## Q57. What is the Java Memory Model (JMM)?

### 🔑 Quick Answer

> The JMM defines **rules for how threads see changes made by other threads** to shared variables. It specifies when writes by one thread become **visible** to reads by another thread through **happens-before** relationships.

### 📖 Step-by-Step Explanation

```
Without JMM guarantees:
  Thread-1: x = 42;   flag = true;
  Thread-2: if (flag) print(x);  → might print 0! 😱

Why?
  1. CPU caches: Each CPU core has its own cache → Thread-2 may see stale value
  2. Compiler reordering: JIT may reorder x=42 and flag=true
  3. Store buffers: Write may sit in CPU store buffer, not yet flushed

JMM says:
  "Without proper synchronization, there are NO guarantees about
   when (or IF) another thread sees your writes."

Solution → happens-before relationships:
  - synchronized block entry/exit
  - volatile read/write
  - Thread.start() / Thread.join()
  - These FORCE visibility across threads
```

```
Java Memory Architecture:

  Thread-1                    Thread-2
  ┌──────────┐               ┌──────────┐
  │ CPU Cache │               │ CPU Cache │
  │  x = 42  │               │  x = 0   │  ← stale!
  └────┬─────┘               └────┬─────┘
       │                          │
       ▼          Flush           ▼
  ═══════════════════════════════════════
  │         MAIN MEMORY (Heap)          │
  │              x = 42                 │
  ═══════════════════════════════════════
       
  Without volatile/synchronized, Thread-2 
  may NEVER see x = 42
```

### 🗣️ How to Explain in Interview

> *"The Java Memory Model defines the visibility guarantees between threads. Without synchronization, a write by one thread is NOT guaranteed to be visible to another thread — because each CPU core has its own cache, and the compiler can reorder instructions. The JMM uses happens-before relationships to establish visibility: if action A happens-before action B, then the effects of A are guaranteed to be visible to B. synchronized blocks, volatile variables, Thread.start(), and Thread.join() create happens-before edges. Understanding the JMM is critical for writing correct concurrent code."*

### ⚡ Key Points to Remember

1. **JMM** = rules for cross-thread visibility
2. **Without synchronization** = no visibility guarantee
3. **CPU caches** + **compiler reordering** = root causes
4. **happens-before** = the formal guarantee mechanism
5. **synchronized, volatile, Thread.start/join** create happens-before edges

---

<a id="q58"></a>

## Q58. What is the volatile keyword?

### 🔑 Quick Answer

> `volatile` guarantees **visibility** and **ordering** for a single variable across threads. Every read goes to **main memory** (not cache), and every write flushes to main memory immediately. Does NOT guarantee atomicity of compound operations.

### 📖 Step-by-Step Explanation

```
Without volatile:
  private boolean running = true;

  Thread-1: running = false;       // writes to CPU-1 cache
  Thread-2: while (running) { }    // reads from CPU-2 cache → INFINITE LOOP!

With volatile:
  private volatile boolean running = true;

  Thread-1: running = false;       // writes directly to main memory
  Thread-2: while (running) { }    // reads from main memory → sees false → exits ✅
```

**What volatile guarantees:**
```
✅ Visibility: All threads see the latest value immediately
✅ Ordering: Prevents reordering of reads/writes around volatile access
❌ Atomicity: volatile does NOT make count++ atomic!
```

**volatile does NOT work for:**
```java
private volatile int count = 0;

// Thread-1: count++; → read(0) → add(0+1=1) → write(1)
// Thread-2: count++; → read(0) → add(0+1=1) → write(1)
// Expected: 2, Actual: 1 → RACE CONDITION! ❌

// Use AtomicInteger instead:
private AtomicInteger count = new AtomicInteger(0);
count.incrementAndGet(); // Atomic! ✅
```

### 💻 Code Example

```java
// Classic use: shutdown flag
public class Worker implements Runnable {
    private volatile boolean running = true;

    @Override
    public void run() {
        while (running) {   // Always reads latest value from main memory
            doWork();
        }
    }

    public void shutdown() {
        running = false;    // Immediately visible to all threads
    }
}
```

### 🗣️ How to Explain in Interview

> *"volatile ensures visibility — every write goes to main memory, every read comes from main memory, not the CPU cache. It also prevents instruction reordering around volatile accesses. The classic use case is a boolean flag like 'running' that one thread sets to false and another thread checks in a loop. But volatile does NOT guarantee atomicity — count++ is still a race condition because it's three operations: read, increment, write. For atomic compound operations, I use AtomicInteger or synchronized."*

### ⚡ Key Points to Remember

1. **Visibility** guaranteed — reads/writes go to main memory
2. **Ordering** guaranteed — no reordering around volatile access
3. **Atomicity NOT** guaranteed — count++ still broken
4. Best for: **flags, status variables, published references**
5. For compound ops: use **AtomicInteger** or **synchronized**

---

<a id="q59"></a>

## Q59. Difference between volatile and synchronized?

### 🔑 Quick Answer

| Feature | volatile | synchronized |
|---------|----------|-------------|
| **Visibility** | ✅ Yes | ✅ Yes |
| **Atomicity** | ❌ No | ✅ Yes |
| **Mutual exclusion** | ❌ No | ✅ Yes (one thread at a time) |
| **Scope** | Single variable | Block of code |
| **Performance** | Faster (no locking) | Slower (lock overhead) |
| **Use case** | Flags, simple reads/writes | Compound operations |

### 📖 Step-by-Step Explanation

```
volatile:
  Thread-1: flag = true;    → visible to Thread-2 ✅
  Thread-2: if (flag) ...   → sees latest value ✅
  
  BUT:
  Thread-1: count++;        → NOT atomic ❌
  Thread-2: count++;        → lost update ❌

synchronized:
  Thread-1: synchronized(lock) { count++; }  → only one thread at a time ✅
  Thread-2: synchronized(lock) { count++; }  → waits, then runs ✅
  
  → Atomic compound operations ✅
  → Visibility guaranteed (on lock release → all writes flushed) ✅
```

### 🗣️ How to Explain in Interview

> *"volatile provides visibility without locking — like a lightweight synchronization for single-variable reads and writes. synchronized provides both visibility AND mutual exclusion — it ensures only one thread executes the block at a time, making compound operations atomic. I use volatile for simple state flags — like a shutdown boolean or a published configuration reference. I use synchronized or locks when I need to do read-modify-write — like incrementing a counter or checking and updating a map entry."*

### ⚡ Key Points to Remember

1. **volatile** = visibility only, no atomicity, no locking
2. **synchronized** = visibility + atomicity + mutual exclusion
3. **volatile** is faster (no lock contention)
4. Use volatile for: **simple flags/references**
5. Use synchronized for: **compound/multi-step operations**

---

<a id="q60"></a>

## Q60. What is the happens-before relationship?

### 🔑 Quick Answer

> A formal guarantee in the JMM: if action A **happens-before** action B, then all effects of A are **visible** to B. It's the foundation for understanding thread safety — without a happens-before edge, there's no visibility guarantee.

### 📖 Step-by-Step Explanation

```
Happens-before rules (most important):

1. Program order:     Within a thread, earlier statements happen-before later ones
2. Monitor lock:      unlock(m) happens-before subsequent lock(m)
3. Volatile:          volatile write happens-before subsequent volatile read
4. Thread start:      Thread.start() happens-before any action in started thread
5. Thread join:       All actions in thread happen-before join() returns
6. Transitivity:      If A h-b B, and B h-b C, then A h-b C
```

```
Example: How synchronized creates happens-before:

  Thread-1:                        Thread-2:
  x = 42;                         
  synchronized(lock) {     ──→     synchronized(lock) {
    flag = true;                     if (flag) {
  } // unlock happens-before         print(x); // GUARANTEED 42 ✅
                                   } // lock happens-after
```

```
Example: How volatile creates happens-before:

  Thread-1:              Thread-2:
  x = 42;               
  volatile_flag = true;  ──→  if (volatile_flag) {
                                print(x); // GUARANTEED 42 ✅
  // All writes BEFORE volatile write are visible
  // to any thread that reads the volatile variable
```

### 🗣️ How to Explain in Interview

> *"Happens-before is the JMM's formal guarantee for visibility. If action A happens-before action B, then everything A wrote is visible when B executes. The key rules: monitor unlock happens-before the next lock — so when I exit a synchronized block, the next thread entering that block sees all my writes. Volatile write happens-before the next volatile read — and importantly, ALL writes before the volatile write become visible, not just the volatile variable itself. Thread.start() makes everything before start() visible to the new thread. Thread.join() makes everything in the joined thread visible after join returns."*

### ⚡ Key Points to Remember

1. **Happens-before** = formal visibility guarantee
2. **synchronized** = unlock h-b lock on same monitor
3. **volatile** = write h-b read of same variable (+ all prior writes!)
4. **Thread.start/join** = create h-b edges
5. **Transitive**: A h-b B, B h-b C → A h-b C

---

<a id="q61"></a>

## Q61. What is instruction reordering?

### 🔑 Quick Answer

> The **compiler** (JIT) and **CPU** may reorder instructions for performance **as long as single-threaded behavior is preserved**. But in multi-threaded code, this can cause other threads to **see operations in unexpected order**, leading to bugs.

### 📖 Step-by-Step Explanation

```
Original code:
  int x = 1;    // (1)
  int y = 2;    // (2)
  flag = true;  // (3)

Compiler/CPU may reorder to:
  flag = true;  // (3) ← moved up!
  int x = 1;    // (1)
  int y = 2;    // (2)

Single thread: Same result (no dependency between these statements)
Multi-thread:  Thread-2 sees flag=true but x and y are NOT yet written! 💀
```

**The "Double-Checked Locking" problem (classic interview example):**

```java
// BROKEN without volatile:
private static Singleton instance;

public static Singleton getInstance() {
    if (instance == null) {           // (1) check
        synchronized (Singleton.class) {
            if (instance == null) {   // (2) double-check
                instance = new Singleton(); // (3) create
                // JVM may reorder:
                //   a. Allocate memory
                //   b. Assign reference to instance  ← OTHER THREAD SEES non-null!
                //   c. Call constructor               ← NOT YET DONE!
            }
        }
    }
    return instance;
}

// FIXED with volatile:
private static volatile Singleton instance;
// volatile prevents reordering of constructor and reference assignment
```

### 🗣️ How to Explain in Interview

> *"Instruction reordering is an optimization by the JIT compiler and CPU — they rearrange instructions for performance, as long as single-threaded behavior is unchanged. But in multi-threaded code, this causes bugs. The classic example is double-checked locking: without volatile, the JVM might assign the reference before completing the constructor — so another thread sees a non-null but partially constructed object. volatile prevents this by adding memory barriers that prevent reordering. Generally, synchronized blocks and volatile variables establish ordering boundaries that prevent harmful reordering."*

### ⚡ Key Points to Remember

1. **Compiler (JIT) + CPU** both reorder instructions
2. **Single-threaded** semantics preserved, but **multi-thread** breaks
3. **volatile** inserts **memory barriers** to prevent reordering
4. Classic bug: **Double-Checked Locking** without volatile
5. **synchronized** also prevents reordering (within critical section)

---

<a id="q62"></a>

## Q62. What is the visibility problem?

### 🔑 Quick Answer

> When one thread's writes are **not seen** by another thread because the value is cached in the CPU cache and not flushed to main memory. This is the most **common and subtle concurrency bug**.

### 📖 Step-by-Step Explanation

```
The visibility problem in action:

  Main Memory:   running = true

  CPU Core 1 (Thread-1):          CPU Core 2 (Thread-2):
  ┌──────────────────┐            ┌──────────────────┐
  │ Cache: running=true│           │ Cache: running=true│
  │                    │           │                    │
  │ running = false;   │           │ while(running) {   │
  │ Cache: running=false│          │   // reads cache   │
  │ (NOT flushed!)    │           │   // still true!   │
  └──────────────────┘            │   // INFINITE LOOP │
                                  └──────────────────┘

  Thread-1 wrote false to its cache, but Thread-2 never sees it!
```

**Demonstration:**

```java
// This program may NEVER terminate!
public class VisibilityBug {
    private static boolean running = true;  // ← NOT volatile!

    public static void main(String[] args) throws Exception {
        Thread worker = new Thread(() -> {
            while (running) {  // JIT may hoist: if (running) while(true)
                // do work
            }
            System.out.println("Stopped");  // May NEVER print!
        });
        worker.start();

        Thread.sleep(1000);
        running = false;  // Thread-1 writes, but worker may never see it
        System.out.println("Flag set to false");
    }
}
```

**Three fixes:**

```java
// Fix 1: volatile
private static volatile boolean running = true;

// Fix 2: synchronized
synchronized(lock) { running = false; }  // write
synchronized(lock) { while(running) {} } // read

// Fix 3: AtomicBoolean
private static AtomicBoolean running = new AtomicBoolean(true);
```

### 🗣️ How to Explain in Interview

> *"The visibility problem is when one thread writes to a variable but another thread can't see the change. This happens because modern CPUs have per-core caches — a write may sit in CPU core 1's cache and never be flushed to main memory where CPU core 2 can see it. The JIT compiler makes it worse — it can hoist the variable read out of a loop, turning while(running) into if(running) while(true). The fix is volatile, which forces every read and write through main memory, or synchronized, which flushes all writes on lock release and refreshes all reads on lock acquire."*

### ⚡ Key Points to Remember

1. **CPU caches** = each core has private cache
2. **JIT hoisting** = compiler optimization reads once, caches forever
3. **volatile** = forces main memory read/write
4. **synchronized** = flushes on unlock, refreshes on lock
5. This is the **#1 subtle concurrency bug** in Java

---

> **🎯 Navigation:** [← Concurrent Collections (Q51-56)](05-concurrent-collections.md) | [Next → Deadlock & Problems (Q63-69)](07-deadlock-problems.md) | [📋 All Sections](README.md)
