# 🏢 Accenture — Java Spring Boot Developer Interview Experience (Multiple Rounds)

> Multiple evaluation rounds by different interviewers (Rohit, Kaushik, Anbu, Alok, and others). Heavy focus on Spring Boot REST implementation, exception handling, microservices patterns, threading, security, and scenario-based production questions.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Section A: Spring Boot REST API — Full Implementation Walkthrough

---

<a id="q1"></a>
## Q1. How to expose a class as a REST API? Walk through Controller → Service → Repository → Entity with all annotations

### 📝 One-Liner
Create a layered architecture: `@RestController` handles HTTP, `@Service` holds business logic, `@Repository` interface extends JpaRepository, `@Entity` maps to DB table — Spring Boot auto-wires everything.

### 🔑 Quick Answer
Full flow: Define `@Entity` with JPA annotations → create `@Repository` extending `JpaRepository` → define `Service` interface + `@Service` implementation → create `@RestController` with `@RequestMapping`. Spring's DI auto-connects the layers. *(Poora flow: Entity → Repository → Service → Controller — har layer ka apna annotation, Spring sab inject kar deta hai)*

### 💻 Code
```java
// 1. ENTITY — maps to 'accounts' table
@Entity
@Table(name = "accounts")
public class Account {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String accountHolder;

    @Column(unique = true, nullable = false)
    private String accountNumber;

    @Column(nullable = false)
    private BigDecimal balance;

    @Enumerated(EnumType.STRING)
    @Column(nullable = false)
    private AccountStatus status; // ACTIVE, INACTIVE, CLOSED

    @CreationTimestamp
    private LocalDateTime createdAt;

    @UpdateTimestamp
    private LocalDateTime updatedAt;

    // constructors, getters, setters
}

// 2. REPOSITORY — Spring Data JPA
@Repository
public interface AccountRepository extends JpaRepository<Account, Long> {
    Optional<Account> findByAccountNumber(String accountNumber);
    List<Account> findByStatus(AccountStatus status);
    List<Account> findByAccountHolderContainingIgnoreCase(String name);
}

// 3. SERVICE INTERFACE
public interface AccountService {
    AccountDTO createAccount(CreateAccountRequest request);
    AccountDTO getById(Long id);
    List<AccountDTO> getAll();
    AccountDTO update(Long id, UpdateAccountRequest request);
    void delete(Long id);
}

// 4. SERVICE IMPLEMENTATION
@Service
public class AccountServiceImpl implements AccountService {

    private final AccountRepository accountRepository;

    // Constructor injection (preferred over @Autowired on field)
    public AccountServiceImpl(AccountRepository accountRepository) {
        this.accountRepository = accountRepository;
    }

    @Override
    @Transactional
    public AccountDTO createAccount(CreateAccountRequest request) {
        Account account = new Account();
        account.setAccountHolder(request.getAccountHolder());
        account.setAccountNumber(request.getAccountNumber());
        account.setBalance(request.getInitialBalance());
        account.setStatus(AccountStatus.ACTIVE);
        Account saved = accountRepository.save(account);
        return mapToDTO(saved);
    }

    @Override
    public AccountDTO getById(Long id) {
        Account account = accountRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Account", "id", id));
        return mapToDTO(account);
    }
    // ... other methods
}

// 5. CONTROLLER
@RestController
@RequestMapping("/api/v1/accounts")
public class AccountController {

    private final AccountService accountService;

    public AccountController(AccountService accountService) {
        this.accountService = accountService;
    }

    @PostMapping
    public ResponseEntity<AccountDTO> create(@Valid @RequestBody CreateAccountRequest request) {
        AccountDTO created = accountService.createAccount(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(created);
    }

    @GetMapping("/{id}")
    public ResponseEntity<AccountDTO> getById(@PathVariable Long id) {
        return ResponseEntity.ok(accountService.getById(id));
    }

    @GetMapping
    public ResponseEntity<List<AccountDTO>> getAll() {
        return ResponseEntity.ok(accountService.getAll());
    }

    @PutMapping("/{id}")
    public ResponseEntity<AccountDTO> update(@PathVariable Long id,
            @Valid @RequestBody UpdateAccountRequest request) {
        return ResponseEntity.ok(accountService.update(id, request));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        accountService.delete(id);
        return ResponseEntity.noContent().build();
    }
}
```

### ⚡ Remember
- **Layer annotations**: `@Entity` (JPA), `@Repository` (DAO), `@Service` (business), `@RestController` (web)
- Constructor injection is preferred — no need for `@Autowired` when single constructor
- Always use DTOs for API response — never expose entity directly
- `@Valid` triggers bean validation on request body
- Service interface → implementation pattern enables easy mocking in tests

---

<a id="q2"></a>
## Q2. How to declare @ManyToOne / @OneToMany relationships with annotation details?

### 📝 One-Liner
`@ManyToOne` on the child entity (FK holder), `@OneToMany(mappedBy)` on the parent — always mark the owning side and set `FetchType.LAZY` on collections.

### 🔑 Quick Answer
The entity that holds the foreign key is the **owning side** (`@ManyToOne`). The inverse side uses `@OneToMany(mappedBy = "fieldName")`. Use `cascade` for lifecycle propagation and `orphanRemoval` for auto-deletion of detached children. *(Foreign key wali entity owning side hai — uspe @ManyToOne lagta hai, parent pe @OneToMany mappedBy lagta hai)*

### 💻 Code
```java
// PARENT — Department
@Entity
public class Department {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;

    @OneToMany(mappedBy = "department",     // refers to field in Account
               cascade = CascadeType.ALL,    // persist/merge/remove cascades
               orphanRemoval = true,         // delete child if removed from list
               fetch = FetchType.LAZY)       // ALWAYS lazy for collections
    private List<Account> accounts = new ArrayList<>();

    // Helper method to maintain both sides
    public void addAccount(Account account) {
        accounts.add(account);
        account.setDepartment(this);
    }

    public void removeAccount(Account account) {
        accounts.remove(account);
        account.setDepartment(null);
    }
}

// CHILD — Account (owning side, holds FK)
@Entity
public class Account {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String accountNumber;

    @ManyToOne(fetch = FetchType.LAZY)       // LAZY even for ManyToOne
    @JoinColumn(name = "department_id",       // FK column name
                nullable = false)
    private Department department;
}

// MANY-TO-MANY
@Entity
public class Student {
    @ManyToMany
    @JoinTable(name = "student_course",
        joinColumns = @JoinColumn(name = "student_id"),
        inverseJoinColumns = @JoinColumn(name = "course_id"))
    private Set<Course> courses = new HashSet<>();
}
```

### ⚠️ Pitfalls
- Forgetting `mappedBy` → creates duplicate join table
- Not using helper methods → **only owning side persists** the relationship
- `CascadeType.ALL` on `@ManyToOne` → deleting child deletes parent! (use only on `@OneToMany`)
- EAGER fetch on `@OneToMany` → N+1 performance issue

### ⚡ Remember
- **Owning side**: entity with FK (`@ManyToOne`, `@JoinColumn`)
- **Inverse side**: `@OneToMany(mappedBy = "...")` — does NOT own the relationship
- Always use `FetchType.LAZY` on collections
- `orphanRemoval = true` auto-deletes children removed from parent's collection

---

<a id="q3"></a>
## Q3. Custom Exception handling — @RestControllerAdvice, custom exception class, error codes

### 📝 One-Liner
Define custom exception classes with error codes/messages, throw them from service layer, and handle globally with `@RestControllerAdvice` + `@ExceptionHandler` for consistent API error responses.

### 🔑 Quick Answer
Three steps: **(1)** Create custom exception class extending `RuntimeException` with error code + message. **(2)** Throw from service layer. **(3)** Catch globally with `@RestControllerAdvice`. This keeps controllers clean and ensures every error returns a structured JSON response. *(Custom exception banao with code + message, service se throw karo, @RestControllerAdvice se globally handle karo)*

### 💻 Code
```java
// 1. Error response structure
public class ErrorResponse {
    private int status;
    private String errorCode;
    private String message;
    private LocalDateTime timestamp;
    private String path;

    public ErrorResponse(int status, String errorCode, String message, String path) {
        this.status = status;
        this.errorCode = errorCode;
        this.message = message;
        this.timestamp = LocalDateTime.now();
        this.path = path;
    }
    // getters
}

// 2. Custom exception classes
public class ResourceNotFoundException extends RuntimeException {
    private final String errorCode;

    public ResourceNotFoundException(String resource, String field, Object value) {
        super(String.format("%s not found with %s: '%s'", resource, field, value));
        this.errorCode = "ERR_NOT_FOUND";
    }
    public String getErrorCode() { return errorCode; }
}

public class BusinessException extends RuntimeException {
    private final String errorCode;

    public BusinessException(String errorCode, String message) {
        super(message);
        this.errorCode = errorCode;
    }
    public String getErrorCode() { return errorCode; }
}

public class DuplicateResourceException extends RuntimeException {
    private final String errorCode;

    public DuplicateResourceException(String resource, String field, Object value) {
        super(String.format("%s already exists with %s: '%s'", resource, field, value));
        this.errorCode = "ERR_DUPLICATE";
    }
    public String getErrorCode() { return errorCode; }
}

// 3. Global exception handler
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<ErrorResponse> handleNotFound(
            ResourceNotFoundException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            404, ex.getErrorCode(), ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(error);
    }

    @ExceptionHandler(DuplicateResourceException.class)
    public ResponseEntity<ErrorResponse> handleDuplicate(
            DuplicateResourceException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            409, ex.getErrorCode(), ex.getMessage(), request.getRequestURI());
        return ResponseEntity.status(HttpStatus.CONFLICT).body(error);
    }

    @ExceptionHandler(BusinessException.class)
    public ResponseEntity<ErrorResponse> handleBusiness(
            BusinessException ex, HttpServletRequest request) {
        ErrorResponse error = new ErrorResponse(
            400, ex.getErrorCode(), ex.getMessage(), request.getRequestURI());
        return ResponseEntity.badRequest().body(error);
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(
            MethodArgumentNotValidException ex, HttpServletRequest request) {
        String message = ex.getBindingResult().getFieldErrors().stream()
            .map(e -> e.getField() + ": " + e.getDefaultMessage())
            .collect(Collectors.joining("; "));
        ErrorResponse error = new ErrorResponse(
            400, "ERR_VALIDATION", message, request.getRequestURI());
        return ResponseEntity.badRequest().body(error);
    }

    // Catch-all for unexpected errors — DON'T expose internal details
    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleGeneric(
            Exception ex, HttpServletRequest request) {
        // Log full stack trace internally
        log.error("Unexpected error at {}: {}", request.getRequestURI(), ex.getMessage(), ex);
        // Return generic message to client — never expose stack trace
        ErrorResponse error = new ErrorResponse(
            500, "ERR_INTERNAL", "An unexpected error occurred", request.getRequestURI());
        return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(error);
    }
}

// 4. Throwing from service
@Service
public class AccountServiceImpl implements AccountService {
    @Transactional
    public AccountDTO createAccount(CreateAccountRequest request) {
        if (accountRepository.findByAccountNumber(request.getAccountNumber()).isPresent()) {
            throw new DuplicateResourceException("Account", "accountNumber",
                request.getAccountNumber());
        }
        // ... save logic
    }
}
```

