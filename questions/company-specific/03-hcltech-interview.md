# 🏢 HCLTech — Java Spring Boot Developer Interview Experience (3 Rounds)

> Walk-in interview for Java Spring Boot Developer role. Rounds: Online Assessment, Face-to-Face Technical, Managerial. Coding was on online compiler (no IDE — output mandatory).

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Round 1: Online Assessment (Screening)

_Standard online coding/MCQ round — cleared to proceed to walk-in._

---

## Round 2: Face-to-Face Technical

---

<a id="q1"></a>
## Q1. What are the key features introduced in Java 8?

### 📝 One-Liner
Java 8 introduced Lambda expressions, Stream API, Functional Interfaces, Optional, default/static methods in interfaces, and the new Date/Time API.

### 🔑 Quick Answer
Java 8 was a landmark release: Lambdas enable functional-style programming, Streams allow declarative collection processing, `Optional` handles null safely, and `@FunctionalInterface` ensures single abstract method contracts. Default methods in interfaces solved the interface evolution problem. *(Java 8 sabse bada change tha — lambdas aur streams se code concise aur functional ho gaya)*

### 📖 How It Works
```
Java 8 Major Features:
├── Lambda Expressions      → (params) -> expression
├── Functional Interfaces   → @FunctionalInterface (Predicate, Function, Consumer, Supplier)
├── Stream API              → filter/map/reduce on collections
├── Optional<T>             → null-safe container
├── Default Methods         → interface mein method body
├── Static Methods          → interface mein static utility methods
├── Method References       → Class::method shorthand
├── Date/Time API           → java.time (LocalDate, LocalDateTime, ZonedDateTime)
├── CompletableFuture       → async programming support
└── Nashorn JS Engine       → JavaScript runtime (removed in Java 15)
```

### 🗣️ Answering Approach
"Java 8 was a paradigm shift. The biggest additions were Lambda expressions for concise anonymous functions, the Stream API for declarative data processing pipelines, and Optional to combat null pointer issues. Functional interfaces like Predicate, Function, Consumer, and Supplier provide standard contracts for lambdas. Default methods in interfaces solved backward compatibility — existing implementations don't break when new methods are added. The java.time package finally gave us an immutable, thread-safe date/time API replacing the problematic java.util.Date."

### 💻 Code
```java
// Lambda + Stream + Optional
List<String> names = Arrays.asList("Alice", "Bob", "Charlie", "David");

// Filter + Map + Collect
List<String> result = names.stream()
    .filter(name -> name.length() > 3)
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// [ALICE, CHARLIE, DAVID]

// Optional
Optional<String> first = names.stream()
    .filter(n -> n.startsWith("Z"))
    .findFirst();
String value = first.orElse("Not Found"); // "Not Found"

// Default method in interface
interface Greeter {
    void greet(String name);
    default void greetAll(List<String> names) {
        names.forEach(this::greet);
    }
}

// Method reference
names.forEach(System.out::println);
```

### ⚠️ Pitfalls
- Streams are **not reusable** — calling terminal operation twice throws `IllegalStateException`
- `Optional.get()` without `isPresent()` check still throws `NoSuchElementException`
- Default methods can cause **diamond problem** if two interfaces have same default method

### ⚡ Remember
- Java 8 = Lambdas + Streams + Optional + Functional Interfaces + Default Methods + java.time
- `@FunctionalInterface` is optional annotation but good practice
- Streams are lazy — intermediate ops don't execute until terminal op

---

<a id="q2"></a>
## Q2. Explain the `static` keyword in Java and its role in OOP concepts

### 📝 One-Liner
`static` means the member belongs to the class rather than an instance — it's shared across all objects and accessible without creating an instance.

### 🔑 Quick Answer
Static applies to variables (shared state), methods (utility functions), blocks (class initialization), nested classes (no outer reference), and imports. In OOP, static breaks pure OO since it operates at class level, not instance level — no polymorphism on static methods. *(static matlab class ka member hai, object ka nahi — sabhi objects share karte hain)*

### 📖 How It Works
```
static keyword usage:
├── static variable     → shared across all instances (class variable)
├── static method       → called via ClassName.method(), no 'this' access
├── static block        → runs once when class loads (class initialization)
├── static nested class → does NOT hold reference to outer class
├── static import       → import static java.lang.Math.PI
└── static in interface → Java 8+ static utility methods
```

### 💻 Code
```java
public class Counter {
    private static int count = 0;     // shared across all instances
    private String name;

    static {
        System.out.println("Class loaded!"); // runs once
    }

    public Counter(String name) {
        this.name = name;
        count++;                        // incremented by every instance
    }

    public static int getCount() {      // no 'this' access
        // System.out.println(name);    // COMPILE ERROR — can't access instance var
        return count;
    }
}
```

### 🆚 vs.
| Aspect | Static | Instance |
|--------|--------|----------|
| Belongs to | Class | Object |
| Memory | Method Area (Metaspace) | Heap |
| Access | ClassName.member | object.member |
| Override | Cannot (hidden) | Can be overridden |
| `this` reference | Not available | Available |

### ⚡ Remember
- Static methods **cannot be overridden** — only hidden (method hiding)
- Static blocks execute in order of declaration, once per class loading
- `static` + `final` = compile-time constant (inlined by compiler)
- Avoid mutable static state in multi-threaded apps (use `AtomicInteger` instead)

---

<a id="q3"></a>
## Q3. What is Serialization in Java? When and how is it used?

### 📝 One-Liner
Serialization converts an object's state to a byte stream for storage/transmission; deserialization reconstructs the object back from bytes.

### 🔑 Quick Answer
Implement `Serializable` (marker interface), use `ObjectOutputStream` to write and `ObjectInputStream` to read. `serialVersionUID` ensures version compatibility. `transient` fields are skipped during serialization. *(Serialization = object ko bytes mein convert karna taaki save ya network pe bhej sako)*

### 📖 How It Works
```
   Object                    Byte Stream                    Object
 ┌─────────┐  serialize    ┌─────────────┐  deserialize  ┌─────────┐
 │ Employee │ ───────────► │ 01 AC ED 00 │ ────────────► │ Employee │
 │ name=Ram │  ObjectOut   │ 05 73 72 00 │  ObjectIn     │ name=Ram │
 │ age=30   │  putStream   └─────────────┘  putStream    │ age=30   │
 │ pwd=**** │                                             │ pwd=null │ ← transient
 └─────────┘                                             └─────────┘
```

### 💻 Code
```java
public class Employee implements Serializable {
    private static final long serialVersionUID = 1L;
    private String name;
    private int age;
    private transient String password; // NOT serialized

    // Serialize
    try (ObjectOutputStream oos = new ObjectOutputStream(new FileOutputStream("emp.ser"))) {
        oos.writeObject(new Employee("Ram", 30, "secret"));
    }

    // Deserialize
    try (ObjectInputStream ois = new ObjectInputStream(new FileInputStream("emp.ser"))) {
        Employee emp = (Employee) ois.readObject();
        // emp.password will be null (transient)
    }
}
```

### ⚠️ Pitfalls
- Missing `serialVersionUID` → JVM generates one based on class structure; any change breaks deserialization
- Parent class not `Serializable` → parent fields won't be serialized (parent must have no-arg constructor)
- `transient` + `static` both skip serialization but for different reasons
- **Security risk**: deserializing untrusted data can lead to remote code execution

### ⚡ Remember
- `Serializable` = marker interface (no methods)
- `Externalizable` = full control (must implement `readExternal`/`writeExternal`)
- Modern preference: use JSON (Jackson/Gson) over Java serialization
- `serialVersionUID` is a version control for the class

---

<a id="q4"></a>
## Q4. How does HashMap work internally? What changed after Java 8?

### 📝 One-Liner
HashMap uses an array of buckets with hash-based indexing; Java 8 converts long collision chains (≥8 nodes) from LinkedList to balanced Red-Black Tree for O(log n) lookup.

### 🔑 Quick Answer
`put(key, value)`: compute `hashCode()` → spread bits → bucket index via `(n-1) & hash`. Collisions handled by chaining. Java 8 optimization: when a bucket has ≥8 nodes AND table size ≥64, the linked list converts to a Red-Black Tree (treeifies). Untreeifies back at ≤6 nodes. *(Java 8 mein HashMap ka worst case O(n) se O(log n) ho gaya tree conversion ki wajah se)*

