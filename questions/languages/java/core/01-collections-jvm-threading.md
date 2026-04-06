# ☕ Java Core — Collections, JVM & Thread Safety (Q1–Q5)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. How does HashMap work internally? Explain buckets, hashing mechanism, and resizing process.

### 📝 One-Liner
HashMap stores key-value pairs in an array of buckets; the key's hashCode determines the bucket index, and it resizes (doubles) when load factor (0.75) is exceeded.

### 🔑 Quick Answer
Internally, HashMap uses an **array of Node<K,V>** (called buckets). When you call `put(key, value)`: **(1)** `key.hashCode()` is computed and **spread** (XOR with upper bits) to reduce collisions. **(2)** Bucket index = `hash & (n-1)` where n = array length. **(3)** If the bucket is empty, a new Node is placed there. If occupied (**collision**), nodes form a **linked list** (or a **red-black tree** if list length ≥ 8 — Java 8+ optimization). **(4)** On `get(key)`, same hash → same bucket → traverse list/tree → `equals()` match. **(5)** **Resizing**: when size > capacity × loadFactor (default 16 × 0.75 = 12), array doubles to 32 and ALL entries are **rehashed**. *(hashCode se bucket milta hai, equals se sahi key milti hai)*

### 📖 How It Works
```
HashMap Internal Structure (Java 8+):

table[] (array of buckets, default size = 16)
  Index:  [0]  [1]  [2]  [3]  [4]  [5]  ...  [15]
           │         │
           ↓         ↓
         Node      Node → Node → Node     (linked list for collisions)
        (K1,V1)   (K2,V2) (K3,V3) (K4,V4)
                                    ↓
                             If list ≥ 8 nodes → converts to Red-Black Tree
                             O(n) → O(log n) lookup

put(key, value) Flow:
  1. hash = spread(key.hashCode())
     // spread: hash ^ (hash >>> 16) — mixes high bits into low bits
  2. index = hash & (table.length - 1)    // bitwise AND = fast modulo
  3. If table[index] == null → new Node(hash, key, value)
  4. If table[index] != null:
     a. Check if same key (hash match + equals()) → replace value
     b. Different key → add to linked list / tree

get(key) Flow:
  1. hash = spread(key.hashCode())
  2. index = hash & (table.length - 1)
  3. Traverse bucket: compare hash first (fast), then equals() (slow)

Resizing (rehash):
  size > 16 × 0.75 = 12 → double to 32
  ALL entries re-distributed (hash & (32-1) gives different index)
  Old bucket 5 → might split into bucket 5 and bucket 21

  Capacity: 16 → 32 → 64 → 128 → ...  (always power of 2)
```

### 🗣️ Answering Approach
"HashMap works with an array of buckets — default size 16. When putting a key-value pair, the key's hashCode is computed and spread using XOR with its upper 16 bits to reduce collision clustering. The bucket index is determined by bitwise AND with array length minus one. If multiple keys map to the same bucket — a collision — they form a linked list, which converts to a red-black tree when the list reaches 8 nodes, an optimization added in Java 8 to prevent O(n) degradation. When the number of entries exceeds capacity times load factor — default 0.75, so 12 for size 16 — the array doubles and all entries are rehashed. In my project, I always override both hashCode and equals when using custom objects as keys, and I set initial capacity when the expected size is known to avoid unnecessary resizing."

