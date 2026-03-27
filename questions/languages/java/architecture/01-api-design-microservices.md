# 🏗️ API Design, DevOps & Microservices (Q15–Q20)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q15"></a>
## Q15. How would you design pagination and sorting in a REST API?

### 📝 One-Liner
Use Spring Data's Pageable (page, size, sort params) → return a PageResponse DTO with data + metadata (totalElements, totalPages, hasNext) → use keyset pagination for deep pages.

### 🔑 Quick Answer
**Standard approach**: Accept `page`, `size`, `sort` query params → Spring Data's `Pageable` handles it automatically → return a wrapper DTO with `content` (data), `page`, `size`, `totalElements`, `totalPages`, `hasNext`. Limit `size` to prevent abuse (max 100). Default sort by relevance or created_at desc. **For large datasets** (millions of rows): use **keyset/cursor pagination** (`?after=lastId`) instead of OFFSET — OFFSET scans all skipped rows, keyset is constant-time. *(Page + size + sort = standard REST pagination — bade data ke liye keyset use karo)*

### 📖 How It Works
```
Standard OFFSET Pagination:
  GET /api/orders?page=0&size=20&sort=createdAt,desc

  Spring generates:
    SELECT * FROM orders ORDER BY created_at DESC LIMIT 20 OFFSET 0
  
  Response:
  {
    "content": [...20 orders...],
    "page": 0,
    "size": 20,
    "totalElements": 5432,
    "totalPages": 272,
    "hasNext": true,
    "hasPrevious": false
  }

  Page 100:  OFFSET 2000 → DB scans 2020 rows, returns 20 ← SLOW!

Keyset/Cursor Pagination (better for deep pages):
  GET /api/orders?size=20&after=eyJpZCI6MTAwMH0=  (Base64 encoded cursor)
  
  Spring generates:
    SELECT * FROM orders WHERE id < 1000 ORDER BY id DESC LIMIT 20
  
  Response:
  {
    "content": [...20 orders...],
    "cursors": {
      "after": "eyJpZCI6OTgxfQ==",   ← encode last item's ID
      "before": "eyJpZCI6MTAwMH0="
    },
    "hasNext": true
  }
  → Always O(1) with index, regardless of page depth!
```

### 🗣️ How to Say in Interview
"I implement pagination using Spring Data's Pageable interface — the controller accepts page, size, and sort as query parameters, and I pass the Pageable directly to the repository method. I wrap the Page result in a custom PageResponse DTO that includes the content plus metadata like totalElements, totalPages, and hasNext. I cap the maximum page size at 100 to prevent clients from requesting the entire dataset. For very large datasets — say millions of audit log entries — I use keyset pagination where the client passes the last seen ID as a cursor, and the query uses a WHERE clause instead of OFFSET. This keeps query performance constant regardless of how deep the user pages. I also add proper indexes on the sort columns."

### 💻 Code
```java
// Controller — Standard pagination
@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;
    
    @GetMapping
    public ResponseEntity<PageResponse<OrderDTO>> getOrders(
            @RequestParam(defaultValue = "0") int page,
            @RequestParam(defaultValue = "20") int size,
            @RequestParam(defaultValue = "createdAt,desc") String[] sort) {
        
        size = Math.min(size, 100); // cap max size
        Pageable pageable = PageRequest.of(page, size, parseSort(sort));
        Page<OrderDTO> result = orderService.getOrders(pageable);
        return ResponseEntity.ok(PageResponse.from(result));
    }
    
    private Sort parseSort(String[] sort) {
        // sort=createdAt,desc&sort=amount,asc → multi-column sort
        List<Sort.Order> orders = new ArrayList<>();
        for (int i = 0; i < sort.length; i += 2) {
            String field = sort[i];
            Sort.Direction dir = (i + 1 < sort.length && sort[i + 1].equalsIgnoreCase("asc"))
                    ? Sort.Direction.ASC : Sort.Direction.DESC;
            orders.add(new Sort.Order(dir, field));
        }
        return Sort.by(orders);
    }
}

// PageResponse DTO — wraps Spring's Page
public record PageResponse<T>(
    List<T> content,
    int page,
    int size,
    long totalElements,
    int totalPages,
    boolean hasNext,
    boolean hasPrevious
) {
    public static <T> PageResponse<T> from(Page<T> page) {
        return new PageResponse<>(
            page.getContent(), page.getNumber(), page.getSize(),
            page.getTotalElements(), page.getTotalPages(),
            page.hasNext(), page.hasPrevious()
        );
    }
}

// Repository — Spring Data handles Pageable automatically
public interface OrderRepository extends JpaRepository<Order, Long> {
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
    
    // Keyset pagination for deep pages
    @Query("SELECT o FROM Order o WHERE o.id < :cursor ORDER BY o.id DESC")
    List<Order> findNextPage(@Param("cursor") Long cursor, Pageable pageable);
}

// Service
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    
    public Page<OrderDTO> getOrders(Pageable pageable) {
        return orderRepo.findAll(pageable).map(this::toDTO);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **OFFSET performance degrades** with depth: page 10000 scans 200,000 rows before returning 20 *(OFFSET 200000 = 2 lakh rows skip karne mein time lagega)*
- **count(*)** for totalElements is expensive on large tables — consider `@Query(countQuery = ...)` or skip count for infinite scroll
- **Sort column MUST be indexed** — sorting without index = full table scan + in-memory sort
- **Non-deterministic sort** = inconsistent paging (items shift between pages). Always include unique column (id) as tiebreaker: `sort=createdAt,desc&sort=id,desc`
- **Don't expose** internal DB column names in API — validate/whitelist allowed sort fields

### 🆚 vs. Comparison
| Aspect | Offset Pagination | Keyset (Cursor) |
|--------|-------------------|-----------------|
| URL | `?page=5&size=20` | `?after=abc123&size=20` |
| DB query | `OFFSET 100 LIMIT 20` | `WHERE id < 1000 LIMIT 20` |
| Deep page perf | O(offset+limit) 🐌 | O(limit) ⚡ |
| Jump to page N | ✅ Easy | ❌ Sequential only |
| Total count | Available | Expensive/skipped |
| Use case | Admin tables, small data | Infinite scroll, large data |

### ⚡ Remember
- Spring `Pageable` = page + size + sort → automatic SQL generation
- Cap `size` to prevent abuse (max 100)
- Index sort columns + include unique tiebreaker (id)
- Deep pages → keyset pagination (`WHERE id < cursor`)
- Return metadata: totalElements, totalPages, hasNext

### 🔗 Follow-ups
- [Q14 → SQL query optimization (OFFSET vs keyset)](#q14)
- [Q17 → API performance bottleneck](#q17)

---

<a id="q16"></a>
## Q16. Explain the step-by-step flow of JWT authentication.

### 📝 One-Liner
Client sends credentials → server validates & returns signed JWT (header.payload.signature) → client sends JWT in Authorization header → server validates signature without DB lookup → stateless authentication.

### 🔑 Quick Answer
**Login flow**: (1) Client sends username+password to `/auth/login`. (2) Server validates credentials against DB. (3) Server creates JWT with claims (userId, roles, exp) and signs it with a secret key (HS256) or private key (RS256). (4) Server returns JWT in response. (5) Client stores JWT and sends it in `Authorization: Bearer <token>` header on every request. (6) Server's JWT filter extracts token → validates signature → extracts claims → sets SecurityContext → request proceeds. **No session, no DB lookup on every request** — the token IS the proof. *(JWT = digitally signed ID card — server verify karta hai bina DB check kiye)*

### 📖 How It Works
```
JWT Structure:
  eyJhbGciOiJIUzI1NiJ9.eyJzdWIiOiJ1c2VyMSIsInJvbGVzIjpbIkFETUlOIl0sImV4cCI6MTcwMH0.abc123signature

  Header:    {"alg": "HS256", "typ": "JWT"}
  Payload:   {"sub": "user1", "roles": ["ADMIN"], "iat": 1700000, "exp": 1700360}
  Signature: HMAC-SHA256(base64(header) + "." + base64(payload), SECRET_KEY)

