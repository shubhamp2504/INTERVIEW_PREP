# ☕ Java Exception Handling — Advanced & Production Patterns (Q28–Q42)

> **Audience**: 2+ years experience | Senior-level depth expected
> **Focus**: JVM internals, performance impact, microservice patterns, async exception handling, design maturity
> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q28"></a>
## Q28. Why is RuntimeException not mandatory to handle? When should you create a custom RuntimeException?

### 📝 One-Liner
RuntimeException (unchecked) represents **programming errors** (null pointer, bad index, illegal arguments) that shouldn't be caught-and-recovered — the fix is correct code, not catch blocks. Create custom ones for **business rule violations** that callers can't reasonably recover from.

### 🔑 Quick Answer
**Why unchecked**: These indicate bugs — NullPointerException means you forgot a null check, not that you should catch it. Forcing callers to handle every possible programming error would make code unreadable. **Create custom RuntimeException when**: (1) Business rule violation: `InsufficientBalanceException`. (2) Domain-specific: `OrderAlreadyCompletedException`. (3) Wrapping checked exceptions in service layers: `ServiceException(cause)`. *(RuntimeException programming bug hai — catch karne se nahi, code theek karne se fix hoti hai. Custom banao business violations ke liye)*

### 💻 Code
```java
// Custom business exception (unchecked)
public class InsufficientBalanceException extends RuntimeException {
    private final BigDecimal currentBalance;
    private final BigDecimal requestedAmount;

    public InsufficientBalanceException(BigDecimal current, BigDecimal requested) {
        super("Insufficient balance: have " + current + ", need " + requested);
        this.currentBalance = current;
        this.requestedAmount = requested;
    }
    // getters for structured error response
}

// Usage in service — no try-catch clutter in callers
public void withdraw(Long accountId, BigDecimal amount) {
    Account account = repo.findById(accountId)
        .orElseThrow(() -> new ResourceNotFoundException("Account", accountId));
    if (account.getBalance().compareTo(amount) < 0) {
        throw new InsufficientBalanceException(account.getBalance(), amount);
    }
    account.debit(amount);
}
```

### ⚡ Remember
> Unchecked = programming errors (fix the code) | Custom RuntimeException for business rules | Include context in exception (IDs, amounts) | Handle globally with `@ControllerAdvice`

---

<a id="q29"></a>
## Q29. Can a finally block be skipped? Real-world scenarios

### 📝 One-Liner
Yes — `finally` is skipped when: (1) `System.exit()` is called, (2) JVM crashes (OutOfMemoryError, StackOverflow), (3) Thread is killed (`Thread.stop()` — deprecated), (4) Infinite loop or deadlock in try/catch, (5) Daemon thread when all non-daemon threads exit.

### 🔑 Quick Answer
In normal execution, `finally` ALWAYS runs — even if try has `return`, or catch throws. The JVM guarantees this by inserting finally code before each exit path at bytecode level. **Exceptions**: `System.exit(0)` triggers JVM shutdown hooks but NOT finally. `Runtime.halt()` kills JVM immediately. `kill -9` on the process terminates forcefully. *(Finally hamesha chalta hai EXCEPT jab JVM hi band ho jaye — System.exit, crash, ya kill command)*

### 💻 Code
```java
// finally executes even with return
public int test() {
    try {
        return 1;        // return value stored temporarily
    } finally {
        System.out.println("Finally runs!"); // ✅ RUNS before return
        // return 2;     // ⚠️ Would OVERRIDE try's return! Never do this.
    }
}

// finally DOES NOT run with System.exit
try {
    System.exit(0);      // JVM shuts down
} finally {
    System.out.println("Never printed!"); // ❌ SKIPPED
}
```

### ⚡ Remember
> `finally` always runs EXCEPT: `System.exit()`, JVM crash, `kill -9`, daemon thread death | Don't return in finally (overrides try's return) | Use try-with-resources instead for cleanup

---

<a id="q30"></a>
## Q30. What happens if an exception is thrown in finally? Which exception survives?

### 📝 One-Liner
If both try/catch and finally throw exceptions, the **finally exception wins** — the original exception is **silently suppressed** (lost). In try-with-resources, the original exception survives and the close exception is added as a **suppressed exception**.

