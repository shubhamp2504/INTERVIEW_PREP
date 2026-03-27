# 🏗️ System Design — E-Commerce & Payment Systems (Q1–Q4)

> **Source**: American Express Java Backend Interview (2-4 YOE, March 2026)  
> **Coverage**: Payment gateway, shopping cart, checkout with fraud detection, scalable transaction processing

---

<a id="q1"></a>
## Q1. Design a payment gateway system that handles retries and failures

### 📝 One-Liner
A payment gateway must be **idempotent** (safe to retry), handle **partial failures** with compensating transactions, use **circuit breakers** for downstream PSP calls, and maintain an **audit trail** of every state transition.

### 🔑 Quick Answer
**Core components**: API layer (accepts payment request with idempotency key) → Validation service (amount, card, fraud score) → Payment orchestrator (routes to PSP — Stripe/Razorpay/Adyen) → State machine (INITIATED → PROCESSING → SUCCESS/FAILED/REQUIRES_RETRY) → Notification service (webhook/email). **Retry strategy**: exponential backoff with jitter, max 3 retries, idempotency key ensures the same charge isn't applied twice. **Failure handling**: circuit breaker on PSP calls, fallback to secondary PSP, dead letter queue for manual review. *(Payment gateway = idempotent hona chahiye, retry safe hona chahiye, aur har state change ka audit trail rakhna chahiye)*

### 📖 Architecture

```
Client → API Gateway → Payment Service
                          │
                  ┌───────┴────────┐
                  │  Idempotency   │
                  │  Key Check     │
                  │  (Redis/DB)    │
                  └───────┬────────┘
                          │
                  ┌───────┴────────┐
                  │  Payment       │
                  │  State Machine │
                  │  INITIATED →   │
                  │  PROCESSING →  │
                  │  SUCCESS/FAIL  │
                  └───────┬────────┘
                          │
            ┌─────────────┼─────────────┐
            ▼             ▼             ▼
      ┌──────────┐ ┌──────────┐ ┌──────────┐
      │ Stripe   │ │ Razorpay │ │ Adyen    │  ← PSP (Payment Service Providers)
      │ (Primary)│ │(Fallback)│ │(Fallback)│
      └──────────┘ └──────────┘ └──────────┘
            │                         │
            ▼                         ▼
      Circuit Breaker           Circuit Breaker
      (Resilience4j)            (Resilience4j)
            │
            ▼
      ┌──────────┐    ┌──────────┐
      │ Webhook  │    │ Audit    │
      │ Callback │    │ Log (DB) │
      └──────────┘    └──────────┘
```

### 🗣️ Answering Approach
"I'd design the payment gateway with idempotency as the foundation. Every payment request includes a client-generated idempotency key — before processing, I check Redis for this key. If it exists, I return the cached result without re-charging. The payment follows a state machine: INITIATED → PROCESSING → SUCCESS or FAILED. I persist every state transition to a payment_events table for auditability. For PSP communication, I wrap calls in a circuit breaker — if Stripe is down, the circuit opens and I fail fast or route to a fallback PSP like Razorpay. Retries use exponential backoff with jitter to avoid thundering herd. For async payment confirmations, I use webhooks from the PSP and reconcile against my state. Failed payments that exhaust retries go to a dead letter queue for manual review. The key design principle is that any operation can be safely retried without double-charging the customer."

### 💻 Key Code Patterns

```java
// ✅ Idempotency check
@PostMapping("/payments")
public ResponseEntity<PaymentResponse> processPayment(
        @RequestHeader("Idempotency-Key") String idempotencyKey,
        @Valid @RequestBody PaymentRequest request) {

    // Check if already processed
    Optional<Payment> existing = paymentRepository.findByIdempotencyKey(idempotencyKey);
    if (existing.isPresent()) {
        return ResponseEntity.ok(toResponse(existing.get()));  // return cached result
    }

    Payment payment = paymentService.initiate(request, idempotencyKey);
    return ResponseEntity.status(HttpStatus.CREATED).body(toResponse(payment));
}

// ✅ Retry with exponential backoff
@Retryable(
    value = {PSPTimeoutException.class},
    maxAttempts = 3,
    backoff = @Backoff(delay = 1000, multiplier = 2, maxDelay = 10000)
)
public PaymentResult chargeViaStripe(PaymentRequest req) {
    return stripeClient.charge(req);  // Circuit breaker wraps this too
}

// ✅ State machine transitions
public enum PaymentState {
    INITIATED, PROCESSING, SUCCESS, FAILED, REQUIRES_MANUAL_REVIEW
}

@Entity
public class PaymentEvent {
    @Id @GeneratedValue
    private Long id;
    private String paymentId;
    @Enumerated(EnumType.STRING)
    private PaymentState fromState;
    @Enumerated(EnumType.STRING)
    private PaymentState toState;
    private String reason;
    private LocalDateTime timestamp;
}
```

