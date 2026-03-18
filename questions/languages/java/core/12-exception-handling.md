# ☕ Core Java — Exception Handling (Q1–Q27)

> **Source**: 240 Core Java Interview Questions PDF  
> **Coverage**: Exception basics, try/catch/finally, checked/unchecked, throw/throws, custom exceptions, Throwable hierarchy

---

<a id="q1"></a>
## Q1. What is an exception in Java?

### 📝 One-Liner
An exception is an **abnormal event** during program execution that disrupts normal flow — represented as an object in `java.lang`.

### 🔑 Quick Answer
An exception is a runtime error represented as an object. When an abnormal situation occurs, JVM creates an exception object with: name, description, and stack trace location. All exception classes extend `java.lang.Throwable`. Exceptions can be created by JVM (e.g., `NullPointerException`) or by application code (using `throw`). *(Exception = program execution ke dauran abnormal situation, ek object ke roop me)*

### 📖 How It Works

```
Throwable Hierarchy:
┌──────────────┐
│  Throwable   │
├──────┬───────┤
│ Error│Exception│
│      ├────────┤
│      │RuntimeException│ → Unchecked
│      │IOException     │ → Checked
│      │SQLException    │ → Checked
└──────┴────────────────┘
```

### ⚡ Remember
`Exception = abnormal event object | JVM or code creates it | java.lang.Throwable hierarchy`

---

<a id="q2"></a>
## Q2. State some situations where exceptions may arise?

### 📝 One-Liner
Array out of bounds, number format errors, invalid class casts, trying to instantiate abstract classes.

### 🔑 Quick Answer
Common exception scenarios: (1) `ArrayIndexOutOfBoundsException` — accessing invalid index. (2) `NumberFormatException` — invalid string to number conversion. (3) `ClassCastException` — invalid type casting. (4) `InstantiationException` — creating instance of abstract class/interface. (5) `NullPointerException` — method call on null. (6) `StackOverflowError` — infinite recursion. (7) `FileNotFoundException` — accessing missing file. *(Common exceptions yaad rakho — interview me scenario poochte hain)*

### ⚡ Remember
`NPE, ArrayIndex, ClassCast, NumberFormat, FileNotFound — ye sabse common hain`

---

<a id="q3"></a>
## Q3. What is Exception handling in Java?

### 📝 One-Liner
A mechanism to **gracefully handle runtime errors** so the program continues normally instead of crashing.

### 🔑 Quick Answer
Exception handling lets you: (1) Separate error-handling code from normal code. (2) Prevent abrupt program termination. (3) Propagate errors up the call stack. Done with `try-catch-finally` blocks or `throws` declarations. Without exception handling, the program terminates immediately when an error occurs. *(Exception handling = error aane pe program crash na ho, gracefully handle ho)*

### ⚡ Remember
`Exception handling = try/catch/finally + throws | Program crash nahi karta, handle karta hai`

---

<a id="q4"></a>
## Q4. What is an Error in Java?

### 📝 One-Liner
An Error is a **serious, unrecoverable problem** (subclass of `Throwable`) caused by the environment — not the application.

### 🔑 Quick Answer
Errors are subclass of `Throwable`, not `Exception`. They represent conditions that applications **should not try to catch** — typically caused by JVM or system issues: `OutOfMemoryError`, `StackOverflowError`, `NoClassDefFoundError`. Unlike exceptions, errors cannot be recovered from programmatically. *(Error = system-level problem, recover nahi ho sakta — OutOfMemory, StackOverflow)*

### 🆚 vs.
| Error | Exception |
|---|---|
| Unrecoverable | Recoverable |
| Caused by environment/JVM | Caused by application logic |
| Should NOT catch | Should catch and handle |
| `OutOfMemoryError` | `IOException` |

### ⚡ Remember
`Error = unrecoverable + JVM/system issue | Don't catch errors, fix root cause`

---

<a id="q5"></a>
## Q5. What are advantages of Exception handling?

### 📝 One-Liner
Separates error code from normal flow, enables error categorization, and supports automatic call-stack propagation.

### 🔑 Quick Answer
**(1) Separation** — error handling code separate from business logic. **(2) Categorization** — handle specific exceptions differently (FileNotFound vs SQL vs NPE). **(3) Call stack propagation** — if a method can't handle it, exception automatically propagates to the caller until a handler is found. **(4) Prevents abrupt termination** — program continues after handling. *(Exception handling ke advantages: clean code, specific handling, automatic propagation)*

