# 🌱 Spring Framework Internals (Q6–Q10)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q6"></a>
## Q6. Describe the complete lifecycle of a Spring Bean.

### 📝 One-Liner
Spring Bean lifecycle: Instantiate → Populate properties (DI) → BeanNameAware/BeanFactoryAware → BeanPostProcessor.before → @PostConstruct/InitializingBean → BeanPostProcessor.after → Ready → @PreDestroy/DisposableBean → Destroyed.

### 🔑 Quick Answer
The lifecycle has **8 key phases**: **(1) Instantiation** — constructor called. **(2) Dependency Injection** — @Autowired fields/setters populated. **(3) Aware interfaces** — BeanNameAware, BeanFactoryAware, ApplicationContextAware called (if implemented). **(4) BeanPostProcessor.postProcessBeforeInitialization** — BEFORE custom init (AOP proxies, validation). **(5) Init callbacks** — @PostConstruct, then InitializingBean.afterPropertiesSet(), then custom init-method. **(6) BeanPostProcessor.postProcessAfterInitialization** — AFTER custom init (where AOP proxies are typically created). **(7) Bean is ready** — used by application. **(8) Destruction** — @PreDestroy, DisposableBean.destroy(), custom destroy-method. *(Banao → DI → Aware → Before → Init → After → Use → Destroy)*

### 📖 How It Works
```
Spring Bean Lifecycle (complete flow):

  ┌──────────────────────────────────────────────────────┐
  │ 1. INSTANTIATION                                      │
  │    new MyService() via constructor (or factory method)│
  ├──────────────────────────────────────────────────────┤
  │ 2. DEPENDENCY INJECTION                               │
  │    @Autowired fields, setter, constructor injection   │
  ├──────────────────────────────────────────────────────┤
  │ 3. AWARE CALLBACKS (if implemented)                  │
  │    BeanNameAware.setBeanName("myService")            │
  │    BeanFactoryAware.setBeanFactory(factory)          │
  │    ApplicationContextAware.setApplicationContext(ctx) │
  ├──────────────────────────────────────────────────────┤
  │ 4. BeanPostProcessor.postProcessBeforeInitialization │
  │    → @PostConstruct detection happens here            │
  │    → Custom processing BEFORE init                    │
  ├──────────────────────────────────────────────────────┤
  │ 5. INITIALIZATION CALLBACKS (in this order)          │
  │    a. @PostConstruct method                           │
  │    b. InitializingBean.afterPropertiesSet()          │
  │    c. Custom init-method (from @Bean(initMethod=...))│
  ├──────────────────────────────────────────────────────┤
  │ 6. BeanPostProcessor.postProcessAfterInitialization  │
  │    → AOP proxies created here (wraps the bean)       │
  │    → @Transactional, @Cacheable proxies              │
  ├──────────────────────────────────────────────────────┤
  │ 7. BEAN READY — used by application                  │
  ├──────────────────────────────────────────────────────┤
  │ 8. DESTRUCTION (on context shutdown)                 │
  │    a. @PreDestroy method                              │
  │    b. DisposableBean.destroy()                       │
  │    c. Custom destroy-method                          │
  └──────────────────────────────────────────────────────┘
```

### 🗣️ How to Say in Interview
"The Spring Bean lifecycle starts with instantiation via the constructor, then dependency injection populates all @Autowired fields and setters. Next, any Aware interfaces are called — like ApplicationContextAware. Then BeanPostProcessors run their 'before' method — this is where annotations like @PostConstruct are detected. The init callbacks run in order: @PostConstruct, then InitializingBean if implemented, then any custom init-method. After init, BeanPostProcessors run their 'after' method — this is where AOP proxies for @Transactional and @Cacheable are created, wrapping the original bean. The bean is then ready for use. On shutdown, @PreDestroy runs, then DisposableBean, then custom destroy-method. In my project, I use @PostConstruct for loading initial cache data and @PreDestroy for graceful connection cleanup."