### 💻 Code
```java
// Internal Node structure (simplified)
static class Node<K,V> {
    final int hash;
    final K key;
    V value;
    Node<K,V> next;  // linked list chain
}

// How put() works (simplified)
public V put(K key, V value) {
    int hash = spread(key.hashCode());        // step 1: hash
    int index = hash & (table.length - 1);    // step 2: bucket index
    
    if (table[index] == null) {
        table[index] = new Node(hash, key, value);  // step 3: empty bucket
    } else {
        // step 4: collision — traverse chain
        Node<K,V> node = table[index];
        while (node != null) {
            if (node.hash == hash && node.key.equals(key)) {
                node.value = value;  // same key → replace
                return;
            }
            node = node.next;
        }
        // add new node to chain
    }
    if (++size > threshold) resize();  // step 5: check resize
}

// Best practice: set initial capacity
Map<String, Order> orders = new HashMap<>(1024);  // avoid resizing
// Formula: expectedSize / 0.75 + 1, rounded to power of 2

// Custom key — MUST override hashCode + equals
public class EmployeeId {
    private final String department;
    private final int id;
    
    @Override
    public int hashCode() {
        return Objects.hash(department, id);
    }
    
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof EmployeeId e)) return false;
        return id == e.id && Objects.equals(department, e.department);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Not overriding hashCode+equals** → custom key objects won't match on `get()` — each `new` creates different hash *(hashCode override nahi kiya toh key milegi nahi)*
- **Mutable keys** → if key fields change after `put()`, hash changes → entry becomes unreachable *(key badal gaya toh entry kho jayegi)*
- **Not thread-safe** → concurrent put can corrupt internal structure (infinite loop in Java 7, data loss in Java 8+)
- **Large initial capacity** wastes memory; **too small** causes excessive resizing
- **Treeify threshold = 8**, untreeify = 6 — don't rely on tree behavior for performance

### 🎯 Tricky Interview Qs

**Q: Why is capacity always a power of 2?**
Because `hash & (n-1)` works as a fast modulo only when n is a power of 2. Example: `hash & 15` = `hash % 16`. *(Power of 2 se bitwise AND fast modulo ban jaata hai)*

**Q: What happens if two keys have the same hashCode AND equals() returns true?**
The second `put()` overwrites the first value — it's the same logical key.

**Q: Why was linked list → tree conversion added in Java 8?**
Denial-of-service protection. Attackers could craft keys with identical hashCodes, causing O(n) degradation. Trees guarantee O(log n).

**Q: What happens during concurrent put() without synchronization?**
In Java 7: infinite loop during resize (cyclic linked list). In Java 8+: data loss, corrupted structure, `ConcurrentModificationException` on iteration.

### ⚡ Remember
- hashCode → bucket index, equals → key match within bucket *(hashCode = kahan, equals = kaun)*
- Default: capacity=16, loadFactor=0.75, resize at 12 entries
- Collision: linked list → tree at 8 nodes (Java 8+)
- Capacity always power of 2 (fast modulo via bitwise AND)
- Always override both hashCode AND equals together

### 🔗 Follow-ups
- [Q2 → HashMap vs ConcurrentHashMap](#q2)
- How does TreeMap differ? (sorted keys, O(log n) all ops)
- What is IdentityHashMap? (uses == instead of equals)

---

<a id="q2"></a>
## Q2. What is the difference between HashMap and ConcurrentHashMap?

### 📝 One-Liner
HashMap is not thread-safe (fast, single-threaded use); ConcurrentHashMap uses segment/node-level locking for safe concurrent access without locking the entire map.

### 🔑 Quick Answer
**HashMap**: no synchronization → fast but unsafe for concurrent access. Multiple threads doing `put()` can corrupt internal structure. **ConcurrentHashMap**: thread-safe with **fine-grained locking**. Java 7 used segment locking (16 segments); Java 8+ uses **node-level CAS + synchronized** on individual bucket heads — only the affected bucket is locked, not the entire map. Other differences: ConcurrentHashMap does NOT allow null keys or values (ambiguity in concurrent context), while HashMap allows one null key and many null values. *(HashMap = single thread ke liye, ConcurrentHashMap = multiple threads ke liye — per-bucket lock)*

### 📖 How It Works
```
HashMap (no sync):
  Thread-1: put("A", 1)  ──→ ┐
  Thread-2: put("B", 2)  ──→ ├─→ RACE CONDITION! Data corruption
  Thread-3: get("A")     ──→ ┘

Collections.synchronizedMap (global lock):
  Thread-1: put("A", 1)  ──→ 🔒 LOCK entire map
  Thread-2: put("B", 2)  ──→ ⏳ WAITING...    (everyone waits)
  Thread-3: get("A")     ──→ ⏳ WAITING...
  → Correct but SLOW (all ops serialized)

ConcurrentHashMap (per-bucket lock, Java 8+):
  Bucket[0]: Thread-1 put("A",1) → 🔒 lock bucket[0] only
  Bucket[5]: Thread-2 put("B",2) → 🔒 lock bucket[5] only  ← PARALLEL!
  Bucket[0]: Thread-3 get("A")   → no lock needed for reads (volatile)
  → Correct AND FAST (different buckets = no contention)

Java 8+ Internal:
  - CAS (Compare-And-Swap) for new bucket insertion (lock-free)
  - synchronized(bucketHead) only for collision chains
  - Reads are lock-free (volatile Node references)
  - size() uses CounterCell array (distributed counting)
```

### 🗣️ Answering Approach
"The key difference is thread safety. HashMap has no synchronization — concurrent modifications can corrupt its internal structure. ConcurrentHashMap provides thread safety with minimal contention using node-level locking in Java 8+. When inserting into an empty bucket, it uses lock-free CAS. For collisions, it synchronizes only on the bucket head node — so threads accessing different buckets proceed in parallel. Reads don't require locks at all because node references are volatile. In my project, I use ConcurrentHashMap for shared caches — like a session cache accessed by hundreds of concurrent requests — where a synchronized wrapper would be a bottleneck."

### 💻 Code
```java
// HashMap — single-threaded only
Map<String, Integer> map = new HashMap<>();  // not safe for concurrent use