### 🔑 Quick Answer
**Traditional try-finally**: Finally exception replaces original — debugging nightmare because you lose the root cause. **try-with-resources**: Java 7+ — if `close()` throws, the original exception is preserved and the close-exception is added via `addSuppressed()`. Access with `getSuppressed()`. *(Finally mein exception aaye toh original exception kho jaata hai! try-with-resources mein original survive karta hai, close ka exception suppressed mein jaata hai)*

### 💻 Code
```java
// ❌ BAD: Original exception lost
try {
    throw new BusinessException("Payment failed"); // Original
} finally {
    throw new IOException("Close failed");          // Wins! Original LOST
}
// Only IOException propagates — BusinessException silently eaten!

// ✅ GOOD: try-with-resources preserves original
try (Connection conn = dataSource.getConnection()) {
    throw new BusinessException("Payment failed");  // Original preserved
}
// If conn.close() throws IOException:
// BusinessException propagates (primary)
// IOException added as suppressed
// catch (BusinessException e) {
//     e.getSuppressed(); // [IOException: Close failed]
// }
```

### ⚡ Remember
> Finally exception **silently kills** original exception | try-with-resources preserves original + adds suppressed | Always prefer try-with-resources for AutoCloseable | Never throw in finally manually

---

<a id="q31"></a>
## Q31. Why is exception handling expensive? How to optimize for high-performance systems?

### 📝 One-Liner
Creating an exception is expensive because the JVM captures the **entire stack trace** (`fillInStackTrace()`) by walking all stack frames — this costs ~100× more than a normal return. Optimize by avoiding exceptions for control flow and pre-validating.

### 🔑 Quick Answer
**Cost breakdown**: (1) `new Exception()` → JVM walks the call stack, creates StackTraceElement[] — O(stack depth). (2) Each frame: class name, method name, file name, line number. (3) Deep call stacks (Spring Boot = 30-50 frames) = expensive. **Optimization**: (1) Pre-validate instead of catching: `if (map.containsKey(k))` vs catch NSEE. (2) Override `fillInStackTrace()` to return `this` for known exceptions. (3) Use `Optional` instead of throwing for absent values. (4) Avoid exceptions in tight loops.

### 💻 Code
```java
// Exception creation cost
long start = System.nanoTime();
for (int i = 0; i < 1_000_000; i++) {
    new RuntimeException("test"); // ~100ns each → 100ms total
}
// vs normal object
for (int i = 0; i < 1_000_000; i++) {
    new Object(); // ~1ns each → 1ms total
}
// ~100× slower!

// Optimization: Override fillInStackTrace for known exceptions
public class LightweightValidationException extends RuntimeException {
    @Override
    public synchronized Throwable fillInStackTrace() {
        return this; // Skip stack trace — 10× faster creation
    }
}

// Optimization: Pre-validate instead of catch
// ❌ Slow (exception for control flow)
try { Integer.parseInt(input); } catch (NumberFormatException e) { /* not a number */ }
// ✅ Fast (pre-validate)
if (input.matches("-?\\d+")) { Integer.parseInt(input); }
```

### ⚡ Remember
> `fillInStackTrace()` is the bottleneck (~100× slower) | Override for high-frequency known exceptions | Never use exceptions for flow control | Pre-validate > catch | `Optional` > throwing for absent values

---

<a id="q32"></a>
## Q32. How does exception propagation work internally in the JVM?

### 📝 One-Liner
When an exception is thrown, the JVM walks the **call stack frame by frame** searching for a matching **exception handler** (catch block) in the method's exception table. If none found in current frame, it pops the frame and checks the caller.

### 🔑 Quick Answer
**JVM internals**: Each method has an **exception table** in its bytecode — entries of `[startPC, endPC, handlerPC, catchType]`. When `athrow` executes: (1) JVM checks current method's exception table for a matching range + type. (2) If found → jump to handlerPC. (3) If NOT found → pop current stack frame, repeat in caller. (4) If reaches top of stack with no handler → `Thread.getUncaughtExceptionHandler()` is called → stack trace printed → thread terminates.

### 📖 How It Works
```
Call stack when exception occurs:
  main() → service() → repository() → throw new SQLException()

JVM Exception Resolution:
  1. Check repository() exception table → no handler for SQLException
  2. Pop repository() frame
  3. Check service() exception table → has catch(SQLException) at line 25 ✅
  4. Jump to catch block handler PC
  5. Continue execution in service()

If no handler found in ANY frame:
  → Thread.UncaughtExceptionHandler
  → Default: print stack trace to stderr
  → Thread terminates (does NOT kill JVM unless it's the main thread)

Bytecode Exception Table (javap -c):
  Exception table:
    from   to  target type
      0    12    15   Class java/sql/SQLException
```