### 💻 Code
```java
@Service
public class OrderService implements InitializingBean, DisposableBean,
        BeanNameAware, ApplicationContextAware {
    
    @Autowired
    private OrderRepository repository;  // Step 2: DI
    
    private String beanName;
    private ApplicationContext context;

    // Step 3: Aware callbacks
    @Override
    public void setBeanName(String name) { this.beanName = name; }
    @Override
    public void setApplicationContext(ApplicationContext ctx) { this.context = ctx; }
    
    // Step 5a: @PostConstruct (most commonly used)
    @PostConstruct
    public void init() {
        log.info("Bean '{}' initialized. Loading cache...", beanName);
        loadCacheFromDb();
    }
    
    // Step 5b: InitializingBean
    @Override
    public void afterPropertiesSet() {
        log.info("afterPropertiesSet called (after @PostConstruct)");
    }
    
    // Step 8a: @PreDestroy
    @PreDestroy
    public void cleanup() {
        log.info("Shutting down. Flushing pending operations...");
        flushPendingWrites();
    }
    
    // Step 8b: DisposableBean
    @Override
    public void destroy() {
        log.info("DisposableBean.destroy called (after @PreDestroy)");
    }
}

// Custom BeanPostProcessor (framework-level, not typical app code)
@Component
public class LoggingBeanPostProcessor implements BeanPostProcessor {
    @Override
    public Object postProcessBeforeInitialization(Object bean, String beanName) {
        // Step 4: runs BEFORE @PostConstruct
        return bean;
    }
    @Override
    public Object postProcessAfterInitialization(Object bean, String beanName) {
        // Step 6: runs AFTER init — AOP proxies created here
        return bean;  // or return a proxy wrapping the bean
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **@PostConstruct** runs AFTER DI → all dependencies available. Constructors run BEFORE DI → dependencies are null *(constructor mein @Autowired fields null honge — @PostConstruct mein available)*
- **AOP proxy** created in step 6 → if you store `this` reference in @PostConstruct, it's the raw bean not the proxy → @Transactional won't work on self-calls
- **Prototype beans** don't get destroy callbacks — Spring doesn't manage their lifecycle after creation
- **@PostConstruct** is from Jakarta EE (javax.annotation) — needs dependency in some setups
- **Order**: @PostConstruct → InitializingBean → custom init-method (if all three exist)

### 🎯 Tricky Interview Qs

**Q: When are AOP proxies created in the lifecycle?**
In `postProcessAfterInitialization` (Step 6) — after all init callbacks. So @PostConstruct sees the RAW bean, not the proxy. This is why self-invocation bypasses AOP. *(AOP proxy init ke BAAD banta hai — @PostConstruct mein raw bean hota hai)*

**Q: What's the difference between @PostConstruct and constructor?**
Constructor runs BEFORE DI — @Autowired fields are null. @PostConstruct runs AFTER DI — all dependencies are injected and available.

### ⚡ Remember
- Lifecycle: **Instantiate → DI → Aware → BPP.before → Init → BPP.after → Ready → Destroy**
- @PostConstruct = most common init hook (runs after DI)
- AOP proxies created in BPP.after (step 6)
- Destroy order: @PreDestroy → DisposableBean → custom destroy
- Prototype beans: no destroy callback *(prototype ka destruction Spring handle nahi karta)*

### 🔗 Follow-ups
- [Q7 → Dependency Injection internals](#q7)
- [Q8 → @Transactional proxy (created in step 6)](#q8)
- [Q9 → @Component vs @Service vs @Repository](#q9)

---

<a id="q7"></a>
## Q7. How does Dependency Injection function internally in Spring?

### 📝 One-Liner
Spring DI scans for beans (@Component), stores definitions in BeanFactory, resolves dependencies by type/name, and injects via constructor, setter, or field reflection — all during ApplicationContext startup.

### 🔑 Quick Answer
DI happens in three phases: **(1) Bean definition loading** — @ComponentScan finds classes annotated with @Component/@Service/@Repository/@Controller, @Configuration/@Bean methods, and XML definitions. Each becomes a `BeanDefinition` stored in `BeanDefinitionRegistry`. **(2) Dependency resolution** — Spring builds a dependency graph. For each bean, it resolves @Autowired dependencies **by type** first, then **by qualifier/name** if ambiguous. Circular dependencies are handled with early reference exposure (for singletons only). **(3) Injection** — **Constructor injection** (preferred): dependencies passed as constructor args. **Field injection**: uses `java.lang.reflect.Field.set()`. **Setter injection**: calls setter method. *(Bean definitions load karo → dependency graph banao → inject karo)*

### 📖 How It Works
```
DI Internal Flow (ApplicationContext startup):

Phase 1: SCAN & REGISTER
  @ComponentScan("com.myapp")
    ↓
  ClassPathBeanDefinitionScanner scans packages
    ↓
  Finds: OrderService, OrderRepository, PaymentGateway
    ↓
  Creates BeanDefinition for each:
    {name: "orderService", class: OrderService, scope: singleton,
     dependencies: [OrderRepository, PaymentGateway]}
    ↓
  Stored in BeanDefinitionRegistry (part of BeanFactory)

Phase 2: RESOLVE DEPENDENCIES
  Building dependency graph:
    OrderService → needs OrderRepository, PaymentGateway
    OrderRepository → needs DataSource
    PaymentGateway → needs RestTemplate
    DataSource → needs properties
    RestTemplate → no deps

  Creation order (topological sort):
    properties → DataSource → RestTemplate → OrderRepository
    → PaymentGateway → OrderService

Phase 3: CREATE & INJECT
  For each bean (in dependency order):
    1. Instantiate (call constructor)
    2. Resolve @Autowired:
       - By TYPE first (find bean of matching class)
       - If multiple matches → by @Qualifier or field name
       - If still ambiguous → exception
    3. Inject:
       - Constructor: pass as args (preferred)
       - Field: Field.setAccessible(true); field.set(bean, dep)
       - Setter: invoke setter method

Type Resolution:
  @Autowired OrderRepository repo;
  → finds beans of type OrderRepository → 1 match → inject
  → if 2 matches (OrderRepoImpl, MockOrderRepo):
     → check @Qualifier → check field name → ambiguous? error!
```

### 🗣️ How to Say in Interview
"Spring's DI works in three phases during ApplicationContext startup. First, it scans the classpath using @ComponentScan, finding all annotated classes and @Bean methods, creating BeanDefinitions stored in the registry. Second, it resolves the dependency graph — topologically sorting beans so what's needed first gets created first. Third, for each bean, it instantiates via constructor and injects dependencies. For @Autowired, it resolves by type first — looking up beans assignable to the field type. If there are multiple candidates, it falls back to @Qualifier then field name. Constructor injection is preferred because it makes dependencies explicit, supports immutability via final fields, and fails fast if a dependency is missing. In my project, I exclusively use constructor injection — generated by Lombok's @RequiredArgsConstructor."

### 💻 Code
```java
// Constructor injection (PREFERRED — immutable, testable, fail-fast)
@Service
@RequiredArgsConstructor  // Lombok generates constructor
public class OrderService {
    private final OrderRepository repository;      // injected via constructor
    private final PaymentGateway paymentGateway;   // injected via constructor
    // no @Autowired needed when single constructor
    
