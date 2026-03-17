# ☕ Java 8 Streams — Coding Problems: Filter, Sort, Transform & Reduce (Q1–Q16)

> **Source**: Commonly asked Java 8 Stream coding problems in interviews  
> **Coverage**: String operations, filtering, sorting, conversions, reductions — all with Streams + Lambdas  
> **Setup**: All examples use this Employee record where needed:
> ```java
> record Employee(String name, String dept, String city, double salary) {}
> ```

---

<a id="q1"></a>
## Q1. Remove duplicates from a list using Streams

### 💻 Code

```java
List<Integer> numbers = List.of(1, 2, 3, 2, 4, 5, 3, 6, 1);

// ✅ Using distinct()
List<Integer> unique = numbers.stream()
    .distinct()
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]

// ✅ Using toSet() — loses insertion order
Set<Integer> uniqueSet = new HashSet<>(numbers);

// ✅ Preserve order + remove duplicates
List<Integer> orderedUnique = numbers.stream()
    .collect(Collectors.toCollection(LinkedHashSet::new))
    .stream().collect(Collectors.toList());
```

### ⚡ Key Point
- `distinct()` uses `equals()/hashCode()` — O(n) with internal HashSet
- For custom objects, override `equals()` and `hashCode()`

---

<a id="q2"></a>
## Q2. Find the first non-repeated character in a string

### 💻 Code

```java
String input = "swiss";

// ✅ LinkedHashMap preserves insertion order
Character result = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() == 1)
    .map(Map.Entry::getKey)
    .findFirst()
    .orElse(null);
// Output: 'w'   (s repeated, w is first non-repeated)
```

### 🗣️ Interview Script
"I use groupingBy with a LinkedHashMap to count character frequencies while preserving insertion order. Then I filter for count == 1 and take the first entry. LinkedHashMap is critical here — a regular HashMap won't preserve the original order of characters." *(LinkedHashMap zaroori hai — insertion order preserve karta hai, warna pehla non-repeated character galat mil sakta hai)*

---

<a id="q3"></a>
## Q3. Find the frequency of each character in a string

### 💻 Code

```java
String input = "programming";

Map<Character, Long> freq = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));
// {p=1, r=2, o=1, g=2, a=1, m=2, i=1, n=1}

// ✅ Sorted by frequency (descending)
freq.entrySet().stream()
    .sorted(Map.Entry.<Character, Long>comparingByValue().reversed())
    .forEach(e -> System.out.println(e.getKey() + " = " + e.getValue()));
// r=2, g=2, m=2, p=1, o=1, a=1, i=1, n=1
```

---

<a id="q4"></a>
## Q4. Reverse a string using Streams

### 💻 Code

```java
String input = "Hello World";

// ✅ Approach 1: chars() + StringBuilder
String reversed = input.chars()
    .mapToObj(c -> String.valueOf((char) c))
    .reduce("", (a, b) -> b + a);
// "dlroW olleH"

// ✅ Approach 2: collect with StringBuilder (more efficient)
String reversed2 = input.chars()
    .mapToObj(c -> (char) c)
    .collect(StringBuilder::new, (sb, c) -> sb.insert(0, c), StringBuilder::append)
    .toString();

// ✅ Approach 3: Simple (non-stream but often accepted)
String reversed3 = new StringBuilder(input).reverse().toString();
```

### ⚡ Key Point
- StringBuilder.reverse() is simplest; stream approach shows API knowledge
- Interviewer wants to see you can use `reduce()` or `collect()` creatively

---

<a id="q5"></a>
## Q5. Sort a list of integers using Streams

### 💻 Code

```java
List<Integer> numbers = List.of(5, 3, 8, 1, 9, 2, 7);

// ✅ Ascending
List<Integer> ascending = numbers.stream()
    .sorted()
    .collect(Collectors.toList());
// [1, 2, 3, 5, 7, 8, 9]

// ✅ Descending
List<Integer> descending = numbers.stream()
    .sorted(Comparator.reverseOrder())
    .collect(Collectors.toList());
// [9, 8, 7, 5, 3, 2, 1]
```

---

<a id="q6"></a>
## Q6. Sort a list of employees by salary using Streams

### 💻 Code

