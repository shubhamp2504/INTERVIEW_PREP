# 🍃 Spring Boot — Fundamentals & Getting Started (Q1–Q5)

> **Source**: Spring Boot Interview Questions (Basic → Advanced) + General  
> **Coverage**: What is Spring Boot, features, Spring vs Spring Boot, config files, DevTools

---

<a id="q1"></a>
## Q1. What is Spring Boot?

### 📝 One-Liner
Spring Boot is an **opinionated framework on top of Spring** that auto-configures everything so you can build production-ready apps with **minimal configuration** — just write code and run.

### 🔑 Quick Answer
**Spring Boot** = Spring Framework + opinionated defaults + embedded server + auto-configuration. Without Spring Boot, you'd manually configure `DataSource`, `ViewResolver`, `DispatcherServlet`, transaction managers, and dozens of XML/Java config files. Spring Boot looks at your classpath, detects what libraries you have, and **auto-configures beans** for you. Add `spring-boot-starter-web` → you get Tomcat, Jackson, Spring MVC, all auto-configured. Add `spring-boot-starter-data-jpa` → you get HikariCP, Hibernate, JPA, all wired up. You write `@SpringBootApplication` + `main()` and it just works. *(Spring Boot = Spring Framework + automatic configuration + embedded server — bina XML likhe seedha code likho aur chalao)*

### 📖 How It Works (Detailed Explanation)

```
Traditional Spring (pre-Boot):
  1. Create web.xml → configure DispatcherServlet
  2. Create applicationContext.xml → define beans
  3. Configure DataSource, TransactionManager, ViewResolver manually
  4. Package as WAR → deploy to external Tomcat
  5. 200+ lines of config before writing a single endpoint

Spring Boot:
  1. Add starter dependencies (spring-boot-starter-web)
  2. Write @SpringBootApplication class
  3. Write @RestController with endpoints
  4. Run main() → embedded Tomcat starts → app is live
  5. ~10 lines of config in application.yml (if any)
```

**Key concept**: Spring Boot doesn't replace Spring — it sits **on top** of Spring Framework, preconfiguring everything with sensible defaults. You can override any default. It follows the **convention over configuration** philosophy — if 90% of projects configure things the same way, make that the default.

### 🗣️ Interview Script
"Spring Boot is an opinionated framework built on top of the Spring ecosystem that dramatically reduces boilerplate configuration. In a traditional Spring project, you'd write hundreds of lines of XML or Java config for DataSource, DispatcherServlet, transaction management, and so on. Spring Boot eliminates this by looking at your classpath and auto-configuring beans based on what libraries you've included. If I add spring-boot-starter-web, I automatically get an embedded Tomcat server, Jackson for JSON, and Spring MVC — all configured and ready. I just write a main method with @SpringBootApplication, define my controllers, and the app runs standalone as a JAR. It follows convention over configuration — sensible defaults that I can override when needed."

### 💻 Code Example

```java
// ✅ Entire Spring Boot application — that's it
@SpringBootApplication   // = @Configuration + @EnableAutoConfiguration + @ComponentScan
public class MyApplication {
    public static void main(String[] args) {
        SpringApplication.run(MyApplication.class, args);
        // Embedded Tomcat starts on port 8080
        // All beans auto-configured
        // Ready to serve requests
    }
}

@RestController
@RequestMapping("/api/users")
public class UserController {
    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    @GetMapping
    public List<UserDTO> getAll() {
        return userService.findAll();
    }
}
```

```xml
<!-- ❌ Traditional Spring — web.xml alone was this long -->
<web-app>
    <servlet>
        <servlet-name>dispatcher</servlet-name>
        <servlet-class>org.springframework.web.servlet.DispatcherServlet</servlet-class>
        <init-param>
            <param-name>contextConfigLocation</param-name>
            <param-value>/WEB-INF/applicationContext.xml</param-value>
        </init-param>
        <load-on-startup>1</load-on-startup>
    </servlet>
    <servlet-mapping>
        <servlet-name>dispatcher</servlet-name>
        <url-pattern>/</url-pattern>
    </servlet-mapping>
</web-app>
<!-- Plus applicationContext.xml, mvc-config.xml, datasource.xml... -->
```

