# 🔥 Production Debugging — JVM, Memory & Performance (Q1–Q5)

> **"Production just broke. Let's see how you think."**
> These are scenario-based diagnostic questions — show the interviewer your reasoning: Symptoms → Hypothesis → Diagnosis → Fix.

---

<a id="q1"></a>
## Q1. Your Java API suddenly starts returning very slow responses after a traffic spike. What internal issue could cause this?

### 📝 One-Liner
Most likely **GC thrashing** (heap nearly full → constant Full GC pauses) or **thread/connection pool exhaustion** (all threads busy → requests queue up) — both triggered by traffic exceeding capacity.

### 🔑 Quick Answer
**Three main suspects** after a traffic spike:

**(1) GC Pressure / Thrashing**: spike creates massive object allocation → Young Gen fills fast → frequent Minor GCs → objects promoted to Old Gen → Old Gen fills → Full GC (stop-the-world) → if GC can't reclaim enough, it runs repeatedly spending >90% of time in GC → all request threads frozen during GC pauses. **Symptom**: response times spike uniformly across ALL endpoints. CPU high but app frozen.

**(2) Thread Pool Exhaustion**: Tomcat default = 200 threads. Spike sends 500 concurrent requests → 200 served, 300 queued. If those 200 are slow (waiting on DB/downstream), queue backs up → timeout cascade. **Symptom**: specific slow endpoints degrade first. Thread dump shows 200 threads BLOCKED or WAITING.

**(3) Connection Pool Saturation**: HikariCP default = 10 connections. 200 threads compete for 10 DB connections → 190 threads wait → request latency = wait time + query time. **Symptom**: DB query time looks normal, but end-to-end latency huge. *(Traffic spike ke baad slow — GC thrash, thread pool full, ya DB connection pool chhota)*

### 📖 How to Diagnose
```
Step-by-step Production Diagnosis:

1. CHECK METRICS DASHBOARD (Grafana):
   → CPU high + GC pause spikes?          → GC problem (go to step 2a)
   → Thread count at max?                  → Thread pool exhaustion (go to step 2b)
   → DB connection wait time high?         → Connection pool saturation (go to step 2c)

2a. GC PROBLEM:
   $ jstat -gcutil <pid> 1000              # GC stats every 1s
   → Old Gen (O) at 95%+ → Full GC every few seconds
   $ jmap -histo:live <pid> | head -20     # top memory consumers
   $ jcmd <pid> GC.heap_dump /tmp/dump.hprof  # heap dump
   → Analyze with Eclipse MAT → find leak or resize heap

2b. THREAD POOL EXHAUSTION:
   $ jstack <pid> > thread_dump.txt
   $ grep -c "RUNNABLE\|BLOCKED\|WAITING" thread_dump.txt
   → If 200 threads BLOCKED on same lock → lock contention
   → If 200 threads WAITING on I/O → slow downstream service
   → Increase pool or fix the slow dependency

2c. CONNECTION POOL SATURATION:
   HikariCP metric: hikaricp_connections_pending > 0
   → Increase pool size: spring.datasource.hikari.maximum-pool-size
   → But check: do queries NEED to be faster? (indexing, caching?)

Timeline:
  Normal:    Req → Thread (1ms) → DB conn (1ms) → Query (5ms) → Response
  Saturated: Req → Thread WAIT 3s → DB conn WAIT 5s → Query (5ms) → Response 8s+
```

### 🗣️ How to Say in Interview
"When an API slows down after a traffic spike, I check three things in order. First, GC metrics — if the JVM is spending more than 10% of time in garbage collection with Old Gen near capacity, we're GC thrashing. I'd take a heap dump and analyze with MAT to find if there's a memory leak or if we simply need more heap. Second, I check thread dumps — if all 200 Tomcat threads are BLOCKED or WAITING, the pool is exhausted, usually because a downstream service slowed down and threads are stuck waiting on I/O. The fix there is either increasing the pool or adding timeouts and circuit breakers to the slow dependency. Third, I check HikariCP metrics — if connection pending count is high, the DB connection pool is the bottleneck. I'd increase the pool size but also check if queries can be optimized or cached to reduce DB pressure."

