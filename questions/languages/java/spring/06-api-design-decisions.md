# 🍃 Spring Boot — API Layer Design Decisions (Q1–Q4)

> **Source**: Spring Boot Project "Why Did You Choose This?" Interview Questions  
> **Coverage**: @RestController vs @Controller, DTOs vs entities, @Valid, async vs sync endpoints

---

<a id="q1"></a>
## Q1. Why did you choose @RestController instead of @Controller in this API?

### 📝 One-Liner
`@RestController` = `@Controller` + `@ResponseBody` on every method — it tells Spring to **serialize return objects directly to JSON/XML** instead of resolving a view name.

### 🔑 Quick Answer
`@Controller` was designed for **MVC web apps** where methods return a **view name** (like `"home"`) resolved by `ViewResolver` → Thymeleaf/JSP template. `@RestController` was introduced in Spring 4 specifically for **REST APIs** — it puts `@ResponseBody` on every method so the return value is written directly to the HTTP response body via `HttpMessageConverter` (Jackson → JSON). In my project, all endpoints are REST APIs returning JSON — using `@Controller` would force me to add `@ResponseBody` on every single method. `@RestController` is the cleaner, correct choice. *(Agar sirf JSON API banana hai toh @RestController sahi hai — @Controller view templates ke liye hai)*

### 📖 How It Works (Detailed Explanation)

```
@Controller (MVC — View Resolution):
  return "users"  → ViewResolver → users.html template → HTML response

@RestController (REST API — Direct Serialization):
  return userList  → HttpMessageConverter (Jackson) → JSON response

Under the hood:
  @RestController = @Controller + @ResponseBody
```

**When you'd still use `@Controller`**: (1) Server-side rendered pages (Thymeleaf, JSP). (2) Hybrid apps where some endpoints return views AND some return JSON — use `@Controller` class-level + selective `@ResponseBody` on JSON methods. (3) Redirect/forward scenarios (`"redirect:/login"`). **Why `@RestController` for APIs**: (1) Every method auto-serializes — no noisy `@ResponseBody` everywhere. (2) Makes intent crystal clear — this class serves REST endpoints. (3) Content negotiation still works — Jackson for JSON, JAXB for XML based on `Accept` header.

### 🗣️ Answering Approach
"I chose @RestController because this project is a pure REST API — every endpoint returns JSON, never a view template. @RestController combines @Controller and @ResponseBody, so every method's return value is automatically serialized to JSON by Jackson. If I used @Controller, I'd have to add @ResponseBody on every single method, which is just noise. The only time I'd use @Controller is if I had a server-side rendered UI with Thymeleaf or needed hybrid endpoints that return both views and JSON. Since this is a microservice with only REST endpoints, @RestController is the natural and clean choice."

### 💻 Code Example

```java
// ✅ @RestController — correct for REST APIs
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id);  // Jackson → JSON automatically
    }

    @PostMapping
    public ResponseEntity<UserDTO> createUser(@Valid @RequestBody CreateUserRequest req) {
        UserDTO user = userService.create(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}

// ❌ @Controller without @ResponseBody — Spring looks for a view!
@Controller
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id);
        // ERROR! Spring tries to resolve "UserDTO@3f2c1a" as a view name → 404/500
    }
}

// ✅ @Controller WITH @ResponseBody — works, but noisy
@Controller
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    @ResponseBody    // ← must add on every method
    public UserDTO getUser(@PathVariable Long id) {
        return userService.findById(id);
    }
}

// ✅ Hybrid: @Controller for mixed view + JSON endpoints
@Controller
public class HybridController {

    @GetMapping("/dashboard")
    public String dashboard(Model model) {
        model.addAttribute("stats", statsService.get());
        return "dashboard";  // → ViewResolver → dashboard.html
    }

    @GetMapping("/api/stats")
    @ResponseBody  // Only this method returns JSON
    public StatsDTO apiStats() {
        return statsService.get();
    }
}
```

