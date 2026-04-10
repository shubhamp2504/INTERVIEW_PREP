# 🌱 Spring Boot — REST, JPA & Internals Advanced Interview (Q1–Q28)

> **Source**: Spring Boot Part 2 — Advanced questions from interviews (continuing from Part 1)  
> **Coverage**: REST API design, Spring Data JPA deep dives, Spring internals, caching, Redis, resilience patterns  
> **Level**: 3+ YOE Java Backend Developer  
> **Key**: At this level, interviews focus on real-world problems, debugging skills, and internal working of frameworks — not just definitions  
> **Cross-references**: Several JPA questions overlap with existing database files — cross-linked below

---

## 🔹 Spring Boot REST

<a id="q1"></a>
## Q1. Design a Book Management REST API

### 📝 One-Liner
RESTful resource design: `/api/books` with standard CRUD verbs, proper status codes, pagination, validation, and DTO separation.

### 🔑 Quick Answer
**Endpoints**: `GET /api/books` (list with pagination), `GET /api/books/{id}` (details), `POST /api/books` (create, 201 Created), `PUT /api/books/{id}` (full update), `PATCH /api/books/{id}` (partial update), `DELETE /api/books/{id}` (204 No Content). Use DTOs to decouple API contract from entity. Add validation (`@Valid`), pagination (`Pageable`), exception handling (`@ControllerAdvice`). *(REST API design mein resource naming, HTTP verbs, status codes, pagination, validation — sab cover karo)*

### 💻 Code
```java
@RestController
@RequestMapping("/api/books")
public class BookController {
    private final BookService bookService;

    @GetMapping
    public Page<BookResponseDTO> getAllBooks(Pageable pageable) {
        return bookService.findAll(pageable);
    }

    @GetMapping("/{id}")
    public BookResponseDTO getBook(@PathVariable Long id) {
        return bookService.findById(id);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public BookResponseDTO createBook(@Valid @RequestBody BookCreateDTO dto) {
        return bookService.create(dto);
    }

    @PutMapping("/{id}")
    public BookResponseDTO updateBook(@PathVariable Long id, 
                                       @Valid @RequestBody BookUpdateDTO dto) {
        return bookService.update(id, dto);
    }

    @PatchMapping("/{id}")
    public BookResponseDTO patchBook(@PathVariable Long id, 
                                      @RequestBody Map<String, Object> updates) {
        return bookService.partialUpdate(id, updates);
    }

    @DeleteMapping("/{id}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void deleteBook(@PathVariable Long id) {
        bookService.delete(id);
    }
}

// DTO separation
record BookCreateDTO(@NotBlank String title, @NotBlank String author, 
                     @NotNull String isbn, @Positive double price) {}
record BookResponseDTO(Long id, String title, String author, 
                       String isbn, double price, LocalDateTime createdAt) {}
```

### ⚡ Remember
- Nouns for URLs (`/books`), HTTP verbs for actions (GET/POST/PUT/DELETE)
- Status codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 404 Not Found
- Separate DTOs for create, update, and response — never expose entities directly

---

<a id="q2"></a>
## Q2. How to send/receive data in XML instead of JSON in Spring Boot?

### 📝 One-Liner
Add `jackson-dataformat-xml` dependency and use `produces/consumes = MediaType.APPLICATION_XML_VALUE` or set `Accept`/`Content-Type` headers to `application/xml`.

### 🔑 Quick Answer
Spring Boot uses Jackson for serialization. By default, only JSON is configured. For XML: (1) Add `jackson-dataformat-xml` dependency, (2) Use `Accept: application/xml` header for response format or `@GetMapping(produces = "application/xml")`, (3) Use `Content-Type: application/xml` for request body. Spring auto-discovers the XML mapper via classpath. *(XML support chahiye toh jackson-dataformat-xml dependency add karo, phir Accept header se control karo — JSON ya XML)*

### 💻 Code
```xml
<!-- pom.xml -->
<dependency>
    <groupId>com.fasterxml.jackson.dataformat</groupId>
    <artifactId>jackson-dataformat-xml</artifactId>
</dependency>
```

```java
// Supports BOTH JSON and XML based on Accept header
@GetMapping(value = "/{id}", produces = {MediaType.APPLICATION_JSON_VALUE, 
                                          MediaType.APPLICATION_XML_VALUE})
public BookDTO getBook(@PathVariable Long id) {
    return bookService.findById(id);
}

// Force XML only
@GetMapping(value = "/xml/{id}", produces = MediaType.APPLICATION_XML_VALUE)
public BookDTO getBookXml(@PathVariable Long id) {
    return bookService.findById(id);
}

// Client sends XML: Content-Type: application/xml
@PostMapping(consumes = MediaType.APPLICATION_XML_VALUE)
public BookDTO createFromXml(@RequestBody BookDTO dto) { ... }
```

### ⚡ Remember
- Add `jackson-dataformat-xml` → auto-configured by Spring Boot
- `Accept` header controls response format; `Content-Type` controls request format
- `produces`/`consumes` in mapping annotation to restrict formats

---

<a id="q3"></a>
## Q3. Design a REST API to fetch large data from DB (scalable & high performance)?

### 📝 One-Liner
Use cursor-based pagination, database streaming, projection DTOs (not full entities), caching, and async processing — avoid `findAll()` loading everything into memory.

### 🔑 Quick Answer
**Never** `findAll()` for large data. Use: (1) **Cursor-based pagination** (`WHERE id > :lastId LIMIT :size` — better than offset for large datasets), (2) **Projection DTOs** (select only needed columns), (3) **Streaming** (`Stream<Entity>` with `@QueryHints(FETCH_SIZE)` for row-by-row processing), (4) **Caching** (Redis for frequently accessed data), (5) **Async/reactive** for non-blocking reads, (6) **DB read replicas** to offload read traffic. *(Bada data ek baar memory mein mat laao — pagination, projection, streaming, caching use karo)*

### 📖 How It Works (Detailed Explanation)

| Strategy | When | How |
|----------|------|-----|
| **Offset pagination** | Small-medium data, simple UI | `LIMIT 20 OFFSET 100` |
| **Cursor pagination** | Large data, infinite scroll | `WHERE id > 12345 LIMIT 20` (uses index) |
| **Keyset pagination** | Very large, sorted data | `WHERE (created_at, id) > (:last_ts, :last_id)` |
| **Streaming** | Export/batch processing | `Stream<T>` with flush per batch |
| **Projection DTO** | Always (reduce data transfer) | `SELECT new DTO(e.id, e.name)` |