### 💻 Code — Preventing the Issue
```java
// application.yml — properly sized pools
server:
  tomcat:
    threads:
      max: 200
      min-spare: 20
    accept-count: 100     # OS-level queue when all threads busy

spring:
  datasource:
    hikari:
      maximum-pool-size: 30    # formula: threads / avg_queries_per_request
      minimum-idle: 10
      connection-timeout: 5000  # fail fast, don't wait 30s (default)
      leak-detection-threshold: 30000  # log if connection held > 30s

# Proactive: GC logging always enabled
# JVM args: -Xlog:gc*:file=gc.log:time,level,tags -XX:+HeapDumpOnOutOfMemoryError
```

### ⚠️ Related Pitfalls
- **Don't just increase heap** — if it's a leak, more heap just delays the crash *(Heap badhane se leak fix nahi hota — sirf crash postpone hota hai)*
- **Default connection timeout is 30s** (HikariCP) — in a spike, 200 threads waiting 30s each = disaster. Set to 3-5s
- **Tomcat accept-count** — after threads AND queue full, OS drops connections. Monitor `tomcat.connections.current`

### ⚡ Remember
- **GC thrash** → `jstat -gcutil`, heap dump, MAT analysis
- **Thread pool full** → `jstack`, thread dump, look for BLOCKED/WAITING
- **Connection pool full** → HikariCP `pending` metric, reduce timeout, increase pool
- Always enable: **GC logs** + **HeapDumpOnOutOfMemoryError** + **`/actuator/prometheus`**
- Size pools: Tomcat threads > HikariCP connections > downstream capacity

### 🔗 Cross-References
- core/03 → JVM GC Tuning (G1/ZGC selection, heap sizing, JFR)
- core/01 Q-OOM → OutOfMemoryError causes and MAT analysis

---

<a id="q2"></a>
## Q2. After deploying a small change, the JVM memory usage suddenly doubles. What could explain this?

### 📝 One-Liner
Most likely a **memory leak introduced by the change** — a growing collection/cache without eviction, an unclosed resource, a classloader leak from a library upgrade, or accidentally loading a much larger dataset into memory.

### 🔑 Quick Answer
**Suspect 1: New code introduces a memory leak** — a static collection that grows but never shrinks (e.g., added a local cache `Map<String, Object>` without eviction), or registered listeners/callbacks that hold strong references to large objects. **Suspect 2: Library/dependency change** — new version loads more classes (Metaspace growth), or a library caches aggressively internally. **Suspect 3: Query change** — SQL query returning 10x more rows (removed WHERE clause, added a JOIN) and loading all into memory. **Suspect 4: Classloader leak** — hot-reload tools (Spring DevTools) or dynamic proxy generation creating new classes on each request. **Diagnosis**: compare heap dumps before and after deploy → find the dominating object type that appeared/grew. *(Small change se memory double — leak ya data loading change hoga — heap dump compare karo)*

### 📖 How to Diagnose
```
Step-by-step:

1. COMPARE before vs after:
   $ jcmd <pid> GC.heap_dump before_deploy.hprof   # take before deploy
   ... deploy ...
   $ jcmd <pid> GC.heap_dump after_deploy.hprof    # take after

2. ANALYZE with Eclipse MAT:
   → Open both → "Dominator Tree" → compare top retained objects
   → New: 500MB of byte[] arrays? → large payload/response body buffered
   → New: 200MB of HashMap$Node? → unbounded cache introduced
   → New: 100MB of Class objects? → classloader/metaspace leak

3. CHECK GIT DIFF for red flags:
   → New static Map/List without size limit?
   → New query without LIMIT/pagination?
   → New @Cacheable without maxSize/TTL?
   → New Thread or ThreadLocal without cleanup?
   → Library version bump?

4. METASPACE check:
   $ jcmd <pid> VM.metaspace
   → If metaspace growing linearly → dynamic class generation leak

Common Memory Leak Patterns:
  static Map<K,V> cache = new HashMap<>();   // grows forever ❌
  eventBus.register(listener);                // never unregistered ❌
  threadLocal.set(largeObj);                  // never removed in pool ❌
  entityManager.find(All);                    // loads entire table ❌
  BufferedReader reader = new BufferedReader(...)  // never closed ❌
```