### ⚡ Remember
`Separate error code | Categorize exceptions | Auto propagation | No abrupt termination`

---

<a id="q6"></a>
## Q6. In how many ways can we do exception handling?

### 📝 One-Liner
Two ways: **(1)** `try-catch` block to handle locally; **(2)** `throws` clause to delegate to the caller.

### 🔑 Quick Answer
**(1) try-catch**: Surround risky code in `try`, handle in `catch`. **(2) throws declaration**: Declare in method signature — delegates responsibility to caller. Choose based on who should handle: current method (try-catch) or caller (throws). *(Do tarike: try-catch se khud handle karo, ya throws se caller ko de do)*

### ⚡ Remember
`try-catch = handle here | throws = let caller handle`

---

<a id="q7"></a>
## Q7. List five keywords related to Exception handling?

### 📝 One-Liner
`try`, `catch`, `finally`, `throw`, `throws` — the five pillars of Java exception handling.

### 🔑 Quick Answer

| Keyword | Purpose |
|---|---|
| `try` | Wraps code that might throw exception |
| `catch` | Handles the thrown exception |
| `finally` | Always executes (cleanup code) |
| `throw` | Explicitly throws an exception object |
| `throws` | Declares exceptions a method may throw |

*(5 keywords: try, catch, finally, throw, throws — interview me jaroor poochenge)*

### ⚡ Remember
`try-catch-finally = handling | throw = explicit throw | throws = method declaration`

---

<a id="q8"></a>
## Q8. Explain try and catch keywords?

### 📝 One-Liner
`try` wraps risky code; `catch` handles the exception thrown by the `try` block — they form a unit.

### 🔑 Quick Answer
`try` block contains code that might throw an exception. `catch` block catches and handles the exception. They must be paired — catch cannot exist without try. If no exception in try, catch is skipped. JVM ignores try-catch entirely if no exception-causing code exists. *(try = risk wala code | catch = exception handle karo | Ye pair me chalte hain)*

### 💻 Code Example

```java
try {
    int result = 10 / 0;  // ArithmeticException
} catch (ArithmeticException e) {
    System.out.println("Cannot divide by zero: " + e.getMessage());
}
// Program continues normally after catch
```

### ⚡ Remember
`try { risky } catch(Exception e) { handle } | Try-catch = ek unit`

---

<a id="q9"></a>
## Q9. Can we have try block without catch block?

### 📝 One-Liner
Yes, but only if `finally` block is present — a try block requires **at least** catch or finally (or both).

### 🔑 Quick Answer
`try` alone → compile error. `try-catch` → valid. `try-finally` → valid. `try-catch-finally` → valid. A try block without both catch and finally will not compile. *(try akela nahi chal sakta — catch ya finally me se ek toh chahiye)*

### 💻 Code Example

```java
// ✅ Valid: try-finally (no catch)
try {
    // code
} finally {
    // cleanup
}

// ❌ Invalid: try alone
// try { }  → compile error
```

### ⚡ Remember
`try needs catch OR finally (or both) | try alone = compile error`

---

<a id="q10"></a>
## Q10. Can we have multiple catch blocks for a try block?

### 📝 One-Liner
Yes — multiple catch blocks handle **different exception types**; order must be **child to parent**.

### 🔑 Quick Answer
Multiple catch blocks handle different exceptions from the same try. JVM checks each catch in order — first matching type executes, rest are skipped. **Order matters**: specific (child) exceptions first, general (parent) last. Putting parent before child → compile error: "unreachable catch block." *(Multiple catch = haan, lekin order child → parent hona chahiye)*

### 💻 Code Example

```java
try {
    // code that may throw multiple exceptions
} catch (FileNotFoundException e) {     // specific first
    System.out.println("File not found");
} catch (IOException e) {               // parent after child
    System.out.println("IO error");
} catch (Exception e) {                 // most general last
    System.out.println("General error");
}

// ❌ Wrong order → compile error:
// catch (Exception e) { }           // catches everything first
// catch (IOException e) { }          // unreachable!
```

### ⚡ Remember
`Multiple catch = yes | Order: child → parent | Wrong order = compile error`

---

<a id="q11"></a>
## Q11. Explain importance of finally block?

### 📝 One-Liner
`finally` **always executes** (exception or not) — used for cleanup like closing connections, streams, sockets.

