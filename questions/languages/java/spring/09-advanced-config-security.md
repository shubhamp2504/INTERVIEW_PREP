# 🍃 Spring Boot — Advanced Configuration, Security & Internals (Q1–Q6)

> **Source**: Spring Boot Interview Questions (Advanced) + American Express (2-4 YOE, March 2026)  
> **Coverage**: Performance tuning, logging, Spring Security, filters/interceptors/AOP, ApplicationContext vs BeanFactory, multiple data sources

---

<a id="q1"></a>
## Q1. How do you improve performance in a Spring Boot application?

### 📝 One-Liner
Performance tuning covers **multiple layers**: JVM flags, connection pool sizing, lazy initialization, caching, async processing, database query optimization, and response compression — not a single magic switch.

### 🔑 Quick Answer
**(1) Connection Pool** — tune HikariCP (`maximum-pool-size`, `minimum-idle`). **(2) JPA/Hibernate** — avoid N+1 queries (`JOIN FETCH`, `@EntityGraph`), use DTO projections, enable batch inserts. **(3) Caching** — `@Cacheable` for repeated reads (Redis/Caffeine). **(4) Async** — `@Async` for non-blocking operations; WebClient for reactive HTTP calls. **(5) Lazy Initialization** — `spring.main.lazy-initialization=true` reduces startup time. **(6) JVM Tuning** — G1GC/ZGC, heap sizing (`-Xms`/`-Xmx`), GC logging. **(7) Response Compression** — `server.compression.enabled=true`. **(8) Database Indexing** — proper indexes on query columns. **(9) Pagination** — never `findAll()` on large tables; use `Pageable`. **(10) Virtual Threads** — Java 21+ with `spring.threads.virtual.enabled=true`. *(Performance kisi ek jagah se nahi — pool, cache, query, JVM sab jagah tune karna padta hai)*

### 📖 How It Works (Detailed Explanation)

```
Performance Tuning Layers:

┌───────────────────────────────────────────┐
│ 1. JVM          │ Heap, GC, Virtual Threads│
├───────────────────────────────────────────┤
│ 2. Framework    │ Lazy init, compression,  │
│                 │ Tomcat threads            │
├───────────────────────────────────────────┤
│ 3. Application  │ @Async, @Cacheable,      │
│                 │ pagination, DTOs          │
├───────────────────────────────────────────┤
│ 4. Database     │ HikariCP, indexes, N+1,  │
│                 │ batch inserts, projections│
├───────────────────────────────────────────┤
│ 5. Infrastructure│ CDN, Redis, load balance │
└───────────────────────────────────────────┘
```

### 🗣️ Interview Script
"Performance tuning in Spring Boot is a multi-layer effort. At the database layer, I tune HikariCP pool size, use DTO projections instead of loading full entities, add proper indexes, and fix N+1 queries with JOIN FETCH or @EntityGraph. At the application layer, I use @Cacheable with Caffeine or Redis for repeated reads, @Async for non-blocking operations like email sending, and pagination for large datasets. At the framework level, I enable response compression, configure Tomcat thread pool, and use lazy initialization to speed up startup. At the JVM level, I size the heap appropriately, choose G1GC or ZGC, and enable GC logging for production diagnostics. In my current project, the biggest wins came from fixing N+1 queries — one endpoint went from 200ms to 15ms — and adding Redis caching for frequently accessed reference data."

### 💻 Code Example

```yaml
# ✅ Spring Boot performance configuration
server:
  tomcat:
    threads:
      max: 200               # Tomcat worker threads
      min-spare: 20
  compression:
    enabled: true             # GZIP response compression
    min-response-size: 1024   # compress responses > 1KB
    mime-types: application/json,text/html,text/xml

spring:
  main:
    lazy-initialization: true  # beans created on first use (faster startup)
  datasource:
    hikari:
      maximum-pool-size: 15
      minimum-idle: 5
  jpa:
    open-in-view: false        # CRITICAL: disable OSIV to prevent lazy queries in view
    properties:
      hibernate:
        default_batch_fetch_size: 16    # batch lazy loads
        jdbc.batch_size: 50             # batch inserts
        order_inserts: true
        order_updates: true
  threads:
    virtual:
      enabled: true            # Java 21+ virtual threads for Tomcat
```

```java
// ✅ Fix N+1 with @EntityGraph
@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {

    // ❌ N+1: loads orders, then 1 query per order.items
    List<Order> findByUserId(Long userId);

    // ✅ Single query with JOIN FETCH
    @EntityGraph(attributePaths = {"items", "items.product"})
    List<Order> findWithItemsByUserId(Long userId);
}

// ✅ Caching for repeated reads
@Service
public class ProductService {

    @Cacheable(value = "products", key = "#id")
    public ProductDTO findById(Long id) {
        return productRepository.findProjectedById(id);
    }

    @CacheEvict(value = "products", key = "#id")
    public void update(Long id, UpdateRequest req) { ... }
}

// ✅ DTO projection — don't load full entity
public interface ProductSummary {
    Long getId();
    String getName();
    BigDecimal getPrice();
}
// SELECT id, name, price FROM products — no entity overhead
```

