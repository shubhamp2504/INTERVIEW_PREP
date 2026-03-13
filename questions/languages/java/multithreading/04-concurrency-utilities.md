# ⚙️ Java Concurrency Utilities (Q37–Q50)

> 🔑 Quick Answer → 📖 Step-by-Step Explanation → 🗣️ How to Say in Interview → 💻 Code → ⚡ Remember → 🔗 Follow-ups

---

<a id="q37"></a>

## Q37. What is ExecutorService?

### 🔑 Quick Answer

> A **thread pool management API** from `java.util.concurrent`. You submit tasks (Runnable/Callable), it manages thread creation, reuse, and lifecycle. **Never create threads manually** in production — use ExecutorService.

### 📖 Step-by-Step Explanation

```
WITHOUT ExecutorService:
  Task 1 → new Thread() → runs → thread destroyed
  Task 2 → new Thread() → runs → thread destroyed
  Task 3 → new Thread() → runs → thread destroyed
  → Thread creation overhead × 1000 = SLOW ❌

WITH ExecutorService:
  Create pool (4 threads) once
  Task 1 → Thread-1 runs it → Thread-1 returns to pool
  Task 2 → Thread-2 runs it → Thread-2 returns to pool
  Task 3 → Thread-1 reused  → Thread-1 returns to pool
  → Thread reuse, bounded resources ✅
```

### 💻 Code Example

```java
ExecutorService executor = Executors.newFixedThreadPool(4);

// Submit Runnable (no return)
executor.submit(() -> System.out.println("Task 1"));

// Submit Callable (returns result)
Future<String> future = executor.submit(() -> {
    Thread.sleep(1000);
    return "Task 2 result";
});
String result = future.get();  // Blocks until done

// ALWAYS shut down when done
executor.shutdown();                          // Graceful (finish pending)
executor.awaitTermination(30, TimeUnit.SECONDS); // Wait for completion
```

### 🗣️ How to Explain in Interview

> *"ExecutorService is the production standard for running concurrent tasks. Instead of creating threads manually — which is expensive and can lead to resource exhaustion — I create a thread pool that reuses threads. I submit tasks via submit() and get a Future for results. The key methods: submit() for tasks, shutdown() for graceful stop, and awaitTermination() to wait for pending tasks. In Spring, I use ThreadPoolTaskExecutor which wraps Java's ExecutorService with proper lifecycle management."*

### ⚡ Key Points to Remember

1. **Thread pool** — reuses threads, bounded resources
2. `submit(Runnable)` or `submit(Callable<V>)` → returns `Future`
3. Always call **shutdown()** when done
4. **Never** create threads manually in production
5. Spring: use **ThreadPoolTaskExecutor** instead

---

<a id="q38"></a>

## Q38. What is a thread pool?

### 🔑 Quick Answer

> A **pre-created set of reusable threads** that wait for tasks. When a task arrives, an available thread executes it and returns to the pool. Avoids the overhead of creating/destroying threads per task.

### 📖 Step-by-Step Explanation

```
Thread Pool (size = 4):
  ┌──────────────────────────────────┐
  │  Pool:  [T1] [T2] [T3] [T4]     │ ← 4 threads ready
  │  Queue: [Task5] [Task6] [Task7]  │ ← Tasks waiting
  └──────────────────────────────────┘

  Task-1 arrives → T1 picks it up → executes → T1 returns to pool
  Task-2 arrives → T2 picks it up → executes → T2 returns to pool
  Task-5 arrives → No free thread → goes to QUEUE → waits for T1/T2/T3/T4

Benefits:
  ✅ Thread reuse (no creation overhead)
  ✅ Bounded threads (prevent resource exhaustion)
  ✅ Task queuing (handle bursts)
  ✅ Lifecycle management (clean shutdown)
```

### 🗣️ How to Explain in Interview

