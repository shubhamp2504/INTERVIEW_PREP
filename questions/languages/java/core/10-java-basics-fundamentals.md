# ☕ Core Java — Java Basics & Fundamentals (Q1–Q25)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Static blocks, constructors, overloading/overriding, super/this, platform independence, JIT, bytecode, main method, encoding, operators

---

<a id="q1"></a>
## Q1. What are static blocks and static initializers in Java?

### 📝 One-Liner
Static blocks are code blocks declared with the `static` keyword that execute **exactly once** when the class is first loaded — before any constructor runs.

### 🔑 Quick Answer
Static blocks initialize static fields in a class. They run when `ClassLoader` loads the class, even before constructors. Multiple static blocks execute in the order they appear. Use them for complex static field initialization (e.g., populating a static Map). *(Static block = class load hote hi ek baar chalega, constructor se pehle)*

### 📖 How It Works

```
Class Loading Timeline:
┌──────────────────────────────────────┐
│  1. ClassLoader loads .class file    │
│  2. Static fields get default values │
│  3. Static blocks execute (in order) │
│  4. Class is ready to use            │
│  5. Constructor runs on new()        │
└──────────────────────────────────────┘
```

Static blocks execute **once per classloader**, not per instance. If you have three `new MyClass()` calls, the static block runs only on the first load. Multiple static blocks run top-to-bottom. Static blocks can only access static members — no `this` reference available.

### 🗣️ Answering Approach
"Static blocks execute exactly once when the class is loaded by the ClassLoader — even before any constructor runs. I use them for complex static field initialization, like populating a lookup Map or loading native libraries. They run in order of declaration and can only access static members since there's no instance context yet."

### 💻 Code Example

```java
public class DatabaseConfig {
    private static final Map<String, String> CONFIG = new HashMap<>();

    // Static block - runs once when class loads
    static {
        CONFIG.put("url", "jdbc:mysql://localhost:3306/db");
        CONFIG.put("driver", "com.mysql.cj.jdbc.Driver");
        System.out.println("Static block executed - config loaded");
    }

    public DatabaseConfig() {
        System.out.println("Constructor executed");
    }
    // Output on first new DatabaseConfig():
    // Static block executed - config loaded
    // Constructor executed
    // Second new DatabaseConfig() → only "Constructor executed"
}
```

### ⚠️ Pitfalls
- Exception in static block → `ExceptionInInitializerError` (class becomes unusable)
- Cannot use `this` or instance variables inside static blocks
- Static blocks make unit testing harder (run before test setup)

### 🆚 vs.
| Static Block | Instance Initializer Block |
|---|---|
| Runs once per class load | Runs on every `new` before constructor |
| `static { ... }` | `{ ... }` (no static keyword) |
| Only accesses static members | Can access instance + static members |

### 🎯 Tricky Follow-ups
- **Q: If a static block throws an exception, can the class be used later?** → No, `NoClassDefFoundError` on subsequent attempts.
- **Q: Can static blocks access instance variables?** → No, compilation error — no instance exists yet.

### ⚡ Remember
`Static block = class load → ek baar → constructor se pehle`

### 🔗 Follow-ups
→ What is the execution order of: static block, instance block, constructor, and parent class?

---

<a id="q2"></a>
## Q2. How to call one constructor from another constructor in Java?

### 📝 One-Liner
Use `this()` to call another constructor in the **same class** — it must be the first statement in the constructor.

### 🔑 Quick Answer
Within the same class, use `this(args)` to invoke another constructor — called **constructor chaining**. `this()` must be the first statement. You cannot use both `this()` and `super()` in the same constructor. Use `super(args)` to call parent class constructor. *(Constructor chaining = ek constructor se doosra bulaana)*

### 📖 How It Works

```
Constructor Chaining:
┌──────────────────────────────┐
│  new Employee("John")        │
│    ↓                         │
│  Employee(String name) {     │
│    this(name, 0);  ← chains  │
│  }                           │
│    ↓                         │
│  Employee(String name, int age) {
│    super();  ← parent        │
│    this.name = name;         │
│    this.age = age;           │
│  }                           │
└──────────────────────────────┘
```

**Rules**: (1) `this()` must be first statement. (2) Cannot have both `this()` and `super()` in same constructor. (3) No recursive constructor calls (compile error). (4) If neither `this()` nor `super()` is present, compiler inserts `super()` implicitly.