### ⚠️ Common Pitfalls
- **Premature optimization** — profile first (`/actuator/metrics`, flame graphs), then fix bottlenecks
- **`open-in-view=true`** (default!) — keeps Hibernate session open in view layer → lazy queries in controllers → performance killer
- **Caching everything** — cache stale data causes bugs; cache selectively with proper eviction
- **Pool size too large** — connections consume memory on DB server; `(2 × CPU) + spindles` is the formula

### ⚡ Remember (Quick Recall)
- **Profile first** → identify bottleneck → optimize that layer
- Top wins: N+1 fixes, caching, connection pool tuning, `open-in-view=false`
- Lazy init = faster startup, virtual threads = Java 21+ scalability
- Pagination + DTO projections = reduce data transferred

### 🔗 Related Topics
- [HikariCP pool sizing](../../production-debugging/01-jvm-memory-performance.md)
- [Caching patterns](../architecture/02-caching-architecture-patterns.md)
- [JVM performance tuning](../core/03-jvm-performance-tuning.md)

---

<a id="q2"></a>
## Q2. How do you configure logging in a Spring Boot application?

### 📝 One-Liner
Spring Boot uses **SLF4J + Logback** by default — configure via `application.yml` for simple setups or `logback-spring.xml` for advanced patterns like JSON structured logging, file rotation, and per-profile log levels.

### 🔑 Quick Answer
**(1) Default stack**: SLF4J (API) + Logback (implementation) — included in `spring-boot-starter`. **(2) Basic config in application.yml**: `logging.level.com.myapp=DEBUG`, `logging.file.name=app.log`. **(3) Advanced: `logback-spring.xml`** — supports Spring profiles, async appenders, JSON layout, file rolling policies. **(4) Production best practices**: WARN/ERROR for root, structured JSON logging for ELK/Splunk, async appender for performance, MDC for request tracing. **(5) Alternatives**: Log4j2 (exclude Logback, add `spring-boot-starter-log4j2`). *(Spring Boot me Logback default hai — basic config application.yml me, advanced config logback-spring.xml me karo)*

### 📖 How It Works (Detailed Explanation)

```
Logging Architecture:

Your Code → SLF4J API → Logback (default) → Console / File / ELK
         Logger.info("msg")
         Logger.error("msg", ex)

Log Levels (priority order):
  TRACE → DEBUG → INFO → WARN → ERROR

Set level=WARN → only WARN and ERROR logged
Set level=DEBUG → DEBUG, INFO, WARN, ERROR logged
```

### 🗣️ Interview Script
"Spring Boot uses SLF4J as the logging API with Logback as the default implementation — both are included in the base starter. For basic configuration, I use application.yml to set log levels per package — DEBUG for my app, WARN for third-party libraries. For production, I use a logback-spring.xml file with Spring profile-specific configuration. In development, I use a colored console appender. In production, I use structured JSON logging so logs can be parsed by ELK or Splunk, with an async appender to prevent logging from blocking request threads. I also use MDC — Mapped Diagnostic Context — to attach a correlation ID to every log line, which is essential for tracing requests across microservices. One thing I always ensure: never log at DEBUG level in production, and always use async appenders for high-throughput services."

### 💻 Code Example

```yaml
# ✅ Basic logging in application.yml
logging:
  level:
    root: WARN                          # default for all packages
    com.myapp: INFO                     # my code
    com.myapp.service: DEBUG            # more verbose for services
    org.springframework.web: INFO
    org.hibernate.SQL: DEBUG            # see SQL queries (dev only)
  file:
    name: logs/application.log          # write to file
  logback:
    rollingpolicy:
      max-file-size: 10MB              # rotate at 10MB
      max-history: 30                  # keep 30 days
```

```xml
<!-- ✅ Advanced: logback-spring.xml — profile-specific -->
<configuration>

    <!-- Dev profile: colored console -->
    <springProfile name="dev">
        <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
            <encoder>
                <pattern>%d{HH:mm:ss} %highlight(%-5level) [%thread] %cyan(%logger{36}) - %msg%n</pattern>
            </encoder>
        </appender>
        <root level="DEBUG">
            <appender-ref ref="CONSOLE"/>
        </root>
    </springProfile>

    <!-- Prod profile: JSON structured + async + file rotation -->
    <springProfile name="prod">
        <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
            <file>logs/app.log</file>
            <rollingPolicy class="ch.qos.logback.core.rolling.SizeAndTimeBasedRollingPolicy">
                <fileNamePattern>logs/app.%d{yyyy-MM-dd}.%i.log.gz</fileNamePattern>
                <maxFileSize>50MB</maxFileSize>
                <maxHistory>30</maxHistory>
                <totalSizeCap>1GB</totalSizeCap>
            </rollingPolicy>
            <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>
        </appender>

        <appender name="ASYNC" class="ch.qos.logback.classic.AsyncAppender">
            <queueSize>1024</queueSize>
            <discardingThreshold>0</discardingThreshold>
            <appender-ref ref="FILE"/>
        </appender>

        <root level="WARN">
            <appender-ref ref="ASYNC"/>
        </root>
        <logger name="com.myapp" level="INFO"/>
    </springProfile>
</configuration>
```

