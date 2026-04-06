# 🌱 Spring Boot Internals — Advanced Deep Dive

> Advanced questions on Spring Boot startup flow, configuration loading, fat JAR, CommandLineRunner, and internal mechanisms.

---

## Q1. How Does Spring Boot Load application.properties Internally?

### 📝 One-Liner
Spring Boot uses `PropertySourceLoader` implementations to load properties from predefined locations in a specific order.

### 🔑 Quick Answer
`ConfigFileApplicationListener` triggers `PropertySourceLoader` (Properties + YAML loaders) → searches `classpath:/`, `classpath:/config/`, `file:./`, `file:./config/` → merges into `Environment` in priority order. *(pehle se defined locations se load hota hai, priority order mein merge hota hai)*

### 📖 How It Works
1. **Trigger**: `ConfigFileApplicationListener` fires during `ApplicationEnvironmentPreparedEvent`
2. **Loaders**: `PropertiesPropertySourceLoader` (.properties) and `YamlPropertySourceLoader` (.yml/.yaml)
3. **Search Locations** (in priority order, last wins):
   - `classpath:/`
   - `classpath:/config/`
   - `file:./`
   - `file:./config/`
4. **Profile Resolution**: Base properties first → profile-specific (`application-{profile}.properties`) overlay
5. **Merge**: All sources merged into Spring `Environment` abstraction

### 🗣️ Answering Approach
"Spring Boot uses ConfigFileApplicationListener to load properties during the environment-prepared phase. It searches four default locations — classpath root, classpath config, file root, and file config — using PropertySourceLoader implementations for both .properties and .yml formats. Properties are merged with a well-defined precedence, where file-system config wins over classpath."

### 💻 Code
```java
// Custom location via command line
java -jar app.jar --spring.config.location=file:/opt/config/

// Custom additional location (doesn't replace defaults)
java -jar app.jar --spring.config.additional-location=file:/opt/extra/

// Programmatic access
@Value("${server.port}")
private int port;

// Or via Environment
@Autowired
private Environment env;
String port = env.getProperty("server.port");
```

### ⚡ Remember
- Priority (highest to lowest): command-line → JNDI → system properties → OS env → profile-specific → application.properties → @PropertySource → defaults
- `spring.config.location` REPLACES defaults; `additional-location` ADDS to them
- In Spring Boot 2.4+: `ConfigDataEnvironmentPostProcessor` replaces the old listener

---

## Q2. What is the Exact Startup Flow of a Spring Boot Application?

### 📝 One-Liner
`SpringApplication.run()` bootstraps the app through environment preparation, context creation, bean registration, auto-configuration, and server startup.

### 🔑 Quick Answer
`main()` → `SpringApplication.run()` → create `SpringApplication` instance → determine web type → load `SpringFactoriesLoader` → prepare Environment → create ApplicationContext → register bean definitions → refresh context (auto-config, component scan, bean creation) → start embedded server → fire `ApplicationReadyEvent`. *(step by step context banta hai, beans create hote hain, server start hota hai)*

### 📖 How It Works
```
1. main() calls SpringApplication.run(App.class, args)
2. new SpringApplication(App.class)
   ├── Detect web application type (SERVLET / REACTIVE / NONE)
   ├── Load ApplicationContextInitializers (from spring.factories)
   └── Load ApplicationListeners (from spring.factories)
3. run(args) begins:
   ├── Create & start StopWatch
   ├── Create BootstrapContext
   ├── Fire ApplicationStartingEvent
   ├── Prepare Environment
   │   ├── Create ConfigurableEnvironment
   │   ├── Configure property sources & profiles
   │   └── Fire ApplicationEnvironmentPreparedEvent
   ├── Print Banner (Spring Boot ASCII art)
   ├── Create ApplicationContext
   │   └── AnnotationConfigServletWebServerApplicationContext (for web)
   ├── Prepare Context
   │   ├── Set environment
   │   ├── Post-process context
   │   ├── Apply initializers
   │   ├── Fire ApplicationContextInitializedEvent
   │   └── Register main class as bean definition
   ├── Refresh Context (most expensive step)
   │   ├── Invoke BeanFactoryPostProcessors
   │   ├── Process @Configuration, @ComponentScan, @Import
   │   ├── Apply auto-configuration classes
   │   ├── Create all singleton beans
   │   └── Start embedded web server
   ├── Fire ApplicationStartedEvent
   ├── Call Runners (CommandLineRunner, ApplicationRunner)
   └── Fire ApplicationReadyEvent
```

