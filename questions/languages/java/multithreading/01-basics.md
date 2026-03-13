# 🧵 Core Java Multithreading Basics (Q1–Q17)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>

## Q1. What is multithreading in Java?

### 🔑 Quick Answer

> Multithreading is the ability to execute **multiple threads simultaneously** within a single process. Each thread runs independently but shares the **same memory space** (heap), enabling concurrent execution of tasks.

### 📖 Step-by-Step Explanation

**Step 1 — Single-threaded vs Multi-threaded:**

```
SINGLE-THREADED:
  main thread: Task A (3 sec) → Task B (2 sec) → Task C (4 sec)
  Total: 9 seconds (sequential)

MULTI-THREADED:
  Thread-1: Task A (3 sec) ─────→
  Thread-2: Task B (2 sec) ───→
  Thread-3: Task C (4 sec) ───────→
  Total: 4 seconds (parallel — limited by slowest task)
```

**Step 2 — What threads share and don't share:**

```
SHARED (same for all threads in a process):
  ✅ Heap memory (objects, instance variables)
  ✅ Method area (class definitions, static variables)
  ✅ Open files and network connections

NOT SHARED (each thread has its own):
  ❌ Stack (local variables, method calls)
  ❌ Program counter (current instruction)
  ❌ Register values
```

### 🗣️ How to Explain in Interview

> *"Multithreading lets a single Java process execute multiple threads concurrently. Each thread is a lightweight unit of execution with its own call stack and program counter, but they share the same heap memory. This means threads can work on different tasks simultaneously — like one thread handling a user request while another writes to a database. The shared memory is both the advantage — threads can communicate easily — and the challenge — you need synchronization to avoid data corruption."*

### 💻 Code Example

```java
public class MultithreadingDemo {
    public static void main(String[] args) {
        Thread t1 = new Thread(() -> {
            for (int i = 0; i < 5; i++)
                System.out.println("Thread-1: " + i);
        });
        
        Thread t2 = new Thread(() -> {
            for (int i = 0; i < 5; i++)
                System.out.println("Thread-2: " + i);
        });
        
        t1.start();  // Starts new thread
        t2.start();  // Starts another thread
        // Both run concurrently — output interleaved
    }
}
```

### ⚡ Key Points to Remember

1. Multiple threads within **one process**
2. **Shared heap**, separate stacks
3. Concurrent ≠ parallel (concurrent = managing multiple tasks; parallel = running simultaneously on multiple cores)
4. Need **synchronization** for shared data
5. Java supports multithreading natively since JDK 1.0

---

<a id="q2"></a>

## Q2. What is the difference between a process and a thread?

### 🔑 Quick Answer

> A **process** is an independent program with its own memory space. A **thread** is a lightweight unit within a process that shares the process's memory. Threads are cheaper to create and faster to switch between.

### 📖 Step-by-Step Explanation

```
PROCESS:
  ┌──────────────────────────────┐
  │ Process A (JVM Instance)     │
  │ Own heap, own stack, own code│
  │ Cannot access Process B's    │
  │ memory directly              │
  └──────────────────────────────┘
  
  ┌──────────────────────────────┐
  │ Process B (Another JVM)      │
  │ Completely isolated          │
  └──────────────────────────────┘

THREADS within a Process:
  ┌────────────────────────────────────────┐
  │ Process A (JVM Instance)               │
  │                                        │
  │  Thread-1    Thread-2    Thread-3      │
  │  [Stack-1]   [Stack-2]   [Stack-3]    │
  │       ↓          ↓          ↓          │
  │  ┌─── SHARED HEAP MEMORY ────────┐    │
  │  │ Objects, instance variables    │    │
  │  └───────────────────────────────┘    │
  └────────────────────────────────────────┘
```

| Feature | Process | Thread |
|---------|---------|--------|
| **Memory** | Own memory space | Shares process memory |
| **Creation cost** | Heavy (new JVM) | Lightweight (~1MB stack) |
| **Communication** | IPC (sockets, pipes) | Shared heap (direct) |
| **Context switch** | Expensive | Cheap |
| **Isolation** | Fully isolated | Not isolated (shared heap) |
| **Crash impact** | Only this process crashes | Can crash entire process |

### 🗣️ How to Explain in Interview

