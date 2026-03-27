# 🏢 American Express — Java Backend Developer Interview Experience (2–4 YOE, March 2026)

> Five sections covering Core Java concurrency, Spring Boot internals, Database fundamentals, DSA coding problems, and System Design architecture. Heavy focus on production-readiness and design thinking.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Section A: Core Java & Concurrency

---

<a id="q1"></a>
## Q1. What is the difference between synchronized, ReentrantLock, and ReadWriteLock? When would you use each?

### 📝 One-Liner
`synchronized` is a built-in monitor lock (simple but rigid), `ReentrantLock` adds flexibility (tryLock, fairness, interruptible), and `ReadWriteLock` optimizes for read-heavy workloads by allowing concurrent readers but exclusive writers.

### 🔑 Quick Answer
**synchronized**: keyword-level, auto-release, no timeout, non-interruptible. **ReentrantLock**: API-level, `tryLock(timeout)`, `lockInterruptibly()`, fairness policy, multiple conditions. **ReadWriteLock**: separate `readLock()` (shared) + `writeLock()` (exclusive) — multiple threads read simultaneously, writing blocks all. *(synchronized simple hai lock ke liye, ReentrantLock zyada control deta hai, ReadWriteLock read-heavy workload ke liye best hai)*

### 💻 Code
```java
// 1. synchronized — simplest
public synchronized void deposit(BigDecimal amount) {
    balance = balance.add(amount);
}

// 2. ReentrantLock — flexible
private final ReentrantLock lock = new ReentrantLock(true); // fair
public void transfer(Account to, BigDecimal amount) {
    if (lock.tryLock(2, TimeUnit.SECONDS)) {
        try { /* transfer logic */ }
        finally { lock.unlock(); } // MUST unlock in finally
    }
}

// 3. ReadWriteLock — read-heavy optimization
private final ReadWriteLock rwLock = new ReentrantReadWriteLock();
public BigDecimal getBalance() {
    rwLock.readLock().lock();     // Multiple readers allowed
    try { return balance; }
    finally { rwLock.readLock().unlock(); }
}
public void setBalance(BigDecimal b) {
    rwLock.writeLock().lock();    // Exclusive — blocks readers + writers
    try { balance = b; }
    finally { rwLock.writeLock().unlock(); }
}
```

### 🆚 vs.
| Feature | synchronized | ReentrantLock | ReadWriteLock |
|---------|-------------|---------------|---------------|
| Lock type | Intrinsic monitor | Explicit lock | Read + Write pair |
| Timeout | ❌ | ✅ `tryLock(time)` | ✅ |
| Fairness | ❌ No | ✅ Constructor | ✅ Constructor |
| Interruptible | ❌ | ✅ `lockInterruptibly()` | ✅ |
| Multiple conditions | ❌ 1 wait-set | ✅ `newCondition()` | ✅ |
| Concurrent reads | ❌ | ❌ | ✅ readLock shared |
| Auto-release | ✅ Block exit | ❌ Manual `unlock()` | ❌ Manual |
| Best for | Simple sync | Complex lock control | Read-heavy (95%+ reads) |

### ⚡ Remember
> `synchronized` for simple cases | `ReentrantLock` for tryLock/fairness/multiple conditions | `ReadWriteLock` when reads >> writes | **Always unlock in finally!**

---

<a id="q2"></a>
## Q2. Explain fail-fast vs fail-safe iterators in Java collections

### 📝 One-Liner
**Fail-fast** iterators (ArrayList, HashMap) throw `ConcurrentModificationException` on structural modification during iteration; **fail-safe** iterators (ConcurrentHashMap, CopyOnWriteArrayList) work on a copy/snapshot and never throw CME.

### 🔑 Quick Answer
Fail-fast uses `modCount` — if collection is modified during iteration, next `hasNext()`/`next()` detects mismatch and throws CME. Fail-safe creates a snapshot (CopyOnWriteArrayList) or uses segments (ConcurrentHashMap) — modifications during iteration are invisible to the iterator. *(Fail-fast turant error deta hai agar koi iteration ke beech mein modify kare, fail-safe copy pe kaam karta hai toh error nahi aata)*

### 💻 Code
```java
// Fail-fast — throws ConcurrentModificationException
List<String> list = new ArrayList<>(List.of("A", "B", "C"));
for (String s : list) {
    if (s.equals("B")) list.remove(s); // 💥 CME!
}
// Fix: use Iterator.remove() or CopyOnWriteArrayList

// Fail-safe — no exception
List<String> cowList = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
for (String s : cowList) {
    if (s.equals("B")) cowList.remove(s); // ✅ Works! Iterates over snapshot
}
```

### ⚡ Remember
> **Fail-fast**: ArrayList, HashMap, HashSet — uses `modCount` | **Fail-safe**: ConcurrentHashMap, CopyOnWriteArrayList — snapshot/segment | Use `Iterator.remove()` to safely remove during iteration