### 🗣️ Answering Approach
"SpringApplication.run() first determines the application type, then loads initializers and listeners from spring.factories. It prepares the Environment by loading properties and profiles, creates the ApplicationContext, and refreshes it — which is where the heavy lifting happens: component scanning, auto-configuration processing, bean creation, and embedded server startup. Finally, it calls any CommandLineRunner/ApplicationRunner beans."

### ⚡ Remember
- `refresh()` is the most expensive step — all beans created here
- Events fire in order: Starting → EnvironmentPrepared → ContextInitialized → Started → Ready
- Failed startup → `ApplicationFailedEvent`
- Banner can be customized or disabled: `spring.main.banner-mode=off`

---

## Q3. What is the Role of SpringFactoriesLoader?

### 📝 One-Liner
`SpringFactoriesLoader` loads class names from `META-INF/spring.factories` files to discover auto-configurations, listeners, and initializers.

### 🔑 Quick Answer
It's Spring's SPI (Service Provider Interface) mechanism — reads `META-INF/spring.factories` from all JARs on classpath, finds implementations keyed by interface type. This is HOW auto-configuration works. *(ye Spring ka plugin discovery mechanism hai — sab JARs ke spring.factories padhta hai)*

### 📖 How It Works
1. **Location**: Each JAR can have `META-INF/spring.factories`
2. **Format**: Key = interface/annotation class, Value = comma-separated implementation classes
3. **Loading**: `SpringFactoriesLoader.loadFactoryNames(type, classLoader)` returns matching classes
4. **Used for**:
   - Auto-configuration classes (`EnableAutoConfiguration` key)
   - `ApplicationContextInitializer` implementations
   - `ApplicationListener` implementations
   - `EnvironmentPostProcessor` implementations

### 💻 Code
```properties
# META-INF/spring.factories (Spring Boot 2.x)
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
  com.example.MyAutoConfiguration,\
  com.example.AnotherAutoConfiguration

org.springframework.context.ApplicationListener=\
  com.example.MyStartupListener
```
```java
// Spring Boot 3.x uses META-INF/spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports
// One class per line, no key=value format
com.example.MyAutoConfiguration
com.example.AnotherAutoConfiguration
```

### ⚠️ Pitfalls / Gotchas
- Spring Boot 3.x moved auto-configs to `AutoConfiguration.imports` file *(3.x mein spring.factories deprecated hai auto-config ke liye)*
- `spring.factories` still used for initializers, listeners, environment post-processors
- All entries loaded from ALL JARs — can cause conflicts if duplicate

### ⚡ Remember
- Spring Boot 2.x: `META-INF/spring.factories` for everything
- Spring Boot 3.x: `META-INF/spring/...AutoConfiguration.imports` for auto-config
- This is the SPI mechanism behind "magic" auto-configuration
- `@Conditional` annotations filter which loaded classes actually apply

---

## Q4. What Happens if application.yml and application.properties Both Exist?

### 📝 One-Liner
Both are loaded, with `.properties` taking higher precedence over `.yml` in the same location.

### 🔑 Quick Answer
Spring Boot loads BOTH files. If the same property exists in both, `.properties` wins over `.yml` at the same location. However, location precedence still applies — `file:./config/application.yml` beats `classpath:application.properties`. *(same jagah pe .properties jeetega, lekin file system classpath se upar hai)*

### 📖 How It Works
- Both `PropertiesPropertySourceLoader` and `YamlPropertySourceLoader` run
- Same property in both? `.properties` wins (loaded later = higher priority)
- **Precedence rules remain**: location > format
  ```
  file:./config/application.yml        (highest)
  file:./config/application.properties
  file:./application.yml
  file:./application.properties
  classpath:/config/application.yml
  classpath:/config/application.properties
  classpath:application.yml
  classpath:application.properties     (lowest)
  ```
- Profile-specific files always override base files regardless of format

