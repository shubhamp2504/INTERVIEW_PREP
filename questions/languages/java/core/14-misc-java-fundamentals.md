# ☕ Core Java — Misc Java Fundamentals (Q1–Q35)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Reference variables, constructors, static/instance, wrapper classes, type conversion, packages/imports, interfaces, enums, varargs

---

<a id="q1"></a>
## Q1. What are reference variables in Java?

### 📝 One-Liner
A reference variable **holds the memory address** of an object on the heap — it's not the object itself.

### 🔑 Quick Answer
`Employee emp = new Employee();` — `emp` is the reference variable (lives on stack), the `Employee` object lives on heap. A reference can: (1) Point to only one type (or its subtypes). (2) Point to multiple objects over time (unless `final`). (3) Be assigned to interface types if the class implements it. `final` reference → can't be reassigned, but the object's state can still change. *(Reference variable = object ka address hold karta hai, object nahi hai khud)*

### 💻 Code Example

```java
Employee emp = new Employee("Alice");  // emp = reference → heap object
emp = new Employee("Bob");             // ✅ emp now points to new object
final Employee mgr = new Employee("Carol");
// mgr = new Employee("Dave");         // ❌ final reference can't be reassigned
mgr.setName("Dave");                   // ✅ object state CAN change
```

### ⚡ Remember
`Reference = address on stack → object on heap | final ref = can't reassign, CAN mutate object`

---

<a id="q2"></a>
## Q2. Will compiler create default constructor if parameterized exists?

### 📝 One-Liner
No — if you define **any** constructor, the compiler does **not** generate a default no-arg constructor.

### 🔑 Quick Answer
Compiler creates a default no-arg constructor **only when no constructors are defined**. Moment you write even one parameterized constructor, the default disappears. If you need both, explicitly define the no-arg constructor. This matters with inheritance: if parent has only parameterized constructor, child must explicitly call `super(args)`. *(Agar ek bhi constructor likha toh compiler default nahi banayega — khud likhna padega)*

### 💻 Code Example

```java
class Car {
    Car(String model) { }  // parameterized constructor
}
// Car c = new Car();  → ❌ compile error! No default constructor
// Fix: explicitly add  Car() { this("Unknown"); }
```

### ⚡ Remember
`Any constructor defined → no default | Want both → explicitly define no-arg`

---

<a id="q3"></a>
## Q3. Can we have a method name same as class name?

### 📝 One-Liner
Yes, it's legal but **not recommended** — it compiles fine but generates a warning (confusing with constructor).

### 🔑 Quick Answer
A method can have the same name as the class. The difference from a constructor: the method has a **return type**. Constructors have no return type. The compiler distinguishes them. However, this creates confusion and should be avoided. *(Haan, chal toh jayega lekin warning aayega — avoid karo, confusing hai)*

### ⚡ Remember
`Method with class name = allowed but bad practice | Constructor has NO return type`

---

<a id="q4"></a>
## Q4. Can we override constructors in Java?

### 📝 One-Liner
No — constructors are **not inherited**, so they **cannot be overridden**. Only methods can be overridden.

### 🔑 Quick Answer
Overriding requires inheritance — the subclass redefines a parent method. Constructors are not inherited; `Child` doesn't get `Parent()` as its own constructor. Constructors can be **overloaded** (multiple constructors in same class with different params) but never overridden. *(Constructor override nahi ho sakta — inherit hi nahi hota)*

### ⚡ Remember
`Constructors not inherited → cannot be overridden | Can be overloaded`

---

<a id="q5"></a>
## Q5. Can static methods access instance variables?

### 📝 One-Liner
No — static methods belong to the **class**, not an instance, so they cannot access instance (non-static) variables.

### 🔑 Quick Answer
Static methods execute without an object context — there's no `this` reference. They can only access `static` members. To access instance variables from a static context, you need an explicit object reference. Trying without one → compile error: "Cannot make a static reference to the non-static field." *(Static method = class ka hai, instance variable ke liye object chahiye)*