### 🗣️ Answering Approach
"I use this() for constructor chaining within the same class — it must be the first statement. This avoids code duplication when multiple constructors share common initialization logic. The most specific constructor does the actual work, and simpler constructors delegate to it with default values."

### 💻 Code Example

```java
public class Employee {
    private String name;
    private int age;
    private String dept;

    // No-arg constructor chains to 2-arg
    public Employee() {
        this("Unknown", 0);  // ⭐ must be first statement
    }

    // 1-arg chains to 3-arg
    public Employee(String name) {
        this(name, 0, "General");
    }

    // 2-arg chains to 3-arg
    public Employee(String name, int age) {
        this(name, age, "General");
    }

    // Primary constructor - does the real work
    public Employee(String name, int age, String dept) {
        // super() called implicitly here
        this.name = name;
        this.age = age;
        this.dept = dept;
    }
}
```

### ⚠️ Pitfalls
- `this()` and `super()` **cannot coexist** — both require first statement position
- Circular constructor chaining → compile error: `recursive constructor invocation`

### 🆚 vs.
| `this()` | `super()` |
|---|---|
| Calls constructor in same class | Calls constructor in parent class |
| Used for constructor chaining | Used for inheritance chain |
| Cannot be used with `super()` | Cannot be used with `this()` |

### ⚡ Remember
`this() = same class ka constructor, super() = parent ka | Pehla statement hona chahiye`

---

<a id="q3"></a>
## Q3. What is method overriding in Java?

### 📝 One-Liner
Method overriding is when a subclass provides its **own implementation** of a method already defined in its superclass — same name, same parameters, same (or covariant) return type.

### 🔑 Quick Answer
A subclass redefines a superclass method with the **same signature** (name + parameter types). At runtime, JVM calls the subclass version when the object is of subclass type — this is **runtime polymorphism**. Rules: (1) Same method name and parameters. (2) Return type must be same or covariant (subtype). (3) Access modifier cannot be more restrictive. (4) Cannot override `static`, `final`, or `private` methods. *(Override = bacche ne parent ka tarika apne hisaab se badal diya)*

### 📖 How It Works

```
Method Resolution at Runtime:
┌────────────────────────────┐
│  Animal a = new Dog();     │
│  a.speak();                │
│                            │
│  Compile time: checks      │
│    Animal has speak()? ✓   │
│  Runtime: actual object    │
│    is Dog → Dog.speak()    │
└────────────────────────────┘
```

The JVM uses the **actual object type** (not reference type) to determine which method to call. This is dynamic dispatch / late binding. `@Override` annotation is optional but recommended — it catches typos at compile time.

### 🗣️ Answering Approach
"Method overriding lets a subclass provide a specific implementation of a method defined in its parent. The key is same name, same parameters, and same or covariant return type. At runtime, JVM uses the actual object type to decide which version to call — that's dynamic polymorphism. I always use @Override annotation to catch mistakes at compile time."

### 💻 Code Example

```java
class Animal {
    public String speak() {
        return "Some sound";
    }
}

class Dog extends Animal {
    @Override
    public String speak() {  // ⭐ same signature, overridden
        return "Bark!";
    }
}

// Runtime polymorphism
Animal a = new Dog();
a.speak();  // → "Bark!" (Dog's version called)
```

### ⚠️ Pitfalls
- `private`, `static`, `final` methods **cannot** be overridden
- Widening access is OK (`protected` → `public`), narrowing is NOT (`public` → `private`)
- Overriding with wrong parameter types → that's overloading, not overriding!

### 🆚 vs.
| Overriding | Overloading |
|---|---|
| Between parent-child classes | Within same class (or inherited) |
| Same name, same params | Same name, different params |
| Runtime polymorphism | Compile-time polymorphism |
| `@Override` annotation | No special annotation |

### ⚡ Remember
`Override = same signature + subclass → runtime polymorphism (dynamic dispatch)`

---

<a id="q4"></a>
## Q4. What is the super keyword in Java?

### 📝 One-Liner
`super` refers to the **parent class** — used to access parent's variables, methods, and constructors that are hidden or overridden.

