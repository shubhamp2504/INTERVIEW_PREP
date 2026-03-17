# ☕ Core Java — Immutable Classes, Comparable/Comparator & String Internals (Q1–Q3)

> **Source**: EPAM Systems Java Backend Interview  
> **Coverage**: Immutability design, sorting contracts, String pool & mutability

---

<a id="q1"></a>
## Q1. How do you create an immutable class in Java? Why is immutability important?

### 📝 One-Liner
An immutable class is one whose state **cannot change after construction** — achieved by making fields `final`, no setters, class `final`, and defensive copies.

### 🔑 Quick Answer
**(1)** Declare class `final` (prevent subclass override). **(2)** All fields `private final`. **(3)** No setter methods. **(4)** Initialize all fields via constructor. **(5)** If a field is a mutable object (e.g., `List`, `Date`), return **defensive copies** from getters — never expose the internal reference. **Why?** Thread-safe without synchronization (no shared mutable state), safe as HashMap keys (hashCode never changes), simpler reasoning, cacheable. Java examples: `String`, `Integer`, `LocalDate`. *(Immutable class = ek baar banaao, phir kabhi change nahi hoga — thread-safe by design)*

### 📖 How It Works (Detailed Explanation)

```
Immutable Class Contract:
┌─────────────────────────────────────────┐
│  final class Employee {                 │
│    private final String name;           │  ← private + final
│    private final List<String> skills;   │
│                                         │
│    // Constructor: deep copy mutable    │
│    Employee(String n, List<String> s) { │
│      this.name = n;                     │
│      this.skills = List.copyOf(s);      │  ← defensive copy IN
│    }                                    │
│                                         │
│    // Getter: return unmodifiable        │
│    List<String> getSkills() {           │
│      return Collections                 │
│        .unmodifiableList(skills);       │  ← defensive copy OUT
│    }                                    │
│  }                                      │
└─────────────────────────────────────────┘
```

**Key rules**: (1) Class is `final` — if a subclass overrides a getter to return a mutable reference, immutability breaks. (2) Fields are `private final` — no direct access, no reassignment. (3) No setters — state set only in constructor. (4) Defensive copies on the way IN (constructor) and OUT (getters) for any mutable field. If your immutable class holds a `Date` or `List`, always copy it. Java 16+ `record` classes are close to immutable but NOT automatically — mutable fields in records still need defensive copies.

### 🗣️ Interview Script
"An immutable class is one where the state can never change after construction. I follow five rules: make the class final to prevent subclassing, all fields private final, no setters, initialize everything in the constructor, and — the one most people forget — defensive copies for any mutable fields. If my class holds a List, I use List.copyOf in the constructor and return Collections.unmodifiableList from the getter. String is the classic example — its immutability enables the String pool, safe HashMap keys, and thread safety without locks. In my projects I use immutability for DTOs, value objects, and configuration classes — they're simpler to reason about and inherently thread-safe."

### 💻 Code Example

```java
// ✅ Properly immutable class
public final class Money {
    private final BigDecimal amount;
    private final Currency currency;
    private final List<String> tags;

    public Money(BigDecimal amount, Currency currency, List<String> tags) {
        this.amount = amount;              // BigDecimal is already immutable
        this.currency = currency;          // Currency is already immutable
        this.tags = List.copyOf(tags);     // ⭐ defensive copy of mutable List
    }

    public BigDecimal getAmount()   { return amount; }
    public Currency getCurrency()   { return currency; }
    public List<String> getTags()   { return tags; }  // List.copyOf returns unmodifiable

    // ❌ NEVER do this:
    // public List<String> getTags() { return this.tags; }  ← if tags were ArrayList
}

// ❌ Broken immutability - common mistake
public final class BadEvent {
    private final Date timestamp;

    public BadEvent(Date timestamp) {
        this.timestamp = timestamp;  // ❌ caller can mutate: timestamp.setTime(0)
    }

    public Date getTimestamp() {
        return timestamp;            // ❌ caller can mutate the returned Date
    }
}

// ✅ Fixed
public final class GoodEvent {
    private final Date timestamp;

    public GoodEvent(Date timestamp) {
        this.timestamp = new Date(timestamp.getTime());  // ✅ defensive copy IN
    }

    public Date getTimestamp() {
        return new Date(timestamp.getTime());            // ✅ defensive copy OUT
    }
}

// Java 16+ Record — close but NOT automatically immutable for mutable fields
public record OrderInfo(String orderId, List<String> items) {
    public OrderInfo {   // compact constructor
        items = List.copyOf(items);  // ⭐ still need defensive copy!
    }
}
```