### ⚡ Remember
> Exception table per method in bytecode | Stack unwinding: check current → pop → check caller | Uncaught → UncaughtExceptionHandler | Each catch block = entry in exception table | `finally` = duplicated code in bytecode

---

<a id="q33"></a>
## Q33. Difference between Error vs Exception — why catching Error is dangerous

### 📝 One-Liner
`Error` represents **JVM-level catastrophic failures** (OutOfMemoryError, StackOverflowError) that the application **cannot recover from**. Catching them masks the real problem and can leave the JVM in an inconsistent state.

### 🔑 Quick Answer
**Exception** = application-level, recoverable (IOException, SQLException). **Error** = JVM/system-level, unrecoverable (OOM, SOF, InternalError). **Why not catch Error**: (1) OOM — catching it doesn't free memory; next allocation fails too. (2) SOF — stack is corrupted; even the catch block may overflow. (3) Masks root cause — application continues in broken state. **Exception**: Catch `OutOfMemoryError` ONLY for logging/alerting before shutdown, never for recovery.

### 🆚 vs.
| | Exception | Error |
|--|-----------|-------|
| Recoverable | ✅ Yes | ❌ No |
| Source | Application logic | JVM / system |
| Catch? | ✅ Expected | ❌ Dangerous |
| Examples | IOException, NPE | OOM, SOF, InternalError |
| Hierarchy | Throwable → Exception | Throwable → Error |

### ⚡ Remember
> **Error = JVM dying** — don't catch unless logging before exit | **Exception = app issue** — catch and handle | Both extend Throwable | `catch(Throwable t)` catches BOTH — avoid!

---

<a id="q34"></a>
## Q34. What is exception chaining and why it's critical in microservices & debugging

### 📝 One-Liner
Exception chaining wraps a low-level exception as the `cause` of a higher-level exception — preserving the **full root cause chain** for debugging while exposing a clean domain-specific exception to callers.

### 🔑 Quick Answer
**How**: `throw new ServiceException("Payment failed", originalException)` — the `cause` field (from Throwable) links them. **Why critical in microservices**: Service A catches DB error → wraps in ServiceException → Service B catches ServiceException → wraps in APIException. Without chaining, you lose the root cause at each boundary. `getCause()` traverses the chain. *(Exception chaining se root cause preserve hota hai — har layer apna exception throw kare par original cause andar rakh ke)*

### 💻 Code
```java
// ❌ BAD: Root cause lost
try {
    jdbcTemplate.update(sql);
} catch (DataAccessException e) {
    throw new PaymentException("Payment DB error"); // Root cause LOST!
}

// ✅ GOOD: Chain preserves root cause
try {
    jdbcTemplate.update(sql);
} catch (DataAccessException e) {
    throw new PaymentException("Payment DB error", e); // Chained!
}
// Log output: PaymentException: Payment DB error
//   Caused by: DataAccessException: ...
//     Caused by: SQLException: Connection refused

// Microservice chain:
// APIGateway: GatewayException → caused by PaymentServiceException
//   → caused by DataAccessException → caused by SQLException
```

### ⚡ Remember
> **Always pass original exception as cause** | `Throwable(String message, Throwable cause)` constructor | `getCause()` walks the chain | Without chaining, root cause lost at every service boundary

---

<a id="q35"></a>
## Q35. How to design global exception handling in Spring Boot — real project approach

### 📝 One-Liner
Use `@RestControllerAdvice` with `@ExceptionHandler` methods to centralize exception-to-HTTP-response mapping — return structured `ErrorResponse` DTOs with status code, message, timestamp, and path.

### 🔑 Quick Answer
**Architecture**: (1) Define `ErrorResponse` DTO (timestamp, status, message, path, errors[]). (2) Create `@RestControllerAdvice` class. (3) Handle hierarchy: `ResourceNotFoundException` → 404, `ValidationException` → 400, `BusinessException` → 409/422, `AccessDeniedException` → 403, `Exception` → 500 (catch-all). (4) Log at appropriate level (warn for 4xx, error for 5xx). (5) Never expose stack traces in production responses.

