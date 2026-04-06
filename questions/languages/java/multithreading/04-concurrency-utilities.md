# ⚙️ Concurrency Utilities (Q37–Q50)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q37"></a>
## Q37. What is ExecutorService?

### 📝 One-Liner
> Framework for managing thread pools — submit tasks, executor handles thread lifecycle.

### 🔑 Quick Answer
> `ExecutorService` manages a **pool of threads** for you — submit tasks via `submit()`/`execute()`, the pool handles thread creation, reuse, and lifecycle. **Never create raw threads in production** — always use ExecutorService. *(Thread pool manager — tum kaam do, woh thread sambhalega)*

### 📖 How It Works
```
Without ExecutorService:
  new Thread(task1).start();  // new thread every time ❌
  new Thread(task2).start();  // expensive! 1MB stack per thread
  new Thread(task3).start();  // 10000 tasks = 10000 threads = OOM 💀

With ExecutorService:
  ExecutorService pool = Executors.newFixedThreadPool(5);
  pool.submit(task1);  // reuses thread from pool ✅
  pool.submit(task2);  // no new thread created
  pool.submit(task3);  // efficient!
  
  *(5 threads hai — 10000 kaam bhi 5 threads se ho jaayenge, bari bari)*
```

### 💻 Code
```java
ExecutorService executor = Executors.newFixedThreadPool(10);

// Submit Runnable (fire-and-forget)
executor.execute(() -> processOrder(order));

// Submit Callable (get result)
Future<Report> future = executor.submit(() -> generateReport(params));
Report report = future.get(30, TimeUnit.SECONDS);  // timeout lagao!

// Graceful shutdown
executor.shutdown();  // stop accepting, finish running
if (!executor.awaitTermination(60, TimeUnit.SECONDS)) {
    executor.shutdownNow();  // force stop
}
```

### 🗣️ Answering Approach
> *"ExecutorService is the production standard for thread management in Java. I never create raw threads — always use a pool. It handles thread lifecycle, reuse, and queuing. I use execute() for fire-and-forget tasks and submit() when I need a Future result. For shutdown, I always call shutdown() followed by awaitTermination(). In Spring, ThreadPoolTaskExecutor wraps this with additional lifecycle management."*

### ⚠️ Pitfalls / Gotchas
- `Executors.newCachedThreadPool()` = **no upper limit** → can create thousands of threads *(OOM risk!)*
- `Executors.newFixedThreadPool()` = **unbounded queue** → can fill memory *(OOM risk!)*
- In production: use `ThreadPoolExecutor` directly with **bounded queue** *(hamesha limit lagao)*
- Forgetting to `shutdown()` → JVM won't exit *(non-daemon threads keep JVM alive)*

### 🎯 Tricky Interview Qs
**Q: execute() vs submit() — what's the difference?**
> `execute(Runnable)` — void, exceptions lost unless UncaughtExceptionHandler set. `submit()` — returns Future, exceptions captured in Future.get(). *(submit = result milega; execute = fire-and-forget)*