### ⚡ Key Design Decisions
- **Idempotency key** — prevents double-charge on retries
- **State machine** — every payment has auditable state transitions
- **Circuit breaker** — fail fast when PSP is down, route to fallback
- **Exponential backoff + jitter** — prevents thundering herd on retries
- **Dead letter queue** — poisoned payments don't block the pipeline
- **Event sourcing** — payment_events table = complete audit trail

---

<a id="q2"></a>
## Q2. Design a shopping cart service that supports millions of users

### 📝 One-Liner
Shopping cart needs to be **highly available**, **low-latency** (reads), handle **guest + logged-in users**, support **cart merging** on login, and survive server restarts — use Redis for active carts, DB for persistence.

### 🔑 Quick Answer
**Storage strategy**: **Redis** for active carts (sub-millisecond reads, TTL for abandoned carts), **PostgreSQL/DynamoDB** for persistent carts (logged-in users, cross-device sync). **Guest users**: cart stored in Redis keyed by session ID, migrated to user-cart on login. **Scale**: Redis cluster with sharding by user ID, read replicas for high-read scenarios. **Cart operations**: add item (check inventory), update quantity (validate stock), remove item, apply coupon/promo, calculate totals (prices, tax, discounts). *(Shopping cart = Redis me fast access, DB me persistence, guest cart login pe merge karo)*

### 📖 Architecture

```
                    ┌─────────────┐
                    │   Client    │
                    │   (Web/App) │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │ API Gateway │
                    │ (Rate Limit)│
                    └──────┬──────┘
                           │
                  ┌────────┴────────┐
                  │  Cart Service   │
                  │  (Stateless)    │
                  └───┬─────────┬───┘
                      │         │
              ┌───────┴─┐   ┌──┴──────────┐
              │  Redis  │   │  PostgreSQL  │
              │ (Active │   │ (Persistent  │
              │  Carts) │   │  Cart Data)  │
              │ TTL:7d  │   │              │
              └─────────┘   └──────────────┘
                      │
              ┌───────┴────────┐
              │ Inventory Svc  │ ← check stock on add/checkout
              │ (gRPC/Kafka)   │
              └────────────────┘
```

### 🗣️ Answering Approach
"For a shopping cart serving millions of users, I'd use a two-tier storage approach. Active carts are stored in Redis for sub-millisecond reads — keyed by userId or sessionId for guest users. Redis gives me TTL-based expiry for abandoned carts, typically 7 days. For logged-in users, I also persist the cart to PostgreSQL for cross-device sync and recovery after Redis failures. When a guest logs in, I merge their session cart with their persisted user cart — taking the higher quantity for duplicate items. The cart service itself is stateless and horizontally scalable. On every add-to-cart, I do a lightweight inventory reservation check via an async call to the inventory service. For pricing and promotions, I compute totals server-side to prevent client manipulation. At millions of users, I'd shard Redis by userId hash and use read replicas."

### 💻 Key Code Patterns