### 🗣️ Answering Approach
"Both files are loaded. When a property exists in both at the same location, .properties takes precedence over .yml. But location precedence still applies — a .yml in file:./config/ will override a .properties in classpath:/. In practice, I'd recommend using one format per project to avoid confusion."

### ⚡ Remember
- Both loaded — NOT mutually exclusive
- Same location: `.properties` > `.yml`
- Different location: file system > classpath
- Best practice: pick ONE format for the project

---

## Q5. Difference Between @Configuration Class and Normal Class?

### 📝 One-Liner
`@Configuration` classes are CGLIB-proxied so that `@Bean` inter-method calls return the SAME singleton instance, unlike regular classes.

### 🔑 Quick Answer
`@Configuration` = full mode (CGLIB proxy). Calling one `@Bean` method from another returns the existing singleton from the container. A regular `@Component` class with `@Bean` methods runs in "lite" mode — no proxy, method calls create NEW instances each time. *(Configuration class ka proxy banta hai, isliye bean method dobara call karne pe wahi singleton milta hai)*

### 📖 How It Works
```java
// FULL mode (@Configuration) — CGLIB proxy
@Configuration
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // Returns SAME DataSource singleton!
    }
}

// LITE mode (@Component) — NO proxy
@Component
public class AppConfig {
    @Bean
    public DataSource dataSource() {
        return new HikariDataSource();
    }
    
    @Bean
    public JdbcTemplate jdbcTemplate() {
        return new JdbcTemplate(dataSource()); // Creates NEW DataSource! Bug!
    }
}
```

### 🗣️ Answering Approach
"@Configuration classes are enhanced with a CGLIB proxy at startup. This proxy intercepts @Bean method calls — if the bean already exists in the container, it returns the existing singleton instead of creating a new instance. This is called 'full mode.' A regular class with @Bean methods runs in 'lite mode' without the proxy, so inter-method calls create new instances, which can cause subtle bugs."

### ⚠️ Pitfalls / Gotchas
- `@Configuration(proxyBeanMethods = false)` = lite mode for performance (Spring Boot 2.2+) *(proxy off karna ho toh ye use karo)*
- Lite mode is faster (no CGLIB at startup) but loses inter-method singleton guarantee
- Spring Boot's own auto-configs use `proxyBeanMethods = false` for startup speed

### ⚡ Remember
- `@Configuration` = CGLIB proxy = inter-method calls return singleton
- `@Component` + `@Bean` = lite mode = new instance each call
- `proxyBeanMethods = false` → explicit lite mode for performance
- Spring Boot 3.x auto-configs default to `proxyBeanMethods = false`

---

## Q6. What is the Real Use of CommandLineRunner?

### 📝 One-Liner
`CommandLineRunner` runs code immediately after the application context is fully initialized, before the app starts accepting requests.

### 🔑 Quick Answer
It's a hook to execute startup logic — data seeding, cache warming, health checks, migration scripts — after all beans are created but before the app is "ready." Takes raw `String[] args`. `ApplicationRunner` is the same but takes parsed `ApplicationArguments`. *(app puri ready hone ke just pehle kuch kaam karna ho — migrations, cache warm-up — toh ye use karo)*

### 📖 How It Works
1. All beans created → context refreshed → embedded server started
2. `CommandLineRunner.run(String... args)` called
3. Multiple runners? Use `@Order` to control sequence
4. If runner throws exception → application startup FAILS

### 💻 Code
```java
@Component
@Order(1) // runs first
public class DatabaseMigrationRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        log.info("Running database migrations...");
        migrationService.migrate();
    }
}

@Component
@Order(2) // runs second
public class CacheWarmupRunner implements CommandLineRunner {
    @Override
    public void run(String... args) {
        log.info("Warming up cache...");
        cacheService.warmUp();
    }
}

// ApplicationRunner — same but with parsed args
@Component
public class ArgParsingRunner implements ApplicationRunner {
    @Override
    public void run(ApplicationArguments args) {
        if (args.containsOption("init")) {
            seedData();
        }
    }
}
```