### 📖 How It Works
```
Before Java 8:                     After Java 8:
┌───────────────────┐              ┌───────────────────┐
│ Bucket 0 → null   │              │ Bucket 0 → null   │
│ Bucket 1 → A→B→C  │ O(n)        │ Bucket 1 → A→B→C  │ ≤7 nodes: LinkedList
│ Bucket 2 → D      │              │ Bucket 2 → D      │
│ Bucket 3 → null   │              │ Bucket 3 → TreeNode│ ≥8 nodes: Red-Black Tree
│ Bucket 4 → E→F    │              │        ┌──B──┐    │    O(log n)
└───────────────────┘              │       A      D    │
                                   └───────────────────┘
```

**Key changes in Java 8:**
1. **Treeification**: LinkedList → Red-Black Tree at 8 nodes (TREEIFY_THRESHOLD)
2. **Untreeify**: Tree → LinkedList at 6 nodes (UNTREEIFY_THRESHOLD)
3. **Hash spreading**: `hash = key.hashCode() ^ (key.hashCode() >>> 16)` — mixes high bits into low bits
4. **Resize optimization**: nodes either stay at same index or move to `oldIndex + oldCapacity`

### 💻 Code
```java
// Internal hash function (Java 8+)
static final int hash(Object key) {
    int h;
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}

// Bucket index calculation
int index = (n - 1) & hash; // n = table length (always power of 2)
```

### 🆚 vs.
| Aspect | Before Java 8 | After Java 8 |
|--------|---------------|--------------|
| Collision chain | LinkedList only | LinkedList → Red-Black Tree |
| Worst-case get() | O(n) | O(log n) |
| Hash function | Multiple rounds | Single XOR shift |
| Resize | Rehash all | Bit-check optimization |
| Node class | Entry<K,V> | Node<K,V> + TreeNode<K,V> |

### ⚡ Remember
- Default capacity: 16, Load factor: 0.75 → resize at 12 entries
- `TREEIFY_THRESHOLD = 8`, `UNTREEIFY_THRESHOLD = 6`, `MIN_TREEIFY_CAPACITY = 64`
- Null key always goes to bucket 0
- Keys must override both `hashCode()` AND `equals()` correctly

---

<a id="q5"></a>
## Q5. Explain SOLID principles with real examples

### 📝 One-Liner
SOLID = five design principles (SRP, OCP, LSP, ISP, DIP) for writing maintainable, extensible, testable OO code.

### 🔑 Quick Answer
**S**ingle Responsibility — one class, one reason to change. **O**pen/Closed — extend behavior without modifying existing code. **L**iskov Substitution — subtypes must be substitutable for base types. **I**nterface Segregation — many specific interfaces over one fat interface. **D**ependency Inversion — depend on abstractions, not concretions. *(SOLID follow karo toh code flexible aur testable rehta hai — har principle ka ek clear purpose hai)*

### 📖 How It Works
```
S → Single Responsibility    | UserService handles users, EmailService handles emails
O → Open/Closed              | Add new PaymentType via interface, don't modify PaymentProcessor
L → Liskov Substitution      | Rectangle r = new Square() should work without surprises
I → Interface Segregation    | Printer, Scanner, Fax — not one MegaMachine interface
D → Dependency Inversion     | OrderService depends on PaymentGateway interface, not StripeClient
```

### 💻 Code
```java
// SRP — separate concerns
class UserService { void createUser(User u) { /*...*/ } }
class EmailService { void sendWelcome(User u) { /*...*/ } }

// OCP — extend via abstraction
interface PaymentMethod { void pay(double amount); }
class CreditCard implements PaymentMethod { /*...*/ }
class UPI implements PaymentMethod { /*...*/ } // new type, no existing code change

// DIP — depend on abstraction
class OrderService {
    private final PaymentMethod payment; // interface, not concrete class
    OrderService(PaymentMethod payment) { this.payment = payment; }
}
```

### ⚡ Remember
- SRP reduces coupling, OCP enables extensibility, LSP ensures correctness
- ISP prevents forced implementations, DIP makes code testable (mock interfaces)
- Spring naturally encourages DIP through dependency injection

---

<a id="q6"></a>
## Q6. What are the different types of Singleton pattern? How do you ensure thread safety?

### 📝 One-Liner
Singleton ensures only one instance exists; implementations include Eager, Lazy, Double-Checked Locking, Bill Pugh (static inner class), and Enum Singleton.

### 🔑 Quick Answer
Eager initializes at class load (simple, wastes memory if unused). Lazy creates on first access (not thread-safe by default). Double-Checked Locking uses `volatile` + synchronized block. Bill Pugh uses static inner class (lazy + thread-safe). Enum Singleton is the simplest and safest (handles serialization + reflection). *(Enum singleton sabse best hai — thread-safe, serialization-safe, aur reflection-proof)*

### 📖 How It Works
| Type | Thread-Safe | Lazy | Serialization-Safe | Reflection-Safe |
|------|------------|------|-------------------|-----------------|
| Eager | ✅ | ❌ | ❌ (needs readResolve) | ❌ |
| Lazy (unsync) | ❌ | ✅ | ❌ | ❌ |
| Synchronized method | ✅ | ✅ | ❌ | ❌ |
| Double-Checked Lock | ✅ | ✅ | ❌ | ❌ |
| Bill Pugh | ✅ | ✅ | ❌ | ❌ |
| **Enum** | ✅ | ❌ | ✅ | ✅ |

### 💻 Code
```java
// 1. Eager
class EagerSingleton {
    private static final EagerSingleton INSTANCE = new EagerSingleton();
    private EagerSingleton() {}
    public static EagerSingleton getInstance() { return INSTANCE; }
}

// 2. Double-Checked Locking
class DCLSingleton {
    private static volatile DCLSingleton instance;
    private DCLSingleton() {}
    public static DCLSingleton getInstance() {
        if (instance == null) {
            synchronized (DCLSingleton.class) {
                if (instance == null) instance = new DCLSingleton();
            }
        }
        return instance;
    }
}

// 3. Bill Pugh (best non-enum approach)
class BillPughSingleton {
    private BillPughSingleton() {}
    private static class Holder {
        private static final BillPughSingleton INSTANCE = new BillPughSingleton();
    }
    public static BillPughSingleton getInstance() { return Holder.INSTANCE; }
}

// 4. Enum (recommended by Joshua Bloch)
enum EnumSingleton {
    INSTANCE;
    public void doSomething() { /*...*/ }
}
```

### ⚠️ Pitfalls
- `volatile` is mandatory in DCL — without it, partially constructed object can be seen
- Serialization breaks Singleton unless you add `readResolve()` returning existing instance
- Reflection can bypass private constructor — Enum is immune to this

### ⚡ Remember
- **Production choice**: Enum (simplest, safest) or Bill Pugh (lazy + no enum limitation)
- Spring beans are Singleton-scoped by default (managed by container, not the pattern)
- DCL's `volatile` prevents instruction reordering during object construction

---

<a id="q7"></a>
## Q7. Factory Pattern vs Abstract Factory Pattern — What's the difference?

### 📝 One-Liner
Factory creates objects of a single product family; Abstract Factory creates families of related objects through multiple factory methods.

### 🔑 Quick Answer
**Factory Method**: one method returns different implementations of a single interface based on input. **Abstract Factory**: an interface with multiple factory methods that create related/dependent objects as a family. Factory selects one product type; Abstract Factory selects an entire product suite. *(Factory ek type ka object banata hai, Abstract Factory poori family ka — jaise Dark Theme mein dark buttons + dark text fields sab saath mein)*

### 📖 How It Works
```
Factory Method:                    Abstract Factory:
┌──────────────┐                  ┌─────────────────────┐
│ ShapeFactory  │                  │ UIFactory (abstract) │
│ create(type) │                  │ createButton()       │
└──────┬───────┘                  │ createTextField()    │
       │                          └──────────┬──────────┘
  ┌────┼────┐                         ┌──────┼──────┐
  ▼    ▼    ▼                         ▼             ▼
Circle Rect Triangle          DarkUIFactory    LightUIFactory
                              createButton()   createButton()
                              → DarkButton     → LightButton
                              createTextField()createTextField()
                              → DarkTextField  → LightTextField
```

### 💻 Code
```java
// Factory Method
interface Shape { void draw(); }
class Circle implements Shape { public void draw() { System.out.println("Circle"); } }
class Rectangle implements Shape { public void draw() { System.out.println("Rect"); } }

class ShapeFactory {
    public static Shape create(String type) {
        return switch (type) {
            case "circle" -> new Circle();
            case "rect" -> new Rectangle();
            default -> throw new IllegalArgumentException("Unknown: " + type);
        };
    }
}

// Abstract Factory
interface Button { void render(); }
interface TextField { void render(); }
interface UIFactory { Button createButton(); TextField createTextField(); }

class DarkUIFactory implements UIFactory {
    public Button createButton() { return new DarkButton(); }
    public TextField createTextField() { return new DarkTextField(); }
}
```

