# 🏗️ System Design & Distributed Architecture (Q1, Q5, Q9, Q12)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. How would you design scalable and fault-tolerant Java applications for high-traffic enterprise systems?

### 📝 One-Liner
Design with stateless services + horizontal scaling + load balancing + circuit breakers + caching + async messaging + database sharding — no single point of failure.

### 🔑 Quick Answer
**Scalability**: Horizontal scaling (add more instances, not bigger hardware) behind a load balancer. Stateless services (no session state in JVM — store in Redis). Async processing (Kafka/RabbitMQ for non-blocking operations). Caching (Redis L1/L2) to reduce DB load. DB read replicas + connection pooling. **Fault tolerance**: Circuit breakers (Resilience4j) to prevent cascading failures. Retry with exponential backoff. Health checks + auto-restart (K8s liveness/readiness probes). Graceful degradation (return cached/default data when service is down). Multi-AZ deployment for infrastructure redundancy. **Key principle**: design for failure — assume every component CAN fail, and handle it gracefully. *(Har component fail ho sakta hai — isliye stateless rakho, horizontally scale karo, aur failure handle karo)*

### 📖 How It Works
```
High-Traffic Enterprise Architecture:

                    ┌─────────────┐
                    │   CDN       │ ← static assets cached at edge
                    └──────┬──────┘
                           ↓
                    ┌─────────────┐
                    │ API Gateway │ ← rate limiting, auth, routing
                    │ (Kong/Nginx)│
                    └──────┬──────┘
                           ↓
              ┌────────────┼────────────┐
              ↓            ↓            ↓
        ┌──────────┐ ┌──────────┐ ┌──────────┐
        │ Service A│ │ Service A│ │ Service A│  ← horizontal scaling
        │ (Pod 1)  │ │ (Pod 2)  │ │ (Pod 3)  │     (stateless)
        └────┬─────┘ └────┬─────┘ └────┬─────┘
             │             │             │
             └──────┬──────┘─────────────┘
                    ↓
        ┌───────────────────┐     ┌──────────┐
        │   Redis Cache     │     │  Kafka   │ ← async processing
        │ (sessions + data) │     │  Broker  │
        └───────────────────┘     └──────────┘
                    ↓                    ↓
        ┌───────────────────┐     ┌──────────────┐
        │  DB Master        │     │ Consumer Pods │
        │  (writes)         │     │ (background)  │
        ├───────────────────┤     └──────────────┘
        │  DB Replica 1     │ ← read scaling
        │  DB Replica 2     │
        └───────────────────┘

Scalability Strategies:
  1. Stateless services → any instance handles any request
  2. Horizontal scaling → add pods, not bigger servers
  3. Caching → Redis absorbs 90%+ read traffic
  4. Async → Kafka decouples heavy processing
  5. DB read replicas → scale reads independently
  6. Sharding → partition data across DB clusters

Fault Tolerance Strategies:
  1. Circuit breaker → stop calling failed services
  2. Retry + backoff → handle transient failures
  3. Health checks → K8s auto-restarts unhealthy pods
  4. Graceful degradation → serve cached/default when dependencies fail
  5. Multi-AZ deployment → survive datacenter failure
  6. Chaos engineering → proactively test failure scenarios
```

### 🗣️ Answering Approach
"I design for scalability by making services stateless — no in-memory session state, everything stored in Redis or DB — so any instance can handle any request and I can horizontally scale by adding pods. I put a load balancer with API gateway in front for rate limiting and routing. For high throughput, I use Redis caching to absorb most read traffic and Kafka for async processing of non-critical operations like notifications and analytics. For fault tolerance, I use Resilience4j circuit breakers so that when a downstream service fails, we fail fast instead of cascading the failure. Each service has health check endpoints that Kubernetes monitors for automatic restarts. I design for graceful degradation — if the recommendation service is down, the product page still works with a cached or default recommendation. In my project, this architecture handled 10K requests per second with p99 under 100ms, surviving individual service failures without any user-visible impact."