> *"A process is a self-contained execution environment with its own memory — think of each running JVM as a process. A thread is a lightweight execution unit within a process. The key difference is memory: processes have isolated memory spaces, so they communicate through IPC mechanisms like sockets. Threads within the same process share the heap, so they communicate directly through shared variables — which is faster but requires synchronization. Threads are also cheaper to create — about 1MB of stack versus an entire new memory space for a process."*

### ⚡ Key Points to Remember

1. **Process** = heavyweight, isolated memory
2. **Thread** = lightweight, shared memory
3. Thread creation ~**1000× cheaper** than process
4. Context switch between threads is **much faster**
5. Thread crash → **entire process** can crash

---

<a id="q3"></a>

## Q3. What are the advantages of multithreading?

### 🔑 Quick Answer

> **Better CPU utilization** (use all cores), **improved responsiveness** (UI doesn't freeze), **efficient I/O** (do work while waiting for network/disk), and **resource sharing** (threads share memory, cheaper than processes).

### 📖 Step-by-Step Explanation

```
1. CPU UTILIZATION:
   Single thread on 8-core CPU: 12.5% usage (1/8)
   8 threads on 8-core CPU:     100% usage (8/8) ⭐

2. RESPONSIVENESS:
   Single thread: [User clicks] → [Long DB query...waiting...] → [UI updates]
   Multi-thread:  [User clicks] → [UI says "Loading..."]
                  [Background]  → [DB query...done → update UI]

3. EFFICIENT I/O:
   Single thread: [Read file 1...wait] → [Read file 2...wait] → [Process]
   Multi-thread:  [Read file 1...wait]
                  [Read file 2...wait]   → Both ready → Process faster
                  
4. RESOURCE SHARING:
   Threads share heap → no need to copy data between them
   Cheaper than spawning multiple processes
```

### 🗣️ How to Explain in Interview

> *"Four main advantages. First, CPU utilization — modern CPUs have 8-16 cores, and a single thread uses only one core. Multithreading lets us use all cores. Second, responsiveness — in a web server, one thread handles user requests while another processes background tasks, so users don't wait. Third, I/O efficiency — while one thread waits for a database response, another thread can process data. Fourth, resource sharing — threads share heap memory, so there's no overhead of copying data between processes."*

### ⚡ Key Points to Remember

1. **CPU utilization** — use all cores
2. **Responsiveness** — UI/API stays responsive
3. **I/O overlap** — work while waiting for I/O
4. **Resource sharing** — shared heap, no IPC needed
5. Trade-off: **complexity** (synchronization, debugging)

---

<a id="q4"></a>

## Q4. What are the life cycle states of a thread?

### 🔑 Quick Answer

> Six states: **NEW** → **RUNNABLE** → **RUNNING** → **BLOCKED/WAITING/TIMED_WAITING** → **TERMINATED**. Defined in `Thread.State` enum.

### 📖 Step-by-Step Explanation

```
          new Thread()          start()           scheduler picks
    ┌──────────┐    ┌──────────────┐    ┌──────────────┐
    │   NEW    │───→│   RUNNABLE   │───→│   RUNNING    │
    └──────────┘    └──────────────┘    └──────────────┘
                          ↑                    │  │  │
                          │                    │  │  │
                    scheduler resumes          │  │  │
                          │                    │  │  │
                    ┌─────┴──────┐            │  │  │
                    │  BLOCKED   │←───────────┘  │  │  waiting for monitor lock
                    └────────────┘               │  │
                    ┌────────────┐               │  │
                    │  WAITING   │←──────────────┘  │  wait(), join(), park()
                    └────────────┘                  │
                    ┌────────────┐                  │
                    │TIMED_WAITING│←─────────────────┘  sleep(ms), wait(ms)
                    └────────────┘
                                                    │
                          run() completes           │
                    ┌────────────┐                  │
                    │ TERMINATED │←─────────────────┘
                    └────────────┘
```

| State | When | How to Enter |
|-------|------|-------------|
| **NEW** | Thread created, not started | `new Thread()` |
| **RUNNABLE** | Ready to run, waiting for CPU | `start()` called |
| **BLOCKED** | Waiting for a monitor lock | Trying to enter `synchronized` block |
| **WAITING** | Waiting indefinitely | `wait()`, `join()`, `LockSupport.park()` |
| **TIMED_WAITING** | Waiting with timeout | `sleep(ms)`, `wait(ms)`, `join(ms)` |
| **TERMINATED** | Finished execution | `run()` completed or exception thrown |

### 🗣️ How to Explain in Interview

> *"A thread goes through six states. NEW — just created with new Thread(). RUNNABLE — after start() is called, it's ready to run but waiting for the CPU scheduler. When the scheduler picks it, it runs. It can move to BLOCKED if it tries to enter a synchronized block that's held by another thread. WAITING if it calls wait() or join() — it waits indefinitely until another thread signals it. TIMED_WAITING for sleep() or wait() with a timeout. Finally TERMINATED when run() completes. These states are defined in the Thread.State enum, and you can check them with thread.getState()."*

### 💻 Code Example

```java
Thread t = new Thread(() -> {
    try { Thread.sleep(1000); } catch (InterruptedException e) {}
});

System.out.println(t.getState());  // NEW
t.start();
System.out.println(t.getState());  // RUNNABLE
Thread.sleep(100);
System.out.println(t.getState());  // TIMED_WAITING (sleeping)
t.join();
System.out.println(t.getState());  // TERMINATED
```

### ⚡ Key Points to Remember

1. **6 states**: NEW, RUNNABLE, BLOCKED, WAITING, TIMED_WAITING, TERMINATED
2. **start()** → NEW to RUNNABLE (not Running!)
3. **BLOCKED** = waiting for lock, **WAITING** = waiting for signal
4. Check with `thread.getState()`
5. **TERMINATED** is final — thread cannot be restarted

---

<a id="q5"></a>

## Q5. What is the difference between Thread class and Runnable interface?

### 🔑 Quick Answer

> **Runnable** is preferred — it's a functional interface with one method `run()`. **Thread** class extends `java.lang.Thread` directly. Using Runnable allows you to extend another class (Java doesn't support multiple inheritance) and is better for thread pool usage.