### ⚠️ Common Pitfalls
- Forgetting **defensive copies** — `Date`, `List`, `Map` fields can be mutated externally
- Not making class `final` — subclass can break immutability by overriding getters
- Using `Arrays.asList()` — it's a **fixed-size** list but elements are still mutable references
- Thinking `record` = immutable — records with mutable fields still need defensive copies
- Exposing mutable internal state through **reflection** — rare but possible (use Security Manager in sensitive contexts)

### 🆚 Comparison Table

| Aspect | Mutable Class | Immutable Class |
|--------|--------------|-----------------|
| Thread safety | Needs synchronization | Thread-safe by default |
| HashMap key | Risky (hashCode can change) | Safe (hashCode stable) |
| Defensive copies | Not needed | Required for mutable fields |
| Memory | One object, mutate in place | New object per change |
| Examples | `ArrayList`, `Date`, `StringBuilder` | `String`, `Integer`, `LocalDate` |

### 🎯 Tricky Follow-up Questions
- **Q**: Can you make an immutable class with a mutable field without copying?  
  **A**: Use `Collections.unmodifiableList()` wrapper — but the original list reference can still be mutated by the caller. `List.copyOf()` is safer because it creates a true copy.
- **Q**: Why is String immutable in Java?  
  **A**: String pool optimization (reuse), safe as HashMap key, thread-safe, security (class names, URLs, DB connections are Strings).

### ⚡ Remember (Quick Recall)
- 5 rules: `final` class + `private final` fields + no setters + constructor init + defensive copies
- `List.copyOf()` > `Collections.unmodifiableList()` for constructor defense
- `String`, `Integer`, `LocalDate` = immutable; `Date`, `ArrayList` = mutable
- Java records need **explicit defensive copies** for mutable fields

