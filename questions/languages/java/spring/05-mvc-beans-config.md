# 🍃 Spring Boot — DispatcherServlet, HTTP Methods, Bean Scopes & Annotations (Q1–Q4)

> **Source**: Capgemini + Spring Boot Interview Questions (4+ years)  
> **Coverage**: MVC request flow, PUT vs PATCH, bean scopes, @Bean vs @Component

---

<a id="q1"></a>
## Q1. How does DispatcherServlet work in Spring MVC? What is HandlerMapping?

### 📝 One-Liner
`DispatcherServlet` is the **front controller** that receives ALL HTTP requests and dispatches them to the correct controller method using `HandlerMapping` (URL → handler resolution).

### 🔑 Quick Answer
**DispatcherServlet** — single servlet registered by Spring Boot that intercepts all requests (`/`). Flow: **(1)** Receives HTTP request. **(2)** Consults `HandlerMapping` to find which controller method handles this URL + HTTP method. **(3)** Calls `HandlerAdapter` to invoke the controller method. **(4)** Controller returns response (or model+view). **(5)** If view → `ViewResolver` resolves the view name to a template. **(6)** Response sent back. **HandlerMapping** — maps request URL + method to a handler (controller method). `RequestMappingHandlerMapping` is the main implementation — it reads `@RequestMapping`, `@GetMapping`, `@PostMapping` etc. to build a URL → method map at startup. *(DispatcherServlet = front controller jo sab requests receive karta hai; HandlerMapping = URL se sahi controller method dhundhta hai)*

### 📖 How It Works (Detailed Explanation)

```
HTTP Request Flow through DispatcherServlet:

Client: GET /api/users/123
  │
  ▼
┌─────────────────────────┐
│   Servlet Container      │
│   (Tomcat)               │
└───────────┬─────────────┘
            │
            ▼
┌─────────────────────────────────────────────────┐
│   DispatcherServlet (Front Controller)           │
│                                                  │
│  1. HandlerMapping                               │
│     → "GET /api/users/{id}" → UserController     │
│        .getUserById(id)                          │
│                                                  │
│  2. HandlerAdapter                               │
│     → Resolves @PathVariable, @RequestBody       │
│     → Calls controller method                    │
│                                                  │
│  3. Controller returns ResponseEntity<User>      │
│                                                  │
│  4. HttpMessageConverter (Jackson)               │
│     → Converts User object → JSON                │
│                                                  │
│  5. Response: 200 OK + JSON body                 │
└─────────────────────────────────────────────────┘
```

**HandlerMapping implementations**: `RequestMappingHandlerMapping` (annotation-based — most common), `SimpleUrlHandlerMapping` (URL → bean mapping), `BeanNameUrlHandlerMapping` (bean name matches URL). **HandlerAdapter**: bridges DispatcherServlet and the actual handler. For `@Controller` methods, `RequestMappingHandlerAdapter` resolves method arguments (`@PathVariable`, `@RequestParam`, `@RequestBody`), invokes the method, and processes the return value. **Interceptors** (`HandlerInterceptor`) hook into the flow — `preHandle` (before controller), `postHandle` (after controller), `afterCompletion` (after response).

### 🗣️ Answering Approach
"DispatcherServlet is the front controller in Spring MVC — all HTTP requests go through it. When a request arrives, it first consults HandlerMapping to determine which controller method should handle the URL. RequestMappingHandlerMapping reads @GetMapping, @PostMapping annotations at startup to build a URL-to-method map. Once the handler is found, HandlerAdapter invokes the method after resolving all parameters — path variables, request body, query params. For REST APIs with @RestController, the return object is passed through HttpMessageConverters — Jackson converts it to JSON. The response goes back through the servlet. I can add cross-cutting logic using HandlerInterceptors for things like logging, authentication timing, or request correlation IDs."

### 💻 Code Example