### 💻 Code
```java
// ✅ Cursor-based pagination (better than offset for large data)
@Query("SELECT e FROM Employee e WHERE e.id > :lastId ORDER BY e.id ASC")
List<Employee> findNextPage(@Param("lastId") Long lastId, Pageable pageable);

// ✅ Streaming with @QueryHints (row-by-row, low memory)
@QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "50"))
@Query("SELECT e FROM Employee e")
Stream<Employee> streamAll();

// Usage: process in batches, flush periodically
@Transactional(readOnly = true)
public void exportLargeData(OutputStream out) {
    try (Stream<Employee> stream = employeeRepo.streamAll()) {
        stream.map(this::toDTO).forEach(dto -> writeToCsv(out, dto));
    }
}

// ✅ Projection DTO (fetch only needed columns)
@Query("SELECT new com.dto.EmployeeSummary(e.id, e.name, e.department) FROM Employee e")
Page<EmployeeSummary> findSummaries(Pageable pageable);
```

### ⚡ Remember
- Cursor pagination > offset pagination for large datasets (offset scans skipped rows)
- `Stream<T>` + `@Transactional(readOnly = true)` for batch processing
- Projection DTOs reduce network + memory — only fetch needed columns
- Index on cursor column (id, created_at) is critical

---

<a id="q4"></a>
## Q4. When should we use PATCH vs PUT?

### 📝 One-Liner
**PUT** replaces the entire resource (send all fields); **PATCH** partially updates (send only changed fields) — use PUT for full updates, PATCH for partial modifications.

### 🔑 Quick Answer
**PUT**: Idempotent full replacement — send the complete resource representation. Missing fields are set to null/default. **PATCH**: Partial update — send only the fields that changed. Server merges updates into existing resource. Use PUT when client has the full resource. Use PATCH when updating one or two fields (e.g., change status, update email). *(PUT = poora resource replace karo. PATCH = sirf changed fields bhejo — baaki untouched rehta hai)*

### 📖 How It Works (Detailed Explanation)

| Aspect | PUT | PATCH |
|--------|-----|-------|
| What's sent | Complete resource | Only changed fields |
| Missing fields | Set to null/default | Unchanged |
| Idempotent | ✅ Yes | ⚠️ Can be (not guaranteed) |
| Use case | Full resource update | Partial field update |
| Example | Update entire user profile | Change just the email |

```java
// PUT — full replacement (all fields required)
@PutMapping("/{id}")
public BookDTO updateBook(@PathVariable Long id, @Valid @RequestBody BookDTO dto) {
    Book book = bookRepo.findById(id).orElseThrow();
    book.setTitle(dto.title());      // all fields updated
    book.setAuthor(dto.author());
    book.setPrice(dto.price());
    book.setIsbn(dto.isbn());
    return toDTO(bookRepo.save(book));
}

// PATCH — partial update (only provided fields)
@PatchMapping("/{id}")
public BookDTO patchBook(@PathVariable Long id, @RequestBody Map<String, Object> updates) {
    Book book = bookRepo.findById(id).orElseThrow();
    updates.forEach((key, value) -> {
        switch (key) {
            case "title" -> book.setTitle((String) value);
            case "price" -> book.setPrice(((Number) value).doubleValue());
            // only update provided fields
        }
    });
    return toDTO(bookRepo.save(book));
}
```

### ⚡ Remember
- **PUT** = full resource; **PATCH** = partial update
- PUT is idempotent; PATCH should be designed to be idempotent too
- PATCH sends only changed fields → smaller payload → less bandwidth

---

<a id="q5"></a>
## Q5. Custom exception in Java and Global Exception Handling in Spring Boot

### 📝 One-Liner
Create custom exceptions extending `RuntimeException`, handle globally with `@RestControllerAdvice` + `@ExceptionHandler` to return consistent error responses across all APIs.

### 🔑 Quick Answer
**Custom exception**: `class ResourceNotFoundException extends RuntimeException { ... }` — throw when entity not found. **Global handling**: `@RestControllerAdvice` class with `@ExceptionHandler(ResourceNotFoundException.class)` methods that return structured error responses (status, message, timestamp, path). This centralizes error handling — no try-catch in every controller. *(Custom exception banao specific errors ke liye, phir @RestControllerAdvice se globally handle karo — har controller mein try-catch nahi chahiye)*

### 💻 Code
```java
// ✅ Custom exception
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Long id) {
        super(resource + " not found with ID: " + id);
    }
}

// ✅ Error response DTO
record ErrorResponse(int status, String message, String path, LocalDateTime timestamp) {}

// ✅ Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex, HttpServletRequest req) {
        return new ErrorResponse(404, ex.getMessage(), req.getRequestURI(), LocalDateTime.now());
    }
    
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex, 
                                           HttpServletRequest req) {
        String msg = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return new ErrorResponse(400, msg, req.getRequestURI(), LocalDateTime.now());
    }
    
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleGeneral(Exception ex, HttpServletRequest req) {
        return new ErrorResponse(500, "Internal server error", 
                                 req.getRequestURI(), LocalDateTime.now());
    }
}
```

### ⚡ Remember
- Custom exception = `extends RuntimeException` (unchecked)
- `@RestControllerAdvice` = global handler for all controllers
- Order matters: specific exceptions first, generic `Exception.class` last
- Never expose stack traces in production error responses

---

<a id="q6"></a>
## Q6. API is slow or downstream service fails — how to identify the issue?

### 📝 One-Liner
Check APM (Grafana/Datadog) → trace the slow request (distributed tracing) → check DB queries (slow query log) → check downstream health (circuit breaker metrics) → check thread/connection pools.

### 🔑 Quick Answer
**Step-by-step debugging**: (1) **APM dashboards** — identify which endpoint is slow (p95/p99 latency), (2) **Distributed tracing** (Zipkin/Jaeger) — find which span is slow (DB? downstream? serialization?), (3) **DB slow query log** — look for N+1, missing indices, full table scans, (4) **Thread dumps** — `jstack` to check if threads are blocked, (5) **Connection pool metrics** — HikariCP active/pending, HttpClient pool, (6) **Circuit breaker dashboard** — if downstream is failing, check Resilience4j metrics. *(Pehle APM se dekho kaunsa endpoint slow hai, phir trace follow karo — DB query? Downstream? Thread pool exhaustion? Step by step narrow down karo)*

### 📖 How It Works (Detailed Explanation)

