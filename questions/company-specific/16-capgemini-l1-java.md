# 🏢 Capgemini L1 — Java Developer Interview (Q1–Q10)

> **Source**: Real Capgemini L1 interview experience — Java Developer role (2026)  
> **Format**: Interview narrative with detailed answers + cross-references to existing topic coverage  
> **Level**: 2-4 YOE Java Developer  
> **Note**: Existing [14-capgemini-qa-interview.md](14-capgemini-qa-interview.md) covers QA/Testing role; this covers Java Developer L1 role

---

<a id="q1"></a>
## Q1. Tell me about yourself and your current project

### 📝 One-Liner
Structured introduction: current role → tech stack → key contributions → why this role.

### 🔑 Quick Answer
**Template for 2-4 YOE Java Developer:**
"I'm a Java Developer with [X] years of experience. Currently at [Company], I work on [project name] — a [microservices/monolith] application built with Spring Boot, REST APIs, and [database]. My responsibilities include developing new features, writing unit tests, and collaborating on code reviews. I recently [specific achievement: optimized a slow API, implemented a caching layer, migrated a monolith to microservices]. I'm excited about this role at Capgemini because [specific reason]."

### 🗣️ Answering Approach
"Start with current role and tech stack. Mention one specific technical achievement that shows impact. Keep it under 2 minutes. End with why you want this specific role — don't be generic."

*(Apne baare mein bolo — role, tech stack, ek achievement, aur is job ke liye kyun interested ho)*