### 🆚 vs.
| Aspect | Factory Method | Abstract Factory |
|--------|---------------|-----------------|
| Creates | Single product type | Family of related products |
| Abstraction level | One method | Multiple factory methods |
| Use case | "Give me a Shape" | "Give me a themed UI kit" |
| Complexity | Simple | Higher |
| Example | `LoggerFactory.getLogger()` | Cross-platform UI toolkit |

### ⚡ Remember
- Factory = one product hierarchy; Abstract Factory = multiple product hierarchies
- Spring uses both: `BeanFactory` (factory), `AbstractFactoryBean` (abstract factory)
- Real-world: JDBC `DriverManager.getConnection()` is a Factory example

---

<a id="q8"></a>
## Q8. What is the Saga Design Pattern in Microservices?

### 📝 One-Liner
Saga manages distributed transactions across microservices using a sequence of local transactions with compensating actions for rollback — no distributed ACID needed.

### 🔑 Quick Answer
Two types: **Choreography** (event-driven, each service listens and reacts) and **Orchestration** (central coordinator directs the flow). Each step has a compensating transaction to undo if any step fails. Saga replaces 2PC (two-phase commit) which doesn't scale in microservices. *(Saga = distributed transaction ko local transactions ki chain mein todna, fail hone pe compensating action se rollback)*

### 📖 How It Works
```
Choreography Saga (Event-based):
Order Created → [Payment Service] → Payment Done → [Inventory Service] → Reserved → [Shipping]
     ↑                                                                              │
     └──── Order Cancelled ◄── Payment Refunded ◄── Inventory Released ◄────────────┘
                              (compensating transactions on failure)

Orchestration Saga (Coordinator-based):
┌──────────────────┐
│  Saga Orchestrator│
│  (Order Service)  │
└────────┬─────────┘
    Step 1: Debit Payment ──► PaymentService
    Step 2: Reserve Stock  ──► InventoryService
    Step 3: Ship Order     ──► ShippingService
         │ if Step 3 fails:
    Comp 2: Release Stock  ──► InventoryService
    Comp 1: Refund Payment ──► PaymentService
```

### 🆚 vs.
| Aspect | Choreography | Orchestration |
|--------|-------------|---------------|
| Coordination | Decentralized (events) | Centralized (orchestrator) |
| Coupling | Loose | Tighter (to orchestrator) |
| Complexity | Hard to track flow | Easy to understand flow |
| Best for | Simple flows (2-3 services) | Complex flows (4+ services) |
| Tools | Kafka, RabbitMQ | Camunda, Temporal, AWS Step Functions |

### ⚠️ Pitfalls
- No isolation — intermediate states are visible to other transactions
- Compensating actions must be **idempotent** (safe to retry)
- Debugging choreography sagas is hard without distributed tracing
- Orchestrator can become a single point of failure

### ⚡ Remember
- Saga = alternative to 2PC for microservices distributed transactions
- Every step needs a compensating action (undo operation)
- Choreography for simple flows, Orchestration for complex business processes
- Real-world: e-commerce checkout, travel booking (flight + hotel + car)

---

<a id="q9"></a>
## Q9. DiscoveryClient vs RestTemplate vs Feign Client — When to use which?

### 📝 One-Liner
DiscoveryClient resolves service URLs from a registry, RestTemplate makes HTTP calls manually, and Feign Client provides declarative HTTP calls with built-in service discovery.

### 🔑 Quick Answer
**DiscoveryClient**: raw service lookup from Eureka/Consul — you get the URL, you build the call. **RestTemplate**: template-based HTTP client (blocking, deprecated in favor of WebClient). **Feign Client**: declarative interface — just define the API contract, Spring handles the rest (discovery + load balancing + error handling). *(Feign sabse clean hai — interface likho, Spring sab handle kar lega)*

### 📖 How It Works
```
DiscoveryClient:           RestTemplate:              Feign Client:
┌────────────────┐        ┌───────────────┐          ┌──────────────────┐
│ Get instances   │        │ Build URL      │          │ @FeignClient     │
│ from Eureka     │        │ Set headers    │          │ interface        │
│ Pick one (LB)   │        │ Make HTTP call │          │ @GetMapping      │
│ Build URL       │        │ Parse response │          │ ↓                │
│ Make HTTP call  │        └───────────────┘          │ Spring auto-     │
│ Parse response  │        Manual, verbose             │ generates proxy  │
└────────────────┘                                    └──────────────────┘
  Most control              Medium                     Least boilerplate
```

### 💻 Code
```java
// 1. DiscoveryClient — manual everything
@Autowired private DiscoveryClient discoveryClient;

public String callService() {
    List<ServiceInstance> instances = discoveryClient.getInstances("user-service");
    String url = instances.get(0).getUri() + "/api/users/1";
    return new RestTemplate().getForObject(url, String.class);
}

// 2. RestTemplate with @LoadBalanced
@Bean @LoadBalanced
public RestTemplate restTemplate() { return new RestTemplate(); }

public User getUser(Long id) {
    return restTemplate.getForObject("http://user-service/api/users/" + id, User.class);
}

// 3. Feign Client — declarative (recommended)
@FeignClient(name = "user-service")
public interface UserClient {
    @GetMapping("/api/users/{id}")
    User getUserById(@PathVariable Long id);
}

@Autowired private UserClient userClient;
public User getUser(Long id) { return userClient.getUserById(id); }
```

### 🆚 vs.
| Aspect | DiscoveryClient | RestTemplate | Feign Client |
|--------|----------------|-------------|-------------|
| Abstraction | Low | Medium | High |
| Service discovery | Manual | Via @LoadBalanced | Built-in |
| Boilerplate | High | Medium | Minimal |
| Error handling | Manual | Manual | Fallback support |
| Load balancing | Manual | Ribbon/Spring Cloud LB | Built-in |
| **Best for** | Custom logic | Legacy projects | New microservices |

### ⚡ Remember
- Feign Client = **recommended approach** — declarative, clean, integrates with Circuit Breaker
- RestTemplate is deprecated in Spring 5+ (use WebClient or RestClient instead)
- DiscoveryClient useful when you need fine-grained control over instance selection
- All three work with Eureka, Consul, or other service registries

---

<a id="q10"></a>
## Q10. What is the Circuit Breaker pattern? How does it work in microservices?

### 📝 One-Liner
Circuit Breaker prevents cascading failures by stopping calls to a failing service and returning a fallback response — like an electrical circuit breaker that trips to prevent overload.

### 🔑 Quick Answer
Three states: **Closed** (normal, calls pass through), **Open** (service down, return fallback immediately), **Half-Open** (test with limited calls to check recovery). Resilience4j is the standard library in Spring Cloud. Thresholds: failure rate %, slow call rate %, wait duration. *(Circuit Breaker = agar service fail ho rahi hai toh usse call karna band kar do, fallback do, thodi der baad retry karo)*

### 📖 How It Works
```
        ┌─────────────────────────────────────┐
        │          CLOSED (Normal)            │
        │  Calls pass through. Counting       │
        │  failures. If failure rate > 50%    │
        └──────────────┬──────────────────────┘
                       │ threshold exceeded
                       ▼
        ┌─────────────────────────────────────┐
        │          OPEN (Tripped)             │
        │  All calls rejected → fallback      │
        │  Wait for cooldown period           │
        └──────────────┬──────────────────────┘
                       │ wait duration elapsed
                       ▼
        ┌─────────────────────────────────────┐
        │         HALF-OPEN (Testing)         │
        │  Allow limited calls to check       │
        │  If success → CLOSED                │
        │  If failure → OPEN again            │
        └─────────────────────────────────────┘
```

### 💻 Code
```java
// Resilience4j with Spring Boot
@CircuitBreaker(name = "userService", fallbackMethod = "fallbackGetUser")
public User getUser(Long id) {
    return userClient.getUserById(id);
}

public User fallbackGetUser(Long id, Throwable t) {
    return new User(id, "Default User", "Unavailable");
}

// application.yml
// resilience4j.circuitbreaker.instances.userService:
//   failure-rate-threshold: 50
//   slow-call-duration-threshold: 2s
//   wait-duration-in-open-state: 30s
//   permitted-number-of-calls-in-half-open-state: 3
//   sliding-window-size: 10
```

