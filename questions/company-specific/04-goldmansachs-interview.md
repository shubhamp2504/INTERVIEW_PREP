# 🏢 Goldman Sachs — Java Backend Developer Interview (5 Rounds, 38 LPA)

> Direct application for Java Backend Developer role. 5 rounds: OA, DSA + Java, Spring Boot + LLD, Security + DB Optimization, HR. Pattern: Goldman doesn't test topics — they test thinking depth.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Round 1: Online Assessment

---

<a id="q1"></a>
## Q1. Validate a Custom Expression — `{`, `}`, and `*` where `*` can be `{`, `}`, or empty

### 📝 One-Liner
Given a string with `{`, `}`, and `*` (wildcard that can be `{`, `}`, or empty), determine if the expression can be balanced.

### 🔑 Quick Answer
Use two counters tracking the **range** of possible open bracket counts: `low` (minimum possible open) and `high` (maximum possible open). For `{`: both increment. For `}`: both decrement. For `*`: low decrements (treat as `}`), high increments (treat as `{`). If `high < 0`, invalid. Keep `low ≥ 0`. Valid if `low == 0` at end. *(Do counters rakho — low aur high — jo possible open brackets ka range track karte hain)*

### 📖 How It Works
```
String: "{ * }"
Step 1: '{' → low=1, high=1   (definitely 1 open)
Step 2: '*' → low=0, high=2   (* can be }, empty, or {)
Step 3: '}' → low=0, high=1   (close one)
End: low=0 ✅ → VALID

String: "} {"
Step 1: '}' → low=-1→0, high=0  (high < 0 before correction? no, high=-1 → INVALID)
```

### 💻 Code
```java
public boolean isValid(String s) {
    int low = 0, high = 0;
    for (char c : s.toCharArray()) {
        if (c == '{') {
            low++;
            high++;
        } else if (c == '}') {
            low--;
            high--;
        } else { // '*'
            low--;   // treat as '}'
            high++;  // treat as '{'
        }
        if (high < 0) return false;  // too many closing brackets
        low = Math.max(low, 0);      // can't have negative open count
    }
    return low == 0;
}

// Test cases
isValid("{*}")     // true  (* = empty)
isValid("{{**}")   // true  (* = } and empty)
isValid("}*{")     // false (starts with closing)
isValid("***")     // true  (all empty)
```

