# 🧵 Core Multithreading Basics (Q1–Q17)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. What is multithreading in Java?

### 📝 One-Liner
> Multiple threads running concurrently inside a single process, sharing the same memory space.

### 🔑 Quick Answer
> Multithreading means executing **multiple threads simultaneously** within one process. Each thread has its own **stack** but shares the **heap** *(matlab sabhi threads ek hi memory share karte hain)*. Java supports multithreading natively via `Thread` class and `Runnable` interface.

### 📖 How It Works
```
Process (JVM):
  ┌─────────────────────────────────────────┐
  │ HEAP (shared)  ← objects, instances     │
  │           (sabhi threads ka ek hi heap) │
  ├───────────┬───────────┬────────────────┤
  │ Thread-1  │ Thread-2  │ Thread-3       │
  │ own Stack │ own Stack │ own Stack      │
  │ own PC    │ own PC    │ own PC         │
  └───────────┴───────────┴────────────────┘

Thread = lightweight sub-process
  *(Process ke andar chhote kaam karne wale workers)*

Single-threaded:  Task-A ──→ Task-B ──→ Task-C  (slow, sequential)
Multi-threaded:   Task-A ──→
                  Task-B ──→  (fast, parallel!)
                  Task-C ──→
```

### 🗣️ How to Say in Interview
> *"Multithreading is running multiple threads within a single JVM process. Each thread has its own call stack and program counter, but they all share the heap memory — that's why shared data access needs synchronization. In my project, we used multithreading for parallel API calls to three downstream services — reduced response time from 650ms to 220ms by calling them concurrently using CompletableFuture."*

### 💻 Code
```java
// Two ways to create threads
// Way 1: Extend Thread
class MyThread extends Thread {
    public void run() {
        System.out.println(Thread.currentThread().getName() + " running");
    }
}

// Way 2: Implement Runnable (preferred — ek aur class extend kar sakte ho)
class MyTask implements Runnable {
    public void run() {
        System.out.println(Thread.currentThread().getName() + " running");
    }
}

// Usage
new MyThread().start();
new Thread(new MyTask()).start();
new Thread(() -> System.out.println("Lambda thread")).start(); // Java 8+
```

### ⚠️ Pitfalls / Gotchas
- Calling `run()` instead of `start()` → runs on **same thread** *(naya thread nahi banta!)*
- Shared state without sync → **race conditions** *(do threads ek hi variable badal dete hain)*
- Too many threads → **OutOfMemoryError** *(har thread ~1MB stack leta hai)*
- Thread creation is expensive → use **thread pools** instead

### 🎯 Tricky Interview Qs
**Q: Can we start a thread twice?**
> No — `IllegalThreadStateException`. Once TERMINATED, it's dead. *(Ek baar band ho gaya toh wapas start nahi hoga)*

**Q: Is Java multithreading truly parallel?**
> Only on multi-core CPUs. On single-core, it's **concurrent** (time-slicing), not parallel. *(Ek core pe bari bari chalta hai, multiple cores pe sach mein ek saath)*

### ⚡ Remember
1. Thread = own stack + shared heap *(apna stack, sab ka heap)*
2. Always prefer `Runnable` over extending `Thread`
3. `start()` ≠ `run()` — start creates new thread *(start = naya thread, run = wahi thread)*
4. Each thread ~1MB stack → don't create thousands manually
5. Use **ExecutorService** in production, never raw threads

