# ☕ Java 8 Streams — Coding Problems: Grouping, Partitioning & Collectors (Q1–Q12)

> **Source**: Commonly asked Java 8 Stream coding problems in interviews  
> **Coverage**: groupingBy, partitioningBy, downstream collectors, multi-level grouping  
> **Setup**: All examples use this Employee record:
> ```java
> record Employee(String name, String dept, String city, double salary) {}
>
> List<Employee> employees = List.of(
>     new Employee("Alice", "Engineering", "NYC", 95000),
>     new Employee("Bob", "Engineering", "SF", 110000),
>     new Employee("Charlie", "Sales", "NYC", 75000),
>     new Employee("Diana", "Engineering", "SF", 120000),
>     new Employee("Eve", "Sales", "Chicago", 85000),
>     new Employee("Frank", "HR", "NYC", 70000),
>     new Employee("Grace", "HR", "SF", 72000)
> );
> ```

---

<a id="q1"></a>
## Q1. Group employees by department

### 💻 Code

```java
Map<String, List<Employee>> byDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept));
// {
//   Engineering = [Alice, Bob, Diana],
//   Sales       = [Charlie, Eve],
//   HR          = [Frank, Grace]
// }
```

### ⚡ Key Point
- `groupingBy()` default downstream is `toList()`
- Returns `HashMap` — no ordering guarantee; use `LinkedHashMap` supplier for insertion order

---

<a id="q2"></a>
## Q2. Group employees by department and count employees

### 💻 Code

```java
Map<String, Long> countByDept = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept, Collectors.counting()));
// {Engineering=3, Sales=2, HR=2}

// ✅ Sorted by count descending
countByDept.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .forEach(e -> System.out.println(e.getKey() + ": " + e.getValue()));
// Engineering: 3
// Sales: 2
// HR: 2
```

---

<a id="q3"></a>
## Q3. Group employees by department and find highest salary

### 💻 Code

```java
// ✅ Highest salary employee per department
Map<String, Optional<Employee>> highestByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
    ));
// {Engineering=Optional[Diana:120000], Sales=Optional[Eve:85000], HR=Optional[Grace:72000]}

// ✅ Just the salary value (no Optional wrapper)
Map<String, Double> maxSalaryByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
            opt -> opt.map(Employee::salary).orElse(0.0)
        )
    ));
// {Engineering=120000.0, Sales=85000.0, HR=72000.0}
```

### ⚡ Key Point
- `collectingAndThen()` = apply a finisher function after collecting *(groupBy ke baad maxBy se Optional milta hai — collectingAndThen se unwrap karo)*

---

<a id="q4"></a>
## Q4. Group employees by department and average salary

### 💻 Code

```java
Map<String, Double> avgByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.averagingDouble(Employee::salary)
    ));
// {Engineering=108333.33, Sales=80000.0, HR=71000.0}

// ✅ Full statistics per department
Map<String, DoubleSummaryStatistics> statsByDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.summarizingDouble(Employee::salary)
    ));
// Engineering: count=3, sum=325000, min=95000, avg=108333, max=120000
// Sales:       count=2, sum=160000, min=75000, avg=80000,  max=85000
```

### ⚡ Key Point
- `averagingDouble()`, `summingDouble()`, `summarizingDouble()` — three levels of detail
- `DoubleSummaryStatistics` gives min/max/avg/sum/count in a single pass

---

<a id="q5"></a>
## Q5. Group strings by their length

### 💻 Code

```java
List<String> words = List.of("Java", "Go", "Python", "C", "Rust", "Kotlin", "JS");

Map<Integer, List<String>> byLength = words.stream()
    .collect(Collectors.groupingBy(String::length));
// {1=[C], 2=[Go, JS], 4=[Java, Rust], 6=[Python, Kotlin]}

// ✅ Count per length
Map<Integer, Long> countByLength = words.stream()
    .collect(Collectors.groupingBy(String::length, Collectors.counting()));
// {1=1, 2=2, 4=2, 6=2}

// ✅ Sorted by length
words.stream()
    .collect(Collectors.groupingBy(String::length, TreeMap::new, Collectors.toList()));
// TreeMap preserves natural key order: {1=[C], 2=[Go, JS], 4=[Java, Rust], 6=[Python, Kotlin]}
```

---

<a id="q6"></a>
## Q6. Group employees by city

### 💻 Code

