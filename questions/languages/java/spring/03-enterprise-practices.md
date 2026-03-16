# 🌿 Spring — API Security, Code Quality & CI/CD (Q7, Q10, Q11, Q14)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q7"></a>
## Q7. How do you implement secure and scalable REST APIs in Java-based systems?

### 📝 One-Liner
Secure with Spring Security (JWT auth + RBAC + CORS + rate limiting + input validation) + scalable with stateless design + pagination + caching + async processing.

### 🔑 Quick Answer
**Security layers**: **(1)** Authentication — JWT tokens (stateless, no session). **(2)** Authorization — role-based (`@PreAuthorize("hasRole('ADMIN')")`) or method-level security. **(3)** Input validation — `@Valid` + Bean Validation annotations to prevent injection. **(4)** Rate limiting — per-client/IP (Redis counter or Bucket4j). **(5)** CORS — whitelist allowed origins. **(6)** HTTPS — TLS everywhere in production. **(7)** Security headers — CSP, X-Frame-Options, X-Content-Type-Options. **Scalability layers**: **(1)** Stateless services (no server-side session). **(2)** Pagination (`Pageable`) + keyset pagination for deep pages. **(3)** Caching (Redis) for read-heavy data. **(4)** Async processing (Kafka) for non-blocking operations. **(5)** API versioning (URI-based `/v1/` or header-based). *(Secure = JWT + validation + rate limit; Scalable = stateless + pagination + cache)*

### 📖 How It Works
```
Secure + Scalable API Architecture:

  Client Request
       │
       ↓
  ┌──────────────────────────────────┐
  │ 1. HTTPS (TLS termination)      │ ← encrypt in transit
  ├──────────────────────────────────┤
  │ 2. Rate Limiter (Bucket4j/Redis)│ ← prevent abuse
  ├──────────────────────────────────┤
  │ 3. JWT Auth Filter              │ ← authenticate identity
  ├──────────────────────────────────┤
  │ 4. CORS Filter                  │ ← allowed origins only
  ├──────────────────────────────────┤
  │ 5. Input Validation (@Valid)    │ ← prevent injection
  ├──────────────────────────────────┤
  │ 6. Authorization (@PreAuthorize)│ ← RBAC / method security
  ├──────────────────────────────────┤
  │ 7. Controller → Service → DB   │ ← business logic
  ├──────────────────────────────────┤
  │ 8. Response: paginated + HATEOAS│ ← scalable response
  └──────────────────────────────────┘

Scalability Checklist:
  ✅ Stateless (JWT, no server session) → horizontal scaling
  ✅ Paginated responses (Page + Keyset) → bounded data
  ✅ Redis caching → 90%+ read traffic absorbed
  ✅ Async for heavy ops → Kafka event publication
  ✅ Connection pooling (HikariCP) → efficient DB usage
  ✅ Response compression (gzip) → reduce bandwidth
  ✅ API versioning → backward compatibility
```

### 🗣️ How to Say in Interview
"I implement API security in layers. HTTPS is non-negotiable for all environments. Authentication uses JWT tokens with Spring Security — a filter validates the token on every request without server-side sessions, keeping the service stateless for horizontal scaling. Authorization uses @PreAuthorize with role-based access control. All input is validated with Bean Validation annotations — @NotBlank, @Email, @Pattern — which prevents injection at the entry point. I add rate limiting per client using Bucket4j backed by Redis to prevent abuse. For scalability, the APIs are stateless and paginated — I use Spring's Pageable with keyset pagination for deep pages. Read-heavy endpoints have Redis caching. Non-blocking operations like sending emails go through Kafka. I also use API versioning via URI path so we can evolve the API without breaking existing clients."

