# 🔗 Microservices — Feign Client & RestTemplate vs WebClient (Q1–Q2)

> **Source**: Capgemini + Java Developer Interview (4+ years)  
> **Coverage**: Declarative HTTP clients, synchronous vs reactive inter-service communication

---

<a id="q1"></a>
## Q1. What is Feign Client? Why might it not be recommended for production?

### 📝 One-Liner
Feign Client is a **declarative HTTP client** where you define an interface with annotations and Spring generates the implementation — simpler than RestTemplate but has production concerns around thread blocking and limited resilience.

### 🔑 Quick Answer
**OpenFeign** (Spring Cloud) — write an interface with `@FeignClient` and `@GetMapping`/`@PostMapping` annotations → Spring auto-generates the HTTP client. No manual URL building, no RestTemplate boilerplate. Integrates with Eureka (service discovery), load balancing, and Resilience4j. **Production concerns**: **(1)** Synchronous/blocking by default → thread pool exhaustion under load. **(2)** Default HTTP client (URLConnection) has no connection pooling → replace with Apache HttpClient or OkHttp. **(3)** Default timeout = NONE → must configure explicitly. **(4)** Error handling requires custom `ErrorDecoder`. **(5)** No reactive/non-blocking support → WebClient preferred for reactive stacks. *(Feign = interface likho, HTTP call automatic — lekin production mein timeouts, connection pool, aur error handling configure karna zaroori hai)*

### 📖 How It Works (Detailed Explanation)

```
Feign Client Flow:
┌─────────────────────────────────────┐
│ @FeignClient(name = "user-service") │
│ interface UserClient {              │
│   @GetMapping("/users/{id}")        │
│   User getUser(@PathVariable Long id);│
│ }                                   │
└──────────────┬──────────────────────┘
               │
     Spring generates proxy at startup
               │
               ▼
┌─────────────────────────────────────┐
│ 1. Resolve "user-service" → URL     │ ← Service Discovery (Eureka)
│ 2. Load balance across instances    │ ← Spring Cloud LoadBalancer
│ 3. Build HTTP request               │
│ 4. Apply interceptors (auth header) │
│ 5. Send request (blocking I/O)      │
│ 6. Decode response → User object    │ ← Jackson deserialization
│ 7. Handle errors (ErrorDecoder)     │
└─────────────────────────────────────┘
```

**Why not raw Feign in production?** The defaults are development-friendly but production-dangerous: no timeouts (infinite wait), no connection pooling (new connection per request), basic error handling. Must configure: (1) Apache HttpClient 5 backend for connection pooling, (2) explicit connect/read timeouts, (3) Resilience4j circuit breaker + retry, (4) custom ErrorDecoder for meaningful error propagation.

### 🗣️ Interview Script
"Feign Client is a declarative HTTP client from Spring Cloud where I define an interface annotated with @FeignClient and standard Spring MVC annotations — Spring generates the proxy implementation at runtime. It integrates with service discovery, so I reference the service name instead of hardcoding URLs. It's very clean and readable compared to manual RestTemplate calls. However, out of the box, Feign has production concerns. It's synchronous and blocking — each call holds a thread until the response arrives, which can exhaust the thread pool under load. The default HTTP client has no connection pooling, so I always swap in Apache HttpClient. I must explicitly set connect and read timeouts — the defaults are infinite. And I wrap Feign calls with Resilience4j for circuit breaking and retry. For reactive microservices, I prefer WebClient since Feign doesn't support non-blocking I/O."

### 💻 Code Example

```java
// ✅ Basic Feign Client definition
@FeignClient(
    name = "product-service",
    fallbackFactory = ProductClientFallbackFactory.class  // Resilience4j fallback
)
public interface ProductClient {

    @GetMapping("/api/products/{id}")
    Product getProduct(@PathVariable("id") Long productId);

    @GetMapping("/api/products")
    List<Product> getProducts(@RequestParam("category") String category);

    @PostMapping("/api/products")
    Product createProduct(@RequestBody ProductRequest request);
}

// ✅ Usage — inject like any Spring bean
@Service
public class OrderService {
    private final ProductClient productClient;

    public OrderService(ProductClient productClient) {
        this.productClient = productClient;
    }

    public OrderResponse createOrder(OrderRequest request) {
        Product product = productClient.getProduct(request.getProductId());
        // ... business logic
    }
}

// ✅ Enable Feign in main class
@SpringBootApplication
@EnableFeignClients
public class OrderServiceApp { }

// ✅ Fallback for circuit breaker
@Component
public class ProductClientFallbackFactory implements FallbackFactory<ProductClient> {
    @Override
    public ProductClient create(Throwable cause) {
        return new ProductClient() {
            @Override
            public Product getProduct(Long id) {
                log.warn("Fallback for getProduct({}): {}", id, cause.getMessage());
                return Product.defaultProduct();  // cached/default response
            }
            @Override
            public List<Product> getProducts(String category) {
                return Collections.emptyList();
            }
            @Override
            public Product createProduct(ProductRequest request) {
                throw new ServiceUnavailableException("Product service unavailable");
            }
        };
    }
}

// ✅ Request interceptor (e.g., pass auth token)
@Component
public class AuthFeignInterceptor implements RequestInterceptor {
    @Override
    public void apply(RequestTemplate template) {
        String token = SecurityContextHolder.getContext()
            .getAuthentication().getCredentials().toString();
        template.header("Authorization", "Bearer " + token);
    }
}
```