### 💻 Code
```java
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex,
                                                         HttpServletRequest request) {
        log.warn("Resource not found: {}", ex.getMessage());
        return ResponseEntity.status(404).body(ErrorResponse.builder()
            .timestamp(Instant.now())
            .status(404)
            .error("Not Found")
            .message(ex.getMessage())
            .path(request.getRequestURI())
            .build());
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<String> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> f.getField() + ": " + f.getDefaultMessage())
            .toList();
        return ResponseEntity.badRequest().body(ErrorResponse.builder()
            .status(400).error("Validation Failed").fieldErrors(errors).build());
    }

    @ExceptionHandler(Exception.class) // Catch-all
    public ResponseEntity<ErrorResponse> handleGeneric(Exception ex) {
        log.error("Unexpected error", ex); // Full stack trace in logs
        return ResponseEntity.status(500).body(ErrorResponse.builder()
            .status(500).error("Internal Server Error")
            .message("Something went wrong") // Generic — never expose internals!
            .build());
    }
}
```

### ⚡ Remember
> `@RestControllerAdvice` = centralized handler | Structured ErrorResponse DTO | 4xx → warn log, 5xx → error log | Never expose stack traces in API | Handle from specific to generic

---

<a id="q36"></a>
## Q36. Why exceptions should never be used for normal flow control

### 📝 One-Liner
Exceptions are for **exceptional conditions**, not expected outcomes — using them for flow control is 100× slower (stack trace creation), harder to read, and violates the principle of least surprise.

### 🔑 Quick Answer
**Problems**: (1) **Performance** — stack trace creation is O(stack depth), 100× slower than if-else. (2) **Readability** — catch blocks hide business logic; hard to follow flow. (3) **Debugging** — "exception noise" in logs masks real errors. (4) **Semantics** — exceptions signal something went wrong; using them for expected outcomes confuses developers.

### 💻 Code
```java
// ❌ BAD: Exception for flow control
public boolean isValidAge(String input) {
    try {
        int age = Integer.parseInt(input);
        return age > 0 && age < 150;
    } catch (NumberFormatException e) {
        return false; // Using exception as if-else!
    }
}

// ✅ GOOD: Pre-validate
public boolean isValidAge(String input) {
    if (input == null || !input.matches("\\d+")) return false;
    int age = Integer.parseInt(input);
    return age > 0 && age < 150;
}

// ❌ BAD: Looping with exception
try {
    int i = 0;
    while (true) {
        array[i++] = process(); // ArrayIndexOutOfBoundsException ends loop
    }
} catch (ArrayIndexOutOfBoundsException e) { /* done */ }

// ✅ GOOD: Normal loop
for (int i = 0; i < array.length; i++) {
    array[i] = process();
}
```

### ⚡ Remember
> Exceptions = exceptional, not expected | 100× performance cost | Use if-else, Optional, isEmpty() for expected cases | Exception noise hides real errors | "If you expect it, don't throw it"

---

<a id="q37"></a>
## Q37. try-with-resources vs finally — which is safer and why?

### 📝 One-Liner
**try-with-resources** (Java 7+) is safer: automatically closes AutoCloseable resources, handles suppressed exceptions correctly, and is less error-prone (no manual close, no null checks, no accidentally swallowing original exceptions).

### 🔑 Quick Answer
**finally problems**: (1) Must null-check resource before closing. (2) `close()` can throw → masks original exception. (3) Verbose boilerplate. (4) Easy to forget. **try-with-resources**: (1) Auto-calls `close()` in reverse declaration order. (2) Original exception preserved, close-exception added as suppressed. (3) Cleaner code. (4) Compiler-enforced.

### 💻 Code
```java
// ❌ Verbose, error-prone finally
Connection conn = null;
PreparedStatement stmt = null;
try {
    conn = dataSource.getConnection();
    stmt = conn.prepareStatement(sql);
    stmt.executeUpdate();
} catch (SQLException e) {
    throw new DataAccessException(e);
} finally {
    if (stmt != null) try { stmt.close(); } catch (SQLException e) { /* swallowed */ }
    if (conn != null) try { conn.close(); } catch (SQLException e) { /* swallowed */ }
}

// ✅ Clean, safe try-with-resources
try (Connection conn = dataSource.getConnection();
     PreparedStatement stmt = conn.prepareStatement(sql)) {
    stmt.executeUpdate();
} // auto-close in reverse order: stmt first, then conn
  // suppressed exceptions preserved!
```

