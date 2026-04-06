# 📊 Observability & Alerting (Q15–Q17)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q15"></a>
## Q15. Logging & Distributed Tracing (ELK, Jaeger, Zipkin)

### 📝 One-Liner
Centralized logging aggregates logs from all services into one searchable place (ELK Stack); distributed tracing tracks a **single request across multiple microservices** with a unique trace ID (Jaeger/Zipkin) — both are essential for debugging in distributed systems.

### 🔑 Quick Answer
**Centralized Logging**: each microservice emits logs → aggregated into central system → searchable and visualizable. **ELK Stack**: **E**lasticsearch (store + search) + **L**ogstash (ingest + transform) + **K**ibana (visualize + dashboards). Alternative: **EFK** (Fluentd replaces Logstash, lighter). **Structured logging**: JSON format with consistent fields (timestamp, service, traceId, level, message) — enables easy filtering. **Distributed Tracing**: adds a **traceId** (entire request chain) + **spanId** (each service hop) to every request. Visualizes the call chain with timing. **Jaeger** (Uber, CNCF): production-grade, OpenTelemetry native. **Zipkin** (Twitter): simpler, lighter. **OpenTelemetry**: vendor-neutral instrumentation SDK — standard for traces, metrics, logs. **Spring integration**: Micrometer Tracing (formerly Spring Cloud Sleuth) auto-injects traceId/spanId. *(Distributed tracing = ek request kai services se guzarti hai — trace ID se sab connect karo)*

### 📖 How It Works
```
ELK Stack:
  ┌──────────┐  ┌──────────┐  ┌──────────┐
  │ OrderSvc │  │PaymentSvc│  │ ShipSvc  │
  │  (logs)  │  │  (logs)  │  │  (logs)  │
  └────┬─────┘  └────┬─────┘  └────┬─────┘
       └──────────────┴──────────────┘
                      ↓ (Filebeat/Fluentd)
              ┌──────────────┐
              │   Logstash   │ → parse, transform, enrich
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │Elasticsearch │ → index, store, search
              └──────┬───────┘
                     ↓
              ┌──────────────┐
              │   Kibana     │ → dashboards, alerts, queries
              └──────────────┘

Distributed Tracing (Jaeger):
  Client → API Gateway → Order Service → Payment Service → DB
  
  traceId: abc-123 (same across ALL services)
  
  API Gateway:   spanId=1, parentId=none   [──────────────────────────]
  Order Service: spanId=2, parentId=1           [──────────────]
  Payment Svc:   spanId=3, parentId=2                [────────]
  DB Call:       spanId=4, parentId=3                   [───]
  
  → Visualizes: which service took how long
  → Identifies: bottleneck is Payment Service (slow span)

Structured Log (JSON):
  {
    "timestamp": "2024-01-15T10:30:45.123Z",
    "level": "ERROR",
    "service": "order-service",
    "traceId": "abc-123",
    "spanId": "span-2",
    "message": "Payment failed for order ORD-456",
    "exception": "PaymentDeclinedException",
    "userId": "user-789"
  }
```

### 💻 Code Example
```java
// Spring Boot — Structured Logging + Tracing (Micrometer Tracing)
// pom.xml dependencies:
// spring-boot-starter-actuator
// micrometer-tracing-bridge-otel (OpenTelemetry bridge)
// opentelemetry-exporter-zipkin (or jaeger)

// application.yml
management:
  tracing:
    sampling:
      probability: 1.0  # sample 100% in dev; lower in prod (0.1 = 10%)
logging:
  pattern:
    level: "%5p [${spring.application.name},%X{traceId},%X{spanId}]"
  # Output: INFO [order-service,abc123,span456] Processing order

// Logback JSON format (logback-spring.xml)
// <encoder class="net.logstash.logback.encoder.LogstashEncoder"/>

// The traceId is automatically propagated:
@RestController
public class OrderController {
    private static final Logger log = LoggerFactory.getLogger(OrderController.class);

    @GetMapping("/orders/{id}")
    public Order getOrder(@PathVariable String id) {
        log.info("Fetching order {}", id); 
        // Log automatically includes traceId, spanId from MDC
        // → searchable in Kibana: traceId:"abc-123"
        return orderService.findById(id);
    }
}

// Custom span for detailed tracing
@Service
public class PaymentService {
    @Autowired private ObservationRegistry observationRegistry;

    public PaymentResult processPayment(Order order) {
        return Observation.createNotStarted("payment.process", observationRegistry)
            .lowCardinalityKeyValue("payment.method", order.getPaymentMethod())
            .observe(() -> {
                // This creates a child span visible in Jaeger
                return paymentGateway.charge(order);
            });
    }
}
```