> *"A thread pool is a set of pre-created threads that sit idle waiting for work. When a task is submitted, an available thread picks it up, executes it, and returns to the pool for the next task. The advantages: thread creation is expensive — about 1MB of stack memory and OS overhead — so reusing threads saves that cost. The pool also bounds concurrency — instead of creating 10,000 threads for 10,000 requests, I use a pool of 200 and queue the rest. This prevents out-of-memory errors and gives predictable resource usage."*

### ⚡ Key Points to Remember

1. Pre-created **reusable threads**
2. **Bounded concurrency** — prevents resource exhaustion
3. **Task queue** — handles bursts beyond pool capacity
4. Thread creation ~1MB stack + OS overhead
5. Always **size the pool** based on workload type

---

<a id="q39"></a>

## Q39. What are the different thread pool types?

### 🔑 Quick Answer

> Four main types: **FixedThreadPool** (fixed size), **CachedThreadPool** (grows as needed), **SingleThreadExecutor** (one thread), **ScheduledThreadPool** (delayed/periodic tasks).

### 📖 Step-by-Step Explanation

| Pool Type | Threads | Queue | Use Case |
|-----------|---------|-------|----------|
| **newFixedThreadPool(n)** | Fixed n | Unbounded LinkedBlockingQueue | General-purpose, predictable load |
| **newCachedThreadPool()** | 0 → ∞ (grows) | SynchronousQueue (no buffering) | Short-lived, bursty tasks |
| **newSingleThreadExecutor()** | 1 | Unbounded LinkedBlockingQueue | Sequential task execution |
| **newScheduledThreadPool(n)** | Fixed n | DelayedWorkQueue | Timers, periodic tasks |

```
⚠️ Production warnings:
  FixedThreadPool: Unbounded queue → can cause OOM if tasks pile up!
  CachedThreadPool: Unlimited threads → can create 10,000 threads!
  
  ⭐ Production standard: ThreadPoolExecutor with explicit parameters
```

### 🗣️ How to Explain in Interview

> *"Java offers four pool types via Executors factory. FixedThreadPool has a fixed number of threads with an unbounded queue — good for predictable workloads. CachedThreadPool creates threads as needed and reuses idle ones — good for short bursty tasks but dangerous because it can create unlimited threads. SingleThreadExecutor uses one thread for sequential task execution. ScheduledThreadPool for delayed or periodic tasks. However, in production, I use ThreadPoolExecutor directly with explicit core size, max size, queue capacity, and rejection policy — the factory methods hide dangerous defaults like unbounded queues."*

### ⚡ Key Points to Remember

1. **FixedThreadPool** = fixed threads + unbounded queue (⚠️ OOM risk)
2. **CachedThreadPool** = unlimited threads (⚠️ thread explosion)
3. **SingleThread** = sequential execution
4. **Scheduled** = delayed/periodic tasks
5. Production: **ThreadPoolExecutor with explicit params** ⭐

---

<a id="q40"></a>

## Q40. What is Executors.newFixedThreadPool()?

### 🔑 Quick Answer

> Creates a pool with a **fixed number of threads**. If all threads are busy, new tasks wait in an **unbounded queue**. Good for predictable workloads with steady task arrival.

### 💻 Code Example

```java
// Creates 4 threads — no more, no less
ExecutorService pool = Executors.newFixedThreadPool(4);

for (int i = 0; i < 100; i++) {
    final int taskId = i;
    pool.submit(() -> {
        System.out.println("Task " + taskId + " on " + Thread.currentThread().getName());
        Thread.sleep(1000);
    });
}
// 4 threads process 100 tasks — 25 tasks per thread (sequential per thread)

pool.shutdown();
```

```
Internals:
  new ThreadPoolExecutor(
      4,                           // corePoolSize: 4
      4,                           // maxPoolSize: 4 (same = fixed)
      0L, TimeUnit.MILLISECONDS,   // idle timeout (never — fixed)
      new LinkedBlockingQueue<>()  // UNBOUNDED queue ⚠️
  )
```

### 🗣️ How to Explain in Interview