### 🔑 Quick Answer
Finally block runs: (1) After try if no exception. (2) After catch if exception was caught. (3) Even if catch doesn't handle the exception. Use for cleanup code: closing DB connections, file streams, releasing locks. With try-with-resources (Java 7+), many `finally` uses are replaced. *(finally = hamesha chalega — cleanup ke liye: connection band karo, stream close karo)*

### ⚡ Remember
`finally = always runs | Cleanup: close connections/streams | try-with-resources preferred`

---

<a id="q12"></a>
## Q12. Can we have code between try and catch blocks?

### 📝 One-Liner
No — catch must **immediately follow** try. Any code between them causes a compile error.

### 🔑 Quick Answer
```java
// ❌ Illegal
try { } 
System.out.println("illegal");  // compile error
catch (Exception e) { }

// ✅ Legal
try { } catch (Exception e) { }  // catch immediately after try
```

### ⚡ Remember
`try → catch must be immediate | No code between them`

---

<a id="q13"></a>
## Q13. Can we have code between try and finally blocks?

### 📝 One-Liner
No — `finally` must immediately follow the `try` (or `catch`) block, no code in between.

### 🔑 Quick Answer
Same rule: try, catch, finally must form a continuous block with no statements between them. Code between try and finally → compile error. *(try-catch-finally ke beech me koi code nahi aa sakta)*

### ⚡ Remember
`try-catch-finally = continuous block | No gaps allowed`

---

<a id="q14"></a>
## Q14. Can we catch more than one exception in a single catch block?

### 📝 One-Liner
Yes, from **Java 7** — use multi-catch with pipe `|` operator; the catch parameter becomes implicitly `final`.

### 🔑 Quick Answer
Java 7+ multi-catch syntax: `catch (TypeA | TypeB e)`. Reduces code duplication when handling different exceptions the same way. The catch variable `e` is implicitly `final` — cannot be reassigned. Exceptions in multi-catch cannot have parent-child relationship. *(Java 7+ me ek catch me multiple exceptions pipe | se pakad sakte ho)*

### 💻 Code Example

```java
try {
    // code
} catch (ArrayIndexOutOfBoundsException | ArithmeticException e) {
    // e is implicitly final — cannot reassign
    System.out.println("Error: " + e.getMessage());
}

// ❌ Cannot mix parent-child:
// catch (IOException | FileNotFoundException e)  → compile error
// FileNotFoundException IS-A IOException
```

### ⚡ Remember
`Multi-catch = Type1 | Type2 | Java 7+ | e is final | No parent-child mix`

---

<a id="q15"></a>
## Q15. What is default Exception handling in Java?

### 📝 One-Liner
If no handler is found, JVM's **default exception handler** prints the stack trace and **terminates** the program.

### 🔑 Quick Answer
When an exception is thrown and no matching catch is found, the exception propagates up the call stack. If it reaches main() with no handler, JVM's default handler: (1) Prints exception name + description. (2) Prints stack trace (location). (3) Terminates the program. Main disadvantage: **abrupt termination** — that's why we handle exceptions. *(Default handler = stack trace print karo aur program band karo — isliye hum handle karte hain)*

### ⚡ Remember
`No handler → default handler → print stack trace → terminate | That's why we catch exceptions`

---

<a id="q16"></a>
## Q16. Explain throw keyword in Java?

### 📝 One-Liner
`throw` is used to **explicitly throw** an exception object — stops execution immediately and passes control to the nearest catch.

### 🔑 Quick Answer
Syntax: `throw new ExceptionType("message")`. Used for throwing user-defined or runtime exceptions explicitly. After `throw`, execution stops — subsequent statements are unreachable. The thrown object must be a subclass of `Throwable`. JVM then looks for a matching catch block up the call stack. *(throw = manually exception fenko | throw ke baad koi code nahi chalega)*

### 💻 Code Example

```java
public void setAge(int age) {
    if (age < 0) {
        throw new IllegalArgumentException("Age cannot be negative: " + age);
    }
    this.age = age;
}
// throw creates an exception object and transfers control to catch
```

### ⚡ Remember
`throw = explicit throw | Must be Throwable subclass | Code after throw = unreachable`

---

<a id="q17"></a>
## Q17. Can we write any code after throw statement?

### 📝 One-Liner
No — code after `throw` is **unreachable** and causes a compile error.