### ⚠️ Pitfalls
- Fallback method must have **same signature** + Throwable parameter
- Don't set wait duration too low — gives failing service no recovery time
- Circuit breaker per-instance vs per-service: understand the scope
- Without proper monitoring, you won't know when circuits are open

### ⚡ Remember
- Three states: Closed → Open → Half-Open → Closed
- Resilience4j replaced Hystrix (Netflix OSS, deprecated)
- Combine with **Retry** and **Timeout** patterns for robust resilience
- Always provide meaningful fallbacks, not just null/empty

---

<a id="q11"></a>
## Q11. CompletableFuture vs Future — Explain and write sample code

### 📝 One-Liner
`Future` blocks on `get()` with no callback support; `CompletableFuture` supports non-blocking chaining, combining, and exception handling — true async programming.

### 🔑 Quick Answer
`Future` (Java 5): submit task to executor, call `get()` to block and wait. No way to chain or combine results. `CompletableFuture` (Java 8): supports `thenApply`, `thenCompose`, `thenCombine`, `exceptionally` — full async pipeline without blocking. *(Future mein result ke liye block hona padta hai, CompletableFuture mein chain kar sakte ho bina block kiye)*

### 📖 How It Works
```
Future (blocking):
  submit(task) → Future → .get() [BLOCKS thread] → result

CompletableFuture (non-blocking):
  supplyAsync(task)
    .thenApply(transform)
    .thenCompose(asyncOp)
    .thenCombine(otherFuture, merge)
    .exceptionally(handleError)
    .thenAccept(consume)
```

### 💻 Code
```java
// --- Future (Java 5) - blocking approach ---
ExecutorService executor = Executors.newFixedThreadPool(2);
Future<Integer> future = executor.submit(() -> {
    Thread.sleep(1000);
    return 42;
});
Integer result = future.get(); // BLOCKS until done
System.out.println("Result: " + result);
executor.shutdown();

// --- CompletableFuture (Java 8) - non-blocking ---
CompletableFuture<String> cf = CompletableFuture
    .supplyAsync(() -> fetchUserFromDB(userId))       // async DB call
    .thenApply(user -> user.getName().toUpperCase())   // transform
    .thenCombine(
        CompletableFuture.supplyAsync(() -> fetchOrderCount(userId)),
        (name, count) -> name + " has " + count + " orders"
    )
    .exceptionally(ex -> "Error: " + ex.getMessage()); // error handling

cf.thenAccept(System.out::println); // non-blocking consumption

// Running multiple async tasks in parallel
CompletableFuture<Void> all = CompletableFuture.allOf(
    CompletableFuture.supplyAsync(() -> callServiceA()),
    CompletableFuture.supplyAsync(() -> callServiceB()),
    CompletableFuture.supplyAsync(() -> callServiceC())
);
all.join(); // wait for all
```

### 🆚 vs.
| Aspect | Future | CompletableFuture |
|--------|--------|-------------------|
| Java version | 5 | 8 |
| Blocking | `get()` blocks | Non-blocking callbacks |
| Chaining | ❌ Not possible | ✅ thenApply, thenCompose |
| Combining | ❌ | ✅ thenCombine, allOf, anyOf |
| Exception handling | try-catch on get() | exceptionally(), handle() |
| Manual completion | ❌ | ✅ complete(), completeExceptionally() |
| Cancel | cancel(true) | cancel(true) + completeExceptionally |

### ⚡ Remember
- `thenApply` = synchronous transform (like map), `thenCompose` = async chain (like flatMap)
- `thenCombine` = merge two futures, `allOf` = wait for all, `anyOf` = wait for fastest
- Default thread pool: `ForkJoinPool.commonPool()` — pass custom executor for production

---

<a id="q12"></a>
## Q12. map() vs flatMap() in Streams — Explain and implement flattening

### 📝 One-Liner
`map()` transforms each element 1:1; `flatMap()` transforms each element to a stream and flattens all streams into one — useful for nested structures.

### 🔑 Quick Answer
`map(fn)` applies function to each element, maintains same stream count. `flatMap(fn)` applies function that returns a Stream, then merges/flattens all into a single stream. Use `flatMap` when each element maps to multiple values (lists of lists, optional chains). *(map = ek element se ek result, flatMap = ek element se multiple results jo flatten ho jaate hain)*

### 💻 Code
```java
// map — 1:1 transformation
List<String> names = List.of("alice", "bob");
List<String> upper = names.stream()
    .map(String::toUpperCase)
    .collect(Collectors.toList());
// ["ALICE", "BOB"]

// map on nested list — PROBLEM: Stream<List<String>>
List<List<String>> nested = List.of(
    List.of("a", "b"),
    List.of("c", "d"),
    List.of("e")
);
List<List<String>> mapped = nested.stream()
    .map(list -> list)  // still List<List<String>> — NOT flat!
    .collect(Collectors.toList());

// flatMap — SOLUTION: flatten nested to single stream
List<String> flat = nested.stream()
    .flatMap(Collection::stream)  // each List → Stream → merged
    .collect(Collectors.toList());
// ["a", "b", "c", "d", "e"]

// Real-world: get all order items from all orders
List<String> allItems = orders.stream()
    .flatMap(order -> order.getItems().stream())
    .distinct()
    .collect(Collectors.toList());

// flatMap with Optional (Java 9+)
Optional<String> result = Optional.of("hello")
    .flatMap(s -> Optional.of(s.toUpperCase()));
```

### 🆚 vs.
| Aspect | map() | flatMap() |
|--------|-------|-----------|
| Returns | `Stream<R>` | `Stream<R>` (flattened) |
| Function type | `T → R` | `T → Stream<R>` |
| Output size | Same as input | Can be different |
| Use case | Transform values | Flatten nested structures |
| Example | `["a","b"] → ["A","B"]` | `[["a","b"],["c"]] → ["a","b","c"]` |

### ⚡ Remember
- `map` = one-to-one, `flatMap` = one-to-many (then flatten)
- `flatMap` is commonly used for: List<List<T>>, Optional chaining, splitting strings
- Interview tip: "flatMap = map + flatten in one step"

---

<a id="q13"></a>
## Q13. Find the highest salary in each department using Java Streams

### 📝 One-Liner
Group employees by department and find the max salary using `Collectors.groupingBy` + `Collectors.maxBy`.

### 🔑 Quick Answer
Use `stream().collect(Collectors.groupingBy(Employee::getDepartment, Collectors.maxBy(Comparator.comparingDouble(Employee::getSalary))))`. This groups by department and picks the max-salary employee in each group. *(groupingBy se department-wise group karo, maxBy se highest salary nikalo)*

### 💻 Code
```java
record Employee(String name, String department, double salary) {}

List<Employee> employees = List.of(
    new Employee("Alice", "IT", 90000),
    new Employee("Bob", "IT", 120000),
    new Employee("Charlie", "HR", 85000),
    new Employee("Diana", "HR", 95000),
    new Employee("Eve", "Finance", 110000)
);

// Highest salary employee per department
Map<String, Optional<Employee>> highestPerDept = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.maxBy(Comparator.comparingDouble(Employee::salary))
    ));
// {IT=Optional[Employee[name=Bob, salary=120000]],
//  HR=Optional[Employee[name=Diana, salary=95000]],
//  Finance=Optional[Employee[name=Eve, salary=110000]]}

// Just the salary values (no Optional)
Map<String, Double> maxSalaries = employees.stream()
    .collect(Collectors.groupingBy(
        Employee::department,
        Collectors.collectingAndThen(
            Collectors.maxBy(Comparator.comparingDouble(Employee::salary)),
            opt -> opt.map(Employee::salary).orElse(0.0)
        )
    ));
// {IT=120000.0, HR=95000.0, Finance=110000.0}

// Alternative: using toMap with merge function
Map<String, Double> maxSalaries2 = employees.stream()
    .collect(Collectors.toMap(
        Employee::department,
        Employee::salary,
        Math::max
    ));
```

### ⚡ Remember
- `groupingBy` + `maxBy` = classic grouping + aggregation combo
- `collectingAndThen` unwraps Optional from maxBy
- `toMap` with merge function (`Math::max`) is a concise alternative
- This is one of the **most frequently asked** Stream coding questions

---

## Round 3: Managerial Round (Face-to-Face)

---

<a id="q14"></a>
## Q14. What is a Marker Interface? What are its real-time use cases?