### 📖 Step-by-Step Explanation

```
Approach 1: EXTEND Thread
  class MyThread extends Thread {
      public void run() { ... }
  }
  → Tightly coupled to Thread class
  → Cannot extend another class
  → Each instance IS a thread

Approach 2: IMPLEMENT Runnable ⭐
  class MyTask implements Runnable {
      public void run() { ... }
  }
  → Separates task from thread
  → Can extend another class
  → Can be submitted to thread pools
  → Lambda-friendly (functional interface)
```

| Feature | Thread class | Runnable interface |
|---------|-------------|-------------------|
| **Inheritance** | Extends Thread (no other class) | Implements Runnable (can extend another) |
| **Flexibility** | Tightly coupled | Loosely coupled ⭐ |
| **Thread pool** | Cannot submit directly | Can submit to ExecutorService ⭐ |
| **Lambda** | Not a functional interface | Functional interface ⭐ |
| **Reusability** | Task = Thread (1:1) | Task reusable across threads |

### 💻 Code Example

```java
// Approach 1: Extending Thread (not recommended)
class MyThread extends Thread {
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}
new MyThread().start();

// Approach 2: Runnable (recommended) ⭐
class MyTask implements Runnable {
    public void run() {
        System.out.println("Running in: " + Thread.currentThread().getName());
    }
}
new Thread(new MyTask()).start();

// Approach 3: Lambda (best for simple tasks) ⭐⭐
new Thread(() -> System.out.println("Lambda thread!")).start();

// Approach 4: With ExecutorService (production standard) ⭐⭐⭐
ExecutorService executor = Executors.newFixedThreadPool(4);
executor.submit(() -> System.out.println("Thread pool!"));
```

### 🗣️ How to Explain in Interview

> *"I always prefer Runnable over extending Thread. Three reasons: First, Java has single inheritance — if I extend Thread, I can't extend another class. Second, Runnable separates the task from the execution mechanism — the same Runnable can be submitted to a thread pool or used with CompletableFuture. Third, Runnable is a functional interface, so I write it as a lambda. In practice, I rarely use either directly — I use ExecutorService with lambdas for production code."*

### ⚡ Key Points to Remember

1. **Runnable > Thread** (always prefer Runnable)
2. Runnable = **functional interface** → lambda-friendly
3. Runnable → **works with thread pools** (ExecutorService)
4. Thread class → **single inheritance** problem
5. Production: **ExecutorService + lambda** (not Thread/Runnable directly)

---

<a id="q6"></a>

## Q6. What is the difference between Runnable and Callable?

### 🔑 Quick Answer

> **Runnable** has `run()` — returns **void**, can't throw checked exceptions. **Callable** has `call()` — returns a **result** and can throw checked exceptions. Use Callable when you need a return value.

