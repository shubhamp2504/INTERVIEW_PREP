# 🌐 Networking, Protocols & Security (Q11–Q14)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Answering Approach → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q11"></a>
## Q11. Socket Programming — TCP/IP & UDP

### 📝 One-Liner
TCP gives **reliable, ordered, connection-oriented** communication (HTTP, DB connections); UDP gives **fast, unordered, connectionless** communication (video streaming, DNS, gaming) — Java provides `Socket`/`ServerSocket` for TCP and `DatagramSocket` for UDP.

### 🔑 Quick Answer
**TCP**: 3-way handshake (SYN → SYN-ACK → ACK), guarantees delivery + ordering, flow control + congestion control. Slower but reliable. Used for: HTTP, database connections, file transfer. **UDP**: no handshake, no delivery guarantee, no ordering. Fire-and-forget. Fast + low latency. Used for: video/audio streaming, DNS lookups, online gaming, IoT telemetry. **Java TCP**: `ServerSocket` (server side, accepts connections) → `Socket` (client side + accepted connection). **Java UDP**: `DatagramSocket` + `DatagramPacket` (both sides). **NIO**: `java.nio` channels + selectors for non-blocking I/O (high-concurrency servers). **Netty**: production-grade async event-driven network framework (used by gRPC, Kafka, Elasticsearch). *(TCP = guaranteed delivery, slow; UDP = fast but packets kho sakte hain)*

### 📖 How It Works
```
TCP 3-Way Handshake:
  Client                     Server
    |──── SYN ──────────────→ |  (I want to connect)
    |←── SYN-ACK ────────── | (OK, I acknowledge)
    |──── ACK ──────────────→ |  (Connection established)
    |←──── DATA ─────────── | (Reliable, ordered transfer)
    |──── FIN ──────────────→ |  (Close connection)

UDP (no handshake):
  Client                     Server
    |──── Packet 1 ─────────→ |  (maybe received)
    |──── Packet 2 ─────────→ |  (maybe lost)
    |──── Packet 3 ─────────→ |  (maybe received out of order)
    No ACK, no retransmit, no ordering guarantee

Java NIO (non-blocking):
  Selector (1 thread monitors many channels)
    ├── Channel A: readable → process read
    ├── Channel B: writable → process write
    └── Channel C: new connection → accept
  → One thread handles thousands of connections
  → vs. traditional: one thread per connection (doesn't scale)
```

### 💻 Code Example
```java
// TCP Server (traditional blocking I/O)
try (ServerSocket server = new ServerSocket(8080)) {
    while (true) {
        Socket client = server.accept(); // blocks until connection
        new Thread(() -> handleClient(client)).start();
    }
}

// TCP Client
try (Socket socket = new Socket("localhost", 8080);
     PrintWriter out = new PrintWriter(socket.getOutputStream(), true);
     BufferedReader in = new BufferedReader(new InputStreamReader(socket.getInputStream()))) {
    out.println("Hello Server");
    String response = in.readLine();
}

// UDP Sender
DatagramSocket socket = new DatagramSocket();
byte[] data = "Hello".getBytes(StandardCharsets.UTF_8);
InetAddress address = InetAddress.getByName("localhost");
DatagramPacket packet = new DatagramPacket(data, data.length, address, 9090);
socket.send(packet); // fire and forget

// NIO Non-Blocking Server (high concurrency)
Selector selector = Selector.open();
ServerSocketChannel serverChannel = ServerSocketChannel.open();
serverChannel.bind(new InetSocketAddress(8080));
serverChannel.configureBlocking(false);
serverChannel.register(selector, SelectionKey.OP_ACCEPT);

while (true) {
    selector.select(); // blocks until events ready
    for (SelectionKey key : selector.selectedKeys()) {
        if (key.isAcceptable()) { /* accept new connection */ }
        if (key.isReadable()) { /* read data from channel */ }
    }
    selector.selectedKeys().clear();
}
```

### 🗣️ Answering Approach
"For network programming in Java, the choice between TCP and UDP depends on the reliability requirement. TCP with its three-way handshake guarantees delivery and ordering — I use it for any data that must arrive correctly like API calls and database connections. UDP skips the handshake and has no delivery guarantee — I use it for real-time scenarios like metrics streaming or video where speed matters more than occasional packet loss. For high-concurrency servers, I use Java NIO with selectors — a single thread monitors thousands of connections via non-blocking channels, rather than the traditional thread-per-connection model that doesn't scale beyond a few thousand connections. In production, I use Netty rather than raw NIO, as it provides a production-grade event-driven framework used by Kafka, gRPC, and Elasticsearch."