```
Debugging Flowchart:
API slow? ─→ Which endpoint? (APM metrics)
    │
    ├─→ DB query slow?
    │     └─→ Slow query log, EXPLAIN ANALYZE
    │         └─→ Fix: add index, optimize query, use projection
    │
    ├─→ Downstream service slow?
    │     └─→ Distributed trace (Zipkin span duration)
    │         └─→ Fix: timeout + circuit breaker + fallback
    │
    ├─→ Thread pool exhausted?
    │     └─→ jstack, thread pool metrics
    │         └─→ Fix: increase pool, add async processing
    │
    ├─→ Connection pool exhausted?
    │     └─→ HikariCP metrics (active, pending)
    │         └─→ Fix: increase pool, fix connection leaks
    │
    └─→ GC pauses?
          └─→ GC logs (-Xlog:gc*)
              └─→ Fix: tune heap, switch GC algorithm
```

### 🗣️ Answering Approach
"I'd follow a systematic approach. First, check APM dashboards for which endpoint is slow and the latency percentiles. Then trace the specific request via Zipkin to find the slow span — is it a DB call, downstream service, or internal processing? For DB issues, I check the slow query log and run EXPLAIN ANALYZE. For downstream failures, I check circuit breaker metrics — are they in OPEN state? For resource issues, I check HikariCP pool metrics and thread pool utilization. I've found that 80% of production slowness comes from either N+1 queries or connection pool exhaustion."

### ⚡ Remember
- APM → which endpoint → distributed trace → which span → root cause
- Common causes: N+1 queries, connection pool exhaustion, downstream timeouts
- Tools: Grafana, Zipkin/Jaeger, slow query log, `jstack`, HikariCP metrics

---

<a id="q7"></a>
## Q7. Design Redis for a thread-safe environment

### 📝 One-Liner
Redis is single-threaded (inherently atomic for single commands), but for multi-step operations use Lua scripts (atomic execution), WATCH/MULTI/EXEC transactions, or Redisson distributed locks.

### 🔑 Quick Answer
**Redis is inherently thread-safe** for single commands — its single-threaded event loop processes one command at a time. But **multi-step operations** (read-modify-write) need: (1) **Lua scripts** — executed atomically on Redis server, (2) **MULTI/EXEC transactions** with `WATCH` for optimistic locking, (3) **Redisson** distributed locks for complex critical sections, (4) **RedisTemplate** with `execute()` for atomic operations in Spring. *(Redis single-threaded hai — ek command atomic hai. Lekin read-modify-write ke liye Lua script ya distributed lock chahiye)*

### 💻 Code
```java
// ✅ Spring Boot RedisTemplate — atomic operations
@Service
public class RedisCacheService {
    @Autowired private StringRedisTemplate redis;
    
    // Atomic increment (single command — always thread-safe)
    public Long incrementCounter(String key) {
        return redis.opsForValue().increment(key);
    }
    
    // ✅ Lua script for atomic read-modify-write
    public Boolean atomicCompareAndSet(String key, String expected, String newValue) {
        String script = """
            if redis.call('get', KEYS[1]) == ARGV[1] then
                redis.call('set', KEYS[1], ARGV[2])
                return 1
            end
            return 0
            """;
        RedisScript<Boolean> redisScript = RedisScript.of(script, Boolean.class);
        return redis.execute(redisScript, List.of(key), expected, newValue);
    }
    
    // ✅ Distributed lock with Redisson
    // RLock lock = redissonClient.getLock("order:" + orderId);
    // lock.lock(10, TimeUnit.SECONDS);
    // try { processOrder(orderId); }
    // finally { lock.unlock(); }
}
```

### ⚡ Remember
- Redis = single-threaded → single commands are atomic
- Multi-step ops: Lua scripts (atomic), WATCH/MULTI (optimistic), Redisson locks (distributed)
- `SETNX` (SET if Not eXists) for simple lock patterns
- Spring: `RedisTemplate.execute()` for pipelined/scripted ops

---

<a id="q8"></a>
## Q8. Two instances of a microservice accessing the same Redis data — how does it work?

### 📝 One-Liner
Redis is a centralized shared store — both instances connect to the same Redis server, but concurrent read-modify-write operations need distributed locks (Redisson) or Lua scripts to prevent race conditions.

### 🔑 Quick Answer
When M1-Instance1 and M1-Instance2 both access the same Redis key: **Reads are safe** (Redis serves latest value). **Writes that are single commands** (`SET`, `INCR`) are atomic. **Read-modify-write** patterns (read balance → calculate → write new balance) cause race conditions — both instances read same value, both write, one update is lost. **Solutions**: (1) **Distributed lock** (Redisson/Redlock) — only one instance operates at a time, (2) **Lua scripts** — atomic server-side execution, (3) **WATCH+MULTI** — optimistic retry if key changed. *(Do instances same Redis key pe kaam kar rahe hain — read safe hai, single command safe hai, lekin read-modify-write ke liye lock chahiye)*

### 📖 How It Works (Detailed Explanation)

```
Race Condition (without lock):
M1-1: GET balance → 100
M1-2: GET balance → 100
M1-1: SET balance → 100 + 50 = 150
M1-2: SET balance → 100 - 30 = 70   ← WRONG! Should be 120

With Distributed Lock:
M1-1: LOCK("balance-lock") → acquired
M1-1: GET balance → 100 → SET balance → 150
M1-1: UNLOCK
M1-2: LOCK("balance-lock") → acquired
M1-2: GET balance → 150 → SET balance → 120  ✅ CORRECT
```

### ⚡ Remember
- Multiple instances → same Redis server = shared state
- Single Redis commands are atomic (no lock needed)
- Read-modify-write = race condition → use distributed lock or Lua
- Redisson `RLock` = recommended distributed lock for Java

---

<a id="q9"></a>
## Q9. First-level cache vs Second-level cache

> **Cross-reference**: Detailed coverage in [database/04-hibernate-cache-states.md](../database/04-hibernate-cache-states.md)

### 📝 One-Liner
**L1 cache** = per-EntityManager/Session (automatic, within a transaction). **L2 cache** = shared across sessions (configurable, across transactions, needs explicit setup like Ehcache/Redis).

### 🔑 Quick Answer

| Feature | L1 Cache (First Level) | L2 Cache (Second Level) |
|---------|----------------------|------------------------|
| Scope | Per EntityManager/Session | Shared (SessionFactory-wide) |
| Default | ✅ Always ON | ❌ OFF (must configure) |
| Lifetime | Current transaction | Application lifetime |
| Providers | Built into Hibernate | Ehcache, Redis, Hazelcast |
| Eviction | Auto (session closes) | Configurable (TTL, size) |
| Use case | Avoid repeated DB reads in same tx | Frequently accessed, rarely changed data |