### 🗣️ How to Say in Interview
"When memory doubles after a deploy, I don't guess — I take a heap dump and compare it to a pre-deploy baseline. Using Eclipse MAT's dominator tree, I find the object type consuming the most retained memory that wasn't there before. In my experience, the most common cause is an unbounded cache — someone adds a HashMap as a local cache without eviction, and it grows linearly with traffic. Second most common is a query change that accidentally loads a much larger result set, like removing a WHERE clause. I also check Metaspace — if a library upgrade introduces excessive dynamic class generation, that shows up as growing Metaspace. The git diff of the small change usually reveals the culprit within minutes once you know what object type to look for."

### 💻 Code — Common Leaks → Fixes
```java
// ❌ LEAK: Unbounded static cache
public class UserService {
    private static final Map<Long, User> cache = new HashMap<>();

    public User getUser(Long id) {
        return cache.computeIfAbsent(id, this::loadFromDb); // grows forever!
    }
}

// ✅ FIX: Bounded cache with eviction
public class UserService {
    private final Cache<Long, User> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofMinutes(10))
        .build();

    public User getUser(Long id) {
        return cache.get(id, this::loadFromDb);
    }
}

// ❌ LEAK: ThreadLocal in thread pool (thread reused → value persists)
private static final ThreadLocal<List<AuditEntry>> auditLog = new ThreadLocal<>();
public void handleRequest() {
    auditLog.set(new ArrayList<>());
    processRequest();
    // forgot auditLog.remove()! → list stays for next request on same thread
}

// ✅ FIX: Always clean up ThreadLocal
public void handleRequest() {
    auditLog.set(new ArrayList<>());
    try {
        processRequest();
    } finally {
        auditLog.remove(); // ALWAYS clean up
    }
}
```

### ⚡ Remember
- **Heap dump compare** = fastest way to find leak *(Heap dump before vs after — MAT mein compare karo)*
- Top leak causes: static collections, ThreadLocal, unclosed resources, unbounded caches
- **Git diff the deploy** — small change = small search space
- Check **Metaspace** separately (not in heap dump) for class-loading leaks
- Prevention: use Caffeine/Guava cache with maxSize + TTL, never raw HashMap for caching

### 🔗 Cross-References
- core/01 Q-OOM → Memory leak detection with MAT, heap dump analysis
- core/03 → JVM Performance Tuning (heap sizing, GC log analysis)

---

<a id="q3"></a>
## Q3. An API endpoint suddenly starts consuming huge memory for a simple operation. What might be happening?

### 📝 One-Liner
Most likely **loading an unexpectedly large dataset into memory** — an unbound query result, a massive request/response payload, or an N+1 query creating thousands of objects per request.

### 🔑 Quick Answer
**(1) Unbounded query result**: `SELECT * FROM orders` without LIMIT on a table that grew from 100 to 1M rows → loads all into List → OOM. **(2) Large payload**: endpoint receives or returns a huge JSON body (e.g., nested objects, base64 file in JSON) → Jackson deserializes into massive object graph. **(3) N+1 query loading**: JPA lazy loading triggers N separate queries, each loading an entity → thousands of managed entities in persistence context. **(4) String concatenation in loop**: building a huge String with `+=` in a loop creates intermediate String objects (each immutable copy). **(5) Accidental full collection copy**: `new ArrayList<>(hugeList)` or `stream.collect(toList())` on a massive dataset. **Diagnosis**: take heap dump during the request → find the biggest object → trace its allocation path. *(Simple operation mein memory blast — data zyada load ho raha hai, check karo query aur payload)*