### 🆚 vs. Comparison
| Feature | CommandLineRunner | ApplicationRunner | @PostConstruct | @EventListener |
|---------|------------------|-------------------|---------------|----------------|
| When | After context ready | After context ready | During bean init | On specific events |
| Input | Raw String[] args | Parsed ApplicationArguments | None | Event object |
| Scope | Application-wide | Application-wide | Per-bean | Flexible |
| Order | `@Order` supported | `@Order` supported | Per-bean only | `@Order` supported |

### ⚡ Remember
- Runs AFTER context refresh, BEFORE `ApplicationReadyEvent`
- Use cases: DB migration, cache warm-up, data seeding, health pre-checks
- Exception in runner → startup failure (fail-fast)
- `ApplicationRunner` if you need parsed arguments (`--key=value`)

---

## Q7. How Does Spring Boot Handle Exception Translation?

### 📝 One-Liner
Spring translates technology-specific exceptions (JDBC, JPA, Hibernate) into a consistent `DataAccessException` hierarchy via `@Repository` and `PersistenceExceptionTranslationPostProcessor`.

### 🔑 Quick Answer
`@Repository` marks a DAO for exception translation. `PersistenceExceptionTranslationPostProcessor` creates a proxy that catches vendor-specific exceptions (e.g., `ConstraintViolationException`) and wraps them in Spring's `DataAccessException` tree. *(vendor ki exception ko Spring ki standard exception mein convert kar deta hai)*

### 📖 How It Works
1. `@Repository` annotation triggers AOP proxy creation
2. `PersistenceExceptionTranslationPostProcessor` (auto-configured) wraps the bean
3. Proxy catches vendor exceptions → translates using `PersistenceExceptionTranslator`
4. Translators: `HibernateExceptionTranslator`, `JpaExceptionTranslator`, JDBC `SQLExceptionTranslator`

**Exception Hierarchy**:
```
DataAccessException (abstract root)
├── NonTransientDataAccessException
│   ├── DataIntegrityViolationException (constraint violations)
│   ├── DuplicateKeyException
│   └── EmptyResultDataAccessException
├── TransientDataAccessException
│   ├── CannotAcquireLockException
│   └── DeadlockLoserDataAccessException
├── RecoverableDataAccessException
└── UncategorizedDataAccessException
```

### 🗣️ Answering Approach
"Spring uses @Repository and a post-processor to create AOP proxies that intercept vendor-specific exceptions — like Hibernate's ConstraintViolationException — and translate them into Spring's DataAccessException hierarchy. This makes the service layer agnostic to the ORM vendor. For REST APIs, I combine this with @ControllerAdvice to map these to proper HTTP responses."

### ⚡ Remember
- `@Repository` = exception translation + component scanning
- `DataAccessException` is unchecked (RuntimeException)
- Vendor-independent: switch from Hibernate to MyBatis → same exceptions
- For REST: `@ControllerAdvice` + `@ExceptionHandler` maps to HTTP status

---

## Q8. Difference Between @EnableAutoConfiguration and @Import?

### 📝 One-Liner
`@EnableAutoConfiguration` triggers classpath-based conditional configuration from `spring.factories`, while `@Import` explicitly imports specific configuration classes.

### 🔑 Quick Answer
`@EnableAutoConfiguration` = automatic, conditional, discovers configs via `SpringFactoriesLoader`. `@Import` = manual, explicit, always loads the specified class. Auto-config uses `@Conditional*` to decide IF a config applies; `@Import` always applies. *(@EnableAutoConfiguration automatic hai, @Import manual hai — ek discover karta hai, dusra direct load karta hai)*

### 📖 How It Works

| Aspect | @EnableAutoConfiguration | @Import |
|--------|--------------------------|---------|
| Discovery | `spring.factories` / `AutoConfiguration.imports` | Explicit class reference |
| Conditional | ✅ `@ConditionalOnClass`, `@ConditionalOnBean` etc. | ❌ Always loaded |
| Use case | Auto-detect what the app needs | Explicitly compose configurations |
| Scope | All matching auto-configs from ALL JARs | Only specified classes |
| Override | Can exclude with `exclude=` | Remove the import |