### ⚡ Remember
- L1 = automatic, per-session, within transaction
- L2 = manual setup, shared across sessions, Ehcache/Redis
- L1 prevents re-reading same entity in one transaction
- L2 prevents re-querying across different requests/sessions

---

<a id="q10"></a>
## Q10. Design an API that retries 3 times and then returns a fallback response

### 📝 One-Liner
Use **Resilience4j** `@Retry` with maxAttempts=3 + `@CircuitBreaker` with fallback — retries transient failures, then returns a cached/default response on persistent failure.

### 🔑 Quick Answer
Use Resilience4j (preferred in Spring Boot) with `@Retry(name = "myService", fallbackMethod = "fallback")` configured for maxAttempts = 3 with exponential backoff. On 3rd failure, the fallback method returns a cached/default response. Combine with `@CircuitBreaker` to stop retries when the downstream is definitely down. *(3 baar retry karo exponential backoff ke saath, phir bhi fail ho toh fallback response do — cached data ya default message)*

### 💻 Code
```java
// application.yml
// resilience4j:
//   retry:
//     instances:
//       paymentService:
//         maxAttempts: 3
//         waitDuration: 1s
//         exponentialBackoffMultiplier: 2
//         retryExceptions: [java.io.IOException, java.util.concurrent.TimeoutException]

@Service
public class PaymentService {
    
    @Retry(name = "paymentService", fallbackMethod = "paymentFallback")
    @CircuitBreaker(name = "paymentService", fallbackMethod = "paymentFallback")
    public PaymentResponse processPayment(PaymentRequest req) {
        // calls external payment gateway — may fail
        return paymentGateway.charge(req);
    }
    
    // Fallback after 3 retries
    private PaymentResponse paymentFallback(PaymentRequest req, Throwable t) {
        log.warn("Payment failed after retries: {}", t.getMessage());
        return new PaymentResponse("PENDING", "Payment queued for retry", null);
        // Store in DB for async retry via scheduled job
    }
}
```

### ⚡ Remember
- Resilience4j `@Retry` = maxAttempts + exponential backoff
- Fallback = cached/default response after all retries fail
- Combine with `@CircuitBreaker` to fast-fail when service is down
- Retry only on transient exceptions (IOException, TimeoutException), not business errors (400)

---

<a id="q11"></a>
## Q11. Can you implement an LRU Cache?

### 📝 One-Liner
Use `LinkedHashMap(capacity, loadFactor, accessOrder=true)` with `removeEldestEntry()` override, or a HashMap + doubly-linked list for O(1) get/put in interviews.

### 🔑 Quick Answer
**LRU (Least Recently Used)** evicts the least recently accessed entry when capacity is full. Two approaches: (1) **`LinkedHashMap`** with `accessOrder=true` + override `removeEldestEntry()` — simplest. (2) **HashMap + Doubly Linked List** — interview approach with O(1) get and put. *(LRU = jab cache full ho jaye toh sabse purana (least recently used) entry nikaal do nayi ke liye jagah banao)*

### 💻 Code
```java
// ✅ Method 1: LinkedHashMap (production-simple)
public class LRUCache<K, V> extends LinkedHashMap<K, V> {
    private final int capacity;
    
    public LRUCache(int capacity) {
        super(capacity, 0.75f, true); // accessOrder = true (LRU behavior)
        this.capacity = capacity;
    }
    
    @Override
    protected boolean removeEldestEntry(Map.Entry<K, V> eldest) {
        return size() > capacity; // evict when exceeds capacity
    }
}

// ✅ Method 2: HashMap + Doubly Linked List (interview approach)
public class LRUCacheManual<K, V> {
    private final int capacity;
    private final Map<K, Node<K, V>> map = new HashMap<>();
    private final Node<K, V> head = new Node<>(null, null); // dummy head
    private final Node<K, V> tail = new Node<>(null, null); // dummy tail
    
    static class Node<K, V> {
        K key; V value; Node<K, V> prev, next;
        Node(K key, V value) { this.key = key; this.value = value; }
    }
    
    public LRUCacheManual(int capacity) {
        this.capacity = capacity;
        head.next = tail;
        tail.prev = head;
    }
    
    public V get(K key) {                    // O(1)
        Node<K, V> node = map.get(key);
        if (node == null) return null;
        moveToHead(node);                     // mark as recently used
        return node.value;
    }
    
    public void put(K key, V value) {        // O(1)
        Node<K, V> node = map.get(key);
        if (node != null) {
            node.value = value;
            moveToHead(node);
        } else {
            node = new Node<>(key, value);
            map.put(key, node);
            addToHead(node);
            if (map.size() > capacity) {
                Node<K, V> lru = removeTail(); // evict LRU
                map.remove(lru.key);
            }
        }
    }
    
    private void moveToHead(Node<K, V> node) { removeNode(node); addToHead(node); }
    private void addToHead(Node<K, V> node) {
        node.prev = head; node.next = head.next;
        head.next.prev = node; head.next = node;
    }
    private void removeNode(Node<K, V> node) {
        node.prev.next = node.next; node.next.prev = node.prev;
    }
    private Node<K, V> removeTail() {
        Node<K, V> lru = tail.prev; removeNode(lru); return lru;
    }
}
```

### ⚡ Remember
- `LinkedHashMap(cap, 0.75f, true)` + `removeEldestEntry()` = simplest LRU
- HashMap + Doubly Linked List = interview-expected O(1) implementation
- Head = most recently used; Tail = least recently used (eviction candidate)
- Production: Use Caffeine cache with `maximumSize()` — handles LRU internally

---

## 🔹 Spring Data JPA

<a id="q12"></a>
## Q12. Difference between JpaRepository, CrudRepository, and PagingAndSortingRepository

### 📝 One-Liner
`CrudRepository` = basic CRUD. `PagingAndSortingRepository` = CRUD + pagination + sorting. `JpaRepository` = everything + batch operations + flush + JPA-specific features.

### 🔑 Quick Answer

| Interface | Extends | Key Methods |
|-----------|---------|-------------|
| `CrudRepository<T, ID>` | `Repository` | `save()`, `findById()`, `findAll()`, `delete()`, `count()` |
| `PagingAndSortingRepository<T, ID>` | `CrudRepository` | + `findAll(Pageable)`, `findAll(Sort)` |
| `JpaRepository<T, ID>` | `PagingAndSortingRepository` | + `saveAll()`, `flush()`, `saveAndFlush()`, `deleteInBatch()`, `getById()` |

**Use `JpaRepository`** in Spring Boot — it gives you everything. *(Almost always JpaRepository use karo — isme sab milta hai: CRUD + pagination + batch + flush)*