### ⚡ Remember
1. **Never raw threads** — always pool *(production rule #1)*
2. `execute()` = void | `submit()` = Future
3. Always **shutdown()** + awaitTermination *(warna JVM nahi rukega)*
4. Beware **unbounded** pools/queues *(OOM ka darr)*
5. Spring: use **ThreadPoolTaskExecutor**

### 🔗 Follow-ups
→ [Q38. Thread pool concept](#q38) → [Q39. Pool types](#q39)

---

<a id="q38"></a>
## Q38. What is a thread pool?

### 📝 One-Liner
> Pre-created set of reusable threads — tasks go to a queue, threads pick and execute.

### 🔑 Quick Answer
> A fixed set of **pre-created threads** that wait for tasks. When a task arrives, an idle thread picks it up. When done, the thread returns to the pool — **reused, not destroyed**. *(Threads pehle se ready hain — kaam aaya, ek ne utha liya, doosra kaam aaya, doosre ne utha liya)*

### 📖 How It Works
```
Thread Pool (size = 3):
  ┌─────────────────────────────────────────┐
  │ Queue: [Task-4] [Task-5] [Task-6]       │
  │        (waiting for free thread)          │
  ├─────────────────────────────────────────┤
  │ Thread-1: [executing Task-1]             │
  │ Thread-2: [executing Task-2]             │
  │ Thread-3: [executing Task-3]             │
  │ (Task done → pick next from queue)       │
  └─────────────────────────────────────────┘
  *(3 threads hain, 6 kaam aaye — 3 chal rahe, 3 line mein)*
```

### ⚡ Remember
1. **Reuses** threads — no creation overhead
2. **Bounded** = predictable resource use *(kitne threads chahiye pata hai)*
3. Tasks **queued** when all threads busy
4. Foundation of ExecutorService

### 🔗 Follow-ups
→ [Q39. Pool types](#q39) → [Q70. Pool tuning](#q70)

---

<a id="q39"></a>
## Q39. What are different types of thread pools?

### 📝 One-Liner
> Fixed (bounded), Cached (unbounded), Single (one thread), Scheduled (delayed/periodic).

### 🆚 vs. Comparison
| Pool Type | Threads | Queue | Use Case | Production Safe? |
|-----------|---------|-------|----------|-----------------|
| **FixedThreadPool** | Fixed N | Unbounded ⚠️ | General purpose | ⚠️ Queue grows |
| **CachedThreadPool** | 0 → ∞ ⚠️ | SynchronousQueue | Short-lived tasks | ❌ OOM risk |
| **SingleThread** | 1 | Unbounded | Sequential tasks | ⚠️ Queue grows |
| **ScheduledThread** | Fixed N | DelayedWorkQueue | Cron-like tasks | ✅ |
| **ThreadPoolExecutor** | Configurable | Configurable | **Production** ⭐ | ✅ |

### ⚠️ Pitfalls / Gotchas
- **Never use Executors factory** in production (Alibaba coding guidelines) — always use `ThreadPoolExecutor` directly
- CachedThreadPool: **unlimited threads** → 10000 tasks = 10000 threads = OOM *(thread ka koi limit nahi — khatarnak)*
- FixedThreadPool: **unbounded queue** → 10M tasks queued = OOM *(queue ka koi limit nahi — khatarnak)*

### 🗣️ Answering Approach
> *"I never use Executors factory methods in production. CachedThreadPool creates unlimited threads — OOM risk. FixedThreadPool has an unbounded queue — OOM risk. I always create ThreadPoolExecutor directly with bounded core/max pool sizes and a bounded queue. This gives me full control over resource limits and rejection policies."*

### ⚡ Remember
1. **Fixed** = stable, but unbounded queue *(queue se OOM)*
2. **Cached** = unlimited threads *(threads se OOM)*
3. **Production**: always `ThreadPoolExecutor` with bounds ⭐
4. **Never** Executors factory in production code

### 🔗 Follow-ups
→ [Q40. FixedThreadPool](#q40) → [Q41. CachedThreadPool](#q41)

---

<a id="q40"></a>
## Q40. How does FixedThreadPool work?

### 📝 One-Liner
> N fixed threads + unbounded LinkedBlockingQueue — threads reused, extra tasks queued forever.

### 🔑 Quick Answer
> Creates exactly **N threads**. Never more, never less. Excess tasks go to an **unbounded LinkedBlockingQueue**. *(N threads fix — zyada kaam aaye toh queue mein jaayega, nayi thread nahi banegi)*

### ⚠️ Pitfalls / Gotchas
- **Unbounded queue** → if tasks arrive faster than processed → memory grows indefinitely → **OOM** *(queue bharti jaayegi — memory khatam)*
- In production: use `ThreadPoolExecutor` with **bounded queue** + rejection policy

### ⚡ Remember
1. **Fixed N threads** — predictable
2. **Unbounded queue** ⚠️ — OOM risk
3. Production: use bounded queue instead

### 🔗 Follow-ups
→ [Q41. CachedThreadPool](#q41)

---

<a id="q41"></a>
## Q41. How does CachedThreadPool work?

### 📝 One-Liner
> Creates new thread for every task if none idle — no upper limit, 60s idle timeout.

### 🔑 Quick Answer
> **Zero core threads**, max = Integer.MAX_VALUE. Uses SynchronousQueue (no buffering). Creates new thread if no idle thread available. Idle threads die after 60s. *(Koi limit nahi — har kaam ke liye naya thread, 1 minute baad idle thread mar jaata hai)*

### ⚠️ Pitfalls / Gotchas
- **No upper limit** → 10000 concurrent tasks = 10000 threads = **OOM** *(khatarnak — production mein mat use karo)*
- Good ONLY for short-lived, low-volume tasks

### ⚡ Remember
1. **Unlimited threads** ⚠️ — OOM risk
2. SynchronousQueue — no buffering
3. Idle threads die after 60s
4. **Never** in production *(bahut khatarnak)*

### 🔗 Follow-ups
→ [Q42. ScheduledExecutorService](#q42)

---

<a id="q42"></a>
## Q42. What is ScheduledExecutorService?

### 📝 One-Liner
> Thread pool that runs tasks after a delay or at fixed intervals — replacement for Timer.

### 🔑 Quick Answer
> Runs tasks **after a delay** or **at regular intervals**. Replacement for `java.util.Timer`. Thread-safe and handles exceptions better than Timer. *(Cron job jaisa — "har 5 second mein ye karo" ya "2 second baad ye karo")*

### 💻 Code
```java
ScheduledExecutorService scheduler = Executors.newScheduledThreadPool(3);

// Run once after 5 seconds
scheduler.schedule(() -> sendReminder(), 5, TimeUnit.SECONDS);

// Run every 10 seconds (start-to-start)
scheduler.scheduleAtFixedRate(() -> healthCheck(), 0, 10, TimeUnit.SECONDS);

// Run 10 seconds after previous completes (end-to-start)
scheduler.scheduleWithFixedDelay(() -> cleanup(), 0, 10, TimeUnit.SECONDS);
```

### 🆚 vs. Comparison
| | Timer | ScheduledExecutorService |
|-|-------|------------------------|
| Threads | Single *(ek fail = sab fail)* | Pool ✅ |
| Exception | Kills timer ❌ | Isolated ✅ |
| **Use** | Never ❌ | **Always** ⭐ |

### ⚡ Remember
1. `schedule()` = one-time delay
2. `scheduleAtFixedRate()` = every N seconds (start-to-start)
3. `scheduleWithFixedDelay()` = N seconds after finish (end-to-start)
4. **Replace Timer** always *(Timer mein ek fail toh sab fail)*

### 🔗 Follow-ups
→ [Q43. Future](#q43)

---

<a id="q43"></a>
## Q43. What is Future in Java?

### 📝 One-Liner
> Placeholder for a result not yet available — get() blocks until the async task completes.

### 🔑 Quick Answer
> `Future<V>` represents a **result that will be available later**. Call `.get()` to wait for it. It's **blocking** — your thread hangs until the result is ready. *(Abhi result nahi hai — baad mein aayega, get() karo toh ruk ke wait karega)*

### 💻 Code
```java
Future<Report> future = executor.submit(() -> generateReport());

// Blocking! (ruk jaayega jab tak result nahi aata)
Report report = future.get(30, TimeUnit.SECONDS);

// Other operations
boolean done = future.isDone();       // check without blocking
boolean cancelled = future.cancel(true); // cancel task
```

### ⚠️ Pitfalls / Gotchas
- `get()` **blocks indefinitely** without timeout *(timeout mat bhulna — warna hang)*
- Can't compose or chain Futures *(future1 ka result future2 ko dena mushkil)*
- Use **CompletableFuture** instead ⭐

### ⚡ Remember
1. `get()` = **blocking** *(ruk jaata hai)*
2. Always use **timeout** with get() *(warna hang)*
3. Limited API — can't chain or compose
4. Upgrade to **CompletableFuture** ⭐

### 🔗 Follow-ups
→ [Q44. CompletableFuture](#q44)

---

<a id="q44"></a>
## Q44. What is CompletableFuture?

### 📝 One-Liner
> Non-blocking async programming with chaining, composition, and error handling — the production standard.

### 🔑 Quick Answer
> `CompletableFuture` is **Future on steroids** — supports chaining (`thenApply`), composition (`thenCompose`, `thenCombine`), error handling (`exceptionally`), and works **non-blocking**. *(Future ka upgraded version — chain kar sakte ho, combine kar sakte ho, error handle kar sakte ho)*

### 📖 How It Works
```
Future (old way — blocking):
  Result a = futureA.get();     // BLOCKS
  Result b = futureB.get();     // BLOCKS
  Result c = combine(a, b);     // sequential ❌

CompletableFuture (new way — non-blocking):
  CompletableFuture.supplyAsync(() -> fetchA())
    .thenCombine(supplyAsync(() -> fetchB()), (a, b) -> combine(a, b))
    .thenApply(result -> transform(result))
    .exceptionally(ex -> fallback(ex));
  // Everything non-blocking, chained! ✅
  *(Koi wait nahi — sab chain mein chalta hai, error bhi handle)*
```

### 💻 Code
```java
// Parallel API calls — non-blocking
CompletableFuture<User> userFuture = 
    CompletableFuture.supplyAsync(() -> userService.getUser(id));

CompletableFuture<List<Order>> ordersFuture = 
    CompletableFuture.supplyAsync(() -> orderService.getOrders(id));

// Combine results when both ready
CompletableFuture<UserProfile> profileFuture = 
    userFuture.thenCombine(ordersFuture, (user, orders) -> 
        new UserProfile(user, orders));

// Chain transformations
profileFuture
    .thenApply(profile -> enrichWithRecommendations(profile))
    .thenAccept(profile -> sendToClient(profile))
    .exceptionally(ex -> {
        log.error("Failed", ex);
        return null;
    });

// Wait for multiple
CompletableFuture.allOf(future1, future2, future3).join();
```

### ⚠️ Pitfalls / Gotchas
- Default pool is **ForkJoinPool.commonPool()** — shared across app *(sab ke saath share — busy ho sakta hai)*
- Always pass custom executor: `supplyAsync(task, myExecutor)` *(apna pool do)*
- `join()` vs `get()`: join throws unchecked exception, get throws checked *(join = RuntimeException, get = checked)*
- Forgetting error handling → exceptions silently swallowed *(exceptionally lagao — warna error gayab)*

### 🎯 Tricky Interview Qs
**Q: thenApply vs thenCompose?**
> `thenApply(fn)` = map (T → U). `thenCompose(fn)` = flatMap (T → CompletableFuture\<U\>). *(thenApply = seedha transform; thenCompose = jab transform bhi async ho)*

**Q: What's the default thread pool?**
> ForkJoinPool.commonPool() — **always specify custom executor** in production.

### 🗣️ Answering Approach
> *"CompletableFuture is my go-to for async operations. I use supplyAsync to kick off async tasks, thenCombine to merge parallel results, thenApply for transformations, and exceptionally for error handling. In production, I always pass a custom ThreadPoolTaskExecutor — never rely on the common ForkJoinPool which is shared. For multiple parallel calls, allOf() waits for all to complete."*

### ⚡ Remember
1. `supplyAsync()` = start async task *(kaam shuru karo)*
2. `thenApply()` = transform result *(result badlo)*
3. `thenCombine()` = merge two futures *(do results jodo)*
4. `exceptionally()` = error handling *(galti pakdo)*
5. Always **custom executor** *(apna pool do)* ⭐

### 🔗 Follow-ups
→ [Q45. ForkJoinPool](#q45)

---

<a id="q45"></a>
## Q45. What is ForkJoinPool?

### 📝 One-Liner
> Thread pool optimized for divide-and-conquer — splits big tasks into subtasks, uses work-stealing.

### 🔑 Quick Answer
> ForkJoinPool uses **work-stealing** — idle threads steal tasks from busy threads' queues. Optimized for **recursive divide-and-conquer** tasks. Default pool for parallel streams and CompletableFuture. *(Bada kaam chhoto mein todo, idle thread doosre ka kaam chura le — efficient!)*

### 📖 How It Works
```
Fork-Join pattern:
  Big Task (1M records)
    ├── fork → Subtask-1 (500K)
    │            ├── fork → Sub-sub-1 (250K)
    │            └── fork → Sub-sub-2 (250K)
    └── fork → Subtask-2 (500K)
                 ├── fork → Sub-sub-3 (250K)
                 └── fork → Sub-sub-4 (250K)
    
  join ← merge all results
  *(Bada kaam todo → chhote kaam parallel karo → sab ka result jodo)*

Work-stealing:
  Thread-1: [task][task][task]    ← busy
  Thread-2: [idle]               ← steals from Thread-1's queue!
  *(Khaali thread doosre ki queue se kaam le leta hai)*
```

### 💻 Code
```java
// RecursiveTask<V> — returns result
class SumTask extends RecursiveTask<Long> {
    private final int[] arr;
    private final int start, end;
    private static final int THRESHOLD = 10000;
    
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += arr[i];
            return sum;  // base case — direct compute
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(arr, start, mid);
        SumTask right = new SumTask(arr, mid, end);
        left.fork();  // submit to pool (async)
        return right.compute() + left.join();  // compute right, wait for left
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Don't use for I/O tasks** — threads will block, pool starves *(I/O wala kaam yahan mat karo)*
- ForkJoinPool.commonPool() is **shared** — one bad task blocks everything
- Parallel streams use commonPool() — can cause issues *(parallel stream bhi isi pool mein chalta hai)*

### ⚡ Remember
1. **Work-stealing** = idle threads steal from busy *(efficient)*
2. **Divide-and-conquer** pattern (fork → compute → join)
3. **CPU-bound** tasks only *(I/O tasks ke liye mat use karo)*
4. Default for **parallel streams** + CompletableFuture
5. commonPool() is **shared** — pass custom pool in production

### 🔗 Follow-ups
→ [Q46. CountDownLatch](#q46)

---

<a id="q46"></a>
## Q46. What is CountDownLatch?

### 📝 One-Liner
> One-shot barrier: count down from N, waiting thread(s) released when count hits zero.

### 🔑 Quick Answer
> CountDownLatch makes one thread **wait until N events complete**. Call `countDown()` from each completing thread, `await()` blocks until counter reaches 0. **One-shot** — can't reset. *(N kaam hone do, phir aage badho — ek baar use, dobaara nahi)*

### 💻 Code
```java
CountDownLatch latch = new CountDownLatch(3);  // wait for 3 tasks

// 3 worker threads
for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        doWork();
        latch.countDown();  // "mera kaam ho gaya" (3→2→1→0)
    });
}

latch.await(30, TimeUnit.SECONDS);  // main thread waits until count = 0
System.out.println("All 3 done!");
```

### ⚠️ Pitfalls / Gotchas
- **Cannot reset** — once 0, stays 0, can't reuse *(ek baar use — phir naya banana padega)*
- If a thread crashes without countDown() → **latch never reaches 0** → await() hangs forever *(timeout lagao!)*
- Always use `await(timeout)` *(warna hang)*

### 🗣️ Answering Approach
> *"CountDownLatch is for waiting until N parallel tasks complete. I used it in a microservice health check — at startup, I waited for 5 downstream services to respond before marking the app as ready. Each health check thread calls countDown() on completion, and the main thread awaits with a timeout."*

### ⚡ Remember
1. **One-shot** — can't reset *(ek baar ka)*
2. `countDown()` = decrement | `await()` = wait for 0
3. Always **timeout** on await *(warna hang)*
4. Use for: "wait until all N tasks finish"

### 🔗 Follow-ups
→ [Q47. CyclicBarrier](#q47) → [Q50. Comparison](#q50)

---

<a id="q47"></a>
## Q47. What is CyclicBarrier?

### 📝 One-Liner
> Reusable barrier: N threads wait for each other, then all proceed together — can trigger a barrier action.

### 🔑 Quick Answer
> CyclicBarrier makes **all N threads wait at a point** until everyone arrives, then all proceed. **Reusable** — resets automatically after each barrier. Optional **barrier action** runs when all arrive. *(Sab ek jagah ruko — last wala aaya, sab ek saath aage badho — phir se use kar sakte ho)*

### 💻 Code
```java
CyclicBarrier barrier = new CyclicBarrier(3, () -> 
    System.out.println("All 3 arrived — merging results!"));  // barrier action

// 3 threads — each arrives at barrier
for (int i = 0; i < 3; i++) {
    executor.submit(() -> {
        computePartialResult();
        barrier.await();  // "main aa gaya, baaki ka wait karo"
        // all 3 proceed together after this point ✅
    });
}
```

### 🆚 vs. Comparison
| Feature | CountDownLatch | CyclicBarrier |
|---------|---------------|--------------|
| Reset | ❌ One-shot *(ek baar)* | ✅ Reusable *(baar baar)* |
| Who waits | One thread waits for N | N threads wait for each other |
| Action on complete | ❌ | ✅ Barrier action |
| Use case | "Wait for tasks" | "Meet at a point, go together" |

### ⚡ Remember
1. **Reusable** — resets after each barrier *(baar baar use)*
2. All threads **wait for each other** *(sab ek dusre ka wait)*
3. Optional **barrier action** when all arrive
4. Good for: iterative algorithms, phased computation

### 🔗 Follow-ups
→ [Q48. Semaphore](#q48) → [Q50. Comparison](#q50)

---

<a id="q48"></a>
## Q48. What is Semaphore?

### 📝 One-Liner
> Controls concurrent access with permits — acquire to enter, release to exit; limits parallel usage.

### 🔑 Quick Answer
> Semaphore allows up to **N threads** access simultaneously using **permits**. `acquire()` takes a permit (blocks if none available), `release()` returns it. *(N pass hain — jiske paas pass woh andar, baaki bahar wait)*

### 💻 Code
```java
Semaphore semaphore = new Semaphore(3);  // 3 permits = max 3 concurrent

public void accessResource() throws InterruptedException {
    semaphore.acquire();  // take permit (block if none left)
    try {
        useSharedResource();  // max 3 threads here at once
    } finally {
        semaphore.release();  // ALWAYS release in finally!
    }
}

// With timeout — don't wait forever
if (semaphore.tryAcquire(5, TimeUnit.SECONDS)) { ... }
```

### ⚠️ Pitfalls / Gotchas
- **Always release in finally** *(bhool gaye toh permit kho gaya — koi nahi jaa payega)*
- Release without acquire → permit count **increases beyond initial** *(galti se zyada release — limit badh jaayegi)*

### 🗣️ Answering Approach
> *"Semaphore controls concurrent access using permits. I used it to limit connections to an external API that allowed max 5 concurrent calls. acquire() takes a permit, release() returns it. Unlike locks, Semaphore doesn't have ownership — any thread can release. I always release in finally and use tryAcquire with timeout."*

### ⚡ Remember
1. **N permits** = max N concurrent *(N pass — N andar)*
2. `acquire()` = take | `release()` = return
3. **Always release in finally** ⭐
4. No ownership — any thread can release

### 🔗 Follow-ups
→ [Q49. Phaser](#q49) → [Q50. Comparison](#q50)

---

<a id="q49"></a>
## Q49. What is Phaser?

### 📝 One-Liner
> Flexible barrier with dynamic party registration — threads can join/leave between phases.

### 🔑 Quick Answer
> `Phaser` is like CyclicBarrier but with **dynamic parties** — threads can register/deregister between phases. Supports multiple phases. Most flexible synchronizer. *(CyclicBarrier jaisa par log beech mein aa sakte hain aur jaa sakte hain)*

### 💻 Code
```java
Phaser phaser = new Phaser(1);  // 1 = main thread registered

for (int i = 0; i < 3; i++) {
    phaser.register();  // dynamic registration!
    executor.submit(() -> {
        doPhase1();
        phaser.arriveAndAwaitAdvance();  // phase 1 barrier
        doPhase2();
        phaser.arriveAndDeregister();    // done, leave
    });
}

phaser.arriveAndDeregister();  // main thread leaves
```

### ⚡ Remember
1. **Dynamic** parties — register/deregister anytime
2. **Multiple phases** — barrier resets each phase
3. Most **flexible** but most complex
4. Use when party count changes between phases

### 🔗 Follow-ups
→ [Q50. Comparison of all four](#q50)

---

<a id="q50"></a>
## Q50. Compare CountDownLatch, CyclicBarrier, Semaphore, and Phaser

### 📝 One-Liner
> Latch = one-shot count; Barrier = reusable meet-point; Semaphore = permit-based access; Phaser = dynamic barrier.

### 🆚 vs. Comparison
| Feature | CountDownLatch | CyclicBarrier | Semaphore | Phaser |
|---------|---------------|--------------|-----------|--------|
| Reusable | ❌ One-shot | ✅ | ✅ | ✅ |
| Parties | Fixed | Fixed | N/A (permits) | **Dynamic** |
| Pattern | Wait for N tasks | N threads meet | Limit concurrency | Phased execution |
| Analogy | *N kaam hone do* | *Sab ek jagah milo* | *N pass hain* | *Phase wise kaam* |

### 🗣️ Answering Approach
> *"CountDownLatch: one thread waits for N events — I use for startup coordination. CyclicBarrier: N threads wait for each other — for parallel computation phases. Semaphore: limits concurrent access to a resource — like connection pool limiting. Phaser: like a reusable barrier with dynamic parties — when threads join and leave between phases. Most of my production code uses CountDownLatch for simple waiting and Semaphore for rate limiting."*

### 🎯 Tricky Interview Qs
**Q: Can CountDownLatch be reused?**
> No — once count reaches 0, it's done. Use CyclicBarrier for reusable. *(Ek baar zero = khatam, naya banana padega)*

**Q: Can Semaphore be used as a mutex?**
> Yes — Semaphore(1) is effectively a mutex. *(1 permit = ek time pe ek thread — lock jaisa)*

### ⚡ Remember
1. **Latch** = one-shot, wait for N *(ek baar ka countdown)*
2. **Barrier** = reusable, N wait for each other *(sab milo, phir chalo)*
3. **Semaphore** = N permits *(N pass)*
4. **Phaser** = dynamic + phased *(log aate jaate hain)*
5. Interview: know **when to use which** ⭐

### 🔗 Follow-ups
→ [Q90. Limiting concurrent threads](#q90)

---

> **🎯 Navigation:** [← Thread Communication (Q29-36)](03-thread-communication.md) | [Next → Concurrent Collections (Q51-56)](05-concurrent-collections.md) | [📋 All Sections](README.md)
