# ☕ Core Java — Coding Standards, Access Modifiers & Packages (Q1–Q26)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Java coding conventions, IS-A/HAS-A relationships, instanceof, null, access modifiers, packages, identifiers, final, abstract classes/methods

---

<a id="q1"></a>
## Q1. Explain Java Coding Standards for classes?

### 📝 One-Liner
Class names should start with an **uppercase letter**, be **nouns**, and use **PascalCase** for multi-word names.

### 🔑 Quick Answer
Sun/Oracle coding conventions: (1) Start with uppercase. (2) Names should be nouns. (3) Multi-word names: capitalize first letter of each word (PascalCase). Examples: `Employee`, `EmployeeDetails`, `ArrayList`, `TreeSet`, `HashSet`. *(Class name = uppercase start, noun, PascalCase)*

### ⚡ Remember
`Class = PascalCase + Noun | Employee, ArrayList, HttpServletRequest`

---

<a id="q2"></a>
## Q2. Explain Java Coding Standards for interfaces?

### 📝 One-Liner
Interface names start with **uppercase** and should typically be **adjectives** describing a capability.

### 🔑 Quick Answer
Interfaces: (1) Start with uppercase. (2) Names should be adjectives (describe capability/behavior). Examples: `Runnable`, `Serializable`, `Cloneable`, `Comparable`, `Iterable`. Some interfaces use nouns when they represent types: `List`, `Map`, `Set`. *(Interface = uppercase, usually adjective — kya kar sakta hai)*

### ⚡ Remember
`Interface = PascalCase + Adjective | Runnable, Serializable, Comparable`

---

<a id="q3"></a>
## Q3. Explain Java Coding Standards for methods?

### 📝 One-Liner
Method names start with **lowercase**, should be **verbs**, and use **camelCase** for multi-word names.

### 🔑 Quick Answer
Methods: (1) Start with lowercase. (2) Usually verbs. (3) Multi-word: capitalize inner words (camelCase). (4) Combine verb + noun for descriptive names. Examples: `toString()`, `getCarName()`, `calculateSalary()`, `isEmpty()`. *(Method = camelCase + verb | getName(), deleteRecord(), isValid())*

### ⚡ Remember
`Method = camelCase + Verb | getName(), calculateTotal(), isValid()`

---

<a id="q4"></a>
## Q4. Explain Java Coding Standards for variables?

### 📝 One-Liner
Variable names start with **lowercase**, should be **nouns**, short & meaningful, using **camelCase**.

### 🔑 Quick Answer
Variables: (1) Start with lowercase. (2) Should be nouns. (3) Short, meaningful names. (4) Multi-word: capitalize inner words. Examples: `name`, `empName`, `empSalary`, `totalCount`. *(Variable = camelCase + noun, chhota aur meaningful)*

### ⚡ Remember
`Variable = camelCase + Noun | empName, totalCount, maxRetry`

---

<a id="q5"></a>
## Q5. Explain Java Coding Standards for constants?

### 📝 One-Liner
Constants use **UPPER_SNAKE_CASE** — all uppercase with underscores separating words, declared with `static final`.

### 🔑 Quick Answer
Constants: (1) Only uppercase letters. (2) Words separated by underscores. (3) Usually nouns. (4) Declared `static final`. Examples: `MAX_VALUE`, `MIN_VALUE`, `MAX_PRIORITY`, `DEFAULT_TIMEOUT`. *(Constant = UPPER_SNAKE_CASE | MAX_VALUE, API_KEY)*

### ⚡ Remember
`Constant = static final + UPPER_SNAKE_CASE | MAX_VALUE, MIN_PRIORITY`

---

<a id="q6"></a>
## Q6. What is 'IS-A' relationship in Java?

### 📝 One-Liner
IS-A represents **inheritance** — implemented using the `extends` keyword. "A Dog IS-A Animal."

### 🔑 Quick Answer
IS-A relationship means one class inherits from another. A `Car extends Vehicle` means "Car IS-A Vehicle." The main advantage is **code reusability** — the subclass inherits all public/protected members of the parent. IS-A is implemented with `extends` (class) or `implements` (interface). *(IS-A = inheritance = extends keyword, baccha parent ka type hai)*

### 💻 Code Example