### 🔑 Quick Answer
`super` has two forms: (1) `super(args)` calls parent constructor — must be first statement. (2) `super.method()` or `super.variable` accesses parent's hidden member. When a subclass overrides a method, `super.method()` explicitly calls the parent version. *(super = parent class ko refer karta hai)*

### 📖 How It Works

```
super usage:
┌───────────────────────────────────────┐
│  1. super(args)  → parent constructor │
│  2. super.method() → parent method    │
│  3. super.field → parent field        │
└───────────────────────────────────────┘
```

If a constructor doesn't explicitly call `super()` or `this()`, the compiler inserts `super()` (no-arg) automatically. If the parent has no no-arg constructor, you **must** explicitly call `super(args)`.

### 🗣️ Answering Approach
"super refers to the immediate parent class. I use it in two ways: super() in constructors to invoke the parent constructor — which must be the first statement — and super.method() to call the parent's version of an overridden method. It's essential when the subclass needs to extend parent behavior rather than replace it entirely."

### 💻 Code Example

```java
class Vehicle {
    String type = "Vehicle";
    Vehicle(String type) { this.type = type; }
    void start() { System.out.println("Vehicle starting"); }
}

class Car extends Vehicle {
    Car() {
        super("Car");  // ⭐ calls Vehicle(String)
    }

    @Override
    void start() {
        super.start();  // ⭐ calls Vehicle.start() first
        System.out.println("Car engine started");
    }
}
```

### ⚡ Remember
`super() = parent constructor (first line) | super.method() = parent ka overridden method`

---

<a id="q5"></a>
## Q5. Difference between method overloading and method overriding?

### 📝 One-Liner
Overloading = **same name, different params** (compile-time); Overriding = **same name, same params** in subclass (runtime).

### 🔑 Quick Answer

| Aspect | Overloading | Overriding |
|---|---|---|
| Where | Same class | Subclass–superclass |
| Parameters | Must differ | Must be same |
| Return type | Can differ | Must be same/covariant |
| Polymorphism | Compile-time (static) | Runtime (dynamic) |
| Inheritance | Not required | Required |
| `static`/`final` | Can be overloaded | Cannot be overridden |

*(Overloading = same naam, alag params | Overriding = same naam, same params, alag class)*

### 💻 Code Example

```java
// OVERLOADING - same class, different params
class Calculator {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }      // overloaded
    int add(int a, int b, int c) { return a + b + c; }    // overloaded
}

// OVERRIDING - subclass, same signature
class Animal {
    void sound() { System.out.println("generic"); }
}
class Cat extends Animal {
    @Override
    void sound() { System.out.println("meow"); }  // overridden
}
```

### 🎯 Tricky Follow-ups
- **Q: Is return type part of the method signature?** → No, return type alone cannot distinguish overloaded methods.
- **Q: Can we override a method and change only the return type?** → Yes, if covariant (subtype) — from Java 5.

### ⚡ Remember
`Overloading = WHAT you pass (params) | Overriding = WHO you are (object type)`

---

<a id="q6"></a>
## Q6. Difference between abstract class and interface?

### 📝 One-Liner
Abstract class = **partial implementation** with state; Interface = **pure contract** (before Java 8) / trait-like (Java 8+).

### 🔑 Quick Answer

| Feature | Abstract Class | Interface |
|---|---|---|
| Methods | Abstract + concrete | Abstract (+ default/static from Java 8) |
| Variables | Any type | Only `public static final` |
| Constructor | Yes | No |
| Inheritance | Single (`extends`) | Multiple (`implements`) |
| Access modifiers | Any | Methods: `public` (default) |
| Use case | IS-A with shared state | CAN-DO capability |

*(Abstract class = adhura blueprint with state | Interface = sirf contract, koi state nahi)*

### 📖 How It Works

```
When to use which:
┌─────────────────────────────────────┐
│ Abstract Class:                     │
│  - Shared state (fields)            │
│  - Common method implementations    │
│  - IS-A relationship                │
│  - Template method pattern          │
│                                     │
│ Interface:                          │
│  - Multiple inheritance needed      │
│  - Unrelated classes share behavior │
│  - CAN-DO / capability contract     │
│  - API contracts                    │
└─────────────────────────────────────┘
```