    public Order processOrder(OrderRequest request) {
        Order order = repository.save(new Order(request));
        paymentGateway.charge(order.getAmount());
        return order;
    }
}

// Field injection (simple but NOT recommended)
@Service
public class OrderService {
    @Autowired private OrderRepository repository;     // reflection injection
    @Autowired private PaymentGateway paymentGateway;  // can't be final
    // Harder to test, hidden dependencies
}

// Resolving ambiguity
public interface PaymentGateway { void charge(double amount); }

@Service("stripe")
public class StripeGateway implements PaymentGateway { ... }

@Service("paypal")
public class PayPalGateway implements PaymentGateway { ... }

@Service
public class OrderService {
    private final PaymentGateway gateway;
    
    // Option 1: @Qualifier
    public OrderService(@Qualifier("stripe") PaymentGateway gateway) {
        this.gateway = gateway;
    }
    
    // Option 2: @Primary on the bean
    // @Primary @Service("stripe") public class StripeGateway ...
}

// How Spring resolves internally (simplified)
// container.getBean(OrderRepository.class)
//   → scans all BeanDefinitions → finds type match → returns singleton
```

### ⚠️ Pitfalls / Gotchas
- **Circular dependency**: A needs B, B needs A → works with field/setter injection (early reference), FAILS with constructor injection *(constructor injection mein circular dependency crash karti hai)*
- **Field injection** hides dependencies → class can accumulate too many without notice
- **@Autowired(required=false)** → dependency is null if not found (silent failure)
- **Prototype in Singleton** problem: singleton gets ONE prototype instance, not a new one each time → use `ObjectFactory<T>` or `@Lookup`
- **BeanNotOfRequiredTypeException** → you have a proxy (CGLIB) but expected the interface

### 🆚 vs. Comparison
| Injection Type | Pros | Cons |
|---------------|------|------|
| Constructor ⭐ | Immutable (final), fail-fast, testable | Verbose (Lombok fixes this) |
| Field | Concise | Hidden deps, can't be final, reflection |
| Setter | Optional deps | Mutable, order-dependent |

### 🎯 Tricky Interview Qs

**Q: How does Spring handle circular dependencies?**
For singleton beans with field/setter injection: Spring uses "three-level cache" — it exposes an early reference (partially constructed bean) to break the cycle. For constructor injection: impossible — fails with `BeanCurrentlyInCreationException`. *(Constructor circular = fail, field circular = early reference se solve)*

**Q: What happens if two beans match the same type?**
`NoUniqueBeanDefinitionException`. Fix with @Qualifier, @Primary, or match field name to bean name.

### ⚡ Remember
- **Three phases**: scan → resolve dependency graph → inject
- **By type first**, then @Qualifier, then name
- **Constructor injection** = best practice (final, testable, fail-fast) *(constructor = sabse best — final fields, easy testing)*
- Circular dependency: works with field injection, fails with constructor
- @RequiredArgsConstructor (Lombok) = clean constructor injection

### 🔗 Follow-ups
- [Q6 → Bean lifecycle (when DI happens)](#q6)
- [Q8 → @Transactional (proxy injection)](#q8)
- [Q9 → @Component vs @Service vs @Repository](#q9)

---

<a id="q8"></a>
## Q8. What happens internally when you use @Transactional?

### 📝 One-Liner
Spring creates a CGLIB/JDK proxy around your bean; on method call, the proxy intercepts → opens transaction → calls your method → commits on success / rolls back on RuntimeException.

### 🔑 Quick Answer
@Transactional uses **AOP proxy pattern**. At startup (BeanPostProcessor.after), Spring creates a **proxy** wrapping your bean: CGLIB proxy (for classes) or JDK dynamic proxy (for interfaces). When a @Transactional method is called, the **proxy interceptor** (TransactionInterceptor): **(1)** Gets a PlatformTransactionManager. **(2)** Creates or reuses a transaction (based on propagation). **(3)** Calls your actual method. **(4)** On success → commits. **(5)** On RuntimeException/Error → rolls back. On checked exception → commits (default). The proxy sets the transaction on the current thread via `ThreadLocal`, so all DB operations in that thread share the same connection/transaction. *(Proxy method ko intercept karta hai — pehle transaction shuru karta hai, phir tumhara method chalata hai, phir commit/rollback)*

### 📖 How It Works
```
@Transactional Internal Flow:

STARTUP (Bean creation):
  OrderService bean created
    ↓
  BeanPostProcessor.postProcessAfterInitialization
    ↓
  Detects @Transactional → creates CGLIB proxy
    ↓
  Container stores PROXY (not original bean)
  Other beans get the PROXY injected

RUNTIME (method call):
  controller.createOrder(request)
    ↓
  orderService.createOrder(request)  ← THIS IS THE PROXY!
    ↓
  TransactionInterceptor.invoke():
    1. txManager.getTransaction(txDefinition)
       → Get connection from pool
       → connection.setAutoCommit(false)
       → Bind to ThreadLocal (TransactionSynchronizationManager)
    2. try {
         target.createOrder(request)    ← YOUR actual method
         // All DB calls use same connection (from ThreadLocal)
         // repository.save() → uses this transaction's connection
       }
    3. Success? → txManager.commit(txStatus)
                   → connection.commit()
                   → connection.setAutoCommit(true)
                   → return connection to pool
    4. RuntimeException? → txManager.rollback(txStatus)
                           → connection.rollback()
    5. Checked exception? → txManager.commit() ← DEFAULT! (commits!)