### ⚡ Remember
- **Checked exceptions**: must be caught or declared (`IOException`, `SQLException`) — compile-time
- **Unchecked exceptions**: extend `RuntimeException` — runtime, no forced handling
- **Errors**: JVM-level (`OutOfMemoryError`, `StackOverflowError`) — don't catch typically
- `@RestControllerAdvice` = `@ControllerAdvice` + `@ResponseBody`
- **Security**: never expose stack traces, class names, or DB details in API responses

---

## Section B: Spring Core & Boot Concepts

---

<a id="q4"></a>
## Q4. What is IoC (Inversion of Control)? What is DI? Explain @Autowired

### 📝 One-Liner
IoC = framework controls object creation/lifecycle instead of developer; DI = the mechanism Spring uses to inject dependencies into beans; `@Autowired` tells Spring where to inject.

### 🔑 Quick Answer
**IoC (Inversion of Control)**: instead of `new MyService()`, Spring container creates and manages objects (beans). The control of object creation is "inverted" from developer to framework. **DI (Dependency Injection)**: the specific technique — constructor, setter, or field injection. `@Autowired` marks injection points; with a single constructor, it's optional. *(IoC = object banana aur manage karna Spring ke haath mein — DI = Spring khud dependencies inject karta hai)*

### 💻 Code
```java
// Without IoC — developer creates everything
OrderService orderService = new OrderService(new PaymentService(new StripeClient()));

// With IoC + DI — Spring manages everything
@Service
public class OrderService {
    private final PaymentService paymentService;

    // Constructor injection (preferred — @Autowired optional with single constructor)
    public OrderService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}

// Field injection (not recommended — harder to test)
@Service
public class OrderService {
    @Autowired
    private PaymentService paymentService;
}

// Setter injection (for optional dependencies)
@Service
public class OrderService {
    private PaymentService paymentService;

    @Autowired
    public void setPaymentService(PaymentService paymentService) {
        this.paymentService = paymentService;
    }
}
```

### 🆚 vs.
| Injection Type | Testability | Immutability | When to Use |
|---------------|-------------|-------------|-------------|
| Constructor | ✅ Best | ✅ `final` fields | Default choice |
| Setter | ✅ Good | ❌ Mutable | Optional dependencies |
| Field | ❌ Needs reflection | ❌ Mutable | Avoid in production |

### ⚡ Remember
- IoC annotations: `@Component`, `@Service`, `@Repository`, `@Controller`, `@Configuration`
- `@Autowired` is optional on single constructor (Spring 4.3+)
- Constructor injection allows `final` fields → immutable, null-safe

---

<a id="q5"></a>
## Q5. How to handle duplicate beans? @Primary vs @Qualifier

### 📝 One-Liner
When multiple beans of the same type exist, use `@Primary` to set a default or `@Qualifier("name")` to pick a specific one at the injection point.

### 🔑 Quick Answer
`@Primary` marks one bean as the default when multiple candidates exist. `@Qualifier("beanName")` explicitly selects a specific bean at injection point. `@Qualifier` takes precedence over `@Primary`. You can also use `@ConditionalOnProperty` or custom annotations. *(Agar ek type ke do beans hain toh @Primary default set karta hai, @Qualifier specific bean select karta hai)*

### 💻 Code
```java
// Two implementations of PaymentService
@Service
@Primary  // default choice
public class StripePaymentService implements PaymentService { /*...*/ }

@Service
@Qualifier("paypal")
public class PaypalPaymentService implements PaymentService { /*...*/ }

// Injection — uses @Primary (Stripe) by default
@Service
public class OrderService {
    public OrderService(PaymentService paymentService) { } // → StripePaymentService
}

// Injection — override with @Qualifier
@Service
public class RefundService {
    public RefundService(@Qualifier("paypal") PaymentService paymentService) { }
    // → PaypalPaymentService
}
```

### ⚡ Remember
- `@Qualifier` > `@Primary` (qualifier always wins)
- Bean names default to camelCase class name: `stripePaymentService`
- Use `@ConditionalOnProperty` for config-driven bean selection

---

<a id="q6"></a>
## Q6. How to exclude a predefined auto-configuration class in Spring Boot?

### 📝 One-Liner
Use `@SpringBootApplication(exclude = {SomeAutoConfiguration.class})` or set `spring.autoconfigure.exclude` in properties.

### 💻 Code
```java
// Annotation-based exclusion
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApp { }

// Properties-based exclusion
// application.properties:
// spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration
```

### ⚡ Remember
- Use when auto-configured bean conflicts or isn't needed (e.g., no DB yet but starter-data-jpa is included)
- `@EnableAutoConfiguration(exclude = ...)` works the same way
- Check `AUTO-CONFIGURATION REPORT` with `--debug` flag to see what's being auto-configured

---

<a id="q7"></a>
## Q7. Spring vs Spring Boot — What's the difference?

### 📝 One-Liner
Spring is the core framework (IoC, DI, AOP); Spring Boot adds auto-configuration, embedded servers, and opinionated defaults to eliminate boilerplate setup.

### 🆚 vs.
| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| Configuration | Manual XML / Java Config | Auto-configuration |
| Server | External Tomcat/JBoss | Embedded Tomcat/Jetty/Undertow |
| Startup | Deploy WAR to server | `java -jar app.jar` |
| Dependencies | Manual version management | Starter POMs with BOM |
| Properties | Manual PropertyPlaceholder | application.properties/yml auto-loaded |
| Monitoring | Manual setup | Spring Boot Actuator built-in |
| boilerplate | Heavy (web.xml, dispatcher servlet) | Minimal (@SpringBootApplication) |

### 🔑 Quick Answer
Spring Boot is NOT a replacement for Spring — it's a layer ON TOP. Spring provides the core features (IoC, MVC, Security, Data). Spring Boot provides the convenience: auto-configuration detects classpath libraries and auto-configures beans, embedded servers remove the need for external deployment, and starters bundle common dependencies. *(Spring = engine, Spring Boot = ready-made car with engine inside — sab pre-configured aata hai)*

### ⚡ Remember
- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- Spring Boot DevTools, Actuator, Starters — are Boot-only features
- You can still use Spring without Boot (large enterprises sometimes do)

---

<a id="q8"></a>
## Q8. Bean Scopes — How does Singleton differ from Prototype? Are Spring beans thread-safe?

### 📝 One-Liner
Singleton (default): one instance per container, shared across all injections. Prototype: new instance every injection. Spring beans are **NOT automatically thread-safe** — singleton + mutable state = thread-unsafe.

### 🔑 Quick Answer
Singleton scope: Spring creates one instance and reuses it. Prototype: new instance each time `getBean()` is called. Request/Session/Application: web-scoped. **Thread safety**: Singleton beans shared across threads — if they have mutable fields, they're NOT thread-safe. Make them stateless (no mutable fields) or use `ThreadLocal`/`AtomicXxx`. *(Singleton ek hi object sab jagah — agar usme mutable state hai toh thread-safe nahi hai, stateless banao)*

### 💻 Code
```java
@Component
@Scope("singleton") // default — one instance
public class SingletonService {
    // ❌ NOT thread-safe — shared mutable state
    private int counter = 0;
    public void increment() { counter++; }

    // ✅ Thread-safe — no mutable state
    private final UserRepository repo; // injected, immutable reference
    public User findUser(Long id) { return repo.findById(id).orElse(null); }
}

@Component
@Scope("prototype") // new instance every time
public class PrototypeBean { }

// Injecting prototype into singleton — common trap
@Service
public class OrderService {
    @Autowired
    private ObjectProvider<PrototypeBean> protoProvider;

    public void process() {
        PrototypeBean fresh = protoProvider.getObject(); // new instance each time
    }
}
```

### 🆚 vs.
| Aspect | Singleton Scope | Singleton Design Pattern |
|--------|----------------|------------------------|
| Managed by | Spring Container | Developer (static instance) |
| Per what? | Per ApplicationContext | Per JVM/ClassLoader |
| Creation | Container startup | First access (lazy) |
| Instances | One per container (can have multiple containers) | One per ClassLoader |