```java
// ✅ Spring Boot auto-configures DispatcherServlet
// No manual setup needed — starter-web does it all

@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping("/{id}")   // HandlerMapping maps: GET /api/users/{id} → this method
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        User user = userService.findById(id);
        return ResponseEntity.ok(user);  // → Jackson → JSON response
    }

    @PostMapping
    public ResponseEntity<User> createUser(@Valid @RequestBody UserRequest request) {
        User user = userService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(user);
    }
}

// ✅ Custom HandlerInterceptor — cross-cutting logic
@Component
public class RequestTimingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
                            HttpServletResponse response, Object handler) {
        request.setAttribute("startTime", System.currentTimeMillis());
        return true;  // true = continue, false = stop
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
                               HttpServletResponse response,
                               Object handler, Exception ex) {
        long start = (long) request.getAttribute("startTime");
        long duration = System.currentTimeMillis() - start;
        log.info("{} {} → {}ms", request.getMethod(), request.getRequestURI(), duration);
    }
}

// Register interceptor
@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new RequestTimingInterceptor())
                .addPathPatterns("/api/**");
    }
}
```

### ⚠️ Common Pitfalls
- **404 for controller endpoint** — missing `@RequestMapping` or wrong URL pattern → check HandlerMapping debug logs
- **Interceptors don't catch Security exceptions** — Spring Security filters run BEFORE DispatcherServlet
- **Order matters**: Filters → Security → DispatcherServlet → Interceptors → Controller → Advice
- **No `@RestController`** — forgetting means no `@ResponseBody` → Spring tries to resolve a view name

### ⚡ Remember (Quick Recall)
- **DispatcherServlet** = front controller (all requests go through it)
- **HandlerMapping** = URL → controller method resolution
- **HandlerAdapter** = resolves params, invokes controller
- Flow: Request → Filter → DispatcherServlet → HandlerMapping → Interceptor → Controller → Response
- Spring Boot auto-configures everything via `starter-web`

---

<a id="q2"></a>
## Q2. What is the difference between PUT and PATCH in REST APIs?

### 📝 One-Liner
`PUT` **replaces the entire resource** (full update); `PATCH` **modifies only specified fields** (partial update).

### 🔑 Quick Answer
`PUT /users/1` — sends the **complete** resource representation. Replaces ALL fields — any omitted field becomes null/default. **Idempotent** (same request repeated = same result). `PATCH /users/1` — sends only the **fields to change**. Other fields remain untouched. Also **idempotent** when implemented correctly. **Use PUT** when the client has the complete resource. **Use PATCH** when updating one or two fields (e.g., just the email). In Spring: `@PutMapping` and `@PatchMapping`. *(PUT = poora resource replace karo; PATCH = sirf jo change karna hai woh bhejo)*

### 📖 How It Works (Detailed Explanation)

```
Current resource: { "name": "Alice", "email": "a@x.com", "phone": "123" }

PUT /users/1 → { "name": "Alice", "email": "new@x.com" }
Result:        { "name": "Alice", "email": "new@x.com", "phone": null }
               ↑ phone removed because not in PUT body!

PATCH /users/1 → { "email": "new@x.com" }
Result:          { "name": "Alice", "email": "new@x.com", "phone": "123" }
                  ↑ name and phone unchanged!
```

### 🗣️ Answering Approach
"PUT replaces the entire resource — the client sends the complete representation, and any field not included is set to null or default. PATCH applies a partial update — only the fields included in the request body are modified, other fields remain unchanged. Both are idempotent. I use PUT when the client has the full object — like a form where all fields are editable. I use PATCH for targeted updates — like changing just a user's status or email. In Spring Boot, implementing PATCH requires handling null carefully — I can't just use @RequestBody with the entity because I can't distinguish between 'field omitted' and 'field set to null'. I typically use a Map or a DTO with Optional fields for PATCH requests."

### 💻 Code Example