```java
Map<String, List<Employee>> byCity = employees.stream()
    .collect(Collectors.groupingBy(Employee::city));
// {NYC=[Alice, Charlie, Frank], SF=[Bob, Diana, Grace], Chicago=[Eve]}

// ✅ City → list of employee names
Map<String, List<String>> namesByCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::city,
        Collectors.mapping(Employee::name, Collectors.toList())
    ));
// {NYC=[Alice, Charlie, Frank], SF=[Bob, Diana, Grace], Chicago=[Eve]}

// ✅ City → total salary
Map<String, Double> salaryByCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::city,
        Collectors.summingDouble(Employee::salary)
    ));
// {NYC=240000, SF=302000, Chicago=85000}
```

---

<a id="q7"></a>
## Q7. Group words by their first character

### 💻 Code

```java
List<String> words = List.of("apple", "avocado", "banana", "blueberry", "cherry", "coconut", "apricot");

Map<Character, List<String>> byFirstChar = words.stream()
    .collect(Collectors.groupingBy(w -> w.charAt(0)));
// {a=[apple, avocado, apricot], b=[banana, blueberry], c=[cherry, coconut]}

// ✅ Case-insensitive grouping
Map<Character, List<String>> byFirstCharUpper = words.stream()
    .collect(Collectors.groupingBy(w -> Character.toUpperCase(w.charAt(0))));

// ✅ Count per first character
Map<Character, Long> countByFirstChar = words.stream()
    .collect(Collectors.groupingBy(w -> w.charAt(0), Collectors.counting()));
// {a=3, b=2, c=2}
```

---

<a id="q8"></a>
## Q8. Partition numbers into even and odd

### 💻 Code

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

Map<Boolean, List<Integer>> partition = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));

List<Integer> evens = partition.get(true);   // [2, 4, 6, 8, 10]
List<Integer> odds = partition.get(false);   // [1, 3, 5, 7, 9]
```

### 🆚 partitioningBy vs groupingBy

| Aspect | `partitioningBy` | `groupingBy` |
|--------|-----------------|-------------|
| **Key type** | `Boolean` (true/false) | Any type |
| **Groups** | Always exactly 2 | N groups |
| **Empty groups** | Both keys always present | Missing key = no entry |
| **Use when** | Binary split (yes/no) | Multiple categories |

*(partitioningBy = hamesha 2 group deta hai true/false; groupingBy = N groups)*

---

<a id="q9"></a>
## Q9. Multi-level grouping: employees by department then by city

### 💻 Code

```java
// ✅ Two-level grouping
Map<String, Map<String, List<Employee>>> byDeptThenCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.groupingBy(Employee::city)
    ));
// {
//   Engineering → {NYC=[Alice], SF=[Bob, Diana]},
//   Sales       → {NYC=[Charlie], Chicago=[Eve]},
//   HR          → {NYC=[Frank], SF=[Grace]}
// }

// ✅ Department → City → count
Map<String, Map<String, Long>> countByDeptCity = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.groupingBy(Employee::city, Collectors.counting())
    ));
// {Engineering → {NYC=1, SF=2}, Sales → {NYC=1, Chicago=1}, HR → {NYC=1, SF=1}}
```

### ⚡ Key Point
- Nested `groupingBy()` = multi-level grouping
- Downstream collector of outer `groupingBy` is another `groupingBy`

---

<a id="q10"></a>
## Q10. Calculate sum of numbers using Streams

### 💻 Code

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// ✅ Using reduce
int sum = numbers.stream()
    .reduce(0, Integer::sum);
// 55

// ✅ Using mapToInt (avoids boxing/unboxing)
int sum2 = numbers.stream()
    .mapToInt(Integer::intValue)
    .sum();
// 55

// ✅ Sum of employee salaries
double totalSalary = employees.stream()
    .mapToDouble(Employee::salary)
    .sum();
// 627000.0

// ✅ Product using reduce
int product = List.of(1, 2, 3, 4, 5).stream()
    .reduce(1, (a, b) -> a * b);
// 120
```

### ⚡ Key Point
- `mapToInt/mapToDouble.sum()` is preferred — avoids auto-boxing overhead
- `reduce(identity, accumulator)` for custom reductions

---

<a id="q11"></a>
## Q11. Collectors cheat sheet — all essential collectors in one place

### 💻 Reference