Login Flow:
  Client                        Server
    |                              |
    |  POST /auth/login            |
    |  { email, password }  ──────>|
    |                              | Validate credentials (DB)
    |                              | Generate JWT (sign with secret)
    |  { accessToken, refreshToken,|
    |    expiresIn: 3600 }  <──────|
    |                              |
    | Store tokens (httpOnly cookie|
    | or secure storage)           |

Request Flow (every subsequent request):
  Client                        Server (JwtAuthFilter)
    |                              |
    |  GET /api/orders             |
    |  Authorization: Bearer eyJ...|──> Extract token from header
    |                              |──> Validate signature (HMAC/RSA)
    |                              |──> Check expiry (exp claim)
    |                              |──> Extract claims (sub, roles)
    |                              |──> Set SecurityContext
    |                              |──> Proceed to controller ✅
    |  { orders: [...] }    <──────|

Token Refresh Flow:
  Access token expired (401) → Client sends refresh token
  → Server validates refresh token → Issues new access token
  → Client retries original request with new token
```

### 🗣️ How to Say in Interview
"JWT authentication is stateless. During login, the server validates credentials, creates a JWT containing the user ID and roles, signs it with a secret key, and returns it to the client. On subsequent requests, the client includes the JWT in the Authorization Bearer header. A Spring Security filter intercepts the request, extracts the token, validates the signature — which proves the data hasn't been tampered with — checks expiration, and sets the authentication in SecurityContext. No database call is needed on each request — the signed token itself is the proof of identity. I use short-lived access tokens of 15 minutes with a longer-lived refresh token for re-authentication. In my project, I implemented this with Spring Security 6 using a OncePerRequestFilter."

### 💻 Code
```java
// JWT Utility class
@Component
public class JwtUtil {
    @Value("${jwt.secret}")
    private String secretKey;
    
    private static final long ACCESS_TOKEN_EXPIRY = 15 * 60 * 1000;   // 15 min
    private static final long REFRESH_TOKEN_EXPIRY = 7 * 24 * 60 * 60 * 1000; // 7 days
    
    public String generateAccessToken(UserDetails user) {
        return Jwts.builder()
                .subject(user.getUsername())
                .claim("roles", user.getAuthorities().stream()
                        .map(GrantedAuthority::getAuthority).toList())
                .issuedAt(new Date())
                .expiration(new Date(System.currentTimeMillis() + ACCESS_TOKEN_EXPIRY))
                .signWith(getSigningKey())
                .compact();
    }
    
    public String extractUsername(String token) {
        return extractClaims(token).getSubject();
    }
    
    public boolean isTokenValid(String token, UserDetails user) {
        String username = extractUsername(token);
        return username.equals(user.getUsername()) && !isExpired(token);
    }
    
    private Claims extractClaims(String token) {
        return Jwts.parser().verifyWith(getSigningKey()).build()
                .parseSignedClaims(token).getPayload();
    }
    
    private boolean isExpired(String token) {
        return extractClaims(token).getExpiration().before(new Date());
    }
    
    private SecretKey getSigningKey() {
        return Keys.hmacShaKeyFor(Decoders.BASE64.decode(secretKey));
    }
}

// JWT Authentication Filter
@Component
@RequiredArgsConstructor
public class JwtAuthFilter extends OncePerRequestFilter {
    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;
    