### 🔑 Quick Answer
After a `throw` statement, JVM stops execution of the current method and passes control to the exception handler. Any statements written after `throw` are flagged by the compiler as "unreachable code." *(throw ke baad code likhoge toh compile error aayega — unreachable code)*

### ⚡ Remember
`Throw → execution stops → code after = compile error (unreachable)`

---

<a id="q18"></a>
## Q18. Explain importance of throws keyword?

### 📝 One-Liner
`throws` declares in the method signature that this method **may throw** specified exceptions — delegating handling to the caller.

### 🔑 Quick Answer
Syntax: `void method() throws IOException, SQLException`. Purpose: (1) For checked exceptions — tells compiler that caller must handle. (2) Delegates exception handling responsibility to the caller. (3) Not needed for unchecked exceptions (though optional). The method doesn't handle the exception — it passes it up. *(throws = method ke signature me declare — caller ko bolo ki handle karo)*

### 🆚 vs.
| `throw` | `throws` |
|---|---|
| Inside method body | In method signature |
| Actually throws exception | Declares possibility |
| Followed by exception object | Followed by exception class names |
| `throw new IOException()` | `void m() throws IOException` |

### ⚡ Remember
`throw = actually throws | throws = declares in signature | throw = body | throws = declaration`

---

<a id="q19"></a>
## Q19. Importance of finally over return statement?

### 📝 One-Liner
`finally` executes **even if** try or catch has a `return` statement — finally runs before the return actually happens.

### 🔑 Quick Answer
If a `return` statement exists in try or catch, the finally block still executes first, then the return completes. This ensures cleanup happens regardless. However, if finally also has a `return`, it **overrides** the try/catch return value (bad practice). *(return hone ke baad bhi finally chalega — phir return hoga)*

### 💻 Code Example

```java
public int getValue() {
    try {
        return 1;          // return planned
    } finally {
        System.out.println("finally runs!");  // ⭐ runs before return
    }
}
// Output: "finally runs!" then returns 1
```

### ⚡ Remember
`finally runs BEFORE return | finally's return overrides try's return (don't do it)`

---

<a id="q20"></a>
## Q20. When will finally block NOT be executed?

### 📝 One-Liner
Only when `System.exit()` is called or the **JVM crashes/shuts down** — otherwise finally always runs.

### 🔑 Quick Answer
finally will NOT execute when: (1) `System.exit(0)` is called in try/catch. (2) JVM crashes or receives `kill -9`. (3) Infinite loop/deadlock in try block prevents reaching finally. (4) Daemon thread's finally if JVM exits. In all other cases, including exceptions, returns, and breaks — finally runs. *(System.exit() ya JVM crash hone pe hi finally nahi chalega)*

### ⚡ Remember
`System.exit() → finally skipped | JVM crash → skipped | Everything else → finally runs`

---

<a id="q21"></a>
## Q21. Can we use catch statement for checked exceptions when no possibility of throwing?

### 📝 One-Liner
No — catching a checked exception that can't possibly be thrown causes a **compile error** ("unreachable catch block").

### 🔑 Quick Answer
If there's no code in the try block that can throw a specific checked exception, the compiler won't allow catching it. This only applies to checked exceptions — catching `Exception` or `RuntimeException` is always allowed since unchecked exceptions can occur anywhere. *(Agar code me checked exception ka chance hi nahi hai, toh catch likhoge toh compile error)*

### ⚡ Remember
`Unreachable catch for checked exception = compile error | Doesn't apply to unchecked`

---

<a id="q22"></a>
## Q22. What are user-defined exceptions?

### 📝 One-Liner
Custom exception classes you create by extending `Exception` (checked) or `RuntimeException` (unchecked).

### 🔑 Quick Answer
Create user-defined exceptions for domain-specific error messages. Extend `Exception` for checked, `RuntimeException` for unchecked (recommended). Include meaningful error message in constructor. Common pattern: create constructor accepting message String that calls `super(message)`. *(Custom exception = apni domain-specific error class, RuntimeException extend karo)*

### 💻 Code Example

```java
// Recommended: unchecked (extends RuntimeException)
public class InsufficientBalanceException extends RuntimeException {
    private final double balance;
    
    public InsufficientBalanceException(double amount, double balance) {
        super("Cannot withdraw " + amount + ", balance: " + balance);
        this.balance = balance;
    }
    
    public double getBalance() { return balance; }
}

// Usage:
if (amount > balance) {
    throw new InsufficientBalanceException(amount, balance);
}
```

