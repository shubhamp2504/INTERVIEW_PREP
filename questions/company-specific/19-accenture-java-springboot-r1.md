# 🏢 Accenture — Round 1 Java Spring Boot Interview (Q1–Q11)

> **Source**: Friend's recent experience — Accenture Round 1 for Custom Software Engineer (ML 10)  
> **Experience**: 3.5 years  
> **Coverage**: Java version features, JVM internals, serialization, thread safety, OOP, concurrency, modern Java (virtual threads, sealed classes), design patterns  
> **Verdict**: Among the toughest Accenture R1s seen — goes deep into JVM and modern Java  
> **⚠️ Alert**: Candidate's screen was flagged for a mobile device in front of it — any malpractice indicator can lead to immediate disqualification  
> **Key Takeaway**: Along with preparation, having luck on your side (easy vs hard interviewer) makes a difference

---

<a id="q1"></a>
## Q1. Which version of Java have you used in your project and what are its features?

### 📝 One-Liner
State your project version (typically Java 8/11/17/21), then list key features introduced in that version — interviewers want practical awareness, not a changelog dump.

### 🔑 Quick Answer
In my current project we use **Java 17** (LTS). Key features: **Sealed classes** (restrict inheritance hierarchy), **Records** (immutable data carriers), **Pattern matching for instanceof** (no explicit cast), **Text blocks** (multi-line strings), **Switch expressions** (yield-based), **Helpful NullPointerExceptions** (pinpoint which variable was null), **Strong encapsulation of JDK internals**. Java 8 added Streams, Lambdas, Optional, default methods. Java 11 added HttpClient, var in lambdas, String convenience methods. Java 21 added virtual threads, record patterns, sequenced collections. *(Jo version project mein use kiya hai uski features batao — interviewer dekhta hai ki tum actually version-aware ho ya bas "Java" likh dete ho resume pe)*

### 📖 How It Works (Detailed Explanation)

**Version-wise important features:**

| Version | Year | Key Features |
|---------|------|-------------|
| **Java 8** | 2014 | Lambda, Streams, Optional, Functional interfaces, default/static methods in interface, `java.time` API, CompletableFuture |
| **Java 11** | 2018 | `var` (local variable), HttpClient, `String.isBlank()/.strip()/.lines()`, `Files.readString()`, Running `.java` directly |
| **Java 17** | 2021 | Sealed classes, Records, Pattern matching instanceof, Switch expressions, Text blocks, Helpful NPE |
| **Java 21** | 2023 | Virtual threads, Sequenced collections, Record patterns, Pattern matching switch, String templates (preview) |

**How to answer:**
1. State exact version: "We use Java 17 LTS in production"
2. List 3-4 features you **actually use** in your project
3. Mention migration benefits: "We migrated from Java 11 to 17 for sealed classes and better GC"

### 🗣️ Answering Approach
"We use Java 17 in our current project. The key features we leverage are: Records for DTOs — they eliminate boilerplate getters/setters/equals/hashCode. Sealed classes to restrict our payment strategy hierarchy. Pattern matching for instanceof to avoid explicit casts in our deserialization code. Text blocks for SQL queries and JSON templates. We also benefit from ZGC improvements in 17 for lower latency. Previously we used Java 11 where we adopted the HttpClient API and var keyword."

### ⚠️ Pitfalls / Gotchas
- Don't list features you haven't used — interviewer will dig deeper
- Don't confuse preview features with GA features
- Know the LTS versions: 8, 11, 17, 21 — most companies use LTS

### ⚡ Remember
- **Java 8** = Streams + Lambdas + Optional (still the most asked)
- **Java 17** = Sealed + Records + Pattern matching (current enterprise standard)
- **Java 21** = Virtual threads (next big migration wave)

### 🔗 Cross-references
- Java versions deep dive → [core/17-java-versions-features.md](../languages/java/core/17-java-versions-features.md)

---

<a id="q2"></a>
## Q2. Explain the classloader hierarchy in Java.

### 📝 One-Liner
Java uses a parent-delegation classloading model: **Bootstrap** → **Platform (Extension)** → **Application (System)** — each loader delegates to its parent first before attempting to load the class itself.

### 🔑 Quick Answer
The JVM classloader hierarchy follows **parent-delegation**: (1) **Bootstrap ClassLoader** — loads `java.lang.*`, `java.util.*` from `rt.jar` / JDK modules (written in C++, has no parent), (2) **Platform ClassLoader** (formerly Extension) — loads `javax.*`, extensions from `jre/lib/ext`, (3) **Application ClassLoader** — loads classes from classpath (`-cp`, Maven/Gradle dependencies). When a class is requested, the loader delegates UP to its parent first — if the parent can load it, done. If not (ClassNotFoundException), the child attempts. Custom classloaders extend `ClassLoader` for hot-deploy, plugin systems, etc. *(Jab class load hota hai, pehle parent se puchta hai — "tu load kar sakta hai?" parent bhi apne parent se puchta hai — jab tak koi load na kare, tab child try karta hai)*

### 📖 How It Works (Detailed Explanation)