### 📝 One-Liner
A marker interface has no methods — it "marks" or tags a class to indicate a capability or behavior that the JVM or framework checks at runtime.

### 🔑 Quick Answer
`Serializable`, `Cloneable`, `Remote` are classic marker interfaces. JVM checks `instanceof Serializable` before serializing. Modern alternative: annotations (`@Entity`, `@Transactional`). Marker interfaces are still useful when you need type checking at compile-time (annotations can't do that). *(Marker interface = empty interface jo class ko tag karti hai — jaise Serializable batata hai ki ye object serialize ho sakta hai)*

### 📖 How It Works
```
Marker Interface:                  Modern Annotation:
interface Serializable {}          @Retention(RUNTIME)
                                   @interface Cacheable {}

class Employee implements          @Cacheable
    Serializable { }              class Employee { }

// Runtime check                   // Runtime check
if (obj instanceof Serializable)   if (cls.isAnnotationPresent(Cacheable.class))
```

### 💻 Code
```java
// Custom marker interface
public interface Auditable {}

public class Employee implements Auditable {
    private String name;
    // ...
}

// Usage — type-safe check
public void save(Object entity) {
    if (entity instanceof Auditable) {
        logAuditTrail(entity);
    }
    repository.save(entity);
}
```

### 🆚 vs.
| Aspect | Marker Interface | Annotation |
|--------|-----------------|------------|
| Compile-time type check | ✅ (instanceof) | ❌ |
| Metadata attachment | ❌ | ✅ (can carry values) |
| Inheritance | ✅ Auto-inherited | Only with @Inherited |
| Examples | Serializable, Cloneable | @Entity, @Override |

### ⚡ Remember
- Java built-in markers: `Serializable`, `Cloneable`, `Remote`, `EventListener`
- Post-Java-5, annotations largely replaced marker interfaces
- Marker interfaces still win when you need **compile-time type safety**

---

<a id="q15"></a>
## Q15. How to handle method overloading when parameters keep increasing (1 to 100)?

### 📝 One-Liner
Use varargs, Builder pattern, or a parameter object — never create 100 overloaded methods.

### 🔑 Quick Answer
Three approaches: **(1) Varargs** `method(String... args)` for same-type params. **(2) Builder pattern** for objects with many optional fields. **(3) Parameter Object** — wrap related params in a POJO. Also: method chaining (fluent API) and Java records for immutable param objects. *(100 parameters ke liye 100 methods mat banao — Builder pattern ya Parameter Object use karo)*

### 📖 How It Works
```
Problem: createUser(name) → createUser(name, email) → createUser(name, email, phone) → ... ×100

Solutions:
1. Varargs:        createUser(String... params)          — simple but type-unsafe
2. Builder:        User.builder().name("X").email("Y")   — most flexible
3. Param Object:   createUser(UserRequest req)            — groups related params
4. Map:            createUser(Map<String, Object> params) — dynamic but type-unsafe
```

### 💻 Code
```java
// Builder Pattern (recommended for many optional params)
public class User {
    private final String name;      // required
    private final String email;     // optional
    private final String phone;     // optional
    private final int age;          // optional

    private User(Builder b) {
        this.name = b.name; this.email = b.email;
        this.phone = b.phone; this.age = b.age;
    }

    public static class Builder {
        private final String name;   // required
        private String email, phone;
        private int age;

        public Builder(String name) { this.name = name; }
        public Builder email(String e) { this.email = e; return this; }
        public Builder phone(String p) { this.phone = p; return this; }
        public Builder age(int a) { this.age = a; return this; }
        public User build() { return new User(this); }
    }
}

// Usage
User user = new User.Builder("Ram")
    .email("ram@example.com")
    .age(30)
    .build();

// Varargs for same-type params
public int sum(int... numbers) {
    return Arrays.stream(numbers).sum();
}

// Java Priority: Exact match > Widening > Autoboxing > Varargs
```

### ⚡ Remember
- **Varargs priority**: Java resolves: exact match → widening → autoboxing → varargs (lowest priority)
- Builder pattern: Lombok's `@Builder` does this automatically
- Effective Java Item 2: "Consider a builder when faced with many constructor parameters"

---

<a id="q16"></a>
## Q16. What are the commonly used methods in the Arrays class? What does BinarySearch return?

### 📝 One-Liner
`java.util.Arrays` provides utility methods for sorting, searching, filling, copying, and converting arrays — `binarySearch` returns the index if found, or `-(insertion point) - 1` if not found.

### 🔑 Quick Answer
Key methods: `sort()`, `binarySearch()`, `copyOf()`, `copyOfRange()`, `fill()`, `equals()`, `deepEquals()`, `asList()`, `stream()`, `toString()`. `binarySearch` **requires sorted array** — returns index if found, else `-(insertion point) - 1`. *(binarySearch ka negative result insertion point batata hai — array sorted hona zaroori hai)*

### 💻 Code
```java
int[] arr = {10, 20, 30, 40, 50};

// Sort
Arrays.sort(arr);

// BinarySearch — array MUST be sorted
int idx1 = Arrays.binarySearch(arr, 30);  // returns 2 (found at index 2)
int idx2 = Arrays.binarySearch(arr, 25);  // returns -3 (-(2)-1 = -3, insertion point 2)
int idx3 = Arrays.binarySearch(arr, 5);   // returns -1 (-(0)-1 = -1, insertion point 0)
int idx4 = Arrays.binarySearch(arr, 55);  // returns -6 (-(5)-1 = -6, insertion point 5)

// Other useful methods
int[] copy = Arrays.copyOf(arr, 10);           // copy with new length (pads with 0)
int[] range = Arrays.copyOfRange(arr, 1, 3);   // [20, 30]
Arrays.fill(arr, 0);                            // all elements = 0
boolean eq = Arrays.equals(arr1, arr2);          // content comparison
List<Integer> list = Arrays.asList(1, 2, 3);    // fixed-size list (no add/remove)
String str = Arrays.toString(arr);               // "[10, 20, 30, 40, 50]"
IntStream stream = Arrays.stream(arr);           // for stream operations
```

### ⚠️ Pitfalls
- `binarySearch` on unsorted array gives **undefined results**
- `Arrays.asList()` returns fixed-size list backed by array — `add()`/`remove()` throws `UnsupportedOperationException`
- `equals()` vs `deepEquals()` — use `deepEquals` for multi-dimensional arrays

### ⚡ Remember
- BinarySearch return: **found → index**, **not found → -(insertion point) - 1**
- Insertion point = index where the element would be inserted to keep array sorted
- `Arrays.sort()` uses Dual-Pivot Quicksort for primitives, TimSort for objects

---

<a id="q17"></a>
## Q17. Reverse a sentence — keep words intact, reverse their order

### 📝 One-Liner
Split the sentence by spaces, reverse the word array, and join back — or use StringBuilder to build result from end to start.

### 💻 Code
```java
// Approach 1: Using Stream
public String reverseSentence(String sentence) {
    String[] words = sentence.trim().split("\\s+");
    return IntStream.rangeClosed(1, words.length)
        .mapToObj(i -> words[words.length - i])
        .collect(Collectors.joining(" "));
}

// Approach 2: Using Collections.reverse
public String reverseSentence2(String sentence) {
    List<String> words = Arrays.asList(sentence.trim().split("\\s+"));
    Collections.reverse(words);
    return String.join(" ", words);
}

// Approach 3: Two-pointer (in-place for char array)
public String reverseSentence3(String sentence) {
    char[] chars = sentence.trim().toCharArray();
    reverse(chars, 0, chars.length - 1);       // reverse entire string
    int start = 0;
    for (int i = 0; i <= chars.length; i++) {
        if (i == chars.length || chars[i] == ' ') {
            reverse(chars, start, i - 1);       // reverse each word back
            start = i + 1;
        }
    }
    return new String(chars);
}

private void reverse(char[] arr, int l, int r) {
    while (l < r) { char t = arr[l]; arr[l++] = arr[r]; arr[r--] = t; }
}

// Test
System.out.println(reverseSentence("Hello World Java"));
// Output: "Java World Hello"
```

### ⚡ Remember
- `split("\\s+")` handles multiple spaces between words
- Two-pointer approach: reverse entire string → reverse each word (in-place)
- Edge cases: leading/trailing spaces, multiple spaces, single word

---

<a id="q18"></a>
## Q18. How to apply code changes without restarting the Spring Boot server (Hot Reload)?

### 📝 One-Liner
Use Spring Boot DevTools for auto-restart, or JRebel for true hot-reload — DevTools restarts the app context within seconds using two classloaders.

### 🔑 Quick Answer
**Spring Boot DevTools** (free): monitors classpath changes, triggers fast restart using restart classloader. **JRebel** (paid): true hot-swap without restart — reloads classes, beans, resources in-place. DevTools also enables live reload for browser, disables caches in dev. *(DevTools se code change karne pe server apne aap restart hota hai — bohot fast hai kyunki sirf app classes reload hoti hain)*

### 📖 How It Works
```
DevTools Architecture:
┌─────────────────────────────────────────┐
│ Base ClassLoader (libraries, JDK)       │ ← never reloaded
├─────────────────────────────────────────┤
│ Restart ClassLoader (your app classes)  │ ← discarded & recreated on change
└─────────────────────────────────────────┘
                    │
        File change detected (classpath)
                    │
        Restart ClassLoader recreated
        (~1-2 seconds restart)
```

### 💻 Code
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>  <!-- not included in production JAR -->
</dependency>
```
```properties
# application.properties
spring.devtools.restart.enabled=true
spring.devtools.livereload.enabled=true
spring.devtools.restart.additional-paths=src/main/java
# Exclude paths from triggering restart
spring.devtools.restart.exclude=static/**,templates/**
```

### 🆚 vs.
| Feature | DevTools | JRebel | JVM HotSwap |
|---------|----------|--------|-------------|
| Cost | Free | Paid (~$500/yr) | Free |
| Speed | ~1-2s restart | Instant | Instant |
| Method body change | ✅ | ✅ | ✅ |
| Add new method/class | ✅ (restart) | ✅ (no restart) | ❌ |
| Change bean config | ✅ (restart) | ✅ | ❌ |
| Production use | ❌ Auto-disabled | ❌ | ❌ |

### ⚡ Remember
- DevTools auto-disables in production (when run as `java -jar`)
- IDE setup: enable "Build project automatically" + "Allow auto-make when running"
- Trigger file: create `.reloadtrigger` for manual restart control
- DevTools also configures: `spring.thymeleaf.cache=false`, `spring.template.cache=false`

---

<a id="q19"></a>
## Q19. How to switch embedded server from Tomcat to Jetty in Spring Boot?

### 📝 One-Liner
Exclude the default Tomcat starter dependency and add the Jetty starter — Spring Boot auto-configures whichever server is on classpath.

### 🔑 Quick Answer
Exclude `spring-boot-starter-tomcat` from `spring-boot-starter-web`, then add `spring-boot-starter-jetty`. Spring Boot auto-detects: Tomcat > Jetty > Undertow (priority order based on classpath). Same approach for Undertow. *(Tomcat ko exclude karo, Jetty add karo — Spring Boot automatically detect kar lega)*

### 💻 Code
```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```
```properties
# Jetty-specific configuration
server.port=8080
server.jetty.threads.max=200
server.jetty.threads.min=8
server.jetty.connection-idle-timeout=30000
```

### 🆚 vs.
| Server | Best For | HTTP/2 | Virtual Threads (Java 21) |
|--------|----------|--------|--------------------------|
| Tomcat | General purpose (default) | ✅ | ✅ |
| Jetty | Async/WebSocket heavy | ✅ | ✅ |
| Undertow | High throughput | ✅ | ✅ |
| Netty | Reactive (WebFlux) | ✅ | N/A |

### ⚡ Remember
- Tomcat is default because `spring-boot-starter-web` includes it
- `spring-boot-starter-webflux` defaults to Netty (reactive)
- All embedded servers support the same `server.*` properties
- For performance testing, Undertow often edges out in raw throughput

---

<a id="q20"></a>
## Q20. What are the different ways to read values from application.properties in Spring Boot?

### 📝 One-Liner
Five main ways: `@Value`, `Environment`, `@ConfigurationProperties`, `@PropertySource`, and programmatic `Properties` loading.

### 🔑 Quick Answer
**@Value("${key}")** — inject single value. **Environment.getProperty()** — programmatic lookup. **@ConfigurationProperties(prefix)** — bind group to POJO (best for structured config). **@PropertySource** — load custom file. **Properties** — raw Java I/O. *(Structured config ke liye @ConfigurationProperties best hai, single value ke liye @Value)*

### 💻 Code
```java
// 1. @Value — single property injection
@Value("${app.name}")
private String appName;

@Value("${app.timeout:5000}")  // with default value
private int timeout;

// 2. Environment — programmatic access
@Autowired
private Environment env;
String dbUrl = env.getProperty("spring.datasource.url");

// 3. @ConfigurationProperties — bind to POJO (recommended for groups)
@Component
@ConfigurationProperties(prefix = "app.mail")
public class MailConfig {
    private String host;
    private int port;
    private String from;
    // getters/setters auto-bound from app.mail.host, app.mail.port, etc.
}

// 4. @PropertySource — load custom properties file
@Configuration
@PropertySource("classpath:custom-config.properties")
public class CustomConfig {
    @Value("${custom.key}")
    private String customValue;
}

// 5. application.properties
// app.name=MyApp
// app.timeout=5000
// app.mail.host=smtp.gmail.com
// app.mail.port=587
// app.mail.from=noreply@myapp.com
```

### ⚠️ Pitfalls
- `@Value` fails at startup if property missing (unless default provided)
- `@ConfigurationProperties` needs `@EnableConfigurationProperties` or `@ConfigurationPropertiesScan`
- `@PropertySource` does NOT support YAML files (only `.properties`)
- Profile-specific: `application-{profile}.properties` overrides base file

### ⚡ Remember
- **@Value**: quick single values | **@ConfigurationProperties**: structured POJO binding
- `@ConfigurationProperties` supports validation with `@Validated` + JSR-303 annotations
- Priority order: command-line args > env vars > application-{profile}.properties > application.properties

---

<a id="q21"></a>
## Q21. How to bind configuration properties to a POJO using @ConfigurationProperties?

### 📝 One-Liner
Annotate a class with `@ConfigurationProperties(prefix = "...")` and Spring Boot auto-binds matching properties from application.properties/yml to the POJO fields.

### 🔑 Quick Answer
Create a POJO with fields matching property keys, annotate with `@ConfigurationProperties(prefix)`. Register via `@Component` or `@EnableConfigurationProperties`. Supports nested objects, lists, maps, and validation. *(Properties file ke values automatically POJO mein bind ho jaate hain — manual parsing ki zaroorat nahi)*

### 💻 Code
```java
// POJO
@Component
@ConfigurationProperties(prefix = "app.datasource")
@Validated
public class DataSourceConfig {
    @NotBlank
    private String url;
    private String username;
    private String password;
    private Pool pool = new Pool();

    public static class Pool {
        private int maxSize = 10;
        private int minIdle = 2;
        private Duration timeout = Duration.ofSeconds(30);
        // getters and setters
    }
    // getters and setters
}
```
```yaml
# application.yml
app:
  datasource:
    url: jdbc:mysql://localhost:3306/mydb
    username: admin
    password: secret
    pool:
      max-size: 20
      min-idle: 5
      timeout: 60s
```
```java
// Usage — inject the config POJO
@Service
public class DatabaseService {
    private final DataSourceConfig config;

    public DatabaseService(DataSourceConfig config) {
        this.config = config;
    }

    public void connect() {
        String url = config.getUrl(); // "jdbc:mysql://localhost:3306/mydb"
        int maxPool = config.getPool().getMaxSize(); // 20
    }
}
```

### ⚡ Remember
- Relaxed binding: `maxSize`, `max-size`, `MAX_SIZE` all map to same field
- Use `@Validated` + `@NotBlank`, `@Min`, `@Max` for startup validation
- Immutable alternative: `@ConstructorBinding` with record classes (Spring Boot 3+)
- Generate metadata: add `spring-boot-configuration-processor` for IDE autocomplete

---

<a id="q22"></a>
## Q22. How to load values from a custom properties file in Spring Boot?

### 📝 One-Liner
Use `@PropertySource("classpath:custom.properties")` on a `@Configuration` class, or use `spring.config.import` (Spring Boot 2.4+) in application.properties.

### 🔑 Quick Answer
Three approaches: **@PropertySource** on config class (classic), **spring.config.import** in application.properties (modern), or **PropertySourcesPlaceholderConfigurer** bean. `@PropertySource` doesn't support YAML — use `spring.config.import` for YAML files. *(Custom properties file load karne ke liye @PropertySource ya spring.config.import use karo)*

### 💻 Code
```java
// Approach 1: @PropertySource (only .properties, not YAML)
@Configuration
@PropertySource("classpath:payment-config.properties")
public class PaymentConfig {
    @Value("${payment.gateway.url}")
    private String gatewayUrl;

    @Value("${payment.retry.count:3}")
    private int retryCount;
}

// Approach 2: spring.config.import (Spring Boot 2.4+, supports YAML)
// In application.properties:
// spring.config.import=classpath:payment-config.properties,classpath:email-config.yml

// Approach 3: Multiple property sources with ordering
@Configuration
@PropertySources({
    @PropertySource("classpath:default-config.properties"),
    @PropertySource(value = "classpath:override-config.properties", ignoreResourceNotFound = true)
})
public class MultiConfig {}

// Approach 4: Programmatic loading
@Bean
public static PropertySourcesPlaceholderConfigurer properties() {
    PropertySourcesPlaceholderConfigurer configurer = new PropertySourcesPlaceholderConfigurer();
    configurer.setLocations(new ClassPathResource("custom.properties"));
    return configurer;
}
```

### ⚡ Remember
- `@PropertySource` does NOT support `.yml` files — only `.properties`
- `spring.config.import` (Spring Boot 2.4+) supports both formats
- `ignoreResourceNotFound = true` prevents startup failure if file missing
- External file: `@PropertySource("file:/opt/config/app.properties")`

---

<a id="q23"></a>
## Q23. How to implement composite primary keys in a JPA entity?

### 📝 One-Liner
Two approaches: `@IdClass` (separate key class, IDs on entity fields) or `@EmbeddedId` (embedded key object) — both require the key class to implement `Serializable` with `equals()`/`hashCode()`.

### 🔑 Quick Answer
**@IdClass**: put `@Id` on each key field in the entity, reference a separate key class. **@EmbeddedId**: embed a key object with `@EmbeddedId` in the entity. `@EmbeddedId` is cleaner for OOP; `@IdClass` is simpler for JPQL queries. *(Composite key = multiple columns milke primary key banate hain — @EmbeddedId ya @IdClass se implement karo)*

### 💻 Code
```java
// --- Approach 1: @EmbeddedId ---
@Embeddable
public class OrderItemId implements Serializable {
    private Long orderId;
    private Long productId;
    // equals() and hashCode() MANDATORY
    @Override
    public boolean equals(Object o) { /* compare both fields */ }
    @Override
    public int hashCode() { return Objects.hash(orderId, productId); }
}