### ⚡ Remember
> try-with-resources = auto-close + suppressed exceptions preserved | Must implement `AutoCloseable` | Reverse order close | Always prefer over manual finally | Java 9: effectively-final resources can be used directly

---

<a id="q38"></a>
## Q38. How to separate business exceptions vs technical exceptions in enterprise apps

### 📝 One-Liner
**Business exceptions** represent domain rule violations (insufficient balance, duplicate order) — return meaningful HTTP 4xx. **Technical exceptions** represent infrastructure failures (DB down, timeout) — return generic HTTP 5xx after logging details. Separate hierarchies prevent leaking internals.

### 🔑 Quick Answer
**Design**: Create two base exception classes — `BusinessException extends RuntimeException` (4xx, user-actionable) and `TechnicalException extends RuntimeException` (5xx, ops-actionable). Business exceptions carry domain context (error codes, messages). Technical exceptions carry infrastructure context (connection info, retry details). `@ControllerAdvice` maps each hierarchy differently.

### 💻 Code
```java
// Business exception hierarchy
public abstract class BusinessException extends RuntimeException {
    private final String errorCode;
    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
}
public class InsufficientFundsException extends BusinessException {
    public InsufficientFundsException(BigDecimal balance, BigDecimal needed) {
        super("PAY-001", "Insufficient funds: need " + needed + ", have " + balance);
    }
}

// Technical exception hierarchy
public abstract class TechnicalException extends RuntimeException {
    public TechnicalException(String message, Throwable cause) {
        super(message, cause);
    }
}
public class DatabaseUnavailableException extends TechnicalException {
    public DatabaseUnavailableException(Throwable cause) {
        super("Database connection failed", cause);
    }
}

// Handler maps differently
@ExceptionHandler(BusinessException.class)
ResponseEntity<ErrorResponse> handleBusiness(BusinessException e) {
    return ResponseEntity.status(422).body(/* errorCode + user message */);
}
@ExceptionHandler(TechnicalException.class)
ResponseEntity<ErrorResponse> handleTechnical(TechnicalException e) {
    log.error("Technical failure", e); // Full details in logs
    return ResponseEntity.status(500).body(/* generic message only */);
}
```

### ⚡ Remember
> **Business** = domain rule violation, 4xx, user sees details | **Technical** = infra failure, 5xx, generic response, full log | Never expose technical details to users | Error codes for API consumers

---

<a id="q39"></a>
## Q39. Multiple catch blocks — ordering rules & pitfalls

### 📝 One-Liner
Catch blocks must be ordered from **most specific to most general** — Java compiler rejects unreachable catch blocks. Multi-catch (`catch (A | B e)`) handles unrelated exceptions in one block without duplicating code.

### 🔑 Quick Answer
**Rule**: Subclass catch must come BEFORE superclass catch. `catch(IOException)` before `catch(Exception)`. Reverse order → compiler error: "exception already caught". **Multi-catch** (Java 7+): `catch (IOException | SQLException e)` — types can't be in same hierarchy. The reference `e` is effectively final.

### 💻 Code
```java
// ✅ Correct order: specific → general
try {
    riskyOperation();
} catch (FileNotFoundException e) {    // Most specific
    log.warn("File not found: {}", e.getMessage());
} catch (IOException e) {              // Parent of FileNotFoundException
    log.error("IO error", e);
} catch (Exception e) {                // Catch-all
    log.error("Unexpected", e);
}

// ❌ Compile error: unreachable catch
try {
    riskyOperation();
} catch (Exception e) { }              // Catches everything!
// catch (IOException e) { }           // ❌ UNREACHABLE — compiler error

// Multi-catch (Java 7+)
try {
    parse(input);
} catch (IOException | ParseException e) {
    log.error("Processing failed: {}", e.getMessage());
    // e is effectively final — can't reassign
}
```

### ⚡ Remember
> Specific → General order | Compiler catches reverse order | Multi-catch `(A | B e)` for unrelated types | `e` is effectively final in multi-catch | Don't catch Exception unless it's a catch-all at the boundary

---

<a id="q40"></a>
## Q40. Exception handling in async code — CompletableFuture, threads, executors

### 📝 One-Liner
Async exceptions don't propagate to the caller thread — in `CompletableFuture`, use `exceptionally()`, `handle()`, or `whenComplete()` to catch; in raw threads, use `Thread.UncaughtExceptionHandler` or `Future.get()` which wraps in `ExecutionException`.