### 🔗 Follow-up Topics
- [Q1 in core/01 → Thread-safe data structures (related: immutability as strategy)](#)
- String pool and `intern()` mechanics
- Value Objects in DDD (always immutable)

---

<a id="q2"></a>
## Q2. What is the difference between Comparable and Comparator? When would you use each?

### 📝 One-Liner
`Comparable` defines the **natural ordering** inside the class itself (`compareTo`); `Comparator` is an **external** sorting strategy passed to sort methods (`compare`).

### 🔑 Quick Answer
`Comparable<T>` → class implements `compareTo(T o)` → defines ONE natural order (e.g., Employee sorted by ID). `Comparator<T>` → separate class/lambda implementing `compare(T a, T b)` → defines ANY number of custom orderings without modifying the original class. Use Comparable for the obvious default sort; use Comparator for alternate sorts, third-party classes you can't modify, or complex multi-field sorting with `Comparator.comparing().thenComparing()`. *(Comparable = class ke andar ek default order, Comparator = bahar se multiple orders define karo)*

### 📖 How It Works (Detailed Explanation)

```
Comparable (natural order — inside class):
┌──────────────────────────────────────┐
│ class Employee implements            │
│        Comparable<Employee> {        │
│   int id;                            │
│   compareTo(Employee o) {            │
│     return this.id - o.id;  ← ONE   │
│   }                                  │
│ }                                    │
│ Collections.sort(list);  ← uses it  │
└──────────────────────────────────────┘

Comparator (custom order — external):
┌──────────────────────────────────────┐
│ Comparator<Employee> byName =        │
│   Comparator.comparing(              │
│     Employee::getName);              │
│                                      │
│ list.sort(byName);  ← any order     │
│ list.sort(byName.reversed());        │
│ list.sort(byName                     │
│   .thenComparing(Employee::getId));  │
└──────────────────────────────────────┘
```

**Contract**: `compareTo` must return negative (this < other), zero (equal), or positive (this > other). Must be consistent with `equals()` — if `a.compareTo(b) == 0` then ideally `a.equals(b) == true` (TreeSet/TreeMap rely on this). **Transitivity**: if a > b and b > c, then a > c. **Integer overflow trap**: `return this.id - o.id` can overflow if values are large — safer: `Integer.compare(this.id, o.id)`.

### 🗣️ Interview Script
"Comparable defines the natural ordering of a class — the class itself implements compareTo to define one default sort order. I use it when there's an obvious way to sort — like employees by ID or dates chronologically. Comparator is external — it lets me define multiple different orderings without modifying the original class. With Java 8's Comparator.comparing and thenComparing, I chain multi-field sorts fluently. I prefer Comparator for flexibility — sorting by name, then by salary, then by hire date — all without touching the entity class. One important gotcha: never use subtraction for compareTo with integers — it can overflow. Always use Integer.compare or Comparator.comparingInt."

### 💻 Code Example

```java
// ✅ Comparable — natural ordering (inside class)
public class Employee implements Comparable<Employee> {
    private int id;
    private String name;
    private double salary;

    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.id, other.id);  // ⭐ safe, no overflow
        // ❌ return this.id - other.id;  ← overflow risk for large values!
    }
}

// Usage: natural sort
List<Employee> employees = getEmployees();
Collections.sort(employees);          // uses compareTo
TreeSet<Employee> sorted = new TreeSet<>(employees);  // uses compareTo

// ✅ Comparator — external custom ordering
Comparator<Employee> byName = Comparator.comparing(Employee::getName);
Comparator<Employee> bySalaryDesc = Comparator.comparingDouble(Employee::getSalary).reversed();
Comparator<Employee> byNameThenSalary = Comparator.comparing(Employee::getName)
        .thenComparingDouble(Employee::getSalary);

employees.sort(byName);               // sort by name
employees.sort(bySalaryDesc);         // sort by salary descending
employees.sort(byNameThenSalary);     // multi-field sort

// ✅ Null-safe comparator
Comparator<Employee> byNameNullSafe = Comparator.comparing(
    Employee::getName, Comparator.nullsLast(Comparator.naturalOrder())
);

// ✅ Sorting third-party class you can't modify
List<Transaction> txns = getTransactions();  // Transaction doesn't implement Comparable
txns.sort(Comparator.comparing(Transaction::getTimestamp).reversed());

// ✅ Stream sorting with Comparator
employees.stream()
    .sorted(Comparator.comparing(Employee::getName))
    .collect(Collectors.toList());
```

### ⚠️ Common Pitfalls
- **Integer overflow** in `compareTo`: `return a - b` overflows for `Integer.MAX_VALUE - (-1)` → use `Integer.compare(a, b)`
- **Inconsistency with equals**: `BigDecimal("1.0").compareTo(BigDecimal("1.00")) == 0` but `equals()` returns `false` → TreeSet treats them as same, HashSet treats them as different
- **Not handling nulls** — `compareTo` with null field → NullPointerException → use `Comparator.nullsFirst/nullsLast`
- **Mutating sort key** after insertion into TreeSet/TreeMap → breaks internal BST structure

### 🆚 Comparison Table

| Aspect | Comparable | Comparator |
|--------|-----------|------------|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T o)` | `compare(T a, T b)` |
| # of orderings | ONE (natural) | MANY (unlimited) |
| Modifies class? | Yes (implements interface) | No (external) |
| Lambda support | No (single method in class) | Yes (`(a, b) -> ...`) |
| Null handling | Manual | `nullsFirst()` / `nullsLast()` |
| Chaining | Not possible | `thenComparing()` |
| Use case | Default sort (ID, date) | Custom/multiple sorts |

### 🎯 Tricky Follow-up Questions
- **Q**: What happens if `compareTo` is inconsistent with `equals`?  
  **A**: TreeSet/TreeMap use `compareTo` for equality — elements considered equal by `compareTo` but not `equals` will be deduplicated in TreeSet but not in HashSet. BigDecimal is the classic example.
- **Q**: Can a class implement Comparable and also have Comparators?  
  **A**: Yes — Comparable defines the default; Comparators provide alternatives. `String` implements Comparable (lexicographic) but also has `String.CASE_INSENSITIVE_ORDER` comparator.

### ⚡ Remember (Quick Recall)
- **Comparable** = "I can compare **myself**" (natural order, `java.lang`)
- **Comparator** = "**Someone else** compares me" (custom order, `java.util`)
- Always use `Integer.compare()`, never subtraction
- `Comparator.comparing().thenComparing()` for multi-field sorts
- TreeSet/TreeMap rely on `compareTo` — must be consistent with `equals`

### 🔗 Follow-up Topics
- TreeMap/TreeSet internals (Red-Black tree, compareTo-based)
- Java 8 Comparator factory methods (`comparing`, `naturalOrder`, `reverseOrder`)
- Sorting stability in Java (TimSort is stable — equal elements keep original order)

---

<a id="q3"></a>
## Q3. What is the difference between String, StringBuilder, and StringBuffer?

### 📝 One-Liner
`String` is **immutable** (every change creates a new object); `StringBuilder` is **mutable + not thread-safe** (fast); `StringBuffer` is **mutable + thread-safe** (synchronized, slower).

### 🔑 Quick Answer
`String` — immutable, stored in String pool, every concatenation creates a new String object → bad in loops. `StringBuilder` — mutable character sequence, NOT synchronized → use in single-threaded code (99% of cases). `StringBuffer` — same API as StringBuilder but all methods are `synchronized` → thread-safe but slower → rarely needed (use StringBuilder + external sync if needed). **Rule**: String for constants/small operations, StringBuilder for building strings in loops, StringBuffer almost never. *(String = badal nahi sakta, StringBuilder = fast + mutable, StringBuffer = synchronized lekin slow)*

### 📖 How It Works (Detailed Explanation)

```
String (immutable):
  String s = "hello";
  s = s + " world";
  ┌─────────┐     ┌─────────────┐
  │ "hello" │     │"hello world"│  ← NEW object created
  └─────────┘     └─────────────┘
  (old object becomes garbage)