```java
class Vehicle { void start() { } }
class Car extends Vehicle { }      // Car IS-A Vehicle
class Motorcycle extends Vehicle { } // Motorcycle IS-A Vehicle

Vehicle v = new Car();  // ✅ works because Car IS-A Vehicle
```

### ⚡ Remember
`IS-A = extends/implements = inheritance = code reuse`

---

<a id="q7"></a>
## Q7. What is 'HAS-A' relationship in Java?

### 📝 One-Liner
HAS-A represents **composition/aggregation** — one class contains a reference to another. "A Car HAS-A Engine."

### 🔑 Quick Answer
HAS-A (composition/aggregation) means one class holds an instance of another class as a field. No special keyword — just use `new` inside the class. Advantage: code reuse without inheritance. Preferred over inheritance in many cases ("favor composition over inheritance"). *(HAS-A = composition/aggregation = ek class ke andar doosri class ka object)*

### 💻 Code Example

```java
class Engine {
    void start() { System.out.println("Engine started"); }
}
class Car {
    private Engine engine = new Engine();  // ⭐ Car HAS-A Engine
    void start() { engine.start(); }
}
// Car HAS-A Engine (not IS-A Engine)
```

### ⚡ Remember
`HAS-A = composition = contains reference | No keyword needed, use new`

---

<a id="q8"></a>
## Q8. Difference between IS-A and HAS-A relationship?

### 📝 One-Liner
IS-A = **inheritance** (extends); HAS-A = **composition** (contains an object) — both enable code reuse.

### 🔑 Quick Answer

| Feature | IS-A (Inheritance) | HAS-A (Composition) |
|---|---|---|
| Keyword | `extends` / `implements` | `new` (field reference) |
| Relationship | "Dog IS-A Animal" | "Car HAS-A Engine" |
| Mechanism | Inheritance | Composition / Aggregation |
| Coupling | Tight (parent-child bound) | Loose (can swap) |
| Flexibility | Fixed at compile time | Can change at runtime |
| Main advantage | Code reuse via hierarchy | Code reuse via delegation |

*(IS-A = inheritance hierarchy | HAS-A = object contain karna | Dono me reuse hota hai)*

### ⚡ Remember
`IS-A = extends (tight coupling) | HAS-A = contains (loose coupling, preferred)`

---

<a id="q9"></a>
## Q9. Explain about instanceof operator in Java?

### 📝 One-Liner
`instanceof` checks if an object is an **instance of a specific class or interface** — returns `true` or `false`.

### 🔑 Quick Answer
Syntax: `reference instanceof Type`. Returns `true` if the reference points to an instance of that type or its subtype. Returns `false` if reference is `null`. Compile error if types are completely incompatible (e.g., `String instanceof Integer`). Used before downcasting to avoid `ClassCastException`. *(instanceof = object kis type ka hai ye check karta hai)*

### 💻 Code Example

```java
Object obj = "Hello";
obj instanceof String;    // true
obj instanceof Object;    // true (String IS-A Object)
obj instanceof Integer;   // false

String s = null;
s instanceof String;      // false (null → always false)

// Use before downcasting
if (animal instanceof Dog) {
    Dog d = (Dog) animal;  // safe cast
}

// Java 16+ pattern matching
if (animal instanceof Dog d) {  // cast + assign in one step
    d.bark();
}
```

### ⚡ Remember
`instanceof = type check | null → false | Use before downcasting`

---

<a id="q10"></a>
## Q10. What does null mean in Java?

### 📝 One-Liner
`null` means a reference variable **doesn't point to any object** — it holds no address.

### 🔑 Quick Answer
`null` is a special literal (not an object, not a type). A reference variable assigned `null` points to nothing. Calling any method on `null` throws `NullPointerException`. `null` can be assigned to any reference type but not to primitives. `null == null` is `true`. `instanceof` returns `false` for null. *(null = koi object nahi hai, reference kahi point nahi kar raha)*

### 💻 Code Example

```java
String s = null;      // s points to nothing
s.length();           // ❌ NullPointerException!
s == null;            // true
s instanceof String;  // false

// int x = null;      // ❌ compile error — primitives can't be null
Integer y = null;     // ✅ wrapper can be null
int z = y;            // ❌ NullPointerException (auto-unboxing null)
```