### 🔑 Quick Answer
**CompletableFuture**: `exceptionally(ex → fallback)` for recovery, `handle((result, ex) → ...)` for both paths, `whenComplete()` for side effects. **ExecutorService**: `Future.get()` throws `ExecutionException` wrapping the original. **@Async (Spring)**: exceptions are lost unless you return `CompletableFuture` and the caller handles it. **Thread**: set `UncaughtExceptionHandler` to catch uncaught exceptions.

### 💻 Code
```java
// CompletableFuture — exception handling
CompletableFuture.supplyAsync(() -> {
        if (true) throw new RuntimeException("API down");
        return "data";
    })
    .exceptionally(ex -> {
        log.error("Fallback triggered: {}", ex.getMessage());
        return "cached-data"; // Recovery value
    })
    .thenAccept(result -> System.out.println(result)); // "cached-data"

// handle() — access both result and exception
CompletableFuture.supplyAsync(() -> riskyCall())
    .handle((result, ex) -> {
        if (ex != null) {
            log.error("Failed", ex);
            return defaultValue;
        }
        return result;
    });

// ExecutorService — exception wrapped in ExecutionException
Future<String> future = executor.submit(() -> { throw new IOException("fail"); });
try {
    future.get(); // Blocks, wraps IOException in ExecutionException
} catch (ExecutionException e) {
    Throwable rootCause = e.getCause(); // IOException
}

// Spring @Async — exception handling
@Async
public CompletableFuture<String> asyncMethod() {
    // Return CompletableFuture so caller can handle exceptions
    return CompletableFuture.completedFuture(process());
}
```

### ⚡ Remember
> Async exceptions DON'T propagate to caller thread! | CF: `exceptionally()` for recovery, `handle()` for both | `Future.get()` wraps in `ExecutionException` | Spring `@Async`: return `CompletableFuture`, never void | Set `UncaughtExceptionHandler` for raw threads

---

<a id="q41"></a>
## Q41. Difference between throw vs throws with use cases

### 📝 One-Liner
`throw` **creates and throws** an exception object at runtime. `throws` **declares** that a method might throw specified checked exceptions — it's a contract in the method signature for callers.

### 🔑 Quick Answer
**throw**: Used inside method body, followed by exception instance (`throw new IOException("fail")`). **throws**: Used in method signature, followed by exception class names (`void read() throws IOException`). Checked exceptions MUST be declared with `throws` or caught. Unchecked exceptions don't need `throws` (but can optionally declare for documentation).

### 🆚 vs.
| | `throw` | `throws` |
|--|---------|----------|
| Location | Method body | Method signature |
| Followed by | Exception instance | Exception class(es) |
| Purpose | Actually throw exception | Declare possibility |
| Multiple | One at a time | Comma-separated list |
| Required for | All exceptions | Checked exceptions only |

### ⚡ Remember
> `throw` = actually throw | `throws` = declare possibility | Checked → must declare or catch | Unchecked → throws optional but informative

---

<a id="q42"></a>
## Q42. Exception handling best practices — enterprise-grade checklist

### 📝 One-Liner
Never catch `Throwable`/`Error`, always chain causes, use specific exceptions, handle at the right level, log and rethrow (don't log and swallow), and prefer try-with-resources for cleanup.

### 🔑 Quick Answer
**Checklist**: (1) Catch specific exceptions, not `Exception`. (2) Always preserve root cause with chaining. (3) Log at the handler level, not at every rethrow. (4) Business exceptions = 4xx, Technical = 5xx. (5) Never swallow exceptions (empty catch). (6) Use try-with-resources for AutoCloseable. (7) Don't use exceptions for flow control. (8) Include context in exception messages (IDs, amounts). (9) Don't catch Error (OOM, SOF). (10) Use `@ControllerAdvice` for centralized API error handling.

### ⚠️ Pitfalls
| Anti-Pattern | Fix |
|--------------|-----|
| `catch(Exception e) {}` (swallow) | At minimum log, ideally rethrow |
| `catch(Throwable t)` | Only catch Exception; Error = JVM dying |
| Log AND rethrow (double logging) | Log at final handler OR rethrow, not both |
| Generic error messages | Include entity ID, operation, context |
| Stack traces in API responses | Log internally, return generic to user |

### ⚡ Remember
> Specific > generic | Chain causes always | Log once at handler | Business vs Technical separation | try-with-resources > finally | Never swallow, never catch Error | Include context in messages