```java
// ✅ PUT — full replacement
@PutMapping("/{id}")
public ResponseEntity<User> updateUser(@PathVariable Long id,
                                       @Valid @RequestBody UserRequest request) {
    User user = userService.findById(id);
    user.setName(request.getName());       // all fields updated
    user.setEmail(request.getEmail());
    user.setPhone(request.getPhone());     // if null in request → becomes null!
    return ResponseEntity.ok(userService.save(user));
}

// ✅ PATCH — partial update (Map approach)
@PatchMapping("/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id,
                                      @RequestBody Map<String, Object> updates) {
    User user = userService.findById(id);
    updates.forEach((key, value) -> {
        switch (key) {
            case "name"  -> user.setName((String) value);
            case "email" -> user.setEmail((String) value);
            case "phone" -> user.setPhone((String) value);
        }
    });
    return ResponseEntity.ok(userService.save(user));
}

// ✅ PATCH — partial update (DTO approach, cleaner)
public record UserPatchRequest(
    Optional<String> name,
    Optional<String> email,
    Optional<String> phone
) {}

@PatchMapping("/{id}")
public ResponseEntity<User> patchUser(@PathVariable Long id,
                                      @RequestBody UserPatchRequest patch) {
    User user = userService.findById(id);
    patch.name().ifPresent(user::setName);
    patch.email().ifPresent(user::setEmail);
    patch.phone().ifPresent(user::setPhone);
    return ResponseEntity.ok(userService.save(user));
}
```

### 🆚 Comparison Table

| Aspect | PUT | PATCH |
|--------|-----|-------|
| Update scope | **Full resource** (replace) | **Partial** (only specified fields) |
| Missing fields | Set to null/default | Left unchanged |
| Idempotent | ✅ Yes | ✅ Yes (when properly designed) |
| Request body | Complete representation | Only changed fields |
| Spring annotation | `@PutMapping` | `@PatchMapping` |
| Use case | Full form submit | Single-field update |

### ⚡ Remember (Quick Recall)
- **PUT** = replace everything (omitted fields → null)
- **PATCH** = update only sent fields (other fields untouched)
- Both are idempotent
- PATCH implementation challenge: distinguish "not sent" vs "explicitly null"
- Use `Map<String, Object>` or DTO with `Optional` fields for PATCH

---

<a id="q3"></a>
## Q3. What are bean scopes in Spring? What is the default scope?

### 📝 One-Liner
Bean scope defines **how many instances** Spring creates — default is **singleton** (one per ApplicationContext); other scopes: prototype, request, session, application.

### 🔑 Quick Answer
**singleton** (default) — ONE instance per Spring container, shared by all injection points. **prototype** — NEW instance every time the bean is requested (`getBean()` or `@Autowired` into a new parent). **request** — one instance per HTTP request (web only). **session** — one instance per HTTP session (web only). **application** — one instance per `ServletContext`. Declare with `@Scope("prototype")` or `@RequestScope`. **Gotcha**: injecting a prototype bean into a singleton → you get ONE prototype instance forever → use `ObjectFactory<T>` or `@Lookup` for fresh instances. *(Default = singleton — poore application mein ek hi instance; Prototype = har baar naya)*

### 📖 How It Works (Detailed Explanation)

```
Singleton (default):
  @Service OrderService  → created ONCE at startup
  
  Controller A  ──→ ┐
  Controller B  ──→ ├── SAME OrderService instance
  Scheduler    ──→ ┘

Prototype:
  @Scope("prototype")
  @Component ReportGenerator

  controller.getBean(ReportGenerator.class)  → NEW instance #1
  controller.getBean(ReportGenerator.class)  → NEW instance #2
  
Request scope:
  @RequestScope
  @Component RequestContext
  
  Request 1 → RequestContext #1  (destroyed after response)
  Request 2 → RequestContext #2  (separate instance)
```

### 🗣️ Answering Approach
"The default bean scope in Spring is singleton — one instance per ApplicationContext, created eagerly at startup and shared across the entire application. This is why Spring beans should be stateless — all request threads share the same instance. Prototype scope creates a new instance every time the bean is requested, but Spring doesn't manage the lifecycle after creation — no @PreDestroy. For web applications, request scope gives one instance per HTTP request, and session scope per user session. The classic pitfall is injecting a prototype bean into a singleton — you only get one instance because the singleton is created once with one prototype injected. The fix is ObjectFactory or @Lookup annotation to get fresh instances on each call."