> *"newFixedThreadPool creates exactly n threads that stay alive for the pool's lifetime. If all threads are busy, tasks queue in an unbounded LinkedBlockingQueue. The risk is that unbounded queue — if tasks arrive faster than they're processed, the queue grows without limit, eventually causing OutOfMemoryError. For production, I create ThreadPoolExecutor explicitly with a bounded queue and a rejection policy like CallerRunsPolicy."*

### ⚡ Key Points to Remember

1. **Fixed** number of threads (core = max)
2. **Unbounded queue** — ⚠️ OOM risk if tasks pile up
3. Threads live for **pool's lifetime** (no timeout)
4. Good for **predictable, steady** workloads
5. Production: bound the queue + add **rejection policy**

---

<a id="q41"></a>

## Q41. What is Executors.newCachedThreadPool()?

### 🔑 Quick Answer

> Creates threads **on demand** with no upper limit. Idle threads are **recycled after 60 seconds**. Uses SynchronousQueue (no buffering — each task needs a thread). Dangerous for production — can create thousands of threads.

### 📖 Step-by-Step Explanation

```
Internals:
  new ThreadPoolExecutor(
      0,                              // corePoolSize: 0 (no permanent threads)
      Integer.MAX_VALUE,              // maxPoolSize: ∞ (UNLIMITED!) ⚠️
      60L, TimeUnit.SECONDS,          // idle threads die after 60s
      new SynchronousQueue<>()        // No queue! Each task needs a thread
  )

Behavior:
  Task arrives → any idle thread? → YES: reuse it
                                 → NO:  create NEW thread (no limit!)

  1000 tasks at once → 1000 threads created! 💀
```

### 🗣️ How to Explain in Interview

> *"CachedThreadPool creates threads as needed with no upper bound. It uses a SynchronousQueue internally, which means there's no task queue — every submitted task needs a thread immediately. Idle threads are recycled after 60 seconds. The danger is thread explosion — if 1000 tasks arrive simultaneously, it creates 1000 threads, potentially causing out-of-memory. I only use this for short-lived, low-volume tasks. For production with unpredictable load, I use a ThreadPoolExecutor with an explicit maximum."*

### ⚡ Key Points to Remember

1. **Unlimited threads** — risk of thread explosion ⚠️
2. **No queue** (SynchronousQueue) — every task gets a thread
3. Idle threads die after **60 seconds**
4. Good for: **short-lived, low-volume** tasks
5. **Never** use for unpredictable or high-volume workloads

---

<a id="q42"></a>

## Q42. What is ScheduledExecutorService?

### 🔑 Quick Answer

> An ExecutorService that can **schedule tasks** to run after a delay or at **fixed intervals**. Replacement for the old `Timer` class with proper thread pool support.

### 💻 Code Example

```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(2);

// Run once after 5 seconds
scheduler.schedule(() -> System.out.println("Delayed task"), 5, TimeUnit.SECONDS);

// Run every 10 seconds (fixed rate — includes execution time)
scheduler.scheduleAtFixedRate(() -> {
    System.out.println("Periodic task: " + LocalTime.now());
}, 0, 10, TimeUnit.SECONDS);

// Run with 10-second delay between end of one and start of next
scheduler.scheduleWithFixedDelay(() -> {
    System.out.println("Delayed periodic: " + LocalTime.now());
}, 0, 10, TimeUnit.SECONDS);
```

### 🗣️ How to Explain in Interview

> *"ScheduledExecutorService lets me run tasks after a delay or periodically. scheduleAtFixedRate runs every N seconds regardless of execution time — if the task takes 3 seconds and interval is 10, the next run starts at 10 seconds. scheduleWithFixedDelay waits N seconds after the previous execution finishes — if the task takes 3 seconds and delay is 10, the next run starts at 13 seconds. I use this for health checks, cache refresh, metrics collection. It's superior to Timer because it uses a thread pool and handles exceptions properly."*

### ⚡ Key Points to Remember