### ⚡ Remember
- `CrudRepository` ⊂ `PagingAndSortingRepository` ⊂ `JpaRepository`
- `JpaRepository` adds: `saveAndFlush()`, `deleteInBatch()`, `getById()` (lazy ref)
- Default choice: `JpaRepository` — no reason to use lesser interfaces

---

<a id="q13"></a>
## Q13. What is the N+1 problem? How to solve it?

> **Cross-reference**: Detailed coverage in [database/01-jpa-sql-transactions.md Q11](../database/01-jpa-sql-transactions.md#q11) and [database/05-hibernate-production-mistakes.md Q14](../database/05-hibernate-production-mistakes.md#q14)

### 📝 One-Liner
N+1 = 1 query for parent + N queries for each child relationship. Fix with `JOIN FETCH`, `@EntityGraph`, or `@BatchSize`.

### 🔑 Quick Answer
Loading 100 orders → 1 query for orders + 100 queries for each order's items = **101 queries** instead of 1-2. **Fixes**: `@Query("SELECT o FROM Order o JOIN FETCH o.items")`, `@EntityGraph(attributePaths = "items")`, or `@BatchSize(size = 50)` on the collection.

### ⚡ Remember
- N+1 = lazy loading triggers separate query per child
- `JOIN FETCH` = single query with JOIN (best for single collection)
- `@EntityGraph` = declarative fetch graph
- `@BatchSize` = Hibernate batches lazy loads (N/batch queries)

---

<a id="q14"></a>
## Q14. What happens if we don't use @Transactional?

### 📝 One-Liner
Without `@Transactional`, each repository call runs in its own auto-committed transaction — partial failures can't be rolled back, lazy loading fails outside the persistence context, and dirty checking doesn't work.

### 🔑 Quick Answer
Without `@Transactional`: (1) **No rollback** — if `save(order)` succeeds but `save(payment)` fails, order is persisted (inconsistent state), (2) **LazyInitializationException** — persistence context closes after each repo call, accessing lazy collections fails, (3) **No dirty checking** — modified entities won't auto-save on commit, (4) **Each repo call** = separate DB connection from pool (inefficient). *(Without @Transactional = har repo call ka alag transaction — ek fail ho toh doosra rollback nahi hoga, inconsistency aayegi)*

### 💻 Code
```java
// ❌ Without @Transactional — DANGEROUS
public void placeOrder(OrderDTO dto) {
    orderRepo.save(order);      // commits immediately ✅
    paymentRepo.save(payment);  // fails → order stuck without payment ❌
    // No rollback for order!
}

// ✅ With @Transactional — all-or-nothing
@Transactional
public void placeOrder(OrderDTO dto) {
    orderRepo.save(order);      // pending (not committed yet)
    paymentRepo.save(payment);  // fails → ENTIRE transaction rolls back
    // Both order and payment rolled back ✅
}
```

### ⚡ Remember
- Without `@Transactional` = auto-commit per repository call
- Partial failures = inconsistent data (no rollback)
- Lazy loading needs open persistence context (needs `@Transactional`)
- Always use `@Transactional` for multi-step business operations

---

<a id="q15"></a>
## Q15. What is Dirty Checking in Hibernate?

### 📝 One-Liner
Dirty checking = Hibernate automatically detects changes to managed entities and generates UPDATE SQL at flush/commit — no explicit `save()` needed for already-managed entities.

### 🔑 Quick Answer
When you load an entity inside a `@Transactional` method and modify its fields, Hibernate snapshots the original state at load time. At flush/commit, it compares current state with the snapshot — if fields changed (dirty), it auto-generates an `UPDATE` query. You don't need to call `save()` again. This only works for **managed entities** (loaded within the current persistence context). *(Entity load karo, fields change karo — Hibernate khud UPDATE query generate karega commit pe. save() call karne ki zarurat nahi)*

### 💻 Code
```java
@Transactional
public void updateEmployeeSalary(Long id, double newSalary) {
    Employee emp = employeeRepo.findById(id).orElseThrow();
    emp.setSalary(newSalary); // modify managed entity
    // NO save() needed! Dirty checking auto-generates UPDATE at commit
}
// Hibernate: UPDATE employee SET salary = ? WHERE id = ?
```

### ⚡ Remember
- Dirty checking = compare snapshot at load vs current state at flush
- Only for **managed** entities (loaded in current transaction)
- No explicit `save()` needed — Hibernate auto-UPDATEs
- Detached entities (outside transaction) need `merge()` to re-attach

---

<a id="q16"></a>
## Q16. Difference between CascadeType.ALL and orphanRemoval = true

### 📝 One-Liner
`CascadeType.ALL` propagates ALL operations (persist, merge, remove) from parent to children. `orphanRemoval = true` automatically deletes children removed from the parent's collection — even without explicit delete.

### 🔑 Quick Answer
**CascadeType.ALL**: When you save/delete a parent, the operation cascades to all children. If you remove a child from the collection but save the parent, the child is NOT deleted — it becomes an orphan. **orphanRemoval=true**: If a child is removed from the parent's collection, Hibernate automatically deletes it from DB — it considers the detached child an "orphan." *(CascadeType.ALL = parent ke saath operation propagate karo. orphanRemoval = collection se nikala toh DB se bhi delete kar do)*

### 💻 Code
```java
@Entity
public class Order {
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, mappedBy = "order")
    private List<OrderItem> items = new ArrayList<>();
    
    public void removeItem(OrderItem item) {
        items.remove(item);    // orphanRemoval=true → Hibernate DELETEs from DB
        item.setOrder(null);
    }
}

// Without orphanRemoval: item removed from list but stays in DB (orphan)
// With orphanRemoval: item removed from list → Hibernate DELETEs it
```

### ⚡ Remember
- `CascadeType.ALL` = propagate persist/merge/remove/refresh/detach
- `orphanRemoval = true` = delete from DB when removed from parent's collection
- Use both together for parent-child ownership (`@OneToMany` with composition)
- Don't use `orphanRemoval` for `@ManyToMany` — shared ownership

---

<a id="q17"></a>
## Q17. Why does LazyInitializationException occur?

### 📝 One-Liner
Occurs when you access a lazy-loaded collection/association AFTER the Hibernate session (persistence context) has closed — typically outside a `@Transactional` method.

### 🔑 Quick Answer
Lazy loading means Hibernate doesn't load the association until you access it. If you load an entity in a `@Transactional` method, return it to the controller, and THEN access the lazy collection — the session is already closed → `LazyInitializationException`. **Fixes**: (1) `JOIN FETCH` in query, (2) `@EntityGraph`, (3) DTO projection (don't return entities), (4) `@Transactional` on the service method that accesses the collection, (5) `Hibernate.initialize()` before session closes. *(Session band ho gayi lekin lazy collection abhi load nahi hua — access karoge toh exception aayega)*