### ⚡ Remember
`Custom exception = extend RuntimeException (unchecked) or Exception (checked) | Meaningful message`

---

<a id="q23"></a>
## Q23. Can we rethrow the same exception from catch handler?

### 📝 One-Liner
Yes — simply `throw e;` inside the catch block to rethrow after logging or partial handling.

### 🔑 Quick Answer
You can rethrow the caught exception using `throw e;` in the catch block. Useful when you want to log and propagate. For checked exceptions, the method must declare the exception in its `throws` clause. From Java 7, the compiler is smarter about rethrowing — it tracks the actual exception type. *(Haan, catch me throw e se wahi exception wapas fek sakte hain — log karo phir propagate karo)*

### 💻 Code Example

```java
public void process() throws IOException {
    try {
        readFile();
    } catch (IOException e) {
        logger.error("File read failed", e);
        throw e;  // ⭐ rethrow after logging
    }
}
```

### ⚡ Remember
`Rethrow = catch → log → throw e | Must declare in throws for checked exceptions`

---

<a id="q24"></a>
## Q24. Can we have nested try statements?

### 📝 One-Liner
Yes — a `try` block can contain another `try-catch` inside it, allowing **layered exception handling**.

### 🔑 Quick Answer
Nested try: inner try's exception → inner catch handles. If inner catch can't handle → propagates to outer catch. Useful when different parts of code throw different exceptions. Don't over-nest — prefer separate methods for clean code. *(Haan, try ke andar try ho sakta hai — inner exception upar propagate hota hai agar handle nahi hua)*

### ⚡ Remember
`Nested try = yes | Inner unhandled → outer catch | Don't over-nest, use methods`

---

<a id="q25"></a>
## Q25. Explain Throwable class and its methods?

### 📝 One-Liner
`Throwable` is the **root class** of Java's exception hierarchy — parent of both `Exception` and `Error`.

### 🔑 Quick Answer
`Throwable` is the superclass of all exceptions and errors. Key methods: (1) `printStackTrace()` — prints name, description, and stack trace. (2) `getMessage()` — returns only the description string. (3) `toString()` — returns name + description. All exceptions and errors inherit these methods. *(Throwable = exception hierarchy ka root | getMessage(), printStackTrace(), toString())*

### 💻 Code Example

```java
try {
    int x = 1/0;
} catch (ArithmeticException e) {
    e.getMessage();       // "/ by zero"
    e.toString();         // "java.lang.ArithmeticException: / by zero"
    e.printStackTrace();  // Full stack trace with line numbers
}
```

### ⚡ Remember
`Throwable = root | getMessage() = description | printStackTrace() = full trace | toString() = name + desc`

---

<a id="q26"></a>
## Q26. When is ClassNotFoundException raised?

### 📝 One-Liner
When JVM tries to load a class **by name** (using `Class.forName()`, `ClassLoader.loadClass()`) and it's **not found** on the classpath.

### 🔑 Quick Answer
ClassNotFoundException is a **checked exception** thrown when a class is loaded dynamically by name (string) and the classloader can't find the `.class` file. Common cause: misspelled class name, missing JAR from classpath. Happens at runtime, not compile time — because the class name is a string variable. *(ClassNotFoundException = Class.forName() se load karte waqt class nahi mili — classpath me nahi hai)*

### 🆚 vs.
| ClassNotFoundException | NoClassDefFoundError |
|---|---|
| Checked Exception | Error |
| Dynamic loading (`Class.forName()`) | Class was available at compile time but missing at runtime |
| Happens first time loading | Class existed, now disappeared |

### ⚡ Remember
`ClassNotFoundException = Class.forName() fail | Checked | Class not on classpath`

---

<a id="q27"></a>
## Q27. When is NoClassDefFoundError raised?

### 📝 One-Liner
When a class was **available at compile time** but the JVM **can't find** its definition at runtime.

### 🔑 Quick Answer
NoClassDefFoundError is an **Error** (not Exception). The class existed when you compiled, but at runtime the `.class` file is missing — maybe deleted, classpath changed, or dependency JAR removed. Different from ClassNotFoundException which is dynamic loading failure. *(NoClassDefFoundError = compile time pe thi, runtime pe gayab — classpath bigad gaya)*

### ⚡ Remember
`NoClassDefFoundError = was there at compile, gone at runtime | Error, not Exception`