1. **schedule()** = one-time delayed execution
2. **scheduleAtFixedRate()** = fixed interval (includes execution time)
3. **scheduleWithFixedDelay()** = fixed gap between executions
4. Replaces **Timer** class (thread pool + better error handling)
5. Use for: **health checks, cache refresh, metrics**

---

<a id="q43"></a>

## Q43. What is Future?

### 🔑 Quick Answer

> A **placeholder for a result** that hasn't been computed yet. Returned by `ExecutorService.submit()`. Call `get()` to block until the result is available. Can check `isDone()` and `cancel()`.

### 💻 Code Example

```java
ExecutorService executor = Executors.newFixedThreadPool(2);

Future<Integer> future = executor.submit(() -> {
    Thread.sleep(2000);  // Simulate long computation
    return 42;
});

// Do other work while computation runs...
System.out.println("Doing other work...");

// Block and get result
int result = future.get();                    // Blocks until done
int result2 = future.get(5, TimeUnit.SECONDS); // Blocks with timeout

System.out.println("Result: " + result);  // 42

// Other methods
boolean done = future.isDone();         // Check if completed
boolean cancelled = future.cancel(true); // Cancel (interrupt if running)
```

### 🗣️ How to Explain in Interview

> *"Future represents a result of an asynchronous computation. When I submit a Callable to ExecutorService, I get a Future back. I can do other work and call future.get() when I need the result — it blocks until the computation is done. The problem with Future is that get() is blocking — no way to register callbacks. That's why CompletableFuture was introduced in Java 8 — it supports thenApply(), thenCompose(), and other non-blocking composition methods."*

### ⚡ Key Points to Remember

1. **Placeholder** for async result
2. `get()` — **blocks** until result ready
3. `get(timeout)` — blocks with **timeout**
4. `isDone()` — check completion
5. Limitation: **blocking only** — no callbacks (use CompletableFuture)

---

<a id="q44"></a>

## Q44. What is CompletableFuture?

### 🔑 Quick Answer

> An enhanced Future with **non-blocking callbacks** and **chaining**. Supports `thenApply()`, `thenCompose()`, `thenCombine()`, `exceptionally()`. The **modern standard** for async programming in Java.

### 📖 Step-by-Step Explanation

```
Future (old):
  result = future.get();  // BLOCKS current thread ❌

CompletableFuture (modern):
  completableFuture
      .thenApply(result -> transform(result))     // Non-blocking chain
      .thenAccept(value -> save(value))            // Non-blocking consume
      .exceptionally(ex -> handleError(ex));       // Error handling
  // Current thread continues immediately ✅
```

### 💻 Code Example

```java
// Async computation with chaining
CompletableFuture<String> future = CompletableFuture
    .supplyAsync(() -> fetchUserFromDB(userId))     // Run async
    .thenApply(user -> user.getEmail())              // Transform (non-blocking)
    .thenApply(email -> sendEmail(email))            // Chain another step
    .exceptionally(ex -> {                           // Handle errors
        log.error("Failed", ex);
        return "fallback@email.com";
    });

// Combine two independent async operations
CompletableFuture<User> userFuture = CompletableFuture.supplyAsync(() -> getUser(id));
CompletableFuture<List<Order>> ordersFuture = CompletableFuture.supplyAsync(() -> getOrders(id));

CompletableFuture<Profile> profileFuture = userFuture
    .thenCombine(ordersFuture, (user, orders) -> new Profile(user, orders));

// Wait for all to complete
CompletableFuture.allOf(future1, future2, future3)
    .thenRun(() -> System.out.println("All done!"));
```

### 🗣️ How to Explain in Interview

> *"CompletableFuture is the modern async API in Java. Unlike Future's blocking get(), CompletableFuture supports non-blocking callbacks. supplyAsync() runs a task on ForkJoinPool. thenApply() transforms the result, thenCompose() for flatMap-style chaining, thenCombine() merges two independent results. exceptionally() handles errors. I use it extensively for calling multiple microservices in parallel — kick off 3 API calls, combine their results, no blocking. allOf() waits for all futures, anyOf() returns when any one completes."*