### 📖 How to Diagnose
```
1. REPRODUCE (if possible):
   → Call the endpoint with same parameters
   → Monitor: jconsole or VisualVM → watch heap graph jump

2. HEAP DUMP during request:
   $ jcmd <pid> GC.heap_dump /tmp/during_request.hprof
   → Open in MAT → "Biggest Objects" → what's consuming memory?
   → 500MB of byte[] → large payload or buffered stream
   → 500MB of HashMap$Node / ArrayList → large collection loaded
   → 500MB of char[] / String → string building or logging

3. CHECK QUERY:
   → Enable Hibernate SQL logging (temporarily):
     spring.jpa.show-sql=true → see actual queries
   → Is it SELECT * without LIMIT?
   → Is it N+1 (one query per item)?

4. CHECK PAYLOAD:
   → Log request Content-Length
   → Is response body unexpectedly large?
   
5. JFR (Java Flight Recorder):
   $ jcmd <pid> JFR.start duration=60s filename=recording.jfr
   → Shows Object Allocation in New TLAB → find which method allocates most
```

### 💻 Code — Common Causes → Fixes
```java
// ❌ PROBLEM: Loading entire table
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    List<Order> findAll();  // 2 million orders → OOM!
}

// ✅ FIX: Pagination
Page<Order> findAll(Pageable pageable);
// Usage: repo.findAll(PageRequest.of(0, 100));

// ✅ FIX: Streaming for export/batch
@Query("SELECT o FROM Order o")
Stream<Order> streamAll(); // processes one at a time, not all in memory

// ❌ PROBLEM: String concatenation in loop
String result = "";
for (Order order : orders) {  // 100K orders
    result += order.toString() + "\n";  // creates new String each iteration!
}

// ✅ FIX: StringBuilder
StringBuilder sb = new StringBuilder();
for (Order order : orders) {
    sb.append(order.toString()).append('\n');
}

// ❌ PROBLEM: Returning entire nested object graph in REST response
@GetMapping("/users/{id}")
public User getUser(@PathVariable Long id) {
    return userRepository.findById(id); // EAGER: loads orders, products, reviews...
}

// ✅ FIX: DTO projection — return only what's needed
@GetMapping("/users/{id}")
public UserSummaryDTO getUser(@PathVariable Long id) {
    return userRepository.findUserSummaryById(id);
}
```

### ⚡ Remember
- **Unbounded queries** = #1 cause (table grew, query didn't adapt) *(Table 100 rows se 1M rows ho gayi — query same rahi)*
- Use **pagination** (`Pageable`) or **streaming** (`Stream<T>`) for large datasets
- **N+1 detection**: enable Hibernate statistics or use Hypersistence Optimizer
- Check **Content-Length** of request/response for payload issues
- **JFR** `Object Allocation in New TLAB` → pinpoints the allocating method

---

<a id="q4"></a>
## Q4. After enabling parallel processing, CPU usage spikes but throughput doesn't improve. Why?

