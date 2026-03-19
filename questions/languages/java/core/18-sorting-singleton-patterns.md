# ☕ Design Patterns, Sorting & Singleton — Interview Questions

> Common Java pattern/utility questions from JPMorgan, Atlassian & product company interviews.

---

## Q1. How to Sort a Map in Java?

### 📝 One-Liner
Sort a Map by keys or values using TreeMap, streams, or LinkedHashMap.

### 🔑 Quick Answer
By keys: use `TreeMap`. By values: stream the entrySet, sort, collect into `LinkedHashMap`. *(keys ke liye TreeMap, values ke liye stream sort karo)*

### 📖 How It Works
- **By keys**: `new TreeMap<>(map)` — natural ordering
- **By values**: `map.entrySet().stream().sorted(Map.Entry.comparingByValue()).collect(...)` 
- Collect into `LinkedHashMap` to preserve insertion order *(LinkedHashMap order maintain karta hai)*
- For reverse order: `comparingByValue(Comparator.reverseOrder())`

### 🗣️ How to Say in Interview
"For sorting by keys, I'd use TreeMap which maintains natural key ordering. For sorting by values, I'd stream the entrySet, sort using Map.Entry.comparingByValue(), and collect into a LinkedHashMap to maintain the sorted order."

### 💻 Code
```java
Map<String, Integer> map = new HashMap<>();
map.put("Banana", 2); map.put("Apple", 5); map.put("Cherry", 1);

// Sort by keys
Map<String, Integer> byKeys = new TreeMap<>(map);
// {Apple=5, Banana=2, Cherry=1}

// Sort by values (ascending)
Map<String, Integer> byValues = map.entrySet().stream()
    .sorted(Map.Entry.comparingByValue())
    .collect(Collectors.toMap(
        Map.Entry::getKey, Map.Entry::getValue,
        (e1, e2) -> e1, LinkedHashMap::new));
// {Cherry=1, Banana=2, Apple=5}

// Sort by values (descending)
Map<String, Integer> byValuesDesc = map.entrySet().stream()
    .sorted(Map.Entry.<String, Integer>comparingByValue().reversed())
    .collect(Collectors.toMap(
        Map.Entry::getKey, Map.Entry::getValue,
        (e1, e2) -> e1, LinkedHashMap::new));
```

### ⚠️ Pitfalls / Gotchas
- HashMap has NO ordering — sorted result must go into LinkedHashMap
- TreeMap sorts by KEYS only, not values
- Merge function `(e1, e2) -> e1` handles duplicate keys in `toMap`

### ⚡ Remember
- By keys → `TreeMap`
- By values → `stream().sorted().collect(LinkedHashMap)`
- Always use `LinkedHashMap` to preserve sort order

---

## Q2. Write a Singleton Class (All Approaches)

### 📝 One-Liner
Singleton ensures only one instance of a class exists throughout the application.

### 🔑 Quick Answer
Best approaches: Enum singleton (simplest, safest), or Bill Pugh (lazy, thread-safe via inner class). Avoid double-checked locking unless necessary. *(Enum singleton sabse safe hai, reflection/serialization se bhi protect karta hai)*

### 📖 How It Works
- **Eager**: Instance created at class loading — simple but wastes memory if unused
- **Lazy (synchronized)**: Thread-safe but slow — entire method synchronized
- **Double-Checked Locking**: Only sync on first creation, use `volatile` *(volatile zaroori hai — instruction reordering se bachne ke liye)*
- **Bill Pugh (Inner Class)**: Lazy + thread-safe via class loader guarantee
- **Enum**: JVM guarantees single instance, serialization-safe, reflection-safe

### 🗣️ How to Say in Interview
"I prefer the Enum singleton for its simplicity and built-in protection against reflection and serialization attacks. For lazy initialization, I'd use the Bill Pugh approach with a static inner holder class, which is thread-safe without synchronization."

### 💻 Code
```java
// 1. Eager Initialization
public class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() {}
    public static EagerSingleton getInstance() { return INSTANCE; }
}

// 2. Double-Checked Locking
public class DCLSingleton {
    private static volatile DCLSingleton instance;
    private DCLSingleton() {}
    public static DCLSingleton getInstance() {
        if (instance == null) {
            synchronized (DCLSingleton.class) {
                if (instance == null) {
                    instance = new DCLSingleton();
                }
            }
        }
        return instance;
    }
}

// 3. Bill Pugh (Best lazy approach)
public class BillPughSingleton {
    private BillPughSingleton() {}
    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    public static BillPughSingleton getInstance() {
        return Holder.INSTANCE;
    }
}

// 4. Enum Singleton (Recommended)
public enum EnumSingleton {
    INSTANCE;
    public void doSomething() { /* ... */ }
}
```

