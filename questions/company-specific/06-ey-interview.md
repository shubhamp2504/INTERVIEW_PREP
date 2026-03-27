# 🏢 EY (Ernst & Young) — Java Spring Boot Developer Interview Experience (Round 2)

> Second round focused on practical knowledge: writing a full Spring Boot CRUD app on pen & paper, AWS serverless deployment, cloud infrastructure design, and DevOps practices.

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

## Section A: Spring Boot CRUD Application — Pen & Paper Walkthrough

---

<a id="q1"></a>
## Q1. Write a complete Spring Boot CRUD application on pen & paper — Controller, Service, Repository, Entity — and explain the full flow

### 📝 One-Liner
A Spring Boot CRUD app follows a layered architecture: `@Entity` → `@Repository` → `@Service` → `@RestController`, with Spring auto-wiring each layer through dependency injection.

### 🔑 Quick Answer
**Flow**: Client sends HTTP request → `@RestController` receives it → delegates to `@Service` for business logic → `@Service` calls `@Repository` (extends `JpaRepository`) → Repository talks to DB via JPA → `@Entity` maps Java object to table rows. Response flows back the same chain. *(Poora flow: request aata hai controller pe → service pe jaata hai → repo pe jaata hai → DB se data aata hai → wapas response banta hai)*

### 📖 How It Works
```
HTTP Request → DispatcherServlet → HandlerMapping
  → @RestController (maps URL to method)
    → @Service (business logic, validation, transformation)
      → @Repository (JpaRepository — CRUD operations auto-generated)
        → @Entity (JPA/Hibernate maps object ↔ table)
          → Database
  ← Response ← ResponseEntity ← DTO ←
```

### 🗣️ Answering Approach
"I'd start by defining the `@Entity` class — say an `Employee` with `@Id`, `@GeneratedValue`, and `@Column` annotations on each field. Then I create a `@Repository` interface extending `JpaRepository<Employee, Long>` — Spring Data auto-generates all CRUD methods. Next, a `@Service` class injects the repository via constructor injection and implements business logic — `save()`, `findById()`, `findAll()`, `deleteById()`. Finally, a `@RestController` with `@RequestMapping(\"/api/employees\")` exposes endpoints — `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping` — each delegating to the service. Spring Boot auto-configures DataSource from `application.properties`, and `@SpringBootApplication` on the main class triggers component scanning and auto-configuration."

### 💻 Code
```java
// ── Entity ──
@Entity
@Table(name = "employees")
public class Employee {
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    @Column(nullable = false, length = 100)
    private String name;

    @Column(nullable = false, unique = true)
    private String email;

    @Column(nullable = false)
    private String department;

    // constructors, getters, setters
}

// ── Repository ──
@Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByDepartment(String department);
    Optional<Employee> findByEmail(String email);
}

// ── Service ──
@Service
public class EmployeeService {

    private final EmployeeRepository repository;

    public EmployeeService(EmployeeRepository repository) {
        this.repository = repository;
    }

    public Employee create(Employee employee) {
        return repository.save(employee);
    }

    public List<Employee> getAll() {
        return repository.findAll();
    }

    public Employee getById(Long id) {
        return repository.findById(id)
            .orElseThrow(() -> new ResourceNotFoundException("Employee not found: " + id));
    }

    public Employee update(Long id, Employee updated) {
        Employee existing = getById(id);
        existing.setName(updated.getName());
        existing.setEmail(updated.getEmail());
        existing.setDepartment(updated.getDepartment());
        return repository.save(existing);
    }

    public void delete(Long id) {
        repository.deleteById(id);
    }
}

// ── Controller ──
@RestController
@RequestMapping("/api/employees")
public class EmployeeController {

    private final EmployeeService service;

    public EmployeeController(EmployeeService service) {
        this.service = service;
    }

    @PostMapping
    public ResponseEntity<Employee> create(@Valid @RequestBody Employee employee) {
        return ResponseEntity.status(HttpStatus.CREATED).body(service.create(employee));
    }

    @GetMapping
    public ResponseEntity<List<Employee>> getAll() {
        return ResponseEntity.ok(service.getAll());
    }

    @GetMapping("/{id}")
    public ResponseEntity<Employee> getById(@PathVariable Long id) {
        return ResponseEntity.ok(service.getById(id));
    }

    @PutMapping("/{id}")
    public ResponseEntity<Employee> update(@PathVariable Long id,
                                           @Valid @RequestBody Employee employee) {
        return ResponseEntity.ok(service.update(id, employee));
    }

    @DeleteMapping("/{id}")
    public ResponseEntity<Void> delete(@PathVariable Long id) {
        service.delete(id);
        return ResponseEntity.noContent().build();
    }
}

// ── application.properties ──
// spring.datasource.url=jdbc:mysql://localhost:3306/ey_db
// spring.datasource.username=root
// spring.datasource.password=secret
// spring.jpa.hibernate.ddl-auto=update
// spring.jpa.show-sql=true
```

### ⚠️ Pitfalls
| Mistake | Fix |
|---------|-----|
| Using `@Autowired` on fields | Use constructor injection — testable, immutable |
| Returning Entity directly | Use DTOs to avoid exposing DB structure |
| No `@Valid` on `@RequestBody` | Add Bean Validation to catch invalid data early |
| Forgetting `@Table` / `@Column` | Defaults work but explicit mapping prevents surprises |
| Using `ddl-auto=update` in production | Use Flyway/Liquibase for schema migration |