```java
List<Employee> employees = List.of(
    new Employee("Alice", "Eng", "NYC", 95000),
    new Employee("Bob", "Eng", "SF", 110000),
    new Employee("Charlie", "Sales", "NYC", 75000),
    new Employee("Diana", "Eng", "SF", 120000)
);

// ✅ Sort by salary ascending
List<Employee> bySalary = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary))
    .collect(Collectors.toList());
// [Charlie:75000, Alice:95000, Bob:110000, Diana:120000]

// ✅ Sort by salary descending
List<Employee> bySalaryDesc = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary).reversed())
    .collect(Collectors.toList());

// ✅ Sort by department, then by salary descending
List<Employee> multiSort = employees.stream()
    .sorted(Comparator.comparing(Employee::dept)
        .thenComparing(Comparator.comparingDouble(Employee::salary).reversed()))
    .collect(Collectors.toList());
```

### ⚡ Key Point
- `Comparator.comparing()` for single field
- `.thenComparing()` for multi-field sort
- `.reversed()` for descending

---

<a id="q7"></a>
## Q7. Find even and odd numbers from a list using Streams

### 💻 Code

```java
List<Integer> numbers = List.of(1, 2, 3, 4, 5, 6, 7, 8, 9, 10);

// ✅ Even numbers
List<Integer> evens = numbers.stream()
    .filter(n -> n % 2 == 0)
    .collect(Collectors.toList());
// [2, 4, 6, 8, 10]

// ✅ Odd numbers
List<Integer> odds = numbers.stream()
    .filter(n -> n % 2 != 0)
    .collect(Collectors.toList());
// [1, 3, 5, 7, 9]

// ✅ Both at once using partitioningBy
Map<Boolean, List<Integer>> partition = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n % 2 == 0));
List<Integer> evens2 = partition.get(true);   // [2, 4, 6, 8, 10]
List<Integer> odds2 = partition.get(false);   // [1, 3, 5, 7, 9]
```

### ⚡ Key Point
- `partitioningBy()` is more efficient — single pass, splits into true/false groups
- This covers original Q11 (even), Q12 (odd), and Q21 (partition) in one

---

<a id="q8"></a>
## Q8. Count numbers greater than a given value

### 💻 Code

```java
List<Integer> numbers = List.of(10, 25, 30, 5, 15, 40, 8);
int threshold = 20;

long count = numbers.stream()
    .filter(n -> n > threshold)
    .count();
// 3 (25, 30, 40)

// ✅ With peek to see which ones
numbers.stream()
    .filter(n -> n > threshold)
    .forEach(n -> System.out.print(n + " "));
// 25 30 40
```

---

<a id="q9"></a>
## Q9. Convert a list into a Set and a Map using Streams

### 💻 Code

```java
List<String> names = List.of("Alice", "Bob", "Alice", "Charlie", "Bob");

// ✅ List → Set (removes duplicates)
Set<String> nameSet = names.stream()
    .collect(Collectors.toSet());
// [Alice, Bob, Charlie]

// ✅ List → Map (name → name length)
Map<String, Integer> nameMap = names.stream()
    .distinct()    // avoid duplicate key exception
    .collect(Collectors.toMap(
        Function.identity(),    // key = name
        String::length          // value = length
    ));
// {Alice=5, Bob=3, Charlie=7}

// ✅ Handle duplicate keys with merge function
Map<String, Integer> nameMap2 = names.stream()
    .collect(Collectors.toMap(
        Function.identity(),
        String::length,
        (existing, replacement) -> existing   // keep first on conflict
    ));

// ✅ Employee list → Map<name, salary>
Map<String, Double> salaryMap = employees.stream()
    .collect(Collectors.toMap(Employee::name, Employee::salary));
```

### ⚡ Key Point
- `toMap()` throws `IllegalStateException` on duplicate keys — always provide merge function or `distinct()` first
- This covers original Q14 (list → set) and Q15 (list → map)

---

<a id="q10"></a>
## Q10. Join list of strings into a single string

### 💻 Code

```java
List<String> words = List.of("Java", "is", "awesome");

// ✅ Simple join
String joined = words.stream()
    .collect(Collectors.joining(" "));
// "Java is awesome"

// ✅ With delimiter, prefix, suffix
String csv = words.stream()
    .collect(Collectors.joining(", ", "[", "]"));
// "[Java, is, awesome]"

// ✅ Join with transformation
String upper = words.stream()
    .map(String::toUpperCase)
    .collect(Collectors.joining(" | "));
// "JAVA | IS | AWESOME"
```

---

<a id="q11"></a>
## Q11. Find the longest and shortest string in a list