### 🔗 Follow-ups
→ [Q2. Process vs Thread](#q2) → [Q4. Thread lifecycle](#q4) → [Q5. Thread vs Runnable](#q5)

---

<a id="q2"></a>
## Q2. What is the difference between a process and a thread?

### 📝 One-Liner
> Process = independent program with own memory; Thread = lightweight unit within process sharing memory.

### 🔑 Quick Answer
| Feature | Process | Thread |
|---------|---------|--------|
| Memory | Own memory space *(apna alag memory)* | Shares heap *(process ka memory share karta hai)* |
| Creation | Heavy (~100ms) | Light (~1ms) |
| Communication | IPC (pipes, sockets) *(mushkil)* | Direct via shared variables *(aasan)* |
| Crash impact | Other processes safe | Can crash entire process |
| Context switch | Expensive | Cheap |

### 📖 How It Works
```
OS Level:
  Process-1 (JVM-1)         Process-2 (JVM-2)
  ┌──────────────────┐      ┌──────────────────┐
  │ Own Heap          │      │ Own Heap          │
  │ Own Code          │      │ Own Code          │
  │ Thread-A Thread-B │      │ Thread-X Thread-Y │
  │ (share this heap) │      │ (share this heap) │
  └──────────────────┘      └──────────────────┘
  ← Completely isolated →    ← Completely isolated →
  
Inter-process: Slow (network/pipes)
  *(Do alag programs ke beech baat karna mushkil)*
Inter-thread:  Fast (shared memory)
  *(Ek program ke andar threads seedha heap se baat karte hain)*
```

### 🗣️ How to Say in Interview
> *"A process is an independent execution unit with its own memory — like two separate JVMs running. A thread is a lightweight unit inside a process — threads share the heap but have their own stack. Communication between threads is fast via shared memory, while inter-process needs IPC mechanisms. The tradeoff is that a crash in one thread can bring down the whole process, while processes are isolated."*

### ⚠️ Pitfalls / Gotchas
- "Threads are always faster" — **wrong**. For CPU-bound work on single core, threads add overhead *(context switching ka kharch)*
- Thread sharing memory = **convenience + danger** — easy to get race conditions

### 🆚 vs. Comparison
| Scenario | Use Process | Use Thread |
|----------|------------|-----------|
| Isolation needed *(ek crash doosre ko na mare)* | ✅ | ❌ |
| Fast communication needed | ❌ | ✅ |
| Microservices | ✅ (separate JVMs) | ❌ |
| Parallel tasks in one app | ❌ | ✅ |

### ⚡ Remember
1. Process = **heavy, isolated, safe**
2. Thread = **light, shared memory, risky if not synced**
3. Thread crash → **whole process dies** *(ek thread fail = pura app fail)*
4. Java threads are **OS-level threads** (not green threads)

### 🔗 Follow-ups
→ [Q1. What is multithreading](#q1) → [Q17. Context switching](#q17)

---

<a id="q3"></a>
## Q3. What are the advantages of multithreading?

### 📝 One-Liner
> Better CPU utilization, responsive apps, parallel I/O, and shared memory communication.

### 🔑 Quick Answer
> 1. **Better CPU utilization** *(CPU idle nahi baithta — ek thread wait kare toh doosra kaam kare)*
> 2. **Responsive UI/API** — background processing doesn't block main thread
> 3. **Parallel I/O** — call 5 APIs simultaneously instead of sequentially
> 4. **Shared memory** — threads communicate faster than processes

### 📖 How It Works
```
Without multithreading:
  API-A → 200ms → API-B → 300ms → API-C → 150ms = 650ms total

With multithreading:
  API-A → 200ms ─╮
  API-B → 300ms ─┼─→ max = 300ms total! ⭐
  API-C → 150ms ─╯
  
  Time saved: 650 - 300 = 350ms (54% faster!)
  *(Teen kaam ek saath karo, sabse slow wala time lagega — baaki free mein)*
```

### 🗣️ How to Say in Interview
> *"The main advantage is better resource utilization — while one thread waits for a database response, the CPU runs another thread. In my project, we parallelized three downstream API calls using CompletableFuture — cut response time from 650ms to 300ms. The second big advantage is responsiveness — we process heavy operations like report generation in background threads, returning 'accepted' immediately to the user."*

### ⚠️ Pitfalls / Gotchas
- Multithreading is NOT always faster — overhead of thread creation, context switching, synchronization
- **Amdahl's Law**: speedup limited by the sequential portion of code *(agar 50% code sequential hai toh max 2x fast hoga, chahe 100 threads lagao)*
- Debugging multithreaded code is **10x harder** — bugs are non-deterministic

### ⚡ Remember
1. **Parallel I/O** = biggest real-world win
2. **Responsive apps** = background processing
3. Not always faster — **overhead exists**
4. Debugging complexity = major disadvantage

### 🔗 Follow-ups
→ [Q72. When to use multithreading](#q72) → [Q73. When to avoid](#q73)

---

<a id="q4"></a>
## Q4. What are the different states in a thread lifecycle?

### 📝 One-Liner
> 6 states: NEW → RUNNABLE → RUNNING → BLOCKED/WAITING/TIMED_WAITING → TERMINATED.

### 🔑 Quick Answer
> Java `Thread.State` enum has **6 states** *(thread ki 6 alag stages hoti hain)*:

| State | Meaning | Hindi |
|-------|---------|-------|
| **NEW** | Created, not started | *Thread bana, start nahi hua* |
| **RUNNABLE** | Ready to run / running | *CPU ke liye ready / chal raha hai* |
| **BLOCKED** | Waiting for monitor lock | *Koi lock pakde baitha hai, ye wait kar raha* |
| **WAITING** | Waiting indefinitely | *Jab tak koi jagaye nahi, soyega* |
| **TIMED_WAITING** | Waiting with timeout | *Time limit ke saath so raha hai* |
| **TERMINATED** | Finished execution | *Kaam khatam, ab nahi chalega* |

### 📖 How It Works
```
        new Thread()
             │
             ▼
          ┌─────┐
          │ NEW │
          └──┬──┘
     start() │
             ▼
        ┌──────────┐  ← Scheduler picks  ──→  [RUNNING on CPU]
        │ RUNNABLE │  
        └────┬─────┘
             │
    ┌────────┼──────────┬─────────────┐
    ▼        ▼          ▼             ▼
 BLOCKED  WAITING  TIMED_WAITING  TERMINATED
 (lock)   (wait)   (sleep/timeout)  (done)
    │        │          │
    └────────┴──────────┘
             │
             ▼
        ┌──────────┐
        │ RUNNABLE │ ← goes back when unblocked
        └──────────┘
```

### 🗣️ How to Say in Interview
> *"Java thread has 6 states defined in Thread.State enum. NEW is when the thread is created but start() hasn't been called. RUNNABLE means it's either running or ready to be picked by the thread scheduler. BLOCKED is when it's waiting to enter a synchronized block — the monitor is held by another thread. WAITING is indefinite waiting used by wait() or join() without timeout. TIMED_WAITING is waiting with a deadline — like sleep(1000) or wait(5000). TERMINATED means the run() method completed or an uncaught exception occurred."*

### ⚠️ Pitfalls / Gotchas
- **RUNNABLE ≠ running** — it means "eligible to run" *(CPU mil sakta hai, but jaruri nahi ki abhi mil raha hai)*
- Java has **no separate RUNNING** state in the enum — it's part of RUNNABLE
- A TERMINATED thread **cannot restart** — `start()` throws `IllegalThreadStateException`
- `sleep()` → TIMED_WAITING, but `wait()` → WAITING *(dono alag state mein jaate hain)*

### 🎯 Tricky Interview Qs
**Q: What's the difference between BLOCKED and WAITING?**
> BLOCKED = waiting for monitor lock (synchronized). WAITING = explicitly called wait()/join(). *(BLOCKED = lock chahiye; WAITING = khud se soya hai)*

**Q: Can a thread go from BLOCKED → WAITING directly?**
> No. Goes BLOCKED → RUNNABLE → then can call wait() to go WAITING.

### ⚡ Remember
1. **6 states**: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED
2. RUNNABLE = ready OR running *(CPU assign ho bhi sakta hai, nahi bhi)*
3. BLOCKED = waiting for **lock** *(lock chahiye)*
4. WAITING = waiting for **notification** *(koi jagaye)*
5. TERMINATED = **dead, no restart** *(wapas start nahi hoga)*

### 🔗 Follow-ups
→ [Q10. sleep vs wait vs yield](#q10) → [Q11. wait vs sleep](#q11)

---

<a id="q5"></a>
## Q5. What is the difference between Thread and Runnable?

### 📝 One-Liner
> Thread = class (single inheritance used up); Runnable = interface (preferred — task separate from thread).

### 🔑 Quick Answer
> `Runnable` is **always preferred** because Java has single inheritance — extending `Thread` wastes your one extends slot. Runnable separates **task** from **thread**, making code reusable with thread pools. *(Runnable use karo — inheritance mat gawao)*

### 🆚 vs. Comparison
| Feature | extends Thread | implements Runnable |
|---------|---------------|-------------------|
| Inheritance | Used up ❌ *(aur extend nahi kar sakte)* | Free ✅ |
| Separation of concerns | Task tied to thread ❌ | Task is independent ✅ |
| Thread pool compatible | ❌ | ✅ `executor.submit(runnable)` |
| Code reuse | Low | High |
| **Verdict** | Avoid | **Always use** ⭐ |

### 💻 Code
```java
// ❌ Don't — wastes inheritance
class MyThread extends Thread {
    public void run() { /* task */ }
}

// ✅ Do — task is separate
class MyTask implements Runnable {
    public void run() { /* task */ }
}

// Best: Lambda (Java 8+)
Runnable task = () -> System.out.println("clean");
executor.submit(task);  // pool mein do — reusable
```

### 🗣️ How to Say in Interview
> *"I always use Runnable over extending Thread. Two reasons: First, Java only allows single inheritance — extending Thread means I can't extend any other class. Second, Runnable separates the task from the threading mechanism — I can submit the same Runnable to different executors, test it independently, and reuse it. In Java 8+, I use lambdas for simple tasks and Callable when I need a return value."*

### ⚠️ Pitfalls / Gotchas
- Extending Thread and accidentally overriding other methods → unexpected behavior
- `Runnable` has no return value — use `Callable<V>` if you need one

### ⚡ Remember
1. **Runnable** = always preferred *(inheritance mat gawao)*
2. Task separate from thread → **testable, reusable**
3. Lambda for simple tasks: `() -> doWork()`
4. Need return value? → **Callable\<V\>**

### 🔗 Follow-ups
→ [Q6. Runnable vs Callable](#q6)

---

<a id="q6"></a>
## Q6. What is the difference between Runnable and Callable?

### 📝 One-Liner
> Runnable = void, no exceptions; Callable = returns value + throws checked exceptions.

### 🆚 vs. Comparison
| Feature | Runnable | Callable\<V\> |
|---------|----------|--------------|
| Return | void *(kuch nahi deta)* | V *(result deta hai)* |
| Exceptions | No checked exceptions | Can throw ✅ |
| Method | `run()` | `call()` |
| Result fetch | — | Via `Future<V>` |

### 💻 Code
```java
// Runnable — fire and forget
Runnable task = () -> System.out.println("no result");

// Callable — returns result
Callable<Integer> task = () -> expensiveComputation();

Future<Integer> future = executor.submit(task);
Integer result = future.get(5, TimeUnit.SECONDS); // timeout lagao!
```

### ⚠️ Pitfalls / Gotchas
- `future.get()` **blocks** — can hang forever *(timeout lagao warna atak jaoge)*
- Use `CompletableFuture` for non-blocking chains

### 🗣️ How to Say in Interview
> *"Runnable is for fire-and-forget — no return. Callable returns a result via Future. In production, I prefer CompletableFuture.supplyAsync() over raw Callable + Future because it supports chaining and non-blocking callbacks. When using raw Future, I always add a timeout to get()."*

### ⚡ Remember
1. Runnable = `void run()` | Callable = `V call()`
2. Callable throws checked exceptions
3. Always **timeout with future.get()** *(warna hang)*
4. Prefer **CompletableFuture** in production

### 🔗 Follow-ups
→ [Q43. Future](#q43) → [Q44. CompletableFuture](#q44)

---

<a id="q7"></a>
## Q7. What happens when you call start() vs run()?

### 📝 One-Liner
> start() = new OS thread created; run() = plain method call on current thread.

### 🔑 Quick Answer
> `start()` creates a **new OS thread** and calls `run()` on it. Calling `run()` directly executes on **same thread**. *(start = naya thread; run = wahi purana thread)*

### 📖 How It Works
```
thread.start():
  main ──→ creates new thread ──→ new thread calls run()
  main continues (parallel)

thread.run():
  main ──→ main calls run() ──→ main continues (sequential)
  *(Koi naya thread nahi bana — bas normal method call)*
```

### 💻 Code
```java
Thread t = new Thread(() -> 
    System.out.println("Thread: " + Thread.currentThread().getName()));

t.start();  // "Thread: Thread-0"  ← naya thread
t.run();    // "Thread: main"      ← wahi main thread
```

### ⚠️ Pitfalls / Gotchas
- Very common beginner mistake — calling `run()`
- `start()` only **once** — second call → `IllegalThreadStateException`

### 🎯 Tricky Interview Qs
**Q: What happens if you call start() twice?**
> `IllegalThreadStateException` *(ek thread ek baar hi start hoga)*

**Q: Is calling run() directly ever useful?**
> For testing — you can call run() on the test thread for deterministic results.

### ⚡ Remember
1. `start()` = **new thread** *(naya thread banta hai)*
2. `run()` = **same thread** *(sirf method call)*
3. `start()` only **once** — else exception
4. Never call `run()` in production

### 🔗 Follow-ups
→ [Q8. What if run() called directly?](#q8)

---

<a id="q8"></a>
## Q8. What happens if you call run() directly?

### 📝 One-Liner
> No new thread — runs as a normal method on the calling thread.

### 🔑 Quick Answer
> Executes on **current thread** like a normal method. No concurrency. *(Naya thread nahi banega — seedha wahi thread pe chalega)*

### 💻 Code
```java
Thread t = new Thread(() -> 
    System.out.println(Thread.currentThread().getName()));

t.run();   // prints "main"     ← WRONG (no new thread)
t.start(); // prints "Thread-0" ← CORRECT (new thread)
```

### ⚡ Remember
1. `run()` = plain method call *(koi naya thread nahi)*
2. Must use `start()` for multithreading
3. Classic interview trick question

### 🔗 Follow-ups
→ [Q7. start() vs run()](#q7)

---

<a id="q9"></a>
## Q9. What is thread scheduling in Java?

### 📝 One-Liner
> OS decides which thread runs when — Java uses preemptive scheduling.

### 🔑 Quick Answer
> Java uses **preemptive scheduling** — the OS can interrupt any thread to give CPU to another. Priority is a **hint, not guarantee**. *(OS decide karta hai kisko CPU milega — programmer ke haath mein nahi)*

### 📖 How It Works
```
Time-slicing (preemptive):
  CPU: [T-1][T-2][T-1][T-3][T-2]
       ├─10ms┤├─10ms┤├─10ms┤├─10ms┤
  *(Har thread ko thoda time milta hai, phir next)*
  
  Priority: 1 (MIN) ← 5 (NORM) → 10 (MAX)
  Higher = MORE LIKELY to get CPU (not guaranteed!)
```

### ⚠️ Pitfalls / Gotchas
- **Never rely on priority** for correctness — behavior is OS-dependent *(Windows pe alag, Linux pe alag)*
- `Thread.yield()` is just a **hint** — OS can ignore it

### 🗣️ How to Say in Interview
> *"Java relies on the OS for thread scheduling, which is preemptive. Thread priorities are hints but aren't guaranteed. In production, I never rely on priority or scheduling order — I use proper synchronization like CountDownLatch or CompletableFuture to coordinate threads."*

### ⚡ Remember
1. **OS controls** scheduling, not Java
2. Priority = **hint**, not guarantee *(OS ki marzi)*
3. Default priority = 5 (NORM_PRIORITY)
4. **Never rely** on scheduling order

### 🔗 Follow-ups
→ [Q10. sleep vs wait vs yield](#q10)

---

<a id="q10"></a>
## Q10. What is the difference between sleep(), wait(), and yield()?

### 📝 One-Liner
> sleep = pause with lock held; wait = release lock and pause; yield = hint to give up CPU.

### 🆚 vs. Comparison
| Feature | sleep() | wait() | yield() |
|---------|---------|--------|---------|
| **Lock** | ❌ Holds *(lock nahi chhodta!)* | ✅ Releases *(lock chhod deta hai)* | ❌ Holds |
| **Class** | Thread | Object | Thread |
| **Resumes** | Time expires | notify()/notifyAll() | Immediately (maybe) |
| **Needs sync?** | No | Yes *(warna error)* | No |
| **State** | TIMED_WAITING | WAITING | RUNNABLE |
| **Use** | Delay | Thread communication | Useless in practice |

### 📖 How It Works
```
sleep(1000):
  Thread: [RUNNING] → sleep → [TIMED_WAITING 1s] → [RUNNABLE]
  Lock: STILL HELD ← (dusre threads block rahenge!)

wait():
  Thread: [RUNNING] → wait → releases lock → [WAITING]
  Other threads can enter synchronized block now ✅
  notify() → [BLOCKED (re-acquire lock)] → [RUNNABLE]

yield():
  Thread: [RUNNING] → yield → [RUNNABLE] (may get CPU right back)
  *(Bas kehta hai "turn de do" — OS mane ya na mane)*
```

### ⚠️ Pitfalls / Gotchas
- `sleep()` inside synchronized → other threads **blocked for entire sleep** *(galat design!)*
- `wait()` outside synchronized → **IllegalMonitorStateException**
- `yield()` is practically **useless** — never use in production

### 🗣️ How to Say in Interview
> *"The critical difference is lock behavior. sleep() pauses but keeps the lock — blocking other threads. wait() releases the lock and goes to WAITING — enabling inter-thread communication like producer-consumer. yield() is a scheduler hint I never use in production. For delays I use ScheduledExecutorService, for coordination I use wait/notify or higher-level primitives like CompletableFuture."*

### 🎯 Tricky Interview Qs
**Q: What if sleep(0)?**
> Like a yield hint — OS can context switch if it wants.

**Q: Can sleep be interrupted?**
> Yes — `Thread.interrupt()` throws `InterruptedException` during sleep.

### ⚡ Remember
1. **sleep** = holds lock, timed pause *(lock nahi chhodta)*
2. **wait** = releases lock, needs notify *(lock chhod deta hai)*
3. **yield** = useless hint, never use
4. wait inside synchronized + while loop *(warna galat hoga)*
5. sleep inside synchronized = **bad practice**

### 🔗 Follow-ups
→ [Q11. wait vs sleep](#q11) → [Q29. Inter-thread communication](#q29)

---

<a id="q11"></a>
## Q11. What is the difference between wait() and sleep()?

### 📝 One-Liner
> wait() releases lock and waits for notification; sleep() holds lock and waits for time.

### 🆚 vs. Comparison
| Feature | wait() | sleep() |
|---------|--------|---------|
| Lock | **Releases** ✅ | **Keeps** ❌ |
| Belongs to | `Object` | `Thread` |
| Wake up by | `notify()` | time expiry |
| Purpose | Thread communication | Delay |
| Needs sync | ✅ Yes | ❌ No |

### ⚠️ Pitfalls / Gotchas
- Always call `wait()` in **while loop** — spurious wakeup ho sakta hai *(bina notify ke uth jaata hai)*
```java
// ❌ Wrong
if (queue.isEmpty()) lock.wait();

// ✅ Correct — re-checks after wakeup
while (queue.isEmpty()) lock.wait();
```

### ⚡ Remember
1. **wait = lock chhodta hai** → communication
2. **sleep = lock rakhta hai** → delay
3. wait() always in **while loop** *(spurious wakeup)*
4. wait() must be in synchronized *(warna error)*

### 🔗 Follow-ups
→ [Q29. Inter-thread communication](#q29) → [Q31. Why wait needs synchronized](#q31)

---

<a id="q12"></a>
## Q12. What is thread starvation?

### 📝 One-Liner
> A thread never gets CPU because higher-priority threads keep running.

### 🔑 Quick Answer
> Starvation: thread is **perpetually denied CPU** by higher-priority or lock-holding threads. Alive but never runs. *(Thread bhuka baitha hai — CPU kabhi milta hi nahi)*

### 📖 How It Works
```
CPU: [High-1][High-2][High-1][High-2]... forever
Low-priority: waiting... waiting... (STARVED!)
*(Bade threads pehle, chhota kabhi chance nahi milta)*
```

### 🗣️ How to Say in Interview
> *"Starvation is when a thread is technically runnable but never gets CPU time because higher-priority threads or lock monopolizers keep running. Fix: use fair locks — ReentrantLock(true) — which uses FIFO ordering, or avoid relying on thread priorities."*

### ⚠️ Pitfalls / Gotchas
- Fair locks fix starvation but cost ~10-20% performance *(fairness = slow, but safe)*
- Don't confuse: deadlock (both stuck), starvation (one stuck), livelock (both active, no progress)

### ⚡ Remember
1. Thread alive but **never gets CPU** *(bhuka thread)*
2. Fix: **ReentrantLock(true)** — fair lock
3. Different from deadlock *(deadlock = jam; starvation = wait forever)*

### 🔗 Follow-ups
→ [Q13. Deadlock](#q13) → [Q14. Livelock](#q14)

---

<a id="q13"></a>
## Q13. What is deadlock?

### 📝 One-Liner
> Two threads permanently waiting for each other's locks — neither can proceed.

### 🔑 Quick Answer
> Thread-1 holds Lock-A, wants Lock-B. Thread-2 holds Lock-B, wants Lock-A. **Neither releases, neither proceeds** — permanent hang. *(Do log ek dusre ka darvaza pakde hain — koi nahi chhodega)*

### 📖 How It Works
```
  Thread-1:                    Thread-2:
  lock(A) ✅                   lock(B) ✅
  lock(B) → WAIT...           lock(A) → WAIT...
         ↑                           ↑
         └──────── DEADLOCK ──────────┘
  *(Application chup chap hang — no error, no log)*
```

### 🗣️ How to Say in Interview
> *"Deadlock is when two or more threads permanently block each other. I've dealt with this in production — the app hung with no errors. We took a thread dump with jstack and saw the deadlock chain. Fix: enforce consistent lock ordering — always lock A before B regardless of which thread. Also use tryLock with timeout for critical sections."*

### ⚠️ Pitfalls / Gotchas
- **No error thrown** — app silently hangs *(koi exception nahi, bus atak jaata hai)*
- With `synchronized`, **no timeout** — waits forever
- Also happens in database transactions (row-level locks)

### 🎯 Tricky Interview Qs
**Q: Can deadlock happen with a single thread?**
> No — needs at least 2 threads. Single thread can infinite-wait (wait() without notify()) but that's not deadlock. *(Akela thread deadlock nahi kar sakta)*

### ⚡ Remember
1. Two+ threads **waiting for each other's locks**
2. **Silent hang** — no exception *(sab chup)*
3. Fix: **lock ordering** ⭐ or `tryLock(timeout)`
4. Detect: **jstack** / ThreadMXBean
5. 4 conditions: mutual exclusion + hold & wait + no preemption + circular wait

### 🔗 Follow-ups
→ [Q64. Four conditions](#q64) → [Q65. Prevention](#q65) → [Q66. Detection](#q66)

---

<a id="q14"></a>
## Q14. What is livelock?

### 📝 One-Liner
> Threads keep responding to each other but making no progress — like two people dodging in a hallway.

### 🔑 Quick Answer
> Threads are **active** (not blocked) but keep **undoing each other's work**. *(Dono chal rahe hain par aage nahi badh rahe — jaise do log ek raaste mein ek dusre ko side de rahe hain aur takra rahe hain)*

### 📖 How It Works
```
Thread-1: "I'll back off" → releases
Thread-2: "I'll back off" → releases
Thread-1: "Oh free, I'll try" → acquires
Thread-2: "Oh free, I'll try" → acquires
→ Repeat forever (kaam nahi ho raha!)

Fix: Random backoff
  Thread-1: wait random(50-200ms)
  Thread-2: wait random(50-200ms)
  → Different delays → one wins ✅
```

### 🆚 vs. Comparison
| | Deadlock | Livelock |
|-|----------|---------|
| State | BLOCKED | RUNNABLE |
| CPU | Zero | High *(CPU jal raha bekaar mein)* |
| Detection | Easy (dump) | Hard (looks busy) |

### ⚡ Remember
1. **Active but no progress** *(busy doing nothing)*
2. Fix: **random backoff**
3. Harder to detect than deadlock

### 🔗 Follow-ups
→ [Q13. Deadlock](#q13) → [Q12. Starvation](#q12)

---

<a id="q15"></a>
## Q15. What is thread interference (race condition)?

### 📝 One-Liner
> Two threads read-modify-write shared data simultaneously — result depends on timing.

### 🔑 Quick Answer
> When two threads access **shared data without sync**, outcome depends on timing. `count++` is read → increment → write (3 ops). *(Do threads ek hi variable ek saath badal rahe hain — result galat aata hai)*

### 📖 How It Works
```
count = 0, count++ = 3 operations:

  Thread-1:  READ(0)  ADD(0+1=1)  WRITE(1)
  Thread-2:       READ(0)  ADD(0+1=1)  WRITE(1) ← STALE!
  
  Expected: 2  Actual: 1  ← LOST UPDATE!
  *(Dono ne 0 padha, dono ne 1 likha — ek increment kho gaya)*
```

### 💻 Code
```java
// ❌ Race condition
private int count = 0;
public void increment() { count++; }  // NOT atomic!

// ✅ Fix 1: synchronized
public synchronized void increment() { count++; }

// ✅ Fix 2: AtomicInteger (better — lock-free hai)
private AtomicInteger count = new AtomicInteger(0);
public void increment() { count.incrementAndGet(); }
```

### ⚠️ Pitfalls / Gotchas
- Race conditions are **non-deterministic** *(testing mein nahi dikhta, production mein fatega)*
- **count++ looks atomic but is NOT** *(ek line hai, par 3 operations hain)*

### ⚡ Remember
1. **count++ = NOT atomic** (3 ops) *(read, add, write)*
2. Fix: **synchronized** or **AtomicInteger**
3. **Non-deterministic** — hardest bugs to find
4. AtomicInteger preferred for simple counters

### 🔗 Follow-ups
→ [Q67. Race condition](#q67) → [Q68. Thread safety](#q68)

---

<a id="q16"></a>
## Q16. What is thread priority?

### 📝 One-Liner
> Integer 1-10 hint to OS — higher = more likely to get CPU (not guaranteed).

### 🔑 Quick Answer
> Priority 1 (MIN) to 10 (MAX), default 5. Just a **hint** to OS — not guaranteed. *(Sirf request hai — "isko pehle chalao" — OS maan bhi sakta hai nahi bhi)*

### ⚠️ Pitfalls / Gotchas
- **Never rely on priority** for correctness *(Windows aur Linux alag behave karte hain)*
- Can cause **starvation** of low-priority threads

### ⚡ Remember
1. Range **1-10**, default **5**
2. Just a **hint** *(OS ki marzi)*
3. **Never** use for correctness
4. Can cause starvation

### 🔗 Follow-ups
→ [Q9. Scheduling](#q9) → [Q12. Starvation](#q12)

---

<a id="q17"></a>
## Q17. What is context switching?

### 📝 One-Liner
> OS saves one thread's state and loads another's — costs 1-10μs plus cache invalidation.

### 🔑 Quick Answer
> OS **pauses one thread**, saves its state, **loads another thread's state**. Direct cost ~1-10μs. Real cost: **cache invalidation**. *(OS ek thread rok ke doosre ko chalu karta hai — isme time lagta hai)*

### 📖 How It Works
```
Thread-1: [registers, PC saved to memory]
       ↓ SWITCH (~1-10μs)
Thread-2: [registers, PC loaded from memory]

Hidden cost: CPU cache invalidation
  Thread-1 data was in L1 cache → Thread-2 needs different data
  → Cache miss → main memory fetch (100x slower!)
  *(Purane thread ka data cache mein tha — naye ko alag chahiye → slow)*
```

### 🗣️ How to Say in Interview
> *"Context switching is the overhead of pausing one thread to run another. Direct cost is 1-10μs, but the indirect cost — CPU cache invalidation — is more significant. The new thread needs different data, causing cache misses that are 100x slower. This is why I size CPU-bound pools equal to cores — more threads means more switching, actually slowing things down."*

### ⚠️ Pitfalls / Gotchas
- Too many threads = too many switches = **worse performance** *(jitne zyada threads, utna slow)*
- This is why CPU-bound: threads = cores

### ⚡ Remember
1. Cost: **1-10μs** direct + **cache miss** indirect
2. **Cache invalidation** = real slowdown
3. More threads ≠ faster *(zyada threads = zyada switching)*
4. CPU-bound: threads = cores

### 🔗 Follow-ups
→ [Q70. Thread pool tuning](#q70) → [Q71. CPU vs I/O bound](#q71)

---

<a id="q18"></a>
## Q18. What is multitasking? What are process-based vs thread-based multitasking?

### 📝 One-Liner
Multitasking = doing multiple things simultaneously; **Process-based** = multiple programs; **Thread-based** = multiple tasks within one program.

### 🔑 Quick Answer
**(1) Process-based multitasking** — running multiple programs concurrently (e.g., MS Word + Calculator). Each process has its own memory space. **(2) Thread-based multitasking** — running multiple parts of one program concurrently (e.g., spell-check while typing in Word). Threads share memory. Java provides built-in support for thread-based multitasking. *(Process-based = alag programs | Thread-based = ek program ke andar alag tasks)*

### ⚡ Remember
`Process multitasking = separate programs | Thread multitasking = within same program | Java = thread-based`

---

<a id="q19"></a>
## Q19. Which Java APIs support threads?

### 📝 One-Liner
`java.lang.Thread`, `java.lang.Runnable`, `java.lang.Object` (wait/notify), and `java.util.concurrent` package.

### 🔑 Quick Answer
(1) `java.lang.Thread` — extend to create thread. (2) `java.lang.Runnable` — implement for task. (3) `java.lang.Object` — `wait()`, `notify()`, `notifyAll()` for inter-thread communication. (4) `java.util.concurrent` — Executor, Future, CompletableFuture, locks, concurrent collections, atomic variables. *(4 main APIs: Thread, Runnable, Object (wait/notify), java.util.concurrent)*

### ⚡ Remember
`Thread + Runnable + Object(wait/notify) + java.util.concurrent`

---

<a id="q20"></a>
## Q20. Explain the main thread in Java?

### 📝 One-Liner
The main thread is the **first thread** started by JVM when `main()` is called — all child threads are spawned from it.

### 🔑 Quick Answer
When JVM calls `main()`, it starts a new thread called the "main thread." All child threads are spawned from main. The main thread is typically the **last to finish** (waits for non-daemon children). It's always a **non-daemon** thread. You can get it via `Thread.currentThread()` inside main(). *(Main thread = JVM ka pehla thread, sab child threads iske andar se bante hain)*

### ⚡ Remember
`Main thread = first thread | Spawns children | Non-daemon | Last to finish`

---

<a id="q21"></a>
## Q21. Can we restart a dead thread in Java?

### 📝 One-Liner
No — calling `start()` on a terminated thread throws `IllegalThreadStateException`.

### 🔑 Quick Answer
Once a thread's `run()` completes, it enters TERMINATED state. Calling `start()` again throws `IllegalThreadStateException`. If you need the same task again, create a **new Thread** object. A thread object is one-use only. *(Dead thread restart nahi ho sakta — naya Thread banao)*

### ⚡ Remember
`Dead thread + start() = IllegalThreadStateException | Create new Thread instead`

---

<a id="q22"></a>
## Q22. Can one thread block another thread?

### 📝 One-Liner
No — a thread can only block **itself** (via `sleep()`, `wait()`, `join()`); it cannot directly block another thread.

### 🔑 Quick Answer
Thread A cannot force Thread B to block. Thread B can only block itself by calling `sleep()`, `wait()`, or `join()`. However, Thread A can **indirectly** cause B to wait by holding a lock that B needs (synchronization). `interrupt()` can request a thread to stop, but it's cooperative. *(Ek thread doosre ko directly block nahi kar sakta — sirf khud ko block kar sakta hai)*

### ⚡ Remember
`Thread blocks itself only | sleep/wait/join = self-blocking | Indirect via lock contention`

---

<a id="q23"></a>
## Q23. Can we overload the run() method?

### 📝 One-Liner
Yes, but Thread's `start()` always calls `run()` with **no arguments** — overloaded versions must be called explicitly.

### 🔑 Quick Answer
You can define `run(int x)` or `run(String s)` — they're valid overloaded methods. But `start()` will only invoke `public void run()` (no-arg). Overloaded versions need explicit calls: `thread.run(5)` — but this runs on the current thread, not a new one. *(run() overload ho sakta hai, lekin start() sirf no-arg run() call karega)*

### ⚡ Remember
`run() overload = yes | start() calls only run() (no-arg) | Overloaded run = explicit call`

---

> **🎯 Navigation:** [Next → Synchronization (Q18-28)](02-synchronization.md) | [📋 All Sections](README.md)