From Java 8, interfaces can have `default` and `static` methods. From Java 9, `private` methods in interfaces. This narrowed the gap but abstract classes still hold state (instance fields) and constructors.

### 🗣️ Answering Approach
"The fundamental difference is that abstract classes can hold state — instance fields, constructors, any access modifier — while interfaces define contracts. Abstract classes support single inheritance; interfaces support multiple. Since Java 8, interfaces can have default and static methods, but they still can't have instance state. I use abstract classes when I need shared state or template method pattern, and interfaces when I want to define a capability that unrelated classes can implement."

### 💻 Code Example

```java
// Abstract class - has state + partial implementation
abstract class Animal {
    protected String name;          // ⭐ instance state
    Animal(String name) { this.name = name; }  // ⭐ constructor
    abstract void sound();          // must override
    void breathe() { System.out.println("breathing"); }  // concrete
}

// Interface - pure contract + default methods
interface Flyable {
    void fly();                     // abstract
    default void glide() {          // default (Java 8+)
        System.out.println("gliding");
    }
}

class Eagle extends Animal implements Flyable {
    Eagle() { super("Eagle"); }
    void sound() { System.out.println("screech"); }
    public void fly() { System.out.println("soaring"); }
}
```

### ⚡ Remember
`Abstract class = state + partial impl + single inheritance | Interface = contract + multiple inheritance`

---

<a id="q7"></a>
## Q7. Why is Java platform independent?

### 📝 One-Liner
Java source code compiles to **bytecode** (.class), which runs on **any JVM** — "write once, run anywhere."

### 🔑 Quick Answer
Java compiler (`javac`) compiles `.java` files into platform-neutral **bytecode** (`.class` files), not native machine code. The JVM (Java Virtual Machine) on each platform interprets/JIT-compiles this bytecode into native instructions. Since JVMs exist for Windows, Linux, macOS, etc., the same `.class` file runs everywhere. *(Bytecode = universal language jo har JVM samajhta hai)*

### 📖 How It Works

```
Java Platform Independence:
┌──────────┐    javac    ┌──────────┐    JVM     ┌──────────┐
│ Hello.java│ ──────────→│Hello.class│──────────→│ Machine   │
│ (source)  │            │(bytecode) │           │  Code     │
└──────────┘            └──────────┘           └──────────┘
                        Platform-neutral    Platform-specific
                                            (Windows/Linux/Mac)
```

The bytecode is the key — it's an intermediate format that any JVM can understand. The JVM itself is platform-dependent (different JVM for each OS), but the bytecode is platform-independent.

### 🗣️ Answering Approach
"Java achieves platform independence through bytecode. The compiler produces .class files containing bytecode — a platform-neutral intermediate format — not native machine code. This bytecode runs on any platform that has a JVM. The JVM is platform-dependent, but the bytecode isn't. So I compile once on Windows, and the same .class file runs on Linux or macOS without recompilation."

### ⚡ Remember
`Java code → bytecode (portable) → JVM (platform-specific) = write once, run anywhere`

---

<a id="q8"></a>
## Q8. What is method overloading in Java?

### 📝 One-Liner
A class having **two or more methods with the same name but different parameter lists** — compile-time polymorphism.

### 🔑 Quick Answer
Method overloading means multiple methods in the same class with the same name but different: (1) number of parameters, (2) types of parameters, or (3) order of parameters. The compiler decides which method to call based on the arguments passed — **static/compile-time polymorphism**. Return type alone **does not** distinguish overloaded methods. *(Overloading = ek hi naam, alag-alag inputs)*

### 💻 Code Example

```java
public class MathUtil {
    int add(int a, int b) { return a + b; }
    double add(double a, double b) { return a + b; }      // diff types
    int add(int a, int b, int c) { return a + b + c; }    // diff count

    // ❌ NOT valid overloading (only return type differs):
    // long add(int a, int b) { return a + b; }  → compile error
}
```

### ⚠️ Pitfalls
- **Return type is NOT part of method signature** — cannot overload by return type alone
- Ambiguity with autoboxing: `add(int, long)` vs `add(long, int)` → compile error if both exist

### ⚡ Remember
`Overloading = same naam + diff params = compile-time binding`

---

<a id="q9"></a>
## Q9. What is difference between C++ and Java?