```yaml
# ✅ Production Feign configuration
spring:
  cloud:
    openfeign:
      client:
        config:
          product-service:
            connect-timeout: 3000       # 3 seconds (default: NONE!)
            read-timeout: 5000          # 5 seconds (default: NONE!)
            logger-level: basic
      httpclient:
        hc5:
          enabled: true                 # Use Apache HttpClient 5 (connection pooling!)
          pool-reuse-policy: fifo
      circuitbreaker:
        enabled: true                   # Resilience4j integration

resilience4j:
  circuitbreaker:
    instances:
      product-service:
        sliding-window-size: 10
        failure-rate-threshold: 50
        wait-duration-in-open-state: 10s
```

### ⚠️ Common Pitfalls
- **No timeout configured** — default is INFINITE → thread hangs forever on slow downstream
- **Default URLConnection client** — no connection pooling → add Apache HttpClient 5 dependency
- **No circuit breaker** — one slow/down service cascades to your service → use Resilience4j
- **Synchronous blocking** — each Feign call blocks a thread → not suitable for high-throughput reactive architectures
- **Token propagation** — security context doesn't automatically flow to Feign calls → use `RequestInterceptor`

### 🆚 Comparison Table

| Aspect | Feign Client | RestTemplate | WebClient |
|--------|-------------|-------------|-----------|
| Style | Declarative (interface) | Imperative | Reactive/imperative |
| Blocking | ✅ Yes (synchronous) | ✅ Yes | ❌ Non-blocking |
| Service discovery | Built-in (`@FeignClient(name=...)`) | Manual / LoadBalanced | Manual / LoadBalanced |
| Boilerplate | **Minimal** ⭐ | Moderate | Moderate |
| Connection pooling | Needs config (Apache HC) | Needs config | Built-in (Reactor Netty) |
| Reactive support | ❌ No | ❌ No | ✅ Yes |
| Modern status | Active | **Deprecated** (Boot 3.2+) | Recommended |

### ⚡ Remember (Quick Recall)
- Feign = **declare interface** → Spring generates HTTP client
- **Production essentials**: timeouts + Apache HttpClient + circuit breaker + error decoder
- Default = no timeout, no pooling, no resilience → configure everything!
- Blocking → not for reactive stacks → use WebClient
- `@FeignClient(name = "service")` resolves via service discovery

---

<a id="q2"></a>
## Q2. What is the difference between RestTemplate and WebClient?

### 📝 One-Liner
`RestTemplate` is **synchronous/blocking** (thread waits for response); `WebClient` is **non-blocking/reactive** (thread released during I/O, callback on response) — WebClient is the modern replacement.

### 🔑 Quick Answer
**RestTemplate** — synchronous HTTP client. Thread blocks until response arrives. Simple API: `restTemplate.getForObject(url, Type.class)`. **Deprecated for new code** since Spring 5/Boot 3.2 (replaced by `RestClient` for sync and `WebClient` for reactive). **WebClient** — non-blocking reactive HTTP client (part of Spring WebFlux). Thread is released during I/O wait → can handle many concurrent requests with fewer threads. Returns `Mono<T>` (single) or `Flux<T>` (stream). Can also be used in **blocking mode** (`.block()`) in non-reactive apps as a RestTemplate replacement. *(RestTemplate = blocking, purana; WebClient = non-blocking, naya — ab WebClient ya RestClient use karo)*

### 📖 How It Works (Detailed Explanation)

```
RestTemplate (synchronous/blocking):
Thread 1: ──── send request ────[WAITING/BLOCKED]──── receive response ────→ continue
                                 ↑ thread does NOTHING
                                   but can't be reused

WebClient (async/non-blocking):
Thread 1: ──── send request ──→ [thread returned to pool]
                                        ...
Thread 2: ←── response callback ──── process response ────→
           ↑ could be a different thread
```

**Thread efficiency**: with RestTemplate, 100 concurrent downstream calls need 100 threads (all blocked waiting). With WebClient, 100 calls can be handled by ~4 threads (event loop). **Backpressure**: WebClient with reactive streams supports backpressure — downstream can signal how much data it can handle. **Spring Boot 3.2+ RestClient**: new synchronous HTTP client with fluent API (modern replacement for RestTemplate in non-reactive apps). Use `WebClient` for reactive, `RestClient` for synchronous.

### 🗣️ Interview Script
"RestTemplate is the traditional synchronous HTTP client — it blocks the calling thread until the downstream service responds. This means 200 concurrent API calls need 200 threads, all idle during I/O. WebClient from Spring WebFlux is non-blocking — the calling thread is released back to the pool while waiting for the response, and a callback processes the response when it arrives. This is significantly more efficient under high concurrency. With RestTemplate, thread pool exhaustion is a real risk when a downstream service is slow. WebClient avoids this entirely. Even in non-reactive applications, I now use WebClient or the newer RestClient from Spring Boot 3.2 as a RestTemplate replacement, since RestTemplate is in maintenance mode. For reactive microservices, WebClient is essential — it returns Mono and Flux types that integrate with the reactive pipeline."