---

<a id="q3"></a>
## Q3. Why are Strings immutable in Java and what benefits does it provide in multithreaded environments?

### 📝 One-Liner
Strings are immutable for **security** (safe for passwords, class names, URLs), **caching** (String Pool reuse saves memory), **hashCode caching** (safe HashMap keys), and **thread safety** (no synchronization needed — immutable objects are inherently thread-safe).

### 🔑 Quick Answer
**Why immutable**: (1) **String Pool** — JVM reuses identical strings; mutation would corrupt shared references. (2) **Security** — class loading, JDBC URLs, file paths use Strings; mutation would be a vulnerability. (3) **HashCode cache** — computed once, cached forever, perfect for HashMap keys. (4) **Thread safety** — multiple threads can share same String without locks. *(String immutable hai toh pool mein share ho sakta hai, hashCode cache hota hai, aur thread-safe by default hai)*

### 🆚 vs.
| | String | StringBuilder | StringBuffer |
|--|--------|--------------|--------------|
| Mutable | ❌ | ✅ | ✅ |
| Thread-safe | ✅ (immutable) | ❌ | ✅ (synchronized) |
| Pool | ✅ String Pool | ❌ | ❌ |
| Performance | New object each concat | Fast (single thread) | Slower (sync overhead) |

### ⚡ Remember
> Immutable = thread-safe without locks | String Pool saves memory | HashCode cached on first call | Use StringBuilder for concatenation in loops

---

<a id="q4"></a>
## Q4. How does the ForkJoinPool work and when should it be used?

### 📝 One-Liner
ForkJoinPool uses a **work-stealing** algorithm where idle threads steal tasks from busy threads' deques — designed for **recursive divide-and-conquer** parallelism (large task → split → process → join results).

### 🔑 Quick Answer
**How**: Task is `fork()`ed into subtasks recursively until small enough to compute directly, then results are `join()`ed back. Each thread has a double-ended queue (deque) — pushes/pops own tasks from tail, steals from other threads' head. **When**: CPU-bound recursive problems (merge sort, tree traversal, image processing). **Default pool**: `ForkJoinPool.commonPool()` — used by `parallelStream()` and `CompletableFuture`. *(Bada task chote tasks mein split karo, parallel process karo, results merge karo — idle threads dusron ka kaam chura lete hain)*

### 📖 How It Works
```
ForkJoinPool (Work-Stealing):

  Thread-1 deque: [TaskA1, TaskA2, TaskA3]  ← pushes own tasks
  Thread-2 deque: [TaskB1]                   ← almost done
  Thread-3 deque: []                         ← idle

  Thread-3 steals TaskA3 from Thread-1's head!
  → All threads stay busy → Maximum CPU utilization

RecursiveTask Example (Sum of array):
  [1, 2, 3, 4, 5, 6, 7, 8]
       fork ↙         ↘ fork
  [1, 2, 3, 4]    [5, 6, 7, 8]
    fork ↙ ↘        fork ↙ ↘
  [1,2] [3,4]    [5,6] [7,8]
   3  +  7    +   11 +  15  = 36 (join)
```

### 💻 Code
```java
// RecursiveTask — returns a result
public class SumTask extends RecursiveTask<Long> {
    private final int[] array;
    private final int start, end;
    private static final int THRESHOLD = 1000;

    public SumTask(int[] array, int start, int end) {
        this.array = array; this.start = start; this.end = end;
    }

    @Override
    protected Long compute() {
        if (end - start <= THRESHOLD) {
            long sum = 0;
            for (int i = start; i < end; i++) sum += array[i];
            return sum;
        }
        int mid = (start + end) / 2;
        SumTask left = new SumTask(array, start, mid);
        SumTask right = new SumTask(array, mid, end);
        left.fork();           // async execute left
        long rightResult = right.compute(); // compute right in current thread
        long leftResult = left.join();      // wait for left
        return leftResult + rightResult;
    }
}

// Usage
ForkJoinPool pool = new ForkJoinPool(); // or ForkJoinPool.commonPool()
long total = pool.invoke(new SumTask(bigArray, 0, bigArray.length));
```

### ⚠️ Pitfalls
| Mistake | Fix |
|---------|-----|
| Using for I/O-bound tasks | FJP is for CPU-bound — use ExecutorService for I/O |
| Blocking inside ForkJoinPool | Blocks work-stealing — use `ManagedBlocker` |
| Too small threshold | Overhead of forking exceeds computation — tune threshold |
| Starving commonPool | All `parallelStream()` uses commonPool — long tasks block others |

### ⚡ Remember
> **Work-stealing** = idle threads steal from busy ones | `RecursiveTask` (returns result) vs `RecursiveAction` (void) | Threshold tuning critical | `commonPool()` shared by parallelStream + CompletableFuture

---

## Section B: Spring Boot & Backend

---

<a id="q5"></a>
## Q5. What is the difference between @Bean and @Component?

