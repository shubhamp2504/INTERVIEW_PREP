# ☕ Core Java — Nested Classes & Inner Classes (Q1–Q12)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Static nested, method-local inner, anonymous inner, member inner classes

---

<a id="q1"></a>
## Q1. What are nested classes in Java?

### 📝 One-Liner
A nested class is a class **declared inside another class** — either static nested or non-static inner class.

### 🔑 Quick Answer
Two types: (1) **Static nested class** — declared with `static`, can be instantiated without outer class instance. (2) **Non-static nested class (Inner class)** — requires outer class instance. Inner classes further split into: member inner, local inner (method-local), and anonymous inner. *(Nested class = ek class ke andar doosri class — static ya non-static)*

### 📖 How It Works

```
Nested Class Types:
┌─────────────────────────────────────┐
│  Nested Classes                     │
│  ├── Static Nested Class            │
│  └── Inner Classes (non-static)     │
│       ├── Member Inner Class        │
│       ├── Local Inner Class         │
│       └── Anonymous Inner Class     │
└─────────────────────────────────────┘
```

### ⚡ Remember
`Nested = static nested + inner | Inner = member + local + anonymous`

---

<a id="q2"></a>
## Q2. What are inner classes (non-static nested classes)?

### 📝 One-Liner
Inner classes are non-static nested classes that have access to the **enclosing class's members** including private — requires an outer class instance.

### 🔑 Quick Answer
Three types of inner classes: (1) **Member inner class** — defined at class level (like a field). (2) **Local inner class** — defined inside a method. (3) **Anonymous inner class** — no name, defined and instantiated in one expression. All inner classes can access outer class's private members. *(Inner class = non-static nested, outer class ke private members bhi access kar sakta hai)*

### ⚡ Remember
`Inner class = non-static | Access outer private | 3 types: member, local, anonymous`

---

<a id="q3"></a>
## Q3. Why use nested classes in Java?

### 📝 One-Liner
Grouping related classes, increasing encapsulation, making code more readable and maintainable, hiding implementation.

### 🔑 Quick Answer
**(1) Grouping** — classes only used by one outer class don't need to be standalone. **(2) Encapsulation** — inner class accesses outer's private members without getters/setters. **(3) Readability** — related logic stays together. **(4) Implementation hiding** — inner class is hidden from outside world. Example: a Button's click handler as inner class — only relevant to that Button. *(Nested class = related code ek jagah, better encapsulation, implementation hide karna)*

### ⚡ Remember
`Why nested: grouping + encapsulation + readability + hide implementation`

---

<a id="q4"></a>
## Q4. Explain static nested classes?

### 📝 One-Liner
A static nested class is declared with `static` inside another class — it **doesn't need** an outer class instance and **cannot access** non-static outer members.

### 🔑 Quick Answer
Static nested class: (1) Declared with `static` keyword. (2) Can be instantiated without outer class instance. (3) Can access outer class's **static members** only. (4) Cannot access instance variables/methods of outer class. (5) Behaves like a top-level class that's packaged inside another for namespace purposes. *(Static nested = outer instance nahi chahiye, sirf static members access kar sakta hai)*

### 💻 Code Example

```java
class Outer {
    private static int staticVar = 10;
    private int instanceVar = 20;

    static class StaticNested {
        void display() {
            System.out.println(staticVar);     // ✅ static access OK
            // System.out.println(instanceVar); // ❌ non-static not accessible
        }
    }
}

// Instantiation — no outer instance needed
Outer.StaticNested nested = new Outer.StaticNested();
```

### ⚡ Remember
`static nested = no outer instance | Only static members accessible | Like packaged top-level class`

---

<a id="q5"></a>
## Q5. How to instantiate static nested classes?

### 📝 One-Liner
`OuterClass.StaticNestedClass ref = new OuterClass.StaticNestedClass();` — no outer instance needed.

### 🔑 Quick Answer
Use the outer class name as a qualifier with dot notation. No need to create an outer class object first. Access static members and methods of outer class directly. *(OuterClass.NestedClass ref = new OuterClass.NestedClass() — simple hai, outer ka object nahi chahiye)*

### ⚡ Remember
`new OuterClass.StaticNested() — no outer instance | Dot notation with outer class name`

---

<a id="q6"></a>
## Q6. Explain method-local inner classes (local inner classes)?

### 📝 One-Liner
A class defined **inside a method** — objects can only be created within that method, and it goes out of scope when the method returns.

### 🔑 Quick Answer
Local inner class: (1) Defined inside a method body. (2) Can only create objects within the method. (3) Exists only during method execution. (4) Can access only **final** (or effectively final) local variables. (5) Cannot have access modifiers (public/private) or be static. (6) Can be defined inside loops and if blocks. *(Method local inner class = method ke andar defined, method ke bahar exist nahi karta)*

### 💻 Code Example

```java
class Outer {
    void myMethod() {
        final String msg = "Hello";  // must be final/effectively final

        class LocalInner {           // ⭐ defined inside method
            void display() {
                System.out.println(msg);  // ✅ can access final local var
            }
        }

        LocalInner inner = new LocalInner();  // ⭐ only usable here
        inner.display();
    }
    // LocalInner is NOT accessible outside myMethod()
}
```