### 📖 Step-by-Step Explanation

```java
// Runnable: No return, no checked exception
@FunctionalInterface
public interface Runnable {
    void run();  // returns nothing
}

// Callable: Returns result, can throw exception
@FunctionalInterface
public interface Callable<V> {
    V call() throws Exception;  // returns V, throws Exception
}
```

| Feature | Runnable | Callable |
|---------|---------|---------|
| **Method** | `run()` | `call()` |
| **Return** | void | V (any type) |
| **Exception** | No checked exceptions | Can throw checked exceptions |
| **Submit to** | `execute()` or `submit()` | `submit()` only |
| **Result** | No Future | Returns `Future<V>` |
| **Since** | JDK 1.0 | JDK 1.5 |

### 💻 Code Example

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

// Runnable: fire-and-forget
executor.submit(() -> System.out.println("No return value"));

// Callable: returns a result
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;  // returns a value!
});

int result = future.get();  // Blocks until result is ready
System.out.println("Result: " + result);  // 42
```

### 🗣️ How to Explain in Interview

> *"Runnable's run() returns void and can't throw checked exceptions — it's fire-and-forget. Callable's call() returns a typed result wrapped in a Future and can throw checked exceptions. I use Runnable for tasks where I don't need a result — like logging or sending notifications. I use Callable when I need to get a result back — like querying a service and returning the response. In modern Java, I mostly use CompletableFuture which supports both patterns with better composition."*

### ⚡ Key Points to Remember

1. **Runnable** = `void run()` — no return, no checked exceptions
2. **Callable** = `V call()` — returns result, throws exceptions
3. Callable → returns **Future<V>** for result retrieval
4. Both are **functional interfaces** → lambda-friendly
5. Modern: **CompletableFuture** supersedes both for complex workflows

---

<a id="q7"></a>

## Q7. What is the difference between start() and run() methods?

### 🔑 Quick Answer

> `start()` creates a **new OS thread** and calls run() on it. `run()` directly executes the method on the **current thread** — no new thread is created. Always use `start()`.

### 📖 Step-by-Step Explanation

```
thread.start():
  main thread → calls start() → JVM creates NEW OS thread
                                 ↓
                                 new thread calls run()
  main thread continues...       new thread runs independently
  
  Result: TWO threads running concurrently ✅

thread.run():
  main thread → calls run() → run() executes ON main thread
                               (just a normal method call!)
  
  Result: ONE thread, sequential execution ❌ (no multithreading!)
```

### 💻 Code Example

```java
Thread t = new Thread(() -> {
    System.out.println("Thread: " + Thread.currentThread().getName());
});

// CORRECT: Creates new thread
t.start();
// Output: "Thread: Thread-0" (different thread!)

// WRONG: Runs on main thread — NOT multithreading!
// t.run();
// Output: "Thread: main" (same thread!)
```

### 🗣️ How to Explain in Interview

> *"start() is the correct way — it tells the JVM to create a new OS thread and invoke run() on that new thread. The calling thread continues immediately without waiting. If you call run() directly, it's just a regular method call on the current thread — no new thread is created, no concurrency happens. This is a common mistake. Also, start() can only be called once per Thread instance — calling it again throws IllegalThreadStateException."*

### ⚡ Key Points to Remember

1. **start()** → new thread → calls run() on it
2. **run()** → same thread → just a normal method call
3. `start()` can be called **only once** per Thread
4. Second call to start() → **IllegalThreadStateException**
5. Always use **start()**, never call run() directly

---

<a id="q8"></a>

## Q8. What happens if you call run() directly instead of start()?

### 🔑 Quick Answer

> No new thread is created. The `run()` method executes on the **calling thread** as a regular method call. You get **sequential execution**, not concurrent.

### 📖 Step-by-Step Explanation

```java
Thread t = new Thread(() -> {
    System.out.println("Running on: " + Thread.currentThread().getName());
});

// Calling run() directly
t.run();   // Prints: "Running on: main"  ← NO new thread!
t.run();   // Prints: "Running on: main"  ← Can call multiple times (just a method)