// Collections.synchronizedMap — correct but slow (global lock)
Map<String, Integer> syncMap = Collections.synchronizedMap(new HashMap<>());
// Every operation acquires the same lock — poor concurrency

// ConcurrentHashMap — correct AND fast (fine-grained locking)
ConcurrentHashMap<String, Integer> concMap = new ConcurrentHashMap<>();

// Atomic compound operations (CHM provides these, HashMap doesn't)
concMap.putIfAbsent("key", 1);          // atomic check-then-put
concMap.computeIfAbsent("key", k -> expensiveCompute(k));  // lazy compute
concMap.merge("key", 1, Integer::sum);  // atomic merge

// WRONG: compound operation on HashMap (race condition)
if (!map.containsKey("key")) {   // Thread A checks: not present
    map.put("key", value);        // Thread B inserts between check and put!
}

// RIGHT: atomic compound op on ConcurrentHashMap
concMap.putIfAbsent("key", value);  // single atomic operation

// ConcurrentHashMap as a cache
private final ConcurrentHashMap<Long, Customer> customerCache = new ConcurrentHashMap<>();

public Customer getCustomer(Long id) {
    return customerCache.computeIfAbsent(id, this::loadFromDb);
    // thread-safe: only one thread computes, others wait for result
}
```

### ⚠️ Pitfalls / Gotchas
- ConcurrentHashMap does NOT allow null keys or null values → NullPointerException *(null allowed nahi hai — HashMap mein hai, CHM mein nahi)*
- `size()` in ConcurrentHashMap is an estimate under concurrent modification
- Iterators are **weakly consistent** — may or may not reflect concurrent modifications
- `computeIfAbsent()` blocks the bucket if computation is slow — don't do I/O inside it
- Replacing HashMap with ConcurrentHashMap doesn't fix logic-level race conditions

### 🆚 vs. Comparison
| Feature | HashMap | ConcurrentHashMap | synchronizedMap |
|---------|---------|-------------------|-----------------|
| Thread-safe | ❌ No | ✅ Yes (fine-grained) | ✅ Yes (global lock) |
| Null key | ✅ One null key | ❌ No | ✅ One null key |
| Null value | ✅ Multiple | ❌ No | ✅ Multiple |
| Locking | None | Per-bucket | Entire map |
| Read perf | Fastest | Fast (lock-free) | Slow (locked) |
| Write perf | Fastest | Fast (per-bucket) | Slow (serialized) |
| Atomic ops | ❌ No | ✅ putIfAbsent, compute | ❌ No |
| Use case | Single-threaded | Multi-threaded ⭐ | Legacy sync |

### 🎯 Tricky Interview Qs

**Q: Why doesn't ConcurrentHashMap allow null keys/values?**
In concurrent context, `get(key)` returning null is ambiguous — does it mean the key doesn't exist, or the value IS null? HashMap can use `containsKey()` to disambiguate, but in ConcurrentHashMap another thread could change the map between calls.

**Q: Is ConcurrentHashMap slower than HashMap for single-threaded use?**
Slightly — CAS and volatile reads have minor overhead. But the difference is negligible. If in doubt, use ConcurrentHashMap for safety.

### ⚡ Remember
- HashMap = no sync, fast, single-threaded *(ek thread = HashMap)*
- ConcurrentHashMap = per-bucket lock, safe, multi-threaded *(multiple threads = CHM)*
- CHM: no null keys/values, atomic compound ops
- Reads are lock-free (volatile), writes lock only the bucket
- `computeIfAbsent()` = thread-safe lazy initialization

### 🔗 Follow-ups
- [Q1 → HashMap internals](#q1)
- [Q5 → Making classes thread-safe](#q5)
- How does ConcurrentHashMap.size() work? (CounterCell distributed counting)

---

<a id="q3"></a>
## Q3. Explain the JVM memory architecture (Heap, Stack, Metaspace).

### 📝 One-Liner
JVM memory has five areas: Heap (objects, shared), Stack (per-thread method frames), Metaspace (class metadata, native memory), Method Area, and PC Registers.

### 🔑 Quick Answer
**(1) Heap**: where ALL objects live. Shared across threads. Divided into **Young Generation** (Eden + Survivor S0/S1) and **Old Generation**. GC runs here. **(2) Stack**: per-thread, stores method call frames (local variables, operand stack, return address). Fixed size (~1MB default). StackOverflowError if too deep. **(3) Metaspace** (Java 8+, replaced PermGen): stores class metadata, method bytecode, constant pool. Lives in **native memory** (not heap) — grows dynamically but can be bounded with `-XX:MaxMetaspaceSize`. *(Heap = objects ka ghar, Stack = har thread ka apna, Metaspace = class ka blueprint)*

### 📖 How It Works
```
JVM Memory Architecture:

┌──────────────────────────────────────────────────────┐
│                    JVM PROCESS                        │
│                                                       │
│  ┌─── HEAP (shared, GC-managed) ─────────────────┐  │
│  │  ┌─── Young Gen ───────────────┐               │  │
│  │  │  Eden    │  S0  │  S1       │  → Minor GC   │  │
│  │  │ (new obj)│(surv)│(surv)     │               │  │
│  │  └─────────────────────────────┘               │  │
│  │  ┌─── Old Gen ────────────────┐               │  │
│  │  │  Long-lived objects         │  → Major GC   │  │
│  │  │  (survived many Minor GCs) │               │  │
│  │  └─────────────────────────────┘               │  │
│  └────────────────────────────────────────────────┘  │
│                                                       │
│  ┌─ Stack (per thread) ─┐  ┌─ Stack (Thread-2) ─┐   │
│  │  Frame: main()        │  │  Frame: run()       │   │
│  │    locals: x=5, y=10  │  │    locals: i=0      │   │
│  │  Frame: calculate()   │  │  Frame: process()   │   │
│  │    locals: result=15  │  │    locals: data=ref  │   │
│  └───────────────────────┘  └─────────────────────┘   │
│                                                       │
│  ┌─── Metaspace (native memory) ─────────────────┐  │
│  │  Class metadata (Employee.class, Order.class)  │  │
│  │  Method bytecode, constant pool                │  │
│  │  Annotations, static final constants           │  │
│  └────────────────────────────────────────────────┘  │
│                                                       │
│  PC Registers (per thread) — current instruction ptr  │
│  Native Method Stack — for JNI calls                  │
└──────────────────────────────────────────────────────┘

Object Lifecycle:
  new Object() → Eden → (survive Minor GC) → Survivor → (survive N GCs) → Old Gen

JVM Flags:
  -Xms512m -Xmx4g         → Heap min/max
  -Xss1m                   → Stack size per thread
  -XX:MaxMetaspaceSize=256m → Metaspace cap
  -XX:NewRatio=2           → Old:Young = 2:1
```

### 🗣️ Answering Approach
"JVM memory is divided into several key areas. The Heap is shared across all threads and stores all object instances — it's divided into Young Generation for short-lived objects and Old Generation for long-lived ones. The garbage collector manages heap memory, with Minor GC cleaning Young Gen frequently and Major GC cleaning Old Gen less frequently. Each thread gets its own Stack that stores method call frames with local variables — it's fixed size, typically 1MB. Metaspace, which replaced PermGen in Java 8, stores class metadata in native memory and grows dynamically. In my project, we tuned the JVM with -Xmx4g for heap, monitored Young Gen sizing to keep Minor GC pauses under 20ms, and set MaxMetaspaceSize to 256m to prevent unbounded growth from dynamic class loading."

### 💻 Code
```java
// Where things live in memory:

public class MemoryExample {
    // Static field → Metaspace (class-level)
    private static final String APP_NAME = "MyApp";
    
    // Instance field → Heap (part of the object)
    private List<String> items = new ArrayList<>();
    
    public void process() {
        // Local primitive → Stack (this thread's frame)
        int count = 10;
        
        // Local reference → Stack, but the object → Heap
        String name = new String("test");
        //  name (ref) → Stack
        //  "test" String object → Heap
        
        // Array object → Heap
        int[] numbers = new int[100];
    }
}

// JVM startup flags (production)
// java -Xms2g -Xmx4g              # Heap: start 2GB, max 4GB
//      -Xss512k                    # Stack: 512KB per thread
//      -XX:MaxMetaspaceSize=256m   # Metaspace cap
//      -XX:+UseG1GC                # G1 garbage collector
//      -XX:+HeapDumpOnOutOfMemoryError
//      -XX:HeapDumpPath=/logs/heap.hprof
//      -jar app.jar