### 📝 One-Liner
`@Component` is a **class-level** stereotype annotation — Spring auto-detects via component scanning. `@Bean` is a **method-level** annotation inside `@Configuration` — you manually instantiate and configure the object.

### 🔑 Quick Answer
Use `@Component` when you own the class (your code). Use `@Bean` when you need to create beans from **third-party classes** you can't annotate, or when **custom initialization** logic is needed. `@Bean` gives full control over construction; `@Component` relies on autowiring. *(Apni class pe @Component lagao, third-party class ka bean banana ho toh @Bean method use karo)*

### 🆚 vs.
| Feature | @Component | @Bean |
|---------|-----------|-------|
| Level | Class | Method (in @Configuration) |
| Detection | Component scanning | Explicit registration |
| Third-party classes | ❌ Can't annotate | ✅ Full control |
| Custom init logic | Limited | ✅ Any Java code |
| Specializations | @Service, @Repository, @Controller | None |

### ⚡ Remember
> `@Component` = auto-scan your classes | `@Bean` = manual creation in config | Third-party library? `@Bean` is the way | Both register in ApplicationContext

---

<a id="q6"></a>
## Q6. How do filters, interceptors, and AOP differ in Spring Boot?

### 📝 One-Liner
**Filters** operate at Servlet level (before DispatcherServlet), **Interceptors** at Spring MVC level (around controller), **AOP** at method level (any Spring bean) — each progressively narrows scope but increases Spring integration.

### 🔑 Quick Answer
**Execution order**: Client → Filter → DispatcherServlet → Interceptor → Controller → AOP (on method). **Filter**: `javax.servlet.Filter`, accesses HttpServletRequest/Response, no Spring context. **Interceptor**: `HandlerInterceptor`, has handler info, preHandle/postHandle/afterCompletion. **AOP**: `@Aspect`, cross-cuts any Spring method, @Before/@After/@Around. *(Filter sabse pehle chalta hai Servlet level pe, Interceptor MVC level pe, AOP method level pe — scope chhota hota jaata hai par control badhta jaata hai)*

### 📖 How It Works
```
Request flow:
  Client
    → Filter (Servlet level — logging, CORS, auth token extraction)
      → DispatcherServlet
        → Interceptor preHandle (Spring MVC — auth check, rate limiting)
          → Controller method
            → AOP @Before (method level — audit, caching, transactions)
              → Service method executes
            → AOP @After
          → Controller returns
        → Interceptor postHandle
      → DispatcherServlet renders
    → Filter (response modification)
  Client
```

### 🆚 vs.
| Feature | Filter | Interceptor | AOP |
|---------|--------|-------------|-----|
| Level | Servlet container | Spring MVC | Spring bean method |
| Interface | `javax.servlet.Filter` | `HandlerInterceptor` | `@Aspect` + pointcut |
| Spring context | ❌ No (Servlet API) | ✅ Yes | ✅ Yes |
| Scope | All requests (incl. static) | Only DispatcherServlet | Any Spring bean method |
| Access to handler | ❌ | ✅ `HandlerMethod` | ✅ `JoinPoint` |
| Use cases | CORS, compression, encoding | Auth, logging, locale | Transactions, caching, auditing |

### ⚡ Remember
> **Filter** = Servlet (broadest, no Spring) | **Interceptor** = MVC (handler-aware) | **AOP** = Method (finest, cross-cutting) | Order: Filter → Interceptor → AOP

---

<a id="q7"></a>
## Q7. What are the differences between ApplicationContext and BeanFactory?

### 📝 One-Liner
`BeanFactory` is the basic container (lazy loading, minimal), `ApplicationContext` extends it with **eager loading, event publishing, i18n, AOP integration, and environment abstraction** — Spring Boot always uses ApplicationContext.

### 🔑 Quick Answer
`BeanFactory`: lazy initialization, basic DI only, `getBean()`. `ApplicationContext`: eager initialization (all singletons at startup → fail-fast), plus events (`ApplicationEvent`), message resolution (i18n), `@Profile` support, AOP auto-proxying. In practice, you always use `ApplicationContext` — `BeanFactory` is for memory-constrained edge cases only.

### ⚡ Remember
> `ApplicationContext` = BeanFactory + eager init + events + i18n + AOP + profiles | Spring Boot = always ApplicationContext | BeanFactory = theoretical, rarely used directly

---

<a id="q8"></a>
## Q8. How do you configure multiple data sources in Spring Boot?

### 📝 One-Liner
Disable auto-configuration, create separate `DataSource` → `EntityManagerFactory` → `TransactionManager` beans for each DB, use `@Primary` on the default, and isolate entities/repositories via `basePackages`.