StringBuilder (mutable, no sync):
  StringBuilder sb = new StringBuilder("hello");
  sb.append(" world");
  ┌────────────────────┐
  │ h│e│l│l│o│ │w│o│r│l│d│  ← SAME object, expanded buffer
  └────────────────────┘

StringBuffer (mutable, synchronized):
  Same as StringBuilder but every method has:
  public synchronized StringBuffer append(String s) { ... }
```

**String pool**: literal Strings like `"hello"` are interned in the pool (one copy shared). `new String("hello")` creates a separate heap object. **StringBuilder internals**: backed by a `char[]` (Java 8-) or `byte[]` (Java 9+ compact strings). Default capacity = 16. When full, grows by `(oldCapacity * 2) + 2`. Pre-size with `new StringBuilder(expectedSize)` to avoid resizing. **Java compiler optimization**: simple concatenations like `"a" + "b" + "c"` at compile time → compiler optimizes. But **loop concatenation** `s += x` in a loop creates a new StringBuilder per iteration → O(n²).

### 🗣️ Interview Script
"String is immutable — every modification creates a new object, which is fine for small operations but terrible in loops where you're concatenating thousands of times — that's O(n²) because each concatenation copies the entire string. StringBuilder is mutable — it maintains an internal character buffer that grows as needed, so appending is amortized O(1). StringBuffer has the same API but every method is synchronized, which adds overhead. In practice, I use StringBuilder exclusively — I've never needed StringBuffer because if I need thread-safe string building, I'd use a StringBuilder within a synchronized block or ThreadLocal rather than paying synchronization cost on every single append call. One optimization tip: if I know the approximate final size, I pass it to the StringBuilder constructor to avoid buffer resizing."

### 💻 Code Example

```java
// ❌ BAD: String concatenation in loop — O(n²)
String result = "";
for (int i = 0; i < 100_000; i++) {
    result += i + ",";  // creates new String + new StringBuilder EACH iteration
}
// ~5000ms for 100K iterations

