# 🍃 Spring Boot — Auto-Configuration, Actuator, Properties & Starters (Q1–Q4)

> **Source**: EPAM Systems Java Backend Interview  
> **Coverage**: Spring Boot internals — how auto-config works, monitoring with Actuator, property binding, starter mechanism

---

<a id="q1"></a>
## Q1. How does Spring Boot Auto-Configuration work internally?

### 📝 One-Liner
Auto-configuration uses `@Conditional` annotations + `META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` to **register beans automatically** based on classpath, existing beans, and property conditions.

### 🔑 Quick Answer
When you add `@SpringBootApplication` (which includes `@EnableAutoConfiguration`), Spring Boot: **(1)** Scans the `AutoConfiguration.imports` file (Spring Boot 3) or `spring.factories` (Boot 2) — listing hundreds of auto-configuration classes. **(2)** Each class has `@Conditional` annotations — `@ConditionalOnClass` (is H2 on classpath?), `@ConditionalOnMissingBean` (did user already define a DataSource?), `@ConditionalOnProperty` (is feature enabled?). **(3)** Only matching configurations create beans. **(4)** User beans always win — `@ConditionalOnMissingBean` ensures your custom bean overrides the auto-configured one. *(Auto-config = classpath dekho, conditions check karo, agar user ne khud nahi banaya toh Spring Boot bana dega)*

### 📖 How It Works (Detailed Explanation)

```
Auto-Configuration Flow:
┌───────────────────────────┐
│ @SpringBootApplication     │
│  └─ @EnableAutoConfiguration│
└─────────────┬─────────────┘
              │
              ▼
┌───────────────────────────────────────┐
│ Load: META-INF/spring/               │
│  org.springframework.boot.            │
│  autoconfigure.AutoConfiguration.imports│
│                                       │
│ → DataSourceAutoConfiguration         │
│ → JpaAutoConfiguration                │
│ → WebMvcAutoConfiguration             │
│ → ... (150+ configurations)           │
└─────────────┬─────────────────────────┘
              │
              ▼
┌───────────────────────────────────────┐
│ Evaluate @Conditional annotations:    │
│                                       │
│ @ConditionalOnClass(DataSource.class) │ ← Is JDBC driver on classpath?
│ @ConditionalOnMissingBean(DataSource) │ ← Did user define one?
│ @ConditionalOnProperty("spring.       │
│   datasource.url")                    │ ← Is URL configured?
│                                       │
│ ALL conditions TRUE → create beans    │
│ ANY condition FALSE → skip entirely   │
└───────────────────────────────────────┘
```

**Key conditionals**: `@ConditionalOnClass` — library present on classpath. `@ConditionalOnMissingBean` — user hasn't already defined this bean (user wins). `@ConditionalOnProperty` — property is set/matches value. `@ConditionalOnWebApplication` — running as web app. **Ordering**: `@AutoConfigureBefore/After` controls the order of auto-configurations. **Debug**: set `debug=true` in `application.properties` → prints **CONDITIONS EVALUATION REPORT** showing which auto-configs matched/didn't match and why.

### 🗣️ Answering Approach
"When Spring Boot starts, @EnableAutoConfiguration triggers loading of auto-configuration classes from META-INF registration files. Each class is guarded by @Conditional annotations — for example, DataSourceAutoConfiguration only activates if a JDBC driver is on the classpath and the user hasn't already defined a DataSource bean. The @ConditionalOnMissingBean pattern is the key — it means user-defined beans always take priority over auto-configured ones. This is why you can override any default by simply defining your own @Bean. When I need to debug what's being auto-configured, I set debug=true in application.properties, which prints a conditions evaluation report showing exactly which configurations activated and which were skipped and why."

### 💻 Code Example