### ⚠️ Common Pitfalls
- **"Spring Boot is a new framework"** — No, it's a layer on top of Spring Framework with auto-configuration
- **"Spring Boot generates code"** — No, it auto-configures beans at runtime based on classpath scanning
- **Overriding defaults** — sometimes auto-configuration conflicts with custom config; use `@ConditionalOnMissingBean` or `spring.autoconfigure.exclude`

### ⚡ Remember (Quick Recall)
- Spring Boot = **Spring + Auto-configuration + Embedded server + Starters**
- `@SpringBootApplication` = `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`
- Convention over configuration — sensible defaults, override when needed
- Run as JAR with embedded Tomcat — no external server needed

### 🔗 Related Topics
- [Auto-configuration internals](04-springboot-internals-epam.md#q1)
- [Spring Boot Starters](04-springboot-internals-epam.md#q4)
- [Spring vs Spring Boot](#q3)

---

<a id="q2"></a>
## Q2. What are the main features of Spring Boot?

### 📝 One-Liner
Spring Boot's killer features: **auto-configuration**, **starter dependencies**, **embedded server**, **Actuator** for monitoring, **DevTools** for development, and **opinionated defaults** — all designed for zero-config production apps.

### 🔑 Quick Answer
**(1) Auto-Configuration** — `@EnableAutoConfiguration` detects classpath libraries and configures beans automatically (H2 on classpath → in-memory DataSource configured). **(2) Starter Dependencies** — `spring-boot-starter-web`, `spring-boot-starter-data-jpa` — curated dependency groups that pull all required libraries with compatible versions. **(3) Embedded Server** — Tomcat/Jetty/Undertow runs inside the JAR; no external server deployment. **(4) Actuator** — production-ready endpoints for health, metrics, env, thread dumps. **(5) Spring Boot DevTools** — auto-restart on code change, LiveReload, relaxed security for dev. **(6) Externalized Configuration** — `application.yml`, env variables, profiles, ConfigServer — config outside code. **(7) Spring Initializr** — web tool to scaffold projects in seconds. *(Auto-configure karta hai, embedded server deta hai, starter se dependency manage hoti hai, Actuator se monitoring hoti hai — sab built-in)*

### 📖 How It Works (Detailed Explanation)

```
Feature Map:

┌──────────────────────────────────────────────┐
│              Spring Boot Features             │
├──────────────────┬───────────────────────────┤
│ Auto-Config      │ Classpath → beans auto-   │
│                  │ configured (@Conditional)  │
├──────────────────┼───────────────────────────┤
│ Starters         │ Curated dependency bundles │
│                  │ (web, data-jpa, security)  │
├──────────────────┼───────────────────────────┤
│ Embedded Server  │ Tomcat/Jetty/Undertow     │
│                  │ inside JAR → java -jar    │
├──────────────────┼───────────────────────────┤
│ Actuator         │ /health, /metrics, /env   │
│                  │ /threaddump, /loggers     │
├──────────────────┼───────────────────────────┤
│ DevTools         │ Auto-restart, LiveReload  │
│                  │ Dev-only defaults          │
├──────────────────┼───────────────────────────┤
│ Externalized     │ application.yml, profiles │
│ Config           │ env vars, Spring Cloud    │
├──────────────────┼───────────────────────────┤
│ CLI / Initializr │ start.spring.io scaffold  │
└──────────────────┴───────────────────────────┘
```

### 🗣️ Interview Script
"Spring Boot has several key features. First, auto-configuration — it detects libraries on the classpath and configures beans automatically using @Conditional annotations. If I have HikariCP and PostgreSQL driver, it auto-creates a DataSource without me writing config. Second, starter dependencies — curated dependency bundles like spring-boot-starter-web that bring Tomcat, Jackson, and Spring MVC with compatible versions. Third, an embedded server — the app runs as a standalone JAR with Tomcat embedded, no WAR deployment needed. Fourth, Actuator — production-ready endpoints for health checks, metrics, and environment info. Fifth, DevTools for development productivity — auto-restart on code changes and LiveReload. And sixth, externalized configuration with application.yml, profiles, and environment variables so the same code runs across dev, staging, and production."

### 💻 Code Example

```xml
<!-- ✅ One starter replaces 10+ individual dependencies -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <!-- Brings: spring-webmvc, spring-web, jackson, tomcat-embed, validation -->
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
    <!-- Brings: hibernate, spring-data-jpa, HikariCP, spring-orm -->
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-actuator</artifactId>
    <!-- Brings: micrometer, health endpoints, metrics -->
</dependency>

<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>
</dependency>
```

```yaml
# ✅ Actuator configuration
management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics,loggers,env
  endpoint:
    health:
      show-details: when_authorized
```

### ⚡ Remember (Quick Recall)
- **7 features**: Auto-config, Starters, Embedded server, Actuator, DevTools, Externalized config, Initializr
- Starters = curated dependency groups with compatible versions
- Embedded Tomcat = `java -jar app.jar` and it's running
- Actuator = production monitoring out of the box

### 🔗 Related Topics
- [Auto-Configuration internals](04-springboot-internals-epam.md#q1)
- [Actuator deep-dive](04-springboot-internals-epam.md#q2)
- [DevTools](#q5)
- [Environment profiles](07-project-infrastructure-decisions.md#q2)

---

<a id="q3"></a>
## Q3. What is the difference between Spring and Spring Boot?

### 📝 One-Liner
**Spring** is the core framework (DI, AOP, MVC, Data, Security); **Spring Boot** is the auto-configuration layer that eliminates boilerplate and lets you start coding immediately.

### 🔑 Quick Answer
**Spring Framework** provides the building blocks — Dependency Injection container, AOP, Spring MVC, Spring Data, Spring Security, Transaction Management. But you configure everything yourself: XML or Java config for every bean, DataSource, DispatcherServlet, ViewResolver. **Spring Boot** wraps Spring Framework and adds: (1) Auto-configuration — beans created automatically. (2) Starter POMs — dependency management. (3) Embedded server — no external Tomcat. (4) Opinionated defaults — production-ready out of the box. **Analogy**: Spring = engine + parts; Spring Boot = fully assembled car ready to drive. *(Spring = sab kuch manually configure karo; Spring Boot = sab automatic, seedha code likho)*

### 📖 How It Works (Detailed Explanation)

| Aspect | Spring Framework | Spring Boot |
|--------|-----------------|-------------|
| **Configuration** | Manual (XML / Java Config) | Auto-configuration |
| **Dependencies** | Manage individually + version compatibility | Starter POMs with BOM |
| **Server** | External (deploy WAR to Tomcat) | Embedded (JAR with Tomcat inside) |
| **Startup** | Configure `DispatcherServlet`, `ContextLoaderListener` | `@SpringBootApplication` + `main()` |
| **Monitoring** | Build from scratch | Actuator built-in |
| **Dev experience** | Slow restart, manual deploy | DevTools auto-restart |
| **Config files** | `applicationContext.xml`, `web.xml` | `application.yml` |
| **Learning curve** | Higher (many XML/config concepts) | Lower (convention over configuration) |
| **Flexibility** | Maximum (you decide everything) | High (override any default) |
| **When to use** | Legacy apps, extreme customization | New projects, microservices, rapid dev |

### 🗣️ Interview Script
"Spring Framework is the core ecosystem — it gives you DI, AOP, MVC, Data access, Security, and more. But in a traditional Spring project, you manually configure every piece: DataSource beans, DispatcherServlet registration, transaction managers, view resolvers — often hundreds of lines of XML or Java config before writing a single business endpoint. Spring Boot sits on top of Spring Framework and adds auto-configuration, starter dependencies, and an embedded server. With Spring Boot, I add spring-boot-starter-web to my POM, write @SpringBootApplication, and I have a running REST API with zero configuration. The core Spring APIs are the same — @Autowired, @Transactional, @Service all work identically. Spring Boot just removes the boilerplate so I can focus on business logic. I think of Spring as the engine and Spring Boot as the fully assembled car."

### 💻 Code Example

```java
// ❌ Traditional Spring — DataSource configuration
@Configuration
public class DataSourceConfig {
    @Bean
    public DataSource dataSource() {
        DriverManagerDataSource ds = new DriverManagerDataSource();
        ds.setDriverClassName("org.postgresql.Driver");
        ds.setUrl("jdbc:postgresql://localhost:5432/mydb");
        ds.setUsername("user");
        ds.setPassword("pass");
        return ds;
    }

    @Bean
    public LocalContainerEntityManagerFactoryBean entityManagerFactory() {
        LocalContainerEntityManagerFactoryBean em = new LocalContainerEntityManagerFactoryBean();
        em.setDataSource(dataSource());
        em.setPackagesToScan("com.myapp.model");
        em.setJpaVendorAdapter(new HibernateJpaVendorAdapter());
        Properties props = new Properties();
        props.put("hibernate.dialect", "org.hibernate.dialect.PostgreSQLDialect");
        props.put("hibernate.hbm2ddl.auto", "validate");
        em.setJpaProperties(props);
        return em;
    }

    @Bean
    public PlatformTransactionManager transactionManager() {
        return new JpaTransactionManager(entityManagerFactory().getObject());
    }
}

// ✅ Spring Boot — same result, ZERO config
// Just add spring-boot-starter-data-jpa + PostgreSQL driver to POM
// application.yml:
//   spring.datasource.url: jdbc:postgresql://localhost:5432/mydb
//   spring.datasource.username: user
//   spring.datasource.password: pass
// DataSource, EntityManagerFactory, TransactionManager → all auto-configured!
```

### ⚡ Remember (Quick Recall)
- **Spring** = framework/toolkit (DI, AOP, MVC, Data, Security)
- **Spring Boot** = auto-configured Spring + embedded server + starters
- Same annotations work in both (`@Autowired`, `@Transactional`, `@Service`)
- Spring Boot doesn't replace Spring — it wraps it with sensible defaults

### 🔗 Related Topics
- [How Auto-Configuration works](04-springboot-internals-epam.md#q1)
- [What is Spring Boot?](#q1)
- [DI Internals](01-spring-framework-internals.md#q7)

---

<a id="q4"></a>
## Q4. What is the difference between application.properties and application.yml?

### 📝 One-Liner
Both configure Spring Boot apps — `.properties` uses **flat key=value** format; `.yml` uses **hierarchical YAML** with indentation — functionally equivalent, `.yml` is more readable for nested configs.

### 🔑 Quick Answer
**application.properties**: `spring.datasource.url=jdbc:postgresql://localhost/mydb` — flat, one key-value per line, no hierarchy. **application.yml**: uses YAML with indentation for hierarchy — `spring: datasource: url: jdbc:postgresql://localhost/mydb`. Both are loaded by Spring Boot's `PropertySource` mechanism and produce identical results. **Why choose YAML**: (1) Nested config is more readable (especially with deep hierarchies). (2) Multi-document support (`---` separator for profiles in one file). (3) Lists are cleaner. **Why choose properties**: (1) Simpler for flat configs. (2) No indentation sensitivity (YAML breaks on wrong indent). (3) IDE support is slightly better for auto-completion. **In my projects**: I use `application.yml` because most configs are nested (spring.datasource, management.endpoints) and YAML reads cleaner. *(Dono same kaam karte hain — .yml hierarchy dikhata hai, .properties flat hota hai; team jo prefer kare wo use karo)*

### 📖 How It Works (Detailed Explanation)

```properties
# ✅ application.properties — flat format
server.port=8080
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=admin
spring.datasource.hikari.maximum-pool-size=15
spring.jpa.hibernate.ddl-auto=validate
management.endpoints.web.exposure.include=health,info,metrics
logging.level.com.myapp=DEBUG
```

```yaml
# ✅ application.yml — hierarchical format (same config)
server:
  port: 8080

spring:
  datasource:
    url: jdbc:postgresql://localhost:5432/mydb
    username: admin
    hikari:
      maximum-pool-size: 15
  jpa:
    hibernate:
      ddl-auto: validate

management:
  endpoints:
    web:
      exposure:
        include: health,info,metrics

logging:
  level:
    com.myapp: DEBUG
```

### 🗣️ Interview Script
"Both files configure a Spring Boot application and are functionally equivalent — Spring loads them into the same property source. The difference is format: application.properties uses flat key=value pairs, while application.yml uses YAML's hierarchical indentation-based structure. I prefer YAML in my projects because most Spring Boot configs are deeply nested — spring.datasource.hikari.maximum-pool-size is much more readable as a YAML tree than a long dotted key. YAML also supports multi-document profiles with the triple-dash separator, and lists are cleaner. The downside of YAML is indentation sensitivity — one wrong space breaks the config silently. Properties files are simpler and slightly better supported by IDE auto-completion. Ultimately it's a team preference, and both produce identical runtime behavior."

### 💻 Code Example

```yaml
# ✅ YAML advantage: multi-profile in one file
spring:
  application:
    name: user-service

---
spring:
  config:
    activate:
      on-profile: dev
  datasource:
    url: jdbc:h2:mem:devdb

---
spring:
  config:
    activate:
      on-profile: prod
  datasource:
    url: jdbc:postgresql://prod-db:5432/users
```

```yaml
# ✅ YAML advantage: lists are cleaner
management:
  endpoints:
    web:
      exposure:
        include:
          - health
          - info
          - metrics
          - loggers
```

```properties
# Same in properties — less readable
management.endpoints.web.exposure.include=health,info,metrics,loggers
```

### 🆚 Quick Comparison

| Aspect | `.properties` | `.yml` |
|--------|--------------|--------|
| **Format** | Flat key=value | Hierarchical indent |
| **Readability** | Good for flat | Better for nested |
| **Indentation** | Not sensitive | Sensitive (wrong space = broken) |
| **Multi-profile** | Separate files only | `---` separator in one file |
| **Lists** | Comma-separated string | Proper YAML list (`- item`) |
| **IDE support** | Excellent | Good (improving) |
| **Both can be used** | ✅ at same time | ✅ properties takes priority |

### ⚠️ Common Pitfalls
- **Mixing tabs and spaces in YAML** — YAML only allows spaces for indentation; tabs cause parse errors
- **Using both simultaneously** — if both exist, `application.properties` values override `application.yml` for same keys
- **YAML boolean gotcha** — `on`, `yes`, `true` all parse as boolean `true` in YAML; quote strings like `"on"` if needed

### ⚡ Remember (Quick Recall)
- **Functionally identical** — both configure Spring Boot the same way
- `.yml` = more readable for nested configs, multi-profile in one file
- `.properties` = simpler, no indentation issues
- YAML: **spaces only**, no tabs
- Team preference — pick one and be consistent

### 🔗 Related Topics
- [Environment profiles](07-project-infrastructure-decisions.md#q2)
- [@Value vs @ConfigurationProperties](04-springboot-internals-epam.md#q3)

---

<a id="q5"></a>
## Q5. What is Spring Boot DevTools?

### 📝 One-Liner
DevTools provides **automatic restart** on code changes, **LiveReload** for browser refresh, **relaxed caching** for templates, and **dev-only defaults** — speeding up the development feedback loop.

### 🔑 Quick Answer
`spring-boot-devtools` is a development-time dependency that makes the edit-compile-test cycle faster. **(1) Automatic Restart** — when you change a Java file, the app restarts in ~2 seconds using two classloaders (base classloader for libraries stays loaded; restart classloader reloads only your classes). **(2) LiveReload** — triggers browser refresh when static resources change (HTML/CSS/JS). **(3) Property Defaults** — disables template caching (Thymeleaf), enables `H2 console`, turns on verbose error pages. **(4) Remote DevTools** — allows remote application restart (rarely used). **Important**: DevTools is automatically disabled in production — it's excluded from `java -jar` packaging. *(DevTools = code change karo, 2 second me app restart ho jaata hai — development speed badh jaati hai)*

### 📖 How It Works (Detailed Explanation)

```
DevTools Restart Mechanism:

Two ClassLoaders:
┌────────────────────────────┐
│  Base ClassLoader          │ ← Loads library JARs (Spring, Jackson, Hibernate)
│  (never restarted)         │    Kept in memory — no reload overhead
├────────────────────────────┤
│  Restart ClassLoader       │ ← Loads YOUR classes (com.myapp.*)
│  (restarted on change)     │    Thrown away + recreated on file change
└────────────────────────────┘

Trigger: .class file changed on disk
  → DevTool's FileWatcher detects change
  → Restart ClassLoader destroyed + recreated
  → Only your classes reload (~1-3 seconds)
  → vs full cold start (~10-15 seconds)
```

### 🗣️ Interview Script
"Spring Boot DevTools is a developer productivity module that speeds up the feedback loop. The main feature is automatic restart — when I save a Java file, the app restarts in about 2 seconds. It achieves this speed with a dual-classloader trick: library JARs are loaded by a base classloader that's never restarted, and only my application classes are reloaded via a restart classloader. This is much faster than a cold start. It also includes LiveReload for browser auto-refresh, disables template caching for instant changes, and enables dev-friendly defaults like the H2 console. It's completely safe for development because it's automatically excluded from the production JAR — the dependency is marked as optional and the Spring Boot Maven plugin strips it during packaging."

### 💻 Code Example

```xml
<!-- ✅ Add to pom.xml — development only -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-devtools</artifactId>
    <scope>runtime</scope>
    <optional>true</optional>  <!-- not transitive to other modules -->
</dependency>
```

```yaml
# ✅ DevTools auto-applies these defaults (no config needed):
# spring.thymeleaf.cache=false          ← template changes visible instantly
# spring.h2.console.enabled=true        ← H2 web console available
# spring.mvc.log-resolved-exception=true

# Customize restart behavior:
spring:
  devtools:
    restart:
      enabled: true
      additional-paths: src/main/resources   # watch these too
      exclude: static/**,public/**           # don't restart for static files
    livereload:
      enabled: true       # browser auto-refresh
```

### ⚠️ Common Pitfalls
- **DevTools in production** — it's auto-excluded from fat JAR, but verify (`java -jar app.jar` won't include it)
- **IDE trigger** — IntelliJ needs "Build → Build Project" (Ctrl+F9) or enable "Build project automatically" in settings; Eclipse auto-compiles on save
- **Restart vs hot-swap** — DevTools does a full context restart (fast), not JVM hot-swap; some changes (new beans, config) require full restart anyway
- **Classloader leaks** — some libraries cache class references; DevTools restart can cause `ClassCastException` with serialized objects across restarts

### ⚡ Remember (Quick Recall)
- **DevTools** = auto-restart (~2s) + LiveReload + dev defaults
- Dual classloader: base (libraries, kept) + restart (your code, reloaded)
- **Auto-excluded** from production JAR
- Mark `<optional>true</optional>` in POM
- Needs IDE build trigger (IntelliJ: Ctrl+F9 or auto-build)

### 🔗 Related Topics
- [Spring Boot features overview](#q2)
- [Production logging config](../../production-debugging/04-services-ops-infra.md)
- [Auto-configuration](04-springboot-internals-epam.md#q1)
