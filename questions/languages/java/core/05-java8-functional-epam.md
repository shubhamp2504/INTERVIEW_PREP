# ☕ Java 8+ — Stream API, Functional Interfaces & Optional (Q1–Q5)

> **Source**: EPAM Systems Java Backend Interview  
> **Coverage**: Stream API overview, map/flatMap, Functional Interface, Predicate/Function/Consumer, Optional

---

<a id="q1"></a>
## Q1. What is the Stream API in Java 8 and why is it useful?

### 📝 One-Liner
Stream API provides a **declarative, pipeline-based** way to process collections — filter, map, reduce — with built-in support for **lazy evaluation** and **parallel processing**.

### 🔑 Quick Answer
A `Stream` is NOT a data structure — it's a **pipeline of operations** on a data source (Collection, array, I/O). **Three parts**: **(1)** Source (`list.stream()`), **(2)** Intermediate operations (lazy — `filter`, `map`, `sorted`, `distinct`, `flatMap`), **(3)** Terminal operation (triggers execution — `collect`, `forEach`, `reduce`, `count`). **Key benefits**: declarative code (what, not how), lazy evaluation (intermediate ops don't execute until terminal op), easy parallelism (`parallelStream()`), and method chaining for readability. *(Stream = collection pe operations ka pipeline — lazy hai, terminal operation pe hi execute hota hai)*

### 📖 How It Works (Detailed Explanation)

```
Stream Pipeline:
┌──────────┐    ┌──────────┐    ┌──────────┐    ┌──────────┐
│  Source   │ →  │  filter  │ →  │   map    │ →  │ collect  │
│ list.     │    │ (lazy)   │    │ (lazy)   │    │(terminal)│
│ stream()  │    │          │    │          │    │ triggers │
└──────────┘    └──────────┘    └──────────┘    └──────────┘
                  Intermediate    Intermediate     Terminal
                  (not executed)  (not executed)   (executes ALL)
```

**Lazy evaluation**: when you write `.filter().map().sorted()`, NOTHING happens. Only when the terminal operation (`.collect()`, `.forEach()`) is called does the pipeline execute — and it processes elements **one at a time** through the entire chain (not filter ALL, then map ALL). This is called **loop fusion**. **Short-circuiting**: operations like `findFirst()`, `limit()`, `anyMatch()` stop processing early. **Streams are single-use** — after a terminal operation, the stream is consumed and cannot be reused.

### 🗣️ Interview Script
"The Stream API is a declarative way to process data pipelines on collections. Instead of writing imperative for-loops with mutable accumulators, I express transformations as a chain of filter, map, and collect operations. The key insight is lazy evaluation — intermediate operations like filter and map don't execute immediately. They build up a pipeline that only runs when a terminal operation like collect or forEach is called. This enables optimizations like loop fusion, where each element flows through the entire pipeline before the next element is processed, and short-circuiting, where findFirst stops after the first match. For CPU-intensive operations on large datasets, I can switch to parallelStream for automatic fork-join parallelism. In my projects, Streams make data transformation code much more readable and less error-prone than manual loops."

### 💻 Code Example

```java
// ✅ Stream pipeline — filter, map, collect
List<String> activeUserEmails = users.stream()
    .filter(u -> u.isActive())                     // intermediate (lazy)
    .filter(u -> u.getAge() > 18)                  // intermediate (lazy)
    .map(User::getEmail)                           // intermediate (lazy)
    .sorted()                                      // intermediate (lazy)
    .distinct()                                    // intermediate (lazy)
    .collect(Collectors.toList());                  // terminal (triggers all)

// ✅ Reduce — aggregate to single value
int totalSalary = employees.stream()
    .mapToInt(Employee::getSalary)
    .sum();

// ✅ Short-circuiting — stops early
Optional<User> firstAdmin = users.stream()
    .filter(u -> u.getRole() == Role.ADMIN)
    .findFirst();  // stops after first match — doesn't process entire list

// ✅ Grouping with Collectors
Map<Department, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::getDepartment));

// ✅ Parallel stream for CPU-intensive work
long count = largeList.parallelStream()
    .filter(this::expensiveValidation)
    .count();

// ❌ Stream is single-use
Stream<String> stream = names.stream().filter(n -> n.length() > 3);
stream.forEach(System.out::println);  // OK
stream.forEach(System.out::println);  // ❌ IllegalStateException: stream already operated upon

// ✅ Creating streams from different sources
Stream.of("a", "b", "c");                    // from values
Arrays.stream(new int[]{1, 2, 3});           // from array
IntStream.range(1, 100);                     // numeric range
Files.lines(Path.of("data.txt"));            // from file (lazy, line-by-line)
Stream.generate(Math::random).limit(10);     // infinite stream (capped)
```