### 📝 One-Liner
Java is **platform-independent, garbage-collected, no pointers, no operator overloading**; C++ gives **low-level control** with pointers, manual memory, and multiple inheritance.

### 🔑 Quick Answer

| Feature | Java | C++ |
|---|---|---|
| Platform | Independent (bytecode + JVM) | Dependent (compiles to native) |
| Memory | Garbage collection (automatic) | Manual (`new`/`delete`) |
| Pointers | No explicit pointers | Full pointer support |
| Multiple inheritance | Via interfaces only | Direct class inheritance |
| Operator overloading | Not supported | Supported |
| Multithreading | Built-in support | Library-dependent |
| Global variables | Not allowed | Allowed |
| Templates/Generics | Generics (type erasure) | Templates (code generation) |

*(Java = safe, portable, managed | C++ = powerful, manual, close to hardware)*

### ⚡ Remember
`Java = safety + portability | C++ = power + control`

---

<a id="q10"></a>
## Q10. What is JIT compiler?

### 📝 One-Liner
JIT (Just-In-Time) compiler is part of the JVM that compiles **frequently-used bytecode** to native machine code **at runtime** for faster execution.

### 🔑 Quick Answer
JIT compiler optimizes Java performance by converting bytecode to native machine code during execution. It doesn't compile the entire program at once — it identifies **hotspots** (frequently executed code paths) and compiles only those. The compiled native code is cached for reuse. This is why Java programs start slow (interpretation phase) but speed up over time (JIT kicks in). *(JIT = runtime pe bytecode ko native code me compile karta hai, baar-baar chalane wale code ke liye)*

### 📖 How It Works

```
JVM Execution Flow:
┌─────────┐   interpret    ┌──────────┐
│Bytecode │ ─────────────→ │ Execution │  (slow, first runs)
└─────────┘                └──────────┘
     │
     │ hotspot detected
     ↓
┌─────────┐   JIT compile  ┌──────────┐
│Bytecode │ ─────────────→ │Native Code│  (fast, cached)
└─────────┘                └──────────┘
```

### ⚡ Remember
`JIT = bytecode → native code at runtime | Only hot code paths | Part of JVM`

---

<a id="q11"></a>
## Q11. What is bytecode in Java?

### 📝 One-Liner
Bytecode is the **intermediate, platform-independent instruction set** stored in `.class` files — executable only by the JVM.

### 🔑 Quick Answer
When `javac` compiles a `.java` file, it generates a `.class` file containing bytecode — a set of machine-independent instructions. Bytecode is not human-readable and not native machine code. Only the JVM can understand it. This is what enables Java's "write once, run anywhere" promise. You can view bytecode using `javap -c ClassName`. *(Bytecode = javac ka output jo sirf JVM samajhta hai)*

### ⚡ Remember
`.java → javac → .class (bytecode) → JVM interprets/JIT compiles → machine code`

---

<a id="q12"></a>
## Q12. Difference between this() and super() in Java?

### 📝 One-Liner
`this()` calls another constructor in the **same class**; `super()` calls a constructor in the **parent class** — both must be the first statement.

### 🔑 Quick Answer

| Feature | `this()` | `super()` |
|---|---|---|
| Purpose | Call same-class constructor | Call parent-class constructor |
| Placement | Must be first statement | Must be first statement |
| Use case | Constructor chaining | Inheritance constructor chain |
| Coexistence | Cannot use with `super()` | Cannot use with `this()` |
| Default | Not added by compiler | Compiler adds `super()` if absent |

*(this() = apni class ka constructor | super() = parent ka constructor | Dono ek saath nahi chal sakte)*

### 💻 Code Example

```java
class Parent {
    Parent(String msg) { System.out.println("Parent: " + msg); }
}
class Child extends Parent {
    Child() {
        this("default");  // ⭐ calls Child(String)
    }
    Child(String msg) {
        super(msg);        // ⭐ calls Parent(String)
    }
}
// new Child() → Child() → Child("default") → Parent("default")
```

### ⚡ Remember
`this() = same class chaining | super() = parent class | Cannot coexist | Always first line`

---

<a id="q13"></a>
## Q13. What is a class in Java?

### 📝 One-Liner
A class is a **blueprint/template** that defines the structure (fields) and behavior (methods) for creating objects.