// Monitor memory at runtime
Runtime rt = Runtime.getRuntime();
long heapMax = rt.maxMemory();        // -Xmx value
long heapUsed = rt.totalMemory() - rt.freeMemory();
double usagePercent = (double) heapUsed / heapMax * 100;
```

### ⚠️ Pitfalls / Gotchas
- **StackOverflowError** = infinite/deep recursion (stack frames exceed -Xss) *(recursion bahut deep gaya — stack bhar gaya)*
- **OutOfMemoryError: Java heap space** = too many objects, possible memory leak
- **OutOfMemoryError: Metaspace** = too many classes loaded (common with Spring devtools, heavy reflection)
- Primitives on stack ≠ primitive wrappers on heap (Integer, Long are objects → Heap)
- String pool lives in Heap (moved from PermGen in Java 7)

### 🎯 Tricky Interview Qs

**Q: Where do static variables live?**
In the Heap as part of the Class object (java.lang.Class instance for that class). The metadata is in Metaspace but static field values are in Heap.

**Q: What's the difference between -Xms and -Xmx?**
-Xms = initial heap size (allocated at startup). -Xmx = maximum heap size. Setting them equal avoids runtime resize overhead. *(Xms = shuru mein kitna, Xmx = maximum kitna le sakta hai)*

**Q: Why was PermGen replaced with Metaspace?**
PermGen had a fixed max size → frequent OutOfMemoryError. Metaspace uses native memory and grows automatically — easier to manage, fewer OOM errors.

### ⚡ Remember
- **Heap** = objects (shared, GC-managed) → Young Gen + Old Gen
- **Stack** = per-thread, method frames, local variables (~1MB)
- **Metaspace** = class metadata, native memory (replaced PermGen in Java 8) *(Metaspace = class ka blueprint, native memory mein rehta hai)*
- `new` anything → Heap; primitives in methods → Stack
- -Xmx (heap), -Xss (stack), -XX:MaxMetaspaceSize (metaspace)

### 🔗 Follow-ups
- [Q4 → OutOfMemoryError causes and troubleshooting](#q4)
- How does Garbage Collection work? (Mark-Sweep-Compact, G1GC)
- What is escape analysis? (JVM optimization — stack allocation for non-escaping objects)

---

<a id="q4"></a>
## Q4. What are common causes of OutOfMemoryError in production? How would you troubleshoot it?

### 📝 One-Liner
Common causes: memory leaks (growing collections/caches), large query results without pagination, excessive threads, or Metaspace overflow from class loading — troubleshoot with heap dumps, MAT, and GC logs.

### 🔑 Quick Answer
**Common causes**: **(1) Memory leak** — objects keep accumulating (e.g., static HashMap/cache that only grows, never evicts). **(2) Large dataset** — loading 1M rows into memory instead of pagination/streaming. **(3) Too many threads** — each thread ~1MB stack → 2000 threads = 2GB just for stacks. **(4) Metaspace overflow** — dynamic class generation (CGLIB proxies, Spring devtools reloading). **(5) Connection/resource leak** — unclosed streams, connections holding buffers. **Troubleshooting**: enable `-XX:+HeapDumpOnOutOfMemoryError`, analyze dump with Eclipse MAT, check GC logs for memory trends, use VisualVM/JFR for live monitoring. *(Sabse common = memory leak — collection badhti rehti hai, shrink nahi hoti)*

### 📖 How It Works
```
OOM Types and Causes:

1. java.lang.OutOfMemoryError: Java heap space
   Code:  static Map<String, byte[]> cache = new HashMap<>();  // grows forever!
   Fix:   Use bounded cache (Caffeine, LRU), pagination, streaming

2. java.lang.OutOfMemoryError: GC overhead limit exceeded
   Cause: GC running >98% of time, recovering <2% memory
   Fix:   Memory leak — heap dump analysis

3. java.lang.OutOfMemoryError: Metaspace
   Cause: Too many classes loaded (Spring proxies, Groovy scripts)
   Fix:   -XX:MaxMetaspaceSize=256m, check class loading count

4. java.lang.OutOfMemoryError: unable to create native thread
   Cause: 2000+ threads × 1MB stack each
   Fix:   Use thread pools (Executors), reduce -Xss

5. java.lang.OutOfMemoryError: Direct buffer memory
   Cause: NIO ByteBuffer.allocateDirect() exhausted
   Fix:   -XX:MaxDirectMemorySize, proper buffer release

Troubleshooting Flow:
  1. Enable: -XX:+HeapDumpOnOutOfMemoryError -XX:HeapDumpPath=/logs/
  2. OOM happens → heap dump generated (.hprof file)
  3. Open in Eclipse MAT (Memory Analyzer Tool)
  4. MAT shows: "Leak Suspects" → biggest retained objects
  5. Suspect: "HashMap holding 2.5 million Entry objects (1.8GB)"
  6. Dominator Tree → trace back to static field in CacheService
  7. Fix: replace with Caffeine cache (maximumSize=10000, expireAfterWrite=1h)
```

### 🗣️ Answering Approach
"In production, the most common OOM cause I've seen is memory leaks — typically a cache or collection that grows unboundedly. My troubleshooting approach has four steps. First, I always have `-XX:+HeapDumpOnOutOfMemoryError` enabled so we get a heap dump automatically. Second, I analyze the dump with Eclipse MAT — the Leak Suspects report immediately shows the largest retained objects and their reference chains. Third, I check GC logs to see the memory trend — a sawtooth pattern where the baseline keeps rising confirms a leak. Fourth, I verify the fix in a staging environment under load test before deploying. In my project, MAT revealed a static HashMap in our audit service holding 2 million entries — we replaced it with a Caffeine cache with a 10,000 entry limit and 1-hour TTL."

### 💻 Code
```java
// CAUSE 1: Unbounded static cache (classic memory leak)
public class AuditService {
    // BAD: grows forever, never cleans up
    private static final Map<String, AuditLog> cache = new HashMap<>();
    