### ⚡ Remember
- Singleton beans: make stateless → inherently thread-safe
- Prototype beans: `@PreDestroy` NOT called by Spring (doesn't manage lifecycle)
- Prototype in Singleton trap: inject `ObjectProvider<T>` or use `@Lookup`
- Request/Session scope: only available in web-aware `ApplicationContext`

---

<a id="q9"></a>
## Q9. @Value annotation and loading config from properties files

### 📝 One-Liner
`@Value("${property.key}")` injects a single value from application.properties; use default with `@Value("${key:default}")`.

### 💻 Code
```java
@Service
public class NotificationService {
    @Value("${app.notification.enabled:true}")
    private boolean enabled;

    @Value("${app.notification.max-retries:3}")
    private int maxRetries;

    @Value("${app.notification.recipients}")
    private List<String> recipients; // comma-separated in properties

    @Value("#{${app.rate-limits}}")  // SpEL for maps
    private Map<String, Integer> rateLimits;
}

// application.properties
// app.notification.enabled=true
// app.notification.max-retries=5
// app.notification.recipients=admin@co.in,ops@co.in
// app.rate-limits={gold:100, silver:50, bronze:10}
```

### ⚡ Remember
- `@Value` resolves at bean creation — missing property without default = startup failure
- For structured config, prefer `@ConfigurationProperties` → type-safe binding to POJO
- SpEL: `@Value("#{systemProperties['user.home']}")` for system properties
- `@PropertySource("classpath:custom.properties")` to load custom files

---

<a id="q10"></a>
## Q10. @Component vs @Configuration — What's the difference?

### 📝 One-Liner
`@Component` registers a class as a Spring bean; `@Configuration` is a special `@Component` that uses CGLIB proxy to ensure `@Bean` methods return singleton instances.

### 🔑 Quick Answer
`@Configuration` classes are proxied by CGLIB — calling a `@Bean` method from another `@Bean` method returns the **same singleton instance** (inter-bean references). With `@Component`, each call to a `@Bean` method creates a **new instance** (lite mode). *(Configuration mein @Bean methods ek doosre ko call karein toh same instance milta hai — Component mein naya ban jaata hai)*

### 💻 Code
```java
@Configuration // CGLIB proxied — inter-bean references work correctly
public class AppConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // ← returns SAME DataSource bean
    }
}

@Component // NOT proxied — lite mode
public class LiteConfig {
    @Bean
    public DataSource dataSource() { return new HikariDataSource(); }

    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // ← creates NEW DataSource! (bug!)
    }
}
```

### 🆚 vs.
| Aspect | @Configuration | @Component |
|--------|---------------|------------|
| CGLIB Proxy | ✅ Yes | ❌ No (lite mode) |
| @Bean inter-references | Singleton (same instance) | New instance each call |
| Use case | Bean definitions, third-party config | Business components |

Also: `@Configuration` vs `@ConfigurationProperties`:
| @Configuration | @ConfigurationProperties |
|---------------|------------------------|
| Define beans, create config | Bind properties to POJO |
| Contains `@Bean` methods | Contains fields matching property keys |
| `@Bean public DataSource ds()` | `@ConfigurationProperties(prefix="app.ds")` |

### ⚡ Remember
- Use `@Configuration` when defining `@Bean` methods
- Use `@Component`/`@Service`/`@Repository` for auto-detected domain beans
- `@ConfigurationProperties(prefix = "app")` for binding properties to POJO — different purpose entirely

---

<a id="q11"></a>
## Q11. How to create an asynchronous service in Spring Boot?

### 📝 One-Liner
Enable with `@EnableAsync`, annotate methods with `@Async`, return `CompletableFuture<T>` — Spring runs the method on a separate thread pool.

### 🔑 Quick Answer
`@Async` makes a method run in a different thread from the caller. Requires `@EnableAsync` on a config class. Default: `SimpleAsyncTaskExecutor` (creates new thread each time — bad for production). Always configure a custom `ThreadPoolTaskExecutor`. Return `CompletableFuture` for async results. *(Async method alag thread pe run hota hai — production mein hamesha custom thread pool configure karo)*

### 💻 Code
```java
@Configuration
@EnableAsync
public class AsyncConfig {
    @Bean("asyncExecutor")
    public Executor asyncExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("AsyncThread-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {

    @Async("asyncExecutor")
    public CompletableFuture<String> sendEmail(String to, String body) {
        // runs on async thread — does not block caller
        emailClient.send(to, body);
        return CompletableFuture.completedFuture("Email sent to " + to);
    }

    @Async("asyncExecutor")
    public void sendSMS(String phone, String message) {
        // fire-and-forget — void return
        smsClient.send(phone, message);
    }
}

// Caller
@Service
public class OrderService {
    @Autowired private NotificationService notificationService;

    public void placeOrder(Order order) {
        orderRepo.save(order);
        // These run async — placeOrder returns immediately
        notificationService.sendEmail(order.getEmail(), "Order placed!");
        notificationService.sendSMS(order.getPhone(), "Order #" + order.getId());
    }
}
```

### ⚠️ Pitfalls
- **Self-invocation**: `this.asyncMethod()` bypasses proxy → runs synchronously
- Default executor creates unlimited threads — always configure pool
- `@Async` on private methods doesn't work (needs proxy interception)
- Exception in void `@Async` method is silently swallowed unless you configure `AsyncUncaughtExceptionHandler`

### ⚡ Remember
- `@Async` + `@Transactional`: `@Async` method gets its own transaction (new thread = new EntityManager)
- For Kafka slow response scenario: use `@Async` to publish message and return immediately
- `CompletableFuture.allOf(f1, f2, f3).join()` to wait for all async results

---

<a id="q12"></a>
## Q12. Spring Boot Actuator — What are its benefits?

### 📝 One-Liner
Actuator provides production-ready endpoints for health checks, metrics, environment info, thread dumps, and more — essential for monitoring and operations.

### 🔑 Quick Answer
Key endpoints: `/actuator/health` (liveness/readiness), `/actuator/metrics` (JVM, HTTP, custom), `/actuator/info` (app info), `/actuator/env` (config properties), `/actuator/loggers` (change log levels at runtime), `/actuator/threaddump`. Used by Kubernetes for pod health probes and by monitoring tools (Prometheus/Grafana). *(Production mein Actuator se health check, metrics, aur runtime debugging hoti hai — Kubernetes readiness/liveness probes yahi use karte hain)*

### 💻 Code
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers,env
  endpoint:
    health:
      show-details: when-authorized
  health:
    db:
      enabled: true
    diskSpace:
      enabled: true
```
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
```

### ⚡ Remember
- **Security**: never expose all endpoints publicly — use Spring Security to restrict
- `/health` is default exposed; others need explicit `include`
- Prometheus + Micrometer: `management.endpoints.web.exposure.include=prometheus`
- Custom health indicator: implement `HealthIndicator` interface
- After deployment check: `curl /actuator/health` returns `{"status": "UP"}`

---

## Section C: Java Core & Threading

---

<a id="q13"></a>
## Q13. HashMap vs Hashtable vs ConcurrentHashMap — Differences explained

### 📝 One-Liner
HashMap = unsynchronized + allows null; Hashtable = synchronized on entire map + no null; ConcurrentHashMap = segment/node-level locking + no null + best concurrent performance.

### 🆚 vs.
| Aspect | HashMap | Hashtable | ConcurrentHashMap |
|--------|---------|-----------|-------------------|
| Thread-safe | ❌ | ✅ (whole map lock) | ✅ (node-level CAS) |
| Null key/value | ✅ / ✅ | ❌ / ❌ | ❌ / ❌ |
| Performance | Fastest (single-thread) | Slowest (global lock) | Best (concurrent) |
| Iterator | Fail-fast | Fail-fast | Weakly consistent |
| Java version | 1.2 | 1.0 (legacy) | 1.5 (improved in 1.8) |
| Lock mechanism | N/A | `synchronized` on every method | CAS + `synchronized` per bucket (Java 8) |

### 🔑 Quick Answer
**HashMap**: not thread-safe, best for single-threaded or externally synchronized use. **Hashtable**: legacy, entire map locked on every operation — terrible throughput. **ConcurrentHashMap**: modern, uses CAS (Compare-And-Swap) at node level in Java 8+ — allows concurrent reads with no locking and concurrent writes on different buckets. *(ConcurrentHashMap sabse best hai concurrent access ke liye — node-level locking se throughput bahut acha hai)*

### 💻 Code
```java
// ConcurrentHashMap — safe atomic operations
ConcurrentHashMap<String, Integer> map = new ConcurrentHashMap<>();
map.put("a", 1);
map.computeIfAbsent("b", k -> expensiveCalculation(k)); // atomic
map.merge("a", 1, Integer::sum); // atomic increment

// HashMap + Collections.synchronizedMap — technically thread-safe but poor performance
Map<String, Integer> synced = Collections.synchronizedMap(new HashMap<>());
// Still need external sync for compound operations (check-then-act)
```

### ⚡ Remember
- **Never use Hashtable** — use ConcurrentHashMap instead
- `ConcurrentHashMap` doesn't allow null keys/values (ambiguity in concurrent context)
- `Collections.synchronizedMap()` wraps with single lock — ConcurrentHashMap is far better
- `ConcurrentHashMap.size()` is approximate in concurrent context; use `mappingCount()`

---

<a id="q14"></a>
## Q14. How does ArrayList grow dynamically? How to avoid ConcurrentModificationException?

### 📝 One-Liner
ArrayList uses an internal array that grows by 50% (1.5x) when full via `Arrays.copyOf()`; ConcurrentModificationException occurs when modifying a list during iteration.

### 🔑 Quick Answer
**Growth**: initial capacity 10; when full: `newCapacity = oldCapacity + (oldCapacity >> 1)` (1.5x). Creates new larger array, copies elements via `System.arraycopy()`. **ConcurrentModification**: iterator detects structural modification via `modCount`. Fix: use `Iterator.remove()`, `CopyOnWriteArrayList`, or `removeIf()`. *(ArrayList full hone pe 1.5 guna bada array banata hai aur copy karta hai — loop mein modify karna ho toh Iterator.remove() ya CopyOnWriteArrayList use karo)*

### 💻 Code
```java
// ArrayList growth
ArrayList<Integer> list = new ArrayList<>(); // default capacity: 10
// When 11th element added: new capacity = 10 + 10/2 = 15
// When 16th element added: new capacity = 15 + 15/2 = 22

// ❌ ConcurrentModificationException
List<String> names = new ArrayList<>(List.of("A", "B", "C"));
for (String name : names) {
    if (name.equals("B")) names.remove(name); // THROWS CME!
}

// ✅ Fix 1: Iterator.remove()
Iterator<String> it = names.iterator();
while (it.hasNext()) {
    if (it.next().equals("B")) it.remove(); // safe
}

// ✅ Fix 2: removeIf (Java 8+)
names.removeIf(name -> name.equals("B"));

// ✅ Fix 3: CopyOnWriteArrayList (for concurrent access)
List<String> safeList = new CopyOnWriteArrayList<>(List.of("A", "B", "C"));
for (String name : safeList) {
    if (name.equals("B")) safeList.remove(name); // safe — iterates over snapshot
}
```

### ⚡ Remember
- Growth: 10 → 15 → 22 → 33 → ... (1.5x each time)
- Pre-size with `new ArrayList<>(expectedSize)` to avoid repeated copies
- `CopyOnWriteArrayList`: safe for concurrent iteration but slow writes (copies entire array on write)
- `removeIf()` is the cleanest single-threaded solution for filtering during iteration

---

<a id="q15"></a>
## Q15. Comparator vs Comparable — When to use which?

### 📝 One-Liner
`Comparable` defines natural ordering inside the class itself (`compareTo`); `Comparator` defines external ordering outside the class (`compare`) — use Comparator for multiple/custom sorts.

### 🔑 Quick Answer
`Comparable<T>`: implement in the class, override `compareTo(T other)` — defines THE natural order (e.g., String alphabetical). `Comparator<T>`: separate class/lambda for custom sorting — can have multiple. Java 8: `Comparator.comparing()` chain for concise multi-field sorting. *(Comparable = class ke andar natural order, Comparator = bahar se custom order — multiple sorts ke liye Comparator use karo)*

### 💻 Code
```java
// Comparable — natural ordering (one way)
public class Employee implements Comparable<Employee> {
    private String name;
    private double salary;

    @Override
    public int compareTo(Employee other) {
        return Double.compare(this.salary, other.salary); // natural order: by salary
    }
}
Collections.sort(employees); // uses compareTo

// Comparator — external, multiple sort strategies
Comparator<Employee> byName = Comparator.comparing(Employee::getName);
Comparator<Employee> bySalaryDesc = Comparator.comparingDouble(Employee::getSalary).reversed();
Comparator<Employee> byNameThenSalary = Comparator.comparing(Employee::getName)
    .thenComparingDouble(Employee::getSalary);

employees.sort(byNameThenSalary);

// City sorting example (asked in Accenture): sort by name, then by population
List<City> cities = /* ... */;
cities.sort(Comparator.comparing(City::getName)
    .thenComparingLong(City::getPopulation)); // same name → lower population first
```

### 🆚 vs.
| Aspect | Comparable | Comparator |
|--------|-----------|------------|
| Package | `java.lang` | `java.util` |
| Method | `compareTo(T)` | `compare(T, T)` |
| Defined in | The class itself | External/lambda |
| Sort orders | One (natural) | Multiple |
| Modifies class | ✅ Yes | ❌ No |

### ⚡ Remember
- `Comparable` = "I can compare myself" | `Comparator` = "someone else compares me"
- Java 8: `Comparator.comparing().thenComparing().reversed()` — most concise
- TreeMap/TreeSet require elements to be `Comparable` OR supply a `Comparator`

---

<a id="q16"></a>
## Q16. Creating threads — Parent to child thread, and Thread lifecycle

### 📝 One-Liner
Create child threads via `new Thread(runnable)`, `ExecutorService`, or `CompletableFuture.supplyAsync()`; lifecycle: NEW → RUNNABLE → RUNNING → BLOCKED/WAITING → TERMINATED.

### 💻 Code
```java
// 1. Thread — from parent context
Thread child = new Thread(() -> {
    System.out.println("Child running in: " + Thread.currentThread().getName());
});
child.start(); // NEW → RUNNABLE

// 2. ExecutorService — managed thread pool
ExecutorService executor = Executors.newFixedThreadPool(4);
Future<String> future = executor.submit(() -> {
    return "Result from child thread";
});
String result = future.get(); // blocks parent until child completes

// 3. CompletableFuture — non-blocking
CompletableFuture.supplyAsync(() -> fetchFromDB())
    .thenApply(data -> transform(data))
    .thenAccept(result -> sendResponse(result));
```

### 📖 How It Works
```
Thread Lifecycle:
  NEW ──start()──► RUNNABLE ──scheduler──► RUNNING
                      ▲                       │
                      │                       ├──sleep()/wait()──► TIMED_WAITING/WAITING
                      │                       │                          │
                      │                       ├──synchronized(locked)──► BLOCKED
                      │                       │                          │
                      │                       │    notify()/timeout       │
                      │◄──────────────────────┼──────────────────────────┘
                      │                       │
                      │                       └──run() exits──► TERMINATED
```

### 🆚 vs.
| Method | Purpose | Resumption |
|--------|---------|------------|
| `sleep(ms)` | Pause current thread, keeps lock | Auto after timeout |
| `wait()` | Release lock, wait for notify | `notify()` / `notifyAll()` |
| `join()` | Wait for another thread to finish | When target thread ends |
| `yield()` | Hint to scheduler, may be ignored | Immediately (hint only) |

### ⚡ Remember
- `sleep()` — Thread class, keeps lock | `wait()` — Object class, releases lock
- `wait()`/`notify()` must be called inside `synchronized` block
- Thread methods on **Thread class**: `start`, `sleep`, `join`, `yield`, `interrupt`
- Thread methods on **Object class**: `wait`, `notify`, `notifyAll`

---

<a id="q17"></a>
## Q17. What is a race condition? How to detect and prevent deadlocks?

### 📝 One-Liner
Race condition: multiple threads access shared mutable state without synchronization, producing unpredictable results. Deadlock: two or more threads waiting forever for each other's locks.

### 🔑 Quick Answer
**Race condition**: `count++` is read-increment-write (3 steps) — two threads can read same value. Fix: `AtomicInteger`, `synchronized`, or `Lock`. **Deadlock**: Thread A holds Lock 1, waits for Lock 2; Thread B holds Lock 2, waits for Lock 1. Fix: consistent lock ordering, timeout with `tryLock()`, avoid nested locks. *(Race condition = do threads same data pe ek saath operate karein — deadlock = do threads ek doosre ka lock hold karke wait karein)*

### 💻 Code
```java
// RACE CONDITION
class UnsafeCounter {
    private int count = 0;
    public void increment() { count++; } // NOT atomic — race condition!
}

// FIX: AtomicInteger
class SafeCounter {
    private final AtomicInteger count = new AtomicInteger(0);
    public void increment() { count.incrementAndGet(); } // CAS — atomic
}

// DEADLOCK example
// Thread 1: lock(A) → lock(B)
// Thread 2: lock(B) → lock(A)  ← deadlock!

// FIX: consistent lock ordering
// Thread 1: lock(A) → lock(B)
// Thread 2: lock(A) → lock(B)  ← same order, no deadlock

// FIX: tryLock with timeout
ReentrantLock lockA = new ReentrantLock();
ReentrantLock lockB = new ReentrantLock();

if (lockA.tryLock(1, TimeUnit.SECONDS)) {
    try {
        if (lockB.tryLock(1, TimeUnit.SECONDS)) {
            try { /* work */ }
            finally { lockB.unlock(); }
        }
    } finally { lockA.unlock(); }
}
```

### ⚡ Remember
- **Detect deadlocks**: `jstack <pid>`, VisualVM, `ThreadMXBean.findDeadlockedThreads()`
- **Prevent**: consistent lock ordering, minimize lock scope, use `tryLock()` with timeout
- **Release locks**: always in `finally` block; `try-with-resources` for `Lock` isn't built-in
- `ReadWriteLock`: multiple concurrent readers, exclusive writer — great for read-heavy scenarios

---

<a id="q18"></a>
## Q18. How to create an immutable class in Java?

### 📝 One-Liner
Make class `final`, all fields `private final`, no setters, deep-copy mutable objects in constructor and getter — or use Java `record` (Java 16+).

### 💻 Code
```java
// Manual immutable class
public final class Money {
    private final BigDecimal amount;
    private final String currency;
    private final List<String> tags;

    public Money(BigDecimal amount, String currency, List<String> tags) {
        this.amount = amount;
        this.currency = currency;
        this.tags = List.copyOf(tags); // defensive copy — immutable list
    }

    public BigDecimal getAmount() { return amount; }
    public String getCurrency() { return currency; }
    public List<String> getTags() { return tags; } // already immutable copy
}

// Java 16+ Record — immutable by default
public record Money(BigDecimal amount, String currency, List<String> tags) {
    public Money { // compact constructor for validation
        tags = List.copyOf(tags); // defensive copy
    }
}
```

### ⚡ Remember
- **String is immutable** (not singleton — each unique literal is pooled via String Pool)
- Rules: `final` class, `private final` fields, no setters, defensive copies
- Mutable fields (Date, List, Map) MUST be deep-copied in constructor and getter
- Immutable objects are inherently thread-safe

---

## Section D: Microservices & Communication

---

<a id="q19"></a>
## Q19. How do microservices communicate? Sync vs Async approaches

### 📝 One-Liner
Synchronous: REST (Feign/RestTemplate/WebClient), gRPC. Asynchronous: Message Queues (Kafka, RabbitMQ). Choose based on whether the caller needs an immediate response.

### 🆚 vs.
| Aspect | Synchronous | Asynchronous |
|--------|------------|-------------|
| Pattern | Request-Response | Event/Message-driven |
| Tools | Feign, WebClient, gRPC | Kafka, RabbitMQ, SQS |
| Coupling | Temporal (caller waits) | Decoupled (fire and forget) |
| Latency | Adds to request path | Non-blocking for caller |
| Failure | Cascading (needs Circuit Breaker) | Retries via dead-letter queue |
| Best for | Queries, real-time responses | Events, notifications, heavy processing |

### 💻 Code
```java
// Sync — Feign Client
@FeignClient(name = "payment-service")
public interface PaymentClient {
    @PostMapping("/api/payments")
    PaymentResponse processPayment(@RequestBody PaymentRequest request);
}

// Async — Kafka producer
@Service
public class OrderEventPublisher {
    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;

    public void publishOrderCreated(Order order) {
        kafkaTemplate.send("order-events", order.getId().toString(),
            new OrderEvent("ORDER_CREATED", order));
    }
}

// Kafka consumer
@KafkaListener(topics = "order-events", groupId = "payment-service")
public void handleOrderEvent(OrderEvent event) {
    if ("ORDER_CREATED".equals(event.getType())) {
        paymentService.processPayment(event.getOrder());
    }
}
```

### ⚡ Remember
- **Sync**: Feign (declarative), WebClient (reactive), RestClient (Spring Boot 3.2+)
- **Async**: Kafka (high-throughput, partitioned), RabbitMQ (routing, exchanges)
- Kafka slow response scenario: use `@Async` to publish and return immediately
- Always pair sync calls with Circuit Breaker + Retry + Timeout

---

<a id="q20"></a>
## Q20. Circuit Breaker + Retry mechanism — What if DB is down?

### 📝 One-Liner
Circuit Breaker prevents cascading failures; Retry with exponential backoff handles transient failures — combine both for resilient services.

### 🔑 Quick Answer
**Retry** (Resilience4j): automatically retry failed operations with configurable delay and max attempts — good for transient DB connection issues. **Circuit Breaker**: if retries keep failing beyond threshold, stop trying entirely and return fallback. Order: Retry → Circuit Breaker (retry wraps the actual call, CB wraps retry). *(Retry = thodi der baad dobara try karo; Circuit Breaker = bahut baar fail ho toh try karna band karo)*

### 💻 Code
```java
@Service
public class AccountService {

    @CircuitBreaker(name = "dbService", fallbackMethod = "dbFallback")
    @Retry(name = "dbRetry")
    public Account getAccount(Long id) {
        return accountRepository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Account", "id", id));
    }

    public Account dbFallback(Long id, Throwable t) {
        // Return cached data or meaningful default
        return cacheService.getCachedAccount(id)
            .orElse(new Account(id, "Unavailable", BigDecimal.ZERO));
    }
}
```
```yaml
# application.yml
resilience4j:
  retry:
    instances:
      dbRetry:
        max-attempts: 3
        wait-duration: 2s
        exponential-backoff-multiplier: 2  # 2s, 4s, 8s
        retry-exceptions:
          - org.springframework.dao.TransientDataAccessException
          - java.sql.SQLTransientConnectionException
  circuitbreaker:
    instances:
      dbService:
        failure-rate-threshold: 50
        wait-duration-in-open-state: 30s
        sliding-window-size: 10
```

### ⚡ Remember
- Retry for **transient failures** (network hiccup, connection timeout)
- Circuit Breaker for **persistent failures** (DB completely down)
- Execution order: `@Retry` → `@CircuitBreaker` (retry runs inside CB's monitoring)
- Always add exponential backoff — avoids thundering herd on recovery

---

<a id="q21"></a>
## Q21. Kafka — Group ID, Offset, and mandatory Producer/Consumer attributes

### 📝 One-Liner
Group ID identifies a consumer group for parallel consumption; Offset tracks position in a partition; Producer needs bootstrap servers + serializers, Consumer needs those + group ID + deserializers.

### 📖 How It Works
```
Topic: "orders" (3 partitions)
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Partition0│  │ Partition1│  │ Partition2│
│ offset: 0│  │ offset: 0│  │ offset: 0│
│        1 │  │        1 │  │        1 │
│        2 │  │        2 │  │        2 │ ← current offset
└──────────┘  └──────────┘  └──────────┘
      │              │              │
Consumer Group "payment-svc" (groupId):
  Consumer A ←───┘              │              │
  Consumer B ←──────────────────┘              │
  Consumer C ←─────────────────────────────────┘
  (each consumer reads from assigned partitions)

Group ID role:
- Same group ID → messages distributed across consumers (load balance)
- Different group ID → each group gets ALL messages (broadcast)
```

### 💻 Code
```yaml
# Producer mandatory config
spring:
  kafka:
    bootstrap-servers: localhost:9092
    producer:
      key-serializer: org.apache.kafka.common.serialization.StringSerializer
      value-serializer: org.springframework.kafka.support.serializer.JsonSerializer
      acks: all          # wait for all replicas
      retries: 3

# Consumer mandatory config
    consumer:
      bootstrap-servers: localhost:9092
      group-id: payment-service    # consumer group
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.springframework.kafka.support.serializer.JsonDeserializer
      auto-offset-reset: earliest  # start from beginning if no committed offset
      enable-auto-commit: false    # manual commit for exactly-once
```

### 🆚 vs. Kafka vs MQ (RabbitMQ)
| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Model | Distributed log | Message broker |
| Throughput | Very high (millions/sec) | Moderate (thousands/sec) |
| Retention | Configurable (days/weeks) | Consumed = deleted |
| Ordering | Per partition | Per queue |
| Replay | ✅ Can re-read old messages | ❌ Once consumed, gone |
| Best for | Event streaming, analytics | Task queues, routing |

### ⚡ Remember
- **Offset**: position of consumer in a partition — committed offset = "I've processed up to here"
- **Group ID**: consumers with same group ID share partitions; different groups get all messages
- `auto-offset-reset: earliest` → read from start; `latest` → read only new messages
- Kafka taking too long? → publish async with `@Async`, don't block API thread

---

## Section E: Database & SQL

---

<a id="q22"></a>
## Q22. PUT vs PATCH — What's the real difference?

### 📝 One-Liner
PUT replaces the **entire resource**; PATCH updates only the **specified fields** — PUT is idempotent (same result every time), PATCH is typically idempotent too but not guaranteed.

### 💻 Code
```java
// PUT — full replacement (must send ALL fields)
@PutMapping("/{id}")
public ResponseEntity<AccountDTO> fullUpdate(@PathVariable Long id,
        @Valid @RequestBody AccountDTO dto) {
    Account account = findOrThrow(id);
    account.setName(dto.getName());        // all fields replaced
    account.setEmail(dto.getEmail());
    account.setBalance(dto.getBalance());
    return ResponseEntity.ok(mapToDTO(accountRepository.save(account)));
}

// PATCH — partial update (only send changed fields)
@PatchMapping("/{id}")
public ResponseEntity<AccountDTO> partialUpdate(@PathVariable Long id,
        @RequestBody Map<String, Object> updates) {
    Account account = findOrThrow(id);
    updates.forEach((key, value) -> {
        switch (key) {
            case "name" -> account.setName((String) value);
            case "email" -> account.setEmail((String) value);
            case "balance" -> account.setBalance(new BigDecimal(value.toString()));
        }
    });
    return ResponseEntity.ok(mapToDTO(accountRepository.save(account)));
}
```

### 🆚 vs.
| Aspect | PUT | PATCH |
|--------|-----|-------|
| Payload | Full resource | Partial fields |
| Missing fields | Set to null/default | Unchanged |
| Idempotent | ✅ Always | ✅ Usually |
| Use case | Full form submit | Single field edit |

---

<a id="q23"></a>
## Q23. Function vs Stored Procedure in SQL — Differences

### 📝 One-Liner
Function MUST return a value and can be used in SELECT; Stored Procedure can return 0 or more values, supports transactions, and is called with CALL/EXEC.

### 🆚 vs.
| Aspect | Function | Stored Procedure |
|--------|----------|-----------------|
| Return | Must return a value | Optional — OUT params or result sets |
| Usage in SQL | ✅ `SELECT fn()` in queries | ❌ Called with `CALL/EXEC` |
| Transaction | Cannot manage TX | Can use COMMIT/ROLLBACK |
| DML allowed | Varies by DB (restrictions) | Full DML (INSERT/UPDATE/DELETE) |
| Try-Catch | Limited | Full exception handling |
| Use case | Calculations, transformations | Business logic, batch operations |

### 💻 Code
```sql
-- Function — returns a value, usable in SELECT
CREATE FUNCTION get_account_balance(acc_id BIGINT)
RETURNS DECIMAL(10,2)
BEGIN
    DECLARE bal DECIMAL(10,2);
    SELECT balance INTO bal FROM accounts WHERE id = acc_id;
    RETURN bal;
END;
-- Usage: SELECT name, get_account_balance(id) FROM accounts;

-- Stored Procedure — business logic with transactions
CREATE PROCEDURE transfer_funds(
    IN from_id BIGINT, IN to_id BIGINT, IN amount DECIMAL(10,2))
BEGIN
    START TRANSACTION;
    UPDATE accounts SET balance = balance - amount WHERE id = from_id;
    UPDATE accounts SET balance = balance + amount WHERE id = to_id;
    COMMIT;
END;
-- Usage: CALL transfer_funds(1, 2, 500.00);
```

---

<a id="q24"></a>
## Q24. INNER JOIN vs LEFT JOIN vs RIGHT JOIN — Explained

### 📝 One-Liner
INNER JOIN returns only matching rows from both tables; LEFT JOIN returns all left rows + nulls for non-matching right; RIGHT JOIN returns all right rows + nulls for non-matching left.

### 📖 How It Works
```
Table A (employees)    Table B (departments)
| id | name  | dept |  | id | dept_name |
|----|-------|------|  |----|-----------|
| 1  | Alice | 10   |  | 10 | IT        |
| 2  | Bob   | 20   |  | 20 | HR        |
| 3  | Carol | 30   |  | 40 | Finance   |
                        (no dept 30!)

INNER JOIN: Alice-IT, Bob-HR              (only matches — Carol excluded, Finance excluded)
LEFT JOIN:  Alice-IT, Bob-HR, Carol-NULL  (all employees — Carol has no matching dept)
RIGHT JOIN: Alice-IT, Bob-HR, NULL-Finance(all departments — Finance has no employee)
FULL JOIN:  Alice-IT, Bob-HR, Carol-NULL, NULL-Finance (everything)
```

### ⚡ Remember
- **INNER**: intersection — only matches
- **LEFT**: all left + matching right (nulls for no match)
- **RIGHT**: all right + matching left (nulls for no match) — rarely used, prefer LEFT JOIN
- **FULL OUTER**: everything from both (not supported in MySQL — use UNION)
- Performance: ensure JOIN columns are **indexed**

---

<a id="q25"></a>
## Q25. How to find indexes on a table? How do indexes improve performance?

### 📝 One-Liner
Indexes are B-Tree (or Hash) structures that allow O(log n) lookups instead of O(n) full table scans — check with `SHOW INDEX` (MySQL) or `pg_indexes` (PostgreSQL).

### 💻 Code
```sql
-- MySQL: show indexes
SHOW INDEX FROM employees;

-- PostgreSQL: show indexes
SELECT * FROM pg_indexes WHERE tablename = 'employees';

-- Oracle
SELECT * FROM user_indexes WHERE table_name = 'EMPLOYEES';

-- Create index
CREATE INDEX idx_emp_dept ON employees(department);
CREATE INDEX idx_emp_name_dept ON employees(name, department);  -- composite

-- Check if query uses index
EXPLAIN SELECT * FROM employees WHERE department = 'IT';
-- Look for: type=ref (index used) vs type=ALL (full scan)
```

### ⚡ Remember
- Index on: WHERE, JOIN, ORDER BY columns
- Don't over-index: each index slows down INSERT/UPDATE
- Composite index: **leftmost prefix rule** — `(A, B, C)` index works for `WHERE A`, `WHERE A AND B`, but NOT `WHERE B` alone
- Functions on indexed columns disable index: `WHERE UPPER(name)` = full scan

---

## Section F: Security & Authentication

---

<a id="q26"></a>
## Q26. How many ways to secure APIs? Explain JWT token in detail

### 📝 One-Liner
Secure APIs with: HTTPS, JWT/OAuth2, API Keys, mTLS certificates, rate limiting, CORS, input validation. JWT has 3 parts: Header (algo).Payload (claims).Signature (verification).

### 📖 How It Works
```
JWT Structure:
eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSIsInJvbGUiOiJBRE1JTiJ9.signature

Part 1 — HEADER (base64):          Part 2 — PAYLOAD (base64):         Part 3 — SIGNATURE:
{                                   {                                   HMACSHA256(
  "alg": "HS256",                    "sub": "user1",                     base64(header) + "." +
  "typ": "JWT"                       "role": "ADMIN",                    base64(payload),
}                                    "iat": 1711324800,                  secret_key
                                     "exp": 1711328400                 )
                                   }

Flow:
1. Login → POST /auth/login {username, password}
2. Server validates credentials
3. Server creates JWT (signs with secret key) → returns to client
4. Client stores JWT (httpOnly cookie or Auth header)
5. Subsequent requests: Authorization: Bearer <JWT>
6. Server verifies signature → extracts claims → authorize
```

### 🆚 vs. Token vs Session
| Aspect | JWT (Token-based) | Session-based |
|--------|-------------------|---------------|
| Storage | Client-side (cookie/header) | Server-side (memory/Redis) |
| Stateless | ✅ Server stores nothing | ❌ Server stores session |
| Scalability | ✅ Horizontal easy | ❌ Sticky sessions or shared store |
| Revocation | ❌ Hard (blacklist needed) | ✅ Delete from session store |
| Size | Larger (full payload) | Small (session ID only) |
| REST-friendly | ✅ Stateless by design | ❌ Stateful |

### ⚡ Remember
- JWT is **not encrypted** — it's base64 encoded (readable). Don't put secrets in payload
- Three parts: Header.Payload.Signature
- Signature ensures **integrity** — any tamper invalidates the token
- Secure transport: always use **HTTPS** — JWT over HTTP is insecure
- Revocation: short-lived access tokens + refresh tokens + Redis blacklist

---

<a id="q27"></a>
## Q27. How to return error details securely without exposing internal information?

### 📝 One-Liner
Return structured error responses with error code + user-friendly message; NEVER expose stack traces, class names, SQL queries, or internal file paths.

### 💻 Code
```java
// ❌ INSECURE — exposes internals to hackers
{
    "error": "org.hibernate.exception.SQLGrammarException",
    "message": "could not execute query: SELECT * FROM users WHERE id = '1' OR '1'='1'",
    "trace": "at com.app.repo.UserRepoImpl.findById(UserRepoImpl.java:42)..."
}

// ✅ SECURE — generic message, internal code for support
{
    "status": 500,
    "errorCode": "ERR_INTERNAL_001",
    "message": "An unexpected error occurred. Please contact support.",
    "timestamp": "2026-03-25T10:30:00",
    "traceId": "abc-123-def"  // correlation ID for internal log lookup
}
```
```java
@RestControllerAdvice
public class SecureExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<ErrorResponse> handleAll(Exception ex, HttpServletRequest req) {
        String traceId = MDC.get("traceId"); // from logging framework
        // Log full details INTERNALLY
        log.error("TraceId={} Path={} Error={}", traceId, req.getRequestURI(), ex.getMessage(), ex);
        // Return GENERIC response to client
        return ResponseEntity.status(500).body(new ErrorResponse(
            500, "ERR_INTERNAL", "An unexpected error occurred", traceId));
    }
}
```

### ⚡ Remember
- **Never expose**: stack traces, SQL errors, file paths, class names, server versions
- Return: error code + user-friendly message + trace ID for debugging
- Log full details **server-side** with correlation/trace ID
- Different error detail levels: DEV (verbose) → PROD (minimal)
- Configure `server.error.include-stacktrace=never` in Spring Boot production

---

<a id="q28"></a>
## Q28. Vault for secrets management — How are passwords stored and read?

### 📝 One-Liner
HashiCorp Vault (or cloud equivalents) stores secrets externally — Spring Cloud Vault auto-injects secrets as properties at startup, keeping credentials out of source code.

### 💻 Code
```yaml
# bootstrap.yml — Spring Cloud Vault integration
spring:
  cloud:
    vault:
      uri: https://vault.example.com:8200
      authentication: TOKEN
      token: ${VAULT_TOKEN}
      kv:
        backend: secret
        default-context: myapp
        application-name: myapp

# Vault stores: secret/myapp/database
# { "username": "admin", "password": "s3cr3t" }

# Accessed as normal Spring properties:
# ${database.username} → admin
# ${database.password} → s3cr3t
```
```java
@Service
public class DatabaseConfig {
    @Value("${database.username}")
    private String dbUsername; // injected from Vault, not properties file

    @Value("${database.password}")
    private String dbPassword; // never in source code or Git
}
```

### ⚡ Remember
- **Never commit secrets** to Git — use Vault, AWS Secrets Manager, Azure Key Vault
- Spring Cloud Vault auto-loads secrets as Spring properties at startup
- Vault supports: dynamic DB credentials, PKI certificates, encryption-as-a-service
- Kubernetes: also use K8s Secrets or External Secrets Operator

---

## Section G: Coding Challenges

---

<a id="q29"></a>
## Q29. Retrieve employees aged 11–30 using Spring Boot (filter by age range)

### 💻 Code
```java
// Repository — derived query
List<Employee> findByAgeBetween(int minAge, int maxAge);

// Service
public List<EmployeeDTO> getEmployeesInAgeRange(int min, int max) {
    return employeeRepository.findByAgeBetween(min, max).stream()
        .map(this::mapToDTO)
        .collect(Collectors.toList());
}

// Controller
@GetMapping("/employees")
public ResponseEntity<List<EmployeeDTO>> getByAgeRange(
        @RequestParam(defaultValue = "11") int minAge,
        @RequestParam(defaultValue = "30") int maxAge) {
    return ResponseEntity.ok(employeeService.getEmployeesInAgeRange(minAge, maxAge));
}

// Alternative: Stream-based filtering
List<Employee> filtered = employees.stream()
    .filter(e -> e.getAge() >= 11 && e.getAge() <= 30)
    .collect(Collectors.toList());
```

---

<a id="q30"></a>
## Q30. Sort a list by name, then by population (same name → lower population first)

### 💻 Code
```java
record City(String name, long population) {}

List<City> cities = List.of(
    new City("Mumbai", 20000000), new City("Delhi", 19000000),
    new City("Mumbai", 12000000), new City("Delhi", 25000000),
    new City("Chennai", 10000000)
);

// Sort: by name ASC, then by population ASC (same name → lower pop first)
List<City> sorted = cities.stream()
    .sorted(Comparator.comparing(City::name)
        .thenComparingLong(City::population))
    .collect(Collectors.toList());
// [Chennai-10M, Delhi-19M, Delhi-25M, Mumbai-12M, Mumbai-20M]
```

---

<a id="q31"></a>
## Q31. Find the second highest value from a list without sorting

### 💻 Code
```java
// Approach 1: Two-pass O(n)
public static int secondHighest(List<Integer> nums) {
    int first = Integer.MIN_VALUE, second = Integer.MIN_VALUE;
    for (int n : nums) {
        if (n > first) {
            second = first;
            first = n;
        } else if (n > second && n != first) {
            second = n;
        }
    }
    return second;
}

// Approach 2: Stream (still O(n) internally but less explicit)
int secondMax = nums.stream()
    .distinct()
    .reduce(new int[]{Integer.MIN_VALUE, Integer.MIN_VALUE}, (acc, n) -> {
        if (n > acc[0]) { acc[1] = acc[0]; acc[0] = n; }
        else if (n > acc[1]) { acc[1] = n; }
        return acc;
    }, (a, b) -> a)[1];

// Approach 3: TreeSet (sorted set, O(n))
TreeSet<Integer> set = new TreeSet<>(nums);
set.pollLast(); // remove highest
int secondHighest = set.last(); // second highest
```

### ⚡ Remember
- Without sorting = O(n) time, O(1) space with two-variable approach
- Handle edge cases: all same values, list size < 2
- Interview tip: mention time/space complexity upfront

---

<a id="q32"></a>
## Q32. Filter accounts by region using parallel stream

### 💻 Code
```java
// Accenture's exact question: filter East region from 1000+ accounts
List<Account> eastAccounts = accounts.parallelStream()
    .filter(a -> "East".equalsIgnoreCase(a.getRegion()))
    .collect(Collectors.toList());

// Note: parallelStream() only helps for large datasets (1000+ and CPU-intensive)
// For simple filtering, sequential stream is often faster due to overhead

// Better pattern with null safety:
List<Account> eastAccounts = accounts.stream()
    .filter(a -> a.getRegion() != null && "East".equalsIgnoreCase(a.getRegion()))
    .collect(Collectors.toList());
```

### ⚡ Remember
- `parallelStream()` uses `ForkJoinPool.commonPool()` — shared across entire app
- Only use for large datasets with CPU-intensive operations
- Always put the constant first in `equals()`: `"East".equalsIgnoreCase(...)` — avoids NPE

---

## Section H: DevOps, Deployment & Production

---

<a id="q33"></a>
## Q33. Agile methodology — Complete process explained

### 📝 One-Liner
Agile uses iterative sprints (1-4 weeks) with ceremonies: Sprint Planning → Daily Standup → Sprint Review → Retrospective, delivering working software incrementally.

### 📖 How It Works
```
Product Backlog → Sprint Planning → Sprint (2 weeks) → Sprint Review → Retro
                       │
                       ├── Daily Standup (15 min): What I did, What I'll do, Blockers
                       │
                       ├── Development + Testing (continuous)
                       │
                       ├── Code Review + PR Merge
                       │
                       └── Sprint Demo to stakeholders

Roles:
- Product Owner: prioritizes backlog, defines acceptance criteria
- Scrum Master: facilitates ceremonies, removes blockers
- Dev Team: self-organizing, cross-functional

Artifacts:
- Product Backlog: all features/stories prioritized
- Sprint Backlog: committed stories for current sprint
- Increment: working software at sprint end
```

### ⚡ Remember
- Story points: estimate effort (Fibonacci: 1, 2, 3, 5, 8, 13)
- Definition of Done: code reviewed, tested, merged, deployed to staging
- Velocity: average story points per sprint (used for planning)
- Retrospective: what went well, what to improve, action items

---

<a id="q34"></a>
## Q34. Jenkins Pipeline — How to create and execute? What are gates?

### 📝 One-Liner
Jenkins Pipeline is a code-defined (Jenkinsfile) CI/CD workflow with stages like Build → Test → Quality Gate → Security Scan → Deploy.

### 💻 Code
```groovy
// Jenkinsfile (Declarative Pipeline)
pipeline {
    agent any
    stages {
        stage('Build') {
            steps {
                sh 'mvn clean compile'
            }
        }
        stage('Test') {
            steps {
                sh 'mvn test'
            }
            post {
                always { junit 'target/surefire-reports/*.xml' }
            }
        }
        stage('Quality Gate') {
            steps {
                sh 'mvn sonar:sonar'
                // Gate: fail if code coverage < 80% or critical issues > 0
                waitForQualityGate abortPipeline: true
            }
        }
        stage('Security Scan') {
            steps {
                sh 'mvn dependency-check:check'
                // Gate: fail on CRITICAL/HIGH vulnerabilities
            }
        }
        stage('Deploy to Staging') {
            when { branch 'develop' }
            steps {
                sh 'kubectl apply -f k8s/staging/'
            }
        }
        stage('Deploy to Prod') {
            when { branch 'master' }
            input { message "Approve production deployment?" }
            steps {
                sh 'kubectl apply -f k8s/production/'
            }
        }
    }
}
```

### ⚡ Remember
- **Gates**: quality gates (SonarQube), security gates (OWASP dependency check), approval gates (manual)
- Declarative vs Scripted: Declarative is simpler, Scripted is more flexible
- `post { always { } success { } failure { } }` for cleanup/notifications
- Hotfix: branch from master → fix → PR to master → cherry-pick to develop

---

<a id="q35"></a>
## Q35. Production issue resolution — How to check logs, narrow down issues?

### 📝 One-Liner
Check logs with Splunk/ELK/Kibana, use trace IDs (Zipkin/Jaeger) to follow request flow, monitoring dashboards (Grafana) for metrics spikes, thread dumps for hangs.

### 📖 How It Works
```
Production Issue Resolution Flow:
1. DETECT: Alert from monitoring (Grafana/PagerDuty/CloudWatch)
   └── Metrics spike: latency, error rate, CPU/memory

2. TRIAGE: Check severity and impact
   ├── P1: Service down → all hands
   ├── P2: Degraded performance → team lead + dev
   └── P3: Minor issue → next sprint

3. DIAGNOSE:
   ├── Logs: Splunk/ELK → search by traceId, timestamp, error code
   ├── Traces: Zipkin/Jaeger → follow request across microservices
   ├── Metrics: Grafana → CPU, memory, DB connection pool, GC pauses
   ├── Thread dump: jstack → check for deadlocks, blocked threads
   └── Heap dump: jmap → check for memory leaks

4. FIX:
   ├── Quick fix: config change (feature flag, increase pool size)
   ├── Hotfix: branch from master → fix → test → deploy
   └── Rollback: if fix is risky, rollback to previous version

5. POST-MORTEM: RCA document, preventive measures, monitoring improvements
```

### 💻 Code
```bash
# Splunk query examples
index=myapp sourcetype=app-logs "ERROR" earliest=-1h
index=myapp traceId="abc-123-def" | table timestamp, service, message
index=myapp "OutOfMemoryError" OR "Connection pool exhausted"

# Kubernetes log check
kubectl logs <pod-name> --tail=500 -f
kubectl logs <pod-name> --previous  # crashed pod logs

# Thread dump
jstack <pid> > thread_dump.txt
# Look for: BLOCKED, WAITING, deadlock

# GC analysis
jstat -gcutil <pid> 1000  # GC stats every 1 second
```

### ⚡ Remember
- **Trace ID** is the single most important tool for microservice debugging
- Spring Boot Actuator `/health` and `/metrics` for quick status check
- `kubectl describe pod <name>` for K8s pod events and restart reasons
- Java profiling tools: VisualVM, JProfiler, async-profiler, Arthas

---

<a id="q36"></a>
## Q36. Hotfix process — Step by step

### 📖 How It Works
```
1. Branch from MASTER (not develop):
   git checkout master
   git checkout -b hotfix/JIRA-123-fix-payment-bug

2. Fix the issue + write test

3. Test locally + run existing test suite

4. PR to MASTER (fast-track review)
   - Code review by at least 1 senior
   - Run pipeline (build + test + security scan)

5. Merge to MASTER → deploy to production

6. Cherry-pick to DEVELOP:
   git checkout develop
   git cherry-pick <hotfix-commit-sha>

7. Post-mortem: update RCA doc, add monitoring/alert
```

### ⚡ Remember
- Hotfix branches from **master** (production code), not develop
- Must be cherry-picked back to develop to prevent regression
- Keep hotfixes minimal — fix only the issue, no refactoring
- Tag the release: `git tag v1.2.1` after hotfix merge

---

## Section I: Scenario-Based Questions

---

<a id="q37"></a>
## Q37. A class is in production and you need to add new properties — how to handle?

### 📝 One-Liner
Use backward-compatible changes: add new fields with defaults, use Flyway/Liquibase for DB migration, feature flags for gradual rollout, and API versioning if contract changes.

### 🔑 Quick Answer
**(1)** Add new fields with `@Column(nullable = true)` or default values — existing rows get null/default. **(2)** DB migration: Flyway `V2__add_new_columns.sql` — `ALTER TABLE ADD COLUMN`. **(3)** API versioning: `/api/v2/accounts` if response structure changes. **(4)** Serialization: Jackson ignores unknown fields by default → old clients unaffected. *(Production class mein naye fields add karo with defaults — purana data break nahi hoga, Flyway se schema migrate karo)*

### 💻 Code
```java
@Entity
public class Account {
    // Existing fields...
    private String accountNumber;
    private BigDecimal balance;

    // NEW fields — nullable with defaults
    @Column(nullable = true)
    private String email;  // null for existing records

    @Column(nullable = false, columnDefinition = "varchar(20) default 'STANDARD'")
    private String accountType = "STANDARD";  // default for existing records
}
```
```sql
-- Flyway: V2__add_email_and_type.sql
ALTER TABLE accounts ADD COLUMN email VARCHAR(255);
ALTER TABLE accounts ADD COLUMN account_type VARCHAR(20) DEFAULT 'STANDARD' NOT NULL;
```

### ⚡ Remember
- Never remove/rename existing columns in production without migration
- Use feature flags to enable new functionality gradually
- Jackson `@JsonIgnoreProperties(ignoreUnknown = true)` on DTOs for forward compatibility
- Blue-green deployment: run old + new version simultaneously during transition

---

<a id="q38"></a>
## Q38. Kafka is taking too long to respond but API should not be blocked — how to handle?

### 📝 One-Liner
Publish to Kafka asynchronously using `@Async` or `KafkaTemplate.send()` (which is already async) — return API response immediately without waiting for Kafka acknowledgment.

### 💻 Code
```java
@Service
public class OrderService {

    @Autowired private KafkaTemplate<String, OrderEvent> kafkaTemplate;
    @Autowired private OrderRepository orderRepository;

    public OrderResponse placeOrder(OrderRequest request) {
        // 1. Save to DB — synchronous (must succeed)
        Order order = orderRepository.save(mapToEntity(request));

        // 2. Publish to Kafka — fire-and-forget (async)
        kafkaTemplate.send("order-events", order.getId().toString(),
            new OrderEvent("ORDER_CREATED", order))
            .whenComplete((result, ex) -> {
                if (ex != null) {
                    log.error("Kafka publish failed for order {}: {}",
                        order.getId(), ex.getMessage());
                    // Save to retry table or dead-letter for later processing
                    retryQueueService.save(new FailedEvent("ORDER_CREATED", order));
                }
            });

        // 3. Return response immediately — don't wait for Kafka
        return new OrderResponse(order.getId(), "Order placed successfully");
    }
}
```

### ⚡ Remember
- `KafkaTemplate.send()` returns `CompletableFuture` — non-blocking by default
- Use callback (`whenComplete`) to handle failures asynchronously
- Implement retry/dead-letter pattern for failed publishes
- Pattern: DB write (sync) → Kafka publish (async) → respond immediately

---

<a id="q39"></a>
## Q39. Singleton bean receiving concurrent requests — how does it behave?

### 📝 One-Liner
Singleton bean is shared across all request threads — each request gets the same instance. If bean has mutable state, concurrent requests cause race conditions; stateless beans are safe.

### 🔑 Quick Answer
Spring's singleton scope means one instance handles ALL requests. Each HTTP request runs on a separate thread, all accessing the same bean. If the bean has no mutable instance fields (stateless), it's perfectly safe. If it has mutable state (counters, caches), threads interfere with each other. *(Singleton = ek instance, sab threads use karein — agar mutable state nahi hai toh safe hai, hai toh race condition hogi)*

### 💻 Code
```java
// ✅ SAFE — stateless (typical Spring service)
@Service
public class AccountService {
    private final AccountRepository repo; // immutable reference
    public Account findById(Long id) { return repo.findById(id).orElse(null); }
    // No mutable fields — 100 concurrent requests are fine
}

// ❌ UNSAFE — mutable state in singleton
@Service
public class CounterService {
    private int requestCount = 0; // shared across all threads!

    public void handleRequest() {
        requestCount++; // RACE CONDITION — lost updates
    }
}

// ✅ FIX — use AtomicInteger or ThreadLocal
@Service
public class SafeCounterService {
    private final AtomicInteger requestCount = new AtomicInteger(0);
    public void handleRequest() { requestCount.incrementAndGet(); }
}
```

### ⚡ Remember
- Spring services/repositories should be **stateless** — use method-local variables
- Request-scoped data: use method parameters, not instance fields
- If you need per-request state: use `@Scope("request")` or `ThreadLocal`
- Controller → Service → Repository chain: all singletons, all stateless = thread-safe

---

<a id="q40"></a>
## Q40. Dynamic repository queries — sometimes query by ID, sometimes by name, sometimes all fields

### 📝 One-Liner
Use Spring Data JPA Specifications (Criteria API) or QueryDSL to build dynamic queries based on which fields are provided.

### 💻 Code
```java
// Specification-based dynamic query
@Repository
public interface AccountRepository extends JpaRepository<Account, Long>,
        JpaSpecificationExecutor<Account> { }

public class AccountSpecifications {
    public static Specification<Account> withFilters(String name, Integer age, String region) {
        return (root, query, cb) -> {
            List<Predicate> predicates = new ArrayList<>();
            if (name != null) predicates.add(cb.like(cb.lower(root.get("name")),
                "%" + name.toLowerCase() + "%"));
            if (age != null) predicates.add(cb.equal(root.get("age"), age));
            if (region != null) predicates.add(cb.equal(root.get("region"), region));
            return cb.and(predicates.toArray(new Predicate[0]));
        };
    }
}

// Service
public List<Account> search(String name, Integer age, String region) {
    return accountRepository.findAll(AccountSpecifications.withFilters(name, age, region));
}

// Controller — all params optional
@GetMapping("/accounts/search")
public List<Account> search(
        @RequestParam(required = false) String name,
        @RequestParam(required = false) Integer age,
        @RequestParam(required = false) String region) {
    return accountService.search(name, age, region);
}
```

### ⚡ Remember
- `JpaSpecificationExecutor` + `Specification<T>` for dynamic WHERE clauses
- QueryDSL is another option (type-safe, generated Q-classes)
- `@Query` with SpEL: works for simple optional params but gets messy
- Spring Data REST: auto-generates CRUD endpoints with query params

---

<a id="q41"></a>
## Q41. Bean instantiation failed in Kubernetes — pod won't start. How to resolve?

### 📝 One-Liner
Check pod events (`kubectl describe pod`), container logs, missing env vars/secrets, config map issues, health probe failures, or missing external service connectivity.

### 📖 How It Works
```
Diagnosis Steps:
1. kubectl describe pod <name>
   └── Events section: CrashLoopBackOff, ImagePullBackOff, etc.

2. kubectl logs <pod-name>
   └── Spring Boot stack trace: BeanCreationException — which bean failed?

3. Common causes:
   ├── Missing env variable → ${DB_PASSWORD} not set in K8s Secret
   ├── External service unreachable → DB host, Vault, Config Server down
   ├── Wrong config map → pointing to wrong DB URL
   ├── Resource limits → OOMKilled (need more memory)
   ├── Health probe fails → readiness probe returns 503 before app is ready
   └── Dependency version mismatch → ClassNotFoundException

4. Fix examples:
   ├── kubectl edit secret <name> → fix missing password
   ├── kubectl edit configmap <name> → correct DB URL
   ├── Increase initialDelaySeconds on readiness probe
   └── kubectl set resources deployment/app --limits=memory=1Gi
```

### ⚡ Remember
- `BeanCreationException` → read the full cause chain — root cause is at the bottom
- Missing properties: `@Value` without default crashes startup → add `${key:default}`
- DB not ready: add `spring.datasource.hikari.initialization-fail-timeout=-1` + retry
- ConfigServer down: use `spring.cloud.config.fail-fast=false` with retry

---

<a id="q42"></a>
## Q42. DB connection pool exhaustion — reasons and fixes

### 📝 One-Liner
Pool exhaustion means all connections are in use and new requests wait/fail — caused by long-running queries, connection leaks, or pool too small for traffic.

### 🔑 Quick Answer
**Causes**: **(1)** Long-running queries holding connections. **(2)** Connection leak — not closing/returning connection properly. **(3)** Pool too small for concurrent load. **(4)** `@Transactional` on long-running method holds connection entire duration. **Fixes**: increase pool size, fix slow queries, add connection timeout, detect leaks. *(Connection pool exhaust hone ke karan: slow queries, connection leak, ya pool size chhota — pehle slow queries fix karo, phir pool tune karo)*

### 💻 Code
```yaml
# HikariCP pool configuration
spring:
  datasource:
    hikari:
      maximum-pool-size: 20          # increase based on load testing
      minimum-idle: 5
      connection-timeout: 30000      # 30s wait before failing
      idle-timeout: 600000           # 10 min idle before closing
      max-lifetime: 1800000          # 30 min max connection age
      leak-detection-threshold: 30000 # warn if connection held > 30s
```

### ⚡ Remember
- **Leak detection**: `leak-detection-threshold` logs warning with stack trace of leak source
- **Pool size formula**: `connections = (core_count * 2) + effective_spindle_count` (HikariCP wiki)
- `@Transactional` on controller method → holds connection for entire request lifecycle (bad!)
- Monitoring: Actuator `/metrics/hikaricp.connections.active` to track usage

---

<a id="q43"></a>
## Q43. REST vs RESTful vs GraphQL — Key differences

### 🆚 vs.
| Aspect | REST (concept) | RESTful (implementation) | GraphQL |
|--------|---------------|------------------------|---------|
| What | Architectural style | REST-compliant API | Query language for APIs |
| Data fetching | Fixed structure | Fixed endpoints | Client specifies exactly what data |
| Over-fetching | ✅ Common | ✅ Common | ❌ Client picks fields |
| Under-fetching | ✅ Multiple calls needed | ✅ Multiple calls needed | ❌ Single query |
| Endpoints | Multiple (/users, /posts) | Multiple | Single (/graphql) |
| Caching | HTTP caching (easy) | HTTP caching (easy) | Complex (needs Apollo) |
| Best for | CRUD APIs | Standard web APIs | Complex, nested data |

### 🔑 Quick Answer
**REST** = architectural principles (stateless, resource-based, HTTP methods). **RESTful** = an API that properly follows REST constraints (proper URIs, status codes, HATEOAS). **GraphQL** = client-defined queries — ask for exactly the data you need in one request. RESTful services may return too much (over-fetching) or too little (under-fetching) creating multiple roundtrips; GraphQL solves this. *(REST ek concept hai, RESTful uska correct implementation hai, GraphQL mein client decide karta hai ki kya data chahiye)*

### ⚡ Remember
- Most "REST APIs" are actually not fully RESTful (missing HATEOAS, improper status codes)
- GraphQL: single endpoint, POST with query body, schema-defined
- Use REST for simple CRUD; GraphQL for complex, nested, multi-relation queries
- REST is cached by HTTP infra (CDN, browsers); GraphQL needs custom caching

---

<a id="q44"></a>
## Q44. Transactions — Propagation types and Isolation levels

### 📝 One-Liner
Propagation defines HOW transactions interact (join existing? new? suspend?); Isolation defines WHAT data concurrent transactions can see.

### 🆚 vs.
**Propagation:**
| Type | Behavior |
|------|----------|
| REQUIRED (default) | Join existing TX, or create new |
| REQUIRES_NEW | Always create new TX (suspend existing) |
| NESTED | Create savepoint within existing TX |
| SUPPORTS | Use TX if exists, else run without |
| NOT_SUPPORTED | Suspend TX, run without |
| MANDATORY | Must run within existing TX, else throw |
| NEVER | Must NOT run within TX, else throw |

**Isolation:**
| Level | Dirty Read | Non-Repeatable | Phantom |
|-------|-----------|----------------|---------|
| READ_UNCOMMITTED | ✅ | ✅ | ✅ |
| READ_COMMITTED | ❌ | ✅ | ✅ |
| REPEATABLE_READ | ❌ | ❌ | ✅ |
| SERIALIZABLE | ❌ | ❌ | ❌ |

### 💻 Code
```java
@Transactional(propagation = Propagation.REQUIRED) // default
public void transferMoney(Long from, Long to, BigDecimal amount) {
    debit(from, amount);    // same TX
    credit(to, amount);     // same TX
    // both succeed or both rollback
}

@Transactional(propagation = Propagation.REQUIRES_NEW)
public void logAudit(AuditEvent event) {
    // separate TX — logged even if outer TX rolls back
    auditRepo.save(event);
}

@Transactional(isolation = Isolation.READ_COMMITTED) // prevents dirty reads
public Account getAccount(Long id) {
    return accountRepo.findById(id).orElseThrow();
}
```

### ⚡ Remember
- Default: `REQUIRED` propagation, `READ_COMMITTED` isolation (for most DBs)
- `REQUIRES_NEW` is useful for audit logs — must persist even if main TX fails
- Higher isolation = more data safety but less concurrency (more locking)
- `SERIALIZABLE` = safest but slowest (full table lock behavior)

---

<a id="q45"></a>
## Q45. Processing 1 million records efficiently — how would you design it?

### 📝 One-Liner
Use Spring Batch with chunk processing, database cursors, parallel steps, and partitioning — never load all records into memory at once.

### 🔑 Quick Answer
**Spring Batch** is the go-to: read in chunks (100-1000), process, write — keeps memory constant. Use `JpaCursorItemReader` (not `JpaPagingItemReader` for large sets). Enable partitioning for parallel processing across threads. For non-batch: use `Stream<T>` from repository with `@Transactional(readOnly = true)` to avoid loading all entities. *(1 million records ko ek baar mein load mat karo — chunks mein process karo Spring Batch se, ya stream use karo)*

### 💻 Code
```java
// Spring Batch — chunk processing
@Bean
public Step processStep(JobRepository jobRepository, PlatformTransactionManager txManager) {
    return new StepBuilder("processStep", jobRepository)
        .<Employee, EmployeeDTO>chunk(500, txManager) // 500 records per chunk
        .reader(reader())
        .processor(processor())
        .writer(writer())
        .taskExecutor(new SimpleAsyncTaskExecutor()) // parallel chunks
        .throttleLimit(4)
        .build();
}

// Stream-based approach (no Spring Batch)
@Repository
public interface EmployeeRepo extends JpaRepository<Employee, Long> {
    @QueryHints(@QueryHint(name = HINT_FETCH_SIZE, value = "500"))
    @Query("SELECT e FROM Employee e")
    Stream<Employee> streamAll(); // cursor-based, memory-efficient
}

@Transactional(readOnly = true)
public void processAllEmployees() {
    try (Stream<Employee> stream = employeeRepo.streamAll()) {
        stream.forEach(emp -> processEmployee(emp));
    }
}
```

### ⚡ Remember
- **Never**: `findAll()` for 1M records → `OutOfMemoryError`
- **Spring Batch**: chunk size 100-1000, partitioned across threads
- **JPA Stream**: `Stream<T>` with `@QueryHint(FETCH_SIZE)` for cursor-based reading
- **Pagination**: use keyset pagination (`WHERE id > lastId LIMIT 1000`) not `OFFSET`
- Monitor: GC pauses, heap usage, DB connection usage during processing

---

## 📊 Summary

| Section | Focus | Questions |
|---------|-------|-----------|
| A | Spring Boot REST Implementation | Q1–Q3 |
| B | Spring Core & Boot Concepts | Q4–Q12 |
| C | Java Core & Threading | Q13–Q18 |
| D | Microservices & Communication | Q19–Q21 |
| E | Database & SQL | Q22–Q25 |
| F | Security & Authentication | Q26–Q28 |
| G | Coding Challenges | Q29–Q32 |
| H | DevOps, Deployment & Production | Q33–Q36 |
| I | Scenario-Based Questions | Q37–Q45 |

**Key Takeaways from Accenture Interviews:**
- Multiple evaluators (Rohit, Kaushik, Anbu, Alok) — each with different focus areas
- Heavy emphasis on **practical implementation** (write full controller/service/repo/entity)
- Exception handling flow asked in great detail (custom exceptions + @RestControllerAdvice)
- Scenario-based questions dominate — production issues, Kafka delays, K8s failures
- Coding round: no IDE, output mandatory — practice writing compilable code
- DevOps knowledge expected: Jenkins, Kubernetes, log analysis, hotfix process
- Security questions: JWT internals, securing APIs, Vault usage
- Threading + concurrency asked repeatedly: race conditions, deadlocks, singleton thread safety
