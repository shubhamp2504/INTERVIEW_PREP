# ☁️ Cloud Infrastructure & Data Processing (Q18–Q21)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q18"></a>
## Q18. Load Balancing (NGINX, Kubernetes, Ribbon)

### 📝 One-Liner
Load balancing distributes incoming traffic across multiple server instances — **server-side** (NGINX, AWS ALB at infrastructure level) or **client-side** (Spring Cloud LoadBalancer in microservices) — using strategies like round-robin, least-connections, or weighted.

### 🔑 Quick Answer
**Server-side LB**: sits between client and servers. Client sends to LB → LB picks a server. Examples: NGINX, HAProxy, AWS ALB/NLB. Client doesn't know about individual servers. **Client-side LB**: client has the server list and picks one itself. Examples: Spring Cloud LoadBalancer (replaced Netflix Ribbon). Used in microservices where services discover each other via registry (Eureka). **Algorithms**: **(1) Round-robin** — cycle through servers in order (default, simple). **(2) Least connections** — route to server with fewest active connections (good for varied request times). **(3) Weighted round-robin** — servers with more capacity get more traffic. **(4) IP hash** — same client IP always goes to same server (sticky sessions). **(5) Random** — surprisingly effective for large clusters. **L4 vs L7**: L4 (transport/TCP level, NLB) vs L7 (application/HTTP level, ALB — can route by URL path, headers). *(Load balancer = traffic ko barabar baanto multiple servers mein)*

### 📖 How It Works
```
Server-Side Load Balancing:
  Client → [Load Balancer] → Server 1
                           → Server 2
                           → Server 3
  Client only knows LB address. LB distributes.

Client-Side Load Balancing (microservices):
  ┌────────────┐    ┌──────────────┐
  │ Order Svc  │───→│ Service      │
  │ (client LB)│    │ Registry     │  (Eureka/Consul)
  └─────┬──────┘    │ Payment Svc: │
        │           │  - 10.0.1.1  │
        │           │  - 10.0.1.2  │
        │           │  - 10.0.1.3  │
        │           └──────────────┘
        │ (picks 10.0.1.2 via round-robin)
        └──────────→ Payment Svc @ 10.0.1.2

Kubernetes Load Balancing:
  External:  Ingress Controller (NGINX) → ClusterIP Service → Pods
  Internal:  Service (kube-proxy iptables) → round-robin across Pods

AWS Load Balancers:
  ALB (L7): HTTP/HTTPS, path-based routing (/api → backend, /web → frontend)
  NLB (L4): TCP/UDP, ultra-low latency, millions of requests/sec
  CLB (Classic): legacy, avoid for new projects
```

### 💻 Code Example
```java
// Spring Cloud LoadBalancer (client-side)
// pom.xml: spring-cloud-starter-loadbalancer

@Configuration
@LoadBalancerClient(name = "payment-service",
    configuration = PaymentLBConfig.class)
public class PaymentLBConfig {
    @Bean
    public ServiceInstanceListSupplier serviceInstanceListSupplier(
            ConfigurableApplicationContext context) {
        return ServiceInstanceListSupplier.builder()
            .withDiscoveryClient()        // get instances from Eureka/Consul
            .withHealthChecks()           // filter unhealthy instances
            .withCaching()                // cache to avoid registry hammering
            .build(context);
    }
}

// Use with WebClient (reactive) or RestTemplate
@Bean
@LoadBalanced // enables client-side LB
public RestTemplate restTemplate() {
    return new RestTemplate();
}

// Call by service name (not IP)
PaymentResponse response = restTemplate.postForObject(
    "http://payment-service/api/payments",  // resolved by LB
    paymentRequest, PaymentResponse.class
);
```

```nginx
# NGINX Server-Side Load Balancing
upstream backend {
    least_conn;                    # algorithm: least connections
    server 10.0.1.1:8080 weight=3; # 3x more traffic
    server 10.0.1.2:8080 weight=1;
    server 10.0.1.3:8080 backup;   # only if others are down
}

server {
    listen 80;
    location /api/ {
        proxy_pass http://backend;
        proxy_next_upstream error timeout; # failover on error
    }
}
```

