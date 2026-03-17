# ☕ Java 8 Streams — findFirst/findAny & Coding Patterns (Q1–Q3)

> **Source**: Capgemini Java Developer Interview (4+ years)  
> **Coverage**: Terminal operation differences, stream-based coding problems asked in interviews

---

<a id="q1"></a>
## Q1. What is the difference between findFirst() and findAny() in Streams?

### 📝 One-Liner
`findFirst()` returns the **first element** in encounter order (deterministic); `findAny()` returns **any element** (non-deterministic, optimized for parallel streams).

### 🔑 Quick Answer
Both return `Optional<T>` and are short-circuiting terminal operations. `findFirst()` — always returns the first element in the stream's encounter order. In parallel streams, it must synchronize to guarantee ordering → slower. `findAny()` — returns any matching element, no ordering guarantee. In parallel streams, whichever thread finds an element first wins → faster. **In sequential streams, both behave identically** (return first element). The difference matters only in **parallel streams**. Use `findFirst()` when you need deterministic results; use `findAny()` when any match is acceptable and you want maximum parallel performance. *(findFirst = pehla element guaranteed; findAny = koi bhi element — parallel mein fast hai)*

### 📖 How It Works (Detailed Explanation)

```
Sequential stream (both behave the same):
  [A, B, C, D].stream().findFirst() → Optional[A]
  [A, B, C, D].stream().findAny()   → Optional[A]  (same!)

Parallel stream (difference appears):
  [A, B, C, D].parallelStream().findFirst() → Optional[A]  (always A)
  [A, B, C, D].parallelStream().findAny()   → Optional[C]  (any! depends on thread)

  Thread 1: processes [A, B]    ┐
  Thread 2: processes [C, D]    ┤→ findAny: whichever thread finishes first wins
                                 → findFirst: must coordinate to ensure A is returned
```

**Encounter order**: defined by the source — `List` has encounter order (index-based), `HashSet` does NOT. For unordered sources, `findFirst()` and `findAny()` behave similarly. **Performance**: in parallel streams, `findFirst()` forces the framework to coordinate across threads to respect ordering → overhead. `findAny()` has no such constraint → each thread can independently return a result.

### 🗣️ Interview Script
"Both findFirst and findAny are short-circuiting terminal operations that return an Optional. In sequential streams, there's no practical difference — both return the first element. The distinction matters in parallel streams. findFirst guarantees the first element in encounter order, which requires thread coordination and reduces parallelism. findAny returns whichever element any thread discovers first, with no ordering guarantee, allowing maximum parallel performance. I use findFirst when I need a deterministic result — like the first matching record chronologically. I use findAny when any match is acceptable — like checking if any user has a specific role — especially in parallel processing."

### 💻 Code Example

```java
// ✅ Sequential — both return first element
List<String> names = List.of("Alice", "Bob", "Charlie");

Optional<String> first = names.stream()
    .filter(n -> n.length() > 3)
    .findFirst();  // Optional[Alice] — always Alice

Optional<String> any = names.stream()
    .filter(n -> n.length() > 3)
    .findAny();    // Optional[Alice] — also Alice (sequential)

// ✅ Parallel — findAny is non-deterministic
Optional<String> parallelAny = names.parallelStream()
    .filter(n -> n.length() > 3)
    .findAny();    // Could be Alice OR Charlie — depends on thread scheduling

Optional<String> parallelFirst = names.parallelStream()
    .filter(n -> n.length() > 3)
    .findFirst();  // Always Alice — ordered, but slower in parallel

// ✅ Use findFirst: need deterministic result
Optional<Transaction> latestFraud = transactions.stream()
    .filter(Transaction::isFraudulent)
    .sorted(Comparator.comparing(Transaction::getTimestamp).reversed())
    .findFirst();  // most recent fraud — must be deterministic

// ✅ Use findAny: just need existence check
boolean hasAdmin = users.parallelStream()
    .filter(u -> u.getRole() == Role.ADMIN)
    .findAny()
    .isPresent();  // any admin will do — findAny is faster in parallel

// ✅ Unordered source — no difference
Set<String> nameSet = Set.of("Alice", "Bob", "Charlie");
nameSet.stream().findFirst();  // no guaranteed order (Set has no encounter order)
nameSet.stream().findAny();    // same — both non-deterministic with Set
```

### 🆚 Comparison Table

| Aspect | findFirst() | findAny() |
|--------|------------|-----------|
| Returns | First in encounter order | Any element |
| Sequential | First element | First element (same) |
| Parallel | First element (coordinated) | Any thread's result |
| Performance (parallel) | Slower (ordering overhead) | **Faster** ⭐ |
| Deterministic | ✅ Yes | ❌ No (in parallel) |
| Use case | Need specific order | Any match is fine |

### ⚡ Remember (Quick Recall)
- **Sequential**: both return first element — no difference
- **Parallel**: `findFirst()` = ordered (slow), `findAny()` = unordered (fast)
- Both return `Optional<T>` and are short-circuiting
- Use `findFirst` for deterministic results, `findAny` for existence checks

---

<a id="q2"></a>
## Q2. Write a program to find duplicate elements in a list using Stream API.

### 📝 One-Liner
Use `Collectors.groupingBy` + `Collectors.counting()` and filter entries with count > 1, OR use a `Set` with `filter(!set.add(e))` for a one-pass approach.

### 🔑 Quick Answer
**Approach 1** (grouping): `stream().collect(groupingBy(identity(), counting()))` → filter entries where count > 1 → collect keys. **Approach 2** (Set trick): `stream().filter(e -> !seen.add(e))` — `Set.add()` returns `false` if element already exists → those are duplicates. **Approach 3** (frequency): `Collections.frequency()` in a filter. Approach 2 is most concise but uses side-effect (mutable Set). Approach 1 is more functional and readable for interviews. *(Duplicates nikalne ke liye groupingBy + counting use karo, ya Set.add() ka trick use karo)*

