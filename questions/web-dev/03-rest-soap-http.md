# 🌐 REST vs SOAP — Interview Questions

---

## Q1. What is the Difference Between REST and SOAP?

### 📝 One-Liner
REST is an architectural style using HTTP methods; SOAP is a protocol with XML-based messaging and strict standards.

### 🔑 Quick Answer
REST: lightweight, uses JSON/HTTP, stateless, flexible. SOAP: heavyweight, XML-only, has WS-Security/ACID transactions, strict contract (WSDL). Modern APIs prefer REST; enterprise/banking systems often use SOAP. *(REST lightweight hai JSON ke saath, SOAP heavyweight hai XML ke saath — banking mein SOAP zyada milta hai)*

### 📖 How It Works

| Aspect | REST | SOAP |
|--------|------|------|
| Type | Architectural style | Protocol |
| Data Format | JSON, XML, plain text | XML only |
| Transport | HTTP | HTTP, SMTP, TCP, JMS |
| Contract | Optional (OpenAPI/Swagger) | Mandatory (WSDL) |
| State | Stateless | Can be stateful |
| Security | HTTPS, OAuth, JWT | WS-Security (enterprise-grade) |
| Transactions | No built-in | WS-AtomicTransaction |
| Caching | Built-in (HTTP caching) | No |
| Performance | Faster (less overhead) | Slower (XML parsing, SOAP envelope) |
| Error Handling | HTTP status codes | SOAP Fault element |

**When to use SOAP**:
- Enterprise integration (banking, insurance) *(bank aur insurance mein SOAP use hota hai)*
- ACID transactions required
- Formal contract (WSDL) needed
- Complex security requirements

**When to use REST**:
- Web/mobile APIs
- Public APIs
- Microservices communication
- When performance matters

### 🗣️ Answering Approach
"REST is lightweight and uses standard HTTP methods with JSON, making it ideal for web and mobile APIs. SOAP is a full protocol with XML messaging, WSDL contracts, and built-in security/transaction support, making it suitable for enterprise systems like banking. In my experience, I've used REST for microservice communication and SOAP when integrating with legacy banking systems."

### 💻 Code
```java
// REST endpoint (Spring)
@RestController
@RequestMapping("/api/users")
public class UserController {
    @GetMapping("/{id}")
    public ResponseEntity<User> getUser(@PathVariable Long id) {
        return ResponseEntity.ok(userService.findById(id));
    }
}

// SOAP endpoint (Spring WS)
@Endpoint
public class UserEndpoint {
    @PayloadRoot(namespace = "http://example.com/users", localPart = "getUserRequest")
    @ResponsePayload
    public GetUserResponse getUser(@RequestPayload GetUserRequest request) {
        return userService.findById(request.getId());
    }
}
```

### ⚡ Remember
- REST = resources + HTTP verbs + JSON → simple, fast
- SOAP = XML + WSDL + WS-* → complex, feature-rich
- Modern microservices: REST (or gRPC for internal)
- Legacy/enterprise: SOAP still alive in banking

---