### 💻 Code Example

```java
// ✅ RestTemplate — synchronous (legacy, maintenance mode)
@Configuration
public class RestTemplateConfig {
    @Bean
    public RestTemplate restTemplate(RestTemplateBuilder builder) {
        return builder
            .connectTimeout(Duration.ofSeconds(3))
            .readTimeout(Duration.ofSeconds(5))
            .build();
    }
}

@Service
public class UserServiceClient {
    private final RestTemplate restTemplate;

    public User getUser(Long id) {
        return restTemplate.getForObject(           // ❌ BLOCKS until response
            "http://user-service/api/users/{id}",
            User.class, id);
    }

    public User createUser(UserRequest request) {
        return restTemplate.postForObject(
            "http://user-service/api/users",
            request, User.class);
    }
}

// ✅ WebClient — non-blocking reactive (modern)
@Configuration
public class WebClientConfig {
    @Bean
    public WebClient webClient(WebClient.Builder builder) {
        return builder
            .baseUrl("http://user-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class UserServiceClient {
    private final WebClient webClient;

    // Reactive (non-blocking) — returns Mono
    public Mono<User> getUserReactive(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class);     // ✅ non-blocking
    }

    // Blocking mode — for use in non-reactive apps
    public User getUserBlocking(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .bodyToMono(User.class)
            .block();                    // blocks (like RestTemplate, but better API)
    }

    // Error handling
    public Mono<User> getUserWithErrorHandling(Long id) {
        return webClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .onStatus(HttpStatusCode::is4xxClientError,
                response -> Mono.error(new UserNotFoundException("User not found: " + id)))
            .onStatus(HttpStatusCode::is5xxServerError,
                response -> Mono.error(new ServiceException("User service error")))
            .bodyToMono(User.class)
            .timeout(Duration.ofSeconds(5))
            .retryWhen(Retry.backoff(3, Duration.ofSeconds(1)));
    }

    // Stream response (Flux)
    public Flux<User> getAllUsers() {
        return webClient.get()
            .uri("/api/users")
            .retrieve()
            .bodyToFlux(User.class);   // streaming response
    }
}

// ✅ Spring Boot 3.2+ RestClient — modern synchronous alternative
@Configuration
public class RestClientConfig {
    @Bean
    public RestClient restClient(RestClient.Builder builder) {
        return builder
            .baseUrl("http://user-service")
            .defaultHeader(HttpHeaders.CONTENT_TYPE, MediaType.APPLICATION_JSON_VALUE)
            .build();
    }
}

@Service
public class UserServiceClient {
    private final RestClient restClient;

    public User getUser(Long id) {
        return restClient.get()
            .uri("/api/users/{id}", id)
            .retrieve()
            .body(User.class);           // synchronous but fluent API
    }
}
```

### ⚠️ Common Pitfalls
- **RestTemplate default = NO timeout** → infinite wait → always configure timeouts
- **Calling `.block()` on WebClient in a reactive pipeline** → blocks the event loop → deadlock
- **RestTemplate connection pool** — default uses `SimpleClientHttpRequestFactory` (no pooling) → configure `HttpComponentsClientHttpRequestFactory`
- **WebClient in non-reactive app** — works fine, but consider `RestClient` (Spring Boot 3.2+) for simpler synchronous code

### 🆚 Comparison Table

| Aspect | RestTemplate | WebClient | RestClient (3.2+) |
|--------|-------------|-----------|-------------------|
| I/O Model | Blocking | **Non-blocking** | Blocking |
| Thread usage | 1 thread per request | Event loop (few threads) | 1 thread per request |
| API style | Template methods | Fluent builder | Fluent builder |
| Reactive | ❌ | ✅ Mono/Flux | ❌ |
| Status | **Maintenance mode** | ✅ Active | ✅ Active (new) |
| Use when | Legacy code | Reactive / high concurrency | Synchronous (modern) |
| Connection pool | Manual config | Built-in (Reactor Netty) | Manual config |

### ⚡ Remember (Quick Recall)
- **RestTemplate** = blocking, maintenance mode, don't use for new code
- **WebClient** = non-blocking, reactive, fewer threads needed
- **RestClient** (Spring Boot 3.2+) = modern synchronous replacement for RestTemplate
- WebClient can work in blocking mode (`.block()`) for non-reactive apps
- Always configure **timeouts** on any HTTP client
- WebClient + Resilience4j = production-ready inter-service communication

### 🔗 Follow-up Topics
- [Q1 → Feign Client (declarative alternative)](#q1)
- [Q20 in architecture/01 → Circuit Breaker with Resilience4j](../languages/java/architecture/01-api-design-microservices.md)
- Reactive programming (Mono, Flux, backpressure)