### ⚡ Key Points to Remember

1. **supplyAsync()** — start async task
2. **thenApply()** — transform result (map)
3. **thenCompose()** — chain dependent async ops (flatMap)
4. **thenCombine()** — merge two independent results
5. **exceptionally()** — error handling
6. **allOf/anyOf** — wait for all/any

---

<a id="q45"></a>

## Q45. What is ForkJoinPool?

### 🔑 Quick Answer

> A specialized thread pool for **divide-and-conquer** tasks. Uses **work-stealing** — idle threads steal tasks from busy threads' queues. Default executor for parallel streams and CompletableFuture.

### 📖 Step-by-Step Explanation

```
Regular ThreadPool:
  T1: [Task1, Task2, Task3]    ← All tasks assigned to T1
  T2: [idle]                   ← Doing nothing!
  T3: [idle]                   ← Doing nothing!
  → Unbalanced, wasted threads

ForkJoinPool (work-stealing):
  T1: [Task1]          ← Working
  T2: [Task2]          ← Working (stole from T1's queue)
  T3: [Task3]          ← Working (stole from T1's queue)
  → Balanced, all threads busy ✅

Divide and conquer:
  Sort 1M items
  ├── Fork: Sort 0-500K
  │   ├── Fork: Sort 0-250K
  │   └── Fork: Sort 250K-500K
  ├── Fork: Sort 500K-1M
  │   ├── Fork: Sort 500K-750K
  │   └── Fork: Sort 750K-1M
  └── Join: Merge all results
```

### 💻 Code Example

```java
// RecursiveTask example: Parallel sum
class SumTask extends RecursiveTask<Long> {
    private final int[] array;
    private final int start, end;
    private static final int THRESHOLD = 10_000;
    
    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            // Small enough — compute directly
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        // Divide into two subtasks
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();   // Submit left to pool
        long rightResult = right.compute();  // Compute right in current thread
        long leftResult = left.join();       // Wait for left
        return leftResult + rightResult;
    }
}

ForkJoinPool pool = new ForkJoinPool();  // Default: CPU cores threads
long total = pool.invoke(new SumTask(data, 0, data.length));
```

### 🗣️ How to Explain in Interview

> *"ForkJoinPool is designed for recursive divide-and-conquer workloads. It uses work-stealing — each thread has its own deque of tasks. When a thread finishes its tasks, it steals from the tail of another thread's deque. This keeps all threads busy with minimal coordination. It's the default pool for Java's parallel streams and CompletableFuture.supplyAsync(). The common pool has threads equal to the number of CPU cores minus one, plus the calling thread. I implement tasks by extending RecursiveTask for results or RecursiveAction for void."*

### ⚡ Key Points to Remember

1. **Work-stealing** — idle threads steal tasks
2. **RecursiveTask<V>** (returns result) / **RecursiveAction** (void)
3. **fork()** = submit subtask, **join()** = wait for result
4. Default pool for **parallel streams** and **CompletableFuture**
5. Default threads = **CPU cores - 1**

---

<a id="q46"></a>

## Q46. What is CountDownLatch?

### 🔑 Quick Answer

> A synchronization aid that lets one or more threads **wait until a set of operations in other threads completes**. Initialize with a count, each operation calls `countDown()`, waiting threads call `await()` which blocks until count reaches zero. **One-time use** — cannot be reset.

### 💻 Code Example

```java
// Main thread waits for 3 services to initialize
CountDownLatch latch = new CountDownLatch(3);

new Thread(() -> { initDatabase();    latch.countDown(); }).start(); // count: 3→2
new Thread(() -> { initCache();       latch.countDown(); }).start(); // count: 2→1
new Thread(() -> { initMessageQueue(); latch.countDown(); }).start(); // count: 1→0

latch.await();  // Main thread blocks until count = 0
System.out.println("All services initialized — starting application!");
```

