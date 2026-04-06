# 🌿 Spring — Aspect-Oriented Programming (Q6)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q6"></a>
## Q6. What is AOP (Aspect-Oriented Programming) and what are its types?

### 📝 One-Liner
AOP separates cross-cutting concerns (logging, security, transactions) from business logic — an Aspect intercepts method calls at defined Joinpoints using Pointcuts, applying Advice (Before, After, Around) without modifying the original code.

### 🔑 Quick Answer
**Problem**: logging, security checks, transaction management, and metrics repeat across hundreds of methods. Copy-pasting this code violates DRY and tangles business logic. **AOP solution**: extract these **cross-cutting concerns** into **Aspects** — self-contained modules that automatically intercept method execution. **Key concepts**: **Aspect** = the module (class with `@Aspect`). **Joinpoint** = the method being intercepted. **Pointcut** = expression defining WHICH methods to intercept. **Advice** = WHAT to do and WHEN: `@Before` (before method), `@After` (after method), `@AfterReturning` (on success), `@AfterThrowing` (on exception), `@Around` (wraps entire method — most powerful). Spring AOP uses **runtime proxies** (CGLIB/JDK dynamic proxy) — the proxy intercepts the call, runs advice, then delegates to the actual method. *(AOP = cross-cutting logic ko alag class mein nikalo — original code change nahi hota)*

### 📖 How It Works
```
Without AOP (tangled concerns):
  public Order createOrder(OrderDTO dto) {
      log.info("Creating order...");              // ← logging (repeated everywhere)
      securityService.checkPermission("ORDER");   // ← security (repeated everywhere)
      long start = System.nanoTime();             // ← metrics (repeated everywhere)
      
      Order order = orderRepo.save(toEntity(dto)); // ← actual business logic
      
      long time = System.nanoTime() - start;      // ← metrics
      log.info("Order created in {}ms", time/1e6); // ← logging
      return order;
  }

With AOP (clean separation):
  public Order createOrder(OrderDTO dto) {
      return orderRepo.save(toEntity(dto));  // ← ONLY business logic!
  }
  // Logging, security, metrics handled by Aspects automatically

How Spring AOP Works (proxy-based):
  Client → Proxy → [Before Advice] → Actual Method → [After Advice] → Response
  
  @Service
  OrderService (bean) → Spring creates CGLIB Proxy at startup
  
  Call: orderService.createOrder(dto)
        ↓
  CGLIB Proxy intercepts:
    1. LoggingAspect.@Before  → logs method entry
    2. SecurityAspect.@Before → checks permission
    3. ─── actual createOrder() executes ───
    4. MetricsAspect.@Around  → records execution time
    5. LoggingAspect.@After   → logs method exit
        ↓
  Response returned to caller

AOP Terminology:
  ┌────────────────────────────────────────────────────────────┐
  │ Aspect     = @Aspect class (LoggingAspect, SecurityAspect)│
  │ Joinpoint  = method execution point being intercepted      │
  │ Pointcut   = expression selecting which joinpoints         │
  │              e.g., "execution(* com.app.service.*.*(..))" │
  │ Advice     = code to execute (Before, After, Around, etc.) │
  │ Weaving    = process of applying aspects to target objects │
  │              Spring AOP = runtime weaving (proxies)        │
  │              AspectJ    = compile-time / load-time weaving │
  └────────────────────────────────────────────────────────────┘
```

### 🗣️ Answering Approach
"AOP lets us separate cross-cutting concerns — like logging, transaction management, and security — from business logic. Instead of calling a logger in every service method, I create a LoggingAspect with a pointcut expression that targets all service-layer methods, and Spring automatically applies it. Spring AOP works through proxies — when the application starts, Spring creates CGLIB proxies around beans that match pointcut expressions. When a method is called, the proxy intercepts it and runs the advice before, after, or around the actual method. The five advice types are @Before, @After, @AfterReturning, @AfterThrowing, and @Around — where Around is the most powerful since it wraps the entire method and can modify arguments, return values, or even skip execution. @Transactional is actually implemented as an Around advice internally. In my project, I used AOP for centralized audit logging — every service method's execution time and user identity was logged without touching any business code."