### ⚡ Remember
`null = no object | Method call on null → NPE | Primitives can't be null`

---

<a id="q11"></a>
## Q11. Can we have multiple classes in a single file?

### 📝 One-Liner
Yes, but **only one class can be public** — and the file name must match the public class name.

### 🔑 Quick Answer
A `.java` file can contain multiple classes, but: (1) Only one can be `public`. (2) The file name must match the public class name. (3) If you try to make two classes public → compile error: "The public type must be defined in its own file." Each class compiles to its own `.class` file. Not recommended — one class per file is the convention. *(Ek file me kai classes ho sakte hain, lekin public sirf ek)*

### ⚡ Remember
`Multiple classes allowed | Only 1 public | Filename = public class name`

---

<a id="q12"></a>
## Q12. What access modifiers are allowed for top-level class?

### 📝 One-Liner
Only `public` and **default** (package-private) — `private` and `protected` are not allowed for top-level classes.

### 🔑 Quick Answer
Top-level class access: (1) `public` — visible everywhere. (2) default (no modifier) — visible only in the same package. If you try `private class` or `protected class` at top level → compile error: "Illegal modifier for the class; only public, abstract, and final are permitted." Inner classes can use all four modifiers. *(Top class = sirf public ya default | private/protected nahi chal sakta)*

### ⚡ Remember
`Top-level class = public or default only | private/protected → compile error`

---

<a id="q13"></a>
## Q13. What are packages in Java?

### 📝 One-Liner
A package is a **namespace mechanism** that groups related classes, interfaces, and enums into a single module.

### 🔑 Quick Answer
Declared with `package <name>;` (first statement in file). Convention: lowercase, reverse domain name (`com.google.search`). Purposes: (1) **Naming conflict resolution** — same class name in different packages. (2) **Visibility control** — default access restricts to same package. (3) **Organization** — logical grouping. *(Package = classes ka folder/group + namespace)*

### 💻 Code Example

```java
// File: com/company/Employee.java
package com.company;  // must be first statement

public class Employee { }

// File: com/other/Employee.java
package com.other;    // different package, same class name → no conflict
public class Employee { }
```

### ⚡ Remember
`Package = namespace + organization + access control | lowercase, reverse domain`

---

<a id="q14"></a>
## Q14. Can we have more than one package statement in a source file?

### 📝 One-Liner
No — **at most one** package statement per source file, and it must be the first statement.

### 🔑 Quick Answer
Only one `package` statement is allowed. Multiple package statements → compile error. If no package is declared, the class belongs to the **default package** (not recommended for production code). *(Ek file me sirf ek package statement, woh bhi pehle line pe)*

### ⚡ Remember
`Max 1 package statement | Must be first line | No package = default package`

---

<a id="q15"></a>
## Q15. Can we define package statement after import statement?

### 📝 One-Liner
No — `package` must be the **first statement** (before imports). Only comments can precede it.

### 🔑 Quick Answer
Order: `package` → `import` → class definition. Package after import → compile error. Comments can appear before the package statement. *(Order yaad rakho: package → import → class)*

### ⚡ Remember
`package → import → class | Package must be first (comments allowed before)`

---

<a id="q16"></a>
## Q16. What are identifiers in Java?

### 📝 One-Liner
Identifiers are **names** given to classes, methods, variables, and other code elements in Java.

### 🔑 Quick Answer
Rules: (1) Must start with letter, underscore `_`, or dollar `$`. (2) Cannot start with a digit. (3) No limit on length (but keep it under 15 chars). (4) Case-sensitive. (5) Cannot use reserved words (`class`, `int`, `public`). From second character onwards, digits are allowed. *(Identifier = naam jo hum dete hain — class, method, variable ko)*

### 💻 Code Example

```java
// ✅ Valid identifiers
int age, _count, $price, myVar2;

// ❌ Invalid identifiers
// int 2name;      → starts with digit
// int class;      → reserved word
// int my-name;    → hyphen not allowed
```

### ⚡ Remember
`Start with letter/_ /$ | No digits at start | No keywords | Case-sensitive`

---

<a id="q17"></a>
## Q17. What are access modifiers in Java?

### 📝 One-Liner
Access modifiers control the **visibility** of classes, methods, and variables — `public`, `private`, `protected`, and default.