```java
// ✅ How a typical auto-configuration class looks (simplified)
@AutoConfiguration
@ConditionalOnClass(DataSource.class)                // only if JDBC driver is on classpath
@ConditionalOnProperty(
    prefix = "spring.datasource", name = "url")      // only if URL is configured
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean                         // ⭐ user-defined DataSource wins!
    public DataSource dataSource(DataSourceProperties props) {
        return DataSourceBuilder.create()
            .url(props.getUrl())
            .username(props.getUsername())
            .password(props.getPassword())
            .build();
    }
}

// ✅ User overrides auto-configuration — just define your own bean
@Configuration
public class MyDataSourceConfig {
    @Bean  // this takes priority — @ConditionalOnMissingBean won't trigger
    public DataSource dataSource() {
        HikariDataSource ds = new HikariDataSource();
        ds.setJdbcUrl("jdbc:postgresql://custom-host/mydb");
        ds.setMaximumPoolSize(20);
        return ds;
    }
}

// ✅ Disable specific auto-configuration
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    SecurityAutoConfiguration.class
})
public class MyApp { }

// Or in application.properties:
// spring.autoconfigure.exclude=org.springframework.boot.autoconfigure.jdbc.DataSourceAutoConfiguration

// ✅ Writing your own auto-configuration (for shared libraries)
@AutoConfiguration
@ConditionalOnClass(MyService.class)
public class MyServiceAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "mylib", name = "enabled", havingValue = "true", matchIfMissing = true)
    public MyService myService() {
        return new DefaultMyService();
    }
}

// Register in: META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// com.mylib.MyServiceAutoConfiguration
```

```properties
# ✅ Debug auto-configuration — CONDITIONS EVALUATION REPORT
debug=true

# Output shows:
# Positive matches: DataSourceAutoConfiguration matched (OnClassCondition, OnPropertyCondition)
# Negative matches: MongoAutoConfiguration did not match (OnClassCondition: missing MongoClient)
```

### ⚠️ Common Pitfalls
- **Not understanding override** — defining a `@Bean` of same type disables auto-config for that bean
- **Wrong exclude** — excluding parent config class, not the actual one (use debug=true to find exact class name)
- **ConditionalOnProperty matchIfMissing** — `matchIfMissing=true` means the config activates even if the property isn't set
- **Order matters** — custom configurations might need `@AutoConfigureBefore/After` to control loading sequence
- **spring.factories vs imports** — Spring Boot 3 moved from `spring.factories` to `AutoConfiguration.imports` file

### 🆚 Comparison Table

| Conditional Annotation | Checks | Example |
|----------------------|--------|---------|
| `@ConditionalOnClass` | Class on classpath | `@ConditionalOnClass(DataSource.class)` |
| `@ConditionalOnMissingBean` | Bean NOT already defined | `@ConditionalOnMissingBean(DataSource.class)` |
| `@ConditionalOnProperty` | Property value | `@ConditionalOnProperty("feature.enabled")` |
| `@ConditionalOnWebApplication` | Web app type | Servlet vs Reactive |
| `@ConditionalOnBean` | Bean already exists | Depend on another config |
| `@ConditionalOnMissingClass` | Class NOT on classpath | Skip if library absent |

### ⚡ Remember (Quick Recall)
- `@EnableAutoConfiguration` → loads from `AutoConfiguration.imports`
- **@ConditionalOnMissingBean** = user beans always win
- `debug=true` → CONDITIONS EVALUATION REPORT
- Exclude: `@SpringBootApplication(exclude = {...})`
- Spring Boot 3: `AutoConfiguration.imports`; Boot 2: `spring.factories`