    @Override
    protected void doFilterInternal(HttpServletRequest request,
            HttpServletResponse response, FilterChain chain) throws Exception {
        
        String authHeader = request.getHeader("Authorization");
        if (authHeader == null || !authHeader.startsWith("Bearer ")) {
            chain.doFilter(request, response);
            return;
        }
        
        String token = authHeader.substring(7);
        String username = jwtUtil.extractUsername(token);
        
        if (username != null && SecurityContextHolder.getContext()
                .getAuthentication() == null) {
            UserDetails user = userDetailsService.loadUserByUsername(username);
            if (jwtUtil.isTokenValid(token, user)) {
                UsernamePasswordAuthenticationToken authToken =
                    new UsernamePasswordAuthenticationToken(user, null, user.getAuthorities());
                authToken.setDetails(new WebAuthenticationDetailsSource()
                    .buildDetails(request));
                SecurityContextHolder.getContext().setAuthentication(authToken);
            }
        }
        chain.doFilter(request, response);
    }
}

// Auth Controller
@RestController
@RequestMapping("/auth")
@RequiredArgsConstructor
public class AuthController {
    private final AuthenticationManager authManager;
    private final JwtUtil jwtUtil;
    private final UserDetailsService userDetailsService;
    
    @PostMapping("/login")
    public ResponseEntity<AuthResponse> login(@RequestBody @Valid LoginRequest req) {
        authManager.authenticate(
            new UsernamePasswordAuthenticationToken(req.email(), req.password()));
        UserDetails user = userDetailsService.loadUserByUsername(req.email());
        String accessToken = jwtUtil.generateAccessToken(user);
        return ResponseEntity.ok(new AuthResponse(accessToken, "Bearer", 900));
    }
}