### 🗣️ Answering Approach
"For observability in microservices, I implement centralized logging with the ELK stack and distributed tracing with OpenTelemetry exported to Jaeger. Every log entry is structured JSON with consistent fields including traceId and spanId — automatically injected by Micrometer Tracing. When a request enters the system, a unique traceId is generated and propagated through all service calls via HTTP headers. This lets me search for a single traceId in Kibana and see all logs across all services for that request, then switch to Jaeger to visualize the timing waterfall and identify which service is the bottleneck. In production, I sample at 10% to control storage costs, but use 100% sampling for error traces. The switch from Spring Cloud Sleuth to Micrometer Tracing gives us vendor-neutral OpenTelemetry compatibility."

### ⚠️ Pitfalls / Gotchas
- **Log volume** can explode in microservices — use appropriate log levels and sampling *(Har cheez DEBUG mein log mat karo — ELK ka disk bhar jaayega)*
- **Trace context propagation** — if you use async threads, manually propagate traceId via MDC or use instrumented executors
- **Sampling rate** — 100% tracing in prod creates massive overhead. Use 10-20% or head-based sampling
- **Correlation across async** — Kafka messages need to carry traceId in headers for end-to-end tracing
- **PII in logs** — never log passwords, tokens, SSN. Use log masking

### ⚡ Remember
- **ELK** = Elasticsearch + Logstash + Kibana (centralized logs)
- **Distributed Tracing** = traceId (request chain) + spanId (each hop) *(traceId = ek request ke saare services ko jodne wala sutra)*
- **OpenTelemetry** = vendor-neutral standard (traces + metrics + logs)
- **Micrometer Tracing** = auto-injects traceId/spanId in Spring Boot
- JSON structured logs → searchable in Kibana
- Sample at 10-20% in production to control cost