    public void log(String event, AuditLog data) {
        cache.put(event + "_" + System.nanoTime(), data);  // only adds, never removes!
    }
}

// FIX: Use bounded cache with eviction
private final Cache<String, AuditLog> cache = Caffeine.newBuilder()
        .maximumSize(10_000)
        .expireAfterWrite(Duration.ofHours(1))
        .build();

// CAUSE 2: Loading entire table into memory
// BAD: loads 5 million rows into List
List<Order> allOrders = orderRepository.findAll();

// FIX: Use pagination or streaming
Page<Order> page = orderRepository.findAll(PageRequest.of(0, 500));
// or
@Query("SELECT o FROM Order o")
Stream<Order> streamAll();  // processes row by row

// CAUSE 3: Resource leak
// BAD: connection never closed on exception
Connection conn = dataSource.getConnection();
Statement stmt = conn.createStatement();
ResultSet rs = stmt.executeQuery("SELECT * FROM large_table");
// if exception here → connection leaked!

// FIX: try-with-resources
try (Connection conn = dataSource.getConnection();
     Statement stmt = conn.createStatement();
     ResultSet rs = stmt.executeQuery("SELECT * FROM large_table")) {
    while (rs.next()) { /* process */ }
}  // auto-closed even on exception

// JVM flags for production
// java -Xmx4g
//      -XX:+UseG1GC
//      -XX:+HeapDumpOnOutOfMemoryError
//      -XX:HeapDumpPath=/logs/heapdump.hprof
//      -Xlog:gc*:file=/logs/gc.log:time,uptime,level,tags
//      -jar app.jar
```

### ⚠️ Pitfalls / Gotchas
- **HeapDumpOnOutOfMemoryError** must be enabled BEFORE OOM happens — add it to your startup script *(pahle se enable karo — OOM ke baad enable karna late hai)*
- Heap dumps can be huge (4GB heap → 4GB+ dump file) — ensure enough disk space
- `System.gc()` is a suggestion, not a command — never rely on it
- Increasing `-Xmx` is a bandaid, not a fix — just delays the OOM if there's a leak
- String concatenation in loops creates many short-lived objects → GC pressure

### 🎯 Tricky Interview Qs

**Q: How do you differentiate a memory leak from insufficient heap?**
Memory leak: heap usage trend keeps rising after each Full GC (baseline creeps up). Insufficient heap: heap usage is flat but near max, GC reclaims significant memory each time. *(Leak = GC ke baad bhi memory nahi ghatti, Insufficient = GC kaam kar raha hai lekin jagah kam hai)*

**Q: Can memory leak happen with garbage collection?**
Yes! If objects are still reachable (referenced by a static field, listener, or thread-local) but logically unused, GC can't collect them. That's a leak — objects the programmer forgot about but GC considers alive.

### ⚡ Remember
- **#1 cause = unbounded cache/collection** (static Map that only grows) *(sabse common = Map mein daalte jao, nikaalte nahi)*
- Always enable `-XX:+HeapDumpOnOutOfMemoryError` in production
- Analyze dumps with Eclipse MAT → Leak Suspects → Dominator Tree
- GC log rising baseline = leak confirmation
- Fix: bounded caches (Caffeine), pagination, try-with-resources

### 🔗 Follow-ups
- [Q3 → JVM Memory Architecture](#q3)
- How does G1GC work? (region-based, predictable pause times)
- What are thread-local leaks? (thread pool + ThreadLocal = classic leak)

---

<a id="q5"></a>
## Q5. How do you make a class thread-safe? Provide a practical example.

### 📝 One-Liner
Make a class thread-safe by protecting shared mutable state using synchronized blocks, concurrent collections, atomic variables, or making the class immutable.

### 🔑 Quick Answer
A class is thread-safe if it behaves correctly when accessed from multiple threads simultaneously. Five strategies: **(1) Immutability** — no mutable state = no synchronization needed (best approach). **(2) synchronized** — lock critical sections. **(3) Concurrent collections** — ConcurrentHashMap, CopyOnWriteArrayList. **(4) Atomic variables** — AtomicInteger, AtomicReference for lock-free thread safety. **(5) ThreadLocal** — each thread gets its own copy (no sharing). The golden rule: if state is shared AND mutable, you MUST synchronize access. *(Shared + mutable = synchronize karna padega — immutable bana do toh tension nahi)*

### 📖 How It Works
```
Thread Safety Decision Tree:

  Is the state shared across threads?
    ├── No → already safe ✅ (local variables, method parameters)
    └── Yes → Is the state mutable?
              ├── No → already safe ✅ (final fields, immutable objects)
              └── Yes → MUST SYNCHRONIZE ⚠️
                        ├── Simple counter? → AtomicInteger (lock-free)
                        ├── Collection? → ConcurrentHashMap/CopyOnWriteArrayList
                        ├── Complex state? → synchronized blocks
                        └── Per-thread state? → ThreadLocal