### 🔑 Quick Answer

| Modifier | Same Class | Same Package | Subclass (diff pkg) | Other Packages |
|---|---|---|---|---|
| `public` | ✅ | ✅ | ✅ | ✅ |
| `protected` | ✅ | ✅ | ✅ | ❌ |
| default | ✅ | ✅ | ❌ | ❌ |
| `private` | ✅ | ❌ | ❌ | ❌ |

*(Access modifier = kaun kahan se access kar sakta hai — visibility control)*

### ⚡ Remember
`public > protected > default > private | Encapsulation ka tool`

---

<a id="q18"></a>
## Q18. Difference between access specifiers and access modifiers in Java?

### 📝 One-Liner
In Java, there's **no distinction** — the term "access modifiers" covers all; "specifiers" is a C++ convention.

### 🔑 Quick Answer
C++ separates: access specifiers (`public`, `private`, `protected`) from access modifiers (`static`, `final`). Java doesn't make this distinction. Java terminology: **Access modifiers** = `public`, `private`, `protected`, default. **Non-access modifiers** = `abstract`, `final`, `static`, `strictfp`, `synchronized`, `volatile`, `transient`. *(Java me specifier nahi, sab modifier hain — access vs non-access)*

### ⚡ Remember
`Java = Access modifiers (public/private/protected/default) + Non-access modifiers (static/final/abstract)`

---

<a id="q19"></a>
## Q19. What access modifiers can be used for a class?

### 📝 One-Liner
Top-level classes: only `public` or default. Inner classes: all four (`public`, `protected`, default, `private`).

### 🔑 Quick Answer
**public class**: visible from same class, same package, different package subclass, different package non-subclass — everywhere. **default class** (no modifier): visible from same class, same package only — not accessible from different packages. *(Top-level class = public ya default | Inner class = saare 4 modifiers allowed)*

### ⚡ Remember
`Top-level = public/default only | Inner class = all 4 modifiers`

---

<a id="q20"></a>
## Q20. Explain access modifiers for methods?

### 📝 One-Liner
Methods can use all four: `public` (everywhere), `protected` (package + subclass), default (package only), `private` (same class only).

### 🔑 Quick Answer
**(1) public** — accessible everywhere. **(2) default** — same class + same package only. **(3) protected** — same package + subclass in different package. **(4) private** — only within declaring class. Choose the **most restrictive** modifier that works — principle of least privilege. *(Method = chaaro modifier allowed | Sabse restricted use karo jo kaam kare)*

### ⚡ Remember
`Methods = all 4 modifiers | Prefer most restrictive | private → default → protected → public`

---

<a id="q21"></a>
## Q21. Explain access modifiers for variables?

### 📝 One-Liner
Variables support all four access modifiers — same visibility rules as methods.

### 🔑 Quick Answer
Same rules as methods: `public` (everywhere), `protected` (package + subclass), default (package), `private` (same class). Best practice: make instance variables `private` and expose via getters/setters (encapsulation). *(Variables = chaaro modifier | Best practice: private + getter/setter)*

### ⚡ Remember
`Variables = all 4 modifiers | Always prefer private for encapsulation`

---

<a id="q22"></a>
## Q22. What is final keyword in Java?

### 📝 One-Liner
`final` prevents modification — final class can't be **extended**, final method can't be **overridden**, final variable can't be **reassigned**.

### 🔑 Quick Answer

| Applied To | Effect |
|---|---|
| `final class` | Cannot be subclassed (e.g., `String`) |
| `final method` | Cannot be overridden in subclass |
| `final variable` | Cannot be reassigned after initialization |

Advantage: security, immutability. Disadvantage: limits OOP (no inheritance, no polymorphism). *(final = ek baar set, phir change nahi — class, method, variable teeno pe lag sakta hai)*

### 💻 Code Example

```java
final class Immutable { }          // ❌ cannot extend
// class Child extends Immutable { }  → compile error

class Parent {
    final void calculate() { }     // ❌ cannot override
}

final int MAX = 100;
// MAX = 200;                       // ❌ compile error
```

### 🎯 Tricky Follow-ups
- **Q: Can a final variable be initialized later?** → Yes, "blank final" — must be initialized in constructor.
- **Q: Is final object truly immutable?** → No! The reference can't change, but the object's state can: `final List<String> list = new ArrayList<>(); list.add("hi");` works.