### 💻 Code
```java
// Stateless service — no session state in JVM
@RestController
@RequiredArgsConstructor
public class OrderController {
    private final OrderService orderService;
    private final RedisTemplate<String, Object> redisTemplate;
    
    @PostMapping("/api/orders")
    public ResponseEntity<OrderDTO> createOrder(
            @RequestHeader("X-Request-Id") String requestId,  // idempotency
            @RequestBody @Valid CreateOrderRequest request) {
        
        // Idempotency check — prevents duplicate orders on retry
        String idempotencyKey = "order:idempotent:" + requestId;
        Boolean isNew = redisTemplate.opsForValue()
                .setIfAbsent(idempotencyKey, "processing", Duration.ofHours(24));
        if (Boolean.FALSE.equals(isNew)) {
            return ResponseEntity.status(HttpStatus.CONFLICT).build();
        }
        
        OrderDTO order = orderService.createOrder(request);
        return ResponseEntity.status(HttpStatus.CREATED).body(order);
    }
}

// Resilience4j — circuit breaker + retry + timeout
@Service
@RequiredArgsConstructor
public class PaymentService {
    private final PaymentClient paymentClient;
    
    @CircuitBreaker(name = "payment", fallbackMethod = "paymentFallback")
    @Retry(name = "payment")
    @TimeLimiter(name = "payment")
    public CompletableFuture<PaymentResult> processPayment(PaymentRequest req) {
        return CompletableFuture.supplyAsync(() -> paymentClient.charge(req));
    }
    
    private CompletableFuture<PaymentResult> paymentFallback(
            PaymentRequest req, Throwable ex) {
        // Queue for later processing instead of failing
        kafkaTemplate.send("payment-retry", req);
        return CompletableFuture.completedFuture(
            PaymentResult.pending("Queued for processing"));
    }
}

// Kubernetes health checks via Actuator
// application.yml:
// management:
//   endpoint:
//     health:
//       show-details: always
//   health:
//     circuitbreakers:
//       enabled: true

@Component
public class ExternalServiceHealthIndicator implements HealthIndicator {
    @Override
    public Health health() {
        // Check critical dependencies
        boolean dbUp = checkDatabase();
        boolean redisUp = checkRedis();
        if (dbUp && redisUp) return Health.up().build();
        return Health.down().withDetail("db", dbUp).withDetail("redis", redisUp).build();
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Stateful services** = scaling nightmare. Session stored in JVM → sticky sessions needed → uneven load *(JVM mein session store kiya toh scale karna mushkil hoga — Redis mein rakho)*
- **Over-scaling without profiling** — sometimes the bottleneck is DB, not app servers. Profile first
- **Circuit breaker not tested** — if you never test failure scenarios, breakers may be misconfigured
- **Caching without invalidation strategy** → stale data in production
- **Synchronous inter-service calls** create tight coupling — prefer async (Kafka) for non-blocking ops

### 🆚 vs. Comparison
| Aspect | Vertical Scaling | Horizontal Scaling |
|--------|-----------------|-------------------|
| Approach | Bigger server (more CPU/RAM) | More servers (add instances) |
| Cost | Expensive, hardware limit | Cheaper, linear scaling ⭐ |
| SPOF | Single machine = SPOF | No SPOF (redundancy) ⭐ |
| Complexity | Simple | Requires stateless design |
| Downtime | Needs restart | Zero-downtime rolling deploys ⭐ |

### ⚡ Remember
- **Stateless** services + Redis for session/state *(Stateless = koi bhi pod, koi bhi request handle kare)*
- **Horizontal scaling** > vertical scaling (add pods, not bigger servers)
- **Circuit breaker** + retry + timeout = fault tolerance trinity
- **Async (Kafka)** for non-critical paths, **sync (REST)** for critical
- **Graceful degradation** — serve something, never crash entirely
- Design for failure: assume every component CAN fail

### 🔗 Follow-ups
- [Q5 → Microservices with Spring Cloud](#q5)
- [Q9 → Event-driven architecture](#q9)
- [Q12 → Distributed transactions](#q12)
- Q20 → Fault tolerance patterns (architecture/01)

---

<a id="q5"></a>
## Q5. How would you design a microservices architecture using Spring Boot and Spring Cloud?

### 📝 One-Liner
Spring Boot for individual services + Spring Cloud for inter-service concerns: Eureka (discovery), Gateway (routing), Config Server (centralized config), OpenFeign (declarative REST), Resilience4j (fault tolerance), Sleuth+Zipkin (tracing).

### 🔑 Quick Answer
**Spring Boot** = framework for building each individual microservice (auto-config, embedded server, Actuator). **Spring Cloud** = toolkit for microservice infrastructure: **(1)** **Service Discovery** (Eureka/Consul) — services register themselves, clients find them by name. **(2)** **API Gateway** (Spring Cloud Gateway) — single entry point, routing, rate limiting, auth. **(3)** **Config Server** (Spring Cloud Config) — centralized configuration (Git-backed), hot-reload. **(4)** **Inter-service communication** (OpenFeign) — declarative REST clients. **(5)** **Fault tolerance** (Resilience4j) — circuit breaker, retry, bulkhead. **(6)** **Distributed tracing** (Micrometer Tracing + Zipkin) — trace requests across services. **(7)** **Load balancing** (Spring Cloud LoadBalancer) — client-side balancing. *(Spring Boot = ek ek service banana; Spring Cloud = services ko connect karna, manage karna)*

### 📖 How It Works
```
Spring Cloud Microservices Architecture:

  Client → API Gateway (Spring Cloud Gateway :8080)
              ├── /api/orders/**  → Order Service (Eureka lookup)
              ├── /api/products/** → Product Service (Eureka lookup)
              ├── /api/users/**   → User Service (Eureka lookup)
              └── Rate limiting, JWT validation, CORS

  ┌──────────────────────────────────────────────────┐
  │              Eureka Server (:8761)                │
  │  ┌─────────────┐ ┌─────────────┐ ┌────────────┐ │
  │  │order-service│ │product-svc  │ │user-service│ │
  │  │ 3 instances │ │ 2 instances │ │ 2 instances│ │
  │  └─────────────┘ └─────────────┘ └────────────┘ │
  └──────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────┐
  │            Config Server (:8888)                  │
  │  Git repo → application.yml per service           │
  │  → /refresh endpoint for hot-reload              │
  └──────────────────────────────────────────────────┘

  ┌──────────────────────────────────────────────────┐
  │        Distributed Tracing (Zipkin :9411)         │
  │  Request-123: Gateway → Order → Product → DB     │
  │  Shows latency breakdown per service              │
  └──────────────────────────────────────────────────┘

Service Communication Flow:
  Order Service needs product details:
    1. OpenFeign client calls "product-service" (logical name)
    2. Spring Cloud LoadBalancer resolves via Eureka → 3 IPs
    3. Picks one instance (round-robin)
    4. Resilience4j wraps call with circuit breaker
    5. Micrometer propagates trace ID in headers
    6. Product Service responds
```

### 🗣️ Answering Approach
"I design microservices with Spring Boot for each service and Spring Cloud for cross-cutting infrastructure. Each service registers with Eureka for service discovery — so services find each other by name, not hardcoded URLs. The API Gateway built with Spring Cloud Gateway provides a single entry point, handling routing, rate limiting, and JWT validation. Configuration is centralized via Spring Cloud Config backed by a Git repository, with hot-reload capability. For inter-service calls, I use OpenFeign for declarative REST clients wrapped with Resilience4j circuit breakers. Distributed tracing with Micrometer and Zipkin lets me trace a single request across all services to diagnose latency. In my project, this architecture had 8 microservices with Spring Cloud, and the Gateway handled 5K requests per second with P99 under 150ms."

### 💻 Code
```java
// 1. EUREKA SERVER
@SpringBootApplication
@EnableEurekaServer
public class EurekaServerApplication {
    public static void main(String[] args) {
        SpringApplication.run(EurekaServerApplication.class, args);
    }
}
// application.yml:
// server.port: 8761
// eureka.client.register-with-eureka: false
// eureka.client.fetch-registry: false

// 2. SERVICE REGISTRATION (each microservice)
// application.yml:
// spring.application.name: order-service
// eureka.client.service-url.defaultZone: http://eureka:8761/eureka

// 3. API GATEWAY (Spring Cloud Gateway)
@SpringBootApplication
public class GatewayApplication { }
// application.yml:
// spring:
//   cloud:
//     gateway:
//       routes:
//         - id: order-service
//           uri: lb://order-service        # Eureka lookup + load balancing
//           predicates:
//             - Path=/api/orders/**
//           filters:
//             - name: CircuitBreaker
//               args:
//                 name: orderCB
//                 fallbackUri: forward:/fallback/orders
//             - name: RequestRateLimiter
//               args:
//                 redis-rate-limiter.replenishRate: 100
//                 redis-rate-limiter.burstCapacity: 200

// 4. OPENFEIGN CLIENT (declarative inter-service calls)
@FeignClient(name = "product-service", fallbackFactory = ProductFallbackFactory.class)
public interface ProductClient {
    @GetMapping("/api/products/{id}")
    ProductDTO getProduct(@PathVariable Long id);
    
    @GetMapping("/api/products/batch")
    List<ProductDTO> getProducts(@RequestParam List<Long> ids);
}

@Component
public class ProductFallbackFactory implements FallbackFactory<ProductClient> {
    @Override
    public ProductClient create(Throwable cause) {
        return new ProductClient() {
            public ProductDTO getProduct(Long id) {
                return ProductDTO.defaultProduct(id);  // cached/default
            }
            public List<ProductDTO> getProducts(List<Long> ids) {
                return List.of();  // empty list as fallback
            }
        };
    }
}

// 5. CONFIG SERVER
@SpringBootApplication
@EnableConfigServer
public class ConfigServerApplication { }
// application.yml:
// spring.cloud.config.server.git.uri: https://github.com/company/config-repo
// spring.cloud.config.server.git.default-label: main

// 6. ORDER SERVICE — using all Spring Cloud features
@SpringBootApplication
@EnableDiscoveryClient
@EnableFeignClients
public class OrderServiceApplication { }

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final ProductClient productClient;   // Feign + Eureka + LoadBalancer
    private final KafkaTemplate<String, OrderEvent> kafka;
    
    @Transactional
    public OrderDTO createOrder(CreateOrderRequest req) {
        // Feign call → product-service (discovered via Eureka)
        ProductDTO product = productClient.getProduct(req.productId());
        
        Order order = orderRepo.save(new Order(req.customerId(),
                product.id(), product.price(), req.quantity()));
        
        // Async event for downstream services
        kafka.send("order-events", new OrderEvent(order.getId(), "CREATED"));
        
        return toDTO(order, product);
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Eureka in production** — many teams now use Kubernetes Service Discovery instead of Eureka. K8s DNS handles service lookup natively *(K8s mein Eureka ki zaroorat nahi — K8s khud service discovery karta hai)*
- **Config Server single point of failure** — use replicas + Git retry. Or use K8s ConfigMaps/Secrets
- **Feign without timeout** → thread hanging forever. Always set connect + read timeouts
- **Too many microservices** — don't split prematurely. Start with modular monolith, extract services when needed
- **Distributed debugging is hard** — tracing + centralized logging (ELK) are mandatory, not optional

### 🆚 vs. Comparison
| Spring Cloud Component | Purpose | K8s Alternative |
|----------------------|---------|----------------|
| Eureka | Service Discovery | K8s DNS + Service |
| Spring Cloud Gateway | API Gateway | Ingress (Nginx/Istio) |
| Config Server | Centralized Config | ConfigMaps + Secrets |
| Resilience4j | Circuit Breaker | Istio (service mesh) |
| Sleuth + Zipkin | Tracing | Jaeger / OpenTelemetry |
| LoadBalancer | Client-side LB | K8s Service (kube-proxy) |

### ⚡ Remember
- **Spring Boot** = build each service / **Spring Cloud** = connect services together
- Eureka (discovery) + Gateway (routing) + Config (centralized) + Feign (communication) *(Eureka se dhundho, Gateway se route karo, Config se manage karo, Feign se baat karo)*
- In K8s: most Spring Cloud components have native alternatives
- Always add: **tracing** (Zipkin), **circuit breakers** (Resilience4j), **centralized logging** (ELK)
- Start as modular monolith → extract services as needed

### 🔗 Follow-ups
- [Q1 → Scalable system design (overall architecture)](#q1)
- [Q9 → Event-driven architecture (Kafka integration)](#q9)
- [Q12 → Distributed transactions (Saga pattern)](#q12)
- Q20 → Fault tolerance (architecture/01)

---

<a id="q9"></a>
## Q9. How do you design event-driven architectures using technologies such as Kafka or RabbitMQ?

### 📝 One-Liner
Event-driven architecture: services communicate by producing and consuming events through a broker (Kafka for high-throughput streaming, RabbitMQ for flexible routing) — achieving loose coupling, scalability, and resilience.

### 🔑 Quick Answer
**Event-driven architecture (EDA)**: instead of services calling each other directly (REST), they produce events (facts about what happened) to a message broker. Other services subscribe and react independently. **Kafka**: distributed log, ordered, persistent, high throughput (millions/sec), consumer groups for parallel processing, exactly-once semantics. Best for: event streaming, audit logs, data pipelines, high-volume systems. **RabbitMQ**: traditional message broker, flexible routing (exchange → binding → queue), acknowledgment-based, message TTL. Best for: task queues, RPC, complex routing patterns, smaller scale. **Key patterns**: Event Sourcing (store events as source of truth), CQRS (separate read/write models), Transactional Outbox (atomic DB + event publishing), Saga (distributed transactions via events). *(Event-driven = service apna kaam kare, event publish kare — dusri services apne aap react karein)*

### 📖 How It Works
```
Event-Driven Architecture:

  Order Service                   Kafka/RabbitMQ                  Consumers
  ┌──────────┐    OrderCreated    ┌──────────┐    ┌─────────────────────┐
  │ POST     │───────────────────>│          │───>│ Inventory Service   │
  │ /orders  │    event           │  Broker  │    │ (deduct stock)      │
  │          │                    │          │───>│ Payment Service     │
  └──────────┘                    │          │    │ (charge customer)   │
                                  │          │───>│ Notification Service│
  Payment Service                 │          │    │ (send email/SMS)    │
  ┌──────────┐    PaymentDone     │          │───>│ Analytics Service   │
  │ Process  │───────────────────>│          │    │ (update dashboard)  │
  │ payment  │    event           └──────────┘    └─────────────────────┘
  └──────────┘

Kafka Topic Architecture:
  Topic: order-events (6 partitions, 3 replicas)
  ┌─────────────────────────────────────────────┐
  │ P0: [order-1][order-4][order-7] ...         │ → Consumer Group A: Inventory
  │ P1: [order-2][order-5][order-8] ...         │ → Consumer Group B: Payment
  │ P2: [order-3][order-6][order-9] ...         │ → Consumer Group C: Email
  │ P3: ...                                      │
  │ P4: ...                                      │
  │ P5: ...                                      │
  └─────────────────────────────────────────────┘
  - Partition key = orderId → all events for same order go to same partition → ordering!
  - Each consumer group gets ALL events independently
  - Within a group, partitions are distributed → parallel processing

Transactional Outbox Pattern (atomic DB + event):
  1. Save order + outbox event in SAME DB transaction
  2. Background poller reads outbox table → publishes to Kafka
  3. Marks outbox entry as published
  → Guarantees: if order is saved, event WILL be published
```

### 🗣️ Answering Approach
"I design event-driven architectures by having services publish domain events to a broker when something significant happens — OrderCreated, PaymentProcessed, InventoryUpdated. I choose Kafka for high-throughput scenarios — it handles millions of events per second with guaranteed ordering within partitions — and RabbitMQ when I need flexible routing patterns like fanout or topic-based routing. The key benefit is loose coupling: the order service publishes an OrderCreated event without knowing who consumes it. Adding a new consumer like analytics doesn't require changing the order service at all. For data consistency, I use the Transactional Outbox pattern — the event is stored in an outbox table within the same database transaction as the business data, then a separate process publishes it to Kafka. This solves the dual-write problem. For distributed transactions across services, I implement the Saga pattern with compensating events."

### 💻 Code
```java
// KAFKA — Event Producer with Transactional Outbox
@Entity
@Table(name = "outbox_events")
public class OutboxEvent {
    @Id @GeneratedValue
    private Long id;
    private String aggregateId;
    private String eventType;
    @Column(columnDefinition = "TEXT")
    private String payload;      // JSON serialized event
    private boolean published;
    @CreationTimestamp
    private Instant createdAt;
}

@Service
@RequiredArgsConstructor
public class OrderService {
    private final OrderRepository orderRepo;
    private final OutboxRepository outboxRepo;
    
    @Transactional  // SAME transaction: order + outbox event
    public OrderDTO createOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(toEntity(req));
        
        // Save event to outbox table (not directly to Kafka!)
        OutboxEvent event = new OutboxEvent();
        event.setAggregateId(order.getId().toString());
        event.setEventType("OrderCreated");
        event.setPayload(toJson(new OrderCreatedEvent(
                order.getId(), order.getCustomerId(), order.getTotal())));
        outboxRepo.save(event);
        
        return toDTO(order);
    }
}

// Outbox publisher — reads unpublished events and sends to Kafka
@Component
@RequiredArgsConstructor
@Slf4j
public class OutboxPublisher {
    private final OutboxRepository outboxRepo;
    private final KafkaTemplate<String, String> kafkaTemplate;
    
    @Scheduled(fixedDelay = 1000)  // every second
    @Transactional
    public void publishPendingEvents() {
        List<OutboxEvent> events = outboxRepo.findByPublishedFalse();
        for (OutboxEvent event : events) {
            kafkaTemplate.send("order-events", event.getAggregateId(), event.getPayload());
            event.setPublished(true);
            outboxRepo.save(event);
        }
    }
}

// KAFKA — Consumer with idempotency
@Component
@RequiredArgsConstructor
@Slf4j
public class InventoryConsumer {
    private final InventoryService inventoryService;
    private final ProcessedEventRepository processedRepo;
    
    @KafkaListener(topics = "order-events", groupId = "inventory-service")
    public void handleOrderEvent(ConsumerRecord<String, String> record) {
        String eventId = record.key() + ":" + record.offset();
        
        // Idempotency check — skip if already processed
        if (processedRepo.existsById(eventId)) {
            log.info("Skipping duplicate event: {}", eventId);
            return;
        }
        
        OrderCreatedEvent event = fromJson(record.value(), OrderCreatedEvent.class);
        inventoryService.reserveStock(event.orderId(), event.items());
        
        processedRepo.save(new ProcessedEvent(eventId, Instant.now()));
    }
}

// RABBITMQ — exchange + routing example
@Configuration
public class RabbitConfig {
    @Bean
    public TopicExchange orderExchange() {
        return new TopicExchange("order-exchange");
    }
    
    @Bean
    public Queue emailQueue() { return new Queue("email-queue"); }
    
    @Bean
    public Queue inventoryQueue() { return new Queue("inventory-queue"); }
    
    @Bean
    public Binding emailBinding() {
        return BindingBuilder.bind(emailQueue())
                .to(orderExchange())
                .with("order.created.*");  // routing key pattern
    }
    
    @Bean
    public Binding inventoryBinding() {
        return BindingBuilder.bind(inventoryQueue())
                .to(orderExchange())
                .with("order.#");  // all order events
    }
}

@Service
public class OrderEventPublisher {
    @Autowired private RabbitTemplate rabbitTemplate;
    
    public void publishOrderCreated(Order order) {
        rabbitTemplate.convertAndSend("order-exchange",
                "order.created." + order.getRegion(),  // routing key
                new OrderCreatedEvent(order));
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **Dual write problem** — writing to DB + Kafka is NOT atomic. If Kafka publish fails after DB commit, event is lost. Use Transactional Outbox or CDC (Debezium) *(DB aur Kafka mein ek saath likhna atomic nahi hai — Outbox pattern use karo)*
- **Consumer idempotency** — Kafka delivers at-least-once. Consumer MUST handle duplicates
- **Message ordering** — Kafka only guarantees order within partition. Use same key for related events
- **Consumer lag** — if consumer is slower than producer, monitor lag. Scale consumer instances (max = partition count)
- **Event schema evolution** — add fields as optional, never remove. Use Avro + Schema Registry for strict contracts

### 🆚 vs. Comparison
| Aspect | Kafka | RabbitMQ |
|--------|-------|----------|
| Model | Distributed log (append-only) | Message broker (queue-based) |
| Throughput | Very high (millions/sec) ⭐ | Moderate (tens of thousands) |
| Ordering | Per-partition ⭐ | Per-queue |
| Retention | Configurable (days/weeks) ⭐ | Until consumed |
| Replay | Yes (re-read old events) ⭐ | No (consumed = gone) |
| Routing | Topic + partition key | Exchange + routing key + bindings ⭐ |
| Use case | Event streaming, data pipelines | Task queues, RPC, complex routing |
| Complexity | Higher (ZooKeeper/KRaft) | Lower (simpler setup) |

### ⚡ Remember
- **EDA** = produce events → broker → consumers react independently
- **Kafka** for high-throughput streaming; **RabbitMQ** for flexible routing *(Kafka = bada data, RabbitMQ = complex routing)*
- **Transactional Outbox** = solve dual-write (DB + event atomically)
- **Idempotent consumers** = at-least-once delivery means duplicates possible
- **Saga pattern** = distributed transactions via compensating events
- Same partition key → ordering guaranteed

### 🔗 Follow-ups
- [Q1 → Scalable system design](#q1)
- [Q12 → Distributed transactions (Saga with events)](#q12)
- Q18 → REST vs Kafka (architecture/01)

---

<a id="q12"></a>
## Q12. What are the best practices for handling distributed transactions and data consistency in microservices?

### 📝 One-Liner
No distributed ACID — use Saga pattern (choreography or orchestration) with compensating transactions, eventual consistency, idempotent operations, and the Transactional Outbox pattern.

### 🔑 Quick Answer
In monoliths, one `@Transactional` covers everything. In microservices, each service has its own database — **no shared transaction**. Options: **(1) Saga Pattern**: break transaction into local steps, each with a compensating action for rollback. **Choreography**: services listen to events and react (decoupled, simpler). **Orchestration**: a central orchestrator directs the flow (easier to track, more control). **(2) Transactional Outbox**: atomic local DB write + event publish. **(3) Eventual Consistency**: accept that data won't be immediately consistent across services — design for it. **(4) Idempotency**: every operation must handle being called multiple times safely. **(5) Two-phase commit (2PC)**: rarely used in microservices — too slow, tight coupling, blocks resources. *(Microservices mein ek @Transactional se kaam nahi chalega — Saga pattern se har step ka ulta step bhi define karo)*

### 📖 How It Works
```
Saga Pattern — Order Placement Example:

  HAPPY PATH:
  Order Service          Payment Service       Inventory Service
       │                       │                      │
       │ 1. Create Order       │                      │
       │ (status: PENDING)     │                      │
       ├─── OrderCreated ─────>│                      │
       │                       │ 2. Charge Payment    │
       │                       │ (status: CHARGED)    │
       │                       ├── PaymentDone ──────>│
       │                       │                      │ 3. Reserve Stock
       │                       │                      │ (status: RESERVED)
       │<──────────── StockReserved ──────────────────│
       │ 4. Confirm Order      │                      │
       │ (status: CONFIRMED)   │                      │

  FAILURE PATH (Inventory fails):
  Order Service          Payment Service       Inventory Service
       │                       │                      │
       │ 1. Create Order ✅    │                      │
       ├─── OrderCreated ─────>│                      │
       │                       │ 2. Charge Payment ✅ │
       │                       ├── PaymentDone ──────>│
       │                       │                      │ 3. Reserve Stock ❌
       │                       │                      │ (OUT OF STOCK!)
       │                       │<── StockFailed ──────│
       │                       │ 4. COMPENSATE:       │
       │                       │ Refund Payment 💰    │
       │<── PaymentRefunded ───│                      │
       │ 5. COMPENSATE:        │                      │
       │ Cancel Order ❌       │                      │

Choreography vs Orchestration:

  CHOREOGRAPHY (event-driven, decentralized):
    Each service → listens to events → does its work → publishes next event
    ✅ Loose coupling, simple for 2-3 services
    ❌ Hard to track flow, scattered logic

  ORCHESTRATION (central coordinator):
    Saga Orchestrator → tells each service what to do → waits for response
    ✅ Clear flow, easy to monitor and debug
    ❌ Single coordinator = potential bottleneck
```

### 🗣️ Answering Approach
"In microservices, traditional distributed transactions like two-phase commit don't work well — they're slow and create tight coupling. I use the Saga pattern where each service performs its local transaction and publishes an event. If a downstream step fails, compensating transactions undo previous steps. For example, in an order flow — if inventory reservation fails after payment was charged, a compensating event triggers a payment refund and order cancellation. I choose choreography for simple 2-3 step flows where services react to events independently, and orchestration with a central Saga orchestrator for complex flows involving 5+ services where visibility and error handling are critical. For data consistency between the database and message broker, I use the Transactional Outbox pattern — the event is stored in the same database transaction as the business data. Every consumer operation is idempotent so that retries don't cause duplicate processing."

### 💻 Code
```java
// SAGA ORCHESTRATOR — controls the workflow
@Service
@RequiredArgsConstructor
@Slf4j
public class OrderSagaOrchestrator {
    private final OrderService orderService;
    private final PaymentClient paymentClient;
    private final InventoryClient inventoryClient;
    
    public OrderResult executeOrderSaga(CreateOrderRequest req) {
        String orderId = null;
        String paymentId = null;
        
        try {
            // Step 1: Create order (PENDING)
            orderId = orderService.createPendingOrder(req);
            
            // Step 2: Charge payment
            paymentId = paymentClient.charge(orderId, req.amount());
            
            // Step 3: Reserve inventory
            inventoryClient.reserve(orderId, req.items());
            
            // All steps succeeded → confirm order
            orderService.confirmOrder(orderId);
            return OrderResult.success(orderId);
            
        } catch (PaymentFailedException e) {
            // Compensate Step 1
            log.warn("Payment failed, cancelling order {}", orderId);
            if (orderId != null) orderService.cancelOrder(orderId);
            return OrderResult.failed("Payment failed: " + e.getMessage());
            
        } catch (InventoryException e) {
            // Compensate Step 2 + Step 1
            log.warn("Inventory failed, refunding payment and cancelling order {}", orderId);
            if (paymentId != null) paymentClient.refund(paymentId);
            if (orderId != null) orderService.cancelOrder(orderId);
            return OrderResult.failed("Inventory unavailable: " + e.getMessage());
        }
    }
}

// SAGA CHOREOGRAPHY — event-driven with Kafka
// Each service listens and reacts independently

// Order Service → publishes OrderCreated
@Service
public class OrderService {
    @Transactional
    public OrderDTO createOrder(CreateOrderRequest req) {
        Order order = orderRepo.save(new Order(req, OrderStatus.PENDING));
        outboxRepo.save(new OutboxEvent("OrderCreated", order.getId(), toJson(order)));
        return toDTO(order);
    }
    
    // Listen for saga completion/failure
    @KafkaListener(topics = "stock-events", groupId = "order-service")
    public void handleStockEvent(StockEvent event) {
        if (event.type().equals("StockReserved")) {
            orderRepo.updateStatus(event.orderId(), OrderStatus.CONFIRMED);
        } else if (event.type().equals("StockFailed")) {
            orderRepo.updateStatus(event.orderId(), OrderStatus.CANCELLED);
        }
    }
}

// Payment Service → listens to OrderCreated, publishes PaymentDone/PaymentFailed
@Component
public class PaymentConsumer {
    @KafkaListener(topics = "order-events", groupId = "payment-service")
    public void handleOrderCreated(OrderCreatedEvent event) {
        try {
            paymentService.charge(event.orderId(), event.amount());
            kafkaTemplate.send("payment-events",
                new PaymentEvent("PaymentDone", event.orderId()));
        } catch (Exception e) {
            kafkaTemplate.send("payment-events",
                new PaymentEvent("PaymentFailed", event.orderId()));
        }
    }
    
    // Compensating transaction: listen for stock failure → refund
    @KafkaListener(topics = "stock-events", groupId = "payment-service")
    public void handleStockFailed(StockEvent event) {
        if (event.type().equals("StockFailed")) {
            paymentService.refund(event.orderId());
            kafkaTemplate.send("payment-events",
                new PaymentEvent("PaymentRefunded", event.orderId()));
        }
    }
}

// Idempotent consumer — handles duplicates safely
@Component
public class InventoryConsumer {
    @KafkaListener(topics = "payment-events", groupId = "inventory-service")
    public void handlePaymentDone(PaymentEvent event) {
        if (!event.type().equals("PaymentDone")) return;
        
        // Idempotency: check if already reserved for this order
        if (inventoryService.isAlreadyReserved(event.orderId())) {
            log.info("Stock already reserved for order {}, skipping", event.orderId());
            return;
        }
        
        try {
            inventoryService.reserveStock(event.orderId());
            kafkaTemplate.send("stock-events",
                new StockEvent("StockReserved", event.orderId()));
        } catch (OutOfStockException e) {
            kafkaTemplate.send("stock-events",
                new StockEvent("StockFailed", event.orderId()));
        }
    }
}
```

### ⚠️ Pitfalls / Gotchas
- **2PC in microservices** — don't do it. It's slow, blocks resources, and ties services together *(2PC = slow + tightly coupled — microservices mein use mat karo)*
- **Compensating transactions are hard** — what if the refund API also fails? Need retry + dead letter queue
- **Eventual consistency confuses users** — order shows "PENDING" briefly. Design UI for this (loading states/spinners)
- **Choreography with many services** → hard to track flow. Switch to orchestration at 5+ services
- **Missing idempotency** → duplicate events = duplicate processing (double charge, double stock deduction)
- **Saga state tracking** — orchestrator needs to persist saga state for recovery after crashes

### 🆚 vs. Comparison
| Aspect | Choreography | Orchestration | 2PC |
|--------|-------------|---------------|-----|
| Control | Decentralized | Central orchestrator | Transaction manager |
| Coupling | Loose ⭐ | Medium | Tight |
| Visibility | Hard to track | Easy to monitor ⭐ | Easy |
| Complexity (few services) | Simple ⭐ | Over-engineered | — |
| Complexity (many services) | Spaghetti events | Clean flow ⭐ | — |
| Performance | Fast (async) ⭐ | Medium | Slow (blocking) |
| Use when | 2-4 services | 5+ services | Legacy/monolith |

### ⚡ Remember
- **No distributed ACID** in microservices — accept eventual consistency
- **Saga** = local transactions + compensating actions *(Saga = har step ka ulta step define karo)*
- **Choreography** (events, decoupled) vs **Orchestration** (coordinator, centralized)
- **Transactional Outbox** = atomic DB + event publish
- **Idempotent consumers** = handle duplicate events safely
- Avoid 2PC — too slow for microservices

### 🔗 Follow-ups
- [Q9 → Event-driven architecture (Outbox pattern)](#q9)
- [Q1 → Scalable architecture (overall design)](#q1)
- Q13 → ACID properties (database/01)
- Q20 → Fault tolerance (architecture/01)

---

<a id="q13"></a>
## Q13. How do you evaluate whether an architecture is truly microservices or a distributed monolith?

### 📝 One-Liner
A **true microservice** can be deployed, scaled, and failed independently — if services must be deployed together, share databases, or break when one goes down, you have a **distributed monolith** with extra network hops.

### 🔑 Quick Answer
**Microservices test**: (1) Can you deploy Service A without redeploying Service B? (2) Does each service own its own database? (3) Can Service A function (even degraded) if Service B is down? (4) Can teams work on services independently without merge conflicts? If the answer to any is "no" — you likely have a distributed monolith. **Common traps**: shared database, synchronous chains (A calls B calls C calls D), tight coupling via shared libraries, coordinated deployments. *(Agar ek service deploy karne ke liye doosri bhi deploy karni padti hai — toh woh microservices nahi hai, distributed monolith hai)*

### 📖 How It Works
```
True Microservices:                    Distributed Monolith:
┌──────────┐  ┌──────────┐           ┌──────────┐  ┌──────────┐
│ Order    │  │ Payment  │           │ Order    │  │ Payment  │
│ Service  │  │ Service  │           │ Service  │──│ Service  │──sync
├──────────┤  ├──────────┤           │          │  │          │  chain
│ Own DB   │  │ Own DB   │           ├──────────┤  ├──────────┤
│ orders   │  │ payments │           │  SHARED DATABASE       │
└──────────┘  └──────────┘           └──────────────────────── ┘
  async events    async events           tight coupling + shared state
  independent     independent            deploy together or nothing works
  can fail alone  can fail alone         cascade failures
```

### 🗣️ Answering Approach
"When someone tells me their architecture is microservices, I ask four questions: Can each service be deployed independently? Does each own its data store? Can one service degrade gracefully if another is down? Can separate teams work without coordinating releases? If any answer is no, it's likely a distributed monolith — you have the complexity of a distributed system without the benefits of microservices. The most common anti-pattern I've seen is a shared database — three services reading from the same tables. Any schema change requires coordinating all three. Another red flag is synchronous call chains — Order → Payment → Inventory → Shipping all synchronous. If Shipping is slow, the entire order flow times out. True microservices communicate via async events, own their data, and use patterns like circuit breakers and bulkheads for fault isolation. I'm not saying every team needs microservices — a well-designed modular monolith is often better than a poorly-designed distributed system."

### 💻 Code Example

```java
// ❌ DISTRIBUTED MONOLITH — synchronous chain, shared models
@RestController
public class OrderController {
    // Order → calls Payment → calls Inventory → calls Shipping
    // If ANY service is down, entire order fails
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest req) {
        PaymentResponse payment = paymentClient.charge(req);    // sync HTTP
        InventoryResponse inv = inventoryClient.reserve(req);    // sync HTTP
        ShippingResponse ship = shippingClient.schedule(req);    // sync HTTP
        return orderRepo.save(new Order(payment, inv, ship));    // shared DB?
    }
}

// ✅ TRUE MICROSERVICES — async events, own database, fault-tolerant
@RestController
public class OrderController {
    @PostMapping("/orders")
    public Order createOrder(@RequestBody OrderRequest req) {
        Order order = orderRepo.save(Order.pending(req));  // own DB
        // Publish event — other services react asynchronously
        kafkaTemplate.send("order-events",
            new OrderCreatedEvent(order.getId(), req.items(), req.customerId()));
        return order;  // returns immediately with PENDING status
    }
}

// Payment service listens independently
@KafkaListener(topics = "order-events")
public void onOrderCreated(OrderCreatedEvent event) {
    paymentService.charge(event);  // own DB, own pace
    kafkaTemplate.send("payment-events", new PaymentProcessedEvent(...));
}

// Circuit breaker for any remaining sync calls
@CircuitBreaker(name = "inventory", fallbackMethod = "fallback")
public InventoryStatus checkStock(String productId) {
    return inventoryClient.check(productId);
}
public InventoryStatus fallback(String productId, Throwable t) {
    return InventoryStatus.UNKNOWN;  // degrade gracefully
}
```

### 🆚 Microservices vs Distributed Monolith

| Aspect | True Microservices | Distributed Monolith |
|--------|-------------------|---------------------|
| **Deployment** | Independent per service | Must deploy together |
| **Database** | Each owns its DB | Shared database |
| **Communication** | Async events (Kafka) | Synchronous HTTP chains |
| **Failure** | Isolated (circuit breaker) | Cascading |
| **Teams** | Independent (Conway's Law) | Must coordinate releases |
| **Shared code** | Minimal (API contracts) | Shared libraries/models |
| **Scaling** | Scale only what's needed | Scale everything or nothing |

### 🎯 Tricky Interview Qs

**Q: Is it always bad to have sync calls between services?**
No — occasional sync calls with circuit breakers are fine. The problem is long synchronous chains (A→B→C→D) where one slow service blocks everything.

**Q: When should you NOT use microservices?**
Small team (< 5-7 devs), early-stage product, simple domain. A modular monolith is simpler to develop, deploy, and debug. Extract microservices only when you hit scaling or team boundaries.

**Q: How do you migrate from distributed monolith to true microservices?**
Strangler Fig pattern: introduce events alongside sync calls, gradually move data ownership, decompose shared DB, replace sync chains with async events one at a time.

### ⚡ Remember
- **Litmus test**: can I deploy, scale, and fail each service independently?
- Shared DB = almost always a distributed monolith
- Sync chains = latency multiplied + cascading failures
- Modular monolith > distributed monolith (simpler, fewer network issues)