### ⚠️ Common Pitfalls
- **Using `@Controller` for REST API** — return value treated as view name → `Circular view path` or `404`
- **Forgetting Jackson on classpath** — `@RestController` needs `jackson-databind` (included by `spring-boot-starter-web`)
- **Returning `String` from `@RestController`** — returns the string literal as response body, does NOT resolve a view
- **XML support** — add `jackson-dataformat-xml` dependency if clients send `Accept: application/xml`

### 🆚 @RestController vs @Controller

| Aspect | `@RestController` | `@Controller` |
|--------|-------------------|---------------|
| **Purpose** | REST APIs (JSON/XML) | MVC views (HTML templates) |
| **Return handling** | Serialized to response body | Resolved as view name |
| **`@ResponseBody`** | Implicit on all methods | Must add per method |
| **Use when** | Pure API / microservice | Thymeleaf/JSP server-rendered UI |
| **Content type** | `application/json` (default) | `text/html` |

### 🎯 Tricky Follow-up Questions
- **"Can you mix views and JSON in one controller?"** → Yes, use `@Controller` + selective `@ResponseBody` on JSON methods
- **"What if you need to return HTML from a REST controller?"** → Return `ResponseEntity<String>` with `Content-Type: text/html` header
- **"How does Spring decide JSON vs XML?"** → Content negotiation based on `Accept` header + available `HttpMessageConverter`s

### ⚡ Remember (Quick Recall)
- `@RestController` = `@Controller` + `@ResponseBody`
- REST APIs → `@RestController` always
- View templates → `@Controller`
- Hybrid → `@Controller` + selective `@ResponseBody`