```java
// ✅ Redis-backed cart with DB fallback
@Service
public class CartService {
    private final RedisTemplate<String, Cart> redisTemplate;
    private final CartRepository cartRepository;

    private static final Duration CART_TTL = Duration.ofDays(7);

    public Cart getCart(String userId) {
        String key = "cart:" + userId;
        Cart cart = redisTemplate.opsForValue().get(key);
        if (cart == null) {
            // Fallback to DB for logged-in users
            cart = cartRepository.findByUserId(userId)
                .orElse(new Cart(userId));
            redisTemplate.opsForValue().set(key, cart, CART_TTL);
        }
        return cart;
    }

    public Cart addItem(String userId, CartItem item) {
        // Verify inventory
        if (!inventoryService.hasStock(item.getProductId(), item.getQuantity())) {
            throw new InsufficientStockException(item.getProductId());
        }
        Cart cart = getCart(userId);
        cart.addOrUpdateItem(item);
        cart.recalculateTotals(pricingService);
        // Write-through: update both Redis and DB
        redisTemplate.opsForValue().set("cart:" + userId, cart, CART_TTL);
        cartRepository.save(cart.toEntity());  // async for performance
        return cart;
    }

    // ✅ Merge guest cart into user cart on login
    public Cart mergeCarts(String sessionId, String userId) {
        Cart guestCart = getCart("session:" + sessionId);
        Cart userCart = getCart(userId);
        for (CartItem item : guestCart.getItems()) {
            userCart.addOrUpdateItem(item);  // higher quantity wins
        }
        redisTemplate.delete("cart:session:" + sessionId);
        redisTemplate.opsForValue().set("cart:" + userId, userCart, CART_TTL);
        return userCart;
    }
}
```

### ⚡ Key Design Decisions
- **Redis + DB** — fast reads (Redis) + durability (DB)
- **TTL** — auto-expire abandoned carts (7 days)
- **Cart merge** — guest → logged-in on authentication
- **Server-side pricing** — never trust client-sent prices
- **Inventory check** — validate stock on add, re-validate at checkout
- **Horizontal scaling** — stateless service + Redis sharding by userId

---

<a id="q3"></a>
## Q3. Design a secure checkout system with fraud detection

### 📝 One-Liner
Checkout = **cart validation** → **inventory lock** → **fraud scoring** → **payment processing** → **order creation** → **confirmation** — with fraud detection running async on features like velocity, device fingerprint, and anomaly detection.

### 🔑 Quick Answer
**Checkout flow**: (1) Validate cart (prices, stock). (2) Reserve inventory (distributed lock, TTL). (3) Fraud check — real-time scoring based on rules + ML model. (4) If fraud score > threshold → hold for manual review or decline. (5) Process payment (idempotent, PSP call). (6) Create order. (7) Release inventory reservation → confirm allocation. (8) Send confirmation. **Fraud signals**: velocity (5 orders in 1 minute), mismatched shipping/billing address, new account + high-value order, device fingerprint mismatch, IP geolocation anomaly. *(Checkout me fraud detection zaroori hai — velocity, device fingerprint, address mismatch sab check karo payment se pehle)*

### 📖 Architecture

```
Checkout Flow:

Cart → [1] Validate Cart (prices + stock)
         │
         ▼
       [2] Reserve Inventory (distributed lock, 10-min TTL)
         │
         ▼
       [3] Fraud Detection ◄── Rules Engine (velocity, limits)
         │                  ◄── ML Model (anomaly score)
         │                  ◄── Device Fingerprint
         │                  ◄── IP Geolocation
         │
    ┌────┴─────────────┐
    │ Score < 30       │ Score 30-70        │ Score > 70
    │ ✅ Auto-approve  │ ⚠️ Manual review   │ ❌ Auto-decline
    │                  │                    │
    ▼                  ▼                    ▼
  [4] Payment        Hold Queue           Decline + Notify
    │
    ▼
  [5] Create Order
    │
    ▼
  [6] Confirm Inventory + Send Email/SMS
```

### 🗣️ Answering Approach
"The checkout system has two parallel concerns: the transaction flow and fraud detection. For the flow, after the user clicks checkout, I first re-validate the cart — check that prices haven't changed and stock is available. Then I reserve inventory with a distributed lock that has a 10-minute TTL in case the checkout is abandoned. Before processing payment, I run a fraud detection step. This combines a rules engine — checking velocity of orders from the same user, card BIN checks, address mismatch — with an ML model that scores the transaction on a 0-100 scale. Scores below 30 are auto-approved, 30-70 go to manual review queue, above 70 are auto-declined. Only after the fraud check passes do I process the payment through the payment gateway. The entire checkout is idempotent so the user can safely retry if anything times out."

### 💻 Key Code Patterns