Strategies (from best to worst):
  1. Immutability (no sync needed)         ← BEST
  2. Atomic variables (lock-free, fast)
  3. Concurrent collections (built-in)
  4. synchronized/ReentrantLock (explicit)
  5. ThreadLocal (per-thread isolation)     ← situational
```

### 🗣️ Answering Approach
"I approach thread safety by first asking: is the state shared AND mutable? If I can make the class immutable — all final fields, no setters — that's the best solution because it needs zero synchronization. For counters and flags, I use atomic variables like AtomicInteger for lock-free updates. For shared collections, I use ConcurrentHashMap or CopyOnWriteArrayList. For complex state changes involving multiple fields, I use synchronized blocks — always on a private lock object, never on `this` to avoid external interference. In my project, I made our rate limiter thread-safe using AtomicInteger for the counter and ConcurrentHashMap for per-client tracking, handling 5000 requests per second safely."

### 💻 Code
```java
// ❌ NOT thread-safe — race condition on counter and map
public class UserStats {
    private int totalRequests = 0;
    private Map<String, Integer> perUserCount = new HashMap<>();
    
    public void recordRequest(String userId) {
        totalRequests++;                          // race condition!
        perUserCount.put(userId,                   // race condition!
            perUserCount.getOrDefault(userId, 0) + 1);
    }
}

// ✅ Strategy 1: Immutable class (best if state doesn't change)
public final class UserConfig {
    private final String name;
    private final List<String> roles;
    
    public UserConfig(String name, List<String> roles) {
        this.name = name;
        this.roles = List.copyOf(roles);  // unmodifiable defensive copy
    }
    
    public String getName() { return name; }
    public List<String> getRoles() { return roles; }  // already unmodifiable
}

// ✅ Strategy 2: Atomic variables + ConcurrentHashMap
public class UserStats {
    private final AtomicInteger totalRequests = new AtomicInteger(0);
    private final ConcurrentHashMap<String, AtomicInteger> perUserCount = new ConcurrentHashMap<>();
    
    public void recordRequest(String userId) {
        totalRequests.incrementAndGet();       // atomic, lock-free
        perUserCount
            .computeIfAbsent(userId, k -> new AtomicInteger(0))  // atomic creation
            .incrementAndGet();                 // atomic increment
    }
    
    public int getTotalRequests() { return totalRequests.get(); }
    public int getUserCount(String userId) {
        AtomicInteger count = perUserCount.get(userId);
        return count != null ? count.get() : 0;
    }
}

// ✅ Strategy 3: synchronized (for complex multi-field updates)
public class BankAccount {
    private final Object lock = new Object();  // private lock object
    private double balance;
    private List<Transaction> history = new ArrayList<>();
    
    public void transfer(BankAccount target, double amount) {
        // Lock ordering prevents deadlock (always lock lower ID first)
        Object firstLock = this.hashCode() < target.hashCode() ? this.lock : target.lock;
        Object secondLock = this.hashCode() < target.hashCode() ? target.lock : this.lock;
        
        synchronized (firstLock) {
            synchronized (secondLock) {
                if (this.balance >= amount) {
                    this.balance -= amount;
                    target.balance += amount;
                    this.history.add(new Transaction(-amount));
                    target.history.add(new Transaction(amount));
                }
            }
        }
    }
}

// ✅ Strategy 4: ThreadLocal (per-thread isolation)
public class DateFormatter {
    // SimpleDateFormat is NOT thread-safe — use ThreadLocal
    private static final ThreadLocal<SimpleDateFormat> formatter =
        ThreadLocal.withInitial(() -> new SimpleDateFormat("yyyy-MM-dd"));
    