@Entity
public class OrderItem {
    @EmbeddedId
    private OrderItemId id;
    private int quantity;
    private double price;
}

// JPQL: SELECT o FROM OrderItem o WHERE o.id.orderId = :orderId

// --- Approach 2: @IdClass ---
@Entity
@IdClass(OrderItemId.class)
public class OrderItem {
    @Id private Long orderId;
    @Id private Long productId;
    private int quantity;
    private double price;
}

// JPQL: SELECT o FROM OrderItem o WHERE o.orderId = :orderId
// (simpler query — no .id. prefix needed)

// Repository
public interface OrderItemRepo extends JpaRepository<OrderItem, OrderItemId> {
    List<OrderItem> findByIdOrderId(Long orderId); // @EmbeddedId
    // or
    List<OrderItem> findByOrderId(Long orderId);   // @IdClass
}
```

### 🆚 vs.
| Aspect | @EmbeddedId | @IdClass |
|--------|-------------|----------|
| Key location | Embedded object in entity | Separate class, @Id on entity fields |
| JPQL access | `o.id.orderId` | `o.orderId` (simpler) |
| OOP style | ✅ Encapsulated | Less OO |
| HQL clarity | Need prefix | Direct field access |
| Best for | Clean domain model | Simple JPQL queries |

### ⚡ Remember
- Key class MUST: implement `Serializable`, override `equals()` + `hashCode()`
- `@EmbeddedId` is generally preferred (better encapsulation)
- Both work with Spring Data JPA repositories: `JpaRepository<Entity, KeyClass>`

---

<a id="q24"></a>
## Q24. How do derived (query method) queries work in Spring Data JPA?

### 📝 One-Liner
Spring Data JPA auto-generates SQL from method names in repository interfaces — `findByNameAndAge` becomes `SELECT ... WHERE name = ? AND age = ?`.

### 🔑 Quick Answer
Define method in repository interface following naming convention: `findBy` + property + operator (And, Or, Between, Like, OrderBy, etc.). Spring parses the method name and generates the query at startup. No implementation needed. *(Method ka naam likho, Spring automatically SQL generate kar dega — implementation likhne ki zaroorat nahi)*

### 💻 Code
```java
public interface EmployeeRepository extends JpaRepository<Employee, Long> {