### 🔑 Quick Answer
**Steps**: (1) Define DB properties in `application.yml` with different prefixes (`spring.primary.*`, `spring.secondary.*`). (2) Create `@Configuration` class per data source. (3) Use `DataSourceBuilder` to create DataSource from properties. (4) Create `LocalContainerEntityManagerFactoryBean` with correct entity package scanning. (5) Create `PlatformTransactionManager` per datasource. (6) Mark one as `@Primary`. *(Do databases lagane hain toh har ek ke liye alag DataSource, EntityManager, TransactionManager banana padta hai — @Primary se default set karo)*

### 💻 Code
```java
@Configuration
@EnableTransactionManagement
@EnableJpaRepositories(
    basePackages = "com.amex.repo.primary",
    entityManagerFactoryRef = "primaryEntityManagerFactory",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDbConfig {

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean primaryEntityManagerFactory(
            EntityManagerFactoryBuilder builder) {
        return builder.dataSource(primaryDataSource())
            .packages("com.amex.entity.primary")
            .persistenceUnit("primary")
            .build();
    }

    @Primary
    @Bean
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}
// Repeat for secondary with @Qualifier instead of @Primary
```

### ⚡ Remember
> Disable auto-config | Separate DataSource + EMF + TxManager per DB | `@Primary` for default | `basePackages` isolates repos/entities | Don't mix transactions across data sources

---

<a id="q9"></a>
## Q9. How would you design a scalable REST API in Spring Boot for high traffic?

### 📝 One-Liner
Stateless design + connection pooling + caching (Redis) + async processing + pagination + rate limiting + horizontal scaling behind a load balancer — keep controllers thin, services transactional, responses paginated.

### 🔑 Quick Answer
**Key strategies**: (1) Stateless — no server-side sessions, JWT for auth. (2) Connection pooling — HikariCP for DB, WebClient for HTTP. (3) Caching — Redis/Caffeine for hot data, `@Cacheable`. (4) Async — `@Async`, CompletableFuture, Kafka for decoupling. (5) Pagination — `Pageable` + keyset for large datasets. (6) Rate limiting — Resilience4j `@RateLimiter`. (7) API versioning — URI or header-based. (8) GZIP compression — reduce payload. (9) Horizontal scaling — Docker + K8s + ALB. (10) Monitoring — Micrometer + Prometheus + Grafana.

### ⚡ Remember
> Stateless + Cache + Async + Paginate + Rate-limit + GZIP + Connection pool + Horizontal scale | This is a common system design discussion point

---

## Section C: Database & Backend Fundamentals

---

<a id="q10"></a>
## Q10. What is database indexing and how does it improve query performance?

### 📝 One-Liner
An index is a **separate data structure** (typically B+ tree) that stores sorted column values with pointers to actual rows — turning full table scans into O(log n) lookups.

### 🔑 Quick Answer
**Without index**: DB scans every row (O(n)). **With index**: B+ tree lookup (O(log n)). **Types**: (1) Primary/Clustered — rows physically sorted by PK. (2) Secondary/Non-clustered — separate structure pointing to rows. (3) Composite — multi-column index (order matters!). **Trade-off**: Faster reads, slower writes (index must be updated). *(Index ek sorted pointer structure hai — full scan ki jagah tree search hota hai, bohot fast)*

### 💻 Code
```sql
-- Single column index
CREATE INDEX idx_email ON employees(email);

-- Composite index (order matters!)
CREATE INDEX idx_dept_salary ON employees(department, salary);
-- ✅ Works: WHERE department = 'IT' AND salary > 50000
-- ✅ Works: WHERE department = 'IT'
-- ❌ Doesn't use index: WHERE salary > 50000 (leftmost prefix rule)

-- Check query plan
EXPLAIN ANALYZE SELECT * FROM employees WHERE email = 'john@amex.com';
```

### ⚡ Remember
> B+ tree = O(log n) | Clustered = 1 per table (PK) | Composite = leftmost prefix rule | Faster reads, slower writes | Don't over-index

---

<a id="q11"></a>
## Q11. What is a composite index and when should it be used?

### 📝 One-Liner
A composite index covers **multiple columns** — the column order determines which queries benefit. Follows the **leftmost prefix rule**: index on (A, B, C) supports queries on A, (A,B), and (A,B,C) but NOT on B alone or C alone.

### 🔑 Quick Answer
**When to use**: Queries with multiple WHERE/JOIN conditions on the same columns. **Column order strategy**: (1) Equality columns first. (2) Range columns last. (3) Most selective column first. **Example**: For `WHERE status = 'ACTIVE' AND created_date > '2026-01-01'`, index on `(status, created_date)` — equality column first, range last. *(Composite index mein column order bohot important hai — pehle equality, phir range, nahi toh index use nahi hoga)*