### 💻 Code
```java
// @EnableAutoConfiguration — automatic discovery
@SpringBootApplication // includes @EnableAutoConfiguration
public class App { }

// Exclude specific auto-config
@SpringBootApplication(exclude = DataSourceAutoConfiguration.class)
public class App { }

// @Import — explicit, always loaded
@Configuration
@Import({SecurityConfig.class, CacheConfig.class})
public class AppConfig { }

// @Import with ImportSelector (dynamic)
@Import(MyImportSelector.class)
public class AppConfig { }
// ImportSelector can dynamically decide WHICH classes to import
```

### ⚡ Remember
- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- Auto-config: discovered + conditional + excludable
- `@Import`: explicit + unconditional + direct
- `ImportSelector`: programmatic @Import (used internally by auto-config)

---

## Q9. Why is Spring Boot Perfect for Microservices?

### 📝 One-Liner
Spring Boot provides embedded servers, auto-configuration, cloud-native integrations, and production-ready features that align perfectly with microservice requirements.

### 🔑 Quick Answer
Embedded server (no external Tomcat), fat JAR deployment, auto-config reduces boilerplate, Actuator for health/metrics, Spring Cloud for service discovery/config/circuit-breakers, minimal footprint per service. *(har microservice apna server lekar chalta hai, deploy karna easy, health check built-in)*

### 📖 How It Works
| Microservice Need | Spring Boot Feature |
|-------------------|-------------------|
| Independent deployment | Embedded server + fat JAR |
| Service discovery | Spring Cloud Eureka/Consul integration |
| Configuration management | Externalized config + Config Server |
| Health monitoring | Actuator endpoints |
| Fault tolerance | Resilience4j + Spring Cloud Circuit Breaker |
| API gateway | Spring Cloud Gateway |
| Distributed tracing | Micrometer + Zipkin integration |
| Containerization | Buildpack support, Docker-friendly |
| Fast startup | Minimal footprint, lazy init option |

### 🗣️ Answering Approach
"Spring Boot is ideal for microservices because each service can run independently with its embedded server and fat JAR — no external container needed. Auto-configuration minimizes boilerplate so teams can focus on business logic. Actuator provides production-ready monitoring. And Spring Cloud ecosystem adds service discovery, centralized config, circuit breakers, and distributed tracing — all the infrastructure microservices need."

### ⚡ Remember
- Embedded server = self-contained deployable unit
- Fat JAR = single artifact with all dependencies
- Actuator = `/health`, `/metrics`, `/info` out of the box
- Spring Cloud = Netflix OSS replacement for microservice infra

---

## Q10. Difference Between Fat JAR and Normal JAR?

### 📝 One-Liner
A fat JAR (uber JAR) packages the application AND all its dependencies into a single executable archive, while a normal JAR contains only the project's compiled classes.

### 🔑 Quick Answer
Normal JAR: contains only your `.class` files + resources — needs external classpath for dependencies. Fat JAR: contains your code + ALL dependency JARs nested inside — fully self-contained, run with `java -jar`. Spring Boot uses a custom `BOOT-INF/` layout. *(fat JAR mein sab kuch ek file mein — run karo aur chala jaye)*

### 📖 How It Works
```
Normal JAR:                      Fat JAR (Spring Boot):
├── com/                        ├── BOOT-INF/
│   └── example/                │   ├── classes/        ← your code
│       └── App.class           │   └── lib/            ← ALL dependency JARs
├── META-INF/                   ├── META-INF/
│   └── MANIFEST.MF             │   └── MANIFEST.MF
└── application.properties      └── org/springframework/boot/loader/
                                    └── JarLauncher.class  ← custom classloader
```

**How it runs**:
- `MANIFEST.MF` → `Main-Class: org.springframework.boot.loader.JarLauncher`
- `JarLauncher` creates custom classloader → loads `BOOT-INF/classes` + `BOOT-INF/lib/*.jar`
- Delegates to `Start-Class: com.example.App`

### 💻 Code
```xml
<!-- Spring Boot Maven Plugin creates fat JAR -->
<plugin>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-maven-plugin</artifactId>
</plugin>
```
```bash
# Build
mvn clean package    # creates app-0.0.1-SNAPSHOT.jar (fat JAR)

# Run directly
java -jar app-0.0.1-SNAPSHOT.jar

# Size comparison
# Normal JAR: ~50KB (your code only)
# Fat JAR: ~30MB+ (your code + ALL dependencies + embedded Tomcat)
```