### 🗣️ How to Explain in Interview

> *"CountDownLatch is a one-shot barrier. I set a count — say 3 for 3 services that need to initialize. Each service thread calls countDown() when done. The main thread calls await() and blocks until the count reaches zero. I use this for application startup — wait for database, cache, and message queue to be ready before accepting requests. Also useful in testing — wait for multiple concurrent test threads to complete. It's one-time use — once the count hits zero, it stays there. For reusable barriers, CyclicBarrier is the alternative."*

### ⚡ Key Points to Remember

1. **await()** = block until count reaches 0
2. **countDown()** = decrement count
3. **One-time use** — cannot be reset
4. Use for: **startup coordination**, **test synchronization**
5. Reusable alternative: **CyclicBarrier**

---

<a id="q47"></a>

## Q47. What is CyclicBarrier?

### 🔑 Quick Answer

> A synchronization point where a **fixed number of threads wait for each other**. When all threads arrive at the barrier, they all proceed simultaneously. Unlike CountDownLatch, it's **reusable** — can be used for multiple rounds.

### 💻 Code Example

```java
// 3 threads must all reach the barrier before any can continue
CyclicBarrier barrier = new CyclicBarrier(3, () -> {
    System.out.println("All threads arrived — proceeding!");  // Barrier action
});

for (int i = 0; i < 3; i++) {
    final int id = i;
    new Thread(() -> {
        System.out.println("Thread " + id + " working on phase 1...");
        Thread.sleep(1000 * id);  // Different speeds
        barrier.await();          // Wait for others ← BLOCKS until 3 arrive
        
        System.out.println("Thread " + id + " working on phase 2...");
        barrier.await();          // Reusable! Wait again for phase 2 ← 
    }).start();
}
```

### 🗣️ How to Explain in Interview

> *"CyclicBarrier is for multi-phase computation where threads need to synchronize between phases. All threads work on phase 1, then wait at the barrier until everyone is done, then all move to phase 2 together. Unlike CountDownLatch which is one-shot, CyclicBarrier is reusable — it resets after all threads arrive. I optionally provide a barrier action that runs when all threads arrive — like a summary computation between phases."*

| | CountDownLatch | CyclicBarrier |
|--|---------------|--------------|
| **Reusable** | ❌ No | ✅ Yes |
| **Who waits** | Some threads wait for others | All threads wait for each other |
| **Reset** | Cannot | Automatic after all arrive |
| **Action on completion** | No | Optional barrier action |

### ⚡ Key Points to Remember

1. All threads **wait for each other** at the barrier
2. **Reusable** — auto-resets after each round
3. Optional **barrier action** (runs when all arrive)
4. Use for **multi-phase parallel computation**
5. CountDownLatch = one-shot, CyclicBarrier = reusable

---

<a id="q48"></a>

## Q48. What is Semaphore?

### 🔑 Quick Answer

> Controls access to a resource by maintaining a set of **permits**. `acquire()` takes a permit (blocks if none available), `release()` returns a permit. Used to **limit concurrent access** — like a parking lot with N spots.

### 💻 Code Example

```java
// Only 3 threads can access the resource simultaneously
Semaphore semaphore = new Semaphore(3);

for (int i = 0; i < 10; i++) {
    final int id = i;
    new Thread(() -> {
        try {
            semaphore.acquire();  // Get permit (blocks if none available)
            System.out.println("Thread " + id + " — accessing resource");
            Thread.sleep(2000);   // Use resource
        } finally {
            semaphore.release();  // Return permit
        }
    }).start();
}
// Only 3 threads access resource at a time, others wait
```

### 🗣️ How to Explain in Interview

> *"Semaphore is like a parking lot — it has a fixed number of permits. A thread calls acquire() to get a permit — if all permits are taken, it blocks. When done, it calls release() to return the permit. I use this to limit concurrent access to a resource — like limiting database connections to 10, or API calls to 5 concurrent requests. Unlike synchronized which allows exactly one thread, Semaphore allows N threads. A binary semaphore with 1 permit acts like a mutex."*