### 🆚 vs.
| Aspect | Field Injection | Constructor Injection |
|--------|----------------|----------------------|
| Testability | Hard to mock | Easy to mock |
| Immutability | Mutable | Final fields possible |
| Required deps | Can be null at runtime | Fails fast at startup |
| Spring recommendation | Discouraged | ✅ Preferred |

### 🎯 Tricky Follow-ups
- **Q**: "What happens if you don't add `@Repository`?" → Still works because `JpaRepository` is auto-detected — but `@Repository` adds exception translation (SQL exceptions → Spring's `DataAccessException`).
- **Q**: "Why not put logic in Controller?" → Violates SRP — controller handles HTTP, service handles business logic. Harder to test and reuse.
- **Q**: "How does Spring know which bean to inject?" → By type matching via `ApplicationContext`. If multiple beans of same type exist → use `@Qualifier` or `@Primary`.

### ⚡ Remember
> **Entity → Repository → Service → Controller** | Constructor injection | `@Valid` on request body | DTOs over entities | Flyway over ddl-auto

### 🔗 Follow-ups
- Spring Boot auto-configuration mechanism
- Exception handling with `@ControllerAdvice`
- DTO mapping with MapStruct
- Integration testing with `@SpringBootTest`

---

<a id="q2"></a>
## Q2. Explain all the annotations used in a Spring Boot CRUD application — from entity to configuration

### 📝 One-Liner
Spring Boot uses layered annotations: JPA annotations (`@Entity`, `@Id`, `@Column`) for persistence, stereotype annotations (`@Service`, `@Repository`, `@RestController`) for component roles, and web annotations (`@GetMapping`, `@RequestBody`, `@PathVariable`) for HTTP mapping.

### 🔑 Quick Answer
**Entity layer**: `@Entity`, `@Table`, `@Id`, `@GeneratedValue`, `@Column`, `@Enumerated`. **Repository**: `@Repository` (optional for JpaRepository). **Service**: `@Service`. **Controller**: `@RestController` (= `@Controller` + `@ResponseBody`), `@RequestMapping`, `@GetMapping`/`@PostMapping`/`@PutMapping`/`@DeleteMapping`, `@PathVariable`, `@RequestBody`, `@Valid`. **Config**: `@SpringBootApplication` (= `@Configuration` + `@EnableAutoConfiguration` + `@ComponentScan`), `@Bean`, `@ConfigurationProperties`. *(Har layer ka apna annotation hota hai — Spring isse samajhta hai ki ye component kya karta hai)*

### 📖 How It Works
```
@SpringBootApplication  ← Entry point (3-in-1)
├── @Configuration       → Declares @Bean methods
├── @EnableAutoConfiguration → Auto-configures based on classpath
└── @ComponentScan       → Scans packages for @Component derivatives

Layer Annotations:
  @Entity          → JPA: "this class maps to a DB table"
  @Repository      → Spring: "this is a DAO + exception translation"
  @Service         → Spring: "this holds business logic"
  @RestController  → Spring: "this handles HTTP + returns JSON directly"

HTTP Annotations:
  @RequestMapping("/api/x") → Base URL for all endpoints in controller
  @GetMapping("/{id}")      → Maps GET request
  @PostMapping              → Maps POST request
  @PathVariable             → Extracts value from URL path
  @RequestBody              → Deserializes JSON body → Java object
  @Valid                    → Triggers Bean Validation
```

### 🗣️ Answering Approach
"In a typical Spring Boot CRUD app, I use annotations at every layer. The `@Entity` with `@Table` maps the class to a database table — each field uses `@Column` for column mapping, `@Id` with `@GeneratedValue` for the primary key. The repository interface extends `JpaRepository` — the `@Repository` annotation is optional here but adds exception translation. The service class uses `@Service` — it's a specialization of `@Component` that signals business logic. The controller uses `@RestController` which combines `@Controller` and `@ResponseBody` — so every method return value is automatically serialized to JSON. HTTP methods are mapped with `@GetMapping`, `@PostMapping`, etc., under a base `@RequestMapping` path. Parameters come via `@PathVariable` for URL segments and `@RequestBody` for JSON payloads. At the top, `@SpringBootApplication` on the main class is a 3-in-1 meta-annotation: `@Configuration`, `@EnableAutoConfiguration`, and `@ComponentScan`."

### 💻 Code
```java
// All key annotations in one glance:
@SpringBootApplication                 // 3-in-1: Config + AutoConfig + Scan
public class Application {
    public static void main(String[] args) {
        SpringApplication.run(Application.class, args);
    }
}

@Entity                                // JPA — maps to table
@Table(name = "products")             // Explicit table name
public class Product {
    @Id                                // Primary key
    @GeneratedValue(strategy = IDENTITY) // Auto-increment
    private Long id;

    @Column(nullable = false)          // Column constraints
    private String name;

    @Enumerated(EnumType.STRING)       // Enum as string in DB
    private Category category;
}

@Repository                            // DAO + exception translation
public interface ProductRepository extends JpaRepository<Product, Long> {}

@Service                               // Business logic marker
public class ProductService { /* ... */ }

@RestController                        // @Controller + @ResponseBody
@RequestMapping("/api/products")       // Base path
public class ProductController {
    @GetMapping("/{id}")               // GET /api/products/{id}
    public Product get(@PathVariable Long id) { /* ... */ }

    @PostMapping                       // POST /api/products
    public Product create(@Valid @RequestBody Product p) { /* ... */ }
}

// Config annotations
@Configuration                         // Declares bean definitions
public class AppConfig {
    @Bean                              // Registers return value as bean
    public ModelMapper modelMapper() {
        return new ModelMapper();
    }
}

@ConfigurationProperties(prefix = "app") // Binds properties to POJO
public class AppProperties {
    private String name;
    private int maxRetries;
}
```

### 🎯 Tricky Follow-ups
- **Q**: "Difference between `@Controller` and `@RestController`?" → `@RestController` adds `@ResponseBody` to every method — returns JSON directly. `@Controller` returns view names (Thymeleaf/JSP).
- **Q**: "Can you use `@Component` instead of `@Service`?" → Yes, functionally identical. But `@Service` communicates intent — tells developers "this holds business logic."
- **Q**: "What does `@EnableAutoConfiguration` actually do?" → Reads `META-INF/spring.factories` (or `spring/org.springframework.boot.autoconfigure.AutoConfiguration.imports` in Boot 3) — conditionally registers beans based on classpath jars.

### ⚡ Remember
> **3-in-1**: `@SpringBootApplication` = Config + AutoConfig + Scan | **Stereotype**: Component → Service/Repository/Controller | **REST**: `@RestController` = Controller + ResponseBody

### 🔗 Follow-ups
- Custom annotations with `@AliasFor`
- Conditional annotations (`@ConditionalOnProperty`)
- Profile-specific configuration (`@Profile`)

---

## Section B: AWS Serverless Deployment & Infrastructure

---

<a id="q3"></a>
## Q3. How can a Spring Boot application be deployed in AWS using a serverless approach with Lambda?

### 📝 One-Liner
Use **AWS Lambda + API Gateway** with the `aws-serverless-java-container` library (or Spring Cloud Function) to run a Spring Boot app without managing servers — Lambda handles scaling, you pay only per invocation.

### 🔑 Quick Answer
**Approach**: Package Spring Boot app as a fat JAR or native image → use `aws-serverless-java-container-springboot3` adapter → deploy as Lambda function → front it with API Gateway. Alternatively, use **Spring Cloud Function** to define `Function<Input, Output>` beans and deploy with the AWS Lambda adapter. Cold starts are the main concern — mitigate with SnapStart, GraalVM native image, or provisioned concurrency. *(Spring Boot ko seedha Lambda pe deploy kar sakte ho — adapter use karo jo request convert karta hai, API Gateway se expose karo)*

### 📖 How It Works
```
Traditional:                   Serverless:
┌─────────────┐               ┌──────────────┐
│ EC2 / ECS   │               │ API Gateway  │
│ (always on) │               │  (HTTP proxy)│
│ Spring Boot │               │      ↓       │
│   Tomcat    │               │ AWS Lambda   │
│  (idle $$$) │               │ Spring Boot  │
└─────────────┘               │ (on-demand)  │
                              └──────────────┘

Lambda + Spring Boot Flow:
  1. Client → API Gateway (REST/HTTP API)
  2. API Gateway → Lambda (invokes handler)
  3. Lambda → StreamLambdaHandler (adapter)
  4. Adapter → converts API GW event → HttpServletRequest
  5. Spring Boot DispatcherServlet processes normally
  6. Response → adapter → API Gateway → Client

Cold Start Mitigation:
  SnapStart        → snapshots initialized JVM → ~10x faster
  GraalVM Native   → ahead-of-time compilation → <100ms starts
  Provisioned      → keeps N instances warm → $$ but no cold starts
```

### 🗣️ Answering Approach
"To deploy a Spring Boot app on AWS Lambda, I'd use the `aws-serverless-java-container` library by AWS Labs. I add the dependency, create a `StreamLambdaHandler` that initializes the Spring application context and delegates incoming API Gateway events to Spring's DispatcherServlet. The API Gateway acts as the HTTP frontend — it receives client requests and invokes the Lambda function. Inside Lambda, the adapter converts the API Gateway event to an `HttpServletRequest`, so all my `@RestController` endpoints work without modification. The main challenge is cold starts — Spring Boot initialization can take 5-10 seconds. To mitigate this, I'd use Lambda SnapStart which takes a snapshot of the initialized JVM, or compile to GraalVM native image for sub-100ms starts. For production, I'd keep the Lambda behind a VPC if it needs to access RDS or ElastiCache in private subnets."

### 💻 Code
```java
// ── StreamLambdaHandler.java ──
public class StreamLambdaHandler implements RequestStreamHandler {

    private static final SpringBootLambdaContainerHandler<AwsProxyRequest, AwsProxyResponse> handler;

    static {
        handler = SpringBootLambdaContainerHandler
            .getAwsProxyHandler(Application.class);
    }

    @Override
    public void handleRequest(InputStream input, OutputStream output, Context context)
            throws IOException {
        handler.proxyStream(input, output, context);
    }
}

// ── Alternative: Spring Cloud Function ──
@SpringBootApplication
public class Application {

    @Bean
    public Function<String, String> uppercase() {
        return value -> value.toUpperCase();
    }
}
// Deploy with: spring-cloud-function-adapter-aws
// Lambda handler: org.springframework.cloud.function.adapter.aws.FunctionInvoker

// ── SAM template (template.yaml) ──
// Resources:
//   SpringBootFunction:
//     Type: AWS::Serverless::Function
//     Properties:
//       Handler: com.example.StreamLambdaHandler
//       Runtime: java17
//       MemorySize: 512
//       Timeout: 30
//       SnapStart:
//         ApplyOn: PublishedVersions
//       Events:
//         Api:
//           Type: Api
//           Properties:
//             Path: /{proxy+}
//             Method: ANY
```

### ⚠️ Pitfalls
| Mistake | Fix |
|---------|-----|
| Ignoring cold starts | Use SnapStart / GraalVM native / provisioned concurrency |
| Large deployment package | Exclude unnecessary dependencies, use layers |
| Using embedded Tomcat features | Lambda adapter bypasses Tomcat — avoid server-specific code |
| Not setting memory correctly | More memory = more CPU = faster init. 512MB–1024MB sweet spot |
| VPC-attached Lambda | Adds 6-10s cold start without SnapStart — use VPC only if needed |

### 🆚 vs.
| Aspect | EC2/ECS (Traditional) | Lambda (Serverless) |
|--------|----------------------|---------------------|
| Scaling | Manual / ASG | Automatic, instant |
| Cost model | Pay for uptime | Pay per invocation |
| Cold starts | None | 5-10s (Java), mitigable |
| Max execution | Unlimited | 15 min |
| State | Stateful possible | Stateless |
| Best for | Long-running, high traffic | Bursty, event-driven |

### ⚡ Remember
> **Lambda + API Gateway** for serverless | `aws-serverless-java-container` adapter | SnapStart for cold starts | Spring Cloud Function as alternative | 512MB+ memory for Spring Boot

### 🔗 Follow-ups
- Lambda@Edge vs CloudFront Functions
- Step Functions for orchestrating Lambda
- DynamoDB vs RDS with Lambda

---

<a id="q4"></a>
## Q4. Design the AWS infrastructure for a typical application deployment — Client → Route53 → VPC → Subnets → Application

### 📝 One-Liner
A production AWS architecture routes traffic through **Route53 (DNS) → ALB (load balancer in public subnet) → Application (EC2/ECS/EKS in private subnet) → RDS (in isolated subnet)**, all within a VPC with public, private, and data-tier subnets across multiple AZs.

### 🔑 Quick Answer
**Full flow**: Client → Route53 (DNS resolution) → Internet Gateway → ALB (in public subnet, terminates SSL) → Target Group → App instances (in private subnet) → NAT Gateway (for outbound internet) → RDS/ElastiCache (in isolated/data subnet). Security: ALB allows 443 from `0.0.0.0/0`, app SG allows traffic only from ALB SG, DB SG allows only from app SG. Multi-AZ for HA. *(Client ka request Route53 se resolve hota hai → ALB pe aata hai public subnet mein → woh private subnet mein app ko forward karta hai → app private mein safe rehta hai)*

### 📖 How It Works
```
                        ┌─────────────────────────────────────┐
                        │              AWS Cloud              │
Client → Route53 (DNS)  │                                     │
         ↓              │  ┌───────────── VPC ─────────────┐  │
    Internet Gateway     │  │                               │  │
         ↓              │  │  Public Subnet (AZ-1 & AZ-2)  │  │
    ┌─────────┐         │  │  ┌─────┐     ┌─────────────┐  │  │
    │ Client  │────────────│  │ NAT │     │     ALB      │  │  │
    └─────────┘         │  │  │ GW  │     │ (port 443)   │  │  │
                        │  │  └──┬──┘     └──────┬───────┘  │  │
                        │  │     │               │          │  │
                        │  │  Private Subnet (AZ-1 & AZ-2) │  │
                        │  │     │        ┌──────┴───────┐  │  │
                        │  │     │        │  App (ECS/   │  │  │
                        │  │     ↓        │  EC2/EKS)    │  │  │
                        │  │  (outbound)  └──────┬───────┘  │  │
                        │  │                     │          │  │
                        │  │  Data Subnet (AZ-1 & AZ-2)    │  │
                        │  │              ┌──────┴───────┐  │  │
                        │  │              │   RDS Multi-  │  │  │
                        │  │              │   AZ / Redis  │  │  │
                        │  │              └──────────────┘  │  │
                        │  └───────────────────────────────┘  │
                        └─────────────────────────────────────┘
```

### 🗣️ Answering Approach
"I design the infrastructure within a VPC spanning at least 2 Availability Zones for high availability. The VPC has three subnet tiers: public, private, and data. In the public subnet, I place the Application Load Balancer and a NAT Gateway. The ALB receives HTTPS traffic — Route53 resolves the domain name to the ALB's DNS. The ALB terminates SSL using an ACM certificate and forwards traffic to target groups in the private subnet where the application runs — this could be ECS Fargate tasks, EC2 instances behind an ASG, or EKS pods. The application never has a public IP — it's only accessible through the ALB. For outbound internet access like calling external APIs, the app routes through the NAT Gateway in the public subnet. The data tier has RDS with Multi-AZ failover and ElastiCache — these are in isolated subnets with no internet access at all. Security Groups enforce least privilege: ALB accepts 443 from the internet, app accepts traffic only from the ALB's security group, and the database accepts connections only from the app's security group."

### 🎯 Tricky Follow-ups
- **Q**: "Why not put the app in the public subnet?" → Violates security best practice — direct internet exposure increases attack surface. Private subnet + ALB limits entry points.
- **Q**: "How does the app download packages if it's in a private subnet?" → NAT Gateway in the public subnet allows outbound internet. Or use VPC Endpoints for AWS services (S3, ECR, STS) to avoid NAT costs.
- **Q**: "What if Route53 is not used?" → You can point any DNS provider to the ALB's DNS name via CNAME. Route53 adds health checks, failover routing, latency-based routing.

### ⚡ Remember
> **3 subnet tiers**: Public (ALB, NAT) → Private (App) → Data (RDS) | **Multi-AZ** for HA | ALB terminates SSL | Security Groups chain: Internet→ALB→App→DB

### 🔗 Follow-ups
- VPC Peering vs Transit Gateway
- PrivateLink for service exposure
- CloudFront CDN in front of ALB

---

<a id="q5"></a>
## Q5. How does a client request reach an application deployed inside private subnets? What components expose the service securely?

### 📝 One-Liner
The client request enters via **Internet Gateway → ALB (public subnet) → target group routing → application (private subnet)** — the ALB is the only public-facing component, and Security Groups restrict all direct access.

### 🔑 Quick Answer
**Ingress path**: Client → DNS resolution (Route53) → Internet Gateway → ALB listener (HTTPS 443, public subnet) → ALB routes to target group → app instances in private subnet receive traffic on app port (e.g., 8080). The app has **no public IP** — only the ALB can reach it. **Key components**: Internet Gateway (VPC-level), ALB (public subnet), Target Group (health checks), Security Groups (firewall rules), NACLs (subnet-level). *(App private subnet mein hai — ALB public subnet mein hai — client ka request ALB se hoke app tak pahunchta hai, direct access nahi milta)*

### 📖 How It Works
```
Step-by-step packet journey:

1. Client types https://app.example.com
2. DNS (Route53) resolves → ALB public IP (e.g., 54.x.x.x)
3. Request hits Internet Gateway (VPC edge)
4. IGW routes to ALB in public subnet
5. ALB checks listener rules (port 443, SSL cert from ACM)
6. ALB selects healthy target from Target Group
7. ALB forwards to app instance PRIVATE IP (10.0.2.x:8080)
8. App processes request → response flows back same path

Security layers:
  Layer 1: Route53        → DNS-level (health checks, failover)
  Layer 2: WAF            → Web Application Firewall on ALB (SQL injection, XSS)
  Layer 3: Security Group → ALB SG: inbound 443 from 0.0.0.0/0
  Layer 4: Security Group → App SG: inbound 8080 ONLY from ALB SG
  Layer 5: NACL           → Subnet-level stateless firewall
  Layer 6: No public IP   → App instance unreachable from internet
```

### 🗣️ Answering Approach
"The key insight is that the application in a private subnet has no public IP — it's invisible to the internet. The only public-facing component is the Application Load Balancer sitting in the public subnet. When a client makes a request, Route53 resolves the domain to the ALB's public IP. The request enters the VPC through the Internet Gateway and reaches the ALB. The ALB has a listener on port 443 that terminates SSL using an ACM certificate. Based on routing rules — path-based or host-based — the ALB forwards the request to a target group. The target group contains the application instances registered by their private IPs. The ALB communicates with the app on port 8080 using the private IP — this traffic stays within the VPC, never leaving AWS's network. Security Groups enforce this: the ALB's SG allows inbound 443 from anywhere, but the app's SG only allows inbound traffic from the ALB's security group ID — not from any IP range. This means even if someone discovers the app's private IP, they cannot reach it without going through the ALB."

### 🆚 vs.
| Component | Public Subnet | Private Subnet |
|-----------|--------------|----------------|
| Internet access (inbound) | ✅ Yes (IGW) | ❌ No |
| Internet access (outbound) | ✅ Direct | Via NAT Gateway only |
| Public IP | ✅ Auto-assign possible | ❌ No public IP |
| Use case | ALB, NAT GW, Bastion | App servers, workers |
| Security risk | Higher (exposed) | Lower (isolated) |

### ⚡ Remember
> **ALB in public subnet is the gateway** | App in private subnet = no public IP | SG chaining: ALB SG → App SG | ACM for SSL | WAF for layer-7 protection

### 🔗 Follow-ups
- Bastion host vs SSM Session Manager for SSH
- AWS PrivateLink for inter-service communication
- API Gateway as alternative to ALB

---

## Section C: Cloud & DevOps Practices

---

<a id="q6"></a>
## Q6. What are Security Groups in AWS?

### 📝 One-Liner
Security Groups are **virtual firewalls** at the instance/ENI level that control inbound and outbound traffic using **allow-only rules** — they are stateful, meaning if inbound is allowed, the return traffic is automatically permitted.

### 🔑 Quick Answer
**Security Group** = stateful firewall attached to EC2, RDS, ALB, Lambda (in VPC), etc. Rules define allowed traffic by protocol, port, and source/destination (CIDR or another SG ID). **Key properties**: (1) Default deny all inbound, allow all outbound. (2) Stateful — return traffic auto-allowed. (3) Only allow rules, no deny. (4) Evaluated as a group — if any rule allows, traffic passes. (5) Can reference other SGs as source for chaining. *(Security Group ek virtual firewall hai — instance level pe lagta hai, stateful hai, sirf allow rules hote hain)*

### 📖 How It Works
```
Security Group Chaining (Best Practice):

ALB Security Group (sg-alb):
  Inbound:  TCP 443  from 0.0.0.0/0    ← Internet HTTPS
  Outbound: TCP 8080 to sg-app          ← Only to app SG

App Security Group (sg-app):
  Inbound:  TCP 8080 from sg-alb        ← Only from ALB
  Inbound:  TCP 22   from sg-bastion    ← SSH from bastion only
  Outbound: TCP 5432 to sg-db           ← Only to DB

DB Security Group (sg-db):
  Inbound:  TCP 5432 from sg-app        ← Only from app
  Outbound: (none needed — stateful return)

Stateful behavior:
  Request IN on port 8080 → allowed by inbound rule
  Response OUT on ephemeral port → AUTO-ALLOWED (stateful)
  No outbound rule needed for return traffic!
```

### 🗣️ Answering Approach
"Security Groups are stateful virtual firewalls that operate at the instance level — or more precisely, at the Elastic Network Interface level. Each Security Group has inbound and outbound rules. The key thing is they only have allow rules — there are no deny rules. By default, all inbound traffic is denied and all outbound is allowed. The stateful nature means if I allow inbound traffic on port 8080, the response traffic going back to the client is automatically allowed without an explicit outbound rule. A powerful feature is SG-to-SG referencing — instead of allowing traffic from an IP range, I reference another Security Group ID. For example, my app SG allows inbound on port 8080 only from the ALB's Security Group — this means only traffic originating from the ALB can reach my app, regardless of what IP the ALB has."

### 🆚 vs.
| Feature | Security Group | NACL |
|---------|---------------|------|
| Level | Instance / ENI | Subnet |
| Stateful | ✅ Yes | ❌ No (stateless) |
| Rules | Allow only | Allow + Deny |
| Evaluation | All rules evaluated | Rules evaluated in order |
| Default | Deny all inbound | Allow all |
| Return traffic | Auto-allowed | Must explicitly allow |

### ⚡ Remember
> **Stateful** = return traffic auto-allowed | **Allow-only** rules | Reference SG IDs for chaining | Instance-level firewall | Default: deny inbound, allow outbound

### 🔗 Follow-ups
- Security Group vs NACL decision matrix
- VPC Flow Logs for traffic analysis
- AWS Firewall Manager for multi-account SG management

---

<a id="q7"></a>
## Q7. What is the significance of IAM in AWS?

### 📝 One-Liner
**IAM (Identity and Access Management)** controls **who** (authentication) can do **what** (authorization) on **which** AWS resources — it's the foundation of AWS security, enforcing least-privilege access across users, roles, and services.

### 🔑 Quick Answer
IAM manages: (1) **Users** — individual human identities with credentials. (2) **Groups** — collections of users sharing permissions. (3) **Roles** — assumable identities for services (EC2, Lambda) or cross-account access. (4) **Policies** — JSON documents defining Allow/Deny on specific Actions for specific Resources. Everything in AWS requires IAM permissions — no IAM = no access (implicit deny). *(IAM se control karte ho ki kaun kya kar sakta hai AWS mein — users, roles, policies sab yahan define hote hain)*

### 📖 How It Works
```
IAM Structure:
  Root Account (don't use for daily ops!)
  ├── IAM Users (human identities)
  │   ├── Console password + MFA
  │   └── Access keys (programmatic)
  ├── IAM Groups (logical collection)
  │   ├── Developers (attach dev policies)
  │   └── Admins (attach admin policies)
  ├── IAM Roles (assumable identities)
  │   ├── EC2 Instance Role (app accesses S3)
  │   ├── Lambda Execution Role (accesses DynamoDB)
  │   └── Cross-Account Role (Team B accesses Team A's S3)
  └── IAM Policies (JSON permission docs)
      ├── AWS Managed (pre-built by AWS)
      ├── Customer Managed (your custom policies)
      └── Inline (embedded in user/group/role)

Policy evaluation: Explicit Deny > Explicit Allow > Implicit Deny

Example Policy:
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["s3:GetObject", "s3:PutObject"],
    "Resource": "arn:aws:s3:::my-bucket/*"
  }]
}
```

### 🗣️ Answering Approach
"IAM is the central security service in AWS — it controls authentication and authorization for every API call. Instead of giving root account access to developers, I create IAM Users grouped into IAM Groups with appropriate policies attached. For applications running on AWS services, I use IAM Roles — for example, an EC2 instance assumes a role that grants it read access to an S3 bucket. The app gets temporary credentials automatically, no hardcoded access keys. Policies are JSON documents that specify Effect (Allow/Deny), Action (like `s3:GetObject`), and Resource (ARN of the target). The evaluation logic is: explicit Deny always wins, then explicit Allow, and if nothing matches, it's implicit Deny. For production, I follow least privilege — start with zero permissions and add only what's needed. I also enable MFA for all human users and use AWS Organizations SCPs for account-level guardrails."

### 🎯 Tricky Follow-ups
- **Q**: "Role vs User — when to use which?" → Users for humans, Roles for services and cross-account access. Roles provide temporary credentials, users have long-lived credentials.
- **Q**: "What happens if a policy has both Allow and Deny for the same action?" → Explicit Deny always wins — this is the cardinal rule of IAM policy evaluation.
- **Q**: "How does an EC2 instance get credentials from a role?" → Instance metadata service (IMDS) at `169.254.169.254` — the SDK automatically fetches temporary credentials.

### ⚡ Remember
> **Users** for humans, **Roles** for services | Explicit Deny > Allow > Implicit Deny | Least privilege always | MFA for console users | Never hardcode access keys

### 🔗 Follow-ups
- IAM Identity Center (SSO) for multi-account
- Service Control Policies (SCPs) vs IAM policies
- Permission boundaries for delegated administration

---

<a id="q8"></a>
## Q8. What things do you usually configure in a Dockerfile? Docker, Kubernetes, and containers overview

### 📝 One-Liner
A Dockerfile defines the **build recipe** for a container image — base image, dependencies, app code, ports, and entrypoint. Docker provides containerization, Kubernetes orchestrates containers at scale with scheduling, scaling, and self-healing.

### 🔑 Quick Answer
**Dockerfile essentials**: (1) `FROM` — base image (JRE, Node, Alpine). (2) `WORKDIR` — set working directory. (3) `COPY`/`ADD` — bring in app code/JARs. (4) `RUN` — install dependencies, build. (5) `EXPOSE` — document port. (6) `ENV` — environment variables. (7) `USER` — run as non-root. (8) `ENTRYPOINT`/`CMD` — startup command. **Best practice**: multi-stage build — first stage compiles, second stage runs with minimal image. **K8s**: orchestrates containers with Pods, Deployments, Services, HPA, ConfigMaps. *(Dockerfile mein base image, copy JAR, expose port, aur entrypoint define karte ho — K8s usse scale karta hai)*

### 📖 How It Works
```
Dockerfile (Multi-Stage for Spring Boot):

Stage 1: BUILD
  FROM maven:3.9-eclipse-temurin-17 AS build
  COPY pom.xml .
  COPY src ./src
  RUN mvn clean package -DskipTests

Stage 2: RUN (minimal image)
  FROM eclipse-temurin:17-jre-alpine
  WORKDIR /app
  COPY --from=build target/*.jar app.jar
  EXPOSE 8080
  RUN addgroup -S appgroup && adduser -S appuser -G appgroup
  USER appuser
  ENTRYPOINT ["java", "-XX:MaxRAMPercentage=75.0", "-jar", "app.jar"]

Docker vs K8s:
  Docker     → Build & run single container
  Compose    → Multi-container on one host
  Kubernetes → Multi-container across cluster
    Pod         → Smallest unit (1+ containers)
    Deployment  → Manages replicas + rolling updates
    Service     → Stable IP + DNS for pods
    Ingress     → External HTTP routing
    HPA         → Auto-scale based on CPU/memory
    ConfigMap   → External configuration
    Secret      → Sensitive data (base64, not encrypted!)
```

### 🗣️ Answering Approach
"In a Dockerfile for a Spring Boot app, I use a multi-stage build. The first stage uses a Maven image to compile the project and produce the JAR. The second stage uses a minimal JRE Alpine image — `eclipse-temurin:17-jre-alpine` — and copies only the JAR from the build stage. I set `WORKDIR` to `/app`, `EXPOSE 8080` to document the port, create a non-root user with `RUN adduser` for security, switch to that user with `USER`, and define the `ENTRYPOINT` with JVM flags like `-XX:MaxRAMPercentage=75.0` for container-aware memory. This gives me a small, secure image — typically under 200MB. For orchestration, I deploy this to Kubernetes — define a Deployment with 3 replicas, a Service for internal DNS, an Ingress for external access, and HPA for auto-scaling based on CPU."

### 💻 Code
```dockerfile
# ── Production Dockerfile (Multi-Stage) ──
# Stage 1: Build
FROM maven:3.9-eclipse-temurin-17 AS build
WORKDIR /build
COPY pom.xml .
RUN mvn dependency:go-offline          # Cache dependencies
COPY src ./src
RUN mvn clean package -DskipTests

# Stage 2: Run
FROM eclipse-temurin:17-jre-alpine
WORKDIR /app

# Security: non-root user
RUN addgroup -S spring && adduser -S spring -G spring

# Copy JAR from build stage
COPY --from=build /build/target/*.jar app.jar

# JVM container-aware settings
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC"

EXPOSE 8080

# Health check
HEALTHCHECK --interval=30s --timeout=3s \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

USER spring

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

```yaml
# ── K8s Deployment ──
apiVersion: apps/v1
kind: Deployment
metadata:
  name: spring-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: spring-app
  template:
    spec:
      containers:
        - name: spring-app
          image: registry/spring-app:1.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits:   { cpu: "500m", memory: "1Gi" }
          readinessProbe:
            httpGet:
              path: /actuator/health
              port: 8080
```

### ⚡ Remember
> **Multi-stage** = small image | Non-root `USER` | `MaxRAMPercentage` for containers | K8s: Deployment + Service + Ingress + HPA | Cache dependencies layer separately

### 🔗 Follow-ups
- Docker layer caching optimization
- Kubernetes Secrets vs ConfigMaps
- Helm charts for templating K8s manifests

---

<a id="q9"></a>
## Q9. Have you written Jenkins pipelines and how are they used in CI/CD?

### 📝 One-Liner
A **Jenkins Pipeline** is a script (Declarative or Scripted) defining the entire CI/CD workflow as code — from checkout to build, test, scan, deploy — stored in a `Jenkinsfile` alongside the source code for versioning and reproducibility.

### 🔑 Quick Answer
**Declarative Pipeline** (preferred): uses `pipeline { agent, stages, steps, post }` syntax in a `Jenkinsfile`. Typical stages: Checkout → Build → Unit Test → Code Quality (SonarQube) → Docker Build → Push to Registry → Deploy (Dev → Staging → Prod). Triggered by webhooks (GitHub push), cron, or manual. **Key features**: parallel stages, environment variables, credentials binding, shared libraries, and post-build actions (notifications). *(Jenkins mein Jenkinsfile likhte ho — pipeline as code — stages define karte ho build, test, deploy ke liye)*

### 📖 How It Works
```
Jenkinsfile (Pipeline as Code):

┌────────────────────────────────────────────┐
│ pipeline {                                  │
│   agent any                                │
│   stages {                                 │
│     ┌─────────┐  ┌──────┐  ┌───────────┐  │
│     │Checkout  │→ │Build │→ │Unit Tests │  │
│     │(git)     │  │(mvn) │  │(JUnit)    │  │
│     └─────────┘  └──────┘  └───────────┘  │
│          ↓                                 │
│     ┌───────────┐  ┌────────┐  ┌────────┐ │
│     │SonarQube  │→ │Docker  │→ │Push to │ │
│     │(quality)  │  │Build   │  │ECR/Hub │ │
│     └───────────┘  └────────┘  └────────┘ │
│          ↓                                 │
│     ┌──────────┐  ┌──────────┐  ┌───────┐ │
│     │Deploy    │→ │Deploy    │→ │Deploy │ │
│     │(Dev)     │  │(Staging) │  │(Prod) │ │
│     └──────────┘  └──────────┘  └───────┘ │
│   }                                        │
│   post { success/failure → Slack notify }  │
│ }                                          │
└────────────────────────────────────────────┘
```

### 🗣️ Answering Approach
"Yes, I've written declarative Jenkins pipelines using a `Jenkinsfile` stored in the project repository — this gives us pipeline as code, fully version-controlled. A typical pipeline starts with a `Checkout` stage that pulls the latest code from Git. Then a `Build` stage runs `mvn clean package` to compile and produce the JAR. Next, `Unit Test` runs with JUnit — if tests fail, the pipeline stops. Then I have a `SonarQube` stage for code quality and security scanning. If quality gates pass, I build a Docker image and push it to Amazon ECR. Deployment stages go through Dev first — automated — then Staging with integration tests, and finally Prod with a manual approval gate using `input` step. The `post` block sends Slack notifications on success or failure. I use `credentials()` to bind secrets, `environment` block for stage-specific variables, and shared libraries for reusable pipeline code across projects."

### 💻 Code
```groovy
// ── Jenkinsfile (Declarative Pipeline) ──
pipeline {
    agent any

    environment {
        DOCKER_REGISTRY = 'xxxx.dkr.ecr.ap-south-1.amazonaws.com'
        APP_NAME        = 'spring-boot-app'
        VERSION         = "${BUILD_NUMBER}"
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/org/repo.git'
            }
        }

        stage('Build') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Unit Tests') {
            steps {
                sh 'mvn test'
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn sonar:sonar'
                }
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {
                    docker.build("${APP_NAME}:${VERSION}")
                    docker.withRegistry("https://${DOCKER_REGISTRY}",
                                        'ecr:ap-south-1:aws-creds') {
                        docker.image("${APP_NAME}:${VERSION}").push()
                        docker.image("${APP_NAME}:${VERSION}").push('latest')
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                sh "kubectl set image deployment/${APP_NAME} " +
                   "${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${VERSION} " +
                   "--namespace=dev"
            }
        }

        stage('Deploy to Prod') {
            steps {
                input message: 'Deploy to Production?', ok: 'Approve'
                sh "kubectl set image deployment/${APP_NAME} " +
                   "${APP_NAME}=${DOCKER_REGISTRY}/${APP_NAME}:${VERSION} " +
                   "--namespace=prod"
            }
        }
    }

    post {
        success { slackSend color: 'good', message: "✅ ${APP_NAME} v${VERSION} deployed" }
        failure { slackSend color: 'danger', message: "❌ ${APP_NAME} build failed" }
    }
}
```

### 🆚 vs.
| Feature | Jenkins | GitHub Actions |
|---------|---------|----------------|
| Config file | Jenkinsfile (Groovy) | .github/workflows/*.yml |
| Hosting | Self-hosted (or CloudBees) | GitHub-hosted |
| Plugins | 1800+ plugins | Marketplace actions |
| Pipeline syntax | Declarative / Scripted | YAML |
| Best for | Enterprise, complex flows | GitHub-native projects |

### ⚡ Remember
> **Declarative** over Scripted | `Jenkinsfile` in repo = pipeline as code | `input` for manual approval gates | `post` for notifications | Shared libraries for reuse across teams

### 🔗 Follow-ups
- Jenkins shared libraries architecture
- Blue-green vs canary deployment strategies
- GitOps with ArgoCD as Jenkins alternative