### 🔗 Follow-up Topics
- [Q4 → Spring Boot Starters (how they trigger auto-config)](#q4)
- [Q7 in spring/01 → Dependency Injection internals](../spring/01-spring-framework-internals.md#q7)
- Creating custom Spring Boot starters for shared libraries

---

<a id="q2"></a>
## Q2. What is Spring Boot Actuator and what endpoints does it provide?

### 📝 One-Liner
Actuator exposes **production-ready endpoints** for health checks, metrics, environment info, and application management — essential for monitoring and ops.

### 🔑 Quick Answer
Add `spring-boot-starter-actuator` → get endpoints at `/actuator/*`. **Key endpoints**: `/health` (app + dependency health — DB, disk, Redis), `/metrics` (JVM, HTTP, custom via Micrometer), `/info` (app metadata), `/env` (configuration properties), `/beans` (all Spring beans), `/loggers` (view/change log levels at runtime), `/threaddump`, `/heapdump`. **By default only `/health` is exposed** over HTTP — configure `management.endpoints.web.exposure.include` to expose more. **Security**: always secure Actuator endpoints in production (Spring Security). *(Actuator = production monitoring ke liye health, metrics, logs sab ek jagah milta hai)*

### 📖 How It Works (Detailed Explanation)

```
Actuator Architecture:
┌─────────────────────────────────────────┐
│ Spring Boot Application                  │
│                                          │
│  /actuator/health    → HealthIndicators  │
│    ├── db            → DataSourceHealth  │
│    ├── diskSpace     → DiskSpaceHealth   │
│    ├── redis         → RedisHealth       │
│    └── custom        → MyCustomHealth    │
│                                          │
│  /actuator/metrics   → Micrometer        │
│    ├── jvm.memory.used                   │
│    ├── http.server.requests              │
│    ├── process.cpu.usage                 │
│    └── custom.orders.processed           │
│                                          │
│  /actuator/loggers   → Runtime log level │
│  /actuator/env       → Config properties │
│  /actuator/beans     → All Spring beans  │
│  /actuator/prometheus→ Metrics for Prom  │
└─────────────────────────────────────────┘
```

**Health indicators** auto-register — add a DataSource bean → `DataSourceHealthIndicator` activates. Add Redis → `RedisHealthIndicator` activates. Write custom health checks for business logic (e.g., check if payment gateway is reachable). **Metrics**: backed by Micrometer — automatic metrics for HTTP requests, JVM memory, thread pools, plus custom metrics via `MeterRegistry`. **Kubernetes**: `/health/liveness` and `/health/readiness` probes use Actuator health groups.

### 🗣️ Answering Approach
"Actuator provides production-ready operational endpoints out of the box. The health endpoint is what I use for Kubernetes liveness and readiness probes — it auto-detects and checks database connectivity, disk space, and any dependencies. Metrics are backed by Micrometer, so I get JVM metrics, HTTP request statistics, and custom business metrics that I expose to Prometheus for Grafana dashboards. The loggers endpoint is incredibly useful in production — I can change log levels at runtime via a POST request without restarting the service. I always secure Actuator endpoints with Spring Security — in production, only health is publicly accessible, and the rest require authentication or are restricted to the internal management port."

### 💻 Code Example

```xml
<!-- pom.xml -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>
<!-- For Prometheus metrics export -->
<dependency>
    <groupId>io.micrometer</groupId>
    <artifactId>micrometer-registry-prometheus</artifactId>
</dependency>
```

```properties
# application.properties — Actuator configuration

# Expose specific endpoints (default: only health)
management.endpoints.web.exposure.include=health,metrics,info,loggers,prometheus
# Or expose all: management.endpoints.web.exposure.include=*

# Health endpoint — show details
management.endpoint.health.show-details=when-authorized

# Separate management port (security best practice)
management.server.port=8081

# Kubernetes probes (Spring Boot 2.3+)
management.endpoint.health.probes.enabled=true
management.health.livenessstate.enabled=true
management.health.readinessstate.enabled=true

# Info endpoint
management.info.env.enabled=true
info.app.name=order-service
info.app.version=2.1.0
```

```java
// ✅ Custom Health Indicator
@Component
public class PaymentGatewayHealthIndicator implements HealthIndicator {

    private final PaymentClient paymentClient;

    public PaymentGatewayHealthIndicator(PaymentClient paymentClient) {
        this.paymentClient = paymentClient;
    }

    @Override
    public Health health() {
        try {
            boolean reachable = paymentClient.ping();
            if (reachable) {
                return Health.up()
                    .withDetail("provider", "Stripe")
                    .withDetail("latency", "45ms")
                    .build();
            }
            return Health.down()
                .withDetail("error", "Gateway unreachable")
                .build();
        } catch (Exception e) {
            return Health.down(e).build();
        }
    }
}

// ✅ Custom Metrics
@Service
public class OrderService {

    private final Counter ordersCounter;
    private final Timer orderProcessingTimer;

    public OrderService(MeterRegistry registry) {
        this.ordersCounter = Counter.builder("orders.created")
            .tag("type", "total")
            .register(registry);
        this.orderProcessingTimer = Timer.builder("orders.processing.time")
            .register(registry);
    }

    public Order createOrder(OrderRequest request) {
        return orderProcessingTimer.record(() -> {
            Order order = processOrder(request);
            ordersCounter.increment();
            return order;
        });
    }
}

// ✅ Secure Actuator endpoints
@Configuration
public class ActuatorSecurityConfig {

    @Bean
    public SecurityFilterChain actuatorSecurity(HttpSecurity http) throws Exception {
        return http
            .securityMatcher("/actuator/**")
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/actuator/health/**").permitAll()    // public health
                .requestMatchers("/actuator/**").hasRole("OPS"))       // rest: ops only
            .httpBasic(Customizer.withDefaults())
            .build();
    }
}
```

```bash
# Runtime log level change (no restart needed!) ⭐
curl -X POST http://localhost:8081/actuator/loggers/com.myapp.service \
  -H "Content-Type: application/json" \
  -d '{"configuredLevel": "DEBUG"}'
```

### ⚠️ Common Pitfalls
- **Exposing all endpoints without security** — `/env` and `/heapdump` leak sensitive data → always secure in production
- **show-details=always** — health details (DB URL, Redis host) visible to everyone → use `when-authorized`
- **Not setting management port** — Actuator on same port as API → internal endpoints accessible from internet
- **Forgetting to exclude sensitive endpoints** — `/shutdown` can shut down your app if exposed!
- **Health endpoint too slow** — custom health checks with timeouts → set `management.endpoint.health.cache.time-to-live=10s`

### 🆚 Comparison Table

| Endpoint | Purpose | HTTP | Production Use |
|----------|---------|------|---------------|
| `/health` | App + dependency health | GET | K8s probes, load balancer |
| `/metrics` | JVM, HTTP, custom metrics | GET | Monitoring dashboards |
| `/prometheus` | Prometheus-format metrics | GET | Prometheus scraping |
| `/loggers` | View/change log levels | GET/POST | Runtime debugging |
| `/env` | Configuration properties | GET | Config troubleshooting |
| `/beans` | All Spring beans | GET | Dependency inspection |
| `/threaddump` | Thread dump | GET | Deadlock diagnosis |
| `/heapdump` | Heap dump (binary) | GET | Memory analysis |
| `/info` | App metadata | GET | Version info |

### ⚡ Remember (Quick Recall)
- `spring-boot-starter-actuator` → `/actuator/*`
- Default: only `/health` exposed → configure `exposure.include`
- **Always secure** in production (separate port + Spring Security)
- Custom health: implement `HealthIndicator`
- Custom metrics: inject `MeterRegistry` (Micrometer)
- K8s probes: `/health/liveness` + `/health/readiness`
- Runtime log change: POST to `/loggers/{package}`

### 🔗 Follow-up Topics
- [Q15 in cloud-devops/01 → Distributed Tracing & Observability](../../cloud-devops/01-observability-alerting.md)
- Micrometer + Prometheus + Grafana setup
- Spring Boot Admin (UI dashboard for Actuator)

---

<a id="q3"></a>
## Q3. What is the difference between @Value and @ConfigurationProperties?

### 📝 One-Liner
`@Value` injects **individual** properties with SpEL support; `@ConfigurationProperties` binds an **entire prefix group** to a type-safe POJO — preferred for structured config.

### 🔑 Quick Answer
`@Value("${db.url}")` — injects one property at a time, supports SpEL expressions, works on fields. `@ConfigurationProperties(prefix = "db")` — binds all properties under a prefix to a Java class (e.g., `db.url`, `db.username`, `db.pool-size` → fields in a `DbProperties` class). **@ConfigurationProperties is preferred** because: type-safe, validated with `@Validated` + Bean Validation, IDE auto-complete, relaxed binding (kebab-case → camelCase), immutable with `@ConstructorBinding`. Use `@Value` only for simple one-off values or when SpEL is needed. *(Value = ek property inject karo; ConfigurationProperties = saari related properties ek class mein bind karo)*

### 📖 How It Works (Detailed Explanation)

```
@Value — one property at a time:
┌─────────────────────────────────────┐
│ @Value("${app.timeout:30}")         │
│ private int timeout;                │  ← single value, with default
│                                     │
│ @Value("#{${app.timeout} * 1000}")  │
│ private long timeoutMs;             │  ← SpEL expression
└─────────────────────────────────────┘

@ConfigurationProperties — structured binding:
┌─────────────────────────────────────┐
│ application.yml:                    │
│   datasource:                       │
│     url: jdbc:postgresql://...      │
│     username: admin                 │
│     pool-size: 10                   │  ← kebab-case
│     connection-timeout: 30s         │  ← Duration support
│                                     │
│ @ConfigurationProperties("datasource")│
│ class DatasourceProperties {        │
│   String url;                       │  ← auto-mapped
│   String username;                  │  ← auto-mapped
│   int poolSize;                     │  ← relaxed binding (kebab → camel)
│   Duration connectionTimeout;       │  ← type conversion
│ }                                   │
└─────────────────────────────────────┘
```

**Relaxed binding**: `pool-size` (kebab) → `poolSize` (camel) → `POOL_SIZE` (env var). All map to the same field. **Type conversion**: Strings auto-convert to `Duration`, `DataSize`, `List`, `Map`, enums. **Validation**: Add `@Validated` on the class + `@NotBlank`, `@Min`, `@Max` on fields → fails fast on startup if config is invalid. **Immutable binding** (Spring Boot 2.2+): use `@ConstructorBinding` with a record or final fields.

### 🗣️ Answering Approach
"For individual one-off properties or when I need SpEL expressions, I use @Value. But for any structured configuration — database settings, API client configs, feature flags — I always use @ConfigurationProperties. It binds an entire prefix to a type-safe Java object, so I get compile-time type checking, IDE auto-completion, and relaxed binding where kebab-case properties map to camelCase fields automatically. I also add @Validated with Bean Validation annotations so the application fails fast on startup if required configuration is missing or invalid — much better than discovering a null value at runtime. In Spring Boot 2.2+, I use records or constructor binding for immutable configuration objects."

### 💻 Code Example

```java
// ✅ @Value — simple injection
@Component
public class NotificationService {
    @Value("${notification.email.from:noreply@app.com}")  // with default
    private String fromEmail;

    @Value("${notification.retry-count:3}")
    private int retryCount;

    @Value("#{${notification.retry-count:3} * 2}")  // SpEL expression
    private int maxAttempts;
}

// ✅ @ConfigurationProperties — structured binding (PREFERRED)
@Validated
@ConfigurationProperties(prefix = "app.datasource")
public class DatasourceProperties {

    @NotBlank
    private String url;

    @NotBlank
    private String username;

    private String password;

    @Min(1) @Max(100)
    private int poolSize = 10;  // default value

    private Duration connectionTimeout = Duration.ofSeconds(30);

    private Map<String, String> properties = new HashMap<>();

    // getters and setters (or use Lombok @Data)
    public String getUrl() { return url; }
    public void setUrl(String url) { this.url = url; }
    // ... etc
}

// ✅ Immutable @ConfigurationProperties with record (Spring Boot 3+)
@ConfigurationProperties(prefix = "app.cache")
public record CacheProperties(
    @DefaultValue("true") boolean enabled,
    @DefaultValue("3600") int ttlSeconds,
    @DefaultValue("1000") int maxSize,
    List<String> regions
) { }

// ✅ Enable in main class or config
@SpringBootApplication
@EnableConfigurationProperties({DatasourceProperties.class, CacheProperties.class})
public class MyApp { }

// Or use @ConfigurationPropertiesScan to auto-detect

// ✅ Usage — inject like any bean
@Service
public class DataService {
    private final DatasourceProperties dbProps;   // ⭐ type-safe config

    public DataService(DatasourceProperties dbProps) {
        this.dbProps = dbProps;
    }

    public void connect() {
        // dbProps.getUrl(), dbProps.getPoolSize() — all type-safe!
    }
}
```

```yaml
# application.yml
app:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    password: ${DB_PASSWORD}         # env variable
    pool-size: 20                    # → poolSize (relaxed binding)
    connection-timeout: 30s          # → Duration.ofSeconds(30)
    properties:
      reWriteBatchedInserts: true
      ApplicationName: my-service

  cache:
    enabled: true
    ttl-seconds: 7200
    max-size: 5000
    regions:
      - users
      - products
```

### ⚠️ Common Pitfalls
- **@Value on `static` fields** — doesn't work; Spring injects into instance fields only
- **Missing `@EnableConfigurationProperties`** — @ConfigurationProperties class won't be registered as a bean
- **Validation not triggering** — forgot `@Validated` on the class → Bean Validation annotations ignored
- **Sensitive values in @Value** — `@Value("${db.password}")` shows in logs/debug → use `@ConfigurationProperties` + Spring Cloud Vault
- **Kebab to camel confusion** — `my-property` → `myProperty` works; `MY_PROPERTY` env var also works (relaxed binding)

### 🆚 Comparison Table

| Aspect | @Value | @ConfigurationProperties |
|--------|--------|--------------------------|
| Scope | Single value | Entire prefix group |
| Type-safe | No (String injection) | Yes (typed POJO) |
| SpEL support | ✅ Yes | ❌ No |
| Relaxed binding | ❌ No | ✅ Yes (kebab/camel/env) |
| Validation | Manual | `@Validated` + Bean Validation |
| IDE support | Limited | Full auto-complete |
| Immutable | N/A | `@ConstructorBinding` / records |
| Nested objects | ❌ No | ✅ Yes |
| Meta-data | ❌ No | ✅ `spring-configuration-metadata.json` |
| Use when | Quick, one-off, SpEL | **Structured config (default choice)** |

### ⚡ Remember (Quick Recall)
- **@Value** = simple, one-off, SpEL supported
- **@ConfigurationProperties** = structured, type-safe, validated — **always preferred**
- Relaxed binding: `pool-size` = `poolSize` = `POOL_SIZE`
- `@Validated` + `@NotBlank`/`@Min` = fail-fast on startup
- Records + `@ConfigurationProperties` = immutable config (Boot 3+)

### 🔗 Follow-up Topics
- [Q1 → Auto-Configuration (reads these properties via @ConditionalOnProperty)](#q1)
- Spring Cloud Config (externalized configuration)
- `spring-boot-configuration-processor` for IDE metadata generation

---

<a id="q4"></a>
## Q4. What is a Spring Boot Starter and how does it work?

### 📝 One-Liner
A Starter is a **dependency descriptor** that bundles all libraries + auto-configuration needed for a feature — add one dependency, get everything configured automatically.

### 🔑 Quick Answer
A Spring Boot Starter is a Maven/Gradle dependency that: **(1)** Pulls in all required transitive dependencies (libraries), **(2)** Triggers relevant auto-configurations. Example: `spring-boot-starter-web` brings in Spring MVC, embedded Tomcat, Jackson, validation — and auto-configures DispatcherServlet, message converters, error handling. **No code, no XML** — just add the dependency. Naming convention: official = `spring-boot-starter-{feature}`, custom = `{name}-spring-boot-starter`. You can create custom starters for shared company libraries. *(Starter = ek dependency add karo, sab libraries + config automatic aa jaati hai)*

### 📖 How It Works (Detailed Explanation)

```
What happens when you add spring-boot-starter-web:

pom.xml: spring-boot-starter-web
         │
         ├── spring-web         (Spring MVC)
         ├── spring-webmvc      (DispatcherServlet)
         ├── spring-boot-starter-tomcat (embedded server)
         │   └── tomcat-embed-core
         ├── spring-boot-starter-json
         │   └── jackson-databind (JSON serialization)
         └── spring-boot-starter (core)
             └── spring-boot-autoconfigure
                 │
                 └── Triggers:
                     ├── WebMvcAutoConfiguration    → DispatcherServlet, converters
                     ├── HttpEncodingAutoConfiguration → UTF-8
                     ├── JacksonAutoConfiguration    → ObjectMapper bean
                     └── EmbeddedTomcatAutoConfiguration → Tomcat on port 8080
```

**Starter = Dependencies + Auto-Configuration**. The starter itself is usually just a `pom.xml` with no source code — it's purely a dependency aggregator. The auto-configuration logic lives in the `spring-boot-autoconfigure` module and activates via `@ConditionalOnClass` when the starter's libraries are on the classpath. **Custom starters**: create for shared company infrastructure — e.g., a starter that configures logging format, metrics, security, and health checks uniformly across all microservices.

### 🗣️ Answering Approach
"A Spring Boot Starter is really two things packaged together: a curated set of dependencies and the auto-configuration to wire them up. When I add spring-boot-starter-data-jpa, it pulls in Hibernate, Spring Data JPA, HikariCP connection pool, and the JDBC driver — then auto-configuration creates a DataSource, EntityManagerFactory, and transaction manager based on my application.properties. I just provide the database URL and start coding repositories. For our company, I've created custom starters that standardize logging, metrics, and security configuration across all microservices — each team just adds our starter dependency and gets consistent observability and security out of the box."

### 💻 Code Example

```xml
<!-- ✅ Common starters and what they bring -->

<!-- Web (MVC + Tomcat + Jackson) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- JPA (Hibernate + Spring Data + HikariCP) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- Security (Spring Security + filter chain) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>

<!-- Actuator (health + metrics + endpoints) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
</dependency>

<!-- Test (JUnit 5 + Mockito + AssertJ + Spring Test) -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

```java
// ✅ Creating a Custom Starter (3 parts)

// PART 1: Auto-Configuration module (mylib-spring-boot-autoconfigure)
@AutoConfiguration
@ConditionalOnClass(MyCompanyLogger.class)
@EnableConfigurationProperties(MyCompanyProperties.class)
public class MyCompanyAutoConfiguration {

    @Bean
    @ConditionalOnMissingBean
    public MyCompanyLogger companyLogger(MyCompanyProperties props) {
        return new MyCompanyLogger(props.getAppName(), props.getLogFormat());
    }

    @Bean
    @ConditionalOnMissingBean
    @ConditionalOnProperty(prefix = "mycompany.metrics", name = "enabled", havingValue = "true", matchIfMissing = true)
    public MetricsExporter metricsExporter(MeterRegistry registry) {
        return new PrometheusMetricsExporter(registry);
    }
}

@ConfigurationProperties(prefix = "mycompany")
public class MyCompanyProperties {
    private String appName = "unknown-service";
    private String logFormat = "json";
    // getters + setters
}

// PART 2: Starter module (mycompany-spring-boot-starter)
// Just a pom.xml — NO source code!
```

```xml
<!-- mycompany-spring-boot-starter/pom.xml -->
<project>
    <artifactId>mycompany-spring-boot-starter</artifactId>
    <dependencies>
        <dependency>
            <groupId>com.mycompany</groupId>
            <artifactId>mylib-spring-boot-autoconfigure</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mycompany</groupId>
            <artifactId>mycompany-logging</artifactId>
        </dependency>
        <dependency>
            <groupId>com.mycompany</groupId>
            <artifactId>mycompany-metrics</artifactId>
        </dependency>
    </dependencies>
</project>
```

```java
// PART 3: Usage in any microservice — just add the starter!
// pom.xml: <dependency>mycompany-spring-boot-starter</dependency>
// application.yml:
//   mycompany:
//     app-name: order-service
//     log-format: json
//     metrics:
//       enabled: true
// → Logger, metrics, and monitoring auto-configured! ✅
```

### ⚠️ Common Pitfalls
- **Starter bloat** — `spring-boot-starter-web` includes Tomcat; if you want Netty, exclude Tomcat and add `starter-webflux`
- **Conflicting starters** — `starter-web` (Servlet) + `starter-webflux` (Reactive) together → Spring defaults to Servlet
- **Version conflicts** — always use Spring Boot's dependency management (`spring-boot-dependencies` BOM) to avoid version mismatches
- **Custom starter naming** — official starters: `spring-boot-starter-*`; your starters: `{name}-spring-boot-starter` (avoid the `spring-boot-starter-` prefix)

### 🆚 Comparison Table

| Starter | Brings In | Auto-Configures |
|---------|-----------|----------------|
| `starter-web` | Spring MVC, Tomcat, Jackson | DispatcherServlet, converters, error handling |
| `starter-data-jpa` | Hibernate, HikariCP, Spring Data | DataSource, EntityManager, repositories |
| `starter-security` | Spring Security | Filter chain, login page, CSRF |
| `starter-actuator` | Actuator, Micrometer | Health, metrics, endpoints |
| `starter-test` | JUnit 5, Mockito, AssertJ | Test context, mocks |
| `starter-validation` | Hibernate Validator | @Valid, MethodValidation |

### ⚡ Remember (Quick Recall)
- Starter = **dependency aggregator** + triggers **auto-configuration**
- Official: `spring-boot-starter-{feature}`
- Custom: `{name}-spring-boot-starter`
- Starter pom has **no code** — just dependency list
- Auto-config activates via `@ConditionalOnClass` (classpath detection)
- Use Spring Boot BOM for version management

### 🔗 Follow-up Topics
- [Q1 → Auto-Configuration (triggered by starters)](#q1)
- [Q2 → Actuator (starter-actuator)](#q2)
- Maven BOM (Bill of Materials) and dependency management