    // Simple queries
    List<Employee> findByName(String name);
    List<Employee> findByDepartment(String dept);
    Optional<Employee> findByEmail(String email);

    // Compound conditions
    List<Employee> findByDepartmentAndSalaryGreaterThan(String dept, double salary);
    List<Employee> findByNameContainingIgnoreCase(String keyword);
    List<Employee> findByAgeBetween(int min, int max);

    // Sorting and limiting
    List<Employee> findTop5ByDepartmentOrderBySalaryDesc(String dept);
    List<Employee> findByDepartmentOrderByNameAsc(String dept);

    // Boolean, null checks
    List<Employee> findByActiveTrue();
    List<Employee> findByManagerIsNull();

    // Count and existence
    long countByDepartment(String dept);
    boolean existsByEmail(String email);

    // Delete
    void deleteByDepartment(String dept);
}
```

### ⚠️ Pitfalls
- Very long method names become unreadable → use `@Query` instead
- Nested property: `findByAddress_City(String city)` — use underscore for nested
- Typo in method name = startup failure (Spring validates at bootstrap)
- `findBy` returns empty list (not null) if no results

### ⚡ Remember
- Keywords: `And`, `Or`, `Between`, `LessThan`, `GreaterThan`, `Like`, `In`, `OrderBy`, `Top`, `First`
- For complex queries, use `@Query("SELECT e FROM Employee e WHERE ...")`
- Pagination: add `Pageable` parameter, return `Page<Employee>`
- Spring validates method names at startup — typos caught early

---

<a id="q25"></a>
## Q25. How to fetch only selected columns using JPQL instead of entire entity?

### 📝 One-Liner
Use JPQL constructor expressions (`new DTO(...)`) or Spring Data JPA interface/class projections to select specific columns — avoids fetching full entity.

### 🔑 Quick Answer
Three approaches: **DTO Projection** (JPQL `new` keyword), **Interface Projection** (Spring generates proxy), **Tuple/Object[]** (raw results). Projections improve performance by fetching only needed columns instead of all entity fields. *(Poora entity fetch mat karo — sirf required columns nikalo projection se)*

### 💻 Code
```java
// 1. DTO Projection — JPQL constructor expression
public record EmployeeDTO(String name, String department, double salary) {}

@Query("SELECT new com.example.dto.EmployeeDTO(e.name, e.department, e.salary) " +
       "FROM Employee e WHERE e.department = :dept")
List<EmployeeDTO> findEmployeeSummary(@Param("dept") String dept);

// 2. Interface Projection (Spring auto-generates proxy)
public interface EmployeeNameSalary {
    String getName();
    double getSalary();
}

List<EmployeeNameSalary> findByDepartment(String dept); // auto-projection!

// 3. Native query with projection
@Query(value = "SELECT name, salary FROM employees WHERE dept = :dept", nativeQuery = true)
List<Object[]> findNameSalaryByDept(@Param("dept") String dept);

// 4. Dynamic projection
<T> List<T> findByDepartment(String dept, Class<T> type);
// usage: repo.findByDepartment("IT", EmployeeNameSalary.class);
```

### 🆚 vs.
| Approach | Type-Safe | Performance | Flexibility |
|----------|-----------|-------------|-------------|
| DTO Projection (new) | ✅ | ✅ Best | Need full package path |
| Interface Projection | ✅ | ✅ Good | Auto-proxy generation |
| Object[] | ❌ | ✅ | Manual casting needed |
| Entity fetch | ✅ | ❌ Fetches all columns | Easiest |

### ⚡ Remember
- DTO Projection requires **fully qualified class name** in JPQL `new` expression
- Interface Projection: Spring generates proxy at runtime — just define getters
- Closed projection (exact columns) > Open projection (`@Value` SpEL)
- Always use projections for read-only list/report queries

---

<a id="q26"></a>
## Q26. How to use Enums in a JPA entity class?

### 📝 One-Liner
Use `@Enumerated(EnumType.STRING)` to store enum name as text, or `@Enumerated(EnumType.ORDINAL)` to store ordinal position — STRING is strongly recommended.

### 🔑 Quick Answer
`@Enumerated(EnumType.STRING)` stores the enum constant name ("ACTIVE", "INACTIVE"). `EnumType.ORDINAL` stores the position (0, 1, 2) — dangerous because reordering enums silently corrupts data. For custom mapping, use `@Converter` (AttributeConverter). *(Hamesha STRING use karo — ORDINAL mein enum reorder karne pe data corrupt ho jata hai)*

### 💻 Code
```java
public enum Status {
    ACTIVE, INACTIVE, SUSPENDED
}