```
                ┌─────────────────────┐
                │  Bootstrap Loader   │  ← C++ native, loads java.lang.*, rt.jar
                │  (null parent)      │
                └────────┬────────────┘
                         │ delegates UP ↑
                ┌────────▼────────────┐
                │  Platform Loader    │  ← loads javax.*, jdk.* extensions
                │  (Extension)        │
                └────────┬────────────┘
                         │ delegates UP ↑
                ┌────────▼────────────┐
                │  Application Loader │  ← loads -classpath, your app classes
                │  (System)           │
                └────────┬────────────┘
                         │ delegates UP ↑
                ┌────────▼────────────┐
                │  Custom Loader      │  ← hot-deploy, plugins, isolation
                └─────────────────────┘
```

**Parent-delegation flow for `MyService.class`:**
1. Application Loader receives request → delegates to Platform Loader
2. Platform Loader delegates to Bootstrap Loader
3. Bootstrap can't find `MyService` → returns to Platform
4. Platform can't find `MyService` → returns to Application
5. Application finds `MyService.class` on classpath → loads it

**Why parent-delegation?**
- **Security**: Prevents malicious `java.lang.String` from being loaded by application code
- **Uniqueness**: Same class loaded by same loader = same Class object
- **Hierarchy**: Core classes always loaded by trusted Bootstrap loader

### 🗣️ Answering Approach
"Java uses a parent-delegation classloading model. There are three built-in classloaders: Bootstrap, Platform, and Application. When a class needs loading, the request bubbles up to Bootstrap first. If Bootstrap can load it (core Java classes), it does. Otherwise, Platform tries, then Application. This ensures security — you can't replace java.lang.String with a malicious version — and uniqueness — the same class loaded by the same loader is the same Class object. Custom classloaders are used in application servers like Tomcat for webapp isolation and in plugin systems for hot-reloading."

### 💻 Code
```java
// Check which classloader loaded a class
System.out.println(String.class.getClassLoader());       // null (Bootstrap)
System.out.println(javax.sql.DataSource.class.getClassLoader()); // PlatformClassLoader
System.out.println(MyService.class.getClassLoader());    // AppClassLoader

// Custom classloader example
ClassLoader custom = new URLClassLoader(
    new URL[]{new URL("file:///plugins/")},
    Thread.currentThread().getContextClassLoader()  // parent = app loader
);
Class<?> plugin = custom.loadClass("com.example.Plugin");
```

### ⚠️ Pitfalls / Gotchas
- `getClassLoader()` returns `null` for Bootstrap classes — not an error, Bootstrap is native C++
- `ClassCastException` can occur if same class loaded by two different classloaders
- Tomcat breaks parent-delegation intentionally — child-first loading for webapp isolation

### ⚡ Remember
- **B-P-A**: Bootstrap → Platform → Application (delegates UP, loads DOWN)
- `null` classloader = Bootstrap (native C++)
- Parent-delegation = security + uniqueness
- Tomcat/OSGi = child-first (breaks convention intentionally)

---

<a id="q3"></a>
## Q3. Describe the internal working of the JVM.

### 📝 One-Liner
JVM converts `.class` bytecode into native machine code through: **ClassLoader** (loading) → **Runtime Data Areas** (memory) → **Execution Engine** (JIT compilation + GC).

### 🔑 Quick Answer
JVM architecture has three main components: (1) **ClassLoader Subsystem** — loads, links (verify→prepare→resolve), and initializes `.class` files, (2) **Runtime Data Areas** — Method Area (class metadata), Heap (objects), Stack (per-thread frames), PC Register (per-thread instruction pointer), Native Method Stack, (3) **Execution Engine** — Interpreter (line-by-line bytecode), JIT Compiler (hot methods → native), GC (heap cleanup). Java code → `javac` → bytecode → classloader → memory → execution engine → native code. *(JVM ka kaam hai: class load karo, memory allocate karo, bytecode ko machine code mein convert karo — ye teen steps mein hota hai)*

### 📖 How It Works (Detailed Explanation)

```
  .java → javac → .class (bytecode)
                      │
         ┌────────────▼────────────┐
         │   ClassLoader Subsystem │
         │  Load → Link → Init     │
         └────────────┬────────────┘
                      │
         ┌────────────▼────────────────────────────┐
         │         Runtime Data Areas               │
         │  ┌──────────┐  ┌────────┐  ┌─────────┐  │
         │  │ Method    │  │  Heap  │  │ Stack   │  │
         │  │ Area      │  │(objects│  │(per     │  │
         │  │(metadata, │  │ GC     │  │ thread) │  │
         │  │ static)   │  │ managed│  │         │  │
         │  └──────────┘  └────────┘  └─────────┘  │
         │  ┌──────────────┐  ┌──────────────────┐  │
         │  │ PC Register  │  │ Native Method    │  │
         │  │ (per thread) │  │ Stack            │  │
         │  └──────────────┘  └──────────────────┘  │
         └────────────┬────────────────────────────┘
                      │
         ┌────────────▼────────────┐
         │    Execution Engine      │
         │  Interpreter + JIT + GC  │
         └─────────────────────────┘
```

**Memory areas explained:**
| Area | Scope | Contents |
|------|-------|----------|
| **Method Area** | Shared (all threads) | Class metadata, static variables, constant pool, method bytecode |
| **Heap** | Shared (all threads) | All objects and arrays — managed by GC |
| **Stack** | Per thread | Stack frames (local vars, operand stack, return address) |
| **PC Register** | Per thread | Address of currently executing bytecode instruction |
| **Native Method Stack** | Per thread | For JNI native method calls (C/C++) |