### ⚡ Remember
`Local inner = inside method | Only final vars | No access modifiers | Scope = method only`

---

<a id="q7"></a>
## Q7. Explain features of local inner class?

### 📝 One-Liner
No access specifiers, cannot be static, can use `abstract`/`final`, can only access `final` local variables, can be in loops/if blocks.

### 🔑 Quick Answer
Features: (1) No access specifiers allowed (not public/private/protected). (2) Cannot use `static` modifier. (3) Can use `abstract` and `final`. (4) Cannot declare static members inside. (5) Can only access `final` or effectively final variables of enclosing method. (6) Can be defined inside loops (`for`, `while`) and blocks (`if`). *(Local inner class ke rules: no access modifier, no static, sirf final local variables)*

### ⚡ Remember
`No access modifiers | No static | Only final locals | Can be in loops/if`

---

<a id="q8"></a>
## Q8. Explain anonymous inner classes in Java?

### 📝 One-Liner
An inner class **without a name** — declared and instantiated in a single expression using `new`, typically to implement interfaces or extend classes inline.

### 🔑 Quick Answer
Anonymous inner class: (1) No class name. (2) Declared and instantiated together with `new`. (3) Main purpose: provide interface implementation or extend a class inline. (4) Use when only one instance is needed. (5) Can access all enclosing class members + final local variables. The compiler generates `EnclosingClass$1.class` for it. *(Anonymous inner class = bina naam ka class, ek hi jagah define aur use)*

### 💻 Code Example

```java
// Anonymous inner class implementing Runnable
Runnable r = new Runnable() {  // ⭐ looks like instantiating interface
    @Override
    public void run() {
        System.out.println("Running in anonymous class");
    }
};
new Thread(r).start();

// Since Java 8, prefer lambdas for functional interfaces:
Runnable r2 = () -> System.out.println("Lambda version");
```

### ⚡ Remember
`Anonymous = no name | new Interface() { impl } | One-time use | Java 8+ → prefer lambda`

---

<a id="q9"></a>
## Q9. Explain restrictions for anonymous inner classes?

### 📝 One-Liner
No constructors (no name to call), no static methods/fields, cannot define interfaces anonymously, instantiated only once.

### 🔑 Quick Answer
Restrictions: (1) No constructor — no class name means no constructor name. (2) Cannot define static methods, fields, or classes. (3) Cannot define an interface anonymously. (4) Can only be instantiated once. (5) Can only implement one interface or extend one class (not both). *(Anonymous class = no constructor, no static, sirf ek baar instantiate)*

### ⚡ Remember
`No constructor | No static | One interface OR one class | One instance only`

---

<a id="q10"></a>
## Q10. Can we instantiate an interface in Java?

### 📝 One-Liner
No directly — but `new Runnable() { ... }` creates an **anonymous inner class** that implements the interface, not the interface itself.

### 🔑 Quick Answer
```java
Runnable r = new Runnable() {
    @Override
    public void run() { }
};
```
This looks like instantiating an interface, but it's actually creating an anonymous inner class that implements `Runnable`. The `new Runnable()` is syntactic sugar — the compiler generates a class file `Outer$1.class`. Interfaces themselves cannot be instantiated. *(Interface directly instantiate nahi hota — ye anonymous inner class hai jo interface implement karti hai)*

### ⚡ Remember
`new Interface() { } = anonymous class implementing interface | Interface itself not instantiated`

---

<a id="q11"></a>
## Q11. Explain member inner classes?

### 📝 One-Liner
A non-static class defined at **member level** (like a field) of the enclosing class — can access all outer class members including private.

### 🔑 Quick Answer
Member inner class: (1) Defined at class level (not inside a method). (2) Can be `abstract` or `final`. (3) Can extend a class or implement interfaces. (4) Cannot declare `static` fields/methods. (5) Can use all access modifiers (`public`, `private`, `protected`, default). (6) Has access to all outer class members, including private. Requires outer class instance for instantiation. *(Member inner class = outer class ke member level pe define, private members bhi access kar sakta hai)*

### 💻 Code Example

```java
class Outer {
    private int secret = 42;

    class MemberInner {              // ⭐ member-level inner class
        void reveal() {
            System.out.println(secret);  // ✅ accesses outer's private
        }
    }
}

// Instantiation requires outer instance
Outer outer = new Outer();
Outer.MemberInner inner = outer.new MemberInner();  // ⭐ outer.new
inner.reveal();  // 42
```

### ⚡ Remember
`Member inner = class level | Access private | outer.new InnerClass() to instantiate`

---

<a id="q12"></a>
## Q12. How to instantiate member inner class?

### 📝 One-Liner
`OuterClass.InnerClass inner = outerInstance.new InnerClass();` — requires an outer class object.

### 🔑 Quick Answer
Member inner class needs an outer class instance because it's tied to the outer object's state. Syntax: (1) Create outer: `Outer out = new Outer()`. (2) Create inner: `Outer.Inner in = out.new Inner()`. Cannot instantiate inner class without outer reference. *(Outer ka object chahiye pehle, phir outer.new Inner() se inner banao)*

### ⚡ Remember
`outerRef.new InnerClass() | Outer instance required | Cannot exist without outer`