SELF-INVOCATION PROBLEM:
  @Service
  class OrderService {
    @Transactional
    public void methodA() {
      methodB();  // ← Direct call, BYPASSES proxy!
    }
    @Transactional(propagation = REQUIRES_NEW)
    public void methodB() { ... }
  }
  // methodB's REQUIRES_NEW is IGNORED because proxy is bypassed
```

### 🗣️ How to Say in Interview
"When you annotate a method with @Transactional, Spring creates a proxy around the bean during application startup. The proxy is created in the BeanPostProcessor's postProcessAfterInitialization phase. At runtime, when a @Transactional method is called, the proxy's TransactionInterceptor kicks in — it obtains a database connection from the pool, sets autoCommit to false, and binds the connection to a ThreadLocal. Your method then executes, and all repository calls within it automatically use this same connection. On success, the proxy commits. On any RuntimeException or Error, it rolls back. Importantly, checked exceptions commit by default — you need `rollbackFor = Exception.class` to change that. The biggest gotcha is self-invocation: calling a @Transactional method from within the same class bypasses the proxy."

### 💻 Code
```java
// Basic @Transactional usage
@Service
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentService paymentService;
    private final NotificationService notificationService;
    
    @Transactional  // proxy intercepts this call
    public Order createOrder(OrderRequest request) {
        Order order = orderRepo.save(new Order(request));        // uses tx connection
        paymentService.charge(order);                             // uses same tx
        // If charge() throws RuntimeException → EVERYTHING rolls back
        return order;
    }
    
    @Transactional(readOnly = true)  // optimization: no dirty checks
    public Order getOrder(Long id) {
        return orderRepo.findById(id).orElseThrow();
    }
    
    @Transactional(
        propagation = Propagation.REQUIRED,         // join existing or create new (default)
        isolation = Isolation.READ_COMMITTED,         // default for most DBs
        timeout = 30,                                  // seconds
        rollbackFor = Exception.class                  // roll back on checked exceptions too
    )
    public void processPayment(Long orderId) throws PaymentException {
        // With rollbackFor=Exception.class, even checked exceptions roll back
    }
}

// ❌ Self-invocation problem
@Service
public class ReportService {
    public void generateAll() {
        generateDaily();   // ❌ Direct call, @Transactional IGNORED!
    }
    
    @Transactional
    public void generateDaily() { /* ... */ }
}

// ✅ Fix 1: Inject self (proxy)
@Service
public class ReportService {
    @Autowired private ReportService self;  // injects the PROXY
    
    public void generateAll() {
        self.generateDaily();  // ✅ Goes through proxy → @Transactional works
    }
    
    @Transactional
    public void generateDaily() { /* ... */ }
}

// ✅ Fix 2: Separate into two services
@Service
public class ReportOrchestrator {
    @Autowired private ReportService reportService;
    