**Execution Engine:**
- **Interpreter**: Reads and executes bytecode line-by-line (slow for repeated code)
- **JIT Compiler**: Detects "hot" methods → compiles to native machine code → caches for reuse
- **GC**: Manages heap — Young Gen (Eden → S0 → S1) → Old Gen → Metaspace

### 🗣️ Answering Approach
"JVM has three major subsystems. First, the ClassLoader loads, verifies, and initializes .class files using parent-delegation. Second, Runtime Data Areas organize memory into shared areas — Method Area for class metadata and Heap for objects — plus per-thread areas — Stack for method frames, PC Register for instruction pointers, and Native Method Stack for JNI calls. Third, the Execution Engine runs bytecode through an Interpreter for first-pass execution and JIT Compiler that detects hot methods and compiles them to native code for performance. The Garbage Collector manages heap memory, moving objects through generations — Young Gen for short-lived objects and Old Gen for long-lived ones."

### ⚡ Remember
- **3 subsystems**: ClassLoader → Memory → Execution
- **5 memory areas**: Method Area, Heap (shared) + Stack, PC, Native Stack (per-thread)
- **JIT**: Hot methods get compiled to native code — why Java gets faster over time
- **GC generations**: Young (Eden+S0+S1) → Old → Metaspace

### 🔗 Cross-references
- JVM performance tuning → [core/03-jvm-performance-tuning.md](../languages/java/core/03-jvm-performance-tuning.md)

---

<a id="q4"></a>
## Q4. Write Java code to serialize and deserialize an object.

### 📝 One-Liner
Implement `Serializable`, use `ObjectOutputStream` to write and `ObjectInputStream` to read — mark fields you want to skip with `transient`.

### 🔑 Quick Answer
Serialization converts an object to a byte stream (`ObjectOutputStream.writeObject()`), deserialization restores it (`ObjectInputStream.readObject()`). The class must implement `java.io.Serializable` (marker interface). Use `serialVersionUID` to ensure compatibility across versions. Mark non-serializable or sensitive fields as `transient`. *(Object ko file/network mein save karna hai → serialize karo, wapas object banana hai → deserialize karo — `transient` wale fields skip hote hain)*

### 📖 How It Works (Detailed Explanation)

**Key rules:**
- Class MUST implement `Serializable` — it's a marker interface (no methods)
- `serialVersionUID` prevents `InvalidClassException` during deserialization after class changes
- `transient` fields are NOT serialized (default value on deserialization)
- `static` fields are NOT serialized (belong to class, not object)
- If parent class is not Serializable, its fields won't be serialized

### 🗣️ Answering Approach
"To serialize, the class implements Serializable. I use ObjectOutputStream.writeObject() to convert the object to bytes, and ObjectInputStream.readObject() to restore it. I always declare serialVersionUID explicitly to avoid compatibility issues. Sensitive fields like passwords are marked transient so they're excluded. In production, we typically use Jackson for JSON serialization or Protocol Buffers for performance-critical services rather than Java native serialization, which has security concerns."

### 💻 Code
```java
import java.io.*;

// ✅ Step 1: Implement Serializable + declare serialVersionUID
class Employee implements Serializable {
    private static final long serialVersionUID = 1L;
    
    private int id;
    private String name;
    private transient String password; // ⚠️ NOT serialized
    
    public Employee(int id, String name, String password) {
        this.id = id;
        this.name = name;
        this.password = password;
    }
    
    @Override
    public String toString() {
        return "Employee{id=" + id + ", name='" + name + "', password='" + password + "'}";
    }
}

public class SerializationDemo {
    public static void main(String[] args) throws Exception {
        Employee emp = new Employee(1, "Shubham", "secret123");
        
        // ✅ SERIALIZE — Object → File
        try (ObjectOutputStream oos = new ObjectOutputStream(
                new FileOutputStream("employee.ser"))) {
            oos.writeObject(emp);
            System.out.println("Serialized: " + emp);
            // Output: Employee{id=1, name='Shubham', password='secret123'}
        }
        
        // ✅ DESERIALIZE — File → Object
        try (ObjectInputStream ois = new ObjectInputStream(
                new FileInputStream("employee.ser"))) {
            Employee restored = (Employee) ois.readObject();
            System.out.println("Deserialized: " + restored);
            // Output: Employee{id=1, name='Shubham', password='null'} ← transient!
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Security**: Java native serialization is vulnerable to deserialization attacks — prefer Jackson/Protobuf in production
- **serialVersionUID**: If not declared, JVM auto-generates it — any class change causes `InvalidClassException`
- **transient + default values**: `int` → 0, `String` → null, `boolean` → false after deserialization
- **NotSerializableException**: If any field's type doesn't implement Serializable

### ⚡ Remember
- `Serializable` = marker interface (no methods to implement)
- `transient` = skip during serialization (passwords, caches, connections)
- Always set `serialVersionUID` explicitly for version compatibility
- Production: Use Jackson (JSON) or Protobuf (binary) — not native `ObjectOutputStream`

### 🔗 Cross-references
- Serialization details → [core/16-serialization.md](../languages/java/core/16-serialization.md)

---

<a id="q5"></a>
## Q5. Implement a thread-safe cache in Java.

### 📝 One-Liner
Use `ConcurrentHashMap` for thread-safe reads/writes, add TTL with scheduled cleanup or use Caffeine/Guava for production-grade caching with eviction policies.

### 🔑 Quick Answer
For a simple thread-safe cache: `ConcurrentHashMap` + optional TTL logic. For production: use **Caffeine** (recommended) or **Guava Cache** with configurable eviction (size, time, reference), async refresh, and statistics. The simplest thread-safe approach: `ConcurrentHashMap.computeIfAbsent()` guarantees atomic compute-once semantics. *(Thread-safe cache ke liye `ConcurrentHashMap` use karo — reads lock-free hain, writes segment-level lock. TTL chahiye toh Caffeine use karo)*

### 📖 How It Works (Detailed Explanation)

**Three levels of thread-safe caching:**
1. **Basic**: `ConcurrentHashMap` — thread-safe map, no eviction
2. **With TTL**: `ConcurrentHashMap` + `ScheduledExecutorService` for expiry
3. **Production**: Caffeine with size/time eviction, async loading, stats

### 🗣️ Answering Approach
"For a thread-safe cache, I'd start with ConcurrentHashMap — it provides lock-free reads and segment-level writes. For atomic compute-once, I use computeIfAbsent() which ensures only one thread computes the value. For production with TTL and size limits, I use Caffeine — it supports maximum size eviction, time-based expiry, async refresh, and statistics. I'd avoid synchronized HashMap — ConcurrentHashMap is designed for this purpose."

### 💻 Code
```java
// ✅ Level 1: Simple thread-safe cache with ConcurrentHashMap
import java.util.concurrent.*;