### 🆚 vs. Comparison
| Aspect | TCP | UDP |
|--------|-----|-----|
| Connection | 3-way handshake | Connectionless ⭐ |
| Reliability | Guaranteed delivery ⭐ | Best-effort |
| Ordering | Preserved ⭐ | Not guaranteed |
| Speed | Slower | Faster ⭐ |
| Header size | 20 bytes | 8 bytes ⭐ |
| Use cases | HTTP, DB, file transfer | Streaming, DNS, gaming |
| Java API | Socket / ServerSocket | DatagramSocket |

### ⚡ Remember
- **TCP** = reliable, ordered, connection-oriented (HTTP, DB, files)
- **UDP** = fast, connectionless, no guarantee (streaming, DNS, gaming)
- **NIO** = non-blocking, selector-based (1 thread → many connections)
- **Netty** = production NIO framework (don't use raw NIO)
- Thread-per-connection doesn't scale → use NIO/Netty for high concurrency

### 🔗 Follow-ups
- [Q12 → RPC / gRPC (network layer abstraction)](#q12)
- [Q14 → SSL/TLS Encryption (securing socket connections)](#q14)

---

<a id="q12"></a>
## Q12. RPC — gRPC, Thrift, RMI

### 📝 One-Liner
RPC (Remote Procedure Call) lets you call a method on a remote server as if it were local — **gRPC** (Google, HTTP/2 + Protobuf, streaming), **Thrift** (Facebook, multi-protocol), **Java RMI** (Java-only, uses Java serialization).

### 🔑 Quick Answer
**RPC concept**: client calls a stub (proxy) → stub serializes request → sends over network → server deserializes → executes → sends response back → client gets return value as if it were a local call. **gRPC**: uses HTTP/2 (multiplexed, bidirectional streaming), Protocol Buffers for serialization (binary, compact, schema-defined), language-agnostic (Java, Go, Python, etc.). Supports 4 patterns: unary, server-streaming, client-streaming, bidirectional-streaming. **Apache Thrift**: Facebook's RPC framework, supports multiple serialization formats (binary, compact, JSON) and transport protocols (TSocket, TFramedTransport). **Java RMI**: Java-only, tight coupling, uses Java serialization (slow, security issues). Legacy — avoid in new projects. **Verdict**: gRPC is the modern standard for service-to-service communication. *(gRPC = modern era ka RPC — Protobuf se fast, HTTP/2 se multiplexed)*

### 📖 How It Works
```
gRPC Flow:
  1. Define service in .proto file (contract/schema)
  2. Generate client stub + server skeleton (protoc compiler)
  3. Client calls stub method → Protobuf serializes → HTTP/2 sends
  4. Server receives → deserializes → executes → sends response

  ┌────────────┐       HTTP/2 + Protobuf       ┌────────────┐
  │   Client   │ ──────────────────────────────→│   Server   │
  │  (Stub)    │ ←──────────────────────────────│ (Skeleton) │
  └────────────┘                                └────────────┘

4 Communication Patterns:
  1. Unary:          Client ──request──→ Server ──response──→ Client
  2. Server Stream:  Client ──request──→ Server ══stream══→ Client
  3. Client Stream:  Client ══stream══→ Server ──response──→ Client
  4. Bidirectional:  Client ══stream══⇆══stream══ Server

HTTP/2 Multiplexing:
  Single TCP connection → multiple concurrent streams
  → No head-of-line blocking (unlike HTTP/1.1)
  → Efficient for microservice-to-microservice calls
```

### 💻 Code Example
```protobuf
// order_service.proto — define the contract
syntax = "proto3";
package order;

service OrderService {
  rpc GetOrder (OrderRequest) returns (OrderResponse);           // unary
  rpc StreamOrders (OrderFilter) returns (stream OrderResponse); // server-streaming
}

message OrderRequest {
  string order_id = 1;
}

message OrderResponse {
  string order_id = 1;
  string status = 2;
  double amount = 3;
}
```

```java
// gRPC Server Implementation
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    @Override
    public void getOrder(OrderRequest request, StreamObserver<OrderResponse> responseObserver) {
        OrderResponse response = OrderResponse.newBuilder()
            .setOrderId(request.getOrderId())
            .setStatus("CONFIRMED")
            .setAmount(99.99)
            .build();
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }

    @Override
    public void streamOrders(OrderFilter filter, StreamObserver<OrderResponse> responseObserver) {
        // Server streaming — send multiple responses
        for (Order order : orderRepository.findByFilter(filter)) {
            responseObserver.onNext(toProto(order));
        }
        responseObserver.onCompleted();
    }
}

// gRPC Client
ManagedChannel channel = ManagedChannelBuilder.forAddress("localhost", 9090)
    .usePlaintext() // for dev; use TLS in production
    .build();
OrderServiceGrpc.OrderServiceBlockingStub stub = OrderServiceGrpc.newBlockingStub(channel);
OrderResponse response = stub.getOrder(
    OrderRequest.newBuilder().setOrderId("ORD-123").build()
);
```

### 🗣️ Answering Approach
"For inter-service communication in microservices, I use gRPC over REST when I need high performance and type safety. gRPC uses Protocol Buffers for binary serialization — which is 5-10x smaller and faster than JSON — and HTTP/2 for multiplexing, which allows multiple concurrent request-response streams over a single TCP connection. The service contract is defined in a .proto file, and the protoc compiler generates strongly-typed client stubs and server skeletons in any language. This eliminates the manual mapping layer needed with REST. I particularly leverage server-streaming for real-time data feeds and bidirectional streaming for scenarios like chat or live dashboards. For browser-facing APIs, I still use REST since gRPC-Web has limited browser support."

### 🆚 vs. Comparison
| Aspect | gRPC | REST (JSON) | Thrift | Java RMI |
|--------|------|------------|--------|----------|
| Serialization | Protobuf (binary) ⭐ | JSON (text) | Binary/Compact | Java serialization |
| Transport | HTTP/2 ⭐ | HTTP/1.1 | TCP/HTTP | JRMP |
| Streaming | Bidirectional ⭐ | SSE (one-way) | Limited | No |
| Schema | .proto (strict) ⭐ | OpenAPI (optional) | .thrift | Java interfaces |
| Language | Any ⭐ | Any ⭐ | Any | Java only ❌ |
| Browser | Limited | Full ⭐ | Limited | No |
| Speed | Fastest ⭐ | Moderate | Fast | Slow |

### ⚡ Remember
- **gRPC** = Protobuf + HTTP/2 → fastest, type-safe, streaming *(gRPC = microservice communication ka king)*
- **Protobuf** = binary serialization (5-10x smaller than JSON)
- **HTTP/2** = multiplexed streams on single TCP connection
- 4 patterns: unary, server-stream, client-stream, bidirectional
- Use **REST for browsers**, **gRPC for service-to-service**

### 🔗 Follow-ups
- [Q13 → GraphQL (another API paradigm)](#q13)
- [Q11 → Socket Programming (underlying transport)](#q11)

---

<a id="q13"></a>
## Q13. GraphQL

### 📝 One-Liner
GraphQL lets clients **request exactly the data they need** in a single query — no over-fetching (getting too many fields) or under-fetching (needing multiple REST calls) — with a strongly-typed schema.

### 🔑 Quick Answer
**Problem with REST**: to get user's orders with product details: `GET /users/1` + `GET /users/1/orders` + `GET /products/123` = 3 round trips (under-fetching). Or one endpoint returns everything including unused fields (over-fetching). **GraphQL**: single endpoint (`POST /graphql`), client sends a query specifying exactly which fields they need → server returns exactly that shape. **Schema**: defines types, queries, mutations (writes), subscriptions (real-time). **Resolver**: function that fetches data for each field. **Java**: Spring for GraphQL (`spring-boot-starter-graphql`). **N+1 problem**: if resolver fetches data per-item → use **DataLoader** (batching). *(GraphQL = client decide karta hai kya data chahiye — single request mein sab milta hai)*

### 📖 How It Works
```
REST (multiple round-trips):
  GET /users/1           → { id, name, email, address, ... } (over-fetch)
  GET /users/1/orders    → [{ orderId, ... }]
  GET /products/123      → { productName, price }
  = 3 HTTP requests, extra unwanted fields

GraphQL (single request):
  POST /graphql
  query {
    user(id: 1) {
      name
      orders {
        orderId
        product {
          name
          price
        }
      }
    }
  }
  
  Response:
  {
    "data": {
      "user": {
        "name": "Shubham",
        "orders": [
          { "orderId": "ORD-1", "product": { "name": "Laptop", "price": 999 } }
        ]
      }
    }
  }
  → 1 request, exactly the fields needed

Schema Definition:
  type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
  }
  
  type Order {
    orderId: ID!
    amount: Float!
    product: Product!
  }
  
  type Query {
    user(id: ID!): User
    orders(userId: ID!): [Order!]!
  }
  
  type Mutation {
    createOrder(input: CreateOrderInput!): Order!
  }
```

### 💻 Code Example
```java
// Spring for GraphQL — Controller
@Controller
public class UserController {

    @QueryMapping
    public User user(@Argument Long id) {
        return userService.findById(id);
    }

    @SchemaMapping(typeName = "User", field = "orders")
    public List<Order> orders(User user) {
        return orderService.findByUserId(user.getId());
    }
}

// DataLoader — batch N+1 queries
@Configuration
public class DataLoaderConfig {

    @Bean
    public BatchLoaderRegistry batchLoaderRegistry(OrderService orderService) {
        return registry -> registry.forTypePair(Long.class, List.class)
            .registerMappedBatchLoader((userIds, env) -> {
                // Single query: SELECT * FROM orders WHERE user_id IN (...)
                Map<Long, List<Order>> ordersByUser = orderService.findByUserIds(userIds);
                return Mono.just(ordersByUser);
            });
    }
}

// schema.graphqls (in src/main/resources/graphql/)
type Query {
    user(id: ID!): User
}

type User {
    id: ID!
    name: String!
    email: String!
    orders: [Order!]!
}

type Order {
    orderId: ID!
    amount: Float!
    status: String!
}
```

### 🗣️ Answering Approach
"I use GraphQL when the client has varied data requirements — particularly for mobile apps where bandwidth is limited and different screens need different subsets of data. With REST, you either over-fetch by returning everything or under-fetch requiring multiple round-trips. GraphQL solves this with a single endpoint where the client specifies exactly the fields needed. I implement it in Spring Boot using Spring for GraphQL — defining a schema in SDL, then writing resolver methods annotated with @QueryMapping and @SchemaMapping. The biggest challenge is the N+1 query problem — if the orders resolver fires once per user in a list, you get N separate DB queries. I solve this with DataLoader which batches these into a single IN query. For public APIs, I use REST because GraphQL can enable denial-of-service through deeply nested queries — I add query depth limiting and complexity analysis for any exposed GraphQL endpoint."

### ⚠️ Pitfalls / Gotchas
- **N+1 problem** is the #1 gotcha — use DataLoader for batching *(N+1 queries sabse bada problem hai — DataLoader use karo)*
- **Query complexity attacks** — deeply nested queries can kill the server. Add depth/complexity limits
- **Caching is harder** than REST — REST uses HTTP caching (URL-based), GraphQL responses vary per query
- **File uploads** not natively supported — use multipart or separate REST endpoint
- **Error handling** different — HTTP always returns 200. Errors in response body `errors` field

### 🆚 vs. Comparison
| Aspect | REST | GraphQL | gRPC |
|--------|------|---------|------|
| Over/under-fetching | Common ❌ | None ⭐ | None |
| Round trips | Multiple | Single ⭐ | Single ⭐ |
| Schema | OpenAPI (optional) | SDL (required) ⭐ | .proto (required) ⭐ |
| Caching | HTTP caching ⭐ | Complex | N/A |
| Real-time | SSE/WebSocket | Subscriptions ⭐ | Streaming |
| Browser support | Full ⭐ | Full ⭐ | Limited |
| Best for | Public APIs, simple CRUD | Mobile/BFF, varied clients ⭐ | Service-to-service |

### ⚡ Remember
- **GraphQL** = client picks exact fields → no over/under-fetching
- Single `POST /graphql` endpoint
- **DataLoader** = batch N+1 queries into single IN query
- **Depth limiting** = prevent nested query abuse
- Use for **mobile/BFF**; REST for public APIs; gRPC for internals

### 🔗 Follow-ups
- [Q12 → gRPC (service-to-service alternative)](#q12)
- [Q14 → SSL/TLS (securing API endpoints)](#q14)

---

<a id="q14"></a>
## Q14. SSL/TLS Encryption

### 📝 One-Liner
SSL/TLS encrypts data in transit between client and server using a **TLS handshake** (asymmetric key exchange) followed by **symmetric encryption** for fast bulk data transfer — always enforce HTTPS, use TLS 1.3.

### 🔑 Quick Answer
**TLS (Transport Layer Security)**: successor to SSL. Provides: **(1) Encryption** — data can't be read in transit. **(2) Authentication** — server proves its identity via certificate. **(3) Integrity** — data can't be tampered with (MAC). **TLS Handshake** (TLS 1.2): Client Hello → Server Hello + Certificate → Client key exchange (pre-master secret encrypted with server's public key) → Both derive session key → Symmetric encryption starts. **TLS 1.3**: faster (1-RTT handshake vs 2-RTT in 1.2), removed weak ciphers (RSA key exchange, RC4, SHA-1). **mTLS (Mutual TLS)**: both client AND server present certificates — used in microservices (service mesh like Istio). **Java**: `SSLContext`, `KeyStore`, `TrustStore`. Spring Boot: `server.ssl.*` properties. *(TLS = data transfer encrypt karo — bina TLS sab plaintext mein jaata hai)*

### 📖 How It Works
```
TLS 1.3 Handshake (1-RTT):
  Client                              Server
    |── ClientHello + KeyShare ──────→ |
    |                                  | (select cipher, sign with private key)
    |←── ServerHello + KeyShare ─────  |
    |    + Certificate + Finished      |
    |── Finished ────────────────────→ |
    |          ENCRYPTED DATA          |
    |←════════════════════════════════→|
  
  1 round-trip (TLS 1.2 needed 2 round-trips)

Certificate Trust Chain:
  Root CA (trusted by OS/browser)
    └── Intermediate CA
          └── Server Certificate (your-domain.com)
  
  Client verifies chain of trust from server cert → root CA

mTLS (service-to-service):
  ┌──────────┐                    ┌──────────┐
  │ Service A│  1. A sends cert   │ Service B│
  │ (client) │──────────────────→ │ (server) │
  │          │  2. B sends cert   │          │
  │          │←────────────────── │          │
  │          │  3. Both verify    │          │
  │          │  4. Encrypted comm │          │
  │          │←══════════════════→│          │
  └──────────┘                    └──────────┘
  → Both sides authenticated (zero-trust network)
```

### 💻 Code Example
```java
// Spring Boot — enable HTTPS
// application.yml
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: ${SSL_KEYSTORE_PASSWORD}
    key-store-type: PKCS12
    protocol: TLS
    enabled-protocols: TLSv1.3   # enforce TLS 1.3

// Force HTTPS redirect (Spring Security)
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    public SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        return http
            .requiresChannel(channel -> channel.anyRequest().requiresSecure())
            .build();
    }
}

// mTLS — RestTemplate with client certificate
@Bean
public RestTemplate mtlsRestTemplate() throws Exception {
    KeyStore keyStore = KeyStore.getInstance("PKCS12");
    keyStore.load(new FileInputStream("client-keystore.p12"),
                  keystorePassword.toCharArray());

    KeyStore trustStore = KeyStore.getInstance("PKCS12");
    trustStore.load(new FileInputStream("truststore.p12"),
                    truststorePassword.toCharArray());

    SSLContext sslContext = SSLContextBuilder.create()
        .loadKeyMaterial(keyStore, keyPassword.toCharArray())   // client cert
        .loadTrustMaterial(trustStore, null)                     // server cert trust
        .build();

    HttpClient httpClient = HttpClients.custom()
        .setSSLContext(sslContext)
        .build();
    return new RestTemplate(new HttpComponentsClientHttpRequestFactory(httpClient));
}
```

### 🗣️ Answering Approach
"I enforce TLS 1.3 for all communications in production. TLS provides encryption, authentication, and integrity for data in transit. The TLS 1.3 handshake is a significant improvement over 1.2 — it completes in just one round-trip and removes weak cipher suites. In Spring Boot, I configure HTTPS with a PKCS12 keystore and enforce TLS 1.3 through the enabled-protocols property. For microservice-to-microservice communication, I implement mTLS where both sides present certificates — this is typically handled by a service mesh like Istio which automatically manages certificate rotation and mTLS enforcement without application code changes. For key management, I use vault-based solutions to store and rotate certificates rather than bundling them in the application."

### ⚠️ Pitfalls / Gotchas
- **Never use SSL 3.0 or TLS 1.0/1.1** — deprecated, known vulnerabilities *(Purane SSL/TLS versions mein known vulnerabilities hain — TLS 1.3 enforce karo)*
- **Certificate expiry** — automate renewal with Let's Encrypt / cert-manager in K8s
- **Hardcoded passwords** for keystore — use secrets management (Vault, K8s Secrets)
- **Self-signed certs in prod** — browsers reject them; use proper CA-signed certificates
- **TLS termination** at load balancer → internal traffic unencrypted. Use mTLS internally too

### ⚡ Remember
- **TLS 1.3** = 1-RTT handshake, no weak ciphers, mandatory ⭐
- **mTLS** = both sides authenticate (microservices, zero-trust) *(mTLS = dono taraf certificate verify — zero-trust mein zaroori)*
- **KeyStore** = your cert + private key; **TrustStore** = trusted CA certs
- Automate cert rotation (Let's Encrypt, Istio, cert-manager)
- Never hardcode keystore passwords — use Vault/Secrets

### 🔗 Follow-ups
- [Q11 → Socket Programming (TLS over TCP sockets)](#q11)
- Q16 (architecture/01) → JWT Auth (authentication after TLS)