### 💻 Code
```java
// 1. LOGGING ASPECT — log every service method entry/exit
@Aspect
@Component
@Slf4j
public class LoggingAspect {
    
    // Pointcut: all methods in any class under com.app.service package
    @Pointcut("execution(* com.app.service.*.*(..))")
    public void serviceMethods() {}
    
    @Before("serviceMethods()")
    public void logBefore(JoinPoint jp) {
        log.info("→ {}.{}() called with args: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(),
            Arrays.toString(jp.getArgs()));
    }
    
    @AfterReturning(pointcut = "serviceMethods()", returning = "result")
    public void logAfter(JoinPoint jp, Object result) {
        log.info("← {}.{}() returned: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(), result);
    }
    
    @AfterThrowing(pointcut = "serviceMethods()", throwing = "ex")
    public void logError(JoinPoint jp, Exception ex) {
        log.error("✖ {}.{}() threw: {}",
            jp.getTarget().getClass().getSimpleName(),
            jp.getSignature().getName(), ex.getMessage());
    }
}

// 2. PERFORMANCE ASPECT — @Around wraps entire method
@Aspect
@Component
@Slf4j
public class PerformanceAspect {
    
    @Around("@annotation(com.app.annotation.TrackExecutionTime)")
    public Object trackTime(ProceedingJoinPoint pjp) throws Throwable {
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();  // ← actual method execution
            return result;
        } finally {
            long timeMs = (System.nanoTime() - start) / 1_000_000;
            log.info("⏱ {}.{}() took {}ms",
                pjp.getTarget().getClass().getSimpleName(),
                pjp.getSignature().getName(), timeMs);
        }
    }
}

// Custom annotation for selective AOP
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface TrackExecutionTime {}

// Usage — aspect triggers only on annotated methods
@Service
public class OrderService {
    @TrackExecutionTime  // ← aspect will measure this method
    public Order createOrder(OrderDTO dto) {
        return orderRepo.save(toEntity(dto));
    }
}

// 3. SECURITY ASPECT — check authorization before method
@Aspect
@Component
public class AuthorizationAspect {
    
    @Before("@annotation(requiresRole)")
    public void checkRole(JoinPoint jp, RequiresRole requiresRole) {
        String currentRole = SecurityContextHolder.getContext()
                .getAuthentication().getAuthorities().toString();
        if (!currentRole.contains(requiresRole.value())) {
            throw new AccessDeniedException("Required role: " + requiresRole.value());
        }
    }
}

// 4. COMMON POINTCUT EXPRESSIONS
@Pointcut("execution(* com.app.service.*.*(..))")        // all service methods
@Pointcut("execution(public * *(..))")                    // all public methods
@Pointcut("within(com.app.controller..*)")                // all classes in controller pkg
@Pointcut("@annotation(org.springframework.transaction.annotation.Transactional)")  // @Transactional methods
@Pointcut("bean(*Service)")                               // beans ending with Service
@Pointcut("@within(org.springframework.stereotype.Service)")  // classes annotated @Service
```

### ⚠️ Pitfalls / Gotchas
- **Self-invocation bypasses AOP** — calling `this.method()` from within the same class skips the proxy. Same problem as @Transactional *(Same class ke andar method call kiya toh AOP apply nahi hoga — proxy bypass ho jaata hai)*
- **Spring AOP only intercepts Spring-managed beans** — `new MyClass()` won't have aspects. Use AspectJ for non-Spring objects
- **@Around must call `pjp.proceed()`** — forgetting it means the actual method never executes
- **Pointcut too broad** (e.g., `execution(* *.*(..))`) → intercepts everything including framework internals → performance disaster
- **Advice ordering** when multiple aspects: use `@Order(1)`, `@Order(2)` to control sequence
- **@Around swallowing exceptions** — if you catch and don't rethrow, errors are silently lost

### 🆚 vs. Comparison

**Advice Types:**

| Advice | When it runs | Access to | Use case |
|--------|-------------|-----------|----------|
| `@Before` | Before method | Arguments | Validation, auth, logging entry |
| `@After` | After method (always) | — | Cleanup (like finally) |
| `@AfterReturning` | After success | Return value | Logging result, caching |
| `@AfterThrowing` | After exception | Exception | Error logging, alerting |
| `@Around` | Wraps entire method ⭐ | Everything | Timing, transactions, retry |

**Spring AOP vs AspectJ:**

| Aspect | Spring AOP | AspectJ |
|--------|-----------|---------|
| Weaving | Runtime (proxy) | Compile/Load time |
| Performance | Slight overhead | No overhead (compiled in) |
| Scope | Spring beans only | Any Java class |
| Joinpoints | Method execution only | Fields, constructors, etc. |
| Complexity | Simple ⭐ | Complex (needs ajc compiler) |
| Self-invocation | ❌ Doesn't work | ✅ Works |

### 🎯 Tricky Interview Qs

**Q: How does @Transactional use AOP internally?**
`@Transactional` is an Around advice. The `TransactionInterceptor` (a MethodInterceptor) wraps the method: it begins a transaction before `proceed()`, commits on success, and rolls back on RuntimeException. This is why self-invocation breaks @Transactional — same proxy bypass issue. *(Transaction internally Around advice hai — proxy lagata hai method ke aas-paas)*

**Q: Can you apply AOP to private methods?**
Spring AOP cannot — proxies only intercept public/protected methods on Spring beans. AspectJ (compile-time weaving) can intercept private methods, constructors, and field access.

**Q: What is the difference between JDK Dynamic Proxy and CGLIB Proxy?**
JDK proxy requires an interface — creates a proxy implementing the same interface. CGLIB creates a subclass of the target class — works without interfaces. Spring Boot defaults to CGLIB proxy.

### ⚡ Remember
- **AOP** = separate cross-cutting concerns (logging, security, tx) from business logic
- **5 Advice types**: Before, After, AfterReturning, AfterThrowing, **Around** (most powerful)
- **Pointcut** = expression selecting which methods to intercept
- Spring AOP = **runtime proxies** (CGLIB) → only Spring beans, only method execution
- **Self-invocation** bypasses AOP (same as @Transactional issue) *(this.method() se proxy bypass hota hai)*
- @Transactional = AOP Around advice internally

### 🔗 Follow-ups
- Q8 → @Transactional internals (spring/01, uses AOP)
- Q10 → Global exception handling (spring/01, alternative to AOP)
- [Q7 → Caching with AOP (@Cacheable is AOP-based)](#q7)