### ⚡ Remember
- Fat JAR = `BOOT-INF/lib/` contains all dependency JARs
- Custom classloader (`JarLauncher`) loads nested JARs
- Docker: fat JAR = simple `COPY app.jar` + `java -jar`
- Layered JAR (Spring Boot 2.3+): better Docker caching with layers

---

## Q11. How Does Spring Boot Decide Server Port Priority?

### 📝 One-Liner
Server port follows Spring Boot's property precedence: command-line args > environment variables > application-{profile}.properties > application.properties > default (8080).

### 🔑 Quick Answer
The port is resolved from `server.port` property following the standard externalized config hierarchy. Command-line `--server.port=9090` beats everything; environment variable `SERVER_PORT` beats file-based; profile-specific beats base properties; hard-coded `@Value` or `WebServerFactoryCustomizer` can also set it. *(command line sabse upar, phir env variable, phir profile properties, phir default)*

### 📖 How It Works
**Priority (highest to lowest)**:
1. `--server.port=9090` (command-line arg)
2. `SPRING_APPLICATION_JSON` (inline JSON)
3. `SERVER_PORT` (OS environment variable / Kubernetes)
4. `server.port` in `application-{profile}.properties`
5. `server.port` in `application.properties`
6. `WebServerFactoryCustomizer` bean (programmatic)
7. Default: **8080**

```java
// Programmatic override
@Bean
public WebServerFactoryCustomizer<ConfigurableServletWebServerFactory> customizer() {
    return factory -> factory.setPort(9090);
}
```

### 🗣️ Answering Approach
"Spring Boot resolves server.port through its standard configuration precedence. Command-line arguments have the highest priority, followed by environment variables — which is perfect for Kubernetes where you set SERVER_PORT. Then profile-specific properties, then base application.properties. In production, I typically set it via environment variable for flexibility."

### ⚡ Remember
- `server.port=0` → random available port (useful for tests)
- Kubernetes/Docker: use `SERVER_PORT` env var
- `@LocalServerPort` annotation captures actual port in tests
- Default: 8080 (Tomcat), 8443 for HTTPS

---

## Q12. Why is Spring Boot Preferred for Cloud-Native Apps?

### 📝 One-Liner
Spring Boot provides 12-factor app compliance, containerization support, externalized config, health endpoints, and Spring Cloud integrations out of the box.

### 🔑 Quick Answer
Embedded server for container-friendly deployment, externalized config for environment portability, Actuator for Kubernetes probes, GraalVM native image support for fast startup, Buildpacks for container images without Dockerfile, Spring Cloud for service mesh patterns. *(cloud ke liye sab built-in aata hai — health check, config, container support)*

### 📖 How It Works
| 12-Factor Principle | Spring Boot Support |
|--------------------|--------------------|
| I. Codebase | Git + standard project structure |
| II. Dependencies | Maven/Gradle + starters |
| III. Config | Externalized config, ConfigServer |
| IV. Backing Services | Auto-configured DataSource, Redis, Kafka |
| V. Build/Release/Run | Fat JAR + Docker layered images |
| VI. Processes | Stateless, no in-process sessions |
| VII. Port Binding | Embedded server, self-contained |
| VIII. Concurrency | Scalable via containers |
| IX. Disposability | Fast startup, graceful shutdown |
| X. Dev/Prod Parity | Profiles, Testcontainers |
| XI. Logs | Console logging → log aggregator |
| XII. Admin Processes | CommandLineRunner, Actuator |

### 🗣️ Answering Approach
"Spring Boot is cloud-native by design. It follows 12-factor app principles — externalized config, stateless processes, port binding via embedded servers. Actuator provides health and readiness probes for Kubernetes. Buildpacks create OCI images without a Dockerfile. Spring Cloud adds service discovery, config server, and circuit breakers. And with GraalVM native image support, startup drops from seconds to milliseconds."

### ⚡ Remember
- Kubernetes: `/actuator/health/liveness` + `/actuator/health/readiness` probes
- Buildpacks: `mvn spring-boot:build-image` → OCI image without Dockerfile
- GraalVM native: `spring-boot:build-image -Pnative` for ~50ms startup
- Graceful shutdown: `server.shutdown=graceful` (Spring Boot 2.3+)

---