### ⚠️ Common Pitfalls
- **Reusing a stream** — consumed streams throw `IllegalStateException`
- **Side effects in intermediate operations** — `map()` should be pure; don't modify external state in lambdas
- **parallelStream() for I/O** — parallel streams use common ForkJoinPool → blocking I/O starves other tasks → use virtual threads or custom executor
- **forEach vs collect** — `forEach` is for side effects; use `collect` when building a result
- **Infinite streams without limit** — `Stream.generate()` without `limit()` runs forever

### 🆚 Comparison Table

| Aspect | For-Loop | Stream API |
|--------|---------|------------|
| Style | Imperative (how) | Declarative (what) |
| Mutability | External variables | Pipeline (no mutation) |
| Parallelism | Manual threading | `.parallelStream()` |
| Lazy evaluation | No | Yes (intermediate ops) |
| Readability | Verbose for transforms | Concise chaining |
| Debugging | Easy (breakpoints) | Harder (use peek()) |
| Performance | Slightly faster (no overhead) | Slight overhead, but negligible |

### 🎯 Tricky Follow-up Questions
- **Q**: When should you NOT use Streams?  
  **A**: Simple iterations, when you need index access, when performance is critical in tight loops (boxing overhead for primitives), or when side effects are the main goal (use for-loop for clarity).
- **Q**: What is loop fusion?  
  **A**: Instead of applying filter to ALL elements, then map to ALL — each element flows through the entire pipeline before the next. Filter + map on element 1, then element 2, etc.

### ⚡ Remember (Quick Recall)
- Stream = Source → Intermediate (lazy) → Terminal (triggers)
- **Lazy** — nothing happens until terminal operation
- **Single-use** — consumed after terminal op
- `parallelStream()` → ForkJoinPool → avoid for I/O
- `collect` for results, `forEach` for side effects