```java
// ✅ Checkout orchestrator
@Service
public class CheckoutService {

    @Transactional
    public OrderResult checkout(String userId, CheckoutRequest request) {
        // Step 1: Validate cart
        Cart cart = cartService.getCart(userId);
        cartValidator.validatePricesAndStock(cart);

        // Step 2: Reserve inventory (distributed lock with TTL)
        InventoryReservation reservation = inventoryService.reserve(
            cart.getItems(), Duration.ofMinutes(10));

        try {
            // Step 3: Fraud check
            FraudScore score = fraudService.evaluate(FraudContext.builder()
                .userId(userId)
                .amount(cart.getTotal())
                .shippingAddress(request.shippingAddress())
                .billingAddress(request.billingAddress())
                .deviceFingerprint(request.deviceFingerprint())
                .ipAddress(request.ipAddress())
                .build());

            if (score.isDeclined()) {
                reservation.release();
                throw new FraudDeclinedException("Transaction declined by fraud check");
            }
            if (score.requiresReview()) {
                return OrderResult.pendingReview(reservation.getId());
            }

            // Step 4: Process payment
            PaymentResult payment = paymentService.charge(
                cart.getTotal(), request.paymentMethod(), reservation.getId());

            // Step 5: Create order
            Order order = orderService.create(userId, cart, payment, request);

            // Step 6: Confirm inventory + notify
            reservation.confirm();
            notificationService.sendConfirmation(order);
            cartService.clear(userId);

            return OrderResult.success(order);

        } catch (Exception e) {
            reservation.release();   // compensating action
            throw e;
        }
    }
}

// ✅ Fraud detection — rules + ML
@Service
public class FraudService {

    public FraudScore evaluate(FraudContext ctx) {
        int score = 0;

        // Rule 1: Velocity — too many orders in short time
        int recentOrders = orderRepository.countByUserIdAndTimestampAfter(
            ctx.getUserId(), Instant.now().minus(Duration.ofHours(1)));
        if (recentOrders > 5) score += 30;

        // Rule 2: Address mismatch
        if (!ctx.getShippingAddress().getCountry()
                .equals(ctx.getBillingAddress().getCountry())) {
            score += 20;
        }

        // Rule 3: New account + high value
        if (ctx.getAccountAge().toDays() < 7 && ctx.getAmount().compareTo(HIGH_VALUE) > 0) {
            score += 25;
        }

        // Rule 4: ML model score
        double mlScore = mlFraudModel.predict(ctx.toFeatureVector());
        score += (int) (mlScore * 40);  // ML contributes up to 40 points

        return new FraudScore(score);
    }
}
```

### ⚡ Key Design Decisions
- **Inventory reservation with TTL** — prevents overselling but auto-releases on abandoned checkouts
- **Fraud before payment** — don't charge then refund; check first
- **Score ranges** — auto-approve, manual review, auto-decline
- **Compensating transactions** — if payment fails after inventory lock, release reservation
- **Idempotency throughout** — checkout, payment, inventory all support safe retries

---

<a id="q4"></a>
## Q4. How would you design a scalable transaction processing system?

### 📝 One-Liner
A scalable transaction system uses **event-driven architecture** with partitioned message queues, **idempotent consumers**, **distributed state management**, and **horizontal scaling** — process millions of transactions/sec by parallelizing across partitions.

### 🔑 Quick Answer
**Ingestion layer**: REST API or message queue receives transaction events. **Processing layer**: Kafka partitioned by `transactionId` or `accountId` (ensures ordering per account). **Consumer group** scales horizontally — each consumer handles a subset of partitions. **Processing**: validate → enrich → apply business rules → persist → emit events. **State management**: each transaction follows a state machine (RECEIVED → VALIDATED → PROCESSED → SETTLED). **Consistency**: Saga pattern for multi-step transactions, Transactional Outbox for reliable event publishing. *(Scalable transaction system = Kafka partitions se parallel processing, idempotent consumers, aur Saga pattern se distributed consistency)*

### 📖 Architecture