public class SimpleCache<K, V> {
    private final ConcurrentHashMap<K, V> cache = new ConcurrentHashMap<>();
    
    public V get(K key) {
        return cache.get(key);
    }
    
    public V getOrCompute(K key, java.util.function.Function<K, V> loader) {
        return cache.computeIfAbsent(key, loader); // atomic compute-once
    }
    
    public void put(K key, V value) {
        cache.put(key, value);
    }
    
    public void evict(K key) {
        cache.remove(key);
    }
}

// ✅ Level 2: Thread-safe cache with TTL
public class TTLCache<K, V> {
    private final ConcurrentHashMap<K, CacheEntry<V>> cache = new ConcurrentHashMap<>();
    private final long ttlMillis;
    
    record CacheEntry<V>(V value, long expiryTime) {
        boolean isExpired() { return System.currentTimeMillis() > expiryTime; }
    }
    
    public TTLCache(long ttlMillis) {
        this.ttlMillis = ttlMillis;
        // Scheduled cleanup every TTL interval
        Executors.newSingleThreadScheduledExecutor()
            .scheduleAtFixedRate(this::cleanup, ttlMillis, ttlMillis, TimeUnit.MILLISECONDS);
    }
    
    public V get(K key) {
        CacheEntry<V> entry = cache.get(key);
        if (entry == null || entry.isExpired()) {
            cache.remove(key);
            return null;
        }
        return entry.value();
    }
    
    public void put(K key, V value) {
        cache.put(key, new CacheEntry<>(value, System.currentTimeMillis() + ttlMillis));
    }
    
    private void cleanup() {
        cache.entrySet().removeIf(e -> e.getValue().isExpired());
    }
}

// ✅ Level 3: Production-grade with Caffeine
// Cache<String, User> cache = Caffeine.newBuilder()
//     .maximumSize(10_000)
//     .expireAfterWrite(Duration.ofMinutes(5))
//     .refreshAfterWrite(Duration.ofMinutes(1))
//     .recordStats()
//     .build(userId -> userService.findById(userId));
```

### ⚠️ Pitfalls / Gotchas
- Don't use `synchronized` on `HashMap` — use `ConcurrentHashMap` directly
- `computeIfAbsent()` blocks other puts on same key's bucket — keep computation fast
- TTL cleanup thread should be daemon to not block JVM shutdown
- Cache stampede: Multiple threads hit cache miss simultaneously — use `computeIfAbsent` to prevent

### ⚡ Remember
- `ConcurrentHashMap` = thread-safe reads (no lock) + segment-level writes
- `computeIfAbsent()` = atomic compute-once (prevents cache stampede)
- Production cache = Caffeine (size + TTL + stats + async refresh)
- Interview tip: Show progression from simple → TTL → production-grade

---

<a id="q6"></a>
## Q6. Can we inherit overridden and overloaded methods?

### 📝 One-Liner
Yes — both overridden and overloaded methods are inherited by subclasses. The child can further override the parent's method or add new overloads.

### 🔑 Quick Answer
**Yes to both.** **Overloaded methods** (same name, different parameters) are all inherited by subclass — the child gets every version. **Overridden methods** (same name, same parameters) — the child's version replaces the parent's at runtime (dynamic polymorphism). A child can also **override one overload** while inheriting the rest untouched. The only exception: `final` methods cannot be overridden, and `private` methods are not inherited (not visible to subclass). *(Haan, dono inherit hote hain — overloaded methods mein saari versions milti hain child ko, aur overridden method mein child ka version runtime pe chalta hai)*

### 📖 How It Works (Detailed Explanation)

```java
class Parent {
    // Overloaded methods
    void process(String s)     { System.out.println("Parent: String"); }
    void process(int n)        { System.out.println("Parent: int"); }
    void process(String s, int n) { System.out.println("Parent: String+int"); }
}

class Child extends Parent {
    // ✅ Override ONE overload — other two are inherited as-is
    @Override
    void process(String s) { System.out.println("Child: String"); }
    
    // ✅ Add NEW overload — extends the overloaded family
    void process(double d) { System.out.println("Child: double"); }
}