### 🔗 Follow-up Topics
- [Q2 → map() vs flatMap()](#q2)
- [Q3 → Functional Interface (lambda foundation)](#q3)
- [Q1 in core/02 → For Loop vs Stream performance benchmark](../core/02-streams-serialization.md#q1)

---

<a id="q2"></a>
## Q2. What is the difference between map() and flatMap() in Streams?

### 📝 One-Liner
`map()` transforms each element **one-to-one** (returns one value); `flatMap()` transforms each element to a **stream** and **flattens** all results into a single stream.

### 🔑 Quick Answer
`map(T → R)` — applies a function to each element, returning one output per input. `flatMap(T → Stream<R>)` — applies a function that returns a Stream for each element, then **merges all those streams** into one flat stream. Use `map` for simple transformations (`User → name`); use `flatMap` when each element maps to **multiple elements** (e.g., `Order → List<Items>` → flat list of all items). Also essential with `Optional.flatMap` to avoid `Optional<Optional<T>>`. *(map = ek input se ek output, flatMap = ek input se multiple outputs jo flat ho jaate hain)*

### 📖 How It Works (Detailed Explanation)

```
map():
[Order1, Order2] → map(o → o.getItems())
Result: [List[A,B], List[C,D]]  ← Stream<List<Item>> (nested!)

flatMap():
[Order1, Order2] → flatMap(o → o.getItems().stream())
Result: [A, B, C, D]  ← Stream<Item> (flattened!)
```

**map** wraps results — if the function returns a collection, you get a Stream of collections. **flatMap** unwraps — it expects the function to return a Stream, then concatenates all those streams. Think of it as `map` + `flatten` in one step. This is the **monad bind** operation in functional programming.

### 🗣️ Interview Script
"map transforms each element one-to-one — like mapping a User to their email. flatMap is for one-to-many transformations where you want a flattened result. If each order has a list of items and I use map to get items, I'd get a Stream of Lists. With flatMap, I return a stream for each order's items, and they all merge into a single flat stream of items. I use flatMap frequently — flattening nested collections, processing lines of files where each line splits into words, and with Optional to chain methods that each return Optional without getting nested Optional of Optional."

### 💻 Code Example

```java
// ✅ map — one-to-one transformation
List<String> names = users.stream()
    .map(User::getName)          // User → String
    .collect(Collectors.toList());
// Input:  [User("Alice"), User("Bob")]
// Output: ["Alice", "Bob"]

// ❌ map with nested collections — produces Stream<List<Item>>
List<List<Item>> nested = orders.stream()
    .map(Order::getItems)        // Order → List<Item>
    .collect(Collectors.toList());
// Result: [[Item1, Item2], [Item3]]  ← still nested!

// ✅ flatMap — flattens nested collections
List<Item> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())  // Order → Stream<Item>
    .collect(Collectors.toList());
// Result: [Item1, Item2, Item3]  ← flat!

// ✅ flatMap — split lines into words
List<String> words = lines.stream()
    .flatMap(line -> Arrays.stream(line.split("\\s+")))
    .collect(Collectors.toList());
// ["hello world", "foo bar"] → ["hello", "world", "foo", "bar"]

// ✅ Optional.map vs Optional.flatMap
Optional<String> city = user.map(User::getAddress)     // Optional<Address>
                            .map(Address::getCity);     // Optional<String> ✅

// If getAddress returns Optional<Address>:
Optional<String> city = user.flatMap(User::getAddress)  // avoids Optional<Optional<Address>>
                            .map(Address::getCity);

// ✅ Practical: get all unique tags from all products
Set<String> allTags = products.stream()
    .flatMap(p -> p.getTags().stream())
    .collect(Collectors.toSet());

// ✅ flatMap with Stream.of for conditional expansion
List<String> expanded = items.stream()
    .flatMap(item -> item.isBundle() 
        ? item.getSubItems().stream().map(Item::getName) 
        : Stream.of(item.getName()))
    .collect(Collectors.toList());
```

### ⚠️ Common Pitfalls
- Using `map` when you need `flatMap` → get `Stream<List<T>>` instead of `Stream<T>`
- Forgetting `.stream()` inside `flatMap` lambda — `flatMap` expects a `Stream`, not a `List`
- **Optional.map vs flatMap**: if your function already returns `Optional`, use `flatMap` to avoid `Optional<Optional<T>>`
- **Performance**: `flatMap` creates intermediate streams per element — for very large datasets, consider manual loops

### 🆚 Comparison Table

| Aspect | map() | flatMap() |
|--------|-------|-----------|
| Transformation | One-to-one | One-to-many |
| Return type | `Stream<R>` | `Stream<R>` (flattened) |
| Function signature | `T → R` | `T → Stream<R>` |
| Nesting | Can produce nested structures | Flattens nested structures |
| Use case | Simple field extraction | Collections of collections |
| Optional | `T → R` | `T → Optional<R>` |

### 🎯 Tricky Follow-up Questions
- **Q**: Can flatMap return an empty stream?  
  **A**: Yes — `flatMap(x -> Stream.empty())` effectively filters out elements. This is flatMap as a combined filter+map.
- **Q**: What's the difference between `flatMap` in Streams vs `flatMap` in Optional?  
  **A**: Same concept — Stream.flatMap flattens `Stream<Stream<T>>` → `Stream<T>`. Optional.flatMap prevents `Optional<Optional<T>>` → `Optional<T>`.

### ⚡ Remember (Quick Recall)
- `map` = **1:1** transform, `flatMap` = **1:N** transform + flatten
- flatMap = `map` + `flatten` in one step
- flatMap lambda must return a **Stream** (not List)
- Optional.flatMap when function returns Optional (avoids nesting)
- Empty stream in flatMap = filter effect

### 🔗 Follow-up Topics
- [Q1 → Stream API overview](#q1)
- [Q5 → Optional (flatMap with Optional)](#q5)
- Reactive Streams (Mono.flatMap, Flux.flatMap — same concept)

---

<a id="q3"></a>
## Q3. What is a Functional Interface in Java?

### 📝 One-Liner
A functional interface has **exactly one abstract method** (SAM) and can be used as the target for **lambda expressions** and **method references**.

### 🔑 Quick Answer
Any interface with a single abstract method (SAM = Single Abstract Method) is a functional interface. Annotate with `@FunctionalInterface` (optional but recommended — compiler enforces). Default and static methods don't count. Examples: `Runnable` (run), `Callable` (call), `Comparator` (compare), `Function<T,R>` (apply), `Predicate<T>` (test), `Consumer<T>` (accept), `Supplier<T>` (get). Lambdas are just **anonymous implementations** of functional interfaces. *(Functional Interface = ek hi abstract method — lambda ka foundation hai)*

### 📖 How It Works (Detailed Explanation)

```
Functional Interface Contract:
┌────────────────────────────────────────┐
│ @FunctionalInterface                   │
│ interface Transformer<T, R> {          │
│   R transform(T input);   ← SAM (1)   │
│                                        │
│   // These DON'T count:                │
│   default void log() { }  ← default   │
│   static void util() { }  ← static    │
│ }                                      │
└────────────────────────────────────────┘

Usage:
  Transformer<String, Integer> t = s -> s.length();  ← lambda
  Transformer<String, Integer> t = String::length;    ← method ref
```

**Why @FunctionalInterface?** Compiler validation — if someone adds a second abstract method, compilation fails. It's like `@Override` — documentation + safety. **Lambda desugaring**: the compiler converts lambdas to `invokedynamic` bytecode instructions (not anonymous classes) → more efficient, no extra .class file. **Method references** are shorthand: `String::length` is equivalent to `s -> s.length()`.

### 🗣️ Interview Script
"A functional interface has exactly one abstract method — the Single Abstract Method contract. This is what enables lambdas in Java. When I write a lambda expression, the compiler matches it to a functional interface that has a compatible method signature. I always annotate with @FunctionalInterface so the compiler enforces the contract — if a team member accidentally adds a second abstract method, it won't compile. Java 8 ships with key functional interfaces in java.util.function — Function for transformation, Predicate for filtering, Consumer for side effects, and Supplier for lazy creation. Under the hood, lambdas use invokedynamic for efficient linking rather than generating anonymous inner classes."

### 💻 Code Example

```java
// ✅ Built-in functional interfaces
Function<String, Integer>  fn = String::length;       // T → R
Predicate<String>          p  = s -> s.length() > 3;  // T → boolean
Consumer<String>           c  = System.out::println;   // T → void
Supplier<LocalDate>        s  = LocalDate::now;        // () → T
UnaryOperator<String>      u  = String::toUpperCase;   // T → T (special Function)
BinaryOperator<Integer>    b  = Integer::sum;          // (T, T) → T
BiFunction<String, Integer, String> bf = String::substring; // (T, U) → R

// ✅ Custom functional interface
@FunctionalInterface
interface Validator<T> {
    boolean validate(T input);

    // default methods are fine — don't count as SAM
    default Validator<T> and(Validator<T> other) {
        return input -> this.validate(input) && other.validate(input);
    }
}

Validator<String> notEmpty = s -> !s.isEmpty();
Validator<String> notTooLong = s -> s.length() < 100;
Validator<String> combined = notEmpty.and(notTooLong);  // ⭐ composable!

// ✅ Lambda vs Anonymous class
// Before Java 8:
Runnable r1 = new Runnable() {
    @Override public void run() { System.out.println("old style"); }
};

// Java 8+:
Runnable r2 = () -> System.out.println("lambda");

// ❌ NOT a functional interface (2 abstract methods)
// @FunctionalInterface  ← compiler error!
interface BadInterface {
    void method1();
    void method2();
}

// ✅ IS a functional interface (Object methods don't count)
@FunctionalInterface
interface StillValid {
    void doWork();
    boolean equals(Object o);  // from Object — doesn't count
    String toString();         // from Object — doesn't count
}
```

### ⚠️ Common Pitfalls
- Thinking `@FunctionalInterface` is required — it's optional; any SAM interface works with lambdas
- **Object methods** (`equals`, `hashCode`, `toString`) don't count toward the SAM limit
- **Default methods** don't count — an interface with 1 abstract + 10 default methods is still functional
- Checked exceptions in lambdas — built-in functional interfaces don't declare checked exceptions → wrap or create custom

### 🆚 Comparison Table

| Functional Interface | Method | Signature | Use Case |
|---------------------|--------|-----------|----------|
| `Function<T,R>` | `apply` | `T → R` | Transform |
| `Predicate<T>` | `test` | `T → boolean` | Filter |
| `Consumer<T>` | `accept` | `T → void` | Side effect |
| `Supplier<T>` | `get` | `() → T` | Lazy creation |
| `UnaryOperator<T>` | `apply` | `T → T` | Same-type transform |
| `BiFunction<T,U,R>` | `apply` | `(T, U) → R` | Two-input transform |
| `Runnable` | `run` | `() → void` | Task execution |
| `Comparator<T>` | `compare` | `(T, T) → int` | Sorting |

### ⚡ Remember (Quick Recall)
- **SAM** = Single Abstract Method = Functional Interface
- `@FunctionalInterface` = compile-time safety (like `@Override`)
- Default/static/Object methods DON'T count
- Lambda = concise implementation of a functional interface
- `invokedynamic` under the hood (not anonymous class)

### 🔗 Follow-up Topics
- [Q4 → Predicate/Function/Consumer deep-dive](#q4)
- Method references: `Class::method`, `object::method`, `Class::new`
- Effective final variables in lambda closures

---

<a id="q4"></a>
## Q4. What is the difference between Predicate, Function, and Consumer?

### 📝 One-Liner
`Predicate<T>` returns **boolean** (filtering); `Function<T,R>` returns a **transformed value** (mapping); `Consumer<T>` returns **void** (side effects).

### 🔑 Quick Answer
All three are in `java.util.function`. **Predicate** → `test(T) → boolean` → used in `filter()`, `removeIf()`, conditionals. **Function** → `apply(T) → R` → used in `map()`, `computeIfAbsent()`, transformations. **Consumer** → `accept(T) → void` → used in `forEach()`, `peek()`, callbacks. All three support **composition**: Predicate has `and/or/negate`, Function has `andThen/compose`, Consumer has `andThen`. There's also `Supplier<T>` → `get() → T` (no input, factory pattern). *(Predicate = test karo, Function = transform karo, Consumer = use karo bina return ke)*

### 📖 How It Works (Detailed Explanation)

```
┌─────────────┐     ┌─────────────┐     ┌─────────────┐
│  Predicate   │     │  Function    │     │  Consumer    │
│  T → boolean │     │  T → R       │     │  T → void    │
│              │     │              │     │              │
│  .test(t)    │     │  .apply(t)   │     │  .accept(t)  │
│              │     │              │     │              │
│  filter()    │     │  map()       │     │  forEach()   │
│  removeIf()  │     │  compute()   │     │  peek()      │
│  anyMatch()  │     │  toMap()     │     │  ifPresent() │
└─────────────┘     └─────────────┘     └─────────────┘
```

**Composition chains**: These interfaces are composable — build complex logic from simple pieces. `Predicate.and()` / `or()` / `negate()` for boolean logic. `Function.andThen()` (apply this, then that) / `compose()` (apply that first, then this). `Consumer.andThen()` for sequential side effects. This is the power of functional programming in Java — small, reusable building blocks.

### 🗣️ Interview Script
"Predicate, Function, and Consumer are the three fundamental functional interfaces that cover most use cases. Predicate takes an input and returns boolean — I use it with stream filter, collection removeIf, and @ConditionalOnProperty-style conditions. Function takes an input and transforms it into a different type — it powers stream map, Map.computeIfAbsent, and DTO conversions. Consumer takes an input and returns nothing — used for side effects like logging, sending events, or forEach. What makes them powerful is composition: I can chain Predicates with and/or, Functions with andThen, and Consumers with andThen to build complex behavior from simple, testable pieces."

### 💻 Code Example

```java
// ✅ Predicate — filtering (T → boolean)
Predicate<Employee> isActive = Employee::isActive;
Predicate<Employee> isSenior = e -> e.getExperience() > 5;
Predicate<Employee> isActiveSenior = isActive.and(isSenior);  // ⭐ composition!

List<Employee> result = employees.stream()
    .filter(isActiveSenior)
    .collect(Collectors.toList());

// Predicate.negate() and Predicate.or()
Predicate<String> notBlank = Predicate.not(String::isBlank);  // Java 11+
Predicate<Employee> seniorOrManager = isSenior.or(e -> e.getRole() == Role.MANAGER);

// ✅ Function — transformation (T → R)
Function<Employee, String> toName = Employee::getName;
Function<String, String> toUpper = String::toUpperCase;
Function<Employee, String> toUpperName = toName.andThen(toUpper);  // ⭐ chain!

List<String> upperNames = employees.stream()
    .map(toUpperName)
    .collect(Collectors.toList());

// Function.compose (reverse order)
Function<String, String> trim = String::trim;
Function<String, String> trimThenUpper = toUpper.compose(trim);  // trim first, then upper

// ✅ Consumer — side effect (T → void)
Consumer<Employee> log = e -> System.out.println("Processing: " + e.getName());
Consumer<Employee> save = employeeRepository::save;
Consumer<Employee> logThenSave = log.andThen(save);  // ⭐ chain!

employees.forEach(logThenSave);

// ✅ Supplier — factory (void → T)
Supplier<List<String>> listFactory = ArrayList::new;
Supplier<LocalDateTime> now = LocalDateTime::now;

// ✅ Practical: configurable processing pipeline
public List<EmployeeDTO> processEmployees(
        List<Employee> employees,
        Predicate<Employee> filter,
        Function<Employee, EmployeeDTO> mapper) {
    return employees.stream()
        .filter(filter)
        .map(mapper)
        .collect(Collectors.toList());
}

// Callers pass different strategies:
processEmployees(all, isActiveSenior, EmployeeDTO::fromEntity);
processEmployees(all, e -> e.getDept() == IT, EmployeeDTO::miniView);
```

### ⚠️ Common Pitfalls
- **Checked exceptions** — built-in interfaces don't throw checked exceptions → wrap with try-catch inside lambda or create custom `ThrowingFunction<T,R>`
- **Heavy side effects in Predicate/Function** — these should be pure; side effects belong in Consumer
- **Primitive variants forgotten** — use `IntPredicate`, `ToIntFunction`, `IntConsumer` to avoid boxing overhead
- **andThen vs compose** — `f.andThen(g)` = f first then g; `f.compose(g)` = g first then f

### 🆚 Comparison Table

| Aspect | Predicate<T> | Function<T,R> | Consumer<T> | Supplier<T> |
|--------|-------------|---------------|-------------|-------------|
| Input | T | T | T | None |
| Output | boolean | R | void | T |
| Method | `test()` | `apply()` | `accept()` | `get()` |
| Stream use | `filter()` | `map()` | `forEach()` | N/A |
| Composition | `and/or/negate` | `andThen/compose` | `andThen` | N/A |
| Bi-variant | `BiPredicate` | `BiFunction` | `BiConsumer` | N/A |

### ⚡ Remember (Quick Recall)
- **Predicate** = yes/no gate → `filter()`, `removeIf()`
- **Function** = transformer → `map()`, `computeIfAbsent()`
- **Consumer** = sink (no return) → `forEach()`, `ifPresent()`
- **Supplier** = source (no input) → `orElseGet()`, factories
- All composable: `.and()`, `.andThen()`, `.compose()`

### 🔗 Follow-up Topics
- [Q3 → Functional Interface (parent concept)](#q3)
- [Q5 → Optional (uses Predicate, Function, Consumer)](#q5)
- Primitive specializations (`IntFunction`, `LongConsumer`, etc.)

---

<a id="q5"></a>
## Q5. What is Optional in Java and how do you use it properly?

### 📝 One-Liner
`Optional<T>` is a container that **may or may not hold a value** — designed to replace `null` returns and force callers to handle the absent case explicitly.

### 🔑 Quick Answer
`Optional.of(value)` — wraps non-null value. `Optional.ofNullable(value)` — wraps possibly-null value. `Optional.empty()` — no value. **Use for**: method return types that might have no result. **Don't use for**: fields, method parameters, collections (return empty collection instead). Key methods: `isPresent()`, `ifPresent(Consumer)`, `map(Function)`, `flatMap()`, `orElse()`, `orElseGet(Supplier)`, `orElseThrow()`. **Never call `.get()` without checking** — use `orElse`/`orElseThrow` instead. *(Optional = null ki jagah use karo — caller ko force karo ki absent case handle kare)*

### 📖 How It Works (Detailed Explanation)

```
Optional Flow:
┌─────────────┐
│ findById(1)  │
│ → Optional   │
└──────┬──────┘
       │
  ┌────▼────┐     ┌──────────────┐
  │ Present? │──Y──│ .map()       │──→ transform
  │          │     │ .ifPresent() │──→ consume
  └────┬─────┘     │ .get()       │──→ unwrap (risky!)
       │           └──────────────┘
       N
       │
  ┌────▼─────────────┐
  │ .orElse(default)  │──→ fallback value
  │ .orElseGet(sup)   │──→ lazy fallback
  │ .orElseThrow()    │──→ throw exception
  └───────────────────┘
```

**orElse vs orElseGet**: `orElse(computeDefault())` — the default is ALWAYS computed even if value is present. `orElseGet(() -> computeDefault())` — lazy, only computed when absent. If default computation is expensive, always use `orElseGet`. **Optional chaining**: `map` for transformations that return non-Optional, `flatMap` when the function itself returns Optional (avoids `Optional<Optional<T>>`). Java 9 added `ifPresentOrElse()`, `or()`, `stream()`.

### 🗣️ Interview Script
"Optional is Java's way of explicitly representing the absence of a value — instead of returning null from a method and hoping the caller checks for it, I return Optional, which forces the caller to decide how to handle the empty case. I use it exclusively for return types — never for fields, parameters, or collections. For consuming the value, I chain map and flatMap instead of isPresentget patterns. orElseThrow for cases where absence is an error — like findById returning empty. One subtle but important distinction: orElse eagerly evaluates its argument, so if the fallback is expensive, I use orElseGet with a Supplier for lazy evaluation. I never call .get() without a check — it defeats the purpose of Optional."

### 💻 Code Example

```java
// ✅ Creating Optionals
Optional<User> present = Optional.of(user);           // must be non-null
Optional<User> maybe   = Optional.ofNullable(user);   // null → empty
Optional<User> empty   = Optional.empty();

// ✅ Consuming — GOOD patterns
// Pattern 1: transform and get with default
String name = findUser(id)
    .map(User::getName)
    .orElse("Unknown");

// Pattern 2: transform chain with flatMap
String city = findUser(id)
    .flatMap(User::getAddress)       // getAddress returns Optional<Address>
    .map(Address::getCity)
    .orElse("N/A");

// Pattern 3: throw if absent
User user = findUser(id)
    .orElseThrow(() -> new UserNotFoundException("User not found: " + id));

// Pattern 4: conditional action
findUser(id).ifPresent(u -> sendWelcomeEmail(u));

// Pattern 5 (Java 9+): ifPresentOrElse
findUser(id).ifPresentOrElse(
    u -> log.info("Found: {}", u.getName()),
    () -> log.warn("User not found")
);

// ✅ orElse vs orElseGet — CRITICAL difference
findUser(id).orElse(createDefaultUser());       // ❌ createDefaultUser() called ALWAYS
findUser(id).orElseGet(() -> createDefaultUser()); // ✅ called only when empty

// ✅ filter — conditional Optional
Optional<User> admin = findUser(id)
    .filter(u -> u.getRole() == Role.ADMIN);  // empty if not admin

// ✅ Stream integration (Java 9+)
List<String> names = userIds.stream()
    .map(this::findUser)                // Stream<Optional<User>>
    .flatMap(Optional::stream)          // Stream<User> (empties removed)
    .map(User::getName)
    .collect(Collectors.toList());

// ❌ BAD patterns — anti-patterns
// 1. Using get() without check
user.get();  // ❌ NoSuchElementException if empty!

// 2. isPresent + get (defeats purpose)
if (opt.isPresent()) { return opt.get(); }  // ❌ just use orElse

// 3. Optional as field
class Order { Optional<Discount> discount; }  // ❌ not serializable, wasteful

// 4. Optional as parameter
void process(Optional<String> name) { }  // ❌ just use @Nullable or overload

// 5. Optional for collections
Optional<List<Item>> items;  // ❌ return empty List instead
```

### ⚠️ Common Pitfalls
- **`.get()` without check** — throws `NoSuchElementException` → always use `orElse`/`orElseThrow`
- **`orElse` with expensive computation** — evaluated even when Optional is present → use `orElseGet`
- **Optional as field** — not `Serializable`, adds memory overhead → use plain field + null
- **Optional wrapping collections** — return `Collections.emptyList()` instead, never `Optional<List<T>>`
- **`Optional.of(null)`** — throws `NullPointerException` → use `Optional.ofNullable()` for nullable values

### 🆚 Comparison Table

| Method | Returns | When Empty | When Present |
|--------|---------|------------|-------------|
| `orElse(default)` | T | Returns default | Returns value (⚠️ default still computed) |
| `orElseGet(supplier)` | T | Calls supplier | Returns value (supplier NOT called) |
| `orElseThrow(supplier)` | T | Throws exception | Returns value |
| `map(function)` | Optional<R> | Empty | Applies function, wraps result |
| `flatMap(function)` | Optional<R> | Empty | Applies function (must return Optional) |
| `filter(predicate)` | Optional<T> | Empty | Empty if predicate false |
| `ifPresent(consumer)` | void | No-op | Calls consumer |

### 🎯 Tricky Follow-up Questions
- **Q**: Should Optional be used for method parameters?  
  **A**: No — use method overloading or `@Nullable`. Optional as parameter is clunky and non-standard.
- **Q**: Why isn't Optional Serializable?  
  **A**: By design — it's meant for return types, not for storing state in fields or transferring across the wire.

### ⚡ Remember (Quick Recall)
- Optional = return type only (not fields, not params, not collections)
- **Never `.get()`** — use `orElse`, `orElseGet`, `orElseThrow`
- `orElse` = eager; `orElseGet` = lazy
- `map` = transform; `flatMap` = unwrap nested Optional
- Java 9: `ifPresentOrElse`, `or()`, `stream()`

### 🔗 Follow-up Topics
- [Q2 → map vs flatMap (same concept in Streams)](#q2)
- [Q4 → Predicate/Function/Consumer (used inside Optional methods)](#q4)
- Null Object Pattern vs Optional
