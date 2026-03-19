# ☕ Java Version Features — Interview Questions

> Key features introduced in Java 7, 8, 11, and 17.

---

## Q1. What are the Key Features of Java 7, 8, 11, and 17?

### 📝 One-Liner
Each Java version adds language features, API improvements, and performance enhancements.

### 🔑 Quick Answer
Java 7: try-with-resources, diamond operator. Java 8: lambdas, streams, Optional. Java 11: var, HttpClient, String methods. Java 17: sealed classes, records, pattern matching. *(har version mein naye features aaye — 8 mein lambdas, 17 mein records)*

### 📖 How It Works

**Java 7 (2011)**:
- Try-with-resources — auto-close resources *(automatically close ho jaate hain)*
- Diamond operator `<>` — type inference for generics
- Multi-catch: `catch (IOException | SQLException e)`
- Switch with strings
- NIO.2 `Files`, `Path` API

**Java 8 (2014)** — biggest change:
- Lambda expressions & functional interfaces
- Stream API for bulk data operations
- `Optional<T>` for null safety
- `default` and `static` methods in interfaces
- `java.time` API (LocalDate, Instant)
- Method references `Class::method`

**Java 11 (2018)** — first LTS after 8:
- `var` for local variables (from Java 10)
- New String methods: `isBlank()`, `strip()`, `lines()`, `repeat()`
- `HttpClient` API (standard)
- `Files.readString()`, `Files.writeString()`
- Single-file execution: `java Hello.java`

**Java 17 (2021)** — current LTS:
- Sealed classes/interfaces — restrict who can extend *(kaun extend kar sakta hai wo control karo)*
- Records — immutable data carriers *(POJO ka shortcut)*
- Pattern matching for instanceof
- Text blocks (from 15, finalized)
- Switch expressions (from 14, finalized)
- Helpful NullPointerException messages

### 🗣️ How to Say in Interview
"Java 8 was the biggest paradigm shift with lambdas and streams. Java 11 added quality-of-life improvements like the HttpClient and String methods. Java 17 brought modern features like records and sealed classes. In my project, we migrated from Java 8 to 17 and leveraged records for DTOs and text blocks for SQL templates."

### 💻 Code
```java
// Java 8: Lambda + Stream
List<String> names = list.stream()
    .filter(s -> s.startsWith("A"))
    .collect(Collectors.toList());

// Java 11: var + new String methods
var greeting = "  Hello  ";
greeting.strip();       // "Hello"
greeting.isBlank();     // false
"ha".repeat(3);         // "hahaha"

// Java 17: Record
record Point(int x, int y) {}
Point p = new Point(1, 2);

// Java 17: Pattern matching
if (obj instanceof String s) {
    System.out.println(s.length()); // no cast needed
}

// Java 17: Sealed class
sealed interface Shape permits Circle, Square {}
record Circle(double radius) implements Shape {}
record Square(double side) implements Shape {}
```

### 🆚 vs. Comparison

| Feature | Java 8 | Java 11 | Java 17 |
|---------|--------|---------|---------|
| LTS | Yes | Yes | Yes |
| Lambdas | ✅ Introduced | ✅ | ✅ |
| var | ❌ | ✅ (Java 10) | ✅ |
| Records | ❌ | ❌ | ✅ |
| Sealed Classes | ❌ | ❌ | ✅ |
| Text Blocks | ❌ | ❌ | ✅ |
| HttpClient | ❌ | ✅ | ✅ |

### ⚡ Remember
- Java 8 = lambdas, streams, Optional, java.time
- Java 11 = var, HttpClient, strip(), isBlank()
- Java 17 = records, sealed, pattern matching, switch expressions
- Always know which version YOUR project uses

---

## Q2. What is Try-with-Resources?

### 📝 One-Liner
A try statement that automatically closes resources implementing `AutoCloseable` when the block exits.

### 🔑 Quick Answer
Declare resources in the `try()` parentheses — they auto-close in reverse order when the block finishes, even if an exception occurs. *(try ke andar resource declare karo, automatically band ho jayega)*

### 📖 How It Works
- Resource must implement `AutoCloseable` (or `Closeable`)
- Declared in `try(Resource r = ...)` 
- Close happens in reverse declaration order
- If both try block and close throw exceptions → close exception is **suppressed** *(close wala exception suppress ho jaata hai, Throwable.getSuppressed() se milta hai)*
- Java 9+: can use effectively final variables declared outside

### 🗣️ How to Say in Interview
"Try-with-resources ensures automatic cleanup of resources like streams, connections, and readers. The resource must implement AutoCloseable. This eliminates the need for verbose finally blocks and prevents resource leaks."

### 💻 Code
```java
// Before Java 7 (verbose)
BufferedReader br = null;
try {
    br = new BufferedReader(new FileReader("file.txt"));
    return br.readLine();
} finally {
    if (br != null) br.close();
}

// Java 7+ try-with-resources
try (BufferedReader br = new BufferedReader(new FileReader("file.txt"))) {
    return br.readLine();
} // br.close() called automatically!

// Multiple resources (closed in reverse order)
try (Connection conn = dataSource.getConnection();
     PreparedStatement ps = conn.prepareStatement(sql);
     ResultSet rs = ps.executeQuery()) {
    while (rs.next()) { /* process */ }
} // rs → ps → conn closed in order

// Java 9+: effectively final variable
BufferedReader br = new BufferedReader(new FileReader("file.txt"));
try (br) { // no re-declaration needed
    return br.readLine();
}
```

### ⚠️ Pitfalls / Gotchas
- Resource must be `AutoCloseable` — can't use with arbitrary objects
- Suppressed exceptions: if try AND close both throw, close exception is suppressed *(close ka exception hide ho jaata hai)*
- Don't return resource reference from try block — it's closed!

### ⚡ Remember
- `AutoCloseable` = try-with-resources compatible
- Reverse close order (last declared = first closed)
- Suppressed exceptions accessible via `getSuppressed()`
- Java 9: effectively final vars allowed in try()

---