### ⚡ Remember
`final class = no extends | final method = no override | final variable = no reassign`

---

<a id="q23"></a>
## Q23. Explain about abstract classes in Java?

### 📝 One-Liner
An abstract class is an **incomplete class** (declared with `abstract`) that cannot be instantiated — it may contain abstract and concrete methods.

### 🔑 Quick Answer
Abstract class: (1) Declared with `abstract` keyword. (2) Cannot be instantiated directly. (3) Can have abstract methods (no body) and concrete methods (with body). (4) Can have 0 or more abstract methods. (5) Subclass must override all abstract methods or itself be abstract. (6) Can have constructors, static methods, instance fields. If a class has even one abstract method, it **must** be declared abstract. *(Abstract class = adhura blueprint, directly object nahi ban sakta)*

### 💻 Code Example

```java
abstract class Shape {
    String color;

    Shape(String color) { this.color = color; }  // constructor ✅

    abstract double area();      // no body — subclass must implement
    
    void display() {             // concrete method ✅
        System.out.println("Color: " + color + ", Area: " + area());
    }
}

class Circle extends Shape {
    double radius;
    Circle(double r) { super("Red"); this.radius = r; }
    
    @Override
    double area() { return Math.PI * radius * radius; }
}

// Shape s = new Shape("Blue");  → ❌ compile error: cannot instantiate
Shape s = new Circle(5.0);       // ✅ via subclass reference
```

### ⚡ Remember
`abstract class = can't instantiate | 0+ abstract methods | Can have constructors + state`

---

<a id="q24"></a>
## Q24. Can we create a constructor in an abstract class?

### 📝 One-Liner
Yes — abstract class constructors are called via `super()` from subclass constructors to initialize shared state.

### 🔑 Quick Answer
Abstract classes can have constructors even though they can't be instantiated directly. The constructor runs when a concrete subclass is created — the subclass constructor calls `super()`. Purpose: initialize common fields defined in the abstract class. *(Haan, abstract class me constructor ho sakta hai — subclass super() se call karta hai)*

### ⚡ Remember
`Abstract class constructor = ✅ | Called via super() from subclass | Initializes shared state`

---

<a id="q25"></a>
## Q25. What are abstract methods in Java?

### 📝 One-Liner
Abstract methods have **no body** (just declaration + semicolon) — the subclass is responsible for providing the implementation.

### 🔑 Quick Answer
Declared with `abstract` keyword and a semicolon instead of body: `public abstract void doWork();`. Rules: (1) No method body. (2) Class containing abstract method must be abstract. (3) Subclass must override all inherited abstract methods or be abstract itself. (4) Cannot be `private`, `static`, or `final`. *(Abstract method = sirf declaration, body nahi — subclass likhega implementation)*

### 💻 Code Example

```java
abstract class Payment {
    abstract void process(double amount);  // ⭐ no body, just signature
}

class CreditCardPayment extends Payment {
    @Override
    void process(double amount) {  // ⭐ must provide body
        System.out.println("Charged $" + amount + " to credit card");
    }
}
```

### ⚡ Remember
`abstract method = no body + semicolon | Subclass must implement | Can't be private/static/final`

---

<a id="q26"></a>
## Q26. Explain about instanceof operator in Java?

### 📝 One-Liner
`instanceof` tests whether an object is of a specific type — returns `true`/`false`, compile-time checks incompatible types.

### 🔑 Quick Answer
Syntax: `ref instanceof Type`. Returns `true` if ref is subtype of Type. `null instanceof Any` → always `false`. Compiler checks compatibility — completely unrelated types cause compile error ("incompatible types"). From Java 16, pattern matching: `if (obj instanceof String s)` combines check + cast. *(instanceof = type check operator, downcasting se pehle use karo)*

### 🎯 Tricky Follow-ups
- **Q: Does instanceof work with interfaces?** → Yes: `obj instanceof Serializable`
- **Q: What about generics?** → Erased at runtime, so `list instanceof List<String>` won't compile — use `list instanceof List<?>`.

### ⚡ Remember
`instanceof = type check | null→false | Compile error for incompatible types | Pattern matching Java 16+`