### ⚡ Key Points to Remember

1. **acquire()** = get permit (blocks if none)
2. **release()** = return permit
3. Permits N = **N concurrent accesses**
4. Use for: **rate limiting**, **connection pools**, **resource throttling**
5. **Binary semaphore** (permits=1) ≈ mutex

---

<a id="q49"></a>

## Q49. What is Phaser?

### 🔑 Quick Answer

> A flexible synchronization barrier that supports **dynamic party registration** and **multiple phases**. Like CyclicBarrier but parties can **join and leave** at any phase. Most advanced barrier in Java.

### 💻 Code Example

```java
Phaser phaser = new Phaser(1);  // Register self (coordinator)

for (int i = 0; i < 3; i++) {
    phaser.register();  // Dynamically register new party
    final int id = i;
    new Thread(() -> {
        System.out.println("Thread " + id + " phase 0 work");
        phaser.arriveAndAwaitAdvance();  // Wait for all → phase 1
        
        System.out.println("Thread " + id + " phase 1 work");
        phaser.arriveAndAwaitAdvance();  // Wait for all → phase 2
        
        phaser.arriveAndDeregister();    // Leave the phaser
    }).start();
}

phaser.arriveAndDeregister();  // Coordinator deregisters
```

### 🗣️ How to Explain in Interview

> *"Phaser is the most flexible barrier. Unlike CyclicBarrier where the number of parties is fixed at construction, Phaser allows dynamic registration and deregistration. Threads can join mid-execution with register() and leave with arriveAndDeregister(). It supports multiple phases — arriveAndAwaitAdvance() waits for all parties and advances to the next phase. I use it when the number of threads changes between phases — like a map-reduce pipeline where some threads finish early."*

### ⚡ Key Points to Remember

1. **Dynamic parties** — threads can join/leave at any phase
2. `arriveAndAwaitAdvance()` = wait + advance to next phase
3. `arriveAndDeregister()` = leave the phaser
4. More flexible than **CyclicBarrier**
5. Use when **thread count varies** between phases

---

<a id="q50"></a>

## Q50. Difference between CountDownLatch, CyclicBarrier, Semaphore, and Phaser?

### 🔑 Quick Answer

| Feature | CountDownLatch | CyclicBarrier | Semaphore | Phaser |
|---------|---------------|--------------|-----------|--------|
| **Purpose** | Wait for events | Wait for each other | Limit access | Dynamic barrier |
| **Reusable** | ❌ No | ✅ Yes | ✅ Yes | ✅ Yes |
| **Parties** | Fixed | Fixed | N/A (permits) | Dynamic |
| **Reset** | Never | Auto | N/A | Auto |

### 🗣️ How to Explain in Interview

> *"Each solves a different coordination problem. CountDownLatch: 'wait until N things happen' — like waiting for 3 services to start; one-shot, can't reuse. CyclicBarrier: 'all threads wait for each other' — like a meeting point between computation phases; reusable. Semaphore: 'limit N concurrent accesses' — like a connection pool; not a barrier, it's a rate limiter. Phaser: 'dynamic barrier' — like CyclicBarrier but threads can join/leave between phases."*

### ⚡ Key Points to Remember

1. **CountDownLatch** = "wait for N events" (one-shot)
2. **CyclicBarrier** = "all wait for all" (reusable phases)
3. **Semaphore** = "limit N concurrent" (resource throttle)
4. **Phaser** = "dynamic parties + phases" (most flexible)
5. Choose based on: **one-shot vs reusable**, **fixed vs dynamic parties**

---

> **🎯 Navigation:** [← Thread Communication (Q29-36)](03-thread-communication.md) | [Next → Concurrent Collections (Q51-56)](05-concurrent-collections.md) | [📋 All Sections](README.md)