// Usage:
Child c = new Child();
c.process("hello");     // Child: String   ← overridden
c.process(42);          // Parent: int     ← inherited
c.process("hi", 10);    // Parent: String+int ← inherited
c.process(3.14);        // Child: double   ← new overload
```

**Rules summary:**

| Method Type | Inherited? | Can Override? | Notes |
|-------------|-----------|---------------|-------|
| Public/protected overloaded | Yes | Yes (each independently) | Child gets all versions |
| Public/protected overridden | Yes (replaced) | Yes | Runtime polymorphism |
| `final` method | Yes (inherited) | No | Compile error if overridden |
| `private` method | No | No (can define same signature — it's a new method) | Not visible to child |
| `static` method | Yes | No (hiding, not overriding) | Resolved at compile time |

### 🗣️ Answering Approach
"Yes, both overloaded and overridden methods are inherited. When a parent has multiple overloaded methods, the child inherits all versions and can selectively override specific overloads while keeping others. For overriding, the child's implementation replaces the parent's at runtime through dynamic dispatch. The child can also add new overloads, extending the method family. The exceptions are: final methods can't be overridden, private methods aren't inherited, and static methods are hidden, not overridden."

### ⚡ Remember
- Overloaded → all versions inherited; child can override any independently
- Overridden → child replaces parent's at runtime (dynamic polymorphism)
- `final` = inherited but not overridable; `private` = not inherited; `static` = hidden not overridden

---

<a id="q7"></a>
## Q7. How do parallel streams work? Explain intermediate and terminal operations.

### 📝 One-Liner
Parallel streams split data into chunks, process them on ForkJoinPool threads concurrently, then merge results. Intermediate operations (filter, map, sorted) are lazy; terminal operations (collect, forEach, reduce) trigger execution.

### 🔑 Quick Answer
**Parallel streams** use the **ForkJoinPool.commonPool()** (default threads = CPU cores - 1). The stream's Spliterator splits the data source into sub-streams → each processed on separate threads → results merged by terminal operation. **Intermediate operations** (filter, map, flatMap, sorted, distinct, peek) are **lazy** — they build a pipeline, nothing executes until a terminal triggers it. **Terminal operations** (collect, forEach, reduce, count, findFirst, toArray, min/max) **trigger pipeline execution** and produce a result or side-effect. *(Parallel stream data ko chunks mein todta hai, alag threads pe process karta hai, phir merge karta hai. Intermediate lazy hai — jab tak terminal nahi call hota, kuch nahi hota)*

### 📖 How It Works (Detailed Explanation)

```
Sequential:  [1, 2, 3, 4, 5, 6, 7, 8]  → single thread processes all

Parallel:    [1, 2, 3, 4, 5, 6, 7, 8]
              │          │
       ┌──────┘          └──────┐
       ▼                         ▼
    [1,2,3,4]              [5,6,7,8]      ← Spliterator splits
    Thread-1                Thread-2       ← ForkJoinPool
       │                         │
    filter+map              filter+map     ← intermediate ops parallel
       │                         │
       └──────┬──────────┬──────┘
              ▼          ▼
           merge / collect                 ← terminal op combines results
```

**Intermediate operations (lazy — build pipeline):**

| Operation | Type | Example |
|-----------|------|---------|
| `filter()` | Stateless | `.filter(x -> x > 10)` |
| `map()` | Stateless | `.map(String::toUpperCase)` |
| `flatMap()` | Stateless | `.flatMap(Collection::stream)` |
| `sorted()` | **Stateful** | `.sorted(Comparator.comparing(Employee::salary))` |
| `distinct()` | **Stateful** | `.distinct()` (uses equals/hashCode) |
| `peek()` | Stateless | `.peek(System.out::println)` (debugging) |
| `limit()` | **Short-circuiting** | `.limit(5)` |
| `skip()` | Stateful | `.skip(10)` |

**Terminal operations (eager — trigger execution):**

| Operation | Returns | Example |
|-----------|---------|---------|
| `collect()` | Collection/value | `.collect(Collectors.toList())` |
| `forEach()` | void | `.forEach(System.out::println)` |
| `reduce()` | Optional/value | `.reduce(0, Integer::sum)` |
| `count()` | long | `.count()` |
| `findFirst()` | Optional | `.findFirst()` (short-circuiting) |
| `anyMatch()` | boolean | `.anyMatch(x -> x > 100)` |
| `toArray()` | Array | `.toArray(String[]::new)` |

### 🗣️ Answering Approach
"Parallel streams leverage the ForkJoinPool to split data into chunks via Spliterator, process them on separate threads, and merge results. Intermediate operations like filter, map, and sorted are lazy — they just build a pipeline description. Nothing executes until a terminal operation like collect or reduce triggers the pipeline. Stateless operations like filter and map parallelize well. Stateful operations like sorted and distinct need synchronization. I use parallel streams only when: the data is large, operations are CPU-intensive, and the source supports efficient splitting like ArrayList. For I/O or small datasets, parallel streams add overhead."

### 💻 Code
```java
// ✅ Parallel stream example with intermediate + terminal
List<Integer> numbers = IntStream.rangeClosed(1, 1_000_000)
    .boxed()
    .collect(Collectors.toList());

long sum = numbers.parallelStream()       // parallel execution
    .filter(n -> n % 2 == 0)              // intermediate (lazy, stateless)
    .mapToLong(n -> n * 2L)               // intermediate (lazy, stateless)
    .sorted()                              // intermediate (lazy, STATEFUL)
    .sum();                                // terminal (triggers execution)