### 💻 Code Example

```java
class Demo {
    int instanceVar = 10;
    static int staticVar = 20;

    static void staticMethod() {
        // System.out.println(instanceVar);  // ❌ compile error
        System.out.println(staticVar);       // ✅ static OK
        Demo d = new Demo();
        System.out.println(d.instanceVar);   // ✅ explicit object OK
    }
}
```

### ⚡ Remember
`static method → no this → no instance vars | Need explicit object reference`

---

<a id="q6"></a>
## Q6. How do we access static members in Java?

### 📝 One-Liner
Use the **class name**: `ClassName.staticMember` — not an instance reference (though it works, it's misleading).

### 🔑 Quick Answer
Static members belong to the class, not instances. Access via class name: `Math.PI`, `Integer.MAX_VALUE`, `Collections.sort()`. Accessing via instance reference works but is **bad practice** — it implies the member belongs to the instance. *(Static members = ClassName.member se access karo, object se nahi)*

### ⚡ Remember
`ClassName.staticMember | Not via instance | Math.PI, Integer.MAX_VALUE`

---

<a id="q7"></a>
## Q7. Can we override static methods in Java?

### 📝 One-Liner
No — static methods cannot be overridden; the subclass version **hides** the parent's static method (method hiding, not overriding).

### 🔑 Quick Answer
Static methods are resolved at **compile time** based on the reference type (not object type). If parent and child both have `static void doWork()`, the child's version **hides** (not overrides) the parent's. This is called **method hiding**. No runtime polymorphism applies. *(Static method override nahi hota — ye method hiding hai, compile-time pe resolve hota hai)*

### 💻 Code Example

```java
class Parent {
    static void greet() { System.out.println("Parent"); }
}
class Child extends Parent {
    static void greet() { System.out.println("Child"); }  // hiding, not overriding
}

Parent p = new Child();
p.greet();  // "Parent" → resolved by reference type, not object type
```

### ⚡ Remember
`Static methods = method hiding (not overriding) | Resolved at compile time by reference type`

---

<a id="q8"></a>
## Q8. Difference between object and reference?

### 📝 One-Liner
An **object** is the actual entity on heap; a **reference** is a variable on stack that holds the object's memory address.

### 🔑 Quick Answer

| Feature | Object | Reference |
|---|---|---|
| Location | Heap memory | Stack memory |
| Nature | Actual data (state + behavior) | Pointer/address to object |
| Assignment | Cannot assign object to object | Can assign ref to ref |
| Pass to method | Cannot pass object directly | Pass reference (value of address) |
| Name | Has no name | Has a variable name |
| GC | Gets garbage collected | Does not get GC'd |

*(Object = heap pe actual cheez | Reference = stack pe address jo object ko point karta hai)*

### ⚡ Remember
`Object = heap (actual) | Reference = stack (address) | Ref can reassign, object can't`

---

<a id="q9"></a>
## Q9. Objects or references — which gets garbage collected?

### 📝 One-Liner
**Objects** get garbage collected (from heap) — references (on stack) simply go out of scope.

### 🔑 Quick Answer
GC reclaims heap memory occupied by objects that have no active references pointing to them. References on the stack are not garbage collected — they disappear when their scope ends (method returns, block exits). Setting a reference to `null` makes the object eligible for GC if no other references exist. *(Objects GC hote hain, references scope se bahar jaate hain)*

### ⚡ Remember
`GC = objects on heap | References just go out of scope | null reference → object eligible for GC`

---

<a id="q10"></a>
## Q10. How many times is finalize() invoked? Who invokes it?

### 📝 One-Liner
`finalize()` is called **at most once** per object by the **garbage collector** before reclaiming memory.

### 🔑 Quick Answer
GC calls `finalize()` once on an object before it's collected. If the object is resurrected in `finalize()` (assigned to a reference), it won't be called again on the next GC cycle. `finalize()` is deprecated since Java 9 — use `try-with-resources` or `Cleaner` instead. *(finalize() = GC ek baar call karta hai, deprecated hai — try-with-resources use karo)*

### ⚡ Remember
`finalize() = once per object | Called by GC | Deprecated since Java 9 | Use try-with-resources`

---

<a id="q11"></a>
## Q11. Can we pass objects as arguments in Java?

### 📝 One-Liner
We pass **references** to objects — not the objects themselves. Java is always **pass-by-value** (the value is the reference/address).

### 🔑 Quick Answer
When you pass an object to a method, you're passing a copy of the reference (the address). The method can modify the object's state via this reference, but cannot change what the original reference points to. Java is strictly pass-by-value — for objects, the "value" is the reference address. *(Object nahi jaata, reference ka copy jaata hai — Java = always pass-by-value)*

### 💻 Code Example

```java
void modify(List<String> list) {
    list.add("new");       // ✅ modifies original object (same reference)
    list = new ArrayList<>();  // ❌ doesn't affect caller's reference
}
```

### ⚡ Remember
`Java = pass-by-value | For objects: value = reference copy | Can mutate, can't reassign`

---

<a id="q12"></a>
## Q12. Explain wrapper classes in Java?

### 📝 One-Liner
Wrapper classes convert **primitives to objects** — one wrapper class for each primitive type (e.g., `int` → `Integer`).

### 🔑 Quick Answer
Before Java 5: manual boxing (`Integer.valueOf(5)`). From Java 5: **autoboxing/unboxing** (automatic conversion). Wrapper classes are **immutable** — once created, value can't change. Used when: (1) Collections need objects (not primitives). (2) Nullability needed. (3) Methods of wrapper classes (parseInt, valueOf). *(Wrapper class = primitive ka object version | int → Integer, char → Character)*

### 🔑 Types

| Primitive | Wrapper |
|---|---|
| `boolean` | `Boolean` |
| `byte` | `Byte` |
| `char` | `Character` |
| `short` | `Short` |
| `int` | `Integer` |
| `long` | `Long` |
| `float` | `Float` |
| `double` | `Double` |

### ⚡ Remember
`Wrapper = primitive → object | Autoboxing Java 5+ | Immutable | int→Integer, char→Character`

---

<a id="q13"></a>
## Q13. Explain about transient variables in Java?

### 📝 One-Liner
`transient` marks a field to be **excluded from serialization** — its value won't be saved when the object is serialized.

### 🔑 Quick Answer
When you serialize an object, `transient` fields are skipped. On deserialization, transient fields get default values (`0` for int, `null` for objects, `false` for boolean). Use for: sensitive data (passwords), derived/calculated fields, non-serializable references. *(transient = serialize nahi hoga, default value aayegi deserialize pe)*

### 💻 Code Example

```java
public class User implements Serializable {
    private String username;
    private transient String password;  // ⭐ won't be serialized
}
// After serialize → deserialize: username = "john", password = null
```

### ⚡ Remember
`transient = skip serialization | Default value on deserialize | Use for sensitive/non-serializable data`

---

<a id="q14"></a>
## Q14. What is type conversion in Java?

### 📝 One-Liner
Assigning a value of one type to a variable of another type — either **widening** (automatic) or **narrowing** (explicit cast).

### 🔑 Quick Answer
Two types: (1) **Widening (implicit)** — smaller to larger type, automatic, no data loss: `int → long`. (2) **Narrowing (explicit)** — larger to smaller type, requires cast, possible data loss: `(byte) longValue`. Widening: `byte → short → int → long → float → double`. *(Type conversion = ek type se doosre me badalna — widening automatic, narrowing manual)*

### ⚡ Remember
`Widening = small→large (auto) | Narrowing = large→small (cast required, data loss risk)`

---

<a id="q15"></a>
## Q15. Explain automatic type conversion (widening)?

### 📝 One-Liner
When two types are compatible and the destination is **larger**, Java automatically converts — no cast needed.

### 🔑 Quick Answer
Conditions for automatic conversion: (1) Types are compatible. (2) Destination type is larger than source. Examples: `int` to `long`, `int` to `float`, `float` to `double`. No data loss in most cases (except `int/long` to `float` may lose precision). *(Widening = chhote se bade me automatic, cast ki zarurat nahi)*

### 💻 Code Example

```java
int i = 100;
long l = i;       // ✅ automatic: int → long
float f = l;      // ✅ automatic: long → float
double d = f;     // ✅ automatic: float → double
```

### ⚡ Remember
`byte→short→int→long→float→double | Automatic, no cast | Safe (mostly)`

---

<a id="q16"></a>
## Q16. Explain narrowing conversion?

### 📝 One-Liner
When destination type is **smaller** than source — requires explicit **cast** and risks data loss.

### 🔑 Quick Answer
Narrowing: larger type → smaller type. Must use explicit cast: `(targetType) value`. May result in data loss or overflow. Examples: `long → int`, `double → float`, `int → byte`. Incorrect cast on objects → `ClassCastException`. *(Narrowing = bade se chhote me — manually cast karo, data loss ho sakta hai)*

### 💻 Code Example

```java
long l = 100000L;
int i = (int) l;     // ✅ explicit cast required
double d = 99.99;
int x = (int) d;     // ✅ x = 99 (decimal lost!)
```

### ⚡ Remember
`Narrowing = explicit cast | Data loss possible | (targetType) value`

---

<a id="q17"></a>
## Q17. Explain the importance of import keyword?

### 📝 One-Liner
`import` makes classes from other packages **available by simple name** — without writing fully qualified class names.

### 🔑 Quick Answer
Declared after `package`, before class definition. `import java.util.List;` (specific) or `import java.util.*;` (wildcard). After compilation, imports are replaced with fully qualified names — it's just syntactic sugar. `java.lang.*` is imported automatically. *(import = doosre package ki class ko chhote naam se use karne ke liye)*

### ⚡ Remember
`import = short name access | package→import→class order | java.lang auto-imported`

---

<a id="q18"></a>
## Q18. Explain naming conventions for packages?

### 📝 One-Liner
All lowercase, starts with **reverse company domain**, followed by project/department structure.

### 🔑 Quick Answer
Convention: reverse domain name + project + module. Example: `com.google.search.indexer`. Always lowercase. Avoids naming conflicts across organizations. Parent directories contain child packages. *(Package naming = reverse domain + project, sab lowercase)*

### ⚡ Remember
`com.company.project.module | All lowercase | Reverse domain`

---

<a id="q19"></a>
## Q19. What is classpath?

### 📝 One-Liner
Classpath is the **path/list of directories and JARs** where JVM looks for `.class` files.

### 🔑 Quick Answer
Set via `CLASSPATH` environment variable or `-cp` flag. Can contain multiple paths separated by `;` (Windows) or `:` (Unix). JVM and compiler use it to find classes. Only parent directories needed — JVM navigates the package structure. `.` means current directory. *(Classpath = JVM ko batao .class files kahan dhundhni hain)*

### ⚡ Remember
`Classpath = where JVM finds .class | -cp flag or CLASSPATH env | ; separated on Windows`

---

<a id="q20"></a>
## Q20. What is a JAR file?

### 📝 One-Liner
JAR (Java Archive) is a **compressed package** of `.class` files, resources, and a manifest — like a ZIP for Java apps.

### 🔑 Quick Answer
Created with `jar` tool. Contains: (1) Compiled `.class` files. (2) Resources (images, configs). (3) `META-INF/MANIFEST.MF` — metadata including main class. JVM can load classes directly from JARs without extracting. Executable JAR: `java -jar app.jar`. *(JAR = Java ka ZIP file — .class files + resources + manifest)*

### ⚡ Remember
`JAR = compressed .class + resources + MANIFEST.MF | java -jar app.jar`

---

<a id="q21"></a>
## Q21. What is the scope/lifetime of instance variables?

### 📝 One-Liner
Instance variables exist as long as the **object exists** — created on `new`, destroyed when GC collects the object.

### 🔑 Quick Answer
Instance variables live on the heap as part of the object. Created when `new` allocates the object, persist until the object has no references and gets garbage collected. Each object has its own copy of instance variables. *(Instance variable = object ke saath paida, object ke saath khatam)*

### ⚡ Remember
`Instance var = heap | Created with new | Destroyed with GC | One copy per object`

---

<a id="q22"></a>
## Q22. Scope/lifetime of class (static) variables?

### 📝 One-Liner
Static variables exist for the **lifetime of the application** — created when class loads, destroyed when class unloads.

### 🔑 Quick Answer
Static variables live in the method area (metaspace). Created when the class is first loaded by ClassLoader. Shared across all instances. Persist until the application terminates (or class is unloaded). Can be accessed even before creating any object. *(Static variable = class load se app end tak — sabke liye ek hi copy)*

### ⚡ Remember
`Static var = class load to app end | Shared across instances | MetaSpace`

---

<a id="q23"></a>
## Q23. Scope/lifetime of local variables?

### 📝 One-Liner
Local variables exist **only during method execution** — created on the stack when method runs, destroyed when method returns.

### 🔑 Quick Answer
Local variables are declared inside methods, constructors, or blocks. Stored on the **stack**. Must be explicitly initialized before use (no default values). Scope: from declaration to end of the block. Not accessible outside the method. *(Local variable = method ke andar, method khatam toh variable bhi khatam)*

### ⚡ Remember
`Local var = stack | Method scope | Must initialize | No default values | GC'd when method returns`

---

<a id="q24"></a>
## Q24. Explain static imports in Java?

### 📝 One-Liner
Static imports (Java 5+) let you use **static members directly** without the class name prefix.

### 🔑 Quick Answer
Syntax: `import static package.Class.staticMember;` or `import static package.Class.*;`. After static import, use `PI` instead of `Math.PI`, `sort()` instead of `Collections.sort()`. Makes code shorter but can reduce readability if overused. *(Static import = static member ko bina class naam ke direct use karo)*

### 💻 Code Example

```java
import static java.lang.Math.PI;
import static java.lang.Math.sqrt;

double area = PI * r * r;       // instead of Math.PI
double root = sqrt(144);        // instead of Math.sqrt(144)
```

### ⚡ Remember
`import static package.Class.member | Use without class prefix | Don't overuse`

---

<a id="q25"></a>
## Q25. Can we define static methods inside an interface?

### 📝 One-Liner
Before Java 8: **No**. From Java 8: **Yes** — interfaces can have `static` methods with body.

### 🔑 Quick Answer
Java 8+ allows static methods in interfaces. They must have a body (implementation). Called using `InterfaceName.method()` — not through implementing class. Not inherited by implementing classes. Also added: `default` methods. Java 9 added `private` methods in interfaces. *(Java 8 se interface me static method allowed hai — InterfaceName.method() se call karo)*

### 💻 Code Example

```java
interface Validator {
    boolean validate(String input);

    static boolean isNullOrEmpty(String s) {  // ⭐ Java 8+ static method
        return s == null || s.isEmpty();
    }
}
Validator.isNullOrEmpty("");  // ✅ call via interface name
```

### ⚡ Remember
`Interface static methods = Java 8+ | Must have body | Called via InterfaceName.method()`

---

<a id="q26"></a>
## Q26. Define interface in Java?

### 📝 One-Liner
An interface is a **100% abstract contract** (pre-Java 8) or **capability blueprint** — implemented using `implements` keyword.

### 🔑 Quick Answer
Interface = collection of abstract methods + constants. Keyword: `interface`. Implementing class uses `implements` and must override all abstract methods. From Java 8: can have `default` and `static` methods. From Java 9: `private` methods. Variables are implicitly `public static final`. Methods implicitly `public abstract`. *(Interface = contract/capability — class ko bataata hai kya karna hai, kaise nahi)*

### ⚡ Remember
`Interface = contract | implements keyword | Variables = public static final | Methods = public abstract`

---

<a id="q27"></a>
## Q27. What is the purpose of interface?

### 📝 One-Liner
Interface defines a **contract** — what a class must do, not how — enabling polymorphism and loose coupling.

### 🔑 Quick Answer
Purpose: (1) Define a contract between caller and implementer. (2) Enable polymorphism — different classes, same interface. (3) Multiple inheritance — a class can implement multiple interfaces. (4) Dynamic method resolution at runtime. (5) Loose coupling — depend on interface, not implementation. *(Interface = contract + multiple inheritance + loose coupling)*

### ⚡ Remember
`Interface = WHAT not HOW | Contract + Polymorphism + Multiple inheritance + Loose coupling`

---

<a id="q28"></a>
## Q28. Explain features of interfaces in Java?

### 📝 One-Liner
All methods implicitly `public abstract`, variables `public static final`, can't instantiate, multiple inheritance, and `implements` keyword.

### 🔑 Quick Answer
Features: (1) Methods are implicitly `public abstract` (before Java 8). (2) Variables are `public static final`. (3) Cannot instantiate. (4) Cannot have constructors. (5) `implements` keyword to implement. (6) Can extend multiple interfaces. (7) Can define class inside interface (inner class). (8) From Java 8: `default` + `static` methods. (9) From Java 9: `private` methods. (10) Multiple inheritance achieved. *(Interface features = 10 points yaad rakho — methods public abstract, variables public static final)*

### ⚡ Remember
`Methods = public abstract | Vars = public static final | No constructors | Multiple extend | implements`

---

<a id="q29"></a>
## Q29. Explain enumeration (enum) in Java?

### 📝 One-Liner
An `enum` is a **set of named constants** — type-safe, each constant is `public static final`, declared with `enum` keyword.

### 🔑 Quick Answer
Java 5+ feature. Each enum constant is an instance of the enum class. Enums can have: fields, methods, constructors (private). Constants declared first, then fields/methods. Enums implicitly extend `java.lang.Enum`. Thread-safe singleton by design. Methods: `values()`, `valueOf()`, `ordinal()`, `name()`. *(enum = named constants ka type-safe set — Days.MONDAY, Status.ACTIVE)*

### 💻 Code Example

```java
public enum Status {
    ACTIVE("Active"), INACTIVE("Inactive"), DELETED("Deleted");

    private final String label;
    Status(String label) { this.label = label; }  // constructor (private)
    public String getLabel() { return label; }
}
// Status.ACTIVE.getLabel() → "Active"
// Status.values() → [ACTIVE, INACTIVE, DELETED]
```

### ⚡ Remember
`enum = named constants | Can have fields/methods | Thread-safe | Extends java.lang.Enum`

---

<a id="q30"></a>
## Q30. Explain restrictions on using enum?

### 📝 One-Liner
Enums **cannot extend** other classes/enums, cannot be **instantiated** with `new`, fields/methods must come **after** constants.

### 🔑 Quick Answer
Restrictions: (1) Cannot extend any class or enum (already extends `java.lang.Enum`). (2) Cannot instantiate with `new`. (3) Fields and methods must be declared after enum constants. (4) Constants must be declared first. (5) Constructor is implicitly private. *(enum ya koi class extend nahi kar sakta, new se create nahi ho sakta)*

### ⚡ Remember
`enum: no extends | no new | constants first | private constructor`

---

<a id="q31"></a>
## Q31. Explain field hiding in Java?

### 📝 One-Liner
When a subclass declares a field with the **same name** as superclass — the subclass field **hides** the parent's (resolved by reference type).

### 🔑 Quick Answer
Unlike method overriding (runtime), field hiding is resolved at **compile time** based on the reference type. Use `super.fieldName` to access parent's hidden field. Best practice: avoid field hiding — it creates confusion. *(Field hiding = same naam ka field parent-child me — compile-time pe reference type decide karta hai)*

### 💻 Code Example

```java
class Parent { String name = "Parent"; }
class Child extends Parent { String name = "Child"; }

Parent p = new Child();
System.out.println(p.name);  // "Parent" → reference type decides (not object type!)
```

### ⚡ Remember
`Field hiding = compile-time | Reference type decides | Use super.field | Avoid it`

---

<a id="q32"></a>
## Q32. Explain Varargs in Java?

### 📝 One-Liner
Varargs (Java 5+) allows a method to accept **variable number of arguments** using ellipsis `...` syntax.

### 🔑 Quick Answer
Syntax: `void method(Type... args)`. Internally treated as an array. Must be the **last parameter**. Only one varargs per method. If no arguments passed, array with size 0 (no null check needed). *(Varargs = variable arguments, 0 se lekar kitne bhi arguments de sakte ho)*

### 💻 Code Example

```java
public int sum(int... numbers) {  // ⭐ varargs = array internally
    int total = 0;
    for (int n : numbers) total += n;
    return total;
}
sum();          // 0 (empty array)
sum(1, 2, 3);  // 6
sum(1, 2, 3, 4, 5);  // 15
```

### ⚡ Remember
`Type... args | Must be last param | Only one per method | Empty = size 0 array`

---

<a id="q33"></a>
## Q33. Where are variables created in memory?

### 📝 One-Liner
Local variables on **stack**, instance variables on **heap** (with object), static variables in **method area** (metaspace).

### 🔑 Quick Answer

| Variable Type | Memory Location | Lifetime |
|---|---|---|
| Local | Stack | Method execution |
| Instance | Heap (part of object) | Until GC collects object |
| Static | Method Area / Metaspace | Application lifetime |

*(Local = stack | Instance = heap | Static = metaspace)*

### ⚡ Remember
`Local→Stack | Instance→Heap | Static→Metaspace`

---

<a id="q34"></a>
## Q34. Can we use Switch statement with Strings?

### 📝 One-Liner
Yes, from **Java 7** — `switch` supports String comparison using `.equals()` internally.

### 🔑 Quick Answer
Before Java 7: only `int`, `char`, `byte`, `short`, and `enum` in switch. Java 7+: String support added. Internally uses `hashCode()` for efficiency, then `.equals()` for verification. Case-sensitive. `null` causes `NullPointerException`. Java 14+: switch expressions with `yield`. *(Java 7 se String switch me chal sakta hai — hashCode + equals internally)*

### 💻 Code Example

```java
String day = "MON";
switch (day) {
    case "MON": System.out.println("Monday"); break;
    case "TUE": System.out.println("Tuesday"); break;
    default: System.out.println("Other");
}
// Java 14+ switch expression:
String result = switch (day) {
    case "MON" -> "Monday";
    case "TUE" -> "Tuesday";
    default -> "Other";
};
```

### ⚡ Remember
`String in switch = Java 7+ | hashCode + equals | Case-sensitive | null → NPE`

---

<a id="q35"></a>
## Q35. How do we copy objects in Java?

### 📝 One-Liner
Assign reference copies the **address** (shallow); for actual copy use `clone()`, copy constructor, or serialization.

### 🔑 Quick Answer
`r2 = r1` → both point to same object (no copy). For actual copies: (1) **Shallow clone** — `Cloneable` + `clone()` (copies fields but references still shared). (2) **Deep copy** — manually copy nested objects. (3) **Copy constructor** — `new Employee(existingEmployee)`. (4) **Serialization** — serialize + deserialize. (5) Mapper libraries like MapStruct. *(r2 = r1 se object copy nahi hota, sirf reference copy hota hai — actual copy ke liye clone/copy constructor chahiye)*

### 💻 Code Example

```java
Employee e1 = new Employee("Alice");
Employee e2 = e1;           // ❌ same object, just ref copy!
e2.setName("Bob");
e1.getName();               // "Bob" — both point to same object

// Copy constructor
Employee e3 = new Employee(e1);  // ✅ new object with copied data
```

### ⚡ Remember
`= copies reference | clone() = shallow | Copy constructor = clean approach | Serialization = deep copy`