### 🔑 Quick Answer
A class is the fundamental unit in OOP. It defines: (1) **State** — instance variables/fields, (2) **Behavior** — methods, (3) **Constructors** — for initialization. Classes are loaded by the JVM's ClassLoader. Every Java application needs at least one class with a `main` method. A class file must have the same name as the public class and end with `.java`. *(Class = object ka blueprint, jaise ghar ka naksha)*

### 💻 Code Example

```java
public class Employee {          // blueprint
    private String name;         // state
    private int age;

    public Employee(String name, int age) {  // constructor
        this.name = name;
        this.age = age;
    }

    public void work() {         // behavior
        System.out.println(name + " is working");
    }
}
// Employee e = new Employee("John", 30);  → creates object from blueprint
```

### ⚡ Remember
`Class = blueprint (fields + methods + constructors) | Object = actual instance from blueprint`

---

<a id="q14"></a>
## Q14. What is an object in Java?

### 📝 One-Liner
An object is an **instance of a class** — it holds actual values (state) and can execute methods (behavior), created with `new`.

### 🔑 Quick Answer
Objects are runtime entities created from class blueprints using the `new` keyword. Each object has: (1) **State** — values stored in fields, (2) **Behavior** — methods it can execute, (3) **Identity** — unique memory address (hashCode). Objects live on the **heap** and are accessed through reference variables on the stack. GC reclaims objects with no references. *(Object = class ka real instance, heap memory me rehta hai)*

### 💻 Code Example

```java
Employee emp = new Employee("Alice", 25);
//  ↑ ref     ↑ new allocates on heap
// emp (stack) → Employee object (heap)
// emp is reference, the thing on heap is the object
```

### 🆚 vs.
| Class | Object |
|---|---|
| Blueprint/template | Instance of class |
| Compile-time entity | Runtime entity |
| No memory allocated | Memory on heap |
| Defined once | Multiple can exist |

### ⚡ Remember
`Object = class ka instance | Heap pe rehta hai | Reference se access hota hai`

---

<a id="q15"></a>
## Q15. What is a method in Java?

### 📝 One-Liner
A method is a **named block of executable code** within a class that performs a specific task and can accept parameters and return a value.

### 🔑 Quick Answer
A method defines behavior — it has a name, return type, parameter list, and body. Signature: `accessModifier returnType methodName(parameterList) { body }`. Methods can have multiple parameters (comma-separated). Methods can be `static` (class-level) or instance (object-level). *(Method = kaam karne ka tarika, class ke andar define hota hai)*

### 💻 Code Example

```java
public float calculateTotal(int quantity, float price) {
    return quantity * price;
}
// access: public | return: float | name: calculateTotal | params: int, float
```

### ⚡ Remember
`Method = returnType + name + (params) + { body } | Class ka behavior define karta hai`

---

<a id="q16"></a>
## Q16. What is encapsulation in Java?

### 📝 One-Liner
Encapsulation is **wrapping data (fields) and methods into a single class** and restricting direct access using access modifiers — data hiding.

### 🔑 Quick Answer
Encapsulation bundles data and the methods that operate on it into a class, then protects the data using `private` access modifier. External code accesses data only through `public` getter/setter methods. The four access modifiers (`public`, `private`, `protected`, default) control visibility. **Analogy**: A car exposes steering and pedals but hides the engine internals. *(Encapsulation = data ko private rakhna, getter/setter se access)*

### 💻 Code Example

```java
public class BankAccount {
    private double balance;  // ⭐ hidden from outside

    public double getBalance() { return balance; }          // controlled read
    public void deposit(double amount) {                    // controlled write
        if (amount > 0) this.balance += amount;
    }
    // ❌ No setBalance() → cannot set arbitrary values
}
```

### ⚡ Remember
`Encapsulation = private fields + public getters/setters = data hiding + controlled access`

---

<a id="q17"></a>
## Q17. Why is main() method public, static, and void in Java?

### 📝 One-Liner
`public` = JVM can access it from anywhere; `static` = no object needed to call it; `void` = returns nothing to JVM.