// ⚠️ Custom thread pool for parallel streams (avoid starving common pool)
ForkJoinPool customPool = new ForkJoinPool(4);
customPool.submit(() ->
    data.parallelStream()
        .filter(this::isValid)
        .collect(Collectors.toList())
).get();
```

### ⚠️ Pitfalls / Gotchas
- **Common ForkJoinPool**: All parallel streams share it — one slow stream blocks others
- **forEach ordering**: `parallelStream().forEach()` doesn't preserve order — use `forEachOrdered()`
- **Stateful operations**: `sorted()`, `distinct()` need cross-thread synchronization — slower
- **Small datasets**: Parallel overhead > benefit for < ~10K elements
- **Shared mutable state**: NEVER modify shared variable inside parallel stream — use `reduce`/`collect`

### ⚡ Remember
- **Lazy**: intermediate operations build pipeline; **Eager**: terminal triggers it
- **Stateless** (filter, map) → parallelize well; **Stateful** (sorted, distinct) → overhead
- Default pool = `ForkJoinPool.commonPool()` (CPU cores - 1 threads)
- Use parallel only for: large data + CPU-intensive + efficient Spliterator

---

<a id="q8"></a>
## Q8. What are virtual threads and how are they different from normal threads?

### 📝 One-Liner
Virtual threads (Java 21) are lightweight, JVM-managed threads that multiplex onto a small pool of OS threads — enabling millions of concurrent threads without the memory/context-switch cost of platform threads.

### 🔑 Quick Answer
**Platform threads** (normal) are 1:1 mapped to OS threads — each consumes ~1MB stack, limited to thousands, expensive context-switch. **Virtual threads** (Java 21, Project Loom) are JVM-managed — they're mounted onto carrier (platform) threads only during CPU work, unmounted during blocking (I/O, sleep). You can create millions of virtual threads. They're ideal for I/O-bound workloads (HTTP calls, DB queries) where threads spend most time waiting. NOT suitable for CPU-bound work (they still share the same ForkJoinPool). *(Platform thread = heavyweight OS thread (1MB each). Virtual thread = lightweight JVM thread (few KB) — lakho bana sakte ho bina system crash kiye)*

### 📖 How It Works (Detailed Explanation)

```
Platform Threads (1:1):
  Java Thread 1 ←→ OS Thread 1 (1MB stack)
  Java Thread 2 ←→ OS Thread 2 (1MB stack)
  Java Thread 3 ←→ OS Thread 3 (1MB stack)
  Max: ~5000-10000 threads (limited by OS)

Virtual Threads (M:N):
  Virtual Thread 1 ─┐
  Virtual Thread 2 ─┤
  Virtual Thread 3 ─┤──→ Carrier Thread 1 (OS Thread)
  Virtual Thread 4 ─┤──→ Carrier Thread 2 (OS Thread)
  ...               │
  Virtual Thread 1M ┘──→ Carrier Thread N (ForkJoinPool)
  Max: millions (limited by heap, few KB each)
```

| Feature | Platform Thread | Virtual Thread |
|---------|----------------|----------------|
| Memory | ~1MB stack per thread | Few KB (grows as needed) |
| Creation cost | Expensive (OS syscall) | Cheap (JVM object) |
| Maximum count | ~5K-10K | Millions |
| Scheduling | OS scheduler | JVM scheduler (ForkJoinPool) |
| Blocking behavior | Blocks OS thread | Unmounts from carrier, frees it |
| Best for | CPU-bound work | I/O-bound work (HTTP, DB, file) |
| Thread pool needed? | Yes (must reuse) | No (create per task, dispose) |

### 🗣️ Answering Approach
"Virtual threads, introduced in Java 21 via Project Loom, are lightweight threads managed by the JVM. Unlike platform threads which map 1:1 to OS threads and cost ~1MB each, virtual threads are just JVM objects costing a few KB. The key innovation is that when a virtual thread blocks on I/O, it unmounts from its carrier thread, freeing it for other virtual threads. This means you can have millions of concurrent virtual threads handling I/O-bound work like HTTP calls or database queries. For CPU-bound work, they don't help since the bottleneck is CPU cores, not threads. The programming model stays the same — Thread.startVirtualThread() or Executors.newVirtualThreadPerTaskExecutor()."

### 💻 Code
```java
// ✅ Creating virtual threads (Java 21+)

// Method 1: Thread.startVirtualThread
Thread vt = Thread.startVirtualThread(() -> {
    System.out.println("Running on: " + Thread.currentThread());
});

// Method 2: Thread.ofVirtual().start()
Thread vt2 = Thread.ofVirtual().name("vt-", 0).start(() -> {
    // I/O-bound work — virtual thread unmounts during blocking
    String response = httpClient.send(request, BodyHandlers.ofString()).body();
});

// Method 3: ExecutorService (recommended for production)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    // Submit 100K tasks — each gets its own virtual thread
    List<Future<String>> futures = IntStream.range(0, 100_000)
        .mapToObj(i -> executor.submit(() -> fetchData(i)))
        .toList();
}
// ⚠️ No thread pool sizing needed — create and forget