### 💻 Code
```java
// ❌ Causes LazyInitializationException
@GetMapping("/orders/{id}")
public Order getOrder(@PathVariable Long id) {
    Order order = orderService.findById(id); // session closes after service method
    order.getItems().size(); // ← BOOM! Session already closed, items not loaded
    return order;
}

// ✅ Fix 1: JOIN FETCH
@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id = :id")
Optional<Order> findByIdWithItems(@Param("id") Long id);

// ✅ Fix 2: DTO projection (best practice)
@Transactional(readOnly = true)
public OrderDTO findById(Long id) {
    Order order = repo.findById(id).orElseThrow();
    return new OrderDTO(order.getId(), order.getStatus(),
        order.getItems().stream().map(this::toItemDTO).toList()); // access within tx
}
```

### ⚡ Remember
- Cause: access lazy collection outside open session/transaction
- Best fix: DTO projection — don't return entities from service layer
- Quick fix: `JOIN FETCH` or `@EntityGraph` in repository query
- Anti-pattern: `spring.jpa.open-in-view=true` (keeps session open in controller — bad practice)

---

<a id="q18"></a>
## Q18. Difference between merge() and persist()

### 📝 One-Liner
`persist()` makes a NEW (transient) entity managed — INSERT. `merge()` copies state of a DETACHED entity into a managed copy — may INSERT or UPDATE depending on whether the entity exists.

### 🔑 Quick Answer
**`persist(entity)`**: Entity must be NEW (transient). Makes it managed. Generates INSERT. The original object becomes managed. **`merge(entity)`**: Entity can be detached or new. Returns a MANAGED COPY (the original stays detached). If entity has an ID and exists → UPDATE. If no ID or doesn't exist → INSERT. Key difference: `persist()` manages the original object; `merge()` returns a new managed copy. *(persist = naya entity save karo (INSERT). merge = detached entity wapas laao (UPDATE ya INSERT) — merge ek copy return karta hai, original detached rehta hai)*

### 💻 Code
```java
// persist() — for NEW entities
Employee emp = new Employee("Shubham"); // transient
entityManager.persist(emp);             // emp is now MANAGED, INSERT queued
emp.setName("Updated");                 // dirty checking works on emp

// merge() — for DETACHED entities
Employee detached = getFromSomewhere(); // detached (from another session)
Employee managed = entityManager.merge(detached); // returns MANAGED copy
// detached is STILL detached!
// managed is the one tracked by persistence context
managed.setName("Updated"); // dirty checking works on managed, NOT detached
```

### ⚡ Remember
- `persist()`: transient → managed (original object). Always INSERT
- `merge()`: detached → returns new managed copy. INSERT or UPDATE
- `save()` in Spring Data = `persist()` if new, `merge()` if existing (checks `@Id`)

---

<a id="q19"></a>
## Q19. You modify an entity and call save() but DB is not updated — why?

### 📝 One-Liner
Possible causes: not within `@Transactional` (no flush), `@Transactional(readOnly = true)` suppressing flushes, detached entity, exception swallowed before commit, or caching returning stale data.

### 🔑 Quick Answer
Common reasons: (1) **Missing `@Transactional`** — changes not flushed to DB, (2) **`readOnly = true`** — some JPA providers skip flush for read-only transactions, (3) **Entity is detached** — `save()` calls `merge()` but you're using the old reference, (4) **Exception before commit** — transaction rolled back silently, (5) **Second-level cache** — showing cached (stale) data, DB actually updated. *(Entity modify kiya, save() bhi call kiya lekin DB update nahi hua — check karo: @Transactional hai? readOnly toh nahi? exception toh nahi aayi silently?)*

### 💻 Code
```java
// ❌ Problem 1: No @Transactional
public void updateSalary(Long id, double salary) {
    Employee emp = repo.findById(id).orElseThrow();
    emp.setSalary(salary);
    repo.save(emp); // may not flush without transaction context!
}

// ❌ Problem 2: readOnly = true
@Transactional(readOnly = true) // suppresses dirty checking + flush!
public void updateSalary(Long id, double salary) {
    Employee emp = repo.findById(id).orElseThrow();
    emp.setSalary(salary); // dirty checking disabled → no UPDATE
}

// ✅ Fix: proper @Transactional
@Transactional
public void updateSalary(Long id, double salary) {
    Employee emp = repo.findById(id).orElseThrow();
    emp.setSalary(salary);
    // save() not even needed — dirty checking auto-updates at commit
}
```

### ⚡ Remember
- Check `@Transactional` presence and it's not `readOnly = true`
- Dirty checking only works on managed entities within a transaction
- If using `merge()`, use the RETURNED object, not the original
- Enable SQL logging (`spring.jpa.show-sql=true`) to verify queries

---

<a id="q20"></a>
## Q20. Optimistic vs Pessimistic Locking