### ⚡ Remember
- O(n) time, O(1) space — greedy range approach
- `low` = minimum possible open brackets, `high` = maximum possible
- Clamp `low` to 0 (can't be negative — just means some `*` were treated as `}`)
- Similar to LeetCode #678 (Valid Parenthesis String) but with `{}`

---

<a id="q2"></a>
## Q2. K Most Frequent Words — sorted by frequency and lexicographical order

### 📝 One-Liner
Given a list of words, return the top-k most frequent words sorted by frequency (descending) and lexicographically (ascending) for ties.

### 🔑 Quick Answer
Count frequencies with HashMap, then use a PriorityQueue (min-heap of size k) with custom comparator: compare by frequency ascending, then by lexicographic descending (reverse for min-heap). Or sort the frequency map entries directly. *(Pehle frequency count karo, phir PriorityQueue ya sorting se top-k nikalo)*

### 💻 Code
```java
public List<String> topKFrequent(String[] words, int k) {
    // Step 1: Count frequencies
    Map<String, Integer> freq = new HashMap<>();
    for (String word : words) {
        freq.merge(word, 1, Integer::sum);
    }

    // Step 2: Min-heap of size k (reverse comparator for min-heap)
    PriorityQueue<String> pq = new PriorityQueue<>((a, b) -> {
        int freqCompare = freq.get(a) - freq.get(b); // ascending freq
        if (freqCompare != 0) return freqCompare;
        return b.compareTo(a); // descending lexicographic (reverse for min-heap)
    });

    for (String word : freq.keySet()) {
        pq.offer(word);
        if (pq.size() > k) pq.poll(); // remove least frequent
    }

    // Step 3: Build result (reverse because min-heap gives smallest first)
    LinkedList<String> result = new LinkedList<>();
    while (!pq.isEmpty()) {
        result.addFirst(pq.poll());
    }
    return result;
}

// Alternative: Sort approach (simpler, O(n log n))
public List<String> topKFrequentSort(String[] words, int k) {
    Map<String, Integer> freq = new HashMap<>();
    for (String w : words) freq.merge(w, 1, Integer::sum);

    return freq.entrySet().stream()
        .sorted((a, b) -> {
            int fc = b.getValue() - a.getValue(); // desc frequency
            return fc != 0 ? fc : a.getKey().compareTo(b.getKey()); // asc lexicographic
        })
        .limit(k)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}

// Test
String[] words = {"the", "day", "is", "sunny", "the", "the", "sunny", "is", "is"};
topKFrequent(words, 4); // ["the", "is", "sunny", "day"]
```

### ⚡ Remember
- Heap approach: O(n log k) — better when k << n
- Sort approach: O(n log n) — simpler code
- Tie-breaking: same frequency → lexicographically smaller word comes first
- LeetCode #692 — Top K Frequent Words

---

## Round 2: DSA + Java Concepts

---

<a id="q3"></a>
## Q3. Reverse a Linked List in Pairs — Input: 1→2→3→4→5, Output: 2→1→4→3→5

### 📝 One-Liner
Swap adjacent pairs of nodes in a linked list — every two consecutive nodes swap positions; an odd last node stays in place.

### 🔑 Quick Answer
Iterative: use a dummy head, process two nodes at a time. For each pair (first, second): rewire `prev → second → first → next_pair`. Recursive: swap first two, recursively process rest. *(Pairs mein swap karo — do nodes lelo, unhe ulta karo, aage badho)*

### 💻 Code
```java
// Definition
class ListNode {
    int val;
    ListNode next;
    ListNode(int val) { this.val = val; }
}

// Iterative — O(n) time, O(1) space
public ListNode swapPairs(ListNode head) {
    ListNode dummy = new ListNode(0);
    dummy.next = head;
    ListNode prev = dummy;

    while (prev.next != null && prev.next.next != null) {
        ListNode first = prev.next;       // node 1
        ListNode second = prev.next.next; // node 2

        // Rewire: prev → second → first → rest
        first.next = second.next;
        second.next = first;
        prev.next = second;

        prev = first; // move to next pair
    }
    return dummy.next;
}

// Recursive — elegant but O(n) stack space
public ListNode swapPairsRecursive(ListNode head) {
    if (head == null || head.next == null) return head;
    ListNode second = head.next;
    head.next = swapPairsRecursive(second.next); // recurse on rest
    second.next = head;                           // swap
    return second;                                // new head of this pair
}

// 1 → 2 → 3 → 4 → 5
// becomes: 2 → 1 → 4 → 3 → 5
```

### ⚡ Remember
- Dummy node simplifies edge cases (head swap)
- Iterative: O(n) time, O(1) space — preferred for interviews
- Odd-length list: last node stays as-is
- LeetCode #24 — Swap Nodes in Pairs

---

<a id="q4"></a>
## Q4. == vs equals() in Java — What's the difference?

### 📝 One-Liner
`==` compares **reference identity** (same memory address); `equals()` compares **logical equality** (content/value) — must be overridden for custom objects.

### 🔑 Quick Answer
For primitives, `==` compares values. For objects, `==` checks if both references point to the same object in heap. `equals()` (from Object class) defaults to `==` but should be overridden to compare field values. String pool complicates this: `"hello" == "hello"` is true (same pool reference), `new String("hello") == new String("hello")` is false. *(== memory address compare karta hai, equals() content compare karta hai — objects ke liye hamesha equals() use karo)*

### 💻 Code
```java
// String comparison
String s1 = "hello";
String s2 = "hello";
String s3 = new String("hello");

s1 == s2;      // true  (same string pool reference)
s1 == s3;      // false (different objects — pool vs heap)
s1.equals(s3); // true  (same content)

// Integer caching (-128 to 127)
Integer a = 127;
Integer b = 127;
a == b;        // true  (cached)

Integer c = 128;
Integer d = 128;
c == d;        // false (beyond cache range, different objects)
c.equals(d);   // true  (same value)

// Custom class — must override equals + hashCode
public class Employee {
    private Long id;
    private String name;

    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (o == null || getClass() != o.getClass()) return false;
        Employee emp = (Employee) o;
        return Objects.equals(id, emp.id);
    }

    @Override
    public int hashCode() { return Objects.hash(id); }
}
```

### ⚠️ Pitfalls
- Forgetting to override `hashCode()` when overriding `equals()` breaks HashMap/HashSet
- `null.equals(x)` → NullPointerException; use `Objects.equals(a, b)` for null-safe comparison
- String: prefer `"literal".equals(variable)` to avoid NPE

### ⚡ Remember
- `==` → identity (same object?), `equals()` → equivalence (same value?)
- Contract: `a.equals(b)` ⟹ `a.hashCode() == b.hashCode()` (must hold)
- Integer cache: -128 to 127 → `==` works; beyond → use `equals()`

---

<a id="q5"></a>
## Q5. How does HashMap work internally? Explain collision handling in Java 8+

### 📝 One-Liner
HashMap uses hash-based array of buckets; collisions are handled by chaining — LinkedList for ≤7 nodes, Red-Black Tree for ≥8 nodes (Java 8+ optimization).

### 🔑 Quick Answer
`put(key, value)`: compute `hash(key)` → bucket index `(n-1) & hash` → if bucket empty, insert Node. If collision: traverse chain, compare keys via `equals()`. Java 8 change: when chain length ≥ 8 (and table ≥ 64), convert LinkedList to Red-Black Tree for O(log n) lookup instead of O(n). Load factor 0.75 = good balance between space and time — at 75% capacity, resize (double array). *(Java 8 mein collision handling improve hua — long chains tree ban jaate hain, O(n) se O(log n))*

### 📖 How It Works
```
put("A", 1):
1. hash = key.hashCode() ^ (hashCode >>> 16)    // spread high bits
2. index = (table.length - 1) & hash              // bucket index
3. If bucket empty → insert new Node
4. If bucket occupied → collision:
   a. Compare hash, then equals() for each node in chain
   b. If key exists → update value
   c. If key doesn't exist → append to chain
   d. If chain length ≥ 8 AND table ≥ 64 → treeify (Red-Black Tree)
   e. If chain length ≤ 6 after removal → untreeify back to LinkedList

Load Factor = 0.75:
- Why 0.75? Statistical: with good hash, avg bucket occupancy ~0.5 at 75% load
- Lower (0.5) = more space waste, fewer collisions
- Higher (0.9) = more collisions, better space utilization
- 0.75 is the sweet spot (Poisson distribution analysis)
```

### 🎯 Tricky Qs
- **Why 0.75 load factor?** → Based on Poisson distribution — at 0.75, probability of 8+ nodes in a bucket is ~0.00000006
- **Why power of 2 for capacity?** → `(n-1) & hash` works as fast modulo only for power-of-2 sizes
- **Why treeify at 8?** → Poisson distribution shows 8+ in one bucket is extremely unlikely with good hash — if it happens, security attack likely
- **What if key is null?** → Always goes to bucket 0 (hash = 0)

### ⚡ Remember
- Default: capacity=16, load factor=0.75, resize threshold=12
- Java 8: LinkedList → Tree at 8 nodes, Tree → LinkedList at 6 nodes
- Thread-unsafe: use `ConcurrentHashMap` for multi-threaded access
- Keys should be immutable (String, Integer) — mutable keys break HashMap

---

<a id="q6"></a>
## Q6. Explain the Java Memory Model (JMM)

### 📝 One-Liner
JMM defines how threads interact through memory — it specifies visibility, ordering, and atomicity guarantees for shared variables across threads.

### 🔑 Quick Answer
JMM defines **happens-before** relationships that guarantee when one thread's write becomes visible to another thread's read. Without proper synchronization, threads may see stale values due to CPU caches and instruction reordering. Key mechanisms: `volatile` (visibility), `synchronized` (mutual exclusion + visibility), `final` (safe publication). *(JMM batata hai ki ek thread ka change doosre thread ko kab dikhega — bina synchronization ke stale data dikh sakta hai)*

### 📖 How It Works
```
Thread 1 (CPU 1)          Main Memory          Thread 2 (CPU 2)
┌──────────────┐         ┌──────────┐         ┌──────────────┐
│ Local Cache   │         │ x = 0    │         │ Local Cache   │
│ x = 42 (new) │ ──?──►  │ x = ?    │  ──?──► │ x = 0 (stale)│
└──────────────┘         └──────────┘         └──────────────┘

Without volatile/synchronized:
- Thread 1 writes x=42 to its CPU cache
- Thread 2 may NEVER see x=42 (reads from its own cache)

With volatile:
- Write to volatile forces flush to main memory
- Read of volatile forces refresh from main memory
```

**Happens-Before Rules:**
1. **Program order**: within a thread, earlier actions happen-before later actions
2. **Monitor lock**: unlock happens-before subsequent lock on same monitor
3. **Volatile**: write to volatile happens-before subsequent read of same volatile
4. **Thread start**: `thread.start()` happens-before any action in that thread
5. **Thread join**: all actions in thread happen-before `join()` returns
6. **Transitivity**: if A happens-before B, and B happens-before C, then A happens-before C

### 💻 Code
```java
// Problem: without volatile, loop may never end
class VisibilityProblem {
    private boolean running = true; // should be volatile

    public void stop() { running = false; } // Thread 1

    public void run() {
        while (running) { /* busy loop */ } // Thread 2 may NEVER see false
    }
}

// Fix: volatile guarantees visibility
class VisibilityFixed {
    private volatile boolean running = true;
    // Now Thread 2 always reads latest value from main memory
}

// synchronized provides both mutual exclusion AND visibility
synchronized (lock) {
    // everything before this block is visible to next thread entering this block
    sharedData = newValue;
}
```

### ⚡ Remember
- **Visibility**: `volatile`, `synchronized`, `final`, `AtomicXxx`
- **Atomicity**: `synchronized`, `AtomicXxx`, `Lock`
- **Ordering**: `volatile` prevents reordering across the volatile read/write
- JMM ≠ JVM memory structure (heap/stack) — JMM is about concurrency guarantees
- Double-checked locking needs `volatile` precisely because of JMM reordering

---

<a id="q7"></a>
## Q7. Explain thread safety and common concurrency patterns in Java

### 📝 One-Liner
Thread safety means a class behaves correctly when accessed from multiple threads without external synchronization — achieved via immutability, synchronization, atomic operations, or thread-local confinement.

### 🔑 Quick Answer
Four strategies: **(1) Immutability** — no shared mutable state. **(2) Synchronization** — `synchronized`/`Lock` for mutual exclusion. **(3) Atomic classes** — `AtomicInteger`, CAS-based lock-free ops. **(4) Thread confinement** — `ThreadLocal`, stack variables. Concurrent collections: `ConcurrentHashMap`, `CopyOnWriteArrayList`, `BlockingQueue`. *(Thread safety = multiple threads se access hone pe bhi correct behavior — immutable banao ya proper sync karo)*

### 💻 Code
```java
// 1. Immutability (safest)
public final class ImmutableConfig {
    private final String host;
    private final int port;
    public ImmutableConfig(String host, int port) {
        this.host = host; this.port = port;
    }
    public String getHost() { return host; }
    // No setters — inherently thread-safe
}

// 2. Synchronized
public class SafeCounter {
    private int count = 0;
    public synchronized void increment() { count++; }
    public synchronized int getCount() { return count; }
}

// 3. Atomic (lock-free, better throughput)
public class AtomicCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); } // CAS operation
    public int getCount() { return count.get(); }
}

// 4. ThreadLocal (each thread has own copy)
private static final ThreadLocal<SimpleDateFormat> dateFormat =
    ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));

// 5. Concurrent collections
Map<String, Integer> safeMap = new ConcurrentHashMap<>();
List<String> safeList = new CopyOnWriteArrayList<>();
BlockingQueue<Task> taskQueue = new LinkedBlockingQueue<>();
```

### 🆚 vs.
| Approach | Throughput | Complexity | Use Case |
|----------|-----------|------------|----------|
| Immutable | Highest | Low | Config, value objects |
| synchronized | Low (contention) | Low | Simple shared state |
| ReentrantLock | Medium | Medium | Need tryLock, conditions |
| AtomicXxx | High | Low | Counters, flags |
| ConcurrentHashMap | High | Low | Concurrent map access |

### ⚡ Remember
- **Rule of thumb**: prefer immutability > atomic > lock
- `ConcurrentHashMap`: segment-level locking (Java 7) → node-level CAS (Java 8)
- `volatile` alone doesn't make `count++` atomic — use `AtomicInteger`
- `Collections.synchronizedMap()` = single lock for all operations (poor throughput)

---

## Round 3: Spring Boot + Low-Level Design

---

<a id="q8"></a>
## Q8. Design a Rate Limiter — Max 5 requests per user per minute

### 📝 One-Liner
Rate limiter controls request frequency per user — implement using sliding window counter with a `ConcurrentHashMap` of timestamp queues per user.

### 🔑 Quick Answer
Store a queue of timestamps per user. On each request: remove expired timestamps (older than 1 minute), check if remaining count < 5. If yes, allow and add timestamp. If no, reject with 429 Too Many Requests. Use Sliding Window Log for accuracy or Token Bucket for smoother rate control. *(Har user ke liye timestamps ka queue rakho — 1 minute se purane hatao, 5 se kam hain toh allow karo)*

### 📖 How It Works
```
Algorithms:
1. Fixed Window Counter    → Simple but burst at window boundary
2. Sliding Window Log      → Accurate, stores all timestamps (memory heavy)
3. Sliding Window Counter  → Hybrid (weighted previous + current window)
4. Token Bucket            → Smooth, allows bursts up to bucket size
5. Leaky Bucket            → Fixed output rate, drops excess

For "5 req/user/min" — Sliding Window Log:
User "alice" requests:
  t=0:01  → queue: [0:01]           → count=1 ✅
  t=0:15  → queue: [0:01, 0:15]     → count=2 ✅
  t=0:30  → queue: [..., 0:30]      → count=3 ✅
  t=0:45  → queue: [..., 0:45]      → count=4 ✅
  t=0:50  → queue: [..., 0:50]      → count=5 ✅
  t=0:55  → queue: [..., 0:55]      → count=6 ❌ REJECTED (429)
  t=1:02  → remove 0:01 (expired)   → count=5 ✅
```

### 💻 Code
```java
public class SlidingWindowRateLimiter {
    private final int maxRequests;
    private final long windowMillis;
    private final ConcurrentHashMap<String, ConcurrentLinkedDeque<Long>> userRequests
        = new ConcurrentHashMap<>();

    public SlidingWindowRateLimiter(int maxRequests, long windowMillis) {
        this.maxRequests = maxRequests;
        this.windowMillis = windowMillis;
    }

    public boolean allowRequest(String userId) {
        long now = System.currentTimeMillis();
        ConcurrentLinkedDeque<Long> timestamps = userRequests
            .computeIfAbsent(userId, k -> new ConcurrentLinkedDeque<>());

        // Remove expired entries
        while (!timestamps.isEmpty() && now - timestamps.peekFirst() > windowMillis) {
            timestamps.pollFirst();
        }

        if (timestamps.size() < maxRequests) {
            timestamps.addLast(now);
            return true;
        }
        return false; // rate limited
    }
}

// Usage
SlidingWindowRateLimiter limiter = new SlidingWindowRateLimiter(5, 60_000); // 5 req/min
if (!limiter.allowRequest(userId)) {
    return ResponseEntity.status(429).body("Too many requests");
}

// Token Bucket (smoother, allows bursts)
public class TokenBucket {
    private final int capacity;
    private final double refillRate; // tokens per millisecond
    private double tokens;
    private long lastRefill;

    public TokenBucket(int capacity, int tokensPerMinute) {
        this.capacity = capacity;
        this.tokens = capacity;
        this.refillRate = tokensPerMinute / 60_000.0;
        this.lastRefill = System.currentTimeMillis();
    }

    public synchronized boolean tryConsume() {
        refill();
        if (tokens >= 1) { tokens -= 1; return true; }
        return false;
    }

    private void refill() {
        long now = System.currentTimeMillis();
        tokens = Math.min(capacity, tokens + (now - lastRefill) * refillRate);
        lastRefill = now;
    }
}
```

### ⚡ Remember
- Distributed rate limiting: use Redis (INCR + EXPIRE) or dedicated service
- Sliding Window Log: most accurate but memory-intensive
- Token Bucket: industry standard (used by API gateways, AWS, Stripe)
- Always return **429 Too Many Requests** with `Retry-After` header

---

<a id="q9"></a>
## Q9. @Component vs @Service vs @Repository — What's the difference?

### 📝 One-Liner
All three are Spring stereotype annotations that register beans; `@Service` adds semantic meaning for business logic, `@Repository` enables exception translation for persistence layer.

### 🔑 Quick Answer
`@Component` = generic Spring-managed bean. `@Service` = business layer (no extra behavior, just semantic). `@Repository` = persistence layer — enables automatic **PersistenceExceptionTranslation** (converts JDBC/JPA exceptions to Spring's `DataAccessException` hierarchy). All are detected by `@ComponentScan`. *(Technically sab @Component hain — @Service sirf documentation ke liye hai, @Repository exception translation add karta hai)*

### 📖 How It Works
```
@Component (base) ← @Service (semantic only) ← @Repository (+ exception translation)

@Component
├── @Service     → Business logic layer (no extra behavior)
├── @Repository  → Persistence layer (+ PersistenceExceptionTranslation)
├── @Controller  → Web layer (+ request dispatching)
└── @Configuration → Config class (+ CGLIB proxy for @Bean methods)
```

### 💻 Code
```java
@Repository // translates SQLException → DataAccessException
public class UserRepositoryImpl implements UserRepository {
    @PersistenceContext
    private EntityManager em;
    // JPA exceptions automatically translated
}

@Service // semantic — marks business logic
public class UserService {
    private final UserRepository userRepository;
    public UserService(UserRepository repo) { this.userRepository = repo; }
}

@Component // generic bean — utility, helpers
public class EmailValidator {
    public boolean isValid(String email) { /*...*/ }
}
```

### 🆚 vs.
| Annotation | Layer | Extra Behavior | Use Case |
|-----------|-------|---------------|----------|
| @Component | Any | None | Generic utility bean |
| @Service | Business | None (semantic only) | Service classes |
| @Repository | Persistence | Exception translation | DAO classes |
| @Controller | Web | Request handling | MVC controllers |

### ⚡ Remember
- `@Service` has **zero extra behavior** over `@Component` — it's purely for readability
- `@Repository` adds `PersistenceExceptionTranslationPostProcessor`
- All are meta-annotated with `@Component` → detected by component scan
- Use the right annotation for the right layer — it improves code clarity and AOP targeting

---

<a id="q10"></a>
## Q10. Explain Spring Bean Scopes and Bean Lifecycle

### 📝 One-Liner
Spring beans have scopes (singleton, prototype, request, session, application) and lifecycle: instantiation → populate properties → BeanNameAware → BeanFactoryAware → pre-init → @PostConstruct → afterPropertiesSet → custom init → post-init → ready → @PreDestroy → destroy.

### 🔑 Quick Answer
**Singleton** (default): one instance per Spring container. **Prototype**: new instance per injection/request. **Request/Session/Application**: web-scoped. Lifecycle hooks: `@PostConstruct`/`@PreDestroy` (preferred), `InitializingBean`/`DisposableBean`, or `@Bean(initMethod, destroyMethod)`. *(Singleton default hai — poore application mein ek hi instance; Prototype har baar naya object deta hai)*

### 📖 How It Works
```
Bean Lifecycle:
1. Instantiation (constructor)
2. Populate properties (DI)
3. BeanNameAware.setBeanName()
4. BeanFactoryAware.setBeanFactory()
5. ApplicationContextAware.setApplicationContext()
6. BeanPostProcessor.postProcessBeforeInitialization()
7. @PostConstruct
8. InitializingBean.afterPropertiesSet()
9. Custom init-method
10. BeanPostProcessor.postProcessAfterInitialization()
11. ═══ BEAN READY ═══
12. @PreDestroy
13. DisposableBean.destroy()
14. Custom destroy-method
```

### 💻 Code
```java
@Component
@Scope("singleton") // default — one instance per container
public class SingletonService {

    @PostConstruct
    public void init() { System.out.println("Initialized!"); }

    @PreDestroy
    public void cleanup() { System.out.println("Destroying!"); }
}

@Component
@Scope("prototype") // new instance every time
public class PrototypeService { }

// Injecting prototype into singleton — use Provider or ObjectFactory
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<PrototypeService> protoProvider;

    public void process() {
        PrototypeService fresh = protoProvider.getObject(); // new instance each time
    }
}
```

### 🆚 vs.
| Scope | Instances | Destruction | Use Case |
|-------|-----------|-------------|----------|
| singleton | 1 per container | Container shutdown | Stateless services |
| prototype | New per request | NOT managed by Spring | Stateful objects |
| request | 1 per HTTP request | End of request | Request-scoped data |
| session | 1 per HTTP session | Session timeout | User session data |
| application | 1 per ServletContext | App shutdown | Shared app state |

### ⚡ Remember
- Singleton + Prototype injection trap: singleton holds stale prototype reference → use `@Lookup` or `ObjectProvider`
- `@PreDestroy` NOT called for prototype beans (Spring doesn't manage their destruction)
- Custom scope: implement `Scope` interface (e.g., tenant-scoped in multi-tenant apps)

---

<a id="q11"></a>
## Q11. Explain the Spring Boot startup process

### 📝 One-Liner
`SpringApplication.run()` → create ApplicationContext → load auto-configuration → component scan → create beans → start embedded server → ready.

### 🔑 Quick Answer
Startup flow: `main()` → `SpringApplication.run()` → determine app type (servlet/reactive/none) → create `ApplicationContext` → load `spring.factories` for auto-configuration → `@ComponentScan` finds beans → dependency injection → `@PostConstruct` hooks → start embedded server (Tomcat/Jetty) → publish `ApplicationReadyEvent`. *(Spring Boot start hone pe auto-configuration + component scan + embedded server — sab automatic hota hai)*

### 📖 How It Works
```
1. main() → SpringApplication.run(App.class, args)
2. Create SpringApplication instance
   ├── Detect web app type (SERVLET / REACTIVE / NONE)
   ├── Load ApplicationContextInitializers (spring.factories)
   └── Load ApplicationListeners
3. Run:
   ├── Prepare Environment (properties, profiles)
   ├── Create ApplicationContext (AnnotationConfigServletWebServerApplicationContext)
   ├── Load bean definitions:
   │   ├── @SpringBootApplication = @Configuration + @EnableAutoConfiguration + @ComponentScan
   │   ├── @EnableAutoConfiguration → META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
   │   ├── Auto-config classes: DataSourceAutoConfiguration, WebMvcAutoConfiguration, etc.
   │   └── @ComponentScan → find @Component, @Service, @Repository, @Controller
   ├── Refresh context (create all singleton beans, resolve dependencies)
   ├── Call BeanPostProcessors
   ├── @PostConstruct methods
   ├── ApplicationRunner / CommandLineRunner
   └── Start embedded server → application ready!
4. Publish events:
   ApplicationStartingEvent → EnvironmentPrepared → ContextInitialized
   → Prepared → Started → Ready
```

### 🎯 Tricky Qs
- **What does @SpringBootApplication do?** → Meta-annotation: `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- **How does auto-configuration work?** → Loads config from `META-INF/spring/...AutoConfiguration.imports`, applies `@Conditional` annotations
- **What if two auto-configs conflict?** → `@ConditionalOnMissingBean`, `@AutoConfigureBefore`/`@After` for ordering

### ⚡ Remember
- `@SpringBootApplication` = three annotations in one
- Auto-configuration uses `@ConditionalOnClass`, `@ConditionalOnMissingBean`, `@ConditionalOnProperty`
- Startup events: Starting → EnvironmentPrepared → ContextInitialized → Started → Ready
- Customize: implement `ApplicationRunner` or `CommandLineRunner` for post-startup logic

---

<a id="q12"></a>
## Q12. Explain the internals of @Transactional and its common pitfalls

### 📝 One-Liner
`@Transactional` uses AOP proxy to wrap method in a transaction — begin TX before method, commit on success, rollback on RuntimeException; pitfalls include self-invocation and checked exceptions.

### 🔑 Quick Answer
Spring creates a **proxy** (JDK dynamic or CGLIB) around `@Transactional` beans. Proxy intercepts method calls → starts transaction → delegates to real method → commits or rollbacks. **Critical pitfalls**: self-invocation bypasses proxy, checked exceptions don't trigger rollback by default, proxy only intercepts external calls. *(Spring AOP proxy banata hai — method ke aage transaction start karta hai, successfully complete hone pe commit, exception pe rollback)*

### 📖 How It Works
```
External Call:
caller → [Proxy] → begin TX → [Real Bean method] → commit TX → return
                                     │ exception
                                     └──→ rollback TX

Self-Invocation (BROKEN!):
@Service
class OrderService {
    public void process() {
        this.save(); // ❌ calls real method, NOT proxy → no transaction!
    }
    @Transactional
    public void save() { ... }
}
```

### 💻 Code
```java
@Service
public class PaymentService {

    @Transactional // default: propagation=REQUIRED, rollbackFor=RuntimeException
    public void processPayment(Order order) {
        debitAccount(order.getAmount());
        updateOrderStatus(order, "PAID");
        // If RuntimeException → auto rollback both operations
    }

    @Transactional(
        propagation = Propagation.REQUIRES_NEW,  // always new TX
        isolation = Isolation.READ_COMMITTED,
        timeout = 30,
        rollbackFor = {BusinessException.class},  // rollback for checked exception
        readOnly = false
    )
    public void criticalOperation() { /* ... */ }
}

// Fix self-invocation: inject self or use ApplicationContext
@Service
public class OrderService {
    @Autowired private OrderService self; // circular ref resolved by proxy

    public void process() {
        self.save(); // ✅ goes through proxy
    }

    @Transactional
    public void save() { /* ... */ }
}
```

### ⚠️ Pitfalls
| Pitfall | Problem | Fix |
|---------|---------|-----|
| Self-invocation | `this.method()` bypasses proxy | Inject self or use `AopContext` |
| Checked exception | No rollback for `Exception` | `rollbackFor = Exception.class` |
| Private method | Proxy can't intercept | Make method public |
| Final class/method | CGLIB can't subclass | Remove final |
| Read-only on write | Silently ignores writes in some DBs | Remove `readOnly` |
| Wrong propagation | REQUIRES_NEW suspends outer TX | Understand propagation types |

### ⚡ Remember
- Default: rollback only for **unchecked exceptions** (RuntimeException/Error)
- Propagation: REQUIRED (default, join existing), REQUIRES_NEW (always new), NESTED
- Proxy type: JDK (interface-based) or CGLIB (class-based, default in Spring Boot)
- @Transactional on class = applies to all public methods in that class

---

## Round 4: Security + Database Optimization

---

<a id="q13"></a>
## Q13. SQL Query Optimization — Real-world techniques

### 📝 One-Liner
Optimize SQL by adding indexes on WHERE/JOIN columns, avoiding SELECT *, using EXPLAIN to analyze execution plans, and considering denormalization for read-heavy workloads.

### 🔑 Quick Answer
Key techniques: **(1)** Index WHERE, JOIN, ORDER BY columns. **(2)** Replace `SELECT *` with specific columns. **(3)** Use `EXPLAIN`/`EXPLAIN ANALYZE` to see execution plan. **(4)** Denormalize for read-heavy queries. **(5)** Analyze slow query log. **(6)** Use covering indexes, avoid functions on indexed columns. *(SQL optimize karne ke liye pehle EXPLAIN dekho, phir index lagao, SELECT * hatao)*

### 📖 How It Works
```
Optimization Checklist:
├── 1. Indexing
│   ├── B-Tree index on WHERE/JOIN/ORDER BY columns
│   ├── Composite index: column order matters (leftmost prefix rule)
│   ├── Covering index: all SELECT columns in index → no table lookup
│   └── Avoid: indexing low-cardinality columns (boolean, status)
├── 2. Query rewriting
│   ├── Remove SELECT * → specify columns
│   ├── WHERE before JOIN (let DB optimizer handle)
│   ├── EXISTS vs IN (EXISTS for correlated subqueries)
│   └── LIMIT for pagination (with keyset/cursor for large offsets)
├── 3. EXPLAIN analysis
│   ├── type: ALL (bad) → index → range → ref → const (best)
│   ├── rows: estimated rows scanned
│   ├── Extra: Using index (good), Using filesort (bad)
│   └── key: which index was used
├── 4. Denormalization
│   ├── Pre-computed aggregates for dashboards
│   ├── Redundant columns to avoid JOINs
│   └── Materialized views for complex queries
└── 5. Slow log analysis
    ├── Enable slow_query_log (MySQL)
    ├── Set long_query_time threshold
    └── Use pt-query-digest for analysis
```

### 💻 Code
```sql
-- Before: Full table scan
SELECT * FROM orders WHERE YEAR(created_at) = 2025 AND status = 'ACTIVE';

-- After: Use index, avoid function on column
CREATE INDEX idx_orders_created_status ON orders(created_at, status);
SELECT order_id, amount, created_at
FROM orders
WHERE created_at >= '2025-01-01' AND created_at < '2026-01-01'
  AND status = 'ACTIVE';

-- EXPLAIN output
EXPLAIN SELECT ... FROM orders WHERE ...;
-- type: range (good), key: idx_orders_created_status, rows: 1500

-- Covering index (all columns in index → no table access)
CREATE INDEX idx_covering ON orders(status, created_at, order_id, amount);
```

### ⚡ Remember
- **Rule**: index columns in WHERE, JOIN, ORDER BY — but don't over-index (slows writes)
- **EXPLAIN types** (best to worst): const → eq_ref → ref → range → index → ALL
- Functions on indexed columns disable index: `WHERE YEAR(date)` → `WHERE date >= '2025-01-01'`
- Pagination: `OFFSET 10000` is slow → use keyset pagination: `WHERE id > last_seen_id LIMIT 20`

---

<a id="q14"></a>
## Q14. OAuth vs JWT — How do they compare?

### 📝 One-Liner
OAuth 2.0 is an authorization **framework** (delegation protocol); JWT is a **token format** — they work together but solve different problems.

### 🔑 Quick Answer
**OAuth 2.0** defines flows for granting third-party access to resources (Authorization Code, Client Credentials, etc.). **JWT** is a compact, self-contained token format (header.payload.signature) often used as the token in OAuth flows. OAuth can use JWT tokens, opaque tokens, or other formats. *(OAuth ek framework hai jo access delegate karta hai, JWT ek token format hai — dono saath mein use hote hain par alag cheezein hain)*

### 📖 How It Works
```
OAuth 2.0 Authorization Code Flow (with JWT):
┌─────┐     ┌───────────┐     ┌──────────────┐     ┌──────────┐
│User │     │ Client App│     │ Auth Server  │     │ Resource │
│     │     │           │     │ (Keycloak/   │     │ Server   │
│     │     │           │     │  Auth0)      │     │ (API)    │
└─┬───┘     └─────┬─────┘     └──────┬───────┘     └────┬─────┘
  │ 1. Login      │                   │                   │
  │──────────────►│ 2. Redirect to    │                   │
  │               │   auth server     │                   │
  │               │──────────────────►│                   │
  │ 3. User consents                  │                   │
  │──────────────────────────────────►│                   │
  │               │ 4. Auth code      │                   │
  │               │◄──────────────────│                   │
  │               │ 5. Exchange code  │                   │
  │               │   for JWT token   │                   │
  │               │──────────────────►│                   │
  │               │ 6. JWT Access +   │                   │
  │               │   Refresh Token   │                   │
  │               │◄──────────────────│                   │
  │               │ 7. API call with  │                   │
  │               │   Bearer JWT      │                   │
  │               │──────────────────────────────────────►│
  │               │                   │ 8. Validate JWT   │
  │               │ 9. Response       │   (signature +    │
  │               │◄──────────────────────────────────────│   claims)
```

### 🆚 vs.
| Aspect | OAuth 2.0 | JWT |
|--------|-----------|-----|
| What is it | Authorization framework | Token format |
| Purpose | Delegate access | Encode claims |
| Self-contained | N/A | ✅ (no DB lookup) |
| Revocation | Token revocation endpoint | ❌ Hard (see next Q) |
| Expiry | Configurable | `exp` claim |
| Flows | Auth Code, Client Creds, etc. | N/A |
| Token type | Can use JWT or opaque | Is the token |

### ⚡ Remember
- OAuth = **who can access what** (authorization); JWT = **how to represent that**
- JWT structure: `base64(header).base64(payload).signature`
- OAuth without JWT: opaque tokens require server-side DB lookup
- OAuth with JWT: stateless validation — just verify signature + claims
- OIDC (OpenID Connect) adds authentication ON TOP of OAuth 2.0

---

<a id="q15"></a>
## Q15. Can JWTs be invalidated? Explain refresh token flows

### 📝 One-Liner
JWTs are stateless and can't be directly invalidated — workarounds include short expiry + refresh tokens, token blacklists, or token versioning.

### 🔑 Quick Answer
JWT is self-contained — once issued, it's valid until expiry. Invalidation strategies: **(1)** Short-lived access tokens (5-15 min) + refresh tokens. **(2)** Token blacklist in Redis. **(3)** Token version in DB (increment on logout). **(4)** Rotate refresh tokens (one-time use). Refresh token flow: access token expires → client sends refresh token → server issues new access + refresh tokens. *(JWT revoke nahi hota directly — short expiry + refresh token se manage karo; critical cases mein blacklist use karo)*

### 📖 How It Works
```
Refresh Token Flow:
1. Login → Access Token (15 min) + Refresh Token (7 days)
2. API call with Access Token → 200 OK
3. Access Token expires → 401 Unauthorized
4. Client sends Refresh Token to /auth/refresh
5. Server validates Refresh Token (check DB/Redis):
   ├── Valid → Issue new Access Token + new Refresh Token
   │          (old refresh token invalidated — rotation)
   └── Invalid/Expired → 401 → redirect to login

Token Blacklist (for forced invalidation):
┌──────────────────────────────────┐
│ Redis Blacklist                  │
│ key: "blacklist:{jti}"           │
│ value: 1                         │
│ TTL: remaining token expiry time │
└──────────────────────────────────┘
On every request: check if token's JTI is in blacklist
```

### 💻 Code
```java
// Refresh token endpoint
@PostMapping("/auth/refresh")
public ResponseEntity<TokenResponse> refresh(@RequestBody RefreshRequest request) {
    // 1. Validate refresh token
    RefreshToken stored = refreshTokenRepo.findByToken(request.getRefreshToken())
        .orElseThrow(() -> new AuthException("Invalid refresh token"));

    if (stored.getExpiryDate().isBefore(Instant.now())) {
        refreshTokenRepo.delete(stored);
        throw new AuthException("Refresh token expired");
    }

    // 2. Rotate: delete old, create new
    refreshTokenRepo.delete(stored);
    String newAccessToken = jwtService.generateAccessToken(stored.getUser());
    RefreshToken newRefreshToken = refreshTokenRepo.save(
        new RefreshToken(UUID.randomUUID().toString(), stored.getUser(), Instant.now().plusSeconds(604800))
    );

    return ResponseEntity.ok(new TokenResponse(newAccessToken, newRefreshToken.getToken()));
}

// Token blacklist check (for logout/forced invalidation)
public boolean isBlacklisted(String jti) {
    return redisTemplate.hasKey("blacklist:" + jti);
}

public void blacklist(String jti, long remainingTTL) {
    redisTemplate.opsForValue().set("blacklist:" + jti, "1", remainingTTL, TimeUnit.SECONDS);
}
```

### ⚠️ Pitfalls
- Refresh token stored in httpOnly secure cookie (not localStorage — XSS risk)
- Refresh token rotation: if old token reused → possible theft → invalidate all user tokens
- Blacklist must be checked on **every request** — adds latency (Redis mitigates this)
- Don't store sensitive data in JWT payload (it's base64-encoded, not encrypted)

### ⚡ Remember
- Access Token: short-lived (5-15 min), stateless, in Authorization header
- Refresh Token: long-lived (7-30 days), stored in DB/Redis, rotated on use
- Blacklist: for immediate invalidation (logout, password change, suspicious activity)
- Best practice: short access token + refresh rotation + blacklist for critical ops

---

## Round 5: HR + Culture Fit

---

<a id="q16"></a>
## Q16. Why Goldman Sachs?

### 📝 One-Liner
Focus on engineering culture, scale of problems, impact on global financial systems, and career growth in a tech-first environment.

### 🗣️ Answering Approach
"Goldman Sachs stands out for its engineering-first approach to finance. The scale of problems — processing millions of transactions in real-time, building low-latency trading platforms, and ensuring security at a global level — is what excites me as an engineer. I'm drawn to the culture of intellectual rigor and the opportunity to work with some of the brightest minds in technology. The firm's investment in internal platforms and open-source contributions (like Legend, Alloy) shows a genuine commitment to engineering excellence. For my career, GS offers the unique combination of cutting-edge tech challenges with real-world financial impact."

### ⚡ Remember
- Research: GS Engineering blog, open-source projects (Legend/Alloy), tech talks
- Mention specific tech: low-latency systems, data platforms, SecDB, Marquee
- Show genuine interest in financial domain + technical depth
- Avoid: just mentioning brand name or compensation

---

<a id="q17"></a>
## Q17. How do you handle pressure and tight deadlines?

### 🗣️ Answering Approach
"I break the problem into prioritized tasks — identify what must be done versus what's nice-to-have. In a recent project, we had a critical payment integration deadline. I created a task breakdown, communicated with stakeholders about realistic timelines, and focused the team on the critical path. We shipped the core functionality on time and delivered enhancements in the next sprint. I've learned that transparent communication about progress and blockers is more effective than working silently under pressure. I also invest in automation — CI/CD, automated testing — which gives us confidence to ship fast without compromising quality."

### ⚡ Remember
- Use STAR method: Situation → Task → Action → Result
- Emphasize: prioritization, communication, breaking problems down
- Show: you don't just absorb pressure, you manage it systematically
- Real example is stronger than theoretical answer

---

<a id="q18"></a>
## Q18. Describe your experience working with global/distributed teams

### 🗣️ Answering Approach
"I've worked with teams across multiple time zones — coordinating with team members in the US and Europe. Key practices: async communication via documented decisions (Confluence/wiki), overlapping hours for critical sync meetings, and clear ownership of tasks. I learned to write detailed handoff notes so the team in the other timezone could continue without being blocked. Tools like Slack channels organized by project, shared Jira boards, and recorded design reviews helped maintain context. The biggest lesson was that over-communication is better than under-communication in distributed teams."

### ⚡ Remember
- Highlight: async communication, documentation, time-zone management
- Tools: Slack, Jira, Confluence, video calls
- Challenges: mention them honestly + how you overcame them
- GS is global — they want proof you can collaborate across offices

---

<a id="q19"></a>
## Q19. What are your career growth plans?

### 🗣️ Answering Approach
"In the short term, I want to deepen my expertise in distributed systems and low-latency architecture — areas where Goldman Sachs operates at world-class scale. Within 2-3 years, I see myself taking technical leadership of complex projects, mentoring junior developers, and contributing to architectural decisions. Long-term, I aspire to become a principal engineer or architect who can bridge the gap between business requirements and technical solutions. I believe GS provides the right environment — challenging problems, smart peers, and a culture that values engineering depth."

### ⚡ Remember
- Align growth plan with what the company offers
- Show ambition but root it in the current role
- Mention: technical depth, leadership, mentoring — not just promotions
- Avoid: "I want to start my own company" or focus purely on managerial track

---

## 📊 Summary

| Round | Focus | Questions | Difficulty |
|-------|-------|-----------|------------|
| Round 1 | Online Assessment (DSA) | Q1–Q2 | 🟡 Medium |
| Round 2 | DSA + Core Java | Q3–Q7 | 🟡–🔴 Medium-Hard |
| Round 3 | Spring Boot + Low-Level Design | Q8–Q12 | 🔴 Hard |
| Round 4 | Security + DB Optimization | Q13–Q15 | 🔴 Hard |
| Round 5 | HR + Culture Fit | Q16–Q19 | — |

**Key Takeaway from Goldman Sachs:**
> "Goldman doesn't test topics. They test **thinking depth.**"
- Every question had deep follow-ups probing understanding
- They care about WHY, not just WHAT
- System design questions expected real-world trade-off discussions
- Security questions expected production incident awareness