// ❌ DON'T: Pool virtual threads (defeats the purpose)
// Executors.newFixedThreadPool(100) with virtual threads = wrong
```

### ⚠️ Pitfalls / Gotchas
- **Don't pool virtual threads** — create per task, let JVM manage
- **synchronized blocks PIN** the virtual thread to carrier — use `ReentrantLock` instead
- **CPU-bound work** won't benefit — virtual threads excel at I/O waiting
- **ThreadLocal**: Works but each virtual thread gets its own copy — millions of copies = memory issue
- **Not faster per-thread** — they improve throughput by allowing more concurrent I/O operations

### ⚡ Remember
- Platform = 1:1 OS thread (1MB, ~5K max); Virtual = M:N (few KB, millions)
- Virtual threads **unmount on blocking** → carrier freed → massive concurrency
- Best for I/O-bound; useless for CPU-bound
- Don't pool them; don't use `synchronized` (pins carrier) — use `ReentrantLock`

### 🔗 Cross-references
- Java versions features → [core/17-java-versions-features.md](../languages/java/core/17-java-versions-features.md)

---

<a id="q9"></a>
## Q9. Explain sealed classes with a real-time example and their benefits.

### 📝 One-Liner
Sealed classes (Java 17) restrict which classes can extend them using `permits` — enabling exhaustive pattern matching and controlled inheritance hierarchies.

### 🔑 Quick Answer
`sealed class Shape permits Circle, Rectangle, Triangle` — only the permitted classes can extend Shape. Subclasses must be `final`, `sealed`, or `non-sealed`. Benefits: **Exhaustive switch** (compiler knows all subtypes — no default needed), **Domain modeling** (controlled type hierarchy), **API design** (prevent unauthorized extensions). Real-time example: payment methods (`CreditCard`, `UPI`, `NetBanking`) — you don't want random subclasses breaking your payment processing logic. *(Sealed class ek tarah ka "whitelist" hai — sirf permitted classes hi extend kar sakti hain, baaki compilation error)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Real-time example: Payment processing
public sealed interface PaymentMethod 
    permits CreditCard, UPI, NetBanking, Wallet {
    double amount();
}

public record CreditCard(String cardNumber, double amount, String cvv) 
    implements PaymentMethod {} // final by default (record)

public record UPI(String upiId, double amount) 
    implements PaymentMethod {}

public record NetBanking(String bankCode, double amount, String accountId) 
    implements PaymentMethod {}

public non-sealed class Wallet implements PaymentMethod {
    // non-sealed allows further extension (third-party wallets)
    private double amount;
    public double amount() { return amount; }
}
```

**Subclass modifiers:**

| Modifier | Meaning |
|----------|---------|
| `final` | Cannot be extended further (leaf node) |
| `sealed` | Can be extended but only by its own permitted list |
| `non-sealed` | Breaks the seal — anyone can extend (escape hatch) |

**Exhaustive pattern matching (Java 21):**
```java
// ✅ Compiler KNOWS all subtypes — no default case needed!
public double processFee(PaymentMethod pm) {
    return switch (pm) {
        case CreditCard cc -> cc.amount() * 0.02;   // 2% fee
        case UPI upi       -> 0;                      // no fee
        case NetBanking nb -> nb.amount() * 0.01;    // 1% fee
        case Wallet w      -> w.amount() * 0.005;    // 0.5% fee
        // No default needed! Compiler verifies exhaustive coverage
    };
}
```

### 🗣️ Answering Approach
"Sealed classes restrict which classes can extend them, declared with the permits clause. In our payment system, I use a sealed interface PaymentMethod permitting CreditCard, UPI, NetBanking, and Wallet. The key benefit is exhaustive pattern matching — when I write a switch on PaymentMethod, the compiler ensures I handle all subtypes. If someone adds a new payment type but forgets to update the switch, it's a compile error, not a runtime bug. Subclasses must be final, sealed, or non-sealed. We use non-sealed for Wallet to allow third-party wallet integrations while keeping the core types locked down."

### ⚠️ Pitfalls / Gotchas
- Permitted classes must be in the same package (or same module)
- `non-sealed` breaks exhaustive checking for that branch — use sparingly
- Records are implicitly `final` — perfect as sealed class implementations
- Sealed + Records + Pattern matching switch = powerful combination

### ⚡ Remember
- `sealed ... permits` = whitelist of allowed subtypes
- Subclass must be: `final` / `sealed` / `non-sealed`
- Killer feature: **exhaustive switch** → compile-time safety
- Real use cases: payment types, API responses, state machines, AST nodes

---

<a id="q10"></a>
## Q10. What design patterns are you familiar with?

### 📝 One-Liner
Name patterns you've actually used in projects with concrete examples — Singleton (config), Factory (notification channels), Strategy (login types), Observer (event listeners), Builder (complex objects).

### 🔑 Quick Answer
The patterns I've used in production: **Singleton** — Spring beans are singleton by default (ApplicationContext), **Factory** — NotificationFactory creates Email/SMS/Push based on type, **Strategy** — LoginStrategy for OTP/Password/SSO without if-else chains, **Observer** — Spring's @EventListener for order events (audit, email, inventory), **Builder** — Complex DTOs and query builders, **Adapter** — Wrapping third-party payment SDKs into our interface, **Template Method** — Base report generator with custom formatting in subclasses. *(Jo pattern use kiya hai wo example ke saath batao — sirf naam mat gino, interviewer immediately "give me an example" bolega)*

### 📖 How It Works (Detailed Explanation)