### 💻 Code Example

```java
// ✅ Singleton (default) — one instance, shared
@Service  // @Scope("singleton") is implicit
public class OrderService {
    // ⚠️ Must be stateless! All threads share this instance.
    private final OrderRepository repo;  // OK — injected once, used by all
    // ❌ private List<Order> cache = new ArrayList<>(); → race condition!
}

// ✅ Prototype — new instance each time
@Component
@Scope("prototype")
public class ReportGenerator {
    private final List<String> data = new ArrayList<>();  // safe — not shared
    public void addData(String row) { data.add(row); }
}

// ✅ Request scope — one per HTTP request
@Component
@RequestScope   // shortcut for @Scope(value = "request", proxyMode = ScopedProxyMode.TARGET_CLASS)
public class RequestContext {
    private String correlationId;
    private Instant startTime = Instant.now();
    // unique per request — different threads get different instances
}

// ❌ GOTCHA: Prototype in Singleton — broken!
@Service
public class OrderService {
    private final ReportGenerator generator;  // injected ONCE → same instance forever!

    public OrderService(ReportGenerator generator) {
        this.generator = generator;  // ❌ This prototype is now effectively a singleton
    }
}

// ✅ FIX: Use ObjectFactory for fresh prototype instances
@Service
public class OrderService {
    private final ObjectFactory<ReportGenerator> generatorFactory;

    public OrderService(ObjectFactory<ReportGenerator> generatorFactory) {
        this.generatorFactory = generatorFactory;
    }

    public Report generateReport() {
        ReportGenerator gen = generatorFactory.getObject();  // ✅ new instance each time
        gen.addData("...");
        return gen.build();
    }
}

// ✅ Alternative fix: @Lookup
@Service
public abstract class OrderService {
    @Lookup
    protected abstract ReportGenerator createGenerator();  // Spring overrides this

    public Report generateReport() {
        ReportGenerator gen = createGenerator();  // ✅ new prototype each time
        return gen.build();
    }
}
```

### 🆚 Comparison Table

| Scope | Instances | Created | Destroyed | Use Case |
|-------|----------|---------|-----------|----------|
| **singleton** (default) | 1 per container | Startup (eager) | Container shutdown | Stateless services |
| **prototype** | New per request | On demand (lazy) | NOT managed by Spring | Stateful, non-shared |
| **request** | 1 per HTTP request | Request start | Request end | Request-scoped data |
| **session** | 1 per HTTP session | Session creation | Session expiry | User-specific data |
| **application** | 1 per ServletContext | App start | App shutdown | Application-wide config |

### ⚡ Remember (Quick Recall)
- **Default = singleton** (one instance, shared, must be stateless)
- Prototype = new instance per injection/getBean, but Spring won't destroy it
- **Prototype in Singleton trap** → use `ObjectFactory<T>` or `@Lookup`
- Web scopes: `@RequestScope`, `@SessionScope` (require proxy mode)
- Singleton beans should be **thread-safe** (all request threads share them)

---

<a id="q4"></a>
## Q4. What is the difference between @Bean and @Component?

### 📝 One-Liner
`@Component` is a **class-level** annotation for auto-detected beans via classpath scanning; `@Bean` is a **method-level** annotation in `@Configuration` classes for manual bean creation.

### 🔑 Quick Answer
`@Component` (+ `@Service`, `@Repository`, `@Controller`) — annotate a class → Spring auto-detects it during `@ComponentScan` → creates a bean. **You own the class.** `@Bean` — annotate a method in a `@Configuration` class → method return value becomes a bean. **You don't own the class** (third-party libraries) or need custom instantiation logic. `@Bean` gives you full control: constructor args, conditional creation, factory methods. `@Component` is simpler: just annotate and go. *(Component = class pe lagao, auto-scan se bean ban jaayega; Bean = method pe lagao, khud create karke do)*

### 📖 How It Works (Detailed Explanation)