### 🆚 vs. Comparison

| Approach | Thread-Safe | Lazy | Reflection-Safe | Serialization-Safe |
|----------|------------|------|----------------|-------------------|
| Eager | ✅ | ❌ | ❌ | ❌ |
| Synchronized | ✅ | ✅ | ❌ | ❌ |
| DCL | ✅ | ✅ | ❌ | ❌ |
| Bill Pugh | ✅ | ✅ | ❌ | ❌ |
| Enum | ✅ | ❌ | ✅ | ✅ |

### ⚠️ Pitfalls / Gotchas
- Without `volatile` in DCL → broken due to instruction reordering
- Reflection can break all non-enum singletons: `constructor.setAccessible(true)` *(reflection se private constructor bhi call ho sakta hai)*
- Serialization creates new instance on deserialization — add `readResolve()` to prevent

### ⚡ Remember
- **Enum** = best overall (Joshua Bloch recommended)
- **Bill Pugh** = best lazy non-enum approach
- Always make constructor `private`
- `volatile` is MANDATORY for DCL

---

## Q3. Common Design Patterns in Java

### 📝 One-Liner
Design patterns are reusable solutions to common software design problems, categorized as Creational, Structural, and Behavioral.

### 🔑 Quick Answer
Key patterns: Singleton, Factory, Builder (Creational); Adapter, Proxy, Decorator (Structural); Observer, Strategy, Template Method (Behavioral). *(teen category — banane, structure, behavior ke patterns)*

### 📖 How It Works

**Creational** — object creation:
| Pattern | Use Case | Java Example |
|---------|----------|-------------|
| Singleton | One global instance | `Runtime.getRuntime()` |
| Factory Method | Create objects without specifying class | `Calendar.getInstance()` |
| Abstract Factory | Family of related objects | `DocumentBuilderFactory` |
| Builder | Complex object construction | `StringBuilder`, Lombok `@Builder` |
| Prototype | Clone existing objects | `Object.clone()` |

**Structural** — composition:
| Pattern | Use Case | Java Example |
|---------|----------|-------------|
| Adapter | Incompatible interface bridge | `Arrays.asList()` |
| Decorator | Add behavior dynamically | `BufferedReader(FileReader)` |
| Proxy | Control access | Spring AOP proxies |
| Facade | Simplified interface | `SLF4J` logging facade |
| Composite | Tree structures | `Component` in Swing |

**Behavioral** — interaction:
| Pattern | Use Case | Java Example |
|---------|----------|-------------|
| Observer | Event notification | `PropertyChangeListener` |
| Strategy | Swappable algorithms | `Comparator` |
| Template Method | Algorithm skeleton | `AbstractList` |
| Iterator | Sequential access | `Iterator` interface |
| Command | Encapsulate requests | `Runnable` |

### 🗣️ How to Say in Interview
"In my projects, I've used several patterns. Builder for complex configuration objects, Strategy pattern via Comparators for flexible sorting, Observer pattern in event-driven microservices, and Factory pattern in Spring's BeanFactory. Spring itself heavily uses Proxy, Template Method, and Singleton patterns."

### 💻 Code
```java
// Factory Pattern
public interface PaymentProcessor {
    void process(double amount);
}
public class PaymentFactory {
    public static PaymentProcessor create(String type) {
        return switch (type) {
            case "CARD" -> new CardProcessor();
            case "UPI"  -> new UPIProcessor();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// Strategy Pattern
List<String> names = Arrays.asList("Charlie", "Alice", "Bob");
names.sort(Comparator.naturalOrder());     // strategy 1
names.sort(Comparator.reverseOrder());     // strategy 2
names.sort(Comparator.comparingInt(String::length)); // strategy 3

// Builder Pattern
public class UserDTO {
    private final String name;
    private final String email;
    private UserDTO(Builder b) { this.name = b.name; this.email = b.email; }
    public static class Builder {
        private String name, email;
        public Builder name(String n) { this.name = n; return this; }
        public Builder email(String e) { this.email = e; return this; }
        public UserDTO build() { return new UserDTO(this); }
    }
}
```

### ⚡ Remember
- Spring uses: Singleton, Factory, Proxy, Template Method, Observer
- Most asked in interviews: Singleton, Factory, Builder, Strategy, Observer
- Know at least 1 real-world example per pattern
- GoF = 23 patterns total (Creational 5, Structural 7, Behavioral 11)

---