### 🔗 Follow-ups
- [Q16 → Monitoring & Metrics (quantitative: Prometheus/Grafana)](#q16)
- [Q17 → Alerting Systems (act on logs and metrics)](#q17)

---

<a id="q16"></a>
## Q16. Monitoring & Metrics (Prometheus, Grafana, Micrometer)

### 📝 One-Liner
Prometheus **scrapes and stores time-series metrics** (CPU, request count, latency percentiles), Grafana **visualizes** them in dashboards, and Micrometer is the **Java instrumentation library** that exposes metrics from Spring Boot applications.

### 🔑 Quick Answer
**Micrometer**: facade for metrics instrumentation in Java (like SLF4J for logging). Supports Prometheus, Datadog, CloudWatch, InfluxDB. Provides: counters (total requests), gauges (current value like active threads), timers (request duration with percentiles), histograms (distribution). **Prometheus**: pull-based monitoring — scrapes `/actuator/prometheus` endpoint periodically. Stores in time-series DB. PromQL for querying (e.g., `rate(http_requests_total[5m])`). **Grafana**: visualization layer — connects to Prometheus, creates dashboards with graphs, tables, alerts. **Key metrics (RED method)**: **R**ate (requests/sec), **E**rrors (error rate %), **D**uration (latency p50/p95/p99). **USE method** for infrastructure: **U**tilization, **S**aturation, **E**rrors. *(Prometheus = metrics collect karo; Grafana = dikhao; Micrometer = Java mein instrument karo)*

### 📖 How It Works
```
Metrics Pipeline:
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │ Spring Boot  │  │ Spring Boot  │  │ Spring Boot  │
  │ + Micrometer │  │ + Micrometer │  │ + Micrometer │
  │ /prometheus  │  │ /prometheus  │  │ /prometheus  │
  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
         └─────────────────┼─────────────────┘
                    ┌──────┴───────┐
                    │  Prometheus  │ ← scrapes every 15s
                    │  (TSDB)     │ ← stores time-series
                    │  (PromQL)   │ ← query language
                    └──────┬───────┘
                    ┌──────┴───────┐
                    │   Grafana    │ ← dashboards
                    │              │ ← alerts → Slack/PagerDuty
                    └──────────────┘

Metric Types:
  COUNTER (monotonically increasing):
    http_requests_total{method="GET", status="200"} = 15234
    → Use rate() to get requests/second

  GAUGE (current value, goes up and down):
    jvm_threads_live = 45
    system_cpu_usage = 0.73

  TIMER/HISTOGRAM (duration distribution):
    http_server_requests_seconds{quantile="0.5"}  = 0.012  (p50: 12ms)
    http_server_requests_seconds{quantile="0.95"} = 0.089  (p95: 89ms)
    http_server_requests_seconds{quantile="0.99"} = 0.234  (p99: 234ms)

PromQL examples:
  rate(http_requests_total[5m])              → requests/sec over 5 min
  histogram_quantile(0.99, rate(...[5m]))    → p99 latency
  sum(rate(http_requests_total{status=~"5.."}[5m]))
    / sum(rate(http_requests_total[5m]))     → error rate %
```

### 💻 Code Example
```java
// Spring Boot + Micrometer + Prometheus
// pom.xml: spring-boot-starter-actuator, micrometer-registry-prometheus

// application.yml
management:
  endpoints:
    web:
      exposure:
        include: health,prometheus,metrics
  metrics:
    tags:
      application: order-service  # tag all metrics

// Custom business metrics
@Service
public class OrderService {
    private final Counter orderCounter;
    private final Timer orderProcessingTimer;
    private final AtomicInteger activeOrders;

    public OrderService(MeterRegistry registry) {
        this.orderCounter = Counter.builder("orders.created.total")
            .description("Total orders created")
            .tag("type", "online")
            .register(registry);

        this.orderProcessingTimer = Timer.builder("orders.processing.duration")
            .description("Order processing time")
            .publishPercentiles(0.5, 0.95, 0.99) // p50, p95, p99
            .register(registry);

        this.activeOrders = registry.gauge("orders.active",
            new AtomicInteger(0));
    }

    public Order createOrder(OrderRequest request) {
        activeOrders.incrementAndGet();
        try {
            return orderProcessingTimer.record(() -> {
                Order order = processOrder(request);
                orderCounter.increment();
                return order;
            });
        } finally {
            activeOrders.decrementAndGet();
        }
    }
}

// @Timed annotation (simpler)
@Timed(value = "payment.process", percentiles = {0.5, 0.95, 0.99})
public PaymentResult processPayment(PaymentRequest request) {
    return paymentGateway.charge(request);
}
```

### 🗣️ Answering Approach
"I instrument Spring Boot services with Micrometer, exposing metrics to Prometheus via the actuator endpoint. Micrometer is like SLF4J for metrics — a facade that supports multiple backends. I track the RED metrics — Rate, Errors, and Duration — for every service endpoint. For business metrics, I create custom counters for order counts, timers with p50/p95/p99 percentiles for processing duration, and gauges for active connections. Prometheus scrapes these endpoints every 15 seconds, stores them as time-series data, and I query with PromQL to detect anomalies. Grafana dashboards visualize these metrics with graphs and tables, and I configure alerts on thresholds like p99 latency exceeding 500ms or error rate above 1%."

### ⚠️ Pitfalls / Gotchas
- **High-cardinality labels** — `userId` as a Prometheus label creates millions of time-series → OOM *(userId jaise high-cardinality labels mat lagao — Prometheus crash ho jaayega)*
- **Percentiles** — Prometheus histograms vs client-side percentiles have different accuracy. Pre-computed percentiles can't be aggregated across instances
- **Pull vs Push** — Prometheus pulls (scrapes). For short-lived jobs, use Pushgateway
- **Metric naming** — follow conventions: `<namespace>_<name>_<unit>` (e.g., `http_request_duration_seconds`)
- **Dashboard sprawl** — too many dashboards = nobody looks. Keep RED dashboard per service

### ⚡ Remember
- **Micrometer** = Java metrics facade (Counter, Gauge, Timer)
- **Prometheus** = pull-based TSDB + PromQL
- **Grafana** = visualization + alerting dashboards
- **RED** = Rate + Errors + Duration (service metrics)
- **USE** = Utilization + Saturation + Errors (infra metrics)
- Avoid high-cardinality labels (userId ❌, status ✅)

### 🔗 Follow-ups
- [Q15 → Logging & Tracing (qualitative debugging)](#q15)
- [Q17 → Alerting Systems (alert on metrics)](#q17)

---

<a id="q17"></a>
## Q17. Alerting Systems

### 📝 One-Liner
Alerting notifies the right people about the right problems at the right time — using **thresholds on metrics** (Prometheus Alertmanager), **anomaly detection**, and **escalation policies** (PagerDuty/OpsGenie) with proper routing to avoid alert fatigue.

### 🔑 Quick Answer
**Alert pipeline**: Metrics source (Prometheus) → Alert rules (conditions) → Alertmanager (routing, grouping, silencing) → Notification channels (Slack, PagerDuty, email). **Good alerts**: actionable (someone can fix it), based on **symptoms** not causes (alert on "error rate > 5%" not "CPU > 80%"), with runbooks linked. **SLOs (Service Level Objectives)**: target availability (e.g., 99.9% uptime = 43min downtime/month). Alert when **error budget** is burning too fast. **Alert levels**: P1/Critical (pages on-call, service down), P2/Warning (Slack notification, degraded), P3/Info (dashboard only, investigate later). **Anti-pattern**: alerting on every metric → alert fatigue → alerts get ignored. *(Alert = sahi waqt pe sahi insaan ko sahi problem batao — bahut zyada alert = sab ignore)*

### 📖 How It Works
```
Alert Pipeline:
  Prometheus ──evaluates rules──→ Alertmanager ──routes──→ Channels
                                     │
                                     ├── Group alerts (avoid spam)
                                     ├── Throttle (don't repeat every 15s)
                                     ├── Silence (during maintenance)
                                     └── Escalate (if unacknowledged → next on-call)

  ┌─────────────────────────────┐
  │ Prometheus Alert Rule       │
  │                             │
  │ - alert: HighErrorRate      │
  │   expr: rate(http_requests  │
  │   _total{status=~"5.."}    │
  │   [5m]) / rate(http_reqs   │
  │   _total[5m]) > 0.05       │
  │   for: 5m                  │ ← must be true for 5min (avoid flapping)
  │   labels:                  │
  │     severity: critical     │
  │   annotations:             │
  │     summary: "Error rate   │
  │     >5% for order-service" │
  │     runbook: "link/to/doc" │
  └─────────────────────────────┘

Alertmanager routing:
  routes:
    - match: { severity: critical }
      receiver: pagerduty-oncall    ← pages the person on-call
    - match: { severity: warning }
      receiver: slack-engineering   ← Slack channel notification

SLO-based Alerting (modern approach):
  SLO: 99.9% availability = 0.1% error budget
  
  Error budget: 43 minutes/month of downtime allowed
  
  Burn rate alert:
    If consuming error budget 10x faster than normal
    → Page immediately (will exhaust budget in ~4 hours)
    
    If consuming 2x faster
    → Slack warning (will exhaust budget in 2 weeks)
```

### 💻 Code Example
```yaml
# Prometheus alerting rules (prometheus-rules.yml)
groups:
  - name: service-alerts
    rules:
      - alert: HighErrorRate
        expr: |
          sum(rate(http_server_requests_seconds_count{status=~"5.."}[5m]))
          / sum(rate(http_server_requests_seconds_count[5m])) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "Error rate > 5% on {{ $labels.application }}"
          runbook_url: "https://wiki.internal/runbooks/high-error-rate"
        
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            sum(rate(http_server_requests_seconds_bucket[5m])) by (le, application)
          ) > 0.5
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "P99 latency > 500ms on {{ $labels.application }}"

      - alert: PodCrashLooping
        expr: |
          rate(kube_pod_container_status_restarts_total[15m]) * 60 * 15 > 3
        labels:
          severity: critical
        annotations:
          summary: "Pod {{ $labels.pod }} restarting > 3 times in 15min"

# Alertmanager config (alertmanager.yml)
route:
  group_by: ['alertname', 'application']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 4h        # don't spam same alert every 15s
  receiver: slack-default
  routes:
    - match:
        severity: critical
      receiver: pagerduty-oncall
    - match:
        severity: warning
      receiver: slack-engineering

receivers:
  - name: pagerduty-oncall
    pagerduty_configs:
      - service_key: '<pagerduty-key>'
  - name: slack-engineering
    slack_configs:
      - channel: '#engineering-alerts'
        text: '{{ .CommonAnnotations.summary }}'
```

### 🗣️ Answering Approach
"I design alerting around symptoms rather than causes — I alert on 'error rate exceeds 5%' rather than 'CPU above 80%' because high CPU might be normal during a batch job. Every alert must be actionable with a linked runbook explaining what to check and how to remediate. I use Prometheus Alertmanager for routing — critical alerts page the on-call engineer via PagerDuty, warnings go to Slack. I configure the `for` duration to avoid flapping — the condition must persist for 5 minutes before firing. Modern SLO-based alerting is even better: I define an error budget based on our SLO, and alert when we're burning through the budget faster than expected. This directly ties alerts to business impact. The biggest anti-pattern is alert fatigue — too many noisy alerts means critical ones get ignored."

### ⚠️ Pitfalls / Gotchas
- **Alert fatigue** = too many non-actionable alerts → people ignore ALL alerts *(Bahut zyada alerts = sab ignore — har alert actionable hona chahiye)*
- **Flapping** = alert fires/resolves/fires rapidly → use `for` duration (5m+)
- **Missing `for` clause** → every momentary spike triggers alert
- **No runbook** → on-call gets alert at 3 AM with no idea what to do
- **Alert on symptoms, not causes** — CPU/memory alerts alone aren't useful without user-facing impact correlation

### ⚡ Remember
- **Symptom-based** alerts (error rate, latency) > cause-based (CPU, memory)
- **Every alert needs a runbook** — what to check, how to fix
- **`for` duration** = avoid flapping (5m minimum)
- **SLO + error budget** = modern alerting approach
- **Group + throttle** = prevent alert spam (Alertmanager)
- Critical → PagerDuty; Warning → Slack; Info → Dashboard only

### 🔗 Follow-ups
- [Q16 → Monitoring & Metrics (data source for alerts)](#q16)
- [Q15 → Logging & Tracing (investigate after alert fires)](#q15)