```
Transaction Processing Pipeline:

Producers (millions TPS)
    │
    ▼
┌──────────────────────────────────────────────┐
│  Kafka (Partitioned by accountId)             │
│  ┌──────┐ ┌──────┐ ┌──────┐ ┌──────┐        │
│  │ P-0  │ │ P-1  │ │ P-2  │ │ P-N  │        │
│  └──┬───┘ └──┬───┘ └──┬───┘ └──┬───┘        │
└─────┼────────┼────────┼────────┼─────────────┘
      │        │        │        │
      ▼        ▼        ▼        ▼
┌──────────┐┌──────────┐┌──────────┐┌──────────┐
│Consumer-1││Consumer-2││Consumer-3││Consumer-N │
│(Validate ││(Validate ││(Validate ││(Validate  │
│ Enrich   ││ Enrich   ││ Enrich   ││ Enrich    │
│ Process) ││ Process) ││ Process) ││ Process)  │
└────┬─────┘└────┬─────┘└────┬─────┘└────┬─────┘
     │           │           │           │
     ▼           ▼           ▼           ▼
┌──────────────────────────────────────────────┐
│  PostgreSQL / Cassandra (Partitioned)         │
│  Transaction State (RECEIVED → SETTLED)       │
│  + Transactional Outbox table                 │
└──────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────┐
│  Downstream Events       │
│  (Notifications, Reports,│
│   Reconciliation)        │
└──────────────────────────┘
```

### 🗣️ Answering Approach
"For a scalable transaction processing system, I'd use an event-driven architecture with Kafka as the backbone. Transactions are published to a Kafka topic partitioned by account ID — this ensures all transactions for the same account are processed in order within one partition, while different accounts process in parallel across partitions. A consumer group with N consumers handles N partitions, and I can scale by adding more partitions and consumers. Each consumer validates the transaction, enriches it with account data, applies business rules like fraud checks and limits, then persists the result and emits downstream events via the Transactional Outbox pattern — this ensures the database write and event publication are atomic. Every consumer is idempotent using the transaction ID as a deduplication key. For multi-step transactions spanning multiple services, I use the Saga pattern with compensating transactions. The system can handle millions of TPS by scaling Kafka partitions and consumer instances horizontally."

### 💻 Key Code Patterns

```java
// ✅ Kafka consumer — idempotent transaction processing
@KafkaListener(topics = "transactions", groupId = "txn-processor")
public void processTransaction(ConsumerRecord<String, TransactionEvent> record) {
    String txnId = record.key();

    // Idempotency check
    if (processedTxnRepository.existsById(txnId)) {
        log.info("Duplicate txn={}, skipping", txnId);
        return;
    }

    TransactionEvent event = record.value();

    // Step 1: Validate
    validationService.validate(event);

    // Step 2: Enrich (account lookup, FX rates)
    EnrichedTransaction enriched = enrichmentService.enrich(event);

    // Step 3: Apply business rules
    RuleResult rules = ruleEngine.evaluate(enriched);
    if (rules.isDeclined()) {
        persistAsDeclined(txnId, enriched, rules.getReason());
        return;
    }

    // Step 4: Persist + Outbox (atomic)
    transactionService.processAndPublish(txnId, enriched);
}

// ✅ Transactional Outbox — atomic write + event
@Service
public class TransactionService {

    @Transactional
    public void processAndPublish(String txnId, EnrichedTransaction txn) {
        // Write to main table
        Transaction entity = new Transaction(txnId, txn, TransactionState.PROCESSED);
        transactionRepository.save(entity);

        // Write to outbox table (same transaction!)
        OutboxEvent outbox = new OutboxEvent(
            "transaction.processed", txnId, toJson(entity));
        outboxRepository.save(outbox);

        // Mark as processed (idempotency)
        processedTxnRepository.save(new ProcessedTxn(txnId));
    }
    // Separate CDC/poller publishes outbox events to Kafka
}

// ✅ Kafka partitioning — ordering per account
@Bean
public NewTopic transactionTopic() {
    return TopicBuilder.name("transactions")
        .partitions(64)     // 64 partitions = up to 64 consumers
        .replicas(3)        // 3x replication for durability
        .build();
}
// Producer: kafkaTemplate.send("transactions", accountId, event);
// accountId as key → same partition → ordered per account
```

### ⚡ Key Design Decisions
- **Kafka partitioned by accountId** — ordering per account, parallelism across accounts
- **Consumer group** — horizontal scaling (add partitions + consumers)
- **Idempotent consumers** — transaction ID dedup prevents double-processing
- **Transactional Outbox** — atomic DB write + event publishing
- **State machine** — every transaction has auditable lifecycle
- **Backpressure** — Kafka naturally handles backpressure via consumer lag

### 🔗 Related Topics
- [Distributed transactions / Saga](../languages/java/architecture/03-system-design-distributed.md)
- [Event Sourcing](02-coordination-failover-eventsourcing.md)
- [Circuit Breaker / Fault Tolerance](../languages/java/architecture/01-api-design-microservices.md)