### 💻 Code
```sql
-- Composite index for common query pattern
CREATE INDEX idx_status_date ON orders(status, created_date);

-- ✅ Uses full index
SELECT * FROM orders WHERE status = 'PENDING' AND created_date > '2026-01-01';

-- ✅ Uses partial index (leftmost prefix)
SELECT * FROM orders WHERE status = 'PENDING';

-- ❌ Cannot use this index effectively
SELECT * FROM orders WHERE created_date > '2026-01-01';

-- Covering index (avoids table lookup entirely)
CREATE INDEX idx_covering ON orders(status, created_date, amount);
SELECT status, created_date, amount FROM orders WHERE status = 'ACTIVE';
-- All columns in SELECT are in the index → index-only scan!
```

### ⚡ Remember
> **Leftmost prefix rule** | Equality first, range last | Covering index = index-only scan | Don't create both (A,B) and (A) — composite covers single

---

<a id="q12"></a>
## Q12. What is the difference between optimistic locking and pessimistic locking?

### 📝 One-Liner
**Optimistic** assumes conflicts are rare — uses a version column, checks at commit time, throws exception on conflict. **Pessimistic** assumes conflicts are frequent — locks the row immediately with `SELECT FOR UPDATE`, blocks other transactions.

### 🔑 Quick Answer
**Optimistic** (`@Version` in JPA): No DB lock held, allows concurrent reads, detects conflict on update via version mismatch → `OptimisticLockException`. **Pessimistic** (`@Lock(PESSIMISTIC_WRITE)`): DB-level row lock, blocks concurrent access, prevents conflicts but reduces throughput. Use optimistic for low-contention (most web apps), pessimistic for high-contention (inventory, banking). *(Optimistic mein version check hota hai commit pe — agar kisi ne beech mein badal diya toh exception. Pessimistic mein row lock ho jata hai — koi aur touch nahi kar sakta)*

### 🆚 vs.
| Aspect | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Mechanism | `@Version` column check | `SELECT FOR UPDATE` row lock |
| Blocking | ❌ No — detect on commit | ✅ Yes — immediate lock |
| Conflict handling | Exception → retry | Waits → proceeds |
| Throughput | High (no locks) | Lower (blocking) |
| Best for | Low contention (web CRUD) | High contention (inventory) |
| Deadlock risk | None | Possible |
| JPA annotation | `@Version` | `@Lock(PESSIMISTIC_WRITE)` |

### ⚡ Remember
> Optimistic = version check, no lock, retry on conflict | Pessimistic = row lock, blocks others | Low contention → optimistic | High contention → pessimistic

---

<a id="q13"></a>
## Q13. How do you handle database transactions across microservices?

### 📝 One-Liner
Use the **Saga pattern** — a sequence of local transactions where each service commits its own transaction, and compensating transactions are executed on failure to maintain eventual consistency.

### 🔑 Quick Answer
**No distributed 2PC** in microservices (too slow, tightly coupled). **Saga**: Two approaches — (1) **Choreography**: services emit events, each reacts and triggers next step. (2) **Orchestration**: a central orchestrator directs the flow and triggers compensations. Each service has a local transaction + a compensating action (undo). *(Microservices mein ek bada transaction nahi hota — Saga pattern use karo jahan har service apna local transaction karta hai, failure pe compensating transaction undo karta hai)*

### 📖 How It Works
```
Order Saga (Orchestration):
  1. Order Service  → Create Order (PENDING)      Compensate: Cancel Order
  2. Payment Service → Charge Card                  Compensate: Refund
  3. Inventory Service → Reserve Stock              Compensate: Release Stock
  4. Notification → Send Confirmation               No compensation needed

If Step 3 fails (out of stock):
  → Compensate Step 2 (Refund) → Compensate Step 1 (Cancel Order)
  → Eventual consistency maintained!
```

### ⚡ Remember
> Saga = local transactions + compensations | Choreography (events, decoupled) vs Orchestration (central coordinator) | No 2PC in microservices | Idempotent compensations essential

---

## Section D: DSA / Coding Problems

---

<a id="q14"></a>
## Q14. Minimum deletions required to make character frequencies unique in a string

### 📝 One-Liner
Count character frequencies, then greedily reduce duplicate frequencies — for each repeated frequency, decrement until it's unique or zero. Count total decrements as deletions.

### 🔑 Quick Answer
**Approach**: (1) Count frequency of each character. (2) Sort frequencies descending. (3) For each frequency, if already used, decrement until it's unique or 0. (4) Sum all decrements. **Time**: O(n + 26²) ≈ O(n). *(Har character ki frequency count karo, agar do characters ki same frequency hai toh ek ko kam karo jab tak unique na ho jaye)*

### 💻 Code
```java
public int minDeletions(String s) {
    int[] freq = new int[26];
    for (char c : s.toCharArray()) freq[c - 'a']++;

    Set<Integer> used = new HashSet<>();
    int deletions = 0;

    for (int f : freq) {
        if (f == 0) continue;
        while (f > 0 && used.contains(f)) {
            f--;
            deletions++;
        }
        if (f > 0) used.add(f);
    }
    return deletions;
}
// Input: "aaabbbcc" → freq: a=3, b=3, c=2
// b's 3 conflicts with a's 3 → decrement b to 2 (1 deletion)
// c's 2 conflicts with b's 2 → decrement c to 1 (1 deletion)
// Total: 2 deletions → frequencies: {3, 2, 1} all unique
```