```java
// ✅ Using SLF4J in code
@Service
@Slf4j  // Lombok — generates: private static final Logger log = LoggerFactory.getLogger(...)
public class OrderService {

    public OrderDTO placeOrder(CreateOrderRequest req) {
        log.info("Placing order for user={}, items={}", req.userId(), req.items().size());
        try {
            Order order = processOrder(req);
            log.info("Order placed successfully: orderId={}", order.getId());
            return OrderMapper.toDTO(order);
        } catch (Exception ex) {
            log.error("Order failed for user={}: {}", req.userId(), ex.getMessage(), ex);
            throw ex;
        }
    }
}

// ✅ MDC for request correlation (filter-based)
@Component
public class CorrelationFilter extends OncePerRequestFilter {
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws ServletException, IOException {
        String correlationId = Optional.ofNullable(request.getHeader("X-Correlation-ID"))
            .orElse(UUID.randomUUID().toString());
        MDC.put("correlationId", correlationId);
        response.setHeader("X-Correlation-ID", correlationId);
        try {
            chain.doFilter(request, response);
        } finally {
            MDC.clear();
        }
    }
}
```

### ⚠️ Common Pitfalls
- **DEBUG in production** — floods logs, kills disk I/O, slows app; always WARN/ERROR for root in prod
- **Sync file appender under load** — blocking I/O; use `AsyncAppender` wrapper
- **Logging sensitive data** — passwords, tokens, PII in logs → security violation; scrub or mask
- **`toString()` in log params** — `log.debug("user={}", user)` → `user.toString()` called even if DEBUG is disabled; use `{}` placeholder (SLF4J evaluates lazily)

### ⚡ Remember (Quick Recall)
- Default: **SLF4J + Logback** (included automatically)
- Simple: `logging.level.*` in `application.yml`
- Advanced: `logback-spring.xml` with `<springProfile>` tags
- Production: **JSON + async appender + MDC correlation IDs**
- Never DEBUG in prod, always async for high-throughput