### 🗣️ Answering Approach
"I use load balancing at multiple levels. At the infrastructure level, AWS ALB handles external traffic with path-based routing — directing API calls to backend services and static content to frontend. Within the microservices mesh, I use Spring Cloud LoadBalancer for client-side load balancing — each service discovers available instances from Eureka and picks one using round-robin. The advantage of client-side LB is no single point of failure and the client can make intelligent choices. In Kubernetes, the Service abstraction provides internal load balancing via kube-proxy iptables rules, distributing traffic across pods. For the algorithm, I typically use round-robin for homogeneous instances and least-connections when request processing times vary significantly."

### ⚠️ Pitfalls / Gotchas
- **Sticky sessions** (IP hash) break horizontal scaling — avoid if possible, use external session store (Redis) *(Sticky session = ek server pe atka rehta hai — scaling nahi hota)*
- **Health checks** are critical — LB must remove unhealthy instances (or traffic goes to dead servers)
- **Connection draining** — when removing an instance, let active requests finish before stopping
- **Client-side LB cache staleness** — registry cache may have stale entries. Use health-check filtering
- **SSL termination** — terminate TLS at LB or at each service? LB termination reduces service CPU but internal traffic is unencrypted

### ⚡ Remember
- **Server-side** = NGINX, ALB (client doesn't know servers)
- **Client-side** = Spring Cloud LoadBalancer (client picks server) *(Client-side LB = service khud decide karta hai kisko call kare)*
- **Algorithms**: round-robin (default), least-connections (varied latency), weighted
- **L4** (TCP) = NLB, fast; **L7** (HTTP) = ALB, smart routing
- Always configure **health checks** + **connection draining**

### 🔗 Follow-ups
- [Q19 → Kubernetes (K8s service load balancing)](#q19)
- Q5 (architecture/03) → Microservices with Spring Cloud (Eureka + LB)

---

<a id="q19"></a>
## Q19. Kubernetes Container Orchestration

### 📝 One-Liner
Kubernetes (K8s) automates **deployment, scaling, and management** of containerized applications — it handles self-healing (restart failed pods), auto-scaling, rolling updates, service discovery, and load balancing across a cluster.

### 🔑 Quick Answer
**Core concepts**: **Pod** (smallest unit, 1+ containers), **Deployment** (desired state: 3 replicas of my app), **Service** (stable network endpoint for a set of pods), **Ingress** (external HTTP routing), **ConfigMap/Secret** (externalized config), **Namespace** (cluster partitioning). **Architecture**: **Control Plane** (API Server, etcd, Scheduler, Controller Manager) + **Worker Nodes** (kubelet, kube-proxy, container runtime). **Self-healing**: liveness probe fails → restart pod; readiness probe fails → remove from service; pod dies → ReplicaSet creates new one. **Scaling**: HPA (Horizontal Pod Autoscaler) scales pods based on CPU/memory/custom metrics. **Deployment strategies**: Rolling update (default, zero-downtime), Blue-Green (two environments, switch traffic), Canary (small % first, then full rollout). *(K8s = containers ka manager — deploy karo, scale karo, heal karo — sab automatic)*

### 📖 How It Works
```
K8s Architecture:
  ┌─────────────────────────────────────────────────┐
  │                 CONTROL PLANE                    │
  │  ┌──────────┐  ┌──────┐  ┌───────────────────┐ │
  │  │API Server │  │ etcd │  │ Scheduler +       │ │
  │  │(kubectl)  │  │(data)│  │ Controller Mgr    │ │
  │  └──────────┘  └──────┘  └───────────────────┘ │
  └─────────────────────┬───────────────────────────┘
                        │
  ┌─────────────────────┴───────────────────────────┐
  │                  WORKER NODES                    │
  │ ┌────────────────┐  ┌────────────────┐          │
  │ │ Node 1         │  │ Node 2         │          │
  │ │ ┌────┐ ┌────┐  │  │ ┌────┐ ┌────┐  │         │
  │ │ │Pod1│ │Pod2│  │  │ │Pod3│ │Pod4│  │         │
  │ │ └────┘ └────┘  │  │ └────┘ └────┘  │         │
  │ │ kubelet        │  │ kubelet        │         │
  │ │ kube-proxy     │  │ kube-proxy     │         │
  │ └────────────────┘  └────────────────┘         │
  └─────────────────────────────────────────────────┘

Self-Healing:
  Desired: 3 replicas | Actual: 2 (Pod3 crashed)
  → ReplicaSet detects → schedules new Pod on available node
  
  Liveness Probe: GET /actuator/health every 10s
  → 3 failures → kubelet kills pod → creates new one
  
  Readiness Probe: GET /actuator/health/readiness
  → Fails → pod removed from Service (no traffic) but NOT killed

Rolling Update:
  v1: [Pod1-v1] [Pod2-v1] [Pod3-v1]
  → Start [Pod4-v2] → ready → remove [Pod1-v1]
  → Start [Pod5-v2] → ready → remove [Pod2-v1]
  → Start [Pod6-v2] → ready → remove [Pod3-v1]
  v2: [Pod4-v2] [Pod5-v2] [Pod6-v2]
  → Zero downtime ✅
```

### 💻 Code Example
```yaml
# Deployment — desired state declaration
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1        # 1 extra pod during update
      maxUnavailable: 0   # zero downtime
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
        - name: order-service
          image: myregistry/order-service:2.1.0
          ports:
            - containerPort: 8080
          resources:
            requests: { cpu: "250m", memory: "512Mi" }
            limits: { cpu: "500m", memory: "1Gi" }
          livenessProbe:
            httpGet:
              path: /actuator/health/liveness
              port: 8080
            initialDelaySeconds: 30
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /actuator/health/readiness
              port: 8080
            initialDelaySeconds: 10
            periodSeconds: 5
          env:
            - name: DB_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: db-secrets
                  key: password

---
# Service — stable network endpoint
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  selector:
    app: order-service
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP  # internal only

---
# HPA — auto-scale based on CPU
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: order-service-hpa
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: order-service
  minReplicas: 3
  maxReplicas: 10
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70  # scale up when CPU > 70%
```

### 🗣️ Answering Approach
"Kubernetes manages the full lifecycle of containerized microservices. I declare the desired state — three replicas of order-service with specific resource limits — and K8s continuously reconciles actual state to match. For zero-downtime deployments, I use rolling updates with maxUnavailable set to zero, so at least three healthy pods serve traffic throughout the deployment. Spring Boot actuator health endpoints serve as liveness and readiness probes — liveness determines if the container needs a restart, readiness controls whether it receives traffic. For auto-scaling, HPA monitors CPU and custom metrics from Prometheus, scaling pods between a min and max range. Secrets management uses K8s Secrets backed by external vault for sensitive configuration."

### ⚠️ Pitfalls / Gotchas
- **No resource limits** → one pod can starve others (OOM kills) *(Resource limits set karo — nahi toh ek pod saara memory kha jaayega)*
- **Liveness = readiness confusion** — liveness: "is the app alive?" (restart if not); readiness: "can it serve traffic?" Don't make liveness depend on external services
- **Graceful shutdown** — pod receives SIGTERM → must finish in-flight requests before `terminationGracePeriodSeconds` expires (default 30s). Spring: `server.shutdown=graceful`
- **Init containers** — use for DB migration, config setup before main container starts
- **PDB (PodDisruptionBudget)** — prevent cluster operations from killing too many pods at once

### ⚡ Remember
- **Deployment** = desired state (replicas, image version, strategy)
- **Service** = stable endpoint (DNS name resolves to pod IPs)
- **Liveness** = restart unhealthy; **Readiness** = stop traffic to unready *(Liveness = zinda hai? Readiness = traffic le sakta hai?)*
- **HPA** = auto-scale pods on CPU/memory/custom metrics
- **Rolling update** = zero-downtime default strategy
- Always set **resource requests + limits**

### 🔗 Follow-ups
- [Q18 → Load Balancing (K8s services as internal LB)](#q18)
- [Q20 → Cloud-Native / Serverless (K8s in cloud context)](#q20)

---

<a id="q20"></a>
## Q20. Cloud-Native & Serverless (AWS, GCP, Azure, Lambda)

### 📝 One-Liner
Cloud-native apps are designed for the cloud from the start (12-factor, containerized, microservices, CI/CD) — **serverless** (AWS Lambda, Azure Functions) removes server management entirely, executing functions on-demand with pay-per-invocation pricing.

### 🔑 Quick Answer
**Cloud-Native principles**: containerized (Docker/K8s), microservices architecture, declarative APIs (K8s manifests, Terraform), designed for failure (self-healing, circuit breakers). **12-Factor App**: codebase in version control, externalized config (env vars), stateless processes, port binding, disposability (fast startup/shutdown), dev-prod parity. **Serverless (FaaS)**: write a function → cloud runs it on demand → scales to zero when idle → pay only for execution time. **AWS Lambda**: 15min max execution, 10GB memory, triggered by API Gateway, S3, SQS, EventBridge. **Managed services**: RDS (managed DB), SQS/SNS (messaging), S3 (object storage), DynamoDB (NoSQL), ElastiCache (Redis). **IaC**: Terraform or AWS CDK to define infrastructure as code. *(Serverless = server ki chinta nahi — code likho, deploy karo, auto-scale hota hai — idle pe zero cost)*

### 📖 How It Works
```
Cloud-Native Architecture:
  ┌──────────────────────────────────────────────┐
  │                   AWS Cloud                   │
  │                                               │
  │  Route53 → CloudFront → ALB → EKS (K8s)     │
  │              (CDN)      (LB)    ├── Order Svc │
  │                                 ├── Payment   │
  │                                 └── Inventory │
  │                                               │
  │  RDS (PostgreSQL)  ElastiCache  SQS  S3      │
  │  (managed DB)      (Redis)    (queue)(files)  │
  └──────────────────────────────────────────────┘

Serverless (AWS Lambda):
  Event (API request, S3 upload, SQS message)
    ↓
  AWS Lambda Function (your code)
    ↓ (auto-scales: 0 to thousands of instances)
  Response / Side effect

  Pricing: $0.20 per 1M requests + $0.0000166667 per GB-second
  → Idle = $0 (vs. EC2/K8s always running)

12-Factor App Key Points:
  1.  Codebase: one repo per service (Git)
  2.  Dependencies: explicitly declared (pom.xml, package.json)
  3.  Config: environment variables (not hardcoded)
  4.  Backing services: treat DB, cache, queue as attached resources
  5.  Build/Release/Run: strict separation
  6.  Processes: stateless (store state in Redis/DB, not memory)
  7.  Port binding: self-contained (Spring Boot embedded Tomcat)
  8.  Concurrency: scale via processes (horizontal, not threads)
  9.  Disposability: fast startup, graceful shutdown
  10. Dev/Prod parity: keep environments similar
  11. Logs: treat as event streams (stdout → ELK)
  12. Admin processes: one-off tasks (DB migration as K8s Job)
```

### 💻 Code Example
```java
// AWS Lambda (Java) — triggered by API Gateway
public class OrderHandler implements RequestHandler<APIGatewayProxyRequestEvent,
                                                    APIGatewayProxyResponseEvent> {
    private final ObjectMapper mapper = new ObjectMapper();
    private final DynamoDbClient dynamoDb = DynamoDbClient.create();

    @Override
    public APIGatewayProxyResponseEvent handleRequest(
            APIGatewayProxyRequestEvent event, Context context) {
        context.getLogger().log("Processing order request");

        OrderRequest request = mapper.readValue(event.getBody(), OrderRequest.class);
        
        // Save to DynamoDB
        dynamoDb.putItem(PutItemRequest.builder()
            .tableName("Orders")
            .item(Map.of(
                "orderId", AttributeValue.fromS(UUID.randomUUID().toString()),
                "amount", AttributeValue.fromN(request.amount().toString()),
                "status", AttributeValue.fromS("CREATED")
            ))
            .build());

        return new APIGatewayProxyResponseEvent()
            .withStatusCode(201)
            .withBody("{\"status\": \"created\"}");
    }
}

// Spring Cloud Function — portable serverless
// Works on AWS Lambda, Azure Functions, GCP Cloud Functions
@Bean
public Function<OrderRequest, OrderResponse> processOrder() {
    return request -> {
        Order order = orderService.create(request);
        return new OrderResponse(order.getId(), "CREATED");
    };
}
```

```hcl
# Terraform — Infrastructure as Code
resource "aws_lambda_function" "order_handler" {
  function_name = "order-handler"
  runtime       = "java21"
  handler       = "com.example.OrderHandler::handleRequest"
  memory_size   = 512
  timeout       = 30
  
  environment {
    variables = {
      TABLE_NAME = aws_dynamodb_table.orders.name
    }
  }
}

resource "aws_dynamodb_table" "orders" {
  name         = "Orders"
  billing_mode = "PAY_PER_REQUEST"   # serverless pricing
  hash_key     = "orderId"
  
  attribute {
    name = "orderId"
    type = "S"
  }
}
```

### 🗣️ Answering Approach
"I design cloud-native applications following the 12-factor methodology — stateless services in containers orchestrated by Kubernetes on EKS, with externalized configuration via ConfigMaps and Secrets, and all infrastructure defined as code using Terraform. For event-driven workloads with variable traffic, I use AWS Lambda — it scales from zero to thousands of concurrent executions automatically, and we only pay for actual compute time. For a recent project, we used Lambda for processing S3 file uploads and SQS messages, with Spring Cloud Function for portability across cloud providers. The key design decision is choosing between containers on K8s for long-running services with steady traffic, versus serverless for event-driven, bursty workloads where cost optimization matters."

### ⚠️ Pitfalls / Gotchas
- **Cold start** — Lambda cold start can be 1-10s for Java (use SnapStart or GraalVM native image) *(Java Lambda ka cold start slow hai — SnapStart ya GraalVM use karo)*
- **Vendor lock-in** — heavily using AWS-specific services makes multi-cloud hard. Use Spring Cloud Function for portability
- **State management** — serverless functions are stateless. Use DynamoDB/Redis for state, not in-memory
- **15-min limit** — Lambda max execution time. Long tasks need Step Functions or batch processing
- **Terraform state** — store state file in S3 + DynamoDB lock (not locally)

### ⚡ Remember
- **Cloud-native** = 12-factor + containers + microservices + CI/CD
- **Serverless** = event-driven, auto-scale to zero, pay-per-use
- **Cold start** = Java Lambda problem → SnapStart / GraalVM
- **IaC** = Terraform or CDK (version-controlled infrastructure)
- K8s for **steady traffic**; Lambda for **bursty/event-driven**

### 🔗 Follow-ups
- [Q19 → Kubernetes (container orchestration)](#q19)
- [Q21 → Distributed Data Processing (Spark/Flink on cloud)](#q21)

---

<a id="q21"></a>
## Q21. Distributed Data Processing (Spark, Flink)

### 📝 One-Liner
Apache Spark processes **massive datasets in parallel** across a cluster using in-memory computation (batch + micro-batch streaming); Apache Flink provides **true real-time stream processing** with event-time semantics and exactly-once guarantees.

### 🔑 Quick Answer
**Apache Spark**: distributed processing engine. **RDD** (Resilient Distributed Dataset) → **DataFrame/Dataset** (structured, optimized). Lazy evaluation — transformations build a DAG, actions trigger execution. **Spark SQL** for structured queries, **Spark Streaming** (micro-batch: process every 1-5s), **MLlib** for ML. Architecture: Driver (master) → Executors (workers). Data partitioned across executors. **Apache Flink**: true streaming engine (record-by-record, not micro-batch). Supports event-time processing (handle out-of-order events with watermarks), exactly-once state via checkpoints/savepoints. **When to use**: Spark for batch ETL, data warehouse queries, ML pipelines. Flink for real-time event processing, fraud detection, real-time dashboards. *(Spark = big data batch + micro-batch; Flink = true real-time streaming — dono cluster pe parallel chalte hain)*

### 📖 How It Works
```
Spark Architecture:
  ┌──────────────┐
  │ Spark Driver │ → creates execution plan (DAG)
  │ (your code)  │ → distributes tasks to executors
  └──────┬───────┘
  ┌──────┴──────┐  ┌────────────┐  ┌────────────┐
  │ Executor 1  │  │ Executor 2 │  │ Executor 3 │
  │ Partition 1 │  │ Partition 2│  │ Partition 3│
  │ (in-memory) │  │ (in-memory)│  │ (in-memory)│
  └─────────────┘  └────────────┘  └────────────┘
  
  Lazy Evaluation:
    val df = spark.read.csv("data.csv")     // no execution
      .filter(col("age") > 25)               // no execution (transformation)
      .groupBy("city").count()               // no execution (transformation)
      .collect()                              // ACTION → triggers execution!
  
  DAG: Read → Filter → Shuffle → Aggregate → Collect

Flink Architecture (true streaming):
  Source → [Operator 1] → [Operator 2] → Sink
  
  Each record processed immediately (not batched)
  
  Event-Time vs Processing-Time:
    Event arrives at 10:05 but happened at 10:01
    Event-time processing assigns it to 10:01 window ✅
    Processing-time assigns it to 10:05 window ❌
  
  Exactly-Once with Checkpoints:
    State periodically saved to durable storage (S3/HDFS)
    On failure → restore from last checkpoint → replay from Kafka offset
    → No data loss, no duplicates
```

### 💻 Code Example
```java
// Apache Spark — batch processing (Java)
SparkSession spark = SparkSession.builder()
    .appName("OrderAnalytics")
    .master("yarn") // or "local[*]" for development
    .getOrCreate();

Dataset<Row> orders = spark.read()
    .option("header", "true")
    .csv("s3://data-lake/orders/");

Dataset<Row> topCities = orders
    .filter(col("status").equalTo("COMPLETED"))
    .groupBy(col("city"))
    .agg(
        count("*").alias("order_count"),
        sum("amount").alias("total_revenue")
    )
    .orderBy(col("total_revenue").desc())
    .limit(10);

topCities.write()
    .mode(SaveMode.Overwrite)
    .parquet("s3://analytics/top-cities/");

// Apache Flink — real-time stream processing
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.enableCheckpointing(60000); // checkpoint every 60s

DataStream<String> kafkaStream = env.fromSource(
    KafkaSource.<String>builder()
        .setBootstrapServers("kafka:9092")
        .setTopics("order-events")
        .setGroupId("fraud-detection")
        .setValueOnlyDeserializer(new SimpleStringSchema())
        .build(),
    WatermarkStrategy.forBoundedOutOfOrderness(Duration.ofSeconds(5)),
    "Kafka Source"
);

kafkaStream
    .map(json -> parseOrderEvent(json))
    .keyBy(OrderEvent::getUserId)
    .window(TumblingEventTimeWindows.of(Time.minutes(5)))
    .process(new FraudDetectionFunction()) // flag if >10 orders in 5 min
    .addSink(new AlertSink()); // send to alerting system

env.execute("Fraud Detection Pipeline");
```

### 🆚 vs. Comparison
| Aspect | Apache Spark | Apache Flink |
|--------|-------------|-------------|
| Processing model | Micro-batch ⭐ (batch-first) | True streaming ⭐ (stream-first) |
| Latency | Seconds (micro-batch interval) | Milliseconds ⭐ |
| State management | Limited | Rich (keyed state, windows) ⭐ |
| Exactly-once | At-least-once (default) | Exactly-once ⭐ (checkpoints) |
| Event-time | Supported (Structured Streaming) | First-class support ⭐ |
| Batch processing | Excellent ⭐ (native) | Supported (batch as bounded stream) |
| ML support | MLlib ⭐ | Limited |
| Ecosystem | Largest ⭐ (Hadoop, Hive, etc.) | Growing |
| Best for | Batch ETL, analytics, ML | Real-time events, fraud, IoT ⭐ |

### ⚡ Remember
- **Spark** = batch-first, in-memory, DataFrame/SQL, micro-batch streaming *(Spark = big data ka Swiss knife — batch, SQL, ML sab)*
- **Flink** = stream-first, true real-time, event-time, exactly-once
- Spark: lazy evaluation → transformations build DAG → action triggers
- Flink: per-record processing, checkpointed state, watermarks for out-of-order
- Spark for **ETL/analytics/ML**; Flink for **real-time event processing**

### 🔗 Follow-ups
- [Q20 → Cloud-Native (running Spark/Flink on cloud)](#q20)
- Q9 (architecture/03) → Event-Driven Kafka (source for Flink)