@Entity
public class Employee {
    @Id @GeneratedValue
    private Long id;
    private String name;

    @Enumerated(EnumType.STRING)   // stores "ACTIVE", "INACTIVE", etc.
    private Status status;

    @Enumerated(EnumType.ORDINAL)  // stores 0, 1, 2 — NOT recommended
    private Status legacyStatus;
}

// Custom mapping with AttributeConverter (most flexible)
public enum Priority {
    LOW("L"), MEDIUM("M"), HIGH("H"), CRITICAL("C");

    private final String code;
    Priority(String code) { this.code = code; }
    public String getCode() { return code; }

    public static Priority fromCode(String code) {
        return Arrays.stream(values())
            .filter(p -> p.code.equals(code))
            .findFirst()
            .orElseThrow();
    }
}

@Converter(autoApply = true)
public class PriorityConverter implements AttributeConverter<Priority, String> {
    @Override
    public String convertToDatabaseColumn(Priority priority) {
        return priority == null ? null : priority.getCode();
    }
    @Override
    public Priority convertToEntityAttribute(String code) {
        return code == null ? null : Priority.fromCode(code);
    }
}

@Entity
public class Task {
    @Id @GeneratedValue private Long id;
    // @Convert(converter = PriorityConverter.class) — auto if autoApply=true
    private Priority priority;  // stores "L", "M", "H", "C" in DB
}
```

### ⚠️ Pitfalls
- `ORDINAL` breaks when enums are reordered or new values inserted in middle
- `STRING` uses more storage but is safe and readable
- Null enum field → null column in DB (no special handling needed)
- Adding new enum value with `STRING` is safe; with `ORDINAL` is dangerous

### ⚡ Remember
- **Always use `EnumType.STRING`** — ordinal is a ticking time bomb
- `AttributeConverter` for custom DB representations (codes, single chars)
- PostgreSQL: can use native enum type with `@Type` (Hibernate-specific)
- Query: `WHERE e.status = com.example.Status.ACTIVE` in JPQL

---

<a id="q27"></a>
## Q27. How to design and implement a proper REST Controller class in Spring Boot?

### 📝 One-Liner
Use `@RestController` with proper layered architecture: controller → service → repository, following REST conventions for HTTP methods, status codes, and response structure.

### 🔑 Quick Answer
`@RestController` = `@Controller` + `@ResponseBody`. Use `@RequestMapping` for base path, HTTP method annotations (`@GetMapping`, `@PostMapping`, etc.), proper status codes via `ResponseEntity`, validation with `@Valid`, and exception handling with `@ControllerAdvice`. *(Controller sirf request handle kare, business logic service mein rakho — clean layered architecture)*

### 💻 Code
```java
@RestController
@RequestMapping("/api/v1/employees")
@Validated
public class EmployeeController {

    private final EmployeeService employeeService;

    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }

    @GetMapping
    public ResponseEntity<List<EmployeeDTO>> getAll() {
        return ResponseEntity.ok(employeeService.findAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<EmployeeDTO> getById(@PathVariable Long id) {
        return ResponseEntity.ok(employeeService.findById(id));
    }

    @PostMapping
    public ResponseEntity<EmployeeDTO> create(@Valid @RequestBody CreateEmployeeRequest request) {
        EmployeeDTO created = employeeService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.id()).toUri();
        return ResponseEntity.created(location).body(created);
    }

    @PutMapping("/{id}")
    public ResponseEntity<EmployeeDTO> update(
            @PathVariable Long id,
            @Valid @RequestBody UpdateEmployeeRequest request) {
        return ResponseEntity.ok(employeeService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        employeeService.delete(id);
        return ResponseEntity.noContent().build();
    }
}

// Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(EntityNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(EntityNotFoundException ex) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND)
            .body(new ErrorResponse(404, ex.getMessage()));
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining(", "));
        return ResponseEntity.badRequest().body(new ErrorResponse(400, message));
    }
}
```

### ⚡ Remember
- Follow REST conventions: GET (read), POST (create), PUT (full update), PATCH (partial), DELETE
- Return proper status codes: 200 OK, 201 Created, 204 No Content, 400 Bad Request, 404 Not Found
- Use `ResponseEntity` for full control over status code and headers
- Always use `@Valid` for request body validation
- Controller should be thin — delegate business logic to Service layer

---

<a id="q28"></a>
## Q28. How to design API Gateway and use it with Circuit Breaker in real-time microservices?

### 📝 One-Liner
API Gateway (Spring Cloud Gateway) is the single entry point for all client requests — handles routing, rate limiting, authentication, and integrates with Circuit Breaker for resilience.

### 🔑 Quick Answer
**API Gateway** centralizes cross-cutting concerns: routing, load balancing, rate limiting, auth (JWT validation), request/response transformation. **Spring Cloud Gateway** = default choice for Spring ecosystem. Integrate **Resilience4j Circuit Breaker** as a gateway filter to handle downstream service failures gracefully. *(API Gateway = ek darwaza sabhi requests ke liye — yahan pe auth, rate limit, circuit breaker sab lagao)*

### 📖 How It Works
```
Client Request
     │
     ▼
┌─────────────────────────────────────┐
│         API GATEWAY                  │
│  ┌──────────────────────────────┐   │
│  │ Filters Pipeline:            │   │
│  │ 1. Auth Filter (JWT)         │   │
│  │ 2. Rate Limiter              │   │
│  │ 3. Circuit Breaker           │   │
│  │ 4. Request/Response Transform│   │
│  │ 5. Load Balancer             │   │
│  └──────────────────────────────┘   │
│         Route to:                    │
│  /api/users/** → user-service        │
│  /api/orders/** → order-service      │
│  /api/payments/** → payment-service  │
└─────────────────────────────────────┘
```

### 💻 Code
```yaml
# application.yml — Spring Cloud Gateway
spring:
  cloud:
    gateway:
      routes:
        - id: user-service
          uri: lb://USER-SERVICE
          predicates:
            - Path=/api/users/**
          filters:
            - name: CircuitBreaker
              args:
                name: userServiceCB
                fallbackUri: forward:/fallback/users
            - name: RequestRateLimiter
              args:
                redis-rate-limiter.replenishRate: 10
                redis-rate-limiter.burstCapacity: 20

resilience4j:
  circuitbreaker:
    instances:
      userServiceCB:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        sliding-window-size: 10
```
```java
// Fallback controller
@RestController
public class FallbackController {
    @GetMapping("/fallback/users")
    public ResponseEntity<Map<String, String>> userFallback() {
        return ResponseEntity.status(HttpStatus.SERVICE_UNAVAILABLE)
            .body(Map.of("message", "User Service is temporarily unavailable"));
    }
}
```

### ⚡ Remember
- Spring Cloud Gateway replaces Zuul (Netflix, deprecated)
- Gateway pattern: single entry point = centralized auth, logging, rate limiting
- Circuit Breaker at gateway level catches failures before they cascade
- Real-time combo: Gateway + Eureka (discovery) + Circuit Breaker + Config Server
- Don't put business logic in gateway — only cross-cutting concerns

---

## 📊 Summary

| Round | Focus | Questions |
|-------|-------|-----------|
| Round 1 | Online Assessment | Screening |
| Round 2 | Core Java, Design Patterns, Microservices, Coding | Q1–Q13 |
| Round 3 | Core Java, Spring Boot, JPA, Implementation | Q14–Q28 |

**Key Takeaways:**
- Coding on online compiler without IDE — output was mandatory
- Heavy focus on Java 8 features, Streams, and CompletableFuture
- Design patterns: Singleton variants, Factory, Saga were important
- Spring Boot configuration and JPA/Hibernate practical usage
- Microservices patterns: Feign, Circuit Breaker, API Gateway