// Calling start() correctly
t.start(); // Prints: "Running on: Thread-0"  ← NEW thread!
// t.start(); // IllegalThreadStateException! Can't start twice
```

### 🗣️ How to Explain in Interview

> *"Calling run() directly is a mistake. It just executes the method body on the current thread — like calling any other method. No new thread is created, no concurrency happens. The code runs synchronously. start() is the one that creates a new OS thread via the JVM's native methods. Interestingly, run() can be called multiple times since it's just a method, but start() throws IllegalThreadStateException on a second call."*

### ⚡ Key Points to Remember

1. **run() directly** = regular method call on calling thread
2. **No new thread**, no concurrency
3. **run()** can be called multiple times (just a method)
4. **start()** can be called only once
5. This is a **common interview gotcha question**

---

<a id="q9"></a>

## Q9. What is thread scheduling?

### 🔑 Quick Answer

> Thread scheduling is how the **OS or JVM decides which thread gets CPU time**. Java uses **preemptive scheduling** — the OS scheduler allocates time slices to threads. Java influences scheduling via **priorities** but doesn't guarantee exact behavior.

### 📖 Step-by-Step Explanation

```
CPU with 4 cores, 10 runnable threads:

OS Scheduler decides:
  Core 1: Thread-1 (50ms) → Thread-5 (50ms) → Thread-1 (50ms) → ...
  Core 2: Thread-2 (50ms) → Thread-6 (50ms) → Thread-2 (50ms) → ...
  Core 3: Thread-3 (50ms) → Thread-7 (50ms) → Thread-3 (50ms) → ...
  Core 4: Thread-4 (50ms) → Thread-8 (50ms) → Thread-9 (50ms) → ...