```
@Component — auto-detection:
┌──────────────────────────────────┐
│ @Service                          │  ← Spring scans, finds this
│ public class OrderService {       │
│   // Spring creates instance      │
│ }                                 │
└──────────────────────────────────┘

@Bean — manual creation:
┌──────────────────────────────────┐
│ @Configuration                    │
│ public class AppConfig {          │
│                                   │
│   @Bean                           │
│   public RestTemplate restTemplate() {│
│     RestTemplate rt = new RestTemplate();│
│     rt.setConnectTimeout(5000);   │  ← custom config
│     return rt;                    │  ← this object becomes the bean
│   }                               │
│ }                                 │
└──────────────────────────────────┘
```

**When to use @Bean**: (1) Third-party classes you can't annotate (RestTemplate, ObjectMapper, DataSource). (2) Need customization during creation (set timeouts, add interceptors). (3) Conditional bean creation with `@ConditionalOnProperty`. (4) Multiple beans of same type with `@Qualifier`. **When to use @Component**: your own classes where the default constructor or auto-wired constructor suffices.

### 🗣️ Answering Approach
"@Component is a class-level annotation that tells Spring to auto-detect and register the class as a bean through component scanning. I use it for my own classes — services, repositories, controllers. @Bean is a method-level annotation in a @Configuration class where I manually instantiate and configure the bean. I use it for third-party classes that I can't annotate — like configuring a RestTemplate with specific timeouts and interceptors, or an ObjectMapper with custom serialization settings. The key distinction is ownership: if I own the class, I use @Component; if I don't or need custom creation logic, I use @Bean. Both register a bean in the Spring container and support dependency injection."

### 💻 Code Example

```java
// ✅ @Component — your own classes (auto-detected)
@Service  // @Service = @Component + semantic meaning
public class OrderService {
    private final OrderRepository repo;
    public OrderService(OrderRepository repo) { this.repo = repo; }
}

@Repository  // @Repository = @Component + exception translation
public interface OrderRepository extends JpaRepository<Order, Long> { }

// ✅ @Bean — third-party classes / custom config
@Configuration
public class AppConfig {

    @Bean   // RestTemplate is not your class — can't put @Component on it
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .connectTimeout(Duration.ofSeconds(5))
            .readTimeout(Duration.ofSeconds(10))
            .build();
    }

    @Bean
    public ObjectMapper objectMapper() {
        return new ObjectMapper()
            .registerModule(new JavaTimeModule())
            .disable(SerializationFeature.WRITE_DATES_AS_TIMESTAMPS);
    }

    @Bean("primaryDataSource")   // named bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSource primaryDataSource() {
        return DataSourceBuilder.create().build();
    }

    @Bean("secondaryDataSource")  // multiple beans of same type
    @ConfigurationProperties("spring.datasource.secondary")
    public DataSource secondaryDataSource() {
        return DataSourceBuilder.create().build();
    }
}

// ❌ Can't do this — you don't own RestTemplate class:
// @Component
// public class RestTemplate { ... }  ← can't modify library class!
```

### 🆚 Comparison Table

| Aspect | @Component | @Bean |
|--------|-----------|-------|
| Level | Class | Method (in @Configuration) |
| Detection | Auto (component scan) | Manual (explicit method) |
| Class ownership | Your class | Any class (third-party) |
| Customization | Limited (constructor) | Full (any factory logic) |
| Multiple beans same type | ❌ Awkward | ✅ Easy (`@Bean("name")`) |
| Variants | @Service, @Repository, @Controller | — |
| DI | Auto-wired constructor | Method parameters injected |

### ⚡ Remember (Quick Recall)
- **@Component** = "scan and register this class" (your code)
- **@Bean** = "register the return value of this method" (any code)
- Third-party → `@Bean`; your class → `@Component`
- `@Bean` goes in `@Configuration` classes
- Both produce beans managed by Spring container

### 🔗 Follow-up Topics
- [Q9 in spring/01 → @Component vs @Service vs @Repository](01-spring-framework-internals.md#q9)
- [Q3 → Bean scopes (applies to both @Bean and @Component)](#q3)
- `@Configuration` vs `@Component` (full vs lite mode proxy)