### 💻 Code

```java
List<String> words = List.of("Java", "Spring", "Go", "Kubernetes", "AI");

// ✅ Longest string
String longest = words.stream()
    .max(Comparator.comparingInt(String::length))
    .orElse("");
// "Kubernetes"

// ✅ Shortest string
String shortest = words.stream()
    .min(Comparator.comparingInt(String::length))
    .orElse("");
// "Go"

// ✅ Both with reduce
String longest2 = words.stream()
    .reduce((a, b) -> a.length() >= b.length() ? a : b)
    .orElse("");
```

### ⚡ Key Point
- `max(comparingInt(String::length))` — clean one-liner
- Returns `Optional` — handle empty list with `orElse()`

---

<a id="q12"></a>
## Q12. Find duplicate characters in a string

### 💻 Code

```java
String input = "programming";

// ✅ Find duplicate chars
List<Character> duplicates = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()))
    .entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .collect(Collectors.toList());
// [r, g, m]

// ✅ Using Set.add() trick
Set<Character> seen = new HashSet<>();
Set<Character> duplicates2 = input.chars()
    .mapToObj(c -> (char) c)
    .filter(c -> !seen.add(c))
    .collect(Collectors.toSet());
// [r, g, m]
```

---

<a id="q13"></a>
## Q13. Find common elements between two lists

### 💻 Code

```java
List<Integer> list1 = List.of(1, 2, 3, 4, 5);
List<Integer> list2 = List.of(3, 4, 5, 6, 7);

// ✅ Intersection using filter + contains
List<Integer> common = list1.stream()
    .filter(list2::contains)
    .collect(Collectors.toList());
// [3, 4, 5]

// ✅ Better performance: convert list2 to Set first (O(1) lookup)
Set<Integer> set2 = new HashSet<>(list2);
List<Integer> common2 = list1.stream()
    .filter(set2::contains)
    .collect(Collectors.toList());
// [3, 4, 5]
```

### ⚡ Key Point
- `list.contains()` is O(n) per call → O(n²) total
- Convert to `Set` first → O(1) per lookup → O(n) total

---

<a id="q14"></a>
## Q14. Convert list of strings to uppercase

### 💻 Code

```java
List<String> names = List.of("alice", "bob", "charlie");

List<String> upper = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// [ALICE, BOB, CHARLIE]

// ✅ Capitalize first letter only
List<String> capitalized = names.stream()
    .map(s -> s.substring(0, 1).toUpperCase() + s.substring(1))
    .collect(Collectors.toList());
// [Alice, Bob, Charlie]
```

---

<a id="q15"></a>
## Q15. Filter employees with salary greater than a given value

### 💻 Code

```java
double threshold = 90000;

List<Employee> highEarners = employees.stream()
    .filter(e -> e.salary() > threshold)
    .collect(Collectors.toList());
// [Alice:95000, Bob:110000, Diana:120000]

// ✅ Get just names
List<String> names = employees.stream()
    .filter(e -> e.salary() > threshold)
    .map(Employee::name)
    .collect(Collectors.toList());
// [Alice, Bob, Diana]

// ✅ Count
long count = employees.stream()
    .filter(e -> e.salary() > threshold)
    .count();
// 3
```

---

<a id="q16"></a>
## Q16. Merge two lists and check for duplicates using Streams

### 💻 Code

```java
List<Integer> list1 = List.of(1, 2, 3, 4);
List<Integer> list2 = List.of(3, 4, 5, 6);

// ✅ Merge (with duplicates)
List<Integer> merged = Stream.concat(list1.stream(), list2.stream())
    .collect(Collectors.toList());
// [1, 2, 3, 4, 3, 4, 5, 6]

// ✅ Merge (without duplicates)
List<Integer> mergedUnique = Stream.concat(list1.stream(), list2.stream())
    .distinct()
    .collect(Collectors.toList());
// [1, 2, 3, 4, 5, 6]

// ✅ Check if a list has duplicates
boolean hasDuplicates = numbers.size() != numbers.stream().distinct().count();

// ✅ Alternative: Set size check
boolean hasDuplicates2 = numbers.size() != new HashSet<>(numbers).size();
```

### ⚡ Key Point
- `Stream.concat()` for merging two streams
- `Stream.of(list1, list2).flatMap(Collection::stream)` for merging N lists
- This covers original Q27 (merge) and Q28 (check duplicates)