### 🔑 Quick Answer
**(1) `public`** — JVM must call main() from outside the class, requires public access. **(2) `static`** — JVM calls main() without creating an instance of the class — it has no object to call `new` on at startup. **(3) `void`** — main() doesn't return anything meaningful to the JVM; the process exit code is set via `System.exit(int)`. `String[] args` receives command-line arguments. *(main() = entry point | JVM ko bina object banaye call karna hai isliye static, bahar se access isliye public, return kuch nahi isliye void)*

### 💻 Code Example

```java
public class App {
    public static void main(String[] args) {
        // public → accessible by JVM
        // static → no object needed
        // void   → nothing returned
        // args   → command line arguments
        System.out.println("Arguments: " + Arrays.toString(args));
    }
}
// Run: java App hello world → args = ["hello", "world"]
```

### 🎯 Tricky Follow-ups
- **Q: Can we overload main()?** → Yes, but JVM always calls `main(String[])`.
- **Q: Can main() be final?** → Yes, it still works — just can't be overridden (rarely relevant).
- **Q: What if main() is not static?** → `Error: Main method is not static`.

### ⚡ Remember
`public = JVM access | static = no object | void = no return | String[] = CLI args`

---

<a id="q18"></a>
## Q18. Explain about main() method in Java?

### 📝 One-Liner
`main()` is the **entry point** of every Java application — the first method JVM invokes when a program starts.

### 🔑 Quick Answer
Signature: `public static void main(String[] args)`. Every Java application must have at least one main method. JVM starts a new thread (main thread) to execute it. `String args[]` captures command-line arguments. The main thread is always a **non-daemon thread** and is the last thread to finish (waits for all non-daemon children). *(main() = program ka starting point, JVM isse sabse pehle call karta hai)*

### ⚡ Remember
`main(String[] args) = JVM entry point | main thread = non-daemon | args = CLI arguments`

---

<a id="q19"></a>
## Q19. What is a constructor in Java?

### 📝 One-Liner
A constructor is a **special method** (same name as class, no return type) called automatically when an object is created via `new`.

### 🔑 Quick Answer
Constructors initialize objects. Two types: **(1) Default** — no-arg, compiler creates if no constructor defined. **(2) Parameterized** — accepts arguments for custom initialization. Constructor rules: (1) Same name as class. (2) No return type (not even `void`). (3) Called automatically with `new`. (4) Can be overloaded. (5) Cannot be `abstract`, `static`, `final`. *(Constructor = object banate waqt automatic initialize karta hai)*

### 💻 Code Example

```java
public class Car {
    String model;

    // Default constructor
    public Car() {
        this.model = "Unknown";
    }

    // Parameterized constructor
    public Car(String model) {
        this.model = model;
    }
}
// Car c1 = new Car();          → model = "Unknown"
// Car c2 = new Car("Tesla");   → model = "Tesla"
```

### 🎯 Tricky Follow-ups
- **Q: If I define a parameterized constructor, does the compiler still create a default one?** → No! You must explicitly define one if needed.
- **Q: Can constructors be private?** → Yes — used in Singleton pattern.

### ⚡ Remember
`Constructor = class name + no return type + auto-called on new`

---

<a id="q20"></a>
## Q20. What is difference between length and length() method in Java?

### 📝 One-Liner
`length` is a **field** on arrays; `length()` is a **method** on Strings.

### 🔑 Quick Answer

| Feature | `length` | `length()` |
|---|---|---|
| Type | Instance variable (field) | Method |
| Used with | Arrays | Strings |
| Syntax | `array.length` | `str.length()` |
| Returns | Number of elements | Number of characters |

Also note: `size()` is a method on Collection classes (List, Set, Map). *(length = array ka field | length() = String ka method | size() = Collection ka method)*

### 💻 Code Example

```java
int[] arr = {1, 2, 3, 4, 5};
System.out.println(arr.length);      // 5 (field, no parentheses)

String str = "Hello World";
System.out.println(str.length());    // 11 (method, with parentheses)

List<String> list = List.of("a", "b");
System.out.println(list.size());     // 2 (Collection method)
```

### ⚡ Remember
`Array → .length (field) | String → .length() (method) | Collection → .size() (method)`

---

<a id="q21"></a>
## Q21. What is ASCII Code?

### 📝 One-Liner
ASCII (American Standard Code for Information Interchange) is a **7-bit character encoding** supporting 128 characters (0–127), English only.

