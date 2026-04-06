# ☕ Java 8 Coding Questions — From Real Interviews Part 2 (Q1–Q13)

> **Source**: Frequently asked Java 8 coding questions from actual interviews  
> **Coverage**: String parsing → object mapping, multi-field sorting, grouping + sorting, filtering, word frequency, array manipulation, first repeated char  
> **Focus**: Streams API, Comparator chaining, Collectors, functional transformations  
> **Setup**: Employee & Student records used across examples:
> ```java
> record Employee(int id, String name, String dept, double salary) {}
> record EmployeeWithMgr(int id, String name, int managerId, double salary) {}
> record Student(String name, String branch, String address, int marks) {}
> ```

> **Cross-references** (questions covered in other files — not repeated here):
> - First non-repeated character → [08-streams-coding-basic.md Q2](./08-streams-coding-basic.md#q2)
> - Second highest number → [07-streams-coding-patterns.md Q3](./07-streams-coding-patterns.md#q3)
> - Frequency of each element in list → [07-streams-coding-patterns.md Q2](./07-streams-coding-patterns.md#q2)
> - Max and min from list → [07-streams-coding-patterns.md Q3](./07-streams-coding-patterns.md#q3) (DoubleSummaryStatistics)
> - Character frequency → [08-streams-coding-basic.md Q3](./08-streams-coding-basic.md#q3)

---

<a id="q1"></a>
## Q1. Convert given string data into List of Employee objects and sort by ID (ascending)

> 🔥 **Frequently asked** — tests string parsing + stream mapping + sorting in one shot

### 💻 Code

```java
String empDetails1 = "1,AAAAA,develop,234566||3,NNNNN,QA,78977";
String empDetails2 = "4,BBBBB,develop,1234||2,KKKKKK,develop,67678";
String empDetails3 = "5,CCCCC,Manager,89888";

// ✅ Step 1: Combine all strings, split by ||, parse each into Employee
List<Employee> employees = Stream.of(empDetails1, empDetails2, empDetails3)
    .flatMap(s -> Arrays.stream(s.split("\\|\\|")))   // split by ||
    .map(s -> {
        String[] parts = s.split(",");
        return new Employee(
            Integer.parseInt(parts[0].trim()),
            parts[1].trim(),
            parts[2].trim(),
            Double.parseDouble(parts[3].trim())
        );
    })
    .sorted(Comparator.comparingInt(Employee::id))     // sort by ID ascending
    .collect(Collectors.toList());

employees.forEach(System.out::println);
// Employee[id=1, name=AAAAA, dept=develop, salary=234566.0]
// Employee[id=2, name=KKKKKK, dept=develop, salary=67678.0]
// Employee[id=3, name=NNNNN, dept=QA, salary=78977.0]
// Employee[id=4, name=BBBBB, dept=develop, salary=1234.0]
// Employee[id=5, name=CCCCC, dept=Manager, salary=89888.0]
```

### ⚡ Key Points
- `flatMap` flattens multiple `||`-separated entries into a single stream
- `split("\\|\\|")` — pipe `|` is a regex metacharacter, needs double escaping
- `Comparator.comparingInt(Employee::id)` — use `comparingInt` not `comparing` for primitives (avoids boxing)
- *(Pehle sab strings combine karo, phir || se split, phir map se Employee banaao, last mein sort by ID)*

---

<a id="q2"></a>
## Q2. Sort employee list by name; if name is same, then sort by salary

### 💻 Code

```java
List<Employee> employees = List.of(
    new Employee(1, "Alice", "Engineering", 95000),
    new Employee(2, "Bob", "Sales", 75000),
    new Employee(3, "Alice", "HR", 85000),
    new Employee(4, "Bob", "Engineering", 110000),
    new Employee(5, "Charlie", "Sales", 60000)
);

// ✅ Comparator chaining — thenComparing for secondary sort
List<Employee> sorted = employees.stream()
    .sorted(Comparator.comparing(Employee::name)
            .thenComparingDouble(Employee::salary))
    .collect(Collectors.toList());

// Alice (85000) → Alice (95000) → Bob (75000) → Bob (110000) → Charlie (60000)
```

### ⚡ Key Points
- `Comparator.comparing().thenComparing()` — chain multiple sort criteria
- `thenComparingDouble` avoids autoboxing for double fields
- For descending secondary: `.thenComparing(Employee::salary, Comparator.reverseOrder())`
- *(Pehle name se sort hoga, agar name same hai toh salary se sort — chaining use karo)*

---

<a id="q3"></a>
## Q3. Sort list of students based on name, branch, address (triple-field sort)

### 💻 Code

```java
List<Student> students = List.of(
    new Student("Rahul", "CSE", "Mumbai", 85),
    new Student("Amit", "ECE", "Delhi", 90),
    new Student("Rahul", "CSE", "Pune", 78),
    new Student("Rahul", "IT", "Mumbai", 88),
    new Student("Amit", "CSE", "Mumbai", 72)
);

// ✅ Triple-field sort: name → branch → address
List<Student> sorted = students.stream()
    .sorted(Comparator.comparing(Student::name)
            .thenComparing(Student::branch)
            .thenComparing(Student::address))
    .collect(Collectors.toList());

// Amit-CSE-Mumbai → Amit-ECE-Delhi → Rahul-CSE-Mumbai → Rahul-CSE-Pune → Rahul-IT-Mumbai
```

### ⚡ Key Points
- Chain as many `thenComparing` calls as needed — no limit
- Each subsequent field is a tiebreaker for the previous
- *(Jitne bhi fields ho, thenComparing chain karte jao — pehle name, phir branch, phir address)*

---

<a id="q4"></a>
## Q4. Sort employee list by name in descending order

### 💻 Code

```java
// ✅ Approach 1: Comparator.reverseOrder()
List<Employee> descByName = employees.stream()
    .sorted(Comparator.comparing(Employee::name, Comparator.reverseOrder()))
    .collect(Collectors.toList());

// ✅ Approach 2: reversed()
List<Employee> descByName2 = employees.stream()
    .sorted(Comparator.comparing(Employee::name).reversed())
    .collect(Collectors.toList());

// Charlie → Bob → Alice
```

### ⚡ Key Points
- Both approaches work; `Comparator.reverseOrder()` as second arg is cleaner
- ⚠️ Gotcha: `.reversed()` reverses ENTIRE comparator chain — use `reverseOrder()` for specific field
- *(Descending ke liye reversed() ya Comparator.reverseOrder() dono chalega — but chaining mein reverseOrder() better hai)*

---

<a id="q5"></a>
## Q5. Sort list of students by marks; if marks same, then sort by name

### 💻 Code

```java
List<Student> students = List.of(
    new Student("Zara", "CSE", "Mumbai", 90),
    new Student("Amit", "ECE", "Delhi", 85),
    new Student("Priya", "CSE", "Pune", 90),
    new Student("Rahul", "IT", "Mumbai", 85)
);

// ✅ Sort by marks ascending, then by name alphabetically
List<Student> sorted = students.stream()
    .sorted(Comparator.comparingInt(Student::marks)
            .thenComparing(Student::name))
    .collect(Collectors.toList());

// Amit(85) → Rahul(85) → Priya(90) → Zara(90)

// ✅ Marks descending, name ascending
List<Student> sortedDesc = students.stream()
    .sorted(Comparator.comparingInt(Student::marks).reversed()
            .thenComparing(Student::name))
    .collect(Collectors.toList());

// Priya(90) → Zara(90) → Amit(85) → Rahul(85)
```

### ⚡ Key Points
- `comparingInt` for `int` fields — avoids Integer boxing
- ⚠️ `reversed()` after `comparingInt` reverses only marks; `thenComparing` stays ascending
- *(Marks se pehle sort, same marks pe name se — Comparator chaining ka classic pattern)*

---

<a id="q6"></a>
## Q6. Sort strings by length

### 💻 Code

```java
List<String> words = List.of("Java", "Go", "Python", "C", "Rust", "Kotlin", "JavaScript");

// ✅ Sort by length ascending
List<String> sortedByLength = words.stream()
    .sorted(Comparator.comparingInt(String::length))
    .collect(Collectors.toList());
// [C, Go, Java, Rust, Python, Kotlin, JavaScript]

// ✅ Sort by length descending
List<String> descByLength = words.stream()
    .sorted(Comparator.comparingInt(String::length).reversed())
    .collect(Collectors.toList());
// [JavaScript, Python, Kotlin, Java, Rust, Go, C]

// ✅ By length, then alphabetically for same-length strings
List<String> sortedTiebreak = words.stream()
    .sorted(Comparator.comparingInt(String::length)
            .thenComparing(Comparator.naturalOrder()))
    .collect(Collectors.toList());
// [C, Go, Java, Rust, Kotlin, Python, JavaScript]
```

### ⚡ Key Points
- `Comparator.comparingInt(String::length)` — cleanest way
- Add `thenComparing(naturalOrder())` for alphabetical tiebreak on same-length strings
- *(Length se sort karna ho toh String::length method reference pass karo — simple!)*

---

<a id="q7"></a>
## Q7. Group employees by department and sort them by salary (ascending) within each group

### 💻 Code

```java
List<Employee> employees = List.of(
    new Employee(1, "Alice", "Engineering", 95000),
    new Employee(2, "Bob", "Engineering", 110000),
    new Employee(3, "Charlie", "Sales", 75000),
    new Employee(4, "Diana", "Engineering", 85000),
    new Employee(5, "Eve", "Sales", 90000),
    new Employee(6, "Frank", "HR", 70000)
);

// ✅ Group by dept, then sort each group by salary ascending
Map<String, List<Employee>> grouped = employees.stream()
    .sorted(Comparator.comparingDouble(Employee::salary))  // sort first
    .collect(Collectors.groupingBy(Employee::dept));

// OR — sort within each group after grouping (more explicit)
Map<String, List<Employee>> grouped2 = employees.stream()
    .collect(Collectors.groupingBy(Employee::dept))
    .entrySet().stream()
    .collect(Collectors.toMap(
        Map.Entry::getKey,
        e -> e.getValue().stream()
            .sorted(Comparator.comparingDouble(Employee::salary))
            .collect(Collectors.toList())
    ));

// Engineering: Diana(85000) → Alice(95000) → Bob(110000)
// Sales:       Charlie(75000) → Eve(90000)
// HR:          Frank(70000)

// ✅ Cleanest — collectingAndThen with downstream collector
Map<String, List<Employee>> grouped3 = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::dept,
        Collectors.collectingAndThen(
            Collectors.toList(),
            list -> list.stream()
                .sorted(Comparator.comparingDouble(Employee::salary))
                .collect(Collectors.toList())
        )
    ));
```

### ⚡ Key Points
- **Approach 1**: Pre-sort before grouping — simpler but relies on stream ordering preservation in `groupingBy`
- **Approach 3**: `collectingAndThen` — sort within downstream collector — most idiomatic
- Grouping basics covered in → [09-streams-coding-collectors.md](./09-streams-coding-collectors.md)
- *(Group karne ke baad sort karna ho toh collectingAndThen use karo — downstream collector mein sort ho jaayega)*

---

<a id="q8"></a>
## Q8. Print employee names whose salary is greater than 40K

### 💻 Code

```java
List<Employee> employees = List.of(
    new Employee(1, "Alice", "Eng", 95000),
    new Employee(2, "Bob", "Sales", 35000),
    new Employee(3, "Charlie", "HR", 42000),
    new Employee(4, "Diana", "Eng", 38000)
);

// ✅ Filter + map + forEach
employees.stream()
    .filter(e -> e.salary() > 40000)
    .map(Employee::name)
    .forEach(System.out::println);
// Alice
// Charlie

// ✅ Collect as list
List<String> highEarners = employees.stream()
    .filter(e -> e.salary() > 40000)
    .map(Employee::name)
    .collect(Collectors.toList());
// [Alice, Charlie]

// ✅ With salary info
employees.stream()
    .filter(e -> e.salary() > 40000)
    .forEach(e -> System.out.println(e.name() + " → ₹" + e.salary()));
```

### ⚡ Key Points
- Classic `filter → map → collect` pipeline — interview staple
- `map(Employee::name)` extracts just the name — projects Employee to String
- *(Filter se condition lagao, map se name nikalo, collect ya forEach se output lo)*

---

<a id="q9"></a>
## Q9. Return employees whose salary is greater than their manager's salary

> 🔥 **Frequently asked** — tests self-join / lookup pattern with Streams

### 💻 Code

```java
// Employee structure: id, name, managerId, salary
// managerId = -1 means no manager (top-level)
List<EmployeeWithMgr> employees = List.of(
    new EmployeeWithMgr(1, "CEO",     -1, 150000),
    new EmployeeWithMgr(2, "Alice",    1,  160000),  // reports to CEO, earns MORE
    new EmployeeWithMgr(3, "Bob",      1,  120000),  // reports to CEO
    new EmployeeWithMgr(4, "Charlie",  2,  170000),  // reports to Alice, earns MORE
    new EmployeeWithMgr(5, "Diana",    2,   90000),  // reports to Alice
    new EmployeeWithMgr(6, "Eve",      3,  130000)   // reports to Bob, earns MORE
);

// ✅ Step 1: Build a lookup map (id → employee)
Map<Integer, EmployeeWithMgr> empById = employees.stream()
    .collect(Collectors.toMap(EmployeeWithMgr::id, Function.identity()));

// ✅ Step 2: Filter employees earning more than their manager
List<EmployeeWithMgr> result = employees.stream()
    .filter(e -> e.managerId() != -1)                              // skip top-level
    .filter(e -> {
        EmployeeWithMgr manager = empById.get(e.managerId());
        return manager != null && e.salary() > manager.salary();
    })
    .collect(Collectors.toList());

result.forEach(e -> {
    EmployeeWithMgr mgr = empById.get(e.managerId());
    System.out.println(e.name() + " (₹" + e.salary() + ") > Manager "
        + mgr.name() + " (₹" + mgr.salary() + ")");
});
// Alice (₹160000) > Manager CEO (₹150000)
// Charlie (₹170000) > Manager Alice (₹160000)
// Eve (₹130000) > Manager Bob (₹120000)
```

### ⚡ Key Points
- **Pattern**: Build `Map<id, Employee>` first → then filter with lookup
- This is the Streams equivalent of SQL: `SELECT e.* FROM emp e JOIN emp m ON e.mgr_id = m.id WHERE e.salary > m.salary`
- `toMap(EmployeeWithMgr::id, Function.identity())` — identity() avoids lambda `e -> e`
- *(Pehle Map banaao id → employee ka, phir filter mein manager lookup karo — SQL self-join jaisa pattern hai)*

---

<a id="q10"></a>
## Q10. Count frequency of words in a string

### 💻 Code

```java
String text = "java is a good language java is widely used java is versatile";

// ✅ Split by space → group by word → count
Map<String, Long> wordFreq = Arrays.stream(text.split("\\s+"))
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

System.out.println(wordFreq);
// {a=1, good=1, versatile=1, widely=1, used=1, java=3, is=3, language=1}

// ✅ Sorted by frequency descending
wordFreq.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .forEach(e -> System.out.println(e.getKey() + " = " + e.getValue()));
// java = 3
// is = 3
// a = 1  ... etc

// ✅ Case-insensitive frequency
Map<String, Long> caseInsensitive = Arrays.stream(text.split("\\s+"))
    .map(String::toLowerCase)
    .collect(Collectors.groupingBy(Function.identity(), Collectors.counting()));

// ✅ Top N most frequent words
List<Map.Entry<String, Long>> top3 = wordFreq.entrySet().stream()
    .sorted(Map.Entry.<String, Long>comparingByValue().reversed())
    .limit(3)
    .collect(Collectors.toList());
```

### ⚡ Key Points
- **Word frequency** = `split("\\s+")` + `groupingBy(identity(), counting())`
- Different from **character frequency** → see [08-streams-coding-basic.md Q3](./08-streams-coding-basic.md#q3)
- `comparingByValue().reversed()` for descending frequency sort
- *(Character frequency char-by-char hota hai, word frequency space se split karke hota hai — dono ka pattern same hai groupingBy + counting)*

---

<a id="q11"></a>
## Q11. Shift all zeros to the end of list

### 💻 Code

```java
List<Integer> numbers = new ArrayList<>(List.of(0, 1, 0, 3, 0, 5, 2, 0, 4));

// ✅ Approach 1: Comparator trick — zeros sort to end
List<Integer> result = numbers.stream()
    .sorted(Comparator.comparingInt(n -> n == 0 ? 1 : 0))  // non-zero=0 (first), zero=1 (last)
    .collect(Collectors.toList());
// [1, 3, 5, 2, 4, 0, 0, 0, 0]
// ⚠️ Note: this changes relative order of non-zero elements (unstable for non-zeros)

// ✅ Approach 2: Partition (preserves relative order)
Map<Boolean, List<Integer>> partitioned = numbers.stream()
    .collect(Collectors.partitioningBy(n -> n != 0));
List<Integer> result2 = new ArrayList<>(partitioned.get(true));   // non-zeros first
result2.addAll(partitioned.get(false));                            // then zeros
// [1, 3, 5, 2, 4, 0, 0, 0, 0]  — relative order preserved!

// ✅ Approach 3: Two-pass with streams (clean one-liner concept)
List<Integer> result3 = Stream.concat(
    numbers.stream().filter(n -> n != 0),     // non-zeros first
    numbers.stream().filter(n -> n == 0)      // zeros last
).collect(Collectors.toList());
// [1, 3, 5, 2, 4, 0, 0, 0, 0]
```

### ⚡ Key Points
- **Approach 2** (partitioning) is best — preserves relative order, single pass
- **Approach 3** (concat) is most readable but two passes over data
- **Approach 1** (sort) is clever but **not stable** for non-zero relative ordering
- *(Zeros ko end mein bhejna hai toh partition karo — non-zero alag, zero alag — phir join karo)*

---

<a id="q12"></a>
## Q12. Sort numbers that start with digit 1

### 💻 Code

```java
List<Integer> numbers = List.of(15, 200, 1, 100, 12, 50, 8, 130, 19, 3);

// ✅ Filter numbers starting with '1', then sort
List<Integer> startsWithOne = numbers.stream()
    .filter(n -> String.valueOf(n).startsWith("1"))
    .sorted()
    .collect(Collectors.toList());
// [1, 12, 15, 19, 100, 130]

// ✅ Sort entire list — numbers starting with 1 come first, then rest
List<Integer> sortedWithPriority = numbers.stream()
    .sorted(Comparator
        .comparing((Integer n) -> !String.valueOf(n).startsWith("1"))  // true=1(last), false=0(first)
        .thenComparingInt(n -> n))
    .collect(Collectors.toList());
// [1, 12, 15, 19, 100, 130, 3, 8, 50, 200]

// ✅ Just extract and sort numbers starting with 1 from an array
int[] arr = {15, 200, 1, 100, 12, 50, 8, 130, 19, 3};
int[] result = IntStream.of(arr)
    .filter(n -> String.valueOf(n).charAt(0) == '1')
    .sorted()
    .toArray();
// [1, 12, 15, 19, 100, 130]
```

### ⚡ Key Points
- Convert number to String, then `startsWith("1")` — simplest check
- For custom ordering: put "starts with 1" group first using boolean Comparator trick
- `IntStream.of(arr)` for primitive arrays — avoids boxing
- *(Number ko String mein convert karo aur startsWith("1") check karo — simple trick)*

---

<a id="q13"></a>
## Q13. Find first repeated character in a string

### 💻 Code

```java
String input = "swiss";

// ✅ Approach 1: LinkedHashSet — add() returns false for duplicates
Optional<Character> firstRepeated = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.collectingAndThen(
        Collectors.toList(),
        list -> {
            Set<Character> seen = new LinkedHashSet<>();
            return list.stream()
                .filter(c -> !seen.add(c))  // add() returns false if already present
                .findFirst();
        }
    ));
System.out.println(firstRepeated.orElse(null));  // 's' — first char that repeats

// ✅ Approach 2: Frequency map + stream (more readable)
Map<Character, Long> freq = input.chars()
    .mapToObj(c -> (char) c)
    .collect(Collectors.groupingBy(Function.identity(), LinkedHashMap::new, Collectors.counting()));

Optional<Character> firstRepeated2 = freq.entrySet().stream()
    .filter(e -> e.getValue() > 1)
    .map(Map.Entry::getKey)
    .findFirst();
// 's' — first character with count > 1 (LinkedHashMap preserves insertion order)

// ✅ Approach 3: Imperative with Set (often accepted in interviews)
Character firstRep = null;
Set<Character> seen = new HashSet<>();
for (char c : input.toCharArray()) {
    if (!seen.add(c)) {
        firstRep = c;
        break;
    }
}
// 's'
```

### ⚡ Key Points
- **First repeated** ≠ first non-repeated → opposite logic
- `LinkedHashMap` preserves insertion order — critical for "first" guarantee
- **Approach 3** (imperative) is O(n) time, O(k) space — interviewer may prefer this
- Compare with first **non**-repeated → [08-streams-coding-basic.md Q2](./08-streams-coding-basic.md#q2)
- *(First repeated matlab pehla character jo dobara aata hai — Set.add() false return kare toh wahi hai)*

---

## 💡 Interview Observations

Java 8 coding questions focus heavily on these 4 pillars:

| Pillar | Key APIs | Common Questions |
|--------|----------|-----------------|
| **Streams API** | `stream()`, `flatMap()`, `map()`, `filter()` | String parsing, data transformation |
| **Sorting & Comparator** | `comparing()`, `thenComparing()`, `reversed()` | Multi-field sorts, descending order |
| **Filtering & Mapping** | `filter()`, `map()`, `mapToObj()` | Salary thresholds, name extraction |
| **Grouping & Collectors** | `groupingBy()`, `counting()`, `partitioningBy()` | Frequency, department grouping |