Factors that influence scheduling:
  1. Thread priority (1-10, just a hint)
  2. Thread state (RUNNABLE gets CPU, WAITING doesn't)
  3. OS scheduling algorithm (typically round-robin with priority)
  4. Whether thread is CPU-bound or I/O-bound
```

### 🗣️ How to Explain in Interview

> *"Thread scheduling is the OS's mechanism for deciding which runnable threads get CPU time. Java uses preemptive scheduling — the OS gives each thread a time slice, and when it expires, the scheduler can switch to another thread. Java lets you influence scheduling with thread priorities — 1 to 10 — but these are just hints to the OS. You shouldn't rely on priorities for correctness. The actual behavior is platform-dependent — on Windows it's priority-based, on Linux it uses the Completely Fair Scheduler."*

### ⚡ Key Points to Remember

1. **OS controls** thread scheduling (not JVM)
2. **Preemptive** — OS can interrupt a running thread
3. **Time-slicing** — each thread gets a time quantum
4. **Priority** = hint, not guarantee
5. **Platform-dependent** behavior

---

<a id="q10"></a>

## Q10. What is thread priority?

### 🔑 Quick Answer

> An integer from **1 (MIN) to 10 (MAX)**, default **5 (NORM)**. Higher priority threads get **more CPU time**, but it's only a **hint** to the OS — not a guarantee.

### 📖 Step-by-Step Explanation

```java
Thread.MIN_PRIORITY  = 1   // Lowest
Thread.NORM_PRIORITY = 5   // Default
Thread.MAX_PRIORITY  = 10  // Highest

thread.setPriority(8);            // Set priority
int priority = thread.getPriority(); // Get priority
```

```
⚠️ Priority is a HINT, not a guarantee:
  - OS may ignore priorities entirely
  - Different platforms handle priorities differently
  - High-priority thread does NOT always run first
  - Never use priority for correctness — use synchronization
```

### 🗣️ How to Explain in Interview

> *"Thread priority is an integer from 1 to 10 —  MIN_PRIORITY, NORM_PRIORITY at 5, and MAX_PRIORITY at 10. A newly created thread inherits its parent's priority — usually 5. Higher priority means the thread gets more CPU time, but it's just a hint. I never rely on priorities for program correctness — they're platform-dependent and the OS can ignore them. For actual ordering, I use synchronization primitives like CountDownLatch or join()."*

### ⚡ Key Points to Remember

1. Range: **1 (MIN) to 10 (MAX)**, default **5 (NORM)**
2. Inherited from **parent thread**
3. **Hint** only — OS may ignore
4. **Never** use for correctness
5. Use `setPriority()` / `getPriority()`

---

<a id="q11"></a>

## Q11. What is the difference between sleep(), wait(), and yield()?

### 🔑 Quick Answer

> `sleep(ms)` — pauses current thread for time, **keeps lock**. `wait()` — pauses and **releases lock**, needs notify(). `yield()` — **hint** to scheduler to give CPU to other threads, often ignored.

### 📖 Step-by-Step Explanation

| Feature | `sleep(ms)` | `wait()` | `yield()` |
|---------|------------|---------|-----------|
| **Class** | Thread | Object | Thread |
| **Lock** | ❌ Does NOT release | ✅ Releases lock | ❌ Does NOT release |
| **Wake up** | After timeout | notify()/notifyAll() | Immediately (just a hint) |
| **Must be in synchronized?** | No | ✅ Yes (or IllegalMonitorStateException) |No |
| **Purpose** | Pause for time | Wait for condition | Hint to yield CPU |
| **Throws** | InterruptedException | InterruptedException | Nothing |

```
sleep(1000):
  Thread holds lock → sleeps 1000ms → wakes up → continues
  Other threads CANNOT enter the synchronized block ❌

wait():
  Thread RELEASES lock → waits → notify() → reacquires lock → continues
  Other threads CAN enter the synchronized block ✅

yield():
  Thread says "I'm willing to pause" → scheduler may or may not pause it
  Rarely used in practice
```

### 🗣️ How to Explain in Interview

> *"Three very different methods. sleep() pauses the current thread for a specified time but holds onto any locks — other threads can't enter the synchronized block. wait() releases the lock and waits until another thread calls notify() — this is how threads communicate. yield() is a hint to the scheduler saying 'I'm done for now, give someone else a turn' — but the scheduler can ignore it. The key distinction: sleep() keeps the lock, wait() releases it. That's why wait() must be called inside a synchronized block — it needs a lock to release."*

### ⚡ Key Points to Remember

1. **sleep()** = timed pause, **keeps lock**
2. **wait()** = indefinite pause, **releases lock**, needs notify()
3. **yield()** = hint only, usually ignored
4. wait() → **must be in synchronized block**
5. sleep() and wait() throw **InterruptedException**

---

<a id="q12"></a>

## Q12. What is the difference between wait() and sleep()?

### 🔑 Quick Answer

> **wait()** releases the lock and waits for notify() — used for **inter-thread communication**. **sleep()** keeps the lock and pauses for a fixed time — used for **timed delays**.

### 📖 Step-by-Step Explanation

```
SCENARIO: Thread A holds lock on object X

A calls X.wait():
  1. A RELEASES lock on X
  2. A enters WAITING state
  3. Thread B can now acquire lock on X
  4. B calls X.notify()
  5. A wakes up, RE-ACQUIRES lock on X
  6. A continues

A calls Thread.sleep(1000):
  1. A KEEPS lock on X
  2. A enters TIMED_WAITING state
  3. Thread B CANNOT acquire lock on X (BLOCKED!)
  4. After 1000ms, A wakes up
  5. A continues (still holding lock)
```

| | `wait()` | `sleep()` |
|--|---------|----------|
| **Belongs to** | `Object` class | `Thread` class |
| **Lock** | ✅ Releases | ❌ Keeps |
| **Wake condition** | `notify()`/`notifyAll()` | Timer expires |
| **Use case** | Thread coordination | Timed delays |
| **Requires synchronized** | ✅ Yes | ❌ No |

### 🗣️ How to Explain in Interview

> *"The critical difference is lock behavior. wait() releases the monitor lock and enters WAITING state — it's designed for inter-thread communication where one thread waits for a condition that another thread will fulfill. sleep() keeps the lock and enters TIMED_WAITING — it's just a delay. This has real implications: if Thread A calls sleep() inside a synchronized block, Thread B is blocked from entering that block for the entire sleep duration. If A calls wait(), B can enter immediately. That's why wait/notify is the pattern for producer-consumer problems."*

### ⚡ Key Points to Remember

1. **wait()** = Object method, releases lock, needs notify()
2. **sleep()** = Thread method, keeps lock, needs timeout
3. wait() inside `synchronized` → **mandatory**
4. Use wait/notify for **producer-consumer** patterns
5. sleep() causes **unnecessary lock holding**

---

<a id="q13"></a>

## Q13. What is thread starvation?

### 🔑 Quick Answer

> A thread is **perpetually denied CPU time** because higher-priority threads constantly take precedence. The starving thread is technically runnable but never gets to execute.

### 📖 Step-by-Step Explanation

```
Starvation scenario:
  Thread-1 (priority 10): Keeps getting CPU ───────────→
  Thread-2 (priority 10): Keeps getting CPU ───────────→
  Thread-3 (priority 1):  Never gets CPU .............. STARVING!

  Thread-3 is RUNNABLE but scheduler always picks higher-priority threads

Common causes:
  1. Unfair priority scheduling
  2. Synchronized block held for too long by one thread
  3. Unfair lock (non-FIFO ordering)
  4. Thread constantly losing compareAndSwap in CAS loops
```

### 🗣️ How to Explain in Interview

> *"Starvation happens when a thread can't get CPU time because other threads monopolize the resource. For example, if I have high-priority threads constantly running, low-priority threads never get scheduled. Or if one thread holds a synchronized lock for a very long time, other waiting threads starve. The fix is using fair locks — ReentrantLock with fairness=true ensures FIFO ordering. Also, avoid setting extreme thread priorities and keep synchronized blocks as short as possible."*

### ⚡ Key Points to Remember

1. Thread is **runnable but never gets CPU**
2. Caused by: **unfair scheduling**, **long lock holding**
3. Fix: **fair locks** (`new ReentrantLock(true)`)
4. Fix: keep **synchronized blocks short**
5. Different from **deadlock** (deadlock = threads stuck; starvation = thread ignored)

---

<a id="q14"></a>

## Q14. What is thread deadlock?

### 🔑 Quick Answer

> Two or more threads are **permanently blocked**, each waiting for a lock the other holds. Neither can proceed — the program hangs forever.

### 📖 Step-by-Step Explanation

```
Thread-1:  holds Lock-A  → wants Lock-B → BLOCKED (Thread-2 has Lock-B)
Thread-2:  holds Lock-B  → wants Lock-A → BLOCKED (Thread-1 has Lock-A)

Neither can proceed → DEADLOCK! 💀

Timeline:
  T1: synchronized(lockA) {       // T1 holds lockA
  T2:     synchronized(lockB) {   // T2 holds lockB
  T1:         synchronized(lockB) // T1 waits for lockB → BLOCKED
  T2:         synchronized(lockA) // T2 waits for lockA → BLOCKED
  
  → Both waiting forever
```

### 💻 Code Example

```java
Object lockA = new Object();
Object lockB = new Object();

Thread t1 = new Thread(() -> {
    synchronized (lockA) {           // Holds lockA
        Thread.sleep(100);
        synchronized (lockB) { }     // Waits for lockB → DEADLOCK
    }
});

Thread t2 = new Thread(() -> {
    synchronized (lockB) {           // Holds lockB
        Thread.sleep(100);
        synchronized (lockA) { }     // Waits for lockA → DEADLOCK
    }
});
```

### 🗣️ How to Explain in Interview

> *"Deadlock is when two or more threads are permanently blocked in a circular wait. Thread 1 holds lock A and wants lock B, Thread 2 holds lock B and wants lock A — neither can proceed. Four conditions must be present: mutual exclusion, hold and wait, no preemption, and circular wait. To prevent deadlocks, I always acquire locks in a consistent order — if all threads acquire lock A before lock B, circular wait can't happen. I also use tryLock() with a timeout instead of synchronized for complex locking scenarios."*

### ⚡ Key Points to Remember

1. Threads **permanently blocked**, each waiting for the other's lock
2. **Four conditions**: mutual exclusion, hold-and-wait, no preemption, circular wait
3. Fix: **consistent lock ordering**
4. Fix: **tryLock() with timeout**
5. Detect: **jstack**, **VisualVM**, **ThreadMXBean**

---

<a id="q15"></a>

## Q15. What is thread livelock?

### 🔑 Quick Answer

> Threads are **not blocked** but **continuously change state in response to each other** without making progress. Like two people trying to pass each other in a hallway — both step aside the same way, repeatedly.

### 📖 Step-by-Step Explanation

```
Deadlock:  Thread A → BLOCKED → stuck forever
Livelock:  Thread A → running → but doing useless work forever

Example (hallway analogy):
  Person A moves left  → Person B moves left  → STILL BLOCKED
  Person A moves right → Person B moves right → STILL BLOCKED
  ... forever (both active, but no progress)

Code example:
  Thread-1: "I'll back off and retry" → retries → back off → retries → ...
  Thread-2: "I'll back off and retry" → retries → back off → retries → ...
  Both threads are running but accomplishing nothing
```

### 🗣️ How to Explain in Interview

> *"Livelock is like deadlock but worse to detect — threads aren't blocked, they're actively running but making no progress. Imagine two polite people in a hallway — both step aside the same way, both step back, infinitely. In code, this happens when threads respond to each other's state changes in a way that creates an infinite loop of retries. The fix is adding randomized backoff — instead of both threads retrying immediately, add a random delay so they don't synchronize. This is the same approach used in Ethernet collision protocols."*

### ⚡ Key Points to Remember

1. Threads **not blocked** but **no progress** (actively running)
2. Harder to detect than deadlock (threads look alive)
3. Fix: **randomized backoff** (random delay before retry)
4. Different from deadlock: deadlock = stuck; livelock = running uselessly
5. Example: two threads repeatedly yielding to each other

---

<a id="q16"></a>

## Q16. What is thread interference?

### 🔑 Quick Answer

> When multiple threads **read and write shared data simultaneously** without synchronization, their operations **interleave** producing incorrect results. Also called a **race condition**.

### 📖 Step-by-Step Explanation

```
Shared variable: counter = 0

Thread-1: counter++    →  read(0) → add 1 → write(1)
Thread-2: counter++    →  read(0) → add 1 → write(1)

Expected: counter = 2
Actual:   counter = 1  ← Thread interference!

Why? counter++ is NOT atomic — it's 3 operations:
  1. READ value from memory
  2. ADD 1
  3. WRITE back to memory

Interleaving:
  T1: read(0)              → counter still 0
  T2: read(0)              → counter still 0 (T1 hasn't written yet!)
  T1: add 1, write(1)      → counter = 1
  T2: add 1, write(1)      → counter = 1 (overwrites T1's result!)
```

### 🗣️ How to Explain in Interview

> *"Thread interference — also called a race condition — happens when multiple threads access shared mutable data without synchronization. The classic example is counter++. It looks atomic but it's actually three operations: read, increment, write. If Thread A reads 0 and Thread B also reads 0 before A writes, both write 1 — we lose one increment. The fix is either synchronized, AtomicInteger, or volatile depending on the use case. AtomicInteger.incrementAndGet() is the best fix for counters because it uses hardware CAS operations."*

### ⚡ Key Points to Remember

1. **Multiple threads** + **shared mutable data** + **no sync** = interference
2. `counter++` is **NOT atomic** (read + add + write)
3. Fix: **synchronized**, **AtomicInteger**, **volatile** (for visibility only)
4. Same as **race condition**
5. **AtomicInteger** = best fix for counters (lock-free)

---

<a id="q17"></a>

## Q17. What is context switching?

### 🔑 Quick Answer

> When the OS **saves the state** of the current thread and **loads the state** of another thread so it can run. It's the cost of multithreading — too many threads = too much time spent switching instead of working.

### 📖 Step-by-Step Explanation

```
Thread-1 running on CPU Core 1:
  [Save T1 state: registers, PC, stack pointer]  ← Context save (~1-10μs)
  [Load T2 state: registers, PC, stack pointer]   ← Context restore (~1-10μs)
Thread-2 now running on CPU Core 1

Cost per context switch: ~1-10 microseconds
                        + cache invalidation (cold cache)
                        + TLB flush

Example impact:
  100 threads, switching every 10ms:
  10,000 switches/second × 10μs = 100ms/second = 10% overhead!
  
  1000 threads: 100,000 switches/sec = 100% overhead = NO useful work! 💀
```

### 🗣️ How to Explain in Interview

> *"Context switching is the overhead of multithreading. When the OS switches from one thread to another, it saves the current thread's state — registers, program counter, stack pointer — and loads the next thread's state. This takes 1-10 microseconds per switch, but the bigger cost is cache invalidation — the new thread's data isn't in the CPU cache, so there are many cache misses. This is why having too many threads is worse than too few — with 1000 threads, the CPU spends most of its time switching instead of doing useful work. My rule of thumb: threads ≈ number of CPU cores for CPU-bound work."*

### ⚡ Key Points to Remember

1. **Save** current thread state → **Load** next thread state
2. Cost: **1-10μs** per switch + **cache invalidation**
3. Too many threads → **excessive switching** → poor performance
4. CPU-bound: threads ≈ **number of cores**
5. I/O-bound: can have **more threads** (threads spend time waiting)

---

> **🎯 Navigation:** [Next → Synchronization & Locks (Q18-28)](02-synchronization.md) | [📋 All Sections](README.md)