> **Cross-reference**: Detailed coverage in [database/01-jpa-sql-transactions.md Q12](../database/01-jpa-sql-transactions.md#q12)

### 📝 One-Liner
**Optimistic** = `@Version` field, no DB lock, fails on concurrent update (retry). **Pessimistic** = `SELECT FOR UPDATE`, DB-level lock, blocks other transactions until released.

### 🔑 Quick Answer

| Aspect | Optimistic | Pessimistic |
|--------|-----------|-------------|
| Mechanism | `@Version` field (auto-incremented) | DB lock (`SELECT FOR UPDATE`) |
| Conflict detection | At commit time (`OptimisticLockException`) | Prevention (blocks concurrent access) |
| Performance | Better for low-contention (no lock overhead) | Better for high-contention (avoids retries) |
| Use case | Shopping cart, profile updates | Bank transfers, seat booking |

### ⚡ Remember
- Optimistic = version check at commit → `OptimisticLockException` on conflict → retry
- Pessimistic = DB lock at read → blocks others → no conflict possible
- Low contention → optimistic; High contention → pessimistic

---

## 🔹 JPA Advanced

<a id="q21"></a>
## Q21. How to define a composite key in JPA?

### 📝 One-Liner
Two approaches: **`@EmbeddedId`** (embed a composite key class) or **`@IdClass`** (declare multiple `@Id` fields with a separate ID class) — both require `Serializable` + `equals()/hashCode()`.

### 🔑 Quick Answer
When a table has a multi-column primary key (e.g., `student_id + course_id`): **`@EmbeddedId`** — create a separate class annotated `@Embeddable`, use it as single `@EmbeddedId` field. **`@IdClass`** — create a plain ID class, annotate entity with `@IdClass`, mark individual fields with `@Id`. Both ID classes must implement `Serializable` and override `equals()` + `hashCode()`. *(Composite key = multiple columns milke ek primary key banate hain — @EmbeddedId ya @IdClass se define karo)*

### 💻 Code
```java
// ✅ Method 1: @EmbeddedId
@Embeddable
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals() + hashCode() REQUIRED
}

@Entity
public class Enrollment {
    @EmbeddedId
    private EnrollmentId id;
    private LocalDate enrolledDate;
    private String grade;
}

// ✅ Method 2: @IdClass
public class EnrollmentId implements Serializable {
    private Long studentId;
    private Long courseId;
    // equals() + hashCode() REQUIRED
}

@Entity
@IdClass(EnrollmentId.class)
public class Enrollment {
    @Id private Long studentId;
    @Id private Long courseId;
    private LocalDate enrolledDate;
}
```

### ⚡ Remember
- `@EmbeddedId` = single composite key field; `@IdClass` = multiple `@Id` fields
- Both: ID class must implement `Serializable` + `equals()` + `hashCode()`
- Prefer `@EmbeddedId` — cleaner, object-oriented approach

---

<a id="q22"></a>
## Q22. FetchType EAGER vs LAZY

> **Cross-reference**: Related to [database/01-jpa-sql-transactions.md Q11](../database/01-jpa-sql-transactions.md#q11)

### 📝 One-Liner
**LAZY** = load association only when accessed (default for `@OneToMany`/`@ManyToMany`). **EAGER** = load association immediately with the parent (default for `@ManyToOne`/`@OneToOne`).

### 🔑 Quick Answer
LAZY = proxy/placeholder until `.getItems()` is called. EAGER = JOIN/subquery loads association with initial query. **Best practice**: Always use LAZY, fetch eagerly only when needed via `JOIN FETCH` or `@EntityGraph`. EAGER causes N+1 in collections and loads unnecessary data.

### ⚡ Remember
- `@OneToMany`/`@ManyToMany` default = LAZY ✅
- `@ManyToOne`/`@OneToOne` default = EAGER ⚠️ (change to LAZY)
- Always prefer LAZY + fetch when needed via `JOIN FETCH`

---

## 🔹 Spring Internals

<a id="q23"></a>
## Q23. @Component vs @Bean

### 📝 One-Liner
`@Component` = class-level annotation, Spring auto-detects via classpath scanning. `@Bean` = method-level annotation in `@Configuration` class, gives you manual control over bean creation including third-party classes.

### 🔑 Quick Answer
**`@Component`** (+ `@Service`, `@Repository`, `@Controller`): Annotate YOUR class → Spring scans and creates bean automatically. **`@Bean`**: Annotate a METHOD in `@Configuration` → you manually instantiate and return the bean. Use `@Bean` for: third-party classes (you can't add `@Component` to library code), complex initialization, conditional bean creation. *(Apni class hai → @Component. Library ki class hai ya complex setup chahiye → @Bean method likho)*

### 💻 Code
```java
// @Component — auto-detected by classpath scanning
@Service // same as @Component (semantic variant)
public class OrderService { ... }

// @Bean — manual bean creation in @Configuration
@Configuration
public class AppConfig {
    @Bean
    public RestTemplate restTemplate() {
        return new RestTemplateBuilder()
            .setConnectTimeout(Duration.ofSeconds(5))
            .setReadTimeout(Duration.ofSeconds(10))
            .build();
    }
    
    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }
}
```

### ⚡ Remember
- `@Component` = your class, auto-scan; `@Bean` = manual creation, config class
- Use `@Bean` for third-party libraries, complex setup, conditional beans
- `@Service`/`@Repository`/`@Controller` are specialized `@Component`

---

<a id="q24"></a>
## Q24. Does this query work? JOIN FETCH + Pageable

```java
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);
```

### 📝 One-Liner
**It works BUT with a critical warning** — Hibernate fetches ALL results into memory first, then applies pagination in-memory (not in SQL) — defeating the purpose of pagination for large datasets.

### 🔑 Quick Answer
Hibernate can't apply `LIMIT/OFFSET` in SQL alongside `JOIN FETCH` for collections (the row count changes due to the join). So it: (1) Executes the query WITHOUT limit, (2) Loads ALL results into memory, (3) Applies pagination in Java. You'll see the log warning: `HHH000104: firstResult/maxResults specified with collection fetch; applying in memory!`. **Fix**: Two-query approach — first fetch paginated parent IDs, then fetch parents with items. *(JOIN FETCH + Pageable = Hibernate saara data memory mein laake phir pagination karta hai — large data mein OutOfMemory ho sakta hai!)*

### 💻 Code
```java
// ❌ WARNING: In-memory pagination
@Query("SELECT o FROM Order o JOIN FETCH o.items")
Page<Order> findAllWithItems(Pageable pageable);
// HHH000104 warning — loads ALL into memory

// ✅ Fix: Two-query approach
@Query("SELECT o.id FROM Order o")
Page<Long> findOrderIds(Pageable pageable); // paginated IDs

@Query("SELECT o FROM Order o JOIN FETCH o.items WHERE o.id IN :ids")
List<Order> findByIdsWithItems(@Param("ids") List<Long> ids);

// Service: combine
public Page<Order> getOrders(Pageable pageable) {
    Page<Long> idPage = repo.findOrderIds(pageable);
    List<Order> orders = repo.findByIdsWithItems(idPage.getContent());
    return new PageImpl<>(orders, pageable, idPage.getTotalElements());
}
```

### ⚡ Remember
- `JOIN FETCH` + `Pageable` = **in-memory pagination** (dangerous for large data)
- Fix: Two queries — paginate IDs first, then JOIN FETCH for that page
- Watch for `HHH000104` warning in logs

---

<a id="q25"></a>
## Q25. Why does @Transactional not work on private methods?

### 📝 One-Liner
Spring's `@Transactional` works via AOP proxy — proxies can only intercept **public** methods called from **outside** the class. Private methods are invisible to the proxy.

### 🔑 Quick Answer
Spring creates a proxy (JDK dynamic proxy or CGLIB) around your bean. When an external caller invokes a `@Transactional` method, the call goes through the proxy which starts the transaction. **Private methods** can't be proxied — they're not part of the public interface. **Internal calls** (`this.methodB()`) also bypass the proxy because `this` refers to the actual object, not the proxy. *(Spring proxy sirf public methods ko intercept kar sakta hai — private method proxy ke through nahi jaata, isliye @Transactional kaam nahi karta)*

### 📖 How It Works (Detailed Explanation)

```
External call (works):
Caller → Proxy → @Transactional method → real method
         ↑ proxy starts tx

Internal call (BROKEN):
@Service class {
    public void methodA() {
        this.methodB();  // ← bypasses proxy! No transaction!
    }
    
    @Transactional
    public void methodB() { ... } // @Transactional ignored
}
```

### 💻 Code
```java
// ❌ @Transactional on private — SILENTLY IGNORED
@Transactional
private void processPayment() { ... } // No transaction!

// ❌ Internal call — @Transactional BYPASSED
@Service
public class OrderService {
    public void placeOrder() {
        processPayment(); // calls this.processPayment() → bypasses proxy
    }
    
    @Transactional
    public void processPayment() { ... } // Transaction NOT started!
}

// ✅ Fix 1: Inject self (proxy)
@Service
public class OrderService {
    @Lazy @Autowired private OrderService self; // inject proxy
    
    public void placeOrder() {
        self.processPayment(); // goes through proxy → tx works!
    }
}

// ✅ Fix 2: Extract to separate service
@Service
public class PaymentService {
    @Transactional
    public void processPayment() { ... } // external call → proxy works!
}
```

### ⚡ Remember
- `@Transactional` = proxy-based AOP → only public + external calls
- Private methods → silently ignored
- Internal calls (`this.method()`) → bypass proxy
- Fix: inject self or extract to another service

---

<a id="q26"></a>
## Q26. How does @Query work internally? What is SimpleJpaRepository?

### 📝 One-Liner
`@Query` annotations are parsed at startup — Spring creates `NamedQuery` instances from JPQL/SQL, and `SimpleJpaRepository` is the default implementation class behind all `JpaRepository` interfaces.

### 🔑 Quick Answer
**@Query**: At application startup, Spring Data scans repository interfaces and creates proxy implementations. For `@Query` methods, it parses the JPQL/SQL string, validates it against the entity model, and creates a `NamedQuery` or `NativeQuery` object. At runtime, the proxy intercepts the method call, binds parameters, and executes the pre-compiled query. **SimpleJpaRepository**: The concrete class that implements `JpaRepository`. All standard methods (`save()`, `findById()`, `findAll()`, `delete()`) are implemented here using `EntityManager` from JPA. *(Spring startup pe @Query parse karta hai aur validate karta hai — runtime pe bas parameters bind karke execute karta hai. SimpleJpaRepository = JpaRepository ka actual implementation class)*

### 📖 How It Works (Detailed Explanation)

```
Your code:   public interface BookRepo extends JpaRepository<Book, Long> { ... }
                                    │
                         Spring Data creates:
                                    │
                    JdkDynamicProxy(BookRepo) → delegates to:
                      │
              SimpleJpaRepository<Book, Long> ← actual implementation
              Uses: EntityManager.find(), .persist(), .merge(), .createQuery()
```

**@Query processing flow:**
1. **Startup**: Spring scans `@Query` annotation → parses JPQL → validates against entity metamodel
2. **Validation**: If JPQL has typo (`Emplyoee` instead of `Employee`), startup FAILS
3. **Runtime**: Method called → proxy intercepts → binds `:params` → executes via `EntityManager.createQuery()`

### ⚡ Remember
- `SimpleJpaRepository` = default implementation of `JpaRepository` (uses `EntityManager`)
- `@Query` JPQL validated at startup — typos caught early
- Proxy pattern: Spring creates a dynamic proxy for your interface
- Derived queries (`findByName`) → Spring generates JPQL from method name

---

<a id="q27"></a>
## Q27. save() vs saveAndFlush()

> **Cross-reference**: Detailed coverage in [database/03-jpa-persistence-ops.md Q1](../database/03-jpa-persistence-ops.md#q1)

### 📝 One-Liner
`save()` queues the entity for persistence but doesn't immediately write to DB (waits for transaction commit/flush). `saveAndFlush()` immediately writes to DB (executes SQL now).

### 🔑 Quick Answer
**`save()`**: Marks entity as managed in persistence context — actual SQL INSERT/UPDATE happens at flush time (usually transaction commit). **`saveAndFlush()`**: Calls `save()` + immediately flushes → SQL executes NOW. Use `saveAndFlush()` when you need the DB-generated ID immediately, or want to catch constraint violations before commit.

### ⚡ Remember
- `save()` = deferred (at commit/flush); `saveAndFlush()` = immediate
- Use `saveAndFlush()` when you need DB-generated ID right away
- Both: entity becomes managed in persistence context

---

<a id="q28"></a>
## Q28. How to validate REST API request data?

### 📝 One-Liner
Use Jakarta Bean Validation (`@Valid` + `@NotBlank`, `@NotNull`, `@Size`, `@Email`, `@Pattern`, etc.) on DTO fields, with `@Valid` on the controller parameter to trigger validation.

### 🔑 Quick Answer
Add `spring-boot-starter-validation` dependency. Annotate DTO fields with constraints (`@NotBlank`, `@Size(max=100)`, `@Email`, `@Positive`). Add `@Valid` on the `@RequestBody` parameter — Spring auto-validates before the method executes. Handle validation errors in `@RestControllerAdvice` by catching `MethodArgumentNotValidException`. *(DTO pe validation annotations lagao, controller mein @Valid add karo — Spring khud validate karega request aane pe)*

### 💻 Code
```java
// DTO with validation
record CreateBookRequest(
    @NotBlank(message = "Title is required") String title,
    @NotBlank(message = "Author is required") String author,
    @Size(min = 10, max = 13, message = "ISBN must be 10-13 chars") String isbn,
    @Positive(message = "Price must be positive") double price,
    @Email(message = "Invalid email") String authorEmail
) {}

// Controller — @Valid triggers validation
@PostMapping
public BookDTO create(@Valid @RequestBody CreateBookRequest req) {
    return bookService.create(req); // only called if validation passes
}

// Custom validator (cross-field)
@Target(ElementType.TYPE)
@Constraint(validatedBy = DateRangeValidator.class)
public @interface ValidDateRange {
    String message() default "End date must be after start date";
    // ...
}
```

### ⚡ Remember
- `@Valid` on `@RequestBody` = auto-validation
- Common: `@NotBlank`, `@NotNull`, `@Size`, `@Email`, `@Pattern`, `@Positive`
- Handle errors: `@ExceptionHandler(MethodArgumentNotValidException.class)`
- Custom validators: `@Constraint(validatedBy = ...)` for complex rules

---

[← Back to Spring](./README.md) | [← Back to Java](../README.md) | [← Back to Home](../../../../README.md)