// Security Config (Spring Security 6)
@Configuration
@EnableWebSecurity
@RequiredArgsConstructor
public class SecurityConfig {
    private final JwtAuthFilter jwtFilter;
    
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .csrf(csrf -> csrf.disable())  // stateless = no CSRF needed
            .sessionManagement(sm -> sm.sessionCreationPolicy(SessionCreationPolicy.STATELESS))
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/auth/**").permitAll()
                .requestMatchers("/api/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class)
            .build();
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Never store JWT in localStorage** — XSS can steal it. Use httpOnly cookie or secure in-memory storage *(localStorage mein JWT mat rakho — XSS se chori ho sakta hai)*
- **JWT cannot be invalidated** — once issued, it's valid until expiry. For logout: use short expiry + token blacklist (Redis) or refresh token rotation
- **Don't put sensitive data** in JWT payload — it's base64 encoded, NOT encrypted (anyone can decode it)
- **Secret key management** — store in environment variable or vault, never in code/properties
- **Clock skew** — server clocks slightly out of sync can cause premature expiry → add 30s leeway

### 🆚 vs. Comparison
| Aspect | JWT (Token) | Session (Cookie) |
|--------|------------|-----------------|
| State | Stateless ⭐ | Server stores session |
| Scalability | Very high (no shared state) | Needs sticky sessions or Redis |
| Revocation | Hard (blacklist needed) | Easy (delete session) |
| Size | Large (~800 bytes) | Small (session ID ~32 bytes) |
| DB hit per request | None ⭐ | Session store lookup |
| Mobile friendly | Yes (header-based) | Cookie issues on mobile |
| Best for | Microservices, APIs | Traditional web apps |

### 🎯 Tricky Interview Qs

**Q: How do you handle JWT logout?**
Three approaches: (1) Short-lived token (15min) — just let it expire. (2) Token blacklist in Redis — store revoked JTI (JWT ID) until natural expiry. (3) Refresh token rotation — revoke refresh token on logout, access token expires naturally.

**Q: HS256 vs RS256?**
HS256 = symmetric (same secret for sign+verify). RS256 = asymmetric (private key signs, public key verifies). RS256 is better for microservices — auth service signs with private key, all services verify with public key (no secret sharing). *(HS256 = ek hi key, RS256 = alag keys — microservices ke liye RS256 better hai)*

### ⚡ Remember
- JWT = **Header.Payload.Signature** (base64.base64.HMAC)
- Login → validate → sign → return token
- Request → extract → validate signature → set SecurityContext
- **Stateless** — no DB hit per request
- Short access token (15min) + refresh token (7 days)
- httpOnly cookie > localStorage (XSS protection)

### 🔗 Follow-ups
- [Q10 → Global exception handling (401/403)](#q10)
- [Q20 → Microservice fault tolerance (service-to-service auth)](#q20)

---

<a id="q17"></a>
## Q17. If your API performance degrades under load, how would you identify the bottleneck?

### 📝 One-Liner
Systematic: Monitor → Profile → Identify → Fix. Check metrics (latency, throughput, errors, saturation) → narrow down to DB/CPU/memory/network/thread pool → profile the specific layer → fix and verify.

### 🔑 Quick Answer
**(1) Monitor** — check Grafana/Prometheus dashboards: API response time (p50/p95/p99), error rate, throughput. **(2) Infrastructure** — CPU, memory, disk I/O, network. If CPU at 100% → code/algorithm issue. If memory full → leak or caching too much. **(3) Database** — slow query log, connection pool exhaustion (HikariCP metrics), long-running queries, lock contention. **(4) Thread pool** — Tomcat thread pool saturated (200 default), async tasks blocking. **(5) External dependencies** — downstream service slow (check circuit breaker stats). **(6) Application profiling** — thread dump (jstack), heap dump, Async Profiler/VisualVM for hot methods. *(Pehle metrics dekho — kahaan time lag raha hai — phir us layer ko investigate karo)*

### 📖 How It Works
```
Systematic Bottleneck Identification:

Step 1: WHERE is time spent? (API layer breakdown)
  Total: 500ms
  ├── Controller/Validation:   5ms
  ├── Service/Business Logic:  15ms
  ├── Database queries:        420ms  ← BOTTLENECK!
  ├── External API call:       50ms
  └── Serialization:           10ms

Step 2: Narrow down Database (most common bottleneck ~80%)
  Is it slow queries? → EXPLAIN ANALYZE → add index
  Is it too many queries? → N+1 → JOIN FETCH
  Is it connection pool? → HikariCP waiting metric high → increase pool size
  Is it lock contention? → check pg_stat_activity for blocked queries

Step 3: If not DB, check threads
  Thread dump (jstack PID):
    200 threads all WAITING on → HttpClient.execute()
    → Downstream service is slow → add timeout + circuit breaker

Step 4: If not threads, check memory
  GC logs: Full GC every 10 seconds, 500ms pause each
  → Heap too small or memory leak → increase Xmx or fix leak

Monitoring Stack:
  Micrometer → Prometheus → Grafana (dashboards)
  Spring Boot Actuator → /actuator/metrics, /actuator/health
  ELK Stack → log aggregation for error patterns
```

### 🗣️ How to Say in Interview
"I follow a systematic approach. First, I check the high-level metrics — API latency percentiles, error rates, and throughput using Grafana dashboards backed by Prometheus. I look at where time is spent: if p99 latency jumped from 50ms to 2 seconds, I check the breakdown. In 80% of cases, it's the database — I check for slow queries via the slow query log, connection pool saturation in HikariCP metrics, and N+1 queries in Hibernate statistics. If it's not the DB, I take a thread dump to see if threads are blocked waiting on locks or external calls. For memory issues, I check GC logs for frequent pauses. In my project, we had intermittent slowdowns that turned out to be HikariCP connection pool exhaustion — 10 connections with 200 concurrent requests meant 190 threads waiting. Increasing the pool to 30 and fixing a missing @Transactional that was holding connections too long resolved it."

### 💻 Code
```java
// Spring Boot Actuator + Micrometer setup
// pom.xml dependencies:
// spring-boot-starter-actuator
// micrometer-registry-prometheus

// application.yml
// management:
//   endpoints:
//     web:
//       exposure:
//         include: health, metrics, prometheus
//   metrics:
//     tags:
//       application: my-service

// Custom metric for API timing
@RestController
@RequiredArgsConstructor
public class OrderController {
    private final MeterRegistry meterRegistry;
    private final OrderService orderService;
    
    @GetMapping("/api/orders")
    public ResponseEntity<List<OrderDTO>> getOrders() {
        Timer.Sample sample = Timer.start(meterRegistry);
        try {
            List<OrderDTO> orders = orderService.getOrders();
            sample.stop(Timer.builder("api.orders.get")
                    .tag("status", "success").register(meterRegistry));
            return ResponseEntity.ok(orders);
        } catch (Exception e) {
            sample.stop(Timer.builder("api.orders.get")
                    .tag("status", "error").register(meterRegistry));
            throw e;
        }
    }
}

// HikariCP connection pool monitoring
// application.yml:
// spring:
//   datasource:
//     hikari:
//       maximum-pool-size: 20
//       minimum-idle: 5
//       connection-timeout: 30000    # wait for connection
//       leak-detection-threshold: 60000  # detect unreturned connections

// Key Actuator endpoints:
// GET /actuator/metrics/hikaricp.connections.active  → active DB connections
// GET /actuator/metrics/hikaricp.connections.pending  → threads waiting for conn
// GET /actuator/metrics/jvm.memory.used
// GET /actuator/metrics/jvm.gc.pause
// GET /actuator/metrics/http.server.requests  → API latency percentiles

// Health check with DB dependency
@Component
public class DatabaseHealthIndicator implements HealthIndicator {
    @Autowired private DataSource dataSource;
    
    @Override
    public Health health() {
        try (Connection conn = dataSource.getConnection()) {
            return Health.up()
                    .withDetail("database", "reachable")
                    .withDetail("pool.active", getActiveConnections())
                    .build();
        } catch (Exception e) {
            return Health.down().withException(e).build();
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Don't guess** — always measure first. "It feels slow" is not a diagnosis *(andaze se fix mat karo — pehle measure karo)*
- **p50 vs p99** — p50 can be 20ms while p99 is 5 seconds. Always check p95/p99 for real user experience
- **Connection pool too large** is also bad — each DB connection uses memory; more connections = more lock contention at DB level
- **Thread dumps need multiple samples** — one dump is a snapshot; take 3-5 dumps 5s apart to see patterns
- **GC pauses** can cause latency spikes even if code is fast — check GC logs before profiling code

### ⚡ Remember
- **Sequence**: Metrics → Infrastructure → Database → Threads → Memory → Code
- **80% of perf issues = database** (slow queries, N+1, pool exhaustion)
- **Measure, don't guess**: EXPLAIN ANALYZE, thread dumps, GC logs
- **Monitor**: Micrometer + Prometheus + Grafana + Actuator
- **Key metrics**: p95/p99 latency, error rate, throughput, DB pool usage

### 🔗 Follow-ups
- [Q14 → SQL query optimization](#q14)
- [Q20 → Fault tolerance (circuit breakers, timeouts)](#q20)
- [Q15 → Pagination (reducing data volume)](#q15)

---

<a id="q18"></a>
## Q18. REST vs Messaging (Kafka) — when would you choose each approach?

### 📝 One-Liner
REST for synchronous request-response (need immediate answer); Kafka/messaging for asynchronous fire-and-forget (event-driven, decoupled, high throughput) — most systems need both.

### 🔑 Quick Answer
**REST (synchronous)**: Client needs immediate response. Example: user places order → needs order confirmation now. Simple, well-understood, easy to debug. Problem: tight coupling (caller waits, fails if target is down). **Kafka (asynchronous)**: Producer publishes event, consumers process independently. Example: order placed → notify inventory, send email, update analytics — all happen independently. Decoupled (producer doesn't know consumers), resilient (events stored even if consumer is down), high throughput (millions of events/sec). Problem: eventual consistency, harder to debug, no immediate response. **Rule**: REST for queries and commands needing immediate response; Kafka for events, notifications, and cross-service data propagation. *(REST = turant jawab chahiye; Kafka = kaam ho jayega, abhi nahi toh baad mein)*

### 📖 How It Works
```
REST (Synchronous):
  Order Service → REST call → Inventory Service (deduct stock)
  |                                   |
  | POST /api/inventory/deduct        |
  |──────────────────────────────────>| Process
  |                                   | Return result
  |<──────────────────────────────────|
  | Got response ✅ (or timeout ❌)   |
  
  Problem: If Inventory Service is DOWN → Order Service FAILS!

Kafka (Asynchronous / Event-Driven):
  Order Service → Kafka Topic → [Inventory, Email, Analytics]
  |                    |                |         |        |
  | Publish event      |                |         |        |
  |───>  ┌──────────┐ |                |         |        |
  |      │ order.    │ ├───> Inventory Service (deduct stock)
  |      │ created   │ ├───> Email Service (send confirmation)
  |      │           │ └───> Analytics Service (update dashboard)
  |      └──────────┘
  | Done! (doesn't wait) ✅
  
  Benefits:
  - Order Service doesn't know about consumers
  - If Email Service is DOWN → event stays in Kafka → processes when back up
  - Adding new consumers doesn't change Order Service code
```

### 🗣️ How to Say in Interview
"I choose REST for synchronous operations where the client needs an immediate response — like fetching user profiles, validating payment, or submitting orders. Kafka is my choice for asynchronous event-driven communication — when an order is placed, I publish an OrderCreated event to Kafka, and multiple consumers (inventory, email, analytics) independently process it. This gives me loose coupling — the order service doesn't know about downstream consumers — and resilience, since Kafka retains events even if a consumer is temporarily down. In practice, most systems use both: REST for the client-facing API, and Kafka for inter-service event propagation. In my project, order placement was REST to get immediate confirmation, but stock updates, email notifications, and audit logging were all Kafka consumers."

### 💻 Code
```java
// KAFKA PRODUCER — publish event after order creation
@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final KafkaTemplate<String, OrderEvent> kafkaTemplate;
    
    @Transactional
    public OrderDTO createOrder(CreateOrderRequest request) {
        Order order = orderRepo.save(toEntity(request));
        
        // Publish event — consumers process asynchronously
        OrderEvent event = new OrderEvent(order.getId(), order.getCustomerId(),
                order.getItems(), order.getTotalAmount(), Instant.now());
        kafkaTemplate.send("order-events", order.getId().toString(), event);
        
        return toDTO(order);  // immediate response to client
    }
}

// KAFKA CONSUMER — Inventory Service (different microservice)
@Component
@RequiredArgsConstructor
public class InventoryEventConsumer {
    private final InventoryService inventoryService;
    
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handleOrderCreated(OrderEvent event) {
        for (OrderItem item : event.items()) {
            inventoryService.decrementStock(item.productId(), item.quantity());
        }
    }
}

// KAFKA CONSUMER — Email Service (different microservice)
@Component
public class EmailEventConsumer {
    @KafkaListener(topics = "order-events", groupId = "email-service")
    public void handleOrderCreated(OrderEvent event) {
        emailService.sendOrderConfirmation(event.customerId(), event.orderId());
    }
}

// Event class
public record OrderEvent(
    Long orderId,
    Long customerId,
    List<OrderItem> items,
    BigDecimal totalAmount,
    Instant timestamp
) {}

// Kafka Config
@Configuration
public class KafkaConfig {
    @Bean
    public NewTopic orderEventsTopic() {
        return TopicBuilder.name("order-events")
                .partitions(6)
                .replicas(3)
                .config(TopicConfig.RETENTION_MS_CONFIG, String.valueOf(7 * 24 * 3600 * 1000))
                .build();
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Dual write problem**: writing to DB + Kafka is NOT atomic. Solutions: Transactional Outbox pattern or CDC (Change Data Capture) *(DB aur Kafka dono mein likhna atomic nahi hai — Outbox pattern use karo)*
- **Message ordering**: Kafka only guarantees order within a partition. Use same key (e.g., orderId) for related events
- **Idempotency**: consumers may receive duplicates (at-least-once delivery). Make processing idempotent (check if already processed)
- **Debugging is harder** with async — use correlation IDs in events for tracing
- **Don't use Kafka for simple request-response** — REST is simpler and sufficient

### 🆚 vs. Comparison
| Aspect | REST (Sync) | Kafka (Async) |
|--------|-------------|---------------|
| Communication | Request-Response | Publish-Subscribe |
| Coupling | Tight (knows target) | Loose (topic-based) ⭐ |
| Availability | Fails if target down | Events stored, processed later ⭐ |
| Latency | Immediate response | Eventually consistent |
| Throughput | Moderate | Very high (millions/sec) ⭐ |
| Debugging | Easy (request→response) | Harder (distributed tracing) |
| Use case | Queries, commands | Events, notifications |
| Data flow | Point-to-point | Fan-out (1 event → N consumers) ⭐ |

### ⚡ Remember
- **REST** = need answer NOW (query, command) — synchronous
- **Kafka** = fire and forget (event, notification) — asynchronous *(REST = turant jawab, Kafka = event publish karo bhool jao)*
- Most systems use BOTH: REST for client API, Kafka for inter-service
- Kafka guarantees: ordering per partition, at-least-once delivery, retention
- Outbox pattern for atomic DB + Kafka writes

### 🔗 Follow-ups
- [Q20 → Fault tolerance (resilience in async systems)](#q20)
- [Q17 → API performance bottleneck (async as optimization)](#q17)

---

<a id="q19"></a>
## Q19. How would you Dockerize a Spring Boot application?

### 📝 One-Liner
Multi-stage Dockerfile: build with Maven/Gradle image → copy JAR to slim JRE image → expose port → run with proper JVM flags. Use Docker Compose for local dev with DB.

### 🔑 Quick Answer
**(1) Multi-stage Dockerfile**: Stage 1 uses Maven image to build the JAR (compile + test). Stage 2 copies only the JAR into a slim JRE image (eclipse-temurin:21-jre-alpine). This gives a small final image (~200MB vs 800MB). **(2) .dockerignore** to exclude .git, target, node_modules. **(3) Non-root user** for security. **(4) JVM flags** for containers: `-XX:MaxRAMPercentage=75.0` (uses 75% of container memory for heap). **(5) Docker Compose** for local dev with Postgres, Redis. **(6) Health check** in Dockerfile using Actuator endpoint. *(Multi-stage build = chhota image, non-root user = secure, JVM flags = container-aware)*

### 📖 How It Works
```
Multi-stage Build:

Stage 1: Build (Maven + JDK)
  ┌─────────────────────────┐
  │ FROM maven:3.9-eclipse- │
  │   temurin-21 AS build   │
  │ COPY pom.xml + src/     │
  │ RUN mvn package -DskipT │  → produces app.jar (~40MB)
  └─────────────────────────┘
            ↓ (only JAR copied)
Stage 2: Runtime (JRE only)
  ┌─────────────────────────┐
  │ FROM eclipse-temurin:   │
  │   21-jre-alpine         │  → ~180MB base (no JDK, no build tools)
  │ COPY --from=build app.jar│
  │ USER 1001               │  → non-root
  │ ENTRYPOINT ["java",     │
  │   "-jar", "app.jar"]    │
  └─────────────────────────┘
  Final image: ~220MB (vs 800MB single-stage)

Docker Compose (local dev):
  ┌─────────────┐      ┌──────────┐      ┌──────────┐
  │ Spring Boot │─────>│ Postgres │      │  Redis   │
  │  :8080      │      │  :5432   │      │  :6379   │
  └─────────────┘      └──────────┘      └──────────┘
       ↑ depends_on         ↑ healthcheck      ↑
       └── Waits for DB to be ready ──────────┘
```

### 🗣️ How to Say in Interview
"I use a multi-stage Dockerfile. The first stage uses a Maven image to build the JAR — this includes the JDK and build tools. The second stage copies only the JAR into a slim JRE-alpine image, which reduces the final image from around 800MB to 220MB. I run the application as a non-root user for security, and I set JVM flags like MaxRAMPercentage to 75% so the JVM respects the container's memory limit. For local development, I use Docker Compose with Postgres and Redis as services. I also configure a health check using the Spring Actuator health endpoint. For production, the image is built in CI/CD, pushed to a container registry, and deployed to Kubernetes."

### 💻 Code
```dockerfile
# Dockerfile — Multi-stage build

# Stage 1: Build
FROM maven:3.9-eclipse-temurin-21 AS build
WORKDIR /app
COPY pom.xml .
RUN mvn dependency:go-offline          # cache dependencies
COPY src/ src/
RUN mvn package -DskipTests -q        # build JAR

# Stage 2: Runtime
FROM eclipse-temurin:21-jre-alpine
WORKDIR /app

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup
USER appuser

# Copy only the JAR
COPY --from=build /app/target/*.jar app.jar

# Container-aware JVM flags
ENV JAVA_OPTS="-XX:MaxRAMPercentage=75.0 -XX:+UseG1GC -XX:+ExitOnOutOfMemoryError"

EXPOSE 8080

HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD wget -qO- http://localhost:8080/actuator/health || exit 1

ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

```yaml
# docker-compose.yml — Local development
version: '3.8'
services:
  app:
    build: .
    ports:
      - "8080:8080"
    environment:
      - SPRING_DATASOURCE_URL=jdbc:postgresql://db:5432/myapp
      - SPRING_DATASOURCE_USERNAME=postgres
      - SPRING_DATASOURCE_PASSWORD=postgres
      - SPRING_REDIS_HOST=redis
    depends_on:
      db:
        condition: service_healthy
      redis:
        condition: service_started
  
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: myapp
      POSTGRES_USER: postgres
      POSTGRES_PASSWORD: postgres
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U postgres"]
      interval: 5s
      timeout: 5s
      retries: 5
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

volumes:
  pgdata:
```

```text
# .dockerignore
.git
.gitignore
target/
*.md
.idea/
.vscode/
*.iml
docker-compose.yml
```

### ⚠️ Pitfalls / Gotchas
- **Running as root** — Docker default is root. Always create and use non-root user *(root se mat chalao — security risk hai)*
- **Single-stage build** → 800MB image with JDK + Maven inside. Multi-stage = only JRE
- **JVM not container-aware** (old JDK) — uses host memory, not container limit. Java 10+ defaults are container-aware, but explicitly set `MaxRAMPercentage`
- **COPY pom.xml before src/** — Docker caches dependency download layer, only rebuilds when pom changes
- **`-DskipTests` in Docker build** — tests should run in CI pipeline, not during image build
- **Secrets in ENV vars** visible in `docker inspect`. Use Docker secrets or external vault for production

### ⚡ Remember
- **Multi-stage** = small image (JRE, not JDK) *(Multi-stage = chhota image, fast deploy)*
- **Non-root user** = security
- **MaxRAMPercentage=75** = container-aware JVM
- **Layer caching**: COPY pom.xml → download deps → COPY src
- **Docker Compose** for local dev with DB + Redis
- **Health check** with Actuator endpoint

### 🔗 Follow-ups
- [Q20 → Microservice fault tolerance](#q20)
- [Q17 → API performance monitoring (containerized)](#q17)

---

<a id="q20"></a>
## Q20. If two microservices fail during communication, how would you implement fault tolerance?

### 📝 One-Liner
Circuit Breaker (Resilience4j) + Retry + Timeout + Fallback — prevent cascade failures by short-circuiting calls to failing services and returning degraded responses.

### 🔑 Quick Answer
**Circuit Breaker** (main pattern): monitors call failure rate. When failures exceed threshold (e.g., 50% in last 10 calls), it **opens** — all calls immediately fail-fast (no network call). After a wait period, it **half-opens** — allows a few test calls. If they succeed, circuit **closes** (back to normal). Combined with: **Retry** (retry transient failures 2-3 times with exponential backoff), **Timeout** (don't wait forever — fail after 3s), **Fallback** (return cached/default response). **Bulkhead** (isolate resources — failure in one service doesn't exhaust all threads). Spring Boot uses **Resilience4j** for all these patterns. *(Circuit Breaker = agar service baar baar fail ho rahi hai toh call karna hi band kar do — cascade failure ruko)*

### 📖 How It Works
```
Circuit Breaker States:

  CLOSED (normal) ──failure rate > 50%──> OPEN (fail-fast)
       ↑                                       |
       |                                  wait 30 seconds
       |                                       ↓
       └──success rate OK──── HALF-OPEN (test calls)
                                       |
                                  failure ──> OPEN (back to fail-fast)

Timeline:
  Call 1: ✅ (CLOSED)
  Call 2: ✅ (CLOSED)
  Call 3: ❌ timeout (CLOSED, counting failures)
  Call 4: ❌ timeout (CLOSED, 2/4 = 50%)
  Call 5: ❌ timeout (CLOSED → OPEN! 3/5 = 60% > threshold)
  Call 6: ⚡ Instant fail — no network call! (OPEN)
  Call 7: ⚡ Instant fail — no network call! (OPEN)
  ...30 seconds pass...
  Call 8: ❓ Test call (HALF-OPEN) → ✅ Success
  Call 9: ❓ Test call (HALF-OPEN) → ✅ Success
  → CLOSED! (back to normal)

Combined Pattern (Retry → Circuit Breaker → Timeout → Fallback):
  Client call
    └─> Retry (max 3 attempts, exponential backoff)
        └─> Circuit Breaker (fail-fast if open)
            └─> Timeout (3 second limit)
                └─> Actual HTTP call to service
                     ├─ ✅ Success → return response
                     └─ ❌ Fail → Retry → ... → Fallback
```

### 🗣️ How to Say in Interview
"I implement fault tolerance using Resilience4j with four complementary patterns. The circuit breaker monitors the failure rate of calls to a downstream service. When failures exceed a threshold — say 50% over the last 10 calls — it opens the circuit, and all subsequent calls immediately fail-fast without making the network call. This prevents cascading failures. After a wait period, it enters half-open state and allows test calls. I combine this with retry for transient failures with exponential backoff, a timeout so we don't wait indefinitely, and a fallback method that returns a cached or degraded response. I also use the bulkhead pattern to isolate thread pools per downstream service — if the inventory service is slow, it shouldn't exhaust all threads and affect payment calls. In my project, when the recommendation service was down, the circuit breaker activated and we returned cached recommendations instead of failing the entire product page."

### 💻 Code
```java
// Resilience4j with Spring Boot — annotation-based

@Service
@RequiredArgsConstructor
public class ProductService {
    private final InventoryClient inventoryClient;
    private final CacheManager cacheManager;
    
    @CircuitBreaker(name = "inventory", fallbackMethod = "inventoryFallback")
    @Retry(name = "inventory")
    @TimeLimiter(name = "inventory")
    @Bulkhead(name = "inventory")
    public CompletableFuture<InventoryResponse> checkInventory(Long productId) {
        return CompletableFuture.supplyAsync(
            () -> inventoryClient.getStock(productId)
        );
    }
    
    // Fallback — called when circuit is open or all retries exhausted
    private CompletableFuture<InventoryResponse> inventoryFallback(
            Long productId, Throwable ex) {
        // Return cached data or default response
        InventoryResponse cached = cacheManager.getCache("inventory")
                .get(productId, InventoryResponse.class);
        if (cached != null) return CompletableFuture.completedFuture(cached);
        return CompletableFuture.completedFuture(
                new InventoryResponse(productId, -1, "UNKNOWN"));
    }
}

// application.yml — Resilience4j configuration
// resilience4j:
//   circuitbreaker:
//     instances:
//       inventory:
//         sliding-window-size: 10         # evaluate last 10 calls
//         failure-rate-threshold: 50       # open at 50% failure
//         wait-duration-in-open-state: 30s # wait before half-open
//         permitted-number-of-calls-in-half-open-state: 3
//         record-exceptions:
//           - java.io.IOException
//           - java.util.concurrent.TimeoutException
//   retry:
//     instances:
//       inventory:
//         max-attempts: 3
//         wait-duration: 500ms
//         exponential-backoff-multiplier: 2   # 500ms, 1s, 2s
//         retry-exceptions:
//           - java.io.IOException
//   timelimiter:
//     instances:
//       inventory:
//         timeout-duration: 3s
//   bulkhead:
//     instances:
//       inventory:
//         max-concurrent-calls: 25           # isolate thread pool

// Feign client with Resilience4j
@FeignClient(name = "inventory-service", url = "${inventory.url}")
public interface InventoryClient {
    @GetMapping("/api/inventory/{productId}")
    InventoryResponse getStock(@PathVariable Long productId);
}

// Actuator endpoint for circuit breaker status
// GET /actuator/circuitbreakers
// → {"inventory": {"state": "CLOSED", "failureRate": 10.0, ...}}

// Monitoring circuit breaker events
@Component
public class CircuitBreakerEventLogger {
    @PostConstruct
    public void init() {
        CircuitBreaker cb = circuitBreakerRegistry.circuitBreaker("inventory");
        cb.getEventPublisher()
            .onStateTransition(event ->
                log.warn("Circuit breaker state: {} → {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()));
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Retry without backoff** = hammering a failing service → makes it worse *(Retry bina backoff ke lagaya toh fail service pe aur load padega)*
- **Timeout must be < circuit breaker window** — otherwise you're counting timeouts that haven't happened yet
- **Retry + Circuit Breaker order matters**: Retry wraps Circuit Breaker (retry the CB, not retry inside CB)
- **Fallback shouldn't call the same failing service** — use cache, defaults, or a completely different path
- **Bulkhead is critical** but often forgotten — without it, one slow service can exhaust all Tomcat threads

### 🆚 vs. Comparison
| Pattern | Purpose | When |
|---------|---------|------|
| Circuit Breaker | Stop calling failing service | Persistent failures |
| Retry | Retry transient failures | Network glitch, 503 |
| Timeout | Don't wait forever | Slow responses |
| Fallback | Return degraded response | All else fails |
| Bulkhead | Isolate resources | Prevent cascade |
| Rate Limiter | Limit call rate | Protect downstream |

### 🎯 Tricky Interview Qs

**Q: How is circuit breaker different from a simple retry?**
Retry handles transient failures (try again and it might work). Circuit breaker handles persistent failures (service is DOWN — stop trying completely for a while). They complement each other: retry 3 times, then if the circuit opens, fail-fast without any calls.

**Q: What about distributed transactions when a service fails mid-operation?**
Use the Saga pattern: each service performs its local transaction and publishes an event. If a step fails, compensating transactions are triggered to undo previous steps. Example: Order created → Payment deducted → Inventory updated. If inventory fails → compensate by refunding payment → cancelling order. *(Saga pattern = har step ka ulta step bhi define karo — rollback ke liye)*

### ⚡ Remember
- **Circuit Breaker** = CLOSED → OPEN (fail-fast) → HALF-OPEN (test) → CLOSED
- **Resilience4j** annotations: @CircuitBreaker, @Retry, @TimeLimiter, @Bulkhead
- **Order**: Retry → CircuitBreaker → Timeout → actual call → Fallback
- **Fallback** = return cached/default data, NOT call same service
- **Saga pattern** for distributed transactions (compensating actions)

### 🔗 Follow-ups
- [Q18 → REST vs Kafka (async communication for resilience)](#q18)
- [Q19 → Docker (deploying resilient services)](#q19)
- [Q17 → Performance bottleneck (timeouts and circuit breakers)](#q17)

---

## Q16. What is Rate Limiting and How Would You Implement It?

### 📝 One-Liner
Rate limiting controls the number of API requests a client can make in a given time window to protect services.

### 🔑 Quick Answer
Algorithms: Token Bucket (allows bursts), Sliding Window (precise), Fixed Window (simple). Implement at API Gateway level using Redis for distributed state. Return 429 with Retry-After header. *(har client ki request limit karo — Token Bucket sabse common hai)*

### 📖 How It Works
1. **Token Bucket**: Tokens refill at constant rate. Each request costs 1 token. Empty bucket = reject.
2. **Fixed Window**: Count requests in current window (e.g., per minute). Reset at window boundary.
3. **Sliding Window**: Weighted combination of current + previous window for smoother limiting.

**Where to implement**:
- **API Gateway** (Kong, Spring Cloud Gateway): centralized, no app code changes
- **Application Level**: Custom filter/interceptor
- **Distributed**: Redis `INCR` + `EXPIRE` for shared counter

### 💻 Code
```java
// Spring Boot Rate Limiter with Bucket4j
@Bean
public FilterRegistrationBean<RateLimitFilter> rateLimitFilter() {
    Bandwidth limit = Bandwidth.classic(100, Refill.intervally(100, Duration.ofMinutes(1)));
    Bucket bucket = Bucket.builder().addLimit(limit).build();
    // 100 requests per minute, bucket allows short bursts
    return new FilterRegistrationBean<>(new RateLimitFilter(bucket));
}

// Distributed rate limiting with Redis
public boolean isAllowed(String clientId) {
    String key = "rate:" + clientId + ":" + currentMinute();
    Long count = redis.opsForValue().increment(key);
    if (count == 1) redis.expire(key, 60, TimeUnit.SECONDS);
    return count <= MAX_REQUESTS_PER_MINUTE;
}
```

### 🆚 vs. Comparison
| Algorithm | Pros | Cons |
|-----------|------|------|
| Token Bucket | Allows bursts, smooth | Slightly complex |
| Fixed Window | Simple | Edge-of-window burst problem |
| Sliding Window | Precise | More memory (timestamps) |

### ⚡ Remember
- 429 Too Many Requests + `Retry-After` header
- Redis for distributed rate limiting across instances
- Different limits: per-user, per-IP, per-API-key
- Bucket4j or Resilience4j RateLimiter for Java