### ⚡ Remember
> Frequency map → greedy decrement duplicates | HashSet tracks used frequencies | O(n) time | LC #1647 "Minimum Deletions to Make Character Frequencies Unique"

---

<a id="q15"></a>
## Q15. Find the longest prefix where removing one element makes frequencies equal

### 📝 One-Liner
Iterate through the string tracking character frequencies. At each position, check if removing exactly one character would make all remaining frequencies equal. Track the longest valid prefix.

### 🔑 Quick Answer
**Approach**: Maintain frequency counts and frequency-of-frequency counts. At each index, check conditions: (1) All chars have frequency 1 (remove any). (2) Only 1 unique char (remove one occurrence). (3) All chars same frequency and one char has freq+1 (remove from that). (4) All chars same frequency and one char has freq 1 (remove that char entirely). *(String ke prefix mein ek character hatao toh sab ki frequency equal ho jaye — sabse lamba aisa prefix dhundho)*

### 💻 Code
```java
public int maxEqualFreqPrefix(String s) {
    int[] charFreq = new int[26];        // frequency of each char
    int[] freqCount = new int[s.length() + 1]; // count of each frequency
    int maxFreq = 0, result = 0;

    for (int i = 0; i < s.length(); i++) {
        int c = s.charAt(i) - 'a';
        if (charFreq[c] > 0) freqCount[charFreq[c]]--;
        charFreq[c]++;
        freqCount[charFreq[c]]++;
        maxFreq = Math.max(maxFreq, charFreq[c]);
        int len = i + 1;
        int uniqueChars = (int) Arrays.stream(charFreq).filter(f -> f > 0).count();

        // Check if removing 1 element makes all frequencies equal:
        // Case 1: All frequencies are 1 (remove any one)
        // Case 2: Only 1 character exists (remove one occurrence)
        // Case 3: maxFreq * uniqueChars == len - 1 && freqCount[maxFreq] == uniqueChars
        // Case 4: (maxFreq - 1) * uniqueChars + 1 == len
        // Case 5: maxFreq == 1
        if (maxFreq == 1                                           // all freq 1
            || uniqueChars == 1                                     // single char
            || (maxFreq * uniqueChars == len - 1)                  // one extra overall
            || ((maxFreq - 1) * uniqueChars + 1 == len             // one char has +1
                && freqCount[maxFreq] == 1)) {
            result = len;
        }
    }
    return result;
}
```

### ⚡ Remember
> Track char frequency + frequency of frequencies | Check 4-5 edge conditions at each prefix | O(n) time, O(26) space | Requires careful condition enumeration

---

<a id="q16"></a>
## Q16. Find the longest substring without repeating characters

### 📝 One-Liner
Use a **sliding window** with a HashSet/HashMap — expand right pointer adding chars, when duplicate found, shrink left pointer until duplicate removed. Track max window size.

### 🔑 Quick Answer
**Optimized approach**: HashMap stores each character's latest index. When duplicate found, jump left pointer to `max(left, lastIndex + 1)`. **Time**: O(n), **Space**: O(min(n, 26)). *(Sliding window use karo — right pointer se expand karo, duplicate mile toh left pointer aage badhao — max window size track karo)*

### 💻 Code
```java
public int lengthOfLongestSubstring(String s) {
    Map<Character, Integer> lastIndex = new HashMap<>();
    int maxLen = 0, left = 0;

    for (int right = 0; right < s.length(); right++) {
        char c = s.charAt(right);
        if (lastIndex.containsKey(c) && lastIndex.get(c) >= left) {
            left = lastIndex.get(c) + 1; // Jump past the duplicate
        }
        lastIndex.put(c, right);
        maxLen = Math.max(maxLen, right - left + 1);
    }
    return maxLen;
}
// "abcabcbb" → Window: abc(3) → bca(3) → cab(3) → abc(3) → max = 3
// "pwwkew"   → pw(2) → wke(3) → kew(3) → max = 3
```

### ⚡ Remember
> Sliding window + HashMap | Jump left past duplicate | O(n) time | LC #3 classic | Most common string problem in interviews

---

<a id="q17"></a>
## Q17. Solve the Fruit Into Baskets problem using a sliding window approach

### 📝 One-Liner
Find the longest contiguous subarray with **at most 2 distinct** elements — sliding window with a HashMap tracking fruit type counts, shrink window when distinct types exceed 2.

### 🔑 Quick Answer
**Approach**: Expand right, add fruit to map. When `map.size() > 2`, shrink left — decrement count, remove from map when count reaches 0. Track max window size throughout. **Time**: O(n). This is equivalent to "Longest Substring with At Most K Distinct Characters" where K=2.