    public String format(Date date) {
        return formatter.get().format(date);  // each thread has its own instance
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **synchronized(this)** → anyone with a reference can lock your object externally → use private lock *(this pe lock mat karo — bahar se koi bhi lock le sakta hai)*
- **Check-then-act** on ConcurrentHashMap is still a race: `if (!map.containsKey(k)) map.put(k, v)` → use `putIfAbsent()`
- **Atomic != compound-safe**: two atomic ops together aren't atomic → need sync for multi-step changes
- **ThreadLocal leak** in thread pools: thread is reused → old ThreadLocal value persists → always call `remove()`
- **Double-checked locking** is broken without `volatile` (Java Memory Model)

### 🎯 Tricky Interview Qs

**Q: Is a class with only final fields automatically thread-safe?**
Yes, IF the fields themselves are immutable too. A `final List<String>` isn't safe if someone can mutate the list. Use `List.copyOf()` or `Collections.unmodifiableList()`. *(final sirf reference lock karta hai — content bhi immutable hona chahiye)*

**Q: Can you have a thread-safe class with mutable state and no synchronized keyword?**
Yes — using atomic variables (AtomicInteger, AtomicReference) and CAS operations. Also `volatile` for single-variable visibility (but not atomicity of compound operations).

### ⚡ Remember
- **Shared + mutable = must synchronize** *(shared + mutable = danger zone)*
- Best: **immutable** (no sync needed)
- Counter: **AtomicInteger** (lock-free)
- Collection: **ConcurrentHashMap** (per-bucket lock)
- Complex state: **synchronized** on private lock
- Per-thread: **ThreadLocal** (but clean up in pools!)

### 🔗 Follow-ups
- [Q2 → ConcurrentHashMap details](#q2)
- [Q3 → JVM Memory (volatile and happens-before)](#q3)
- What is the Java Memory Model? (happens-before, visibility guarantees)

---

<a id="q6"></a>
## Q6. What is JVM and why is it important?

### 📝 One-Liner
The **Java Virtual Machine (JVM)** is the runtime engine that executes Java bytecode — it provides platform independence ("write once, run anywhere"), automatic memory management (garbage collection), and runtime optimizations (JIT compilation).

### 🔑 Quick Answer
**What**: JVM is a virtual machine that runs `.class` files (bytecode). Java source → `javac` → bytecode → JVM executes. **Why important**: (1) **Platform independence** — same bytecode runs on Windows, Linux, Mac. (2) **Memory management** — automatic garbage collection frees unused objects. (3) **Performance** — JIT compiler converts hot bytecode to native machine code at runtime. (4) **Security** — bytecode verification, sandboxing, ClassLoader isolation. (5) **Language interop** — Kotlin, Scala, Groovy all run on JVM. **JVM ≠ JRE ≠ JDK**: JDK (development kit) ⊃ JRE (runtime) ⊃ JVM (execution engine). *(JVM = Java ka engine — bytecode ko machine code mein convert karta hai, memory manage karta hai, GC chalaata hai)*

### 📖 How It Works
```
Java Source (.java)
       │ javac (compiler)
       ▼
Bytecode (.class)
       │
       ▼
┌────────────────────────────────────────┐
│  JVM                                     │
│  ┌──────────────┐  ┌────────────────┐  │
│  │ Class Loader │  │ Bytecode       │  │
│  │ (loads .class)│  │ Verifier       │  │
│  └──────────────┘  └────────────────┘  │
│  ┌─────────────────────────────────┐  │
│  │ Runtime Data Areas               │  │
│  │  Heap (objects, GC managed)      │  │
│  │  Stack (per thread, frames)      │  │
│  │  Metaspace (class metadata)      │  │
│  │  PC Register | Native Stack      │  │
│  └─────────────────────────────────┘  │
│  ┌─────────────────────────────────┐  │
│  │ Execution Engine                 │  │
│  │  Interpreter (slow, line by line)│  │
│  │  JIT Compiler (fast, hot code)   │  │
│  │  GC (frees unused objects)       │  │
│  └─────────────────────────────────┘  │
└────────────────────────────────────────┘
```

### 🗣️ Answering Approach
"The JVM is the runtime engine that executes Java bytecode. When I compile a `.java` file, `javac` produces platform-independent bytecode. The JVM then loads this bytecode through ClassLoaders, verifies it for safety, and executes it. The JVM has three key responsibilities: first, memory management — it allocates objects on the Heap and automatically reclaims unused memory through garbage collection. Second, performance optimization — the JIT compiler identifies frequently-executed code (hot spots) and compiles them to native machine code, which is why Java can achieve near-C performance. Third, platform independence — the same bytecode runs on any OS with a JVM implementation. For backend applications, understanding JVM is critical because most production issues — memory leaks, GC pauses, thread deadlocks — require JVM-level debugging with tools like JFR, jstack, and GC logs."

### 🆚 JDK vs JRE vs JVM

| Component | Contains | Purpose |
|-----------|----------|---------|
| **JDK** | JRE + javac + debugger + tools | Development |
| **JRE** | JVM + class libraries (java.lang, java.util, etc.) | Running Java apps |
| **JVM** | Execution engine + GC + ClassLoader | Executing bytecode |

### ⚡ Remember
- JVM = **bytecode executor** + **memory manager** + **JIT compiler**
- Write once, run anywhere = bytecode is platform-independent, JVM is platform-specific
- [Q3 → JVM Memory Architecture (Heap, Stack, Metaspace) for deep dive](#q3)
- [Q4 → OOM troubleshooting](#q4)