### 🔑 Quick Answer
ASCII represents characters as numbers: `A`=65, `a`=97, `0`=48, space=32. Extended ASCII uses 8 bits (0–255). Limitation: only supports English characters. Java uses **Unicode** (not ASCII) for Strings and identifiers but uses ASCII for source code input elements. *(ASCII = English characters ka number code, 0-127)*

### ⚡ Remember
`ASCII = 0-127 (7-bit) | English only | Java uses Unicode for broader support`

---

<a id="q22"></a>
## Q22. What is Unicode?

### 📝 One-Liner
Unicode is a **16-bit character set** (0–65,535) supporting characters from **all languages worldwide** — Java's native character encoding.

### 🔑 Quick Answer
Unicode was developed by Unicode Consortium to represent characters from every language. Java uses Unicode for Strings, identifiers, and comments. `char` in Java is 16 bits (2 bytes) to support Unicode. ASCII is a subset of Unicode (first 128 characters are same). This is why you can use non-English characters in Java identifiers and comments. *(Unicode = duniya ki har bhasha ke characters, 16-bit, Java iska native use karta hai)*

### 🆚 vs.
| ASCII | Unicode |
|---|---|
| 7-bit (0–127) | 16-bit (0–65,535) |
| English only | All languages |
| 128 characters | 65,536+ characters |

### ⚡ Remember
`Unicode = 16-bit, all languages | ASCII = 7-bit, English only | Java char = Unicode`

---

<a id="q23"></a>
## Q23. Difference between Character Constant and String Constant?

### 📝 One-Liner
Character constant = **single character** in single quotes (`'A'`); String constant = **sequence of characters** in double quotes (`"Hello"`).

### 🔑 Quick Answer

| Feature | Character Constant | String Constant |
|---|---|---|
| Quotes | Single `'A'` | Double `"Hello"` |
| Type | `char` (primitive) | `String` (object) |
| Size | 2 bytes (one Unicode char) | Variable |
| Memory | Stack | String Pool (heap) |

*(Char = single quotes me ek character | String = double quotes me characters ki sequence)*

### ⚡ Remember
`'A' = char (primitive, 2 bytes) | "A" = String (object, pool)`

---

<a id="q24"></a>
## Q24. What are constants and how to create constants in Java?

### 📝 One-Liner
Constants are **fixed values** that cannot change during execution — created using the `final` keyword.

### 🔑 Quick Answer
Declare constants with `final` keyword (optionally `static final` for class-level constants). Convention: UPPER_SNAKE_CASE. Once assigned, value cannot be changed — reassignment causes compile error. `static final` means one copy shared across all instances. *(Constant = ek baar set, phir kabhi change nahi — final keyword se banaate hain)*

### 💻 Code Example

```java
public class AppConstants {
    public static final int MAX_RETRY = 3;
    public static final String API_URL = "https://api.example.com";
    public static final double PI = 3.14159;

    // ❌ MAX_RETRY = 5;  → compile error: cannot assign a value to final variable
}
```

### ⚡ Remember
`Constants = static final + UPPER_SNAKE_CASE | Value fixed after assignment`

---

<a id="q25"></a>
## Q25. Difference between '>>' and '>>>' operators in Java?

### 📝 One-Liner
`>>` is **signed right shift** (preserves sign bit); `>>>` is **unsigned right shift** (fills with zeros).

### 🔑 Quick Answer

| Operator | Name | Sign Bit | Fill |
|---|---|---|---|
| `>>` | Signed right shift | Preserved | Sign bit (0 or 1) |
| `>>>` | Unsigned right shift | Not preserved | Always 0 |

For positive numbers both give the same result. For negative numbers: `>>` keeps the number negative (fills with 1s), `>>>` makes it positive (fills with 0s). *(>> = sign preserve karta hai | >>> = hamesha 0 se fill karta hai)*

### 💻 Code Example

```java
int a = -8;   // binary: 11111111 11111111 11111111 11111000
a >> 2;       // → -2   (sign preserved, fills with 1)
a >>> 2;      // → 1073741822  (fills with 0, becomes positive)

int b = 16;
b >> 2;       // → 4 (same as 16/4)
b >>> 2;      // → 4 (same for positive numbers)
```

### ⚡ Remember
`>> = signed (preserves negative) | >>> = unsigned (always fills zeros)`