| Pattern | Category | Real Project Example |
|---------|----------|---------------------|
| **Singleton** | Creational | Spring beans, DB connection pool, Config manager |
| **Factory Method** | Creational | `PaymentGatewayFactory.create("razorpay")` |
| **Builder** | Creational | `User.builder().name("X").email("Y").build()` (Lombok) |
| **Strategy** | Behavioral | `LoginStrategy` — OTP, Password, SSO implementations |
| **Observer** | Behavioral | `@EventListener(OrderPlacedEvent.class)` → audit, email, inventory |
| **Template Method** | Behavioral | `AbstractReportGenerator` — subclasses override format() |
| **Adapter** | Structural | Wrap Stripe/Razorpay SDK behind `PaymentGateway` interface |
| **Decorator** | Structural | `BufferedInputStream(FileInputStream)` — add behavior |
| **Proxy** | Structural | Spring AOP — `@Transactional` creates proxy around method |

### 🗣️ Answering Approach
"I use several patterns regularly. Strategy pattern is my go-to for replacing if-else chains — in our login module, we have OTP, Password, and SSO strategies selected at runtime. Factory pattern for creating service instances — our NotificationFactory returns Email, SMS, or Push based on user preference. Observer via Spring's EventListener — when an order is placed, audit logging, email notification, and inventory update all react independently. Builder for complex DTOs — especially with Lombok's @Builder. And Proxy is used implicitly by Spring — every @Transactional annotation creates an AOP proxy. I pick patterns when they solve a real problem, not to show off design skills."

### ⚡ Remember
- Name 4-5 patterns with **your project examples** — not textbook definitions
- **Strategy** = replace if-else; **Factory** = create without knowing exact type
- **Observer** = Spring Events; **Proxy** = Spring AOP / @Transactional
- Anti-pattern: Don't use patterns just because you can — YAGNI applies

---

<a id="q11"></a>
## Q11. Explain OOP principles.

### 📝 One-Liner
Four pillars: **Encapsulation** (hide data), **Abstraction** (hide complexity), **Inheritance** (reuse behavior), **Polymorphism** (same interface, different behavior) — plus SOLID for design quality.

### 🔑 Quick Answer
**Encapsulation** — Bundle data + methods together, expose through controlled access (private fields + public getters). **Abstraction** — Hide internal complexity, expose only what's needed (interfaces, abstract classes). **Inheritance** — Child class reuses parent's behavior, extends or overrides as needed (`extends`, `implements`). **Polymorphism** — Same method call, different behavior at runtime (method overriding) or compile time (method overloading). In Spring Boot: Encapsulation = Service layer hides DB access. Abstraction = Repository interface. Inheritance = BaseEntity for common fields. Polymorphism = multiple implementations of NotificationService. *(Encapsulation = data chhupao, Abstraction = complexity chhupao, Inheritance = code reuse karo, Polymorphism = ek interface, alag behavior)*

### 📖 How It Works (Detailed Explanation)

| Principle | What | Spring Boot Example |
|-----------|------|-------------------|
| **Encapsulation** | Private fields + public methods | `@Entity` with private fields, public getters |
| **Abstraction** | Interface / abstract class | `JpaRepository<User, Long>` — hides SQL |
| **Inheritance** | `extends` / `implements` | `BaseEntity` with `id`, `createdAt`, `updatedAt` |
| **Polymorphism** | Same interface, multiple implementations | `NotificationService` → EmailNotification, SMSNotification |

**SOLID Principles (bonus — often asked as follow-up):**

| Principle | Meaning | Example |
|-----------|---------|---------|
| **S**ingle Responsibility | One class = one reason to change | `OrderService` doesn't handle email |
| **O**pen-Closed | Open for extension, closed for modification | Strategy pattern for new payment types |
| **L**iskov Substitution | Subtypes replaceable for parent type | `List<Shape>` works with Circle, Rectangle |
| **I**nterface Segregation | Small, focused interfaces | `Readable` and `Writable` instead of `ReadWritable` |
| **D**ependency Inversion | Depend on abstractions, not concretions | `@Autowired NotificationService` (interface) |

### 🗣️ Answering Approach
"The four OOP pillars are Encapsulation, Abstraction, Inheritance, and Polymorphism. In my Spring Boot project: Encapsulation — our entities have private fields with validation in setters. Abstraction — repository interfaces hide database implementation; service layer hides business logic complexity. Inheritance — we use a BaseEntity with common fields like id, createdAt, updatedAt that all entities extend. Polymorphism — our NotificationService interface has Email, SMS, and Push implementations. Spring's DI + IoC is essentially dependency inversion from SOLID — we code to interfaces, Spring injects the right implementation. I follow SOLID principles as well — Single Responsibility means my OrderService doesn't handle email notifications, that's delegated to NotificationService."

### ⚡ Remember
- **E-A-I-P**: Encapsulation → Abstraction → Inheritance → Polymorphism
- Spring Boot examples make your answer stand out — don't give textbook `Animal extends Dog`
- SOLID is the follow-up — prepare at least S, O, D with real examples
- Polymorphism = compile-time (overloading) + runtime (overriding)

### 🔗 Cross-references
- OOP principles deep dive → [oops-patterns/01-oop-principles.md](../oops-patterns/01-oop-principles.md)
- OOP real-project examples → [oops-patterns/02-oop-real-project-examples.md](../oops-patterns/02-oop-real-project-examples.md)

---

[← Back to Company Specific](./README.md) | [← Back to Home](../../README.md)