    public void generateAll() {
        reportService.generateDaily();  // ✅ Goes through proxy
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Self-invocation** bypasses proxy → @Transactional ignored *(same class ke andar call karo toh proxy skip ho jaata hai)*
- **Checked exceptions** commit by default! Use `rollbackFor = Exception.class` for full rollback coverage
- **private methods**: @Transactional on private methods is IGNORED (proxy can't intercept)
- **readOnly=true** doesn't prevent writes — it's a hint for optimization (no dirty checking, slave DB routing)
- **Propagation.REQUIRES_NEW** suspends outer transaction → uses NEW connection → can cause pool exhaustion

### 🆚 vs. Comparison
| Propagation | Behavior |
|-------------|----------|
| REQUIRED (default) | Join existing tx, or create new if none |
| REQUIRES_NEW | Always create new tx (suspend existing) |
| NESTED | Savepoint within existing tx |
| SUPPORTS | Use tx if exists, else run without |
| NOT_SUPPORTED | Suspend existing tx, run without |
| MANDATORY | Must have existing tx, else exception |
| NEVER | Must NOT have tx, else exception |

### 🎯 Tricky Interview Qs

**Q: Why does @Transactional not work on private methods?**
Because Spring uses proxies (CGLIB/JDK). Proxies override public/protected methods to add transaction logic. Private methods aren't visible to the proxy class. *(Private method proxy ko dikhta nahi — override nahi kar sakta)*

**Q: What happens with REQUIRES_NEW inside a REQUIRED method?**
Outer transaction is SUSPENDED. A new independent transaction starts. If inner commits but outer fails, inner's changes are NOT rolled back (separate tx). Both use separate DB connections.

### ⚡ Remember
- **Proxy** intercepts → start tx → call method → commit/rollback
- Self-invocation = proxy bypassed = @Transactional ignored *(same class call = proxy skip)*
- Checked exception = **commit** (default!) → use `rollbackFor`
- Private methods: @Transactional ignored
- CGLIB proxy for classes, JDK proxy for interfaces

### 🔗 Follow-ups
- [Q6 → Bean lifecycle (proxy created in step 6)](#q6)
- [Q13 → ACID properties](#q13)
- [Q12 → Concurrent database updates (isolation levels)](#q12)

---

<a id="q9"></a>
## Q9. What is the difference between @Component, @Service, and @Repository?

### 📝 One-Liner
Functionally the same (@Service and @Repository are just specializations of @Component) — but @Repository adds automatic exception translation for database exceptions, and all three convey semantic intent.

### 🔑 Quick Answer
All three are **stereotype annotations** that make a class a Spring bean (auto-detected by @ComponentScan). **@Component** = generic bean. **@Service** = business logic layer (no extra behavior, purely semantic). **@Repository** = data access layer — adds **automatic exception translation**: vendor-specific DB exceptions (SQLIntegrityConstraintViolationException) are translated to Spring's `DataAccessException` hierarchy (DataIntegrityViolationException). **@Controller/@RestController** = web layer. The naming conveys **architectural layer intent** and enables layer-specific processing. *(Teeno Spring bean banate hain — @Repository extra kaam karta hai: DB exceptions ko translate karta hai)*

### 📖 How It Works
```
Annotation Hierarchy:
  @Component  ← base annotation (generic bean)
    ├── @Service     ← semantic: business logic layer
    ├── @Repository  ← semantic + exception translation
    └── @Controller  ← semantic + request mapping

Layer Architecture:
  @RestController  ←  Web/API layer (handles HTTP)
       ↓
  @Service         ←  Business logic layer (orchestration)
       ↓
  @Repository      ←  Data access layer (DB operations)

@Repository Special Behavior:
  Without @Repository:
    orderRepo.save(order) → Hibernate throws
    org.hibernate.exception.ConstraintViolationException
    → Your code must catch Hibernate-specific exceptions
    → Tight coupling to Hibernate!

  With @Repository:
    orderRepo.save(order) → Spring catches Hibernate exception
    → Translates to Spring's DataIntegrityViolationException
    → Your code catches Spring exceptions only
    → DB vendor independent!

  How: PersistenceExceptionTranslationPostProcessor
    (a BeanPostProcessor) wraps @Repository beans in a proxy
    that catches and translates persistence exceptions.
```

### 🗣️ How to Say in Interview
"All three are specializations of @Component and register the class as a Spring bean. The difference is semantic intent and specific behavior. @Service indicates business logic — it has no extra behavior beyond what @Component provides. @Repository indicates the data access layer and adds automatic exception translation — a PersistenceExceptionTranslationPostProcessor wraps the bean in a proxy that converts vendor-specific database exceptions into Spring's DataAccessException hierarchy, making your code database-vendor independent. In my project, I strictly follow the layered convention: @RestController for API endpoints, @Service for business logic, @Repository for data access. It makes the codebase navigable — you can instantly tell what layer a class belongs to."

### 💻 Code
```java
// @Component — generic, for classes that don't fit a specific layer
@Component
public class EmailValidator {
    public boolean isValid(String email) {
        return email != null && email.contains("@");
    }
}

// @Service — business logic layer (no extra behavior vs @Component)
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final PaymentGateway paymentGateway;
    
    @Transactional
    public Order placeOrder(OrderRequest request) {
        Order order = new Order(request);
        orderRepo.save(order);
        paymentGateway.charge(order.getAmount());
        return order;
    }
}

// @Repository — data access layer (exception translation!)
@Repository
@RequiredArgsConstructor
public class OrderRepositoryImpl implements OrderRepository {
    private final JdbcTemplate jdbc;
    
    public Order save(Order order) {
        // If this throws SQLIntegrityConstraintViolationException,
        // Spring translates it to DataIntegrityViolationException
        jdbc.update("INSERT INTO orders (id, amount) VALUES (?, ?)",
                order.getId(), order.getAmount());
        return order;
    }
}

// Catching translated exceptions in service layer
@Service
public class UserService {
    @Autowired private UserRepository userRepo;
    
    public User register(User user) {
        try {
            return userRepo.save(user);
        } catch (DataIntegrityViolationException e) {
            // Spring exception — not Hibernate/JDBC specific!
            throw new DuplicateEmailException("Email already registered");
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- Using @Component for everything "works" but loses semantic meaning and @Repository's exception translation *(sab jagah @Component lagane se kaam chalega, lekin architecture unclear hoga)*
- Spring Data JPA interfaces DON'T need @Repository — it's auto-applied by Spring Data's infrastructure
- @Service is purely semantic — swapping it with @Component changes nothing functionally
- Don't put @Transactional on @Repository — put it on @Service (business boundary)
- Custom exception translation: implement `PersistenceExceptionTranslator`

### 🆚 vs. Comparison
| Annotation | Layer | Extra Behavior | Use For |
|-----------|-------|---------------|---------|
| @Component | Generic | None | Utilities, validators, converters |
| @Service | Business | None (semantic only) | Business logic, orchestration |
| @Repository | Data Access | Exception translation ⭐ | DAOs, data access code |
| @Controller | Web | Request mapping | MVC controllers |
| @RestController | Web API | Request mapping + @ResponseBody | REST endpoints |
| @Configuration | Config | Proxy for @Bean methods | Bean configuration |

### ⚡ Remember
- All three = Spring beans via @ComponentScan *(teeno bean banate hain)*
- @Service = semantic only (business logic)
- **@Repository = exception translation** (DB exception → Spring exception) *(Repository extra kaam karta hai — exceptions translate)*
- @Controller = request mapping
- Follow layer conventions for clean architecture

### 🔗 Follow-ups
- [Q7 → DI internals (how beans are scanned)](#q7)
- [Q10 → Global exception handling](#q10)
- [Q6 → Bean lifecycle](#q6)

---

<a id="q10"></a>
## Q10. How would you implement global exception handling in Spring Boot?

### 📝 One-Liner
Use @RestControllerAdvice + @ExceptionHandler methods to catch exceptions across all controllers and return consistent error responses.

### 🔑 Quick Answer
**@RestControllerAdvice** (= @ControllerAdvice + @ResponseBody) creates a global exception handler. Inside it, **@ExceptionHandler** methods catch specific exceptions and return standardized error responses. Flow: Controller throws exception → Spring's DispatcherServlet catches it → finds matching @ExceptionHandler in @RestControllerAdvice → returns error response. Best practice: create a standard **ErrorResponse** DTO with timestamp, status, message, path. Handle layers: **(1)** Custom business exceptions → 4xx. **(2)** Validation errors (MethodArgumentNotValidException) → 400. **(3)** Catch-all Exception → 500 (log full trace, return safe message). *(Ek jagah sab exceptions handle karo — har controller mein alag try-catch nahi chahiye)*

### 📖 How It Works
```
Exception Handling Flow:

  Client Request
       ↓
  DispatcherServlet → Controller → Service → Repository
                                      ↓
                              throws OrderNotFoundException
                                      ↓
  DispatcherServlet catches exception
       ↓
  Searches @ControllerAdvice classes for matching @ExceptionHandler
       ↓
  GlobalExceptionHandler.handleNotFound(ex)
       ↓
  Returns: 404 {"timestamp": "...", "status": 404,
                 "error": "Not Found", "message": "Order 123 not found",
                 "path": "/api/orders/123"}

Exception Hierarchy (handle specific → generic):
  @ExceptionHandler(OrderNotFoundException.class)     → 404
  @ExceptionHandler(DuplicateEmailException.class)    → 409
  @ExceptionHandler(AccessDeniedException.class)      → 403
  @ExceptionHandler(MethodArgumentNotValidException.class) → 400
  @ExceptionHandler(Exception.class)                  → 500 (catch-all)
```

### 🗣️ How to Say in Interview
"I implement global exception handling using @RestControllerAdvice, which acts as a centralized handler for all controller exceptions. Inside it, I define @ExceptionHandler methods for each exception type. I start with specific business exceptions — like ResourceNotFoundException returning 404 — then validation errors returning 400 with field-level details, then a catch-all for unexpected exceptions returning 500 with a safe generic message while logging the full stack trace. I use a standard ErrorResponse DTO with timestamp, status, message, and path for consistency. In my project, this approach eliminated all try-catch blocks from controllers and gave clients a uniform error format regardless of which endpoint they hit."

### 💻 Code
```java
// Standard error response DTO
public record ErrorResponse(
    LocalDateTime timestamp,
    int status,
    String error,
    String message,
    String path,
    Map<String, String> validationErrors  // only for 400s
) {
    public ErrorResponse(int status, String error, String message, String path) {
        this(LocalDateTime.now(), status, error, message, path, null);
    }
}

// Custom business exceptions
public class ResourceNotFoundException extends RuntimeException {
    public ResourceNotFoundException(String resource, Object id) {
        super(String.format("%s with id '%s' not found", resource, id));
    }
}

public class BusinessRuleException extends RuntimeException {
    public BusinessRuleException(String message) { super(message); }
}

// Global exception handler
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {

    // 404 — Resource not found
    @ExceptionHandler(ResourceNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    public ErrorResponse handleNotFound(ResourceNotFoundException ex, HttpServletRequest request) {
        return new ErrorResponse(404, "Not Found", ex.getMessage(), request.getRequestURI());
    }

    // 409 — Business rule violation
    @ExceptionHandler(BusinessRuleException.class)
    @ResponseStatus(HttpStatus.CONFLICT)
    public ErrorResponse handleBusinessRule(BusinessRuleException ex, HttpServletRequest request) {
        return new ErrorResponse(409, "Conflict", ex.getMessage(), request.getRequestURI());
    }

    // 400 — Validation errors (@Valid)
    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public ErrorResponse handleValidation(MethodArgumentNotValidException ex, HttpServletRequest request) {
        Map<String, String> errors = new HashMap<>();
        ex.getBindingResult().getFieldErrors().forEach(fe ->
                errors.put(fe.getField(), fe.getDefaultMessage()));
        
        return new ErrorResponse(
                LocalDateTime.now(), 400, "Validation Failed",
                "Request validation failed", request.getRequestURI(), errors
        );
    }

    // 403 — Access denied
    @ExceptionHandler(AccessDeniedException.class)
    @ResponseStatus(HttpStatus.FORBIDDEN)
    public ErrorResponse handleAccessDenied(AccessDeniedException ex, HttpServletRequest request) {
        return new ErrorResponse(403, "Forbidden", "Access denied", request.getRequestURI());
    }

    // 500 — Catch-all (log full trace, return safe message)
    @ExceptionHandler(Exception.class)
    @ResponseStatus(HttpStatus.INTERNAL_SERVER_ERROR)
    public ErrorResponse handleAll(Exception ex, HttpServletRequest request) {
        log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        // Don't expose internal details to client!
        return new ErrorResponse(500, "Internal Server Error",
                "An unexpected error occurred", request.getRequestURI());
    }
}

// Controller — clean, no try-catch needed
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;

    @GetMapping("/{id}")
    public Order getOrder(@PathVariable Long id) {
        return orderService.findById(id);  // throws ResourceNotFoundException → auto-handled
    }

    @PostMapping
    public Order createOrder(@Valid @RequestBody OrderRequest request) {
        return orderService.create(request);  // validation errors → auto-handled
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Never expose stack traces** to clients in production — log them server-side, return generic message *(stack trace client ko mat dikhao — hackers ke liye goldmine hai)*
- **Order matters**: Spring picks the MOST SPECIFIC @ExceptionHandler match. Put catch-all `Exception.class` last
- **Security exceptions** (401/403) are often handled by Spring Security's filters BEFORE reaching @ControllerAdvice — need `AuthenticationEntryPoint` / `AccessDeniedHandler` for those
- @ControllerAdvice can be scoped: `@RestControllerAdvice(basePackages = "com.myapp.api")`
- Don't catch and re-throw exceptions in services — let them propagate to the handler

### 🎯 Tricky Interview Qs

**Q: Why not just use try-catch in every controller method?**
Code duplication, inconsistent error format, easy to forget. @RestControllerAdvice centralizes logic in one place — every controller gets consistent handling for free. *(Har controller mein try-catch = code duplicate — ek jagah se sab handle karo)*

**Q: How do you handle exceptions thrown by Spring Security?**
Spring Security uses servlet filters that run BEFORE DispatcherServlet. So @ControllerAdvice can't catch 401/403 from Security. You need custom `AuthenticationEntryPoint` (for 401) and `AccessDeniedHandler` (for 403).

### ⚡ Remember
- **@RestControllerAdvice** + **@ExceptionHandler** = global handling *(ek class mein sab exceptions handle)*
- Standard ErrorResponse DTO (timestamp, status, message, path)
- Specific → generic: 404 → 400 → 409 → 500 (catch-all)
- Log full trace for 500s, return safe message to client
- Security exceptions need separate handlers (filters, not advice)

### 🔗 Follow-ups
- [Q9 → @Component/@Service/@Repository (exception translation in @Repository)](#q9)
- [Q15 → REST API design (error format in pagination)](#q15)
- How does Spring Security exception handling work? (AuthenticationEntryPoint)

---

<a id="q11"></a>
## Q11. What happens if two controller methods are mapped with the same HTTP method and URL?

### 📝 One-Liner
Spring Boot throws an **`IllegalStateException: Ambiguous mapping`** at startup — the application won't even start because `RequestMappingHandlerMapping` cannot resolve which handler to use.

### 🔑 Quick Answer
Spring scans all `@RequestMapping` / `@GetMapping` etc. at startup and registers them in `RequestMappingHandlerMapping`. Each endpoint must have a **unique combination** of: URL path + HTTP method + params + headers + consumes + produces. If two methods resolve to the exact same mapping, Spring fails fast at startup with `Ambiguous mapping. Cannot map 'controllerName' method to {GET [/path]}: There is already 'controllerName' bean method mapped.` **Fix**: change the URL, use different HTTP methods, or differentiate by request params / headers. *(Same URL + same method = Spring startup pe exception — application chalegi hi nahi)*

### 📖 How It Works
```
Startup Flow:
  ApplicationContext initializes
    → RequestMappingHandlerMapping scans all @Controller/@RestController
    → For each @GetMapping/@PostMapping etc.:
        → Build RequestMappingInfo (URL + method + params + headers + consumes + produces)
        → Register in handlerMethods map
        → IF key already exists → throw IllegalStateException (Ambiguous mapping)
    → Application FAILS TO START

Uniqueness key = URL path + HTTP method + params + headers + consumes + produces
  /users + GET                 →  unique ✅
  /users + GET                 →  DUPLICATE ❌ (ambiguous)
  /users + POST                →  unique ✅ (different method)
  /users + GET + params="role"  →  unique ✅ (different params)
```

### 🗣️ Answering Approach
"If two controller methods have the same HTTP method and URL, Spring throws an `IllegalStateException` with an 'Ambiguous mapping' message at startup — the application won't start at all. This happens during the `RequestMappingHandlerMapping` initialization phase where Spring scans all controllers and registers their endpoints. Each endpoint must be unique — uniqueness is determined by the combination of URL path, HTTP method, request parameters, headers, consumes, and produces attributes. So you can have two GET methods on the same URL if they differ by params — for example `@GetMapping(value = "/users", params = "role")` vs `@GetMapping("/users")`. To fix a genuine duplicate, either rename one URL, change the HTTP method, or merge the logic. This is actually a good design — failing fast at startup prevents ambiguous behavior at runtime."

### 💻 Code Example

```java
// ❌ BROKEN — Ambiguous mapping at startup
@RestController
@RequestMapping("/users")
public class UserController {

    @GetMapping("/list")
    public List<User> getUsers() {
        return userService.getUsers();
    }

    @GetMapping("/list")  // SAME URL + SAME HTTP METHOD = 💥
    public List<User> getAllUsers() {
        return userService.getAllUsers();
    }
}
// IllegalStateException: Ambiguous mapping. Cannot map 'userController' method
// to {GET [/users/list]}: There is already 'userController' bean method mapped.

// ✅ FIX 1 — Different URLs
@GetMapping("/list")
public List<User> getUsers() { ... }

@GetMapping("/list/all")
public List<User> getAllUsers() { ... }

// ✅ FIX 2 — Different HTTP methods
@GetMapping("/list")
public List<User> getUsers() { ... }

@PostMapping("/list")  // POST vs GET
public List<User> searchUsers(@RequestBody SearchCriteria criteria) { ... }

// ✅ FIX 3 — Different request params
@GetMapping(value = "/list", params = "active=true")
public List<User> getActiveUsers() { ... }

@GetMapping(value = "/list", params = "active=false")
public List<User> getInactiveUsers() { ... }

// ✅ FIX 4 — Different produces (content negotiation)
@GetMapping(value = "/list", produces = MediaType.APPLICATION_JSON_VALUE)
public List<User> getUsersJson() { ... }

@GetMapping(value = "/list", produces = MediaType.APPLICATION_XML_VALUE)
public List<User> getUsersXml() { ... }
```

### ⚠️ Pitfalls / Gotchas
- **Two different controllers with same mapping** — still ambiguous; Spring scans all beans *(alag controller mein bhi same mapping rakha toh bhi error aayega)*
- **Class-level + method-level overlap** — `@RequestMapping("/api")` on class + `@GetMapping("/users")` on method = `/api/users`; check for accidental duplicates across controllers
- **Profile-based controllers** — two controllers active in same profile with same mapping = crash. Use `@Profile` carefully.

### 🆚 Mapping Uniqueness Factors

| Factor | Example | Can Differentiate? |
|--------|---------|-------------------|
| **URL path** | `/users` vs `/users/all` | ✅ Yes |
| **HTTP method** | GET vs POST | ✅ Yes |
| **params** | `params="role"` | ✅ Yes |
| **headers** | `headers="X-Version=2"` | ✅ Yes |
| **consumes** | `application/json` vs `application/xml` | ✅ Yes |
| **produces** | `application/json` vs `text/csv` | ✅ Yes |
| **Method name** | `getUsers()` vs `getAllUsers()` | ❌ No (JVM name doesn't matter) |

### ⚡ Remember
- Same URL + same method = **startup failure** (not runtime — fails fast)
- Uniqueness = URL + HTTP method + params + headers + consumes + produces
- **Fail-fast is a feature** — prevents ambiguous runtime behavior

---

<a id="q12"></a>
## Q12. What is REST and why is it important for backend design?

### 📝 One-Liner
REST (Representational State Transfer) is an **architectural style** for APIs over HTTP — stateless, resource-based, using standard HTTP methods (GET, POST, PUT, DELETE) for CRUD operations.

### 🔑 Quick Answer
**REST principles**: (1) **Stateless** — each request carries all needed info (no server-side session). (2) **Resource-based** — URLs represent resources (`/users/123`), not actions. (3) **Standard HTTP methods** — GET (read), POST (create), PUT (replace), PATCH (partial update), DELETE (remove). (4) **Representations** — resources can be JSON, XML, etc. (5) **Uniform interface** — consistent URL patterns and status codes. (6) **HATEOAS** (optional) — responses include links to related actions. REST matters because it's the universal contract between frontend, mobile, other services — standardized, cacheable, and scalable. *(REST = HTTP methods + resources + stateless — duniya ka sabse common API style)*

### 📖 How It Works
```
REST Resource Design:
  Resource: User
  Base URL: /api/users

  GET    /api/users          →  List all users (200)
  GET    /api/users/123      →  Get user 123 (200 or 404)
  POST   /api/users          →  Create user (201 + Location header)
  PUT    /api/users/123      →  Replace user 123 (200)
  PATCH  /api/users/123      →  Partial update (200)
  DELETE /api/users/123      →  Delete user 123 (204)

Stateless:
  Request 1: GET /users (+ Authorization: Bearer token)
  Request 2: GET /users/123 (+ Authorization: Bearer token)
  Server doesn't remember Request 1 when processing Request 2
  → Each request is self-contained → scales horizontally
```

### 🗣️ Answering Approach
"REST is an architectural style for designing APIs over HTTP. The key idea is that everything is a resource — identified by a URL — and you use standard HTTP methods to operate on it. GET reads, POST creates, PUT replaces, PATCH partially updates, DELETE removes. Each request is stateless — the server doesn't store client state between requests, so any server instance can handle any request. This is what makes REST APIs horizontally scalable — you can put 10 instances behind a load balancer and it just works. The response format is typically JSON, with proper HTTP status codes — 200 for success, 201 for created, 400 for bad input, 404 for not found, 500 for server error. REST became the de facto standard because it's simple, cacheable, and uses the same HTTP that browsers already speak."

### 💻 Code Example

```java
// ✅ Well-designed REST controller with proper status codes
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    @GetMapping("/{id}")
    public ResponseEntity<Employee> getById(@PathVariable Long id) {
        return employeeService.findById(id)
            .map(ResponseEntity::ok)                          // 200
            .orElseThrow(() -> new ResourceNotFoundException( // 404
                "Employee not found: " + id));
    }

    @PostMapping
    public ResponseEntity<Employee> create(
            @Valid @RequestBody EmployeeRequest request) {
        Employee created = employeeService.create(request);
        URI location = ServletUriComponentsBuilder.fromCurrentRequest()
            .path("/{id}").buildAndExpand(created.getId()).toUri();
        return ResponseEntity.created(location).body(created);  // 201 + Location
    }

    @PutMapping("/{id}")
    public ResponseEntity<Employee> update(
            @PathVariable Long id,
            @Valid @RequestBody EmployeeRequest request) {
        return ResponseEntity.ok(employeeService.update(id, request));  // 200
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        employeeService.delete(id);
        return ResponseEntity.noContent().build();  // 204
    }
}

// Global exception handler for clean error responses
@RestControllerAdvice
public class GlobalExceptionHandler {
    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(ResourceNotFoundException ex) {
        return ResponseEntity.status(404).body(
            new ErrorResponse(404, ex.getMessage(), LocalDateTime.now()));
    }

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneral(Exception ex) {
        return ResponseEntity.status(500).body(
            new ErrorResponse(500, "Internal server error", LocalDateTime.now()));
    }
}
```

### 🆚 REST vs Other API Styles

| Style | Protocol | Format | Use When |
|-------|----------|--------|-----------|
| **REST** | HTTP | JSON/XML | Standard CRUD APIs, web/mobile backends |
| **GraphQL** | HTTP | JSON | Complex queries, frontend-driven data needs |
| **gRPC** | HTTP/2 | Protobuf | Service-to-service, high performance |
| **SOAP** | HTTP/SMTP | XML | Enterprise/legacy, strict contracts |

### ⚡ Remember
- **Resources (nouns)** not actions (verbs): `/users` not `/getUsers`
- **Stateless** = horizontal scaling
- **HTTP status codes**: 2xx success, 4xx client error, 5xx server error
- REST + JSON = most common API pattern for 90% of applications