### 🔗 Related Topics
- [Production logging problems](../../production-debugging/04-services-ops-infra.md)
- [Observability & ELK](../../../cloud-devops/01-observability-alerting.md)
- [Environment profiles](07-project-infrastructure-decisions.md#q2)

---

<a id="q3"></a>
## Q3. What is Spring Security and how does it work?

### 📝 One-Liner
Spring Security is a **framework for authentication and authorization** in Spring apps — it intercepts every request through a **filter chain**, verifies identity (authn), and checks permissions (authz).

### 🔑 Quick Answer
Spring Security works via a **chain of servlet filters** (called `SecurityFilterChain`) that sit before your controllers. **(1) Authentication** — verify who the user is (username/password, JWT, OAuth2). **(2) Authorization** — verify what the user can do (`hasRole('ADMIN')`, `hasAuthority('READ_USERS')`). **Key components**: `SecurityFilterChain` (defines filter chain per URL pattern), `AuthenticationManager` (coordinates authentication), `UserDetailsService` (loads user from DB), `PasswordEncoder` (BCrypt), `SecurityContext` (holds authenticated principal). **Flow**: Request → Security Filters → Authentication → Authorization → Controller. *(Spring Security = filter chain jo har request pe authentication aur authorization check karta hai)*

### 📖 How It Works (Detailed Explanation)

```
Request Flow through Spring Security:

Client: GET /api/users (with JWT token)
  │
  ▼
┌─────────────────────────────────────────┐
│ Security Filter Chain (15+ filters)      │
│                                         │
│  1. CorsFilter                          │
│  2. CsrfFilter (disabled for REST APIs) │
│  3. UsernamePasswordAuthFilter          │
│     (or JwtAuthFilter — custom)         │
│  4. ExceptionTranslationFilter          │
│  5. AuthorizationFilter                 │
│     → checks @PreAuthorize / hasRole()  │
└─────────────┬───────────────────────────┘
              │ ✅ Authenticated + Authorized
              ▼
┌─────────────────────────┐
│   DispatcherServlet     │ → Controller → Service → Response
└─────────────────────────┘
```

### 🗣️ Interview Script
"Spring Security is an authentication and authorization framework that works through a chain of servlet filters. When a request arrives, it passes through about 15 security filters before reaching the DispatcherServlet. For authentication, I typically implement JWT-based auth: a custom JwtAuthenticationFilter extracts the token from the Authorization header, validates it, and sets the SecurityContext with the authenticated principal. For authorization, I use @PreAuthorize annotations on controller or service methods — like hasRole('ADMIN') or hasAuthority('WRITE_ORDERS'). The configuration is done via SecurityFilterChain bean where I define which URLs are public, which require authentication, and where my custom filters sit in the chain. I always disable CSRF for stateless REST APIs, enable CORS for frontend domains, and hash passwords with BCrypt."

### 💻 Code Example

```java
// ✅ Spring Security configuration (Spring Boot 3.x / Spring Security 6)
@Configuration
@EnableWebSecurity
@EnableMethodSecurity   // enables @PreAuthorize
public class SecurityConfig {

    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())           // stateless REST API → no CSRF
            .cors(cors -> cors.configurationSource(corsConfig()))
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**").permitAll()     // login/register public
                .requestMatchers("/actuator/health").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated()                    // everything else needs auth
            )
            .addFilterBefore(jwtAuthFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }

    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();   // hash passwords, never store plain text
    }
}

// ✅ Custom JWT filter
@Component
public class JwtAuthenticationFilter extends OncePerRequestFilter {

    private final JwtService jwtService;
    private final UserDetailsService userDetailsService;

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {

        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }

        String token = authHeader.substring(7);
        String username = jwtService.extractUsername(token);

        if (username != null && SecurityContextHolder.getContext().getAuthentication() == null) {
            UserDetails userDetails = userDetailsService.loadUserByUsername(username);
            if (jwtService.isTokenValid(token, userDetails)) {
                var authToken = new UsernamePasswordAuthenticationToken(
                    userDetails, null, userDetails.getAuthorities());
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}

// ✅ Method-level authorization
@RestController
@RequestMapping("/api/users")
public class UserController {

    @GetMapping
    @PreAuthorize("hasRole('ADMIN')")        // only ADMINs
    public List<UserDTO> getAllUsers() { ... }

    @GetMapping("/me")
    @PreAuthorize("isAuthenticated()")       // any logged-in user
    public UserDTO getCurrentUser(@AuthenticationPrincipal UserDetails principal) {
        return userService.findByUsername(principal.getUsername());
    }
}
```

### ⚠️ Common Pitfalls
- **CSRF disabled without reason** — disable ONLY for stateless REST APIs with token auth; form-based apps need CSRF
- **Storing plain text passwords** — always BCrypt; `PasswordEncoder` is mandatory
- **Filter order** — custom JWT filter must be added `before` UsernamePasswordAuthenticationFilter
- **`@PreAuthorize` not working** — forgot `@EnableMethodSecurity` on config class

### 🆚 Authentication vs Authorization

| Aspect | Authentication (AuthN) | Authorization (AuthZ) |
|--------|----------------------|---------------------|
| **Question** | WHO are you? | WHAT can you do? |
| **Mechanism** | JWT, OAuth2, username/password | Roles, authorities, permissions |
| **Spring** | `AuthenticationManager` | `@PreAuthorize`, `hasRole()` |
| **Failure** | 401 Unauthorized | 403 Forbidden |

### ⚡ Remember (Quick Recall)
- **Security Filter Chain** — 15+ filters before DispatcherServlet
- AuthN = who (JWT/OAuth2) → AuthZ = what (roles/authorities)
- `SecurityFilterChain` bean replaces old `WebSecurityConfigurerAdapter`
- `@EnableMethodSecurity` + `@PreAuthorize` for method-level control
- BCrypt for passwords, stateless sessions for REST APIs

### 🔗 Related Topics
- [JWT authentication flow](../architecture/01-api-design-microservices.md)
- [Secure REST APIs](03-enterprise-practices.md)
- [Filters vs Interceptors vs AOP](#q4)

---

<a id="q4"></a>
## Q4. How do filters, interceptors, and AOP differ in Spring Boot?

### 📝 One-Liner
**Filters** work at the **Servlet level** (before Spring), **Interceptors** work at the **Spring MVC level** (around controllers), and **AOP** works at the **method level** (any bean) — different layers, different use cases.

### 🔑 Quick Answer
**(1) Servlet Filter** (`javax.servlet.Filter`) — runs before/after DispatcherServlet. Has access to raw `HttpServletRequest/Response`. Use for: CORS, security, logging, compression, request wrapping. Works on ALL requests (including static resources). **(2) HandlerInterceptor** — runs inside Spring MVC, around controller methods. Has access to handler info (method, annotations). Use for: auth checks, request timing, audit logging, modifying ModelAndView. **(3) AOP** (`@Aspect`) — works on any Spring bean method, not tied to web layer. Uses pointcuts for method matching. Use for: transaction management (`@Transactional`), logging, retries, caching. **Execution order**: Filter → DispatcherServlet → Interceptor.preHandle → Controller → Interceptor.postHandle → AOP wraps any bean method. *(Filter = servlet level, Interceptor = Spring MVC level, AOP = kisi bhi bean ke method level — teen alag layer hain)*

### 📖 How It Works (Detailed Explanation)

```
Request Lifecycle — Where each operates:

Client Request
  │
  ▼
┌─────────────────────┐
│ Servlet FILTER       │ ← Raw HttpServletRequest/Response
│ (CORS, Security,     │    Before Spring even sees the request
│  Logging, Compress)  │    Can block request entirely
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ DispatcherServlet    │
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ INTERCEPTOR          │ ← Spring MVC context available
│  preHandle()         │    Knows which controller method
│                      │    Can access handler annotations
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Controller Method    │ ◄── AOP @Around wraps this
│ (+ Service, Repo)    │     AOP wraps ANY bean method
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ INTERCEPTOR          │
│  postHandle()        │
│  afterCompletion()   │
└─────────┬───────────┘
          ▼
┌─────────────────────┐
│ Servlet FILTER       │ ← after-processing
│  (response complete) │
└─────────────────────┘
```

### 🗣️ Interview Script
"These three operate at different layers. Servlet Filters are the lowest level — they intercept HTTP requests before they even reach Spring's DispatcherServlet. I use them for cross-cutting HTTP concerns like CORS, request logging, compression, and security. They work on raw HttpServletRequest and Response. Interceptors operate inside Spring MVC — they run after DispatcherServlet has resolved which controller method will handle the request. They're great for controller-specific concerns like request timing, audit logging, or checking custom annotations on the handler method. AOP is not tied to the web layer at all — it works on any Spring bean method using pointcut expressions. I use it for concerns that span multiple layers, like @Transactional on services, method-level logging, or retry logic. In my project, I use a Filter for request correlation IDs, an Interceptor for API request timing, and AOP for audit logging across services."

### 💻 Code Example

```java
// ✅ 1. SERVLET FILTER — raw HTTP level
@Component
@Order(1)   // filter ordering
public class RequestLoggingFilter extends OncePerRequestFilter {

    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain)
            throws ServletException, IOException {
        long start = System.currentTimeMillis();
        String correlationId = UUID.randomUUID().toString();
        MDC.put("correlationId", correlationId);

        try {
            chain.doFilter(request, response);     // continue chain
        } finally {
            long duration = System.currentTimeMillis() - start;
            log.info("[{}] {} {} → {} ({}ms)", correlationId,
                request.getMethod(), request.getRequestURI(),
                response.getStatus(), duration);
            MDC.clear();
        }
    }
}

// ✅ 2. HANDLER INTERCEPTOR — Spring MVC level
@Component
public class ApiTimingInterceptor implements HandlerInterceptor {

    @Override
    public boolean preHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler) {
        // handler = HandlerMethod → knows the controller + method
        if (handler instanceof HandlerMethod hm) {
            log.debug("Entering: {}.{}", hm.getBeanType().getSimpleName(),
                hm.getMethod().getName());
        }
        request.setAttribute("startTime", System.nanoTime());
        return true;   // true = continue, false = block request
    }

    @Override
    public void postHandle(HttpServletRequest request,
            HttpServletResponse response, Object handler,
            ModelAndView modelAndView) {
        // After controller, before view rendering (REST APIs: rarely used)
    }

    @Override
    public void afterCompletion(HttpServletRequest request,
            HttpServletResponse response, Object handler, Exception ex) {
        long start = (long) request.getAttribute("startTime");
        long duration = (System.nanoTime() - start) / 1_000_000;
        log.info("{} {} completed in {}ms", request.getMethod(),
            request.getRequestURI(), duration);
    }
}

@Configuration
public class WebConfig implements WebMvcConfigurer {
    @Override
    public void addInterceptors(InterceptorRegistry registry) {
        registry.addInterceptor(new ApiTimingInterceptor())
            .addPathPatterns("/api/**")            // only API paths
            .excludePathPatterns("/api/auth/**");   // skip auth endpoints
    }
}

// ✅ 3. AOP — any bean method level
@Aspect
@Component
public class AuditAspect {

    @Around("@annotation(com.myapp.annotation.Auditable)")
    public Object auditMethod(ProceedingJoinPoint pjp) throws Throwable {
        String method = pjp.getSignature().toShortString();
        log.info("AUDIT: Entering {}", method);
        long start = System.nanoTime();
        try {
            Object result = pjp.proceed();
            log.info("AUDIT: {} completed in {}ms", method,
                (System.nanoTime() - start) / 1_000_000);
            return result;
        } catch (Exception ex) {
            log.error("AUDIT: {} failed: {}", method, ex.getMessage());
            throw ex;
        }
    }
}

// Usage: annotate any service method
@Service
public class PaymentService {
    @Auditable   // ← AOP intercepts this
    public PaymentResult processPayment(PaymentRequest req) { ... }
}
```

### 🆚 Filter vs Interceptor vs AOP

| Aspect | Servlet Filter | HandlerInterceptor | AOP (@Aspect) |
|--------|---------------|-------------------|---------------|
| **Layer** | Servlet container | Spring MVC | Any Spring bean |
| **Runs** | Before DispatcherServlet | After handler resolved | Around method call |
| **Access** | Raw request/response | Handler method info | Method args, return |
| **Scope** | All requests (inc. static) | Mapped handler paths only | Any bean method |
| **Spring context** | Not guaranteed | ✅ Available | ✅ Available |
| **Use cases** | CORS, security, compress | Auth, timing, audit | Transactions, retry, log |
| **Config** | `@Component` / `FilterRegistrationBean` | `WebMvcConfigurer` | `@Aspect` + pointcut |

### ⚠️ Common Pitfalls
- **Filter can't access Spring beans easily** — use `OncePerRequestFilter` (Spring-managed) instead of raw `javax.servlet.Filter`
- **Interceptor misses non-MVC requests** — static resources, error dispatches may bypass interceptors
- **AOP on private methods** — Spring AOP uses proxies; private methods can't be proxied → use `protected` or `public`
- **AOP self-invocation** — calling an advised method from within the same class bypasses the proxy

### ⚡ Remember (Quick Recall)
- **Filter** = servlet level, raw HTTP, all requests
- **Interceptor** = Spring MVC, handler-aware, path-specific
- **AOP** = method level, any bean, pointcut-based
- Order: Filter → DispatcherServlet → Interceptor → Controller → AOP wraps methods

### 🔗 Related Topics
- [Spring Security filter chain](#q3)
- [AOP deep-dive](02-aop.md)
- [DispatcherServlet flow](05-mvc-beans-config.md#q1)

---

<a id="q5"></a>
## Q5. What is the difference between ApplicationContext and BeanFactory?

### 📝 One-Liner
`BeanFactory` is the **basic IoC container** (lazy, minimal); `ApplicationContext` is the **feature-rich superset** (eager, events, i18n, AOP, profiles) — Spring Boot always uses `ApplicationContext`.

### 🔑 Quick Answer
**BeanFactory** — core container interface: creates beans, manages dependencies, supports lazy initialization by default. That's it. **ApplicationContext** extends BeanFactory and adds: (1) **Eager bean initialization** — creates all singleton beans at startup (fail-fast). (2) **Event publishing** — `ApplicationEvent` / `@EventListener`. (3) **Internationalization** — `MessageSource` for i18n. (4) **Environment & profiles** — `@Profile`, property sources. (5) **AOP integration** — automatic proxy creation. (6) **Annotation support** — `@Autowired`, `@Component` scanning. In practice, **nobody uses raw BeanFactory** — `ApplicationContext` is what `SpringApplication.run()` returns.  *(BeanFactory = basic container (lazy, minimal); ApplicationContext = full-feature container (eager, events, profiles) — Spring Boot me hamesha ApplicationContext use hota hai)*

### 📖 How It Works (Detailed Explanation)

```
BeanFactory (interface):
  ├── getBean(name)
  ├── containsBean(name)
  └── isSingleton(name)
  → Lazy: creates beans only when first requested
  → No events, no i18n, no annotation processing

ApplicationContext (extends BeanFactory):
  ├── Everything BeanFactory does
  ├── Eager initialization (all singletons at startup)
  ├── ApplicationEventPublisher (event system)
  ├── MessageSource (i18n)
  ├── ResourceLoader (classpath/file resources)
  ├── Environment (profiles, properties)
  ├── Annotation processing (@Autowired, @Component)
  └── AOP auto-proxy creation

Implementations:
  AnnotationConfigApplicationContext  ← Java config (@Configuration)
  ClassPathXmlApplicationContext      ← XML config (legacy)
  WebApplicationContext               ← Web apps (servlet context)
  SpringApplication.run() → AnnotationConfigServletWebServerApplicationContext
```

### 🗣️ Interview Script
"BeanFactory is the root IoC container interface in Spring — it provides basic bean creation and dependency injection with lazy initialization. ApplicationContext extends BeanFactory and is what we actually use in every Spring Boot application. It adds eager bean initialization — all singletons are created at startup, so you catch configuration errors immediately rather than at runtime. It also provides an event system with ApplicationEvent and @EventListener, internationalization via MessageSource, environment abstraction with profiles and property sources, and automatic AOP proxy creation. When I call SpringApplication.run(), it returns an ApplicationContext — specifically an AnnotationConfigServletWebServerApplicationContext for web apps. In practice, I never interact with BeanFactory directly."

### 💻 Code Example

```java
// ✅ ApplicationContext — what Spring Boot uses
@SpringBootApplication
public class MyApp {
    public static void main(String[] args) {
        ApplicationContext ctx = SpringApplication.run(MyApp.class, args);
        // ctx is actually AnnotationConfigServletWebServerApplicationContext

        // Features available in ApplicationContext:
        UserService service = ctx.getBean(UserService.class);    // bean lookup
        ctx.getEnvironment().getActiveProfiles();                 // profiles
        ctx.publishEvent(new UserCreatedEvent(user));             // events
    }
}

// ✅ ApplicationContext event system — not available in BeanFactory
@Component
public class NotificationListener {

    @EventListener
    public void onUserCreated(UserCreatedEvent event) {
        log.info("New user created: {}", event.getUser().getEmail());
        emailService.sendWelcome(event.getUser());
    }
}

// ✅ Environment/profiles — ApplicationContext feature
@Service
@Profile("prod")
public class RealPaymentService implements PaymentService { ... }

@Service
@Profile("dev")
public class MockPaymentService implements PaymentService { ... }

// ❌ BeanFactory — raw, minimal (almost never used directly)
BeanFactory factory = new XmlBeanFactory(new ClassPathResource("beans.xml"));
UserService service = factory.getBean(UserService.class);
// No events, no profiles, no annotation processing, lazy only
```

### 🆚 BeanFactory vs ApplicationContext

| Feature | BeanFactory | ApplicationContext |
|---------|------------|-------------------|
| **Bean creation** | Lazy (on demand) | Eager (at startup) |
| **Event system** | ❌ | ✅ `@EventListener` |
| **i18n (MessageSource)** | ❌ | ✅ |
| **Environment / Profiles** | ❌ | ✅ `@Profile` |
| **AOP auto-proxy** | ❌ | ✅ |
| **Annotation processing** | ❌ | ✅ `@Autowired`, `@Component` |
| **Resource loading** | ❌ | ✅ `classpath:`, `file:` |
| **Used in practice** | Legacy/testing | Always (Spring Boot default) |
| **Startup time** | Faster (lazy) | Slightly slower (eager) but fail-fast |

### ⚡ Remember (Quick Recall)
- **BeanFactory** = basic container (lazy, beans on demand, minimal features)
- **ApplicationContext** = full container (eager, events, profiles, AOP, i18n)
- Spring Boot always uses **ApplicationContext**
- Eager init = fail-fast at startup, not at first request
- `SpringApplication.run()` returns `ApplicationContext`

### 🔗 Related Topics
- [DI Internals](01-spring-framework-internals.md#q7)
- [Bean Lifecycle](01-spring-framework-internals.md#q6)
- [Bean Scopes](05-mvc-beans-config.md#q3)

---

<a id="q6"></a>
## Q6. How do you configure multiple data sources in Spring Boot?

### 📝 One-Liner
Define **separate `DataSource`, `EntityManagerFactory`, and `TransactionManager` beans** for each database, use `@Primary` for the default, and partition entities/repositories into different packages per data source.

### 🔑 Quick Answer
Spring Boot auto-configures ONE data source. For multiple: **(1)** Define each `DataSource` bean manually with different properties. **(2)** Create separate `LocalContainerEntityManagerFactoryBean` per data source, scanning different entity packages. **(3)** Create separate `PlatformTransactionManager` per data source. **(4)** Mark one as `@Primary` (used when no qualifier specified). **(5)** Use `@Qualifier` or separate `@Configuration` classes to wire repositories to the correct data source. **Use case**: primary DB for business data + read replica for reporting, or main DB + legacy DB that can't be migrated. *(Multiple databases ke liye alag DataSource, EntityManager, aur TransactionManager beans banao — entity packages alag karo per DB)*

### 📖 How It Works (Detailed Explanation)

```
Project Structure for Multi-DataSource:

com.myapp/
├── primary/                   ← Primary database (PostgreSQL)
│   ├── entity/
│   │   └── User.java
│   ├── repository/
│   │   └── UserRepository.java
│   └── PrimaryDataSourceConfig.java
│
├── reporting/                 ← Reporting database (MySQL read replica)
│   ├── entity/
│   │   └── SalesReport.java
│   ├── repository/
│   │   └── SalesReportRepository.java
│   └── ReportingDataSourceConfig.java
```

### 🗣️ Interview Script
"To configure multiple data sources, I disable Spring Boot's auto-configuration for DataSource and define each one manually. For each database, I create three beans: a DataSource, a LocalContainerEntityManagerFactoryBean, and a PlatformTransactionManager. The key is entity isolation — each EntityManagerFactory scans different entity packages so there's no confusion about which entity belongs to which database. I mark the primary data source with @Primary so it's used by default when no qualifier is specified. Repositories are also separated by package — Spring Data JPA's @EnableJpaRepositories annotation has basePackages and entityManagerFactoryRef attributes that bind each repository to the correct data source. In my project, we used this for a primary PostgreSQL database and a legacy Oracle database for customer data that couldn't be migrated yet."

### 💻 Code Example

```yaml
# ✅ application.yml — two data source configurations
spring:
  datasource:
    primary:
      url: jdbc:postgresql://localhost:5432/main_db
      username: ${PRIMARY_DB_USER}
      password: ${PRIMARY_DB_PASS}
      driver-class-name: org.postgresql.Driver
    reporting:
      url: jdbc:mysql://read-replica:3306/reports_db
      username: ${REPORT_DB_USER}
      password: ${REPORT_DB_PASS}
      driver-class-name: com.mysql.cj.jdbc.Driver
```

```java
// ✅ Primary DataSource Configuration
@Configuration
@EnableJpaRepositories(
    basePackages = "com.myapp.primary.repository",
    entityManagerFactoryRef = "primaryEntityManager",
    transactionManagerRef = "primaryTransactionManager"
)
public class PrimaryDataSourceConfig {

    @Primary
    @Bean
    @ConfigurationProperties("spring.datasource.primary")
    public DataSourceProperties primaryDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Primary
    @Bean
    public DataSource primaryDataSource() {
        return primaryDataSourceProperties()
            .initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    }

    @Primary
    @Bean
    public LocalContainerEntityManagerFactoryBean primaryEntityManager(
            EntityManagerFactoryBuilder builder) {
        return builder
            .dataSource(primaryDataSource())
            .packages("com.myapp.primary.entity")   // scan only primary entities
            .persistenceUnit("primary")
            .build();
    }

    @Primary
    @Bean
    public PlatformTransactionManager primaryTransactionManager(
            @Qualifier("primaryEntityManager") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}

// ✅ Reporting DataSource Configuration
@Configuration
@EnableJpaRepositories(
    basePackages = "com.myapp.reporting.repository",
    entityManagerFactoryRef = "reportingEntityManager",
    transactionManagerRef = "reportingTransactionManager"
)
public class ReportingDataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.reporting")
    public DataSourceProperties reportingDataSourceProperties() {
        return new DataSourceProperties();
    }

    @Bean
    public DataSource reportingDataSource() {
        return reportingDataSourceProperties()
            .initializeDataSourceBuilder()
            .type(HikariDataSource.class)
            .build();
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean reportingEntityManager(
            EntityManagerFactoryBuilder builder) {
        return builder
            .dataSource(reportingDataSource())
            .packages("com.myapp.reporting.entity")
            .persistenceUnit("reporting")
            .build();
    }

    @Bean
    public PlatformTransactionManager reportingTransactionManager(
            @Qualifier("reportingEntityManager") EntityManagerFactory emf) {
        return new JpaTransactionManager(emf);
    }
}

// ✅ Usage — repositories auto-wired to correct data source
@Service
public class DashboardService {

    private final UserRepository userRepo;                    // → primary DB
    private final SalesReportRepository reportRepo;           // → reporting DB

    @Transactional("primaryTransactionManager")               // explicit TX manager
    public void updateUser(Long id, UpdateRequest req) {
        userRepo.save(...);
    }

    @Transactional(value = "reportingTransactionManager", readOnly = true)
    public List<SalesReport> getReports(int year) {
        return reportRepo.findByYear(year);
    }
}
```

### ⚠️ Common Pitfalls
- **Forgetting `@Primary`** — without it, Spring can't decide which DataSource to use for auto-wired components
- **Entity package overlap** — if both EntityManagerFactories scan the same package, entities get duplicated → keep packages strictly separate
- **Cross-database transactions** — `@Transactional` spans ONE data source by default; cross-DB transactions need JTA (`Atomikos`) or Saga
- **Connection pool per data source** — each data source has its own HikariCP pool; total connections = pool1 + pool2

### ⚡ Remember (Quick Recall)
- **3 beans per data source**: DataSource + EntityManagerFactory + TransactionManager
- `@Primary` on the main one, `@Qualifier` for the secondary
- **Separate entity packages** per data source
- `@EnableJpaRepositories(basePackages, entityManagerFactoryRef)` binds repos
- Cross-DB transactions need JTA or Saga

### 🔗 Related Topics
- [Connection pool sizing](../../production-debugging/01-jvm-memory-performance.md)
- [Distributed transactions / Saga](../architecture/03-system-design-distributed.md)
- [Spring Data JPA vs JDBC](07-project-infrastructure-decisions.md#q1)