### 💻 Code
```java
// SECURITY CONFIG — Spring Security 6
@Configuration
@EnableWebSecurity
@EnableMethodSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtAuthFilter jwtFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())              // stateless = no CSRF
            .sessionManagement(sm -> sm.sessionCreationPolicy(STATELESS))
            .cors(cors -> cors.configurationSource(corsConfig()))
            .headers(headers -> headers
                .contentSecurityPolicy(csp -> csp.policyDirectives("default-src 'self'"))
                .frameOptions(frame -> frame.deny()))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/auth/**", "/api/public/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
    
    @Bean
    CorsConfigurationSource corsConfig() {
        CorsConfiguration config = new CorsConfiguration();
        config.setAllowedOrigins(List.of("https://myapp.com", "https://admin.myapp.com"));
        config.setAllowedMethods(List.of("GET", "POST", "PUT", "DELETE"));
        config.setAllowedHeaders(List.of("Authorization", "Content-Type"));
        UrlBasedCorsConfigurationSource source = new UrlBasedCorsConfigurationSource();
        source.registerCorsConfiguration("/api/**", config);
        return source;
    }
}

// RATE LIMITING — per client with Bucket4j + Redis
@Component
public class RateLimitFilter extends OncePerRequestFilter {
    private final Map<String, Bucket> buckets = new ConcurrentHashMap<>();
    
    @Override
    protected void doFilterInternal(HttpServletRequest req,
            HttpServletResponse res, FilterChain chain) throws Exception {
        String clientId = req.getHeader("X-API-Key");
        if (clientId == null) clientId = req.getRemoteAddr();
        
        Bucket bucket = buckets.computeIfAbsent(clientId, k ->
            Bucket.builder()
                .addLimit(Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1))))
                .build());
        
        if (bucket.tryConsume(1)) {
            chain.doFilter(req, res);
        } else {
            res.setStatus(HttpStatus.TOO_MANY_REQUESTS.value());
            res.getWriter().write("{\"error\":\"Rate limit exceeded. Try again later.\"}");
        }
    }
}

// INPUT VALIDATION — prevent injection + bad data
public record CreateOrderRequest(
    @NotNull Long productId,
    @Positive @Max(1000) int quantity,
    @NotBlank @Size(max = 200) String shippingAddress,
    @Email String contactEmail
) {}

@RestController
@RequestMapping("/api/v1/orders")
@RequiredArgsConstructor
public class OrderController {
    
    @PostMapping
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")  // role-based auth
    public ResponseEntity<OrderDTO> createOrder(
            @RequestBody @Valid CreateOrderRequest request) {  // @Valid triggers validation
        OrderDTO order = orderService.create(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
    
    @GetMapping
    public ResponseEntity<PageResponse<OrderDTO>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") @Max(100) int size) {
        return ResponseEntity.ok(orderService.getOrders(PageRequest.of(page, size)));
    }
}

// API VERSIONING — URI-based (simplest)
@RestController
@RequestMapping("/api/v1/products")
public class ProductControllerV1 { /* original */ }

@RestController
@RequestMapping("/api/v2/products")
public class ProductControllerV2 { /* enhanced with new fields */ }
```

### ⚠️ Pitfalls / Gotchas
- **CORS `*` wildcard** in production = security hole. Always whitelist specific origins *(CORS mein * mat rakho — specific domains whitelist karo)*
- **Rate limiting in memory** doesn't work with multiple instances. Use Redis-backed rate limiter
- **@Valid missing** on controller parameter → validation annotations are completely ignored
- **Logging request bodies** containing passwords or tokens → sensitive data leaked to logs
- **HTTPS disabled** "for convenience" in staging → developers test without TLS → bugs in production

### ⚡ Remember
- **Security layers**: HTTPS → Rate Limit → JWT Auth → CORS → Validation → Authorization
- **Stateless** (JWT, no session) = scales horizontally *(Stateless = koi bhi server handle kare)*
- **@Valid** on every request body — first defense against injection
- **Rate limit** per client in Redis — prevent abuse
- **Paginate** all list endpoints — never return unbounded data
- API versioning from day one (`/v1/`)