```java
// ✅ BASIC COLLECTORS
.collect(Collectors.toList())                           // → List<T>
.collect(Collectors.toSet())                            // → Set<T>
.collect(Collectors.toUnmodifiableList())                // → immutable List<T> (Java 10+)
.collect(Collectors.toCollection(TreeSet::new))          // → custom Collection

// ✅ STRING
.collect(Collectors.joining(", "))                       // → "a, b, c"
.collect(Collectors.joining(", ", "[", "]"))             // → "[a, b, c]"

// ✅ MAP
.collect(Collectors.toMap(keyFn, valueFn))               // → Map<K,V>
.collect(Collectors.toMap(keyFn, valueFn, mergeFn))      // handle duplicate keys

// ✅ GROUPING
.collect(Collectors.groupingBy(classifier))              // → Map<K, List<T>>
.collect(Collectors.groupingBy(classifier, downstream))  // with downstream collector
.collect(Collectors.partitioningBy(predicate))           // → Map<Boolean, List<T>>

// ✅ COUNTING / STATISTICS
.collect(Collectors.counting())                          // → Long
.collect(Collectors.summingDouble(fn))                   // → Double
.collect(Collectors.averagingDouble(fn))                 // → Double
.collect(Collectors.summarizingDouble(fn))               // → DoubleSummaryStatistics
.collect(Collectors.maxBy(comparator))                   // → Optional<T>
.collect(Collectors.minBy(comparator))                   // → Optional<T>

// ✅ TRANSFORMING
.collect(Collectors.mapping(fn, downstream))             // map + collect
.collect(Collectors.flatMapping(fn, downstream))         // flatMap + collect (Java 9+)
.collect(Collectors.filtering(pred, downstream))         // filter + collect (Java 9+)
.collect(Collectors.collectingAndThen(collector, finisher))  // post-process result

// ✅ REDUCING
.collect(Collectors.reducing(identity, accumulator))     // general reduction
```

---

<a id="q12"></a>
## Q12. Common patterns — quick reference for interviews

### 💻 One-Liners for Rapid Fire

```java
// Find max element
list.stream().max(Comparator.naturalOrder()).orElseThrow();

// Find min element
list.stream().min(Comparator.naturalOrder()).orElseThrow();

// Second highest
list.stream().distinct().sorted(Comparator.reverseOrder()).skip(1).findFirst();

// Top N elements
list.stream().sorted(Comparator.reverseOrder()).limit(3).toList();

// Any match / All match / None match
list.stream().anyMatch(x -> x > 10);   // true if ANY element > 10
list.stream().allMatch(x -> x > 0);    // true if ALL elements > 0
list.stream().noneMatch(x -> x < 0);   // true if NO element < 0

// Flat map — flatten nested lists
List<List<String>> nested = List.of(List.of("a", "b"), List.of("c", "d"));
List<String> flat = nested.stream()
    .flatMap(Collection::stream)
    .collect(Collectors.toList());
// [a, b, c, d]

// Frequency map
Map<T, Long> freq = list.stream()
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

// Map to different type
List<String> names = employees.stream()
    .map(Employee::name)
    .collect(Collectors.toList());

// Filter + Map + Collect (most common pipeline)
List<String> highEarnerNames = employees.stream()
    .filter(e -> e.salary() > 100000)
    .map(Employee::name)
    .sorted()
    .collect(Collectors.toList());

// Peek for debugging (don't use in production)
list.stream()
    .peek(x -> System.out.println("Before filter: " + x))
    .filter(x -> x > 5)
    .peek(x -> System.out.println("After filter: " + x))
    .collect(Collectors.toList());

// Custom collector (comma-separated string of names)
String result = employees.stream()
    .map(Employee::name)
    .collect(Collectors.joining(", "));
// "Alice, Bob, Charlie, Diana, Eve, Frank, Grace"
```

### ⚡ Stream Pipeline Pattern
```
source.stream()
    .filter(...)       // what to keep
    .map(...)          // transform
    .sorted(...)       // order
    .distinct()        // deduplicate
    .limit(N)          // take first N
    .skip(M)           // skip first M
    .collect(...)      // terminal: gather results
    // OR
    .forEach(...)      // terminal: side effects
    .reduce(...)       // terminal: aggregate
    .count()           // terminal: count
    .findFirst()       // terminal: Optional<T>
    .anyMatch(...)     // terminal: boolean
```

### 🔗 Related Topics
- [findFirst vs findAny](07-streams-coding-patterns.md#q1)
- [Find duplicates (3 approaches)](07-streams-coding-patterns.md#q2)
- [Max salary, per-dept, top-N, second-highest](07-streams-coding-patterns.md#q3)
- [map() vs flatMap()](05-java8-functional-epam.md#q2)
- [Functional Interfaces: Predicate, Function, Consumer](05-java8-functional-epam.md#q4)