### 💻 Code Example

```java
List<Integer> numbers = List.of(1, 2, 3, 2, 4, 5, 3, 6, 1);

// ✅ Approach 1: groupingBy + counting (most readable for interviews)
List<Integer> duplicates = numbers.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .collect(Collectors.toList());
// Output: [1, 2, 3]

// ✅ Approach 2: Set.add() trick (concise, one-pass)
Set<Integer> seen = new HashSet<>();
List<Integer> duplicates2 = numbers.stream()
    .filter(n -> !seen.add(n))   // Set.add returns false if already present
    .distinct()                   // avoid duplicate duplicates
    .collect(Collectors.toList());
// Output: [2, 3, 1]

// ✅ Approach 3: Frequency-based
Set<Integer> duplicates3 = numbers.stream()
    .filter(n -> Collections.frequency(numbers, n) > 1)
    .collect(Collectors.toSet());  // Set to deduplicate
// Output: [1, 2, 3]
// ⚠️ O(n²) — frequency() iterates list for each element

// ✅ Find duplicate strings (case-insensitive)
List<String> words = List.of("Java", "Python", "java", "Go", "python");
Set<String> dupWords = words.stream()
    .map(String::toLowerCase)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .collect(Collectors.toSet());
// Output: [java, python]

// ✅ Count occurrences of each element
Map<Integer, Long> frequencyMap = numbers.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// Output: {1=2, 2=2, 3=2, 4=1, 5=1, 6=1}
```

### ⚡ Remember (Quick Recall)
- **Interview favorite**: `groupingBy(identity(), counting())` → filter count > 1
- **One-liner trick**: `filter(n -> !seen.add(n))` with HashSet (side-effect but concise)
- Always mention **time complexity**: Approach 1 & 2 = O(n), Approach 3 = O(n²)

---

<a id="q3"></a>
## Q3. Write a program to find the employee with maximum salary using Stream API.

### 📝 One-Liner
Use `stream().max(Comparator.comparingDouble(Employee::getSalary))` or `stream().collect(Collectors.maxBy(...))`.

### 🔑 Quick Answer
**Approach 1**: `max()` with Comparator — `employees.stream().max(Comparator.comparingDouble(Employee::getSalary))` → returns `Optional<Employee>`. **Approach 2**: `reduce()` — `stream().reduce((e1, e2) -> e1.getSalary() > e2.getSalary() ? e1 : e2)`. **Approach 3**: `sorted()` + `findFirst()` — sort descending, take first (less efficient, O(n log n) vs O(n)). The `max()` approach is cleanest and O(n). *(Max salary nikalne ke liye stream().max(comparingDouble(getSalary)) use karo)*

### 💻 Code Example

```java
record Employee(String name, String department, double salary) {}

List<Employee> employees = List.of(
    new Employee("Alice", "Engineering", 95000),
    new Employee("Bob", "Engineering", 110000),
    new Employee("Charlie", "Sales", 75000),
    new Employee("Diana", "Engineering", 120000),
    new Employee("Eve", "Sales", 85000)
);

// ✅ Approach 1: max() — cleanest, O(n)
Optional<Employee> highest = employees.stream()
    .max(Comparator.comparingDouble(Employee::salary));
highest.ifPresent(e -> System.out.println(e.name() + ": " + e.salary()));
// Output: Diana: 120000.0

// ✅ Approach 2: reduce()
Optional<Employee> highest2 = employees.stream()
    .reduce((e1, e2) -> e1.salary() > e2.salary() ? e1 : e2);

// ✅ Approach 3: sorted + findFirst (O(n log n) — not recommended)
Optional<Employee> highest3 = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary).reversed())
    .findFirst();

// ✅ Max salary per department
Map<String, Optional<Employee>> maxByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
    ));
// {Engineering=Optional[Diana:120000], Sales=Optional[Eve:85000]}

// ✅ Top N salaries
List<Employee> top3 = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary).reversed())
    .limit(3)
    .collect(Collectors.toList());
// [Diana:120000, Bob:110000, Alice:95000]

// ✅ Sum / Average salary
double totalSalary = employees.stream()
    .mapToDouble(Employee::salary)
    .sum();

OptionalDouble avgSalary = employees.stream()
    .mapToDouble(Employee::salary)
    .average();

// ✅ Statistics in one pass
DoubleSummaryStatistics stats = employees.stream()
    .mapToDouble(Employee::salary)
    .summaryStatistics();
// stats.getMax(), stats.getMin(), stats.getAverage(), stats.getSum(), stats.getCount()

// ✅ Second highest salary
Optional<Double> secondHighest = employees.stream()
    .map(Employee::salary)
    .distinct()
    .sorted(Comparator.reverseOrder())
    .skip(1)
    .findFirst();
// Output: Optional[110000.0]
```

### ⚡ Remember (Quick Recall)
- **Best approach**: `stream().max(Comparator.comparingDouble(...))` → O(n)
- Returns `Optional` — handle empty list case
- **Per-group max**: `groupingBy` + `maxBy`
- **Top N**: `sorted(reversed()).limit(N)`
- **Second highest**: `distinct().sorted(reversed()).skip(1).findFirst()`
- `DoubleSummaryStatistics` for min/max/avg/sum in one pass

### 🔗 Follow-up Topics
- [Q1 in core/05 → Stream API overview](05-java8-functional-epam.md#q1)
- [Q2 in core/05 → map() vs flatMap()](05-java8-functional-epam.md#q2)
- Collectors: `toMap`, `partitioningBy`, `joining`