### 💻 Code
```java
public int totalFruit(int[] fruits) {
    Map<Integer, Integer> basket = new HashMap<>(); // fruit type → count
    int maxFruits = 0, left = 0;

    for (int right = 0; right < fruits.length; right++) {
        basket.merge(fruits[right], 1, Integer::sum);

        while (basket.size() > 2) {
            int leftFruit = fruits[left];
            basket.merge(leftFruit, -1, Integer::sum);
            if (basket.get(leftFruit) == 0) basket.remove(leftFruit);
            left++;
        }
        maxFruits = Math.max(maxFruits, right - left + 1);
    }
    return maxFruits;
}
// [1,2,1]     → {1:2, 2:1} → 3 (all fit in 2 baskets)
// [0,1,2,2]   → {0:1,1:1}=2, add 2→{0:1,1:1,2:1}>2 → shrink → {1:1,2:1}→{1:1,2:2}=3 → max=3
// [1,2,3,2,2] → {1:1,2:1}=2 → {1:1,2:1,3:1}>2 → shrink → max eventually = 4 [2,3,2,2]
```

### ⚡ Remember
> Sliding window + HashMap (size ≤ 2) | Shrink left when >2 distinct | O(n) time | LC #904 | Generalize to "at most K distinct" by changing 2 → K

---

<a id="q18"></a>
## Q18. Solve the Celebrity Problem using a stack-based approach

### 📝 One-Liner
A celebrity is someone **everyone knows** but **knows no one**. Push all people onto stack, pop two at a time — if A knows B, A isn't celebrity (discard); if A doesn't know B, B isn't celebrity (discard). Verify final candidate.

### 🔑 Quick Answer
**Stack approach**: (1) Push all N people. (2) Pop two, check `knows(a, b)` — eliminate one. (3) Repeat until 1 remains (candidate). (4) Verify candidate: everyone knows them AND they know no one. **Time**: O(n), **Space**: O(n) for stack.

### 💻 Code
```java
public int findCelebrity(int n) {
    Stack<Integer> stack = new Stack<>();
    for (int i = 0; i < n; i++) stack.push(i);

    // Eliminate non-celebrities
    while (stack.size() > 1) {
        int a = stack.pop();
        int b = stack.pop();
        if (knows(a, b)) stack.push(b); // A knows B → A is NOT celebrity
        else stack.push(a);             // A doesn't know B → B is NOT celebrity
    }

    // Verify the candidate
    int candidate = stack.pop();
    for (int i = 0; i < n; i++) {
        if (i == candidate) continue;
        if (knows(candidate, i) || !knows(i, candidate)) return -1;
    }
    return candidate;
}

// Two-pointer alternative (O(1) space):
public int findCelebrityTwoPointer(int n) {
    int candidate = 0;
    for (int i = 1; i < n; i++) {
        if (knows(candidate, i)) candidate = i;
    }
    // Verify candidate
    for (int i = 0; i < n; i++) {
        if (i != candidate && (knows(candidate, i) || !knows(i, candidate)))
            return -1;
    }
    return candidate;
}
```

### ⚡ Remember
> Stack: pop 2, eliminate 1 → O(n) | Two-pointer: O(1) space | Always verify final candidate | LC #277 | N-1 comparisons to find candidate, N-1 to verify

---

## Section E: System Design & Architecture

---

<a id="q19"></a>
## Q19. Design a payment gateway system that handles retries and failures

### 📝 One-Liner
Idempotent payments with unique `idempotencyKey`, state machine (INITIATED→PROCESSING→SUCCESS/FAILED), retry with exponential backoff, dead letter queue for unrecoverable failures, reconciliation job for consistency.

### 🔑 Quick Answer
**Key components**: (1) **Idempotency** — unique key per payment, DB stores state, duplicate requests return cached result. (2) **State machine** — INITIATED → PROCESSING → SUCCESS/FAILED/TIMEOUT. (3) **Retry strategy** — exponential backoff (1s, 2s, 4s, max 3 retries) for transient failures. (4) **Circuit breaker** — stop retrying when payment provider is down. (5) **DLQ** — dead letter queue for manual investigation. (6) **Reconciliation** — periodic job compares internal records with provider's records. *(Payment gateway mein idempotency key sabse important hai — retry karo par duplicate charge mat karo)*

### 📖 How It Works
```
Payment Flow:
  Client → API Gateway → Payment Service
    1. Validate request + check idempotency key
    2. Create payment record (state: INITIATED)
    3. Call Payment Provider (Stripe/Razorpay)
       ├── Success → Update state: SUCCESS → Notify
       ├── Failure (4xx) → Update state: FAILED → Return error
       └── Timeout/5xx → Update state: RETRY_PENDING
           → Retry with exponential backoff (max 3)
           ├── Eventually succeeds → SUCCESS
           └── Max retries exhausted → FAILED → DLQ

  Idempotency:
    POST /payments (idempotencyKey: "ord-123-pay-1")
    → Check DB: key exists? → Return cached result
    → Key not found? → Process payment → Store result

  Reconciliation (every hour):
    Compare: Internal DB records vs Payment Provider records
    Flag mismatches → Alert → Manual resolution
```