### 🔗 Related Topics
- [DispatcherServlet & HandlerMapping](05-mvc-beans-config.md#q1)
- [HttpMessageConverters, Content Negotiation](../architecture/01-api-design-microservices.md)

---

<a id="q2"></a>
## Q2. Why is this endpoint returning DTOs instead of entities?

### 📝 One-Liner
Returning **JPA entities directly** from APIs exposes internal database schema, causes lazy-loading exceptions, creates **tight coupling**, and leaks sensitive fields — **DTOs decouple API contract from persistence model**.

### 🔑 Quick Answer
**(1) Security** — entities may contain fields you don't want exposed (`password`, `internalNotes`, `auditColumns`). DTOs expose only what the client needs. **(2) Decoupling** — if you return entities, every DB schema change breaks your API contract. DTOs let the DB evolve independently. **(3) LazyInitializationException** — returning an entity with lazy associations outside a transaction → crash. DTOs are plain POJOs with no proxy baggage. **(4) Performance** — entities may have 20 fields; the API only needs 5. DTOs prevent over-fetching. **(5) API versioning** — you can have `UserV1DTO` and `UserV2DTO` without touching the entity. *(Entity directly bhejne se DB schema expose hota hai, lazy exceptions aate hain, aur API tightly coupled ho jaata hai — DTO se sab clean rehta hai)*

### 📖 How It Works (Detailed Explanation)

```
WITHOUT DTOs (bad):
  Entity (DB schema) ────────────► JSON Response
  - All 20 fields exposed
  - LazyInit on relationships
  - Schema change = API break
  - Internal fields leak

WITH DTOs (good):
  Entity (DB schema) → Mapper → DTO (API contract) → JSON Response
  - Only needed fields
  - No Hibernate proxies
  - DB can change freely
  - Clean API versioning
```

**DTO mapping approaches**: (1) **Manual mapping** — constructor or static factory in DTO (`UserDTO.from(User entity)`). (2) **MapStruct** — compile-time code generation, zero reflection. (3) **ModelMapper** — reflection-based auto-mapping (slower). (4) **Spring Data Projections** — interface/class projections in repository queries (DTO at query level → no entity created). **In my project**: I use MapStruct for complex mappings and manual constructors for simple DTOs. For read-heavy endpoints, I use Spring Data projections to build DTOs directly from queries — no entity materialization.

### 🗣️ Answering Approach
"I never return entities from REST endpoints — I always use DTOs. There are four main reasons. First, security: entities might have sensitive fields like password hash or internal audit flags that shouldn't be in an API response. DTOs expose only what the client needs. Second, decoupling: if I return the entity directly, my API contract is tied to my database schema. Any column rename or new field breaks the API. DTOs give me a buffer. Third, performance: the entity might have 20 fields and lazy associations; the API only needs 5 fields. DTOs prevent over-fetching and avoid LazyInitializationException. Fourth, versioning: I can create UserV1DTO and UserV2DTO while the entity stays the same. For mapping, I use MapStruct which generates code at compile time — zero runtime overhead."

### 💻 Code Example

```java
// ✅ Entity — persistence concern only
@Entity
@Table(name = "users")
public class User {
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    private String name;
    private String email;
    private String passwordHash;       // ⚠️ NEVER expose!
    private String internalNotes;       // ⚠️ internal field
    private LocalDateTime createdAt;
    private LocalDateTime updatedAt;

    @OneToMany(mappedBy = "user", fetch = FetchType.LAZY)
    private List<Order> orders;         // ⚠️ lazy loaded
}

// ✅ DTO — API contract only
public record UserDTO(Long id, String name, String email) {

    public static UserDTO from(User entity) {
        return new UserDTO(entity.getId(), entity.getName(), entity.getEmail());
    }
}

// ✅ Request DTO — input validation
public record CreateUserRequest(
    @NotBlank String name,
    @Email String email,
    @Size(min = 8) String password
) {}

// ✅ Controller uses DTOs, never entities
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable Long id) {
        User entity = userService.findById(id);
        return UserDTO.from(entity);   // ← entity → DTO conversion
    }

    @PostMapping
    public ResponseEntity<UserDTO> create(@Valid @RequestBody CreateUserRequest req) {
        User entity = userService.create(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(UserDTO.from(entity));
    }
}

// ✅ MapStruct — zero-overhead compile-time mapper
@Mapper(componentModel = "spring")
public interface UserMapper {
    UserDTO toDTO(User entity);
    User toEntity(CreateUserRequest request);
}

// ✅ Spring Data Projection — DTO at query level (no entity loaded)
public interface UserSummary {
    Long getId();
    String getName();
    String getEmail();
}

@Repository
public interface UserRepository extends JpaRepository<User, Long> {
    List<UserSummary> findAllProjectedBy();  // SELECT id, name, email — no entity!
}
```

### ⚠️ Common Pitfalls
- **Returning entity directly** → Jackson serializes lazy fields → triggers extra queries or `LazyInitializationException`
- **Using `@JsonIgnore` instead of DTOs** → fragile, easy to forget on new fields, still loads full entity
- **Over-mapping** → creating separate DTOs for every tiny variation is overkill; share DTOs where contracts overlap
- **Circular references** → User → Order → User → `StackOverflow` if serializing entities with bidirectional relations

### 🆚 DTO vs Entity Exposure

| Aspect | Return Entity | Return DTO |
|--------|--------------|------------|
| **Security** | Leaks internal fields | Exposes only selected fields |
| **Coupling** | API tied to DB schema | Independent evolution |
| **Lazy loading** | LazyInitException risk | No Hibernate proxies |
| **Performance** | Over-fetching all columns | Fetch only what's needed |
| **Versioning** | Hard to version API | Easy V1/V2 DTOs |
| **Boilerplate** | Less code | More code (use MapStruct) |

### 🎯 Tricky Follow-up Questions
- **"Isn't creating DTOs extra boilerplate?"** → Yes, but MapStruct/records make it minimal. The trade-off is well worth the decoupling and safety
- **"What about GraphQL — do you still need DTOs?"** → GraphQL handles field selection, but DTOs still help for input validation and domain boundaries
- **"Can Spring Data projections replace DTOs entirely?"** → For read ops yes, but you still need request DTOs for input validation

### ⚡ Remember (Quick Recall)
- **Never return entities from REST APIs** — use DTOs
- 4 reasons: **Security, Decoupling, Lazy-loading safety, Performance**
- MapStruct = compile-time, zero overhead
- Spring Data projections = DTO at query level

### 🔗 Related Topics
- [Lazy vs Eager loading](../../database/01-jpa-sql-transactions.md)
- [JPA entity states](../../database/04-hibernate-cache-states.md#q2)
- [API design patterns](../architecture/01-api-design-microservices.md)

---

<a id="q3"></a>
## Q3. Why are you using validation annotations like @Valid?

### 📝 One-Liner
`@Valid` + Bean Validation annotations (`@NotBlank`, `@Email`, `@Size`) let Spring **validate request payloads at the controller boundary** — fail-fast before bad data reaches service/database layer.

### 🔑 Quick Answer
Without `@Valid`, bad input flows into the service layer, hits the database, and you get cryptic SQL constraint violations or NPEs deep in business logic. **Bean Validation** moves validation to the **entry point** — the controller — using declarative annotations on the request DTO. Spring's `MethodArgumentNotValidResolver` catches violations and throws `MethodArgumentNotValidException`, which my `@ControllerAdvice` converts to a clean `400 Bad Request` with field-level error messages. **Why declarative over manual checks?**: (1) No boilerplate `if/else` validation code. (2) Self-documenting — annotations on the DTO show the contract. (3) Reusable across endpoints. (4) Standard (JSR 380 / Jakarta Validation). *(Request validation controller pe hi honi chahiye — DB tak pahuche bina galat data ko rok do)*

### 📖 How It Works (Detailed Explanation)

```
Request Flow with @Valid:

Client sends POST /api/users  { "name": "", "email": "bad" }
  │
  ▼
Controller: @Valid @RequestBody CreateUserRequest
  │
  ▼
Bean Validation runs BEFORE method body executes:
  - @NotBlank name → FAIL (empty string)
  - @Email email  → FAIL (invalid format)
  │
  ▼
MethodArgumentNotValidException thrown
  │
  ▼
@ControllerAdvice catches → 400 Bad Request
{
  "status": 400,
  "errors": [
    { "field": "name",  "message": "must not be blank" },
    { "field": "email", "message": "must be a valid email" }
  ]
}

✅ Service layer never touched. Database never touched.
```

**Validation layers**: (1) **Controller layer** (`@Valid`) — format, presence, size. (2) **Service layer** — business rules (e.g., "email must be unique"). (3) **Database layer** — constraints as last safety net. **Nested validation**: `@Valid` on a nested object triggers recursive validation. **Groups**: `@Validated(OnCreate.class)` for different validation sets per operation. **Custom validators**: implement `ConstraintValidator<A, T>` for complex rules.

### 🗣️ Answering Approach
"I use @Valid on all request DTOs at the controller level because it's the first line of defense — I want to reject bad input immediately, before it reaches the service layer or database. The annotations on the DTO serve double duty: they validate AND document the contract. For example, @NotBlank on name means 'required, non-empty' — anyone reading the DTO immediately understands the constraint. When validation fails, Spring throws MethodArgumentNotValidException, which my global @ControllerAdvice catches and converts to a structured 400 response with field-level error messages. Without this, bad data would either hit the database and throw a cryptic constraint violation, or I'd need manual if-else checks in every controller method. Bean Validation makes it declarative, reusable, and standard."

### 💻 Code Example

```java
// ✅ Request DTO with validation annotations
public record CreateUserRequest(
    @NotBlank(message = "Name is required")
    String name,

    @NotBlank @Email(message = "Valid email required")
    String email,

    @NotNull @Size(min = 8, max = 100, message = "Password must be 8-100 chars")
    String password,

    @Min(value = 18, message = "Must be at least 18")
    Integer age,

    @Valid    // ← triggers nested validation
    AddressDTO address
) {}

public record AddressDTO(
    @NotBlank String street,
    @NotBlank String city,
    @Pattern(regexp = "^\\d{6}$", message = "PIN must be 6 digits")
    String pin
) {}

// ✅ Controller — @Valid triggers validation before method body
@RestController
@RequestMapping("/api/users")
public class UserController {

    @PostMapping
    public ResponseEntity<UserDTO> create(@Valid @RequestBody CreateUserRequest req) {
        // This code ONLY runs if ALL validations pass
        UserDTO user = userService.create(req);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}

// ✅ @ControllerAdvice handles validation errors globally
@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(MethodArgumentNotValidException.class)
    public ResponseEntity<ErrorResponse> handleValidation(MethodArgumentNotValidException ex) {
        List<FieldError> errors = ex.getBindingResult().getFieldErrors().stream()
            .map(f -> new FieldError(f.getField(), f.getDefaultMessage()))
            .toList();
        return ResponseEntity.badRequest()
            .body(new ErrorResponse(400, "Validation failed", errors));
    }

    record FieldError(String field, String message) {}
    record ErrorResponse(int status, String message, List<FieldError> errors) {}
}

// ✅ Custom validator — business-specific validation
@Target(ElementType.FIELD)
@Retention(RetentionPolicy.RUNTIME)
@Constraint(validatedBy = UniqueEmailValidator.class)
public @interface UniqueEmail {
    String message() default "Email already registered";
    Class<?>[] groups() default {};
    Class<? extends Payload>[] payload() default {};
}

@Component
public class UniqueEmailValidator implements ConstraintValidator<UniqueEmail, String> {
    private final UserRepository userRepository;

    public UniqueEmailValidator(UserRepository userRepository) {
        this.userRepository = userRepository;
    }

    @Override
    public boolean isValid(String email, ConstraintValidatorContext ctx) {
        return email != null && !userRepository.existsByEmail(email);
    }
}

// ✅ Validation groups — different rules for create vs update
public interface OnCreate {}
public interface OnUpdate {}

public record UserRequest(
    @Null(groups = OnCreate.class)       // ID must be null on create
    @NotNull(groups = OnUpdate.class)    // ID required on update
    Long id,

    @NotBlank(groups = {OnCreate.class, OnUpdate.class})
    String name
) {}

@PostMapping
public ResponseEntity<UserDTO> create(
        @Validated(OnCreate.class) @RequestBody UserRequest req) { ... }

@PutMapping("/{id}")
public ResponseEntity<UserDTO> update(
        @Validated(OnUpdate.class) @RequestBody UserRequest req) { ... }
```

### ⚠️ Common Pitfalls
- **`@Valid` vs `@Validated`** — `@Valid` is JSR standard (works on `@RequestBody`); `@Validated` is Spring extension (supports groups + method-level on `@PathVariable`)
- **Forgetting `@Valid` on nested objects** — nested DTO fields won't be validated without `@Valid` annotation on the field
- **`@NotNull` vs `@NotBlank` vs `@NotEmpty`** — `@NotNull`: not null. `@NotEmpty`: not null + not empty. `@NotBlank`: not null + not empty + not whitespace
- **Missing `spring-boot-starter-validation`** in Spring Boot 2.3+ — validation is no longer auto-included

### 🆚 Declarative Validation vs Manual Checks

| Aspect | `@Valid` + Annotations | Manual `if/else` |
|--------|----------------------|-------------------|
| **Boilerplate** | Minimal — annotations | Verbose — every field checked |
| **Reusability** | Same DTO validated everywhere | Logic copied per method |
| **Documentation** | Self-documenting annotations | Rules hidden in code |
| **Standard** | JSR 380 / Jakarta | Custom per project |
| **Error format** | Automatic `FieldError` | Must build manually |
| **Complex rules** | Custom `ConstraintValidator` | Direct Java logic |

### 🎯 Tricky Follow-up Questions
- **"Where do you validate business rules like 'email must be unique'?"** → Service layer, not Bean Validation — DB-dependent checks don't belong on DTOs (or use custom `ConstraintValidator` with care)
- **"How do you validate path variables?"** → Add `@Validated` on the controller class + `@Min` / `@Positive` on `@PathVariable` params
- **"What about programmatic validation?"** → Inject `Validator` bean, call `validator.validate(object)` manually in service layer

### ⚡ Remember (Quick Recall)
- `@Valid` = controller-level, fail-fast, declarative validation
- **DTO annotations** = validate AND document the contract
- `@ControllerAdvice` catches `MethodArgumentNotValidException` → structured 400
- `@Valid` on nested fields for recursive validation
- `@Validated` for groups + method-level param validation

### 🔗 Related Topics
- [Global Exception Handling with @ControllerAdvice](01-spring-framework-internals.md)
- [DTOs vs Entities](#q2)
- [Spring Security + Input Validation](03-enterprise-practices.md)

---

<a id="q4"></a>
## Q4. Why is this API call asynchronous instead of synchronous?

### 📝 One-Liner
You make an API endpoint **async** when it triggers **long-running work** (email, report generation, external calls) but the **client doesn't need to wait** for the result — free up the request thread and respond immediately.

### 🔑 Quick Answer
Synchronous: client sends request → thread blocked until all work completes → response. If the work takes 30 seconds (PDF generation, email sending, batch processing), the client waits 30 seconds and the server thread is held hostage. **Async approach**: the endpoint accepts the request, queues the work to `@Async` / message queue, and **returns 202 Accepted immediately**. The work completes in background. Client can poll a status endpoint or get a callback/webhook. **When to use async**: (1) Work takes >2-3 seconds. (2) Client doesn't need the result in the same response. (3) You want to protect Tomcat's thread pool from being exhausted by slow operations. *(Jab client ko turant result ki zaroorat nahi hai aur kaam lamba hai — tab async karo, thread free karo)*

### 📖 How It Works (Detailed Explanation)

```
SYNCHRONOUS (blocking):
Client → Request → Thread BLOCKED 30s → Response
  ⚠️ Tomcat thread held for 30 seconds
  ⚠️ 200 Tomcat threads = max 200 concurrent slow requests

ASYNCHRONOUS (non-blocking):
Client → Request → Enqueue work → 202 Accepted (instant)
  Background thread picks up work → completes later
  Client polls GET /status/{jobId} or receives webhook
  ✅ Tomcat thread freed immediately
  ✅ Scalable — handles thousands of concurrent requests
```

**Async patterns in Spring Boot**: (1) **`@Async` + `CompletableFuture`** — simplest, thread pool managed by Spring. (2) **Message queue (Kafka/RabbitMQ)** — durable, survives restarts, distributed. (3) **`WebClient` reactive calls** — non-blocking HTTP calls to external services. (4) **`DeferredResult` / `Callable`** — servlet-level async (frees Tomcat thread but still blocks a worker). **My project uses**: `@Async` for lightweight background tasks (email, notifications). Kafka for heavy processing (report generation, data sync). WebClient for non-blocking inter-service HTTP calls.

### 🗣️ Answering Approach
"I made this endpoint async because it triggers a long-running operation — like sending a confirmation email or generating a PDF report — and the client doesn't need the result in the same response. If I kept it synchronous, the Tomcat thread would be blocked for seconds, and with only 200 default threads, we'd quickly run out under load. Instead, the endpoint validates the request, enqueues the work, and returns 202 Accepted immediately with a job ID. The client can poll a status endpoint if needed. For lightweight background tasks I use @Async with a custom thread pool. For heavier, durable work like report generation, I publish to Kafka so the work survives even if the server restarts. The key decision factor is: does the client need the result right now? If no, go async."

### 💻 Code Example

```java
// ✅ Pattern 1: @Async — lightweight background task
@Configuration
@EnableAsync
public class AsyncConfig {

    @Bean("emailExecutor")
    public Executor emailExecutor() {
        ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
        executor.setCorePoolSize(5);
        executor.setMaxPoolSize(10);
        executor.setQueueCapacity(100);
        executor.setThreadNamePrefix("email-");
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        executor.initialize();
        return executor;
    }
}

@Service
public class NotificationService {

    @Async("emailExecutor")  // ← runs in background thread pool
    public CompletableFuture<Void> sendWelcomeEmail(String email, String name) {
        // Takes 2-5 seconds — no need to block the API thread
        emailClient.send(email, "Welcome", buildTemplate(name));
        return CompletableFuture.completedFuture(null);
    }
}

// ✅ Controller — return 202 immediately
@RestController
@RequestMapping("/api/users")
public class UserController {

    private final UserService userService;
    private final NotificationService notificationService;

    @PostMapping
    public ResponseEntity<UserDTO> register(@Valid @RequestBody CreateUserRequest req) {
        UserDTO user = userService.create(req);

        // Fire-and-forget — async email, don't block response
        notificationService.sendWelcomeEmail(user.email(), user.name());

        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}

// ✅ Pattern 2: Async with job tracking (long-running)
@RestController
@RequestMapping("/api/reports")
public class ReportController {

    @PostMapping
    public ResponseEntity<JobStatusDTO> generateReport(
            @Valid @RequestBody ReportRequest req) {
        String jobId = reportService.enqueueReportGeneration(req);
        return ResponseEntity.accepted()    // ← 202 Accepted
            .body(new JobStatusDTO(jobId, "QUEUED",
                "/api/reports/status/" + jobId));
    }

    @GetMapping("/status/{jobId}")
    public JobStatusDTO checkStatus(@PathVariable String jobId) {
        return reportService.getJobStatus(jobId);
        // Returns: { "jobId": "abc", "status": "COMPLETED", "downloadUrl": "/files/abc.pdf" }
    }
}

// ✅ Pattern 3: WebClient — non-blocking external call
@Service
public class PaymentService {

    private final WebClient webClient;

    // Non-blocking call to payment gateway
    public Mono<PaymentResult> processPayment(PaymentRequest req) {
        return webClient.post()
            .uri("/payments")
            .bodyValue(req)
            .retrieve()
            .bodyToMono(PaymentResult.class)
            .timeout(Duration.ofSeconds(5))
            .onErrorResume(ex -> Mono.just(PaymentResult.failed(ex.getMessage())));
    }
}
```

### ⚠️ Common Pitfalls
- **`@Async` self-invocation** — calling an @Async method from the same class bypasses the proxy → runs synchronously
- **No `@EnableAsync`** — @Async does nothing without this on a @Configuration class
- **Default `SimpleAsyncTaskExecutor`** — creates a new thread per call (no pooling); always configure a custom `ThreadPoolTaskExecutor`
- **Lost exceptions** — if @Async method throws and you don't check the `CompletableFuture`, exceptions are silently swallowed
- **Transaction context not propagated** — @Async runs in a different thread → `@Transactional` context from the caller is NOT available

### 🆚 Sync vs Async Decision Matrix

| Factor | Synchronous | Asynchronous |
|--------|------------|-------------|
| **Client needs result immediately** | ✅ Yes | ❌ No (poll/webhook) |
| **Operation < 1 second** | ✅ Sync is fine | Overkill |
| **Operation > 3 seconds** | ❌ Blocks thread | ✅ Free thread fast |
| **Must survive server restart** | Not applicable | Use message queue |
| **Thread pool pressure** | High under load | Low — returns fast |
| **Complexity** | Simple | Job tracking, error handling |
| **Retry on failure** | Client retries | Queue/scheduler retries |

### 🎯 Tricky Follow-up Questions
- **"How do you handle errors in async methods?"** → `AsyncUncaughtExceptionHandler` for void methods; `.exceptionally()` or `.handle()` for `CompletableFuture` return types
- **"How does the client know when async work is done?"** → Polling (`GET /status/{jobId}`), WebSocket push, or webhook callback
- **"@Async vs Kafka — when do you choose which?"** → @Async for lightweight in-process tasks; Kafka for durable, distributed, high-throughput work that must survive restarts

### ⚡ Remember (Quick Recall)
- **Async when**: work is long + client doesn't need immediate result
- `@Async` → easy, in-process background tasks (configure pool!)
- Message queue → durable, distributed heavy work
- Return **202 Accepted** + job ID for long-running endpoints
- Never @Async self-invocation — proxy bypass

### 🔗 Related Topics
- [@Async internals](../multithreading/09-spring-multithreading.md)
- [Kafka vs REST for inter-service communication](../architecture/01-api-design-microservices.md)
- [Connection pool & timeout](../../production-debugging/01-jvm-memory-performance.md)