### ⚡ Remember
- Cross-ref: [Introduction template → hr-behavioral/01-project-behavioral.md Q1](../hr-behavioral/01-project-behavioral.md#q1)

---

<a id="q2"></a>
## Q2. Which Java version are you using? What features were introduced in Java 8?

### 📝 One-Liner
Java 8 introduced Lambda expressions, Stream API, Optional, default methods in interfaces, new Date/Time API, and method references.

### 🔑 Quick Answer
**Key Java 8 features:**
1. **Lambda Expressions** — `(a, b) -> a + b` — concise functional syntax
2. **Stream API** — `list.stream().filter().map().collect()` — functional data processing
3. **Optional** — `Optional.ofNullable(value)` — null safety
4. **Default methods** — methods with body in interfaces
5. **Method References** — `String::toUpperCase` — shorthand for lambdas
6. **New Date/Time API** — `LocalDate`, `LocalDateTime`, `ZonedDateTime`
7. **Functional Interfaces** — `@FunctionalInterface`, `Predicate`, `Function`, `Consumer`

*(Java 8 mein sabse important — Lambda aur Streams. Interview mein coding problem typically stream se solve karte hain)*

### ⚡ Remember
- Cross-ref: [Java 8 deep dive → core/05-java8-streams-functional.md](../languages/java/core/05-java8-streams-functional.md)
- Cross-ref: [Java 8 vs 11 vs 17 vs 21 → core/17-java-evolution-8to21.md](../languages/java/core/17-java-evolution-8to21.md)

---

<a id="q3"></a>
## Q3. What is a Lambda Expression? What advantages does it provide?

### 📝 One-Liner
Lambda = anonymous function enabling functional programming in Java. Advantages: less boilerplate, readable code, enables Stream API, supports functional interfaces.

### 🔑 Quick Answer
```java
// Before Lambda (anonymous inner class)
Comparator<String> comp = new Comparator<String>() {
    @Override
    public int compare(String a, String b) { return a.compareTo(b); }
};

// After Lambda — same logic, 1 line
Comparator<String> comp = (a, b) -> a.compareTo(b);

// Method reference — even shorter
Comparator<String> comp = String::compareTo;
```

**Advantages:**
1. **Less boilerplate** — no anonymous class ceremony
2. **Readable** — intent is clear at a glance
3. **Enables functional patterns** — map, filter, reduce via Streams
4. **Lazy evaluation** — Streams with lambdas are evaluated lazily

### ⚡ Remember
- Cross-ref: [Lambda deep dive → core/05-java8-streams-functional.md](../languages/java/core/05-java8-streams-functional.md)

---

<a id="q4"></a>
## Q4. Find vowels from a given string using Streams

> 🔥 **Live coding question** in Capgemini L1

### 📝 One-Liner
Use `chars()` stream on string, filter for vowels, collect to string or list.

### 🔑 Quick Answer
```java
// Find all vowels from a string
String input = "Capgemini Interview";
String vowels = input.chars()
    .mapToObj(c -> (char) c)
    .filter(c -> "aeiouAEIOU".indexOf(c) != -1)
    .map(String::valueOf)
    .collect(Collectors.joining());
// Output: "aeiiiie"

// Count vowels
long vowelCount = input.chars()
    .filter(c -> "aeiouAEIOU".indexOf(c) != -1)
    .count();
// Output: 7

// Unique vowels
Set<Character> uniqueVowels = input.chars()
    .mapToObj(c -> Character.toLowerCase((char) c))
    .filter(c -> "aeiou".indexOf(c) != -1)
    .collect(Collectors.toSet());
// Output: [a, e, i]
```

### 🗣️ Answering Approach
"I'll use String's chars() stream, filter each character against a vowel set, and collect. I use indexOf on a vowel string rather than a complex regex — it's readable and efficient."

*(String ka chars() stream lo, filter karo vowels ke liye, collect karo — simple aur clean)*

---

<a id="q5"></a>
## Q5. Reverse a string using Streams

> 🔥 **Live coding question** in Capgemini L1

### 📝 One-Liner
Convert to char array, use IntStream to traverse in reverse, or use StringBuilder.reverse().

### 🔑 Quick Answer
```java
// Method 1: Using Stream + reduce
String input = "Capgemini";
String reversed = input.chars()
    .mapToObj(c -> String.valueOf((char) c))
    .reduce("", (a, b) -> b + a);
// Output: "inimegpaC"

// Method 2: Using StringBuilder (recommended for production)
String reversed2 = new StringBuilder(input).reverse().toString();

// Method 3: Using Stream + collect with custom joining
String reversed3 = IntStream.range(0, input.length())
    .mapToObj(i -> String.valueOf(input.charAt(input.length() - 1 - i)))
    .collect(Collectors.joining());
```

### 🗣️ Answering Approach
"If the interviewer wants a Stream-based solution, I use chars() with reduce to build the string in reverse. But for production code, I'd use StringBuilder.reverse() — it's O(n) and doesn't create intermediate string objects."

### ⚡ Remember
- Cross-ref: [String reverse patterns → core/08-java-coding-string-ops.md Q4](../languages/java/core/08-java-coding-string-ops.md#q4)

---

<a id="q6"></a>
## Q6. What types of logging are there? (Log4j, SLF4J, Logback)

### 📝 One-Liner
**SLF4J** = facade/interface for logging, **Logback** = default implementation in Spring Boot, **Log4j2** = alternative high-performance implementation.

### 🔑 Quick Answer

| Component | Role | Analogy |
|-----------|------|---------|
| **SLF4J** | Logging API (interface) | JDBC interface |
| **Logback** | Default implementation | MySQL JDBC driver |
| **Log4j2** | Alternative implementation | PostgreSQL JDBC driver |

**Why SLF4J + Logback (Spring Boot default):**
- SLF4J provides abstraction → switch implementations without code change
- Logback is faster than Log4j 1.x, auto-reloads config
- Spring Boot auto-configures Logback

```java
// ✅ Always use SLF4J facade (NEVER import Log4j directly)
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

private static final Logger log = LoggerFactory.getLogger(OrderService.class);

// Log levels (TRACE < DEBUG < INFO < WARN < ERROR)
log.debug("Processing order: {}", orderId);  // parameterized (no string concat)
log.info("Order created: {} for user: {}", orderId, userId);
log.error("Payment failed for order: {}", orderId, exception); // exception as last arg
```

### 🗣️ Answering Approach
"SLF4J is the logging facade (interface) and Logback is the implementation — Spring Boot uses this combination by default. I always code against SLF4J, never Log4j directly. Key practices: use parameterized messages with {} placeholders instead of string concatenation for performance, use appropriate log levels — DEBUG for developer details, INFO for business events, WARN for recoverable issues, ERROR for failures. In production, I configure JSON-structured logging for ELK integration."

---

<a id="q7"></a>
## Q7. Why is logging important in Spring Boot applications?

### 📝 One-Liner
Logging is your only visibility into production behavior — debugging, auditing, monitoring, alerting, and compliance all depend on proper logging.

### 🔑 Quick Answer
**Why logging matters:**
1. **Production debugging** — can't attach debugger in prod; logs are your only tool
2. **Distributed tracing** — traceId in logs lets you follow a request across 10 microservices
3. **Monitoring & alerting** — error rate from logs triggers PagerDuty alerts
4. **Audit trail** — who did what, when (compliance: PCI-DSS, HIPAA)
5. **Performance analysis** — log response times → identify slow endpoints
6. **Incident response** — what happened before the crash?

**Spring Boot logging best practices:**
```yaml
# application.yml
logging:
  level:
    root: WARN
    com.myapp: INFO           # app logs at INFO
    org.hibernate.SQL: DEBUG  # see SQL queries in dev
  pattern:
    console: "%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - traceId=%X{traceId} - %msg%n"
```

### 🗣️ Answering Approach
"Logging is the foundation of production observability. I can't attach a debugger to a production server — logs are my primary debugging tool. I always include a traceId from distributed tracing (Spring Sleuth) so I can follow one user's request across all microservices. I structure logs as JSON for ELK/Grafana integration. Key rule: log what you need to debug at 3 AM when something breaks, but never log sensitive data like passwords or PII."

### ⚡ Remember
- Cross-ref: [Observability deep dive → cloud-devops/01-observability-alerting.md](../cloud-devops/01-observability-alerting.md)
- Cross-ref: [Logging & monitoring design → system-design/08-backend-scenario-debugging.md Q12](../system-design/08-backend-scenario-debugging.md#q12)

---

<a id="q8"></a>
## Q8. Which design patterns have you used in your project?

### 📝 One-Liner
Commonly used in Java projects: Singleton (Spring beans), Factory (DriverFactory), Strategy (payment/notification), Observer (event listeners), Builder (complex objects).

### 🔑 Quick Answer

| Pattern | Where I Used It | How |
|---------|----------------|-----|
| **Singleton** | Spring beans (`@Service`, `@Component`) | All Spring beans are singleton by default |
| **Factory** | `DriverFactory.createDriver("chrome")` | Create objects without exposing instantiation logic |
| **Strategy** | Payment methods (UPI, Card, Wallet) | Switch behavior at runtime without if-else chains |
| **Observer** | `@EventListener` in Spring | Loose coupling between components |
| **Builder** | Complex DTO construction | `User.builder().name("X").email("Y").build()` |
| **Template Method** | `BaseTest` with hooks | Base class defines skeleton, subclasses override steps |

### 🗣️ Answering Approach
"In Spring Boot, I use design patterns daily without always calling them by name. Every @Service bean is a Singleton. I use Factory pattern in test automation for browser creation. Strategy pattern for payment processing where each payment method (UPI, Card, Wallet) implements a common PaymentStrategy interface — adding a new method means adding a class, not modifying existing code. Observer pattern via Spring's @EventListener for decoupling — when an order is placed, instead of calling notification, inventory, and analytics directly, I publish an OrderPlacedEvent and each listener handles its own logic."

### ⚡ Remember
- Cross-ref: [Design patterns deep dive → core/18-design-patterns-java.md](../languages/java/core/18-design-patterns-java.md)
- Cross-ref: [Strategy Pattern real example → spring/11 Q2](../languages/java/spring/11-springboot-scenario-interviews.md#q2)

---

<a id="q9"></a>
## Q9. Microservices vs Monolithic Architecture — which do you prefer?

### 📝 One-Liner
Start monolithic, migrate to microservices when team/scale demands it. Microservices solve organizational scaling, not technical scaling alone.

### 🔑 Quick Answer

| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| **Complexity** | Simple to build, deploy | Complex (networking, service discovery, distributed tracing) |
| **Deployment** | One deployable unit | Independent deployment per service |
| **Scaling** | Scale everything together | Scale individual services independently |
| **Team** | Works for <10 developers | Essential for 50+ developers (independent teams) |
| **Data** | Single database | Database per service (data isolation) |
| **Debugging** | Easy (single process) | Hard (distributed tracing needed) |

**My answer**: "I prefer starting with a well-structured monolith (modular monolith) and extracting microservices only when specific modules need independent scaling, different tech stacks, or separate team ownership."

### 🗣️ Answering Approach
"I don't have a blanket preference — it depends on the context. For a startup with 5 developers, a monolith is faster to build and debug. For a product with 50+ developers and services that need independent scaling, microservices are the right architecture. My approach: start with a modular monolith where each module has clear boundaries. When a module needs independent scaling, deployment, or is owned by a separate team, extract it into a microservice. The worst mistake is premature microservices — you get distributed system complexity without the organizational benefits."

### ⚡ Remember
- Cross-ref: [Microservices patterns → architecture/05-service-communication.md](../architecture/05-service-communication.md)
- Cross-ref: [System design monolith to microservices → system-design/01-distributed-systems-fundamentals.md](../system-design/01-distributed-systems-fundamentals.md)

---

<a id="q10"></a>
## Q10. What cloud services have you used? (AWS/Azure/GCP basics)

### 📝 One-Liner
Commonly asked to verify practical exposure — mention specific services you've used: EC2/ECS (compute), S3 (storage), RDS (database), Lambda (serverless), CloudWatch (monitoring), EKS (Kubernetes).

### 🔑 Quick Answer

| Need | AWS | Azure | GCP |
|------|-----|-------|-----|
| **Compute** | EC2, ECS, Lambda | VM, App Service, Functions | Compute Engine, Cloud Run |
| **Database** | RDS, DynamoDB | SQL DB, CosmosDB | Cloud SQL, Firestore |
| **Storage** | S3 | Blob Storage | Cloud Storage |
| **Messaging** | SQS, SNS, Kinesis | Service Bus | Pub/Sub |
| **Container** | EKS, ECS | AKS | GKE |
| **Monitoring** | CloudWatch | Monitor | Cloud Monitoring |
| **CI/CD** | CodePipeline | DevOps Pipelines | Cloud Build |

### 🗣️ Answering Approach
"I've primarily worked with AWS. For compute, I use ECS with Fargate for containerized Spring Boot services. For storage, S3 for file uploads with pre-signed URLs. RDS PostgreSQL for relational data. SQS for async message processing between services. CloudWatch for monitoring and alerts. In my recent project, I set up a CI/CD pipeline using CodePipeline: GitHub push triggers build, runs tests, deploys to ECS with blue-green deployment. I've also used Lambda for event-driven tasks like processing S3 upload notifications."

### ⚡ Remember
- Cross-ref: [Cloud infrastructure → cloud-devops/02-cloud-infra-processing.md](../cloud-devops/02-cloud-infra-processing.md)
- Cross-ref: [AWS architecture → system-design/04-product-company-designs.md](../system-design/04-product-company-designs.md)