### 📝 One-Liner
**Lock contention** (threads fight for same lock → serialized despite parallelism), **false sharing** (CPU cache lines invalidated across cores), or **the work is I/O-bound not CPU-bound** (adding CPU threads to an I/O problem doesn't help).

### 🔑 Quick Answer
**(1) Lock contention**: parallelized code has a shared synchronized section → all threads serialize at the lock → effectively single-threaded with overhead of context switching. CPU busy doing context switches, not useful work. **(2) I/O-bound workload**: the bottleneck is database/network, not CPU. 8 parallel threads all waiting on DB → DB becomes bottleneck. CPU high from thread management overhead, not computation. **(3) False sharing**: threads update adjacent fields in same CPU cache line (64 bytes) → every write invalidates the cache line for all other cores → constant cache-line bouncing. **(4) Excessive context switching**: more threads than CPU cores → OS constantly switches between threads → overhead exceeds benefit. **(5) Parallel stream overhead**: `parallelStream()` on small collections → fork/join overhead > computation savings. *(CPU high but throughput same — lock pe sab thread ruk rahe hain ya kaam I/O-bound hai)*

### 📖 How to Diagnose
```
1. THREAD DUMP analysis:
   $ jstack <pid> > dump.txt
   → Many threads BLOCKED on same monitor?  → Lock contention
   → Many threads WAITING (I/O, socket)?    → I/O-bound bottleneck
   → All threads RUNNABLE?                  → False sharing or CPU overhead

2. LOCK CONTENTION detection:
   JFR → Java Monitor Blocked events → which lock? which code?
   $ jcmd <pid> JFR.start duration=30s filename=contention.jfr
   → JMC → "Lock Instances" → top contended locks

3. CONTEXT SWITCH count:
   Linux: $ vmstat 1 → watch 'cs' column (context switches/sec)
   High cs (>50K/s) with low 'us' (user CPU%) → too many threads fighting

4. AMDAHL'S LAW check:
   Speedup = 1 / (S + (1 - S) / N)
   S = serial fraction, N = number of cores
   
   If 50% of code is synchronized (S=0.5):
   With 8 cores: Speedup = 1 / (0.5 + 0.5/8) = 1.78x
   → 8 cores gives only 1.78x improvement! Serial section limits scaling.

   If 5% serial (S=0.05):
   With 8 cores: Speedup = 1 / (0.05 + 0.95/8) = 5.9x ← much better!
```

### 💻 Code — Common Causes → Fixes
```java
// ❌ PROBLEM: Lock contention — synchronized kills parallelism
public class MetricsCollector {
    private final Map<String, Long> counts = new HashMap<>();

    public synchronized void record(String metric) {  // ALL threads serialize here
        counts.merge(metric, 1L, Long::sum);
    }
}

// ✅ FIX: ConcurrentHashMap + atomic operations (no lock)
public class MetricsCollector {
    private final ConcurrentHashMap<String, LongAdder> counts = new ConcurrentHashMap<>();

    public void record(String metric) {  // lock-free!
        counts.computeIfAbsent(metric, k -> new LongAdder()).increment();
    }
}

// ❌ PROBLEM: parallelStream() on I/O-bound work
orders.parallelStream()
    .map(order -> restTemplate.getForObject(  // each call waits 200ms for network!
        "http://payment-service/payments/" + order.getId(), Payment.class))
    .collect(toList());
// → 8 threads all waiting on HTTP → DB/service is the bottleneck, not CPU

// ✅ FIX: Use async I/O (CompletableFuture + dedicated I/O pool)
ExecutorService ioPool = Executors.newFixedThreadPool(30); // I/O pool, more threads OK
List<CompletableFuture<Payment>> futures = orders.stream()
    .map(order -> CompletableFuture.supplyAsync(
        () -> restTemplate.getForObject(...), ioPool))
    .toList();
List<Payment> payments = futures.stream()
    .map(CompletableFuture::join)
    .toList();
```

### ⚡ Remember
- **Amdahl's Law** — parallel speedup limited by the serial (locked) fraction
- **Lock contention** → `ConcurrentHashMap`, `LongAdder`, lock-free algorithms
- **I/O-bound** → more threads OR async I/O, not parallelStream *(I/O-bound kaam pe CPU threads badhana = bekar — async I/O karo)*
- **parallelStream** = only for CPU-bound, large collections (>10K elements)
- **Thread count** = CPU cores for compute; more for I/O-bound

### 🔗 Cross-References
- multithreading/02 → Lock contention reduction strategies
- multithreading/04 → Thread pool sizing, ExecutorService
- multithreading/05 → ConcurrentHashMap, LongAdder

---

<a id="q5"></a>
## Q5. A small bug causes a recursive method to run indefinitely until the application crashes. What error does this lead to?

### 📝 One-Liner
**`StackOverflowError`** — each recursive call adds a stack frame (~100-200 bytes), and with default stack size (~1MB per thread via `-Xss`), roughly 5,000-10,000 frames fills the stack.

### 🔑 Quick Answer
**Error**: `java.lang.StackOverflowError` (subclass of `Error`, not `Exception`). Each method call pushes a frame onto the thread's stack (local variables, operand stack, return address). Stack size is fixed per thread (default ~512KB-1MB, configured via `-Xss`). If recursion has no base case or wrong base case → infinite calls → stack frames accumulate → stack overflow. **Not an OOM** — stack is separate from heap. **Recovery**: the thread that overflows dies, but other threads continue. However, if it's a Tomcat request thread, the request fails with 500. *(Recursive method bina base case = StackOverflowError — stack bhar gaya kyunki har call ek frame daalta hai)*

### 📖 How to Diagnose
```
Stack trace shows:
  java.lang.StackOverflowError
      at com.example.OrderService.calculateTotal(OrderService.java:42)
      at com.example.OrderService.calculateTotal(OrderService.java:42)
      at com.example.OrderService.calculateTotal(OrderService.java:42)
      ... (same line repeating = infinite recursion)

OR more subtle mutual recursion:
      at com.example.A.process(A.java:10)
      at com.example.B.handle(B.java:20)
      at com.example.A.process(A.java:10)
      at com.example.B.handle(B.java:20)
      ... (two methods calling each other)

vs. LEGITIMATE deep recursion (tree traversal):
      at com.example.TreeWalker.visit(TreeWalker.java:15)   // depth 8000
      at com.example.TreeWalker.visit(TreeWalker.java:15)
      → Fix: increase -Xss or convert to iterative with explicit stack

Diagnosis steps:
  1. Read the stack trace — repeating pattern of 1-2 methods = bug
  2. Find the recursive call — check the base case
  3. Fix the base case or convert to iterative
```

### 💻 Code — Bug → Fix
```java
// ❌ BUG: Missing base case
public int factorial(int n) {
    return n * factorial(n - 1); // what about n == 0? → infinite recursion
}

// ✅ FIX: Proper base case
public int factorial(int n) {
    if (n <= 1) return 1;  // base case stops recursion
    return n * factorial(n - 1);
}

// ❌ BUG: Subtle — wrong base case for graph traversal (cycles!)
public void traverse(Node node) {
    System.out.println(node.value);
    for (Node child : node.children) {
        traverse(child);  // if graph has cycles → infinite loop
    }
}

// ✅ FIX: Track visited nodes
public void traverse(Node node, Set<Node> visited) {
    if (!visited.add(node)) return;  // already visited → stop
    System.out.println(node.value);
    for (Node child : node.children) {
        traverse(child, visited);
    }
}

// ✅ FIX: Convert deep recursion to iterative (for legitimate deep structures)
public void traverseIterative(Node root) {
    Deque<Node> stack = new ArrayDeque<>();
    Set<Node> visited = new HashSet<>();
    stack.push(root);
    while (!stack.isEmpty()) {
        Node node = stack.pop();
        if (visited.add(node)) {
            System.out.println(node.value);
            node.children.forEach(stack::push);
        }
    }
}
```

### ⚡ Remember
- **StackOverflowError** = stack frames exceed `-Xss` limit (not heap related)
- Stack trace shows **repeating line pattern** → find the recursive call
- Fix: **base case**, **visited set** for graphs, or **convert to iterative**
- `-Xss2m` increases stack size (for legitimate deep structures, not for infinite bugs)
- Mutual recursion (A→B→A→B) is harder to spot than self-recursion

### 🔗 Cross-References
- core/01 → JVM Memory Areas (Stack vs Heap, StackOverflowError)