### ⚡ Remember
> **Idempotency key** prevents duplicate charges | State machine tracks payment lifecycle | Exponential backoff for retries | Circuit breaker for provider outages | DLQ for unrecoverable | Reconciliation for consistency

---

<a id="q20"></a>
## Q20. Design a shopping cart service that supports millions of users

### 📝 One-Liner
Redis for active carts (fast in-memory, TTL for expiry), persistent DB for checkout, event-driven stock validation, optimistic locking for concurrent updates, CDN for product catalog.

### 🔑 Quick Answer
**Architecture**: (1) **Redis** — primary cart store (key: `cart:{userId}`, value: JSON of items, TTL: 7 days). (2) **Product Service** — cached catalog, stock checks via events. (3) **Cart Service** — CRUD operations on Redis, validates stock on add/checkout. (4) **Checkout** — moves cart to Order DB (PostgreSQL), reserves inventory, initiates payment. **Scaling**: Redis Cluster for sharding, stateless cart service behind ALB, CDN for product images. *(Cart Redis mein rakho — fast hai, TTL se expire hota hai, checkout pe DB mein move karo)*

### ⚡ Remember
> Redis for active carts (TTL) | DB for orders post-checkout | Optimistic locking for concurrent cart updates | Event-driven stock validation | Redis Cluster for millions of users

---

<a id="q21"></a>
## Q21. Design a secure checkout system with fraud detection

### 📝 One-Liner
Multi-layered: input validation → rate limiting → rule-based fraud checks (velocity, geolocation, device fingerprint) → ML risk scoring → 3D Secure for high-risk → audit logging for all decisions.

### 🔑 Quick Answer
**Fraud detection layers**: (1) **Velocity checks** — max 5 orders/hour per user, max 3 failed payments/day. (2) **Geolocation** — IP country vs billing address mismatch. (3) **Device fingerprint** — new device + high value = flag. (4) **ML risk score** — trained on historical fraud data, scores 0-100. (5) **Rules engine** — configurable thresholds (score > 70 → 3DS challenge, > 90 → block). (6) **3D Secure** — redirect to card issuer for additional verification on high-risk. *(Fraud detection layers mein rule-based + ML score + 3DS challenge — high value aur new device pe extra verification)*

### ⚡ Remember
> Velocity + Geo + Device fingerprint + ML score + 3DS challenge | Audit log every decision | PCI-DSS compliance for card data | Never store CVV

---

<a id="q22"></a>
## Q22. How do you ensure high availability and fault tolerance in microservices?

### 📝 One-Liner
Redundancy at every layer: multiple instances behind LB, circuit breakers for cascading failure prevention, health checks, graceful degradation, multi-AZ deployment, and chaos engineering for validation.

### 🔑 Quick Answer
**HA strategies**: (1) **Multiple instances** — min 3 replicas per service behind ALB. (2) **Circuit breaker** — Resilience4j stops calling failing services. (3) **Bulkhead** — isolate thread pools per downstream. (4) **Retry + timeout** — transient failure recovery. (5) **Health checks** — `/actuator/health`, LB removes unhealthy instances. (6) **Multi-AZ** — deploy across 2+ availability zones. (7) **Database** — Multi-AZ RDS with read replicas. (8) **Graceful degradation** — serve cached/stale data when dependency is down. (9) **Chaos engineering** — Netflix Chaos Monkey validates resilience.

### ⚡ Remember
> Replicas + LB | Circuit breaker + Bulkhead + Retry | Multi-AZ | Health checks | Graceful degradation | Chaos testing validates it all

---

<a id="q23"></a>
## Q23. How would you design a scalable transaction processing system?

### 📝 One-Liner
Event-sourced architecture with CQRS: capture every transaction as an immutable event, materialize read views separately, partition by account ID for parallelism, and use exactly-once processing semantics.

### 🔑 Quick Answer
**Core design**: (1) **Event sourcing** — every transaction is an immutable event (no UPDATE, only INSERT). (2) **CQRS** — separate write model (event store) from read model (materialized views). (3) **Partitioning** — Kafka topics partitioned by account ID ensures ordering per account. (4) **Exactly-once** — idempotent consumers + Kafka transactions. (5) **Scaling** — add partitions for throughput, add consumers for processing power. (6) **Reconciliation** — end-of-day batch verifies event store vs materialized views. *(Transaction processing mein event sourcing use karo — har transaction immutable event hai, CQRS se read/write alag, Kafka se partition by account)*

### ⚡ Remember
> Event sourcing (immutable events) | CQRS (separate read/write) | Partition by account ID | Kafka for ordering + throughput | Idempotent consumers | Reconciliation for consistency