// ✅ GOOD: StringBuilder — O(n)
StringBuilder sb = new StringBuilder(700_000);  // pre-size if you can estimate
for (int i = 0; i < 100_000; i++) {
    sb.append(i).append(",");
}
String result = sb.toString();
// ~5ms for 100K iterations — 1000x faster!

// StringBuffer — same API but synchronized (rarely needed)
StringBuffer sbuf = new StringBuffer();
sbuf.append("thread-safe").append(" but slower");

// String pool demonstration
String s1 = "hello";           // pool
String s2 = "hello";           // same pool reference
String s3 = new String("hello"); // heap (separate object)
System.out.println(s1 == s2);     // true  (same pool reference)
System.out.println(s1 == s3);     // false (different objects)
System.out.println(s1.equals(s3)); // true  (same content)

// Java 8+ StringJoiner (cleaner alternative)
StringJoiner sj = new StringJoiner(", ", "[", "]");
sj.add("a").add("b").add("c");
// Output: [a, b, c]

// Collectors.joining (Stream)
String csv = List.of("a", "b", "c").stream()
    .collect(Collectors.joining(", "));
// Output: a, b, c
```

### ⚠️ Common Pitfalls
- **Loop concatenation** with `+=` — creates new StringBuilder per iteration → O(n²)
- **StringBuffer over StringBuilder** — paying sync cost for no reason in single-threaded code
- `new String("literal")` — creating unnecessary heap object; just use `"literal"`
- **Not pre-sizing StringBuilder** — if final string is ~1MB, default 16-char buffer resizes many times
- **String.intern()** abuse — interning too many strings pollutes the String pool (which uses native memory)

### 🆚 Comparison Table

| Aspect | String | StringBuilder | StringBuffer |
|--------|--------|--------------|-------------|
| Mutability | Immutable | Mutable | Mutable |
| Thread-safe | Yes (immutable) | **No** | Yes (synchronized) |
| Performance | Slow for modification | **Fastest** | Slower (sync overhead) |
| Storage | String pool + heap | Heap only | Heap only |
| Since | JDK 1.0 | JDK 1.5 | JDK 1.0 |
| Use when | Constants, small ops | **Loop building** (99%) | Multi-thread append (rare) |

### 🎯 Tricky Follow-up Questions
- **Q**: Does the compiler optimize `"a" + "b" + "c"`?  
  **A**: Yes — compile-time constants are folded: `"a" + "b" + "c"` becomes `"abc"` in bytecode. But runtime variables like `s1 + s2` use `StringBuilder` (Java 8) or `invokedynamic StringConcatFactory` (Java 9+).
- **Q**: Is String really thread-safe?  
  **A**: Yes, because it's immutable — there's no shared mutable state to corrupt. Multiple threads can safely read the same String. This is *why* it's immutable.

### ⚡ Remember (Quick Recall)
- **String** = immutable, pool-able, bad in loops
- **StringBuilder** = mutable, fast, use 99% of the time
- **StringBuffer** = synchronized StringBuilder, almost never needed
- Loop `+=` with String → O(n²); StringBuilder.append() → O(n)
- Pre-size: `new StringBuilder(estimatedSize)` avoids resizing
- Java 9+: `invokedynamic` for string concat (replaces StringBuilder internally)

### 🔗 Follow-up Topics
- String pool and `intern()` mechanics
- Java 9 Compact Strings (byte[] instead of char[])
- [Q1 → Immutable class design (String as example)](#q1)