### 🔗 Follow-ups
- [Q14 → Java security (SQL injection, deserialization)](#q14)
- [Q10 → Code quality practices](#q10)
- Q16 → JWT authentication flow (architecture/01)

---

<a id="q10"></a>
## Q10. What strategies do you use to ensure code quality and maintainability in large enterprise Java projects?

### 📝 One-Liner
Code reviews + automated testing (unit/integration/contract) + static analysis (SonarQube) + consistent coding standards (Checkstyle/Spotless) + SOLID principles + architecture fitness functions.

### 🔑 Quick Answer
**(1) Automated Testing**: unit tests (JUnit 5 + Mockito, 80%+ coverage), integration tests (@SpringBootTest + Testcontainers for real DB), contract tests (Spring Cloud Contract for API consumers). **(2) Static Analysis**: SonarQube (bugs, vulnerabilities, code smells, duplication), SpotBugs (bytecode analysis), Checkstyle/Spotless (formatting enforcement). **(3) Code Reviews**: PR-based workflow, minimum 2 approvers, automated checks must pass before merge. **(4) Design Principles**: SOLID, DRY, clean architecture (controller → service → repository layers), dependency inversion. **(5) Documentation**: API docs (SpringDoc/OpenAPI), ADRs (Architecture Decision Records) for major decisions. **(6) CI enforcement**: all checks run in pipeline — failing quality gate blocks merge. *(Test likho, SonarQube lagao, code review karo, CI mein sab enforce karo)*

### 📖 How It Works
```
Code Quality Pipeline:

  Developer writes code
       │
       ↓
  ┌──────────────────────┐
  │ Local: Pre-commit     │
  │ • Spotless format     │ ← auto-fix formatting
  │ • Unit tests          │ ← fast feedback
  └──────────┬───────────┘
             ↓
  ┌──────────────────────┐
  │ PR Created            │
  │ • CI runs all tests   │
  │ • SonarQube scan      │ ← bugs, smells, coverage
  │ • Security scan       │ ← dependency vulnerabilities
  │ • Contract tests      │ ← API compatibility
  └──────────┬───────────┘
             ↓
  ┌──────────────────────┐
  │ Quality Gate Check    │
  │ • Coverage ≥ 80% ?   │
  │ • 0 blockers/critical?│
  │ • 0 security vulns?  │
  │ → PASS or BLOCK merge│
  └──────────┬───────────┘
             ↓
  ┌──────────────────────┐
  │ Code Review           │
  │ • 2+ approvals needed │
  │ • Focus: design, edge │
  │   cases, naming, DRY  │
  └──────────┬───────────┘
             ↓
  Merge to main ✅

Testing Pyramid:
          ┌──────────┐
          │ E2E      │  ← few, slow, fragile
          │ (3-5%)   │
         ┌┴──────────┴┐
         │Integration  │  ← moderate count, real deps
         │ (15-20%)    │     (Testcontainers + DB)
        ┌┴────────────┴┐
        │   Unit Tests   │  ← many, fast, isolated
        │   (75-80%)     │     (JUnit 5 + Mockito)
        └────────────────┘
```

### 🗣️ How to Say in Interview
"I ensure code quality through automation at every stage. Developers run Spotless formatter and unit tests locally before committing. The CI pipeline runs the full test suite — unit, integration with Testcontainers for real database testing, and contract tests for API compatibility. SonarQube analyzes every PR for bugs, code smells, and security vulnerabilities with a quality gate: minimum 80% coverage, zero critical issues, and zero security vulnerabilities must pass before merge is allowed. Code reviews require two approvals and focus on design decisions, edge cases, and naming — not formatting, since that's automated. For design, we follow SOLID principles and clean architecture with clear separation between controller, service, and repository layers. Architecture Decision Records document major choices for future team members."

### 💻 Code
```java
// UNIT TEST — JUnit 5 + Mockito (fast, isolated)
@ExtendWith(MockitoExtension.class)
class OrderServiceTest {
    @Mock private OrderRepository orderRepo;
    @Mock private ProductClient productClient;
    @InjectMocks private OrderService orderService;
    
    @Test
    void createOrder_validRequest_returnsCreatedOrder() {
        // Given
        when(productClient.getProduct(1L)).thenReturn(new ProductDTO(1L, "Widget", new BigDecimal("9.99")));
        when(orderRepo.save(any())).thenAnswer(inv -> {
            Order o = inv.getArgument(0);
            o.setId(100L);
            return o;
        });
        
        // When
        OrderDTO result = orderService.createOrder(new CreateOrderRequest(1L, 5, "addr", "e@m.com"));
        
        // Then
        assertThat(result.id()).isEqualTo(100L);
        assertThat(result.total()).isEqualByComparingTo("49.95");
        verify(orderRepo).save(any(Order.class));
    }
    
    @Test
    void createOrder_productNotFound_throwsException() {
        when(productClient.getProduct(999L)).thenThrow(new ProductNotFoundException(999L));
        assertThatThrownBy(() -> orderService.createOrder(
                new CreateOrderRequest(999L, 1, "addr", "e@m.com")))
            .isInstanceOf(ProductNotFoundException.class);
    }
}

// INTEGRATION TEST — @SpringBootTest + Testcontainers (real DB)
@SpringBootTest
@Testcontainers
@AutoConfigureMockMvc
class OrderControllerIntegrationTest {
    @Container
    static PostgreSQLContainer<?> postgres = new PostgreSQLContainer<>("postgres:16-alpine");
    
    @DynamicPropertySource
    static void dbProperties(DynamicPropertyRegistry registry) {
        registry.add("spring.datasource.url", postgres::getJdbcUrl);
        registry.add("spring.datasource.username", postgres::getUsername);
        registry.add("spring.datasource.password", postgres::getPassword);
    }
    
    @Autowired private MockMvc mockMvc;
    
    @Test
    void createOrder_returnsCreated() throws Exception {
        mockMvc.perform(post("/api/v1/orders")
                .header("Authorization", "Bearer " + validJwt())
                .contentType(MediaType.APPLICATION_JSON)
                .content("""
                    {"productId": 1, "quantity": 2,
                     "shippingAddress": "123 Main St", "contactEmail": "u@m.com"}
                    """))
            .andExpect(status().isCreated())
            .andExpect(jsonPath("$.id").isNumber());
    }
}

// ARCHITECTURE FITNESS FUNCTION — ArchUnit
@AnalyzeClasses(packages = "com.app")
class ArchitectureTest {
    @ArchTest
    static final ArchRule controllers_should_not_access_repositories =
        noClasses().that().resideInAPackage("..controller..")
            .should().accessClassesThat().resideInAPackage("..repository..");
    
    @ArchTest
    static final ArchRule services_should_not_depend_on_controllers =
        noClasses().that().resideInAPackage("..service..")
            .should().dependOnClassesThat().resideInAPackage("..controller..");
}

// build.gradle — quality plugins
// plugins {
//     id 'com.diffplug.spotless' version '6.25'
//     id 'org.sonarqube' version '5.0'
//     id 'jacoco'
// }
// spotless { java { googleJavaFormat() } }
// jacocoTestReport { reports { xml.required = true } }
// sonar { properties { property "sonar.coverage.jacoco.xmlReportPaths", "build/reports/jacoco/test/*.xml" } }
```

### ⚠️ Pitfalls / Gotchas
- **High coverage ≠ good tests** — 100% coverage with no assertions is useless. Focus on behavior, not lines *(Coverage 90% hai lekin assert nahi lagaya toh kya fayda)*
- **Integration tests without Testcontainers** → H2 in-memory DB behaves differently from production PostgreSQL
- **SonarQube warnings ignored** → tech debt accumulates, becomes unmanageable
- **Code reviews that only check syntax** — automated checkers handle that. Reviews should focus on design
- **Missing contract tests** → producers break consumers unknowingly

### ⚡ Remember
- **Testing pyramid**: many unit tests (80%), moderate integration (15%), few E2E (5%)
- **SonarQube quality gate**: coverage ≥ 80%, 0 critical, 0 security vuln → blocks merge
- **Spotless/Checkstyle** = auto-format → no style debates in PRs
- **Testcontainers** = real DB in tests → catch DB-specific bugs
- **ArchUnit** = enforce architecture rules as tests *(Architecture rules test mein enforce karo)*
- Code review = design + edge cases, NOT formatting

### 🔗 Follow-ups
- [Q11 → CI/CD pipelines (where quality gates run)](#q11)
- [Q7 → Secure REST APIs (security testing)](#q7)

---

<a id="q11"></a>
## Q11. How do you implement CI/CD pipelines and automated testing for Java applications?

### 📝 One-Liner
CI: GitHub Actions/Jenkins builds on every push → compile + test + SonarQube + security scan + Docker image. CD: auto-deploy to staging, manual approval for production, blue-green/canary deployments with rollback.

### 🔑 Quick Answer
**CI (Continuous Integration)**: every push triggers: **(1)** compile + unit tests, **(2)** integration tests (Testcontainers), **(3)** SonarQube analysis + quality gate, **(4)** dependency security scan (OWASP Dependency-Check), **(5)** build Docker image, **(6)** push to container registry. **CD (Continuous Delivery/Deployment)**: **(1)** auto-deploy to staging/QA, **(2)** run smoke tests + E2E tests against staging, **(3)** manual approval gate (for production), **(4)** deploy to production using blue-green or canary strategy, **(5)** health check validation, **(6)** auto-rollback if health checks fail. **Tools**: GitHub Actions/Jenkins for pipeline, Maven/Gradle for build, JUnit 5 for tests, Docker for packaging, Kubernetes for deployment, ArgoCD for GitOps. *(CI = har push pe test + build, CD = staging automatic, production approval ke baad)*

### 📖 How It Works
```
CI/CD Pipeline Flow:

  git push → trigger pipeline
       │
  ┌────┴─────────────────────────────────────────┐
  │ STAGE 1: Build + Unit Test                    │
  │ • mvn compile                                 │
  │ • mvn test (JUnit 5 + Mockito)               │
  │ • 80%+ coverage check (JaCoCo)               │
  │ → ~2 min                                      │
  ├──────────────────────────────────────────────┤
  │ STAGE 2: Integration Test                     │
  │ • Testcontainers (PostgreSQL, Redis, Kafka)  │
  │ • @SpringBootTest with real dependencies     │
  │ → ~5 min                                      │
  ├──────────────────────────────────────────────┤
  │ STAGE 3: Quality Analysis                     │
  │ • SonarQube scan (bugs, smells, coverage)    │
  │ • Quality gate: PASS or FAIL pipeline        │
  │ • OWASP Dependency-Check (CVE scan)          │
  │ → ~3 min                                      │
  ├──────────────────────────────────────────────┤
  │ STAGE 4: Build + Push Image                   │
  │ • docker build (multi-stage)                 │
  │ • docker push to container registry (ECR/GCR)│
  │ • Tag: commit SHA + branch + latest          │
  │ → ~2 min                                      │
  ├──────────────────────────────────────────────┤
  │ STAGE 5: Deploy to Staging                    │
  │ • kubectl apply / helm upgrade               │
  │ • Run smoke tests against staging            │
  │ • E2E tests (API + UI)                       │
  │ → ~5 min                                      │
  ├──────────────────────────────────────────────┤
  │ STAGE 6: Deploy to Production                 │
  │ • Manual approval gate ✋                     │
  │ • Blue-green or canary deployment            │
  │ • Health check validation                    │
  │ • Auto-rollback on failure                   │
  └──────────────────────────────────────────────┘
  Total: ~15-20 min push-to-production

Deployment Strategies:
  Blue-Green: two environments, switch traffic instantly
    [Blue v1.0 ← traffic] [Green v1.1 idle]
    → switch: [Blue v1.0 idle] [Green v1.1 ← traffic]
    → rollback: switch back instantly

  Canary: gradual traffic shift
    v1.0 ← 90% traffic    v1.1 ← 10% traffic
    → monitor metrics → if OK → v1.1 ← 50% → v1.1 ← 100%
    → if metrics bad → rollback to v1.0 ← 100%
```

### 🗣️ How to Say in Interview
"Our CI pipeline triggers on every push. First, it compiles and runs unit tests — about 500 JUnit 5 tests in 2 minutes. Then integration tests run with Testcontainers against a real PostgreSQL and Redis. SonarQube scans for code quality with a quality gate: minimum 80% coverage, zero critical bugs, and zero security vulnerabilities — failing the gate blocks the merge. We also run OWASP Dependency-Check to catch known CVEs in libraries. The pipeline then builds a Docker image with multi-stage build and pushes to our container registry. For CD, staging deployment is automatic with smoke tests running against it. Production requires manual approval, then we deploy using canary strategy — sending 10% traffic to the new version first, monitoring error rates and latency for 15 minutes. If metrics stay healthy, we roll out to 100%. If anything degrades, Kubernetes auto-rolls back. The entire push-to-production cycle takes about 20 minutes."

### 💻 Code
```yaml
# .github/workflows/ci-cd.yml — GitHub Actions pipeline
name: CI/CD Pipeline
on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up JDK 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          cache: 'maven'
      
      - name: Build and Unit Test
        run: mvn verify -Dspring.profiles.active=test
      
      - name: Integration Tests (Testcontainers)
        run: mvn verify -P integration-test
      
      - name: SonarQube Analysis
        env:
          SONAR_TOKEN: ${{ secrets.SONAR_TOKEN }}
        run: mvn sonar:sonar -Dsonar.host.url=https://sonarqube.mycompany.com
      
      - name: OWASP Dependency Check
        run: mvn org.owasp:dependency-check-maven:check
      
      - name: Build Docker Image
        run: |
          docker build -t myregistry.com/order-service:${{ github.sha }} .
          docker tag myregistry.com/order-service:${{ github.sha }} myregistry.com/order-service:latest
      
      - name: Push to Registry
        run: |
          echo "${{ secrets.REGISTRY_PASSWORD }}" | docker login myregistry.com -u ${{ secrets.REGISTRY_USER }} --password-stdin
          docker push myregistry.com/order-service:${{ github.sha }}
          docker push myregistry.com/order-service:latest

  deploy-staging:
    needs: build-and-test
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    steps:
      - name: Deploy to Staging
        run: |
          kubectl set image deployment/order-service \
            order-service=myregistry.com/order-service:${{ github.sha }} \
            --namespace=staging
          kubectl rollout status deployment/order-service --namespace=staging --timeout=300s
      
      - name: Smoke Tests
        run: |
          sleep 30
          curl -f https://staging.myapp.com/actuator/health || exit 1
          mvn verify -P smoke-test -Dbase.url=https://staging.myapp.com

  deploy-production:
    needs: deploy-staging
    if: github.ref == 'refs/heads/main'
    environment: production   # requires manual approval in GitHub
    runs-on: ubuntu-latest
    steps:
      - name: Canary Deploy (10%)
        run: |
          kubectl set image deployment/order-service-canary \
            order-service=myregistry.com/order-service:${{ github.sha }} \
            --namespace=production
      
      - name: Monitor Canary (5 min)
        run: |
          sleep 300
          # Check error rate from Prometheus
          ERROR_RATE=$(curl -s 'http://prometheus:9090/api/v1/query?query=rate(http_server_requests_seconds_count{status=~"5.."}[5m])' | jq '.data.result[0].value[1]')
          if (( $(echo "$ERROR_RATE > 0.01" | bc -l) )); then
            echo "Error rate too high, rolling back"
            kubectl rollout undo deployment/order-service-canary --namespace=production
            exit 1
          fi
      
      - name: Full Rollout
        run: |
          kubectl set image deployment/order-service \
            order-service=myregistry.com/order-service:${{ github.sha }} \
            --namespace=production
          kubectl rollout status deployment/order-service --namespace=production --timeout=600s
```

### ⚠️ Pitfalls / Gotchas
- **No rollback plan** → deployment breaks production with no recovery. Always test rollback before it's needed *(Rollback plan nahi hai toh production mein problem hoga — pehle se test karo)*
- **Slow pipeline** (>30 min) → developers skip it or batch changes. Parallelize stages
- **Flaky tests** → pipeline fails randomly → teams start ignoring failures
- **Secrets in code/logs** → use GitHub Secrets/Vault, never echo credentials
- **Deploy on Friday** → if canary catches issues, team may not be available to respond

### ⚡ Remember
- **CI**: compile → unit test → integration test → SonarQube → Docker build → push
- **CD**: staging (auto) → smoke test → production (manual approval) → canary → full rollout
- **Quality gate** blocks merge: coverage ≥ 80%, 0 critical, 0 CVEs *(Quality gate fail toh merge nahi hoga)*
- **Canary** = gradual rollout + monitoring → auto-rollback on failure
- Push-to-production: ~15-20 minutes total
- Testcontainers for real DB in CI/CD

### 🔗 Follow-ups
- [Q10 → Code quality (tests, SonarQube)](#q10)
- Q19 → Dockerize Spring Boot (architecture/01)

---

<a id="q14"></a>
## Q14. What techniques do you use to secure Java applications against SQL injection, deserialization attacks, and authentication risks?

### 📝 One-Liner
SQL injection: parameterized queries (never concatenate). Deserialization: avoid Java serialization (use JSON), whitelist classes if needed. Auth: bcrypt passwords, JWT with short expiry, MFA, rate-limit login.

### 🔑 Quick Answer
**SQL Injection**: **(1)** Always use **parameterized queries** / prepared statements — NEVER concatenate user input into SQL. JPA/Hibernate does this by default. **(2)** For native queries, use `@Param` named parameters. **(3)** Input validation as first defense. **Deserialization attacks**: **(1)** Avoid Java's `ObjectInputStream` entirely — use JSON (Jackson). **(2)** If you must deserialize, use `ObjectInputFilter` (Java 9+) to whitelist allowed classes. **(3)** Keep libraries updated (Jackson, XStream vulnerabilities are common). **Authentication**: **(1)** Bcrypt/Argon2 for password hashing (never MD5/SHA). **(2)** JWT with short expiry (15min) + refresh token rotation. **(3)** Rate-limit login attempts (prevent brute force). **(4)** MFA (TOTP) for sensitive operations. *(SQL injection = parameterized query se rok; Deserialization = Java serialization avoid kar, JSON use kar; Auth = bcrypt + JWT + rate limit)*

### 📖 How It Works
```
SQL INJECTION:

  VULNERABLE (string concatenation):
    String query = "SELECT * FROM users WHERE email = '" + userInput + "'";
    // userInput = "'; DROP TABLE users; --"
    // Executed: SELECT * FROM users WHERE email = ''; DROP TABLE users; --'
    // → TABLE DELETED! 💥

  SAFE (parameterized):
    @Query("SELECT u FROM User u WHERE u.email = :email")
    User findByEmail(@Param("email") String email);
    // userInput = "'; DROP TABLE users; --"
    // Treated as literal string → no injection possible ✅

DESERIALIZATION ATTACK:

  VULNERABLE (Java ObjectInputStream):
    ObjectInputStream ois = new ObjectInputStream(untrustedInput);
    Object obj = ois.readObject();  // ← can execute ARBITRARY CODE!
    // Attacker crafts byte stream that triggers malicious class constructors
    // → Remote Code Execution (RCE)!

  SAFE:
    // Option 1: Don't use Java serialization at all → use Jackson JSON ⭐
    ObjectMapper mapper = new ObjectMapper();
    User user = mapper.readValue(jsonString, User.class);
    
    // Option 2: Whitelist allowed classes (Java 9+)
    ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
        "com.app.model.*;!*");  // only allow com.app.model classes
    ois.setObjectInputFilter(filter);

AUTHENTICATION:

  VULNERABLE:
    user.setPassword(MD5.hash(password));  // ← rainbow table attack!
    // MD5 is fast = attacker can try billions/sec

  SAFE:
    user.setPassword(bcryptEncoder.encode(password));  // ← slow by design
    // BCrypt: ~100ms per hash → brute force infeasible
    // Salt included → rainbow tables useless
```

### 🗣️ How to Say in Interview
"I address OWASP Top 10 vulnerabilities systematically. For SQL injection, I use parameterized queries exclusively — JPA and Spring Data do this by default, and for native queries I use @Param named parameters, never string concatenation. For deserialization attacks, I avoid Java's built-in serialization entirely — all API communication uses JSON via Jackson. If legacy code requires ObjectInputStream, I use Java 9's ObjectInputFilter to whitelist only known-safe classes. For authentication, passwords are hashed with BCrypt which is intentionally slow to resist brute force. JWTs have short 15-minute expiry with refresh token rotation. Login endpoints are rate-limited to 5 attempts per minute per IP to prevent brute force. I also run OWASP Dependency-Check in our CI pipeline to catch known vulnerabilities in third-party libraries before they reach production."

### 💻 Code
```java
// 1. SQL INJECTION PREVENTION

// SAFE — Spring Data (parameterized automatically)
List<User> findByEmail(String email);  // Spring generates parameterized query

// SAFE — JPQL with @Param
@Query("SELECT u FROM User u WHERE u.email = :email AND u.active = :active")
Optional<User> findActiveByEmail(@Param("email") String email, @Param("active") boolean active);

// SAFE — Native query with @Param (still parameterized)
@Query(value = "SELECT * FROM users WHERE email = :email", nativeQuery = true)
Optional<User> findByEmailNative(@Param("email") String email);

// DANGEROUS — never do this!
// String sql = "SELECT * FROM users WHERE email = '" + email + "'";
// em.createNativeQuery(sql);  // ← SQL INJECTION VULNERABILITY!

// 2. DESERIALIZATION PROTECTION

// Use Jackson for all serialization (safe by default)
@RestController
public class UserController {
    // Jackson deserializes JSON → Java (no code execution risk)
    @PostMapping("/api/users")
    public UserDTO createUser(@RequestBody @Valid CreateUserRequest request) {
        return userService.create(request);
    }
}

// Jackson security — disable dangerous features
@Bean
public ObjectMapper objectMapper() {
    ObjectMapper mapper = new ObjectMapper();
    mapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, true);
    mapper.activateDefaultTyping(
        mapper.getPolymorphicTypeValidator(),
        ObjectMapper.DefaultTyping.NON_FINAL);  // restrict polymorphic types
    return mapper;
}

// If you MUST use Java serialization (legacy) — whitelist classes
ObjectInputFilter filter = ObjectInputFilter.Config.createFilter(
    "com.app.model.User;com.app.model.Order;!*");
ObjectInputStream ois = new ObjectInputStream(inputStream);
ois.setObjectInputFilter(filter);

// 3. AUTHENTICATION SECURITY

@Configuration
public class SecurityConfig {
    // BCrypt password encoder — slow hash (resistant to brute force)
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder(12);  // cost factor 12 ≈ 200ms per hash
    }
}

@Service
@RequiredArgsConstructor
public class AuthService {
    private final PasswordEncoder passwordEncoder;
    private final UserRepository userRepo;
    
    public void register(RegisterRequest req) {
        User user = new User();
        user.setEmail(req.email());
        user.setPassword(passwordEncoder.encode(req.password()));  // BCrypt hash
        userRepo.save(user);
    }
    
    public AuthResponse login(LoginRequest req) {
        User user = userRepo.findByEmail(req.email())
                .orElseThrow(() -> new BadCredentialsException("Invalid credentials"));
        
        if (!passwordEncoder.matches(req.password(), user.getPassword())) {
            // Rate limit: track failed attempts
            loginAttemptService.recordFailure(req.email());
            throw new BadCredentialsException("Invalid credentials");
            // Same message for both wrong email and wrong password → no enumeration
        }
        
        loginAttemptService.recordSuccess(req.email());
        return new AuthResponse(jwtUtil.generateToken(user));
    }
}

// 4. DEPENDENCY VULNERABILITY SCANNING
// pom.xml — OWASP Dependency-Check plugin
// <plugin>
//   <groupId>org.owasp</groupId>
//   <artifactId>dependency-check-maven</artifactId>
//   <version>9.0.0</version>
//   <configuration>
//     <failBuildOnCVSS>7</failBuildOnCVSS>  <!-- fail on high severity -->
//   </configuration>
// </plugin>
```

### ⚠️ Pitfalls / Gotchas
- **toString logging** — logging user input without sanitization can cause log injection *(User input seedha log mein print kiya toh log injection ho sakta hai)*
- **Jackson @JsonTypeInfo** without whitelist → deserialization RCE via polymorphic types
- **Same error message** for wrong email vs wrong password — prevents user enumeration
- **MD5/SHA-1/SHA-256 for passwords** = broken. Use BCrypt/Argon2 (designed to be slow)
- **JWT secret in application.yml** committed to Git → secret exposed. Use env variables or vault
- **OWASP checks disabled** "because they're slow" → CVEs deployed to production

### 🆚 vs. Comparison
| Attack | Prevention | Java/Spring Tool |
|--------|-----------|-----------------|
| SQL Injection | Parameterized queries | JPA @Param, PreparedStatement |
| XSS | Output encoding | Thymeleaf auto-escapes, Jackson |
| Deserialization RCE | Avoid ObjectInputStream | Jackson JSON, ObjectInputFilter |
| Brute Force | Rate limiting + BCrypt | Bucket4j, BCryptPasswordEncoder |
| CSRF | Token or stateless | Spring Security CSRF (or disable for JWT) |
| Dependency CVE | Scan + update | OWASP Dependency-Check, Snyk |

### ⚡ Remember
- **SQL Injection** → parameterized queries ALWAYS (never concatenate) *(Kabhi bhi SQL mein string concatenate mat karo)*
- **Deserialization** → avoid Java serialization, use Jackson JSON
- **Passwords** → BCrypt (slow by design), never MD5/SHA
- **JWT** → short expiry (15min) + refresh token, secret in vault
- **Rate limit** login + sensitive endpoints
- **OWASP Dependency-Check** in CI → catch CVEs before production

### 🔗 Follow-ups
- [Q7 → Secure REST APIs (full security layers)](#q7)
- [Q11 → CI/CD (OWASP check in pipeline)](#q11)
- Q16 → JWT authentication flow (architecture/01)
