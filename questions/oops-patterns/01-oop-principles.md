# 🏗️ OOP Principles — 4 Pillars + SOLID (Q1)

> 📝 One-Liner → 🔑 Quick Answer → 📖 How It Works → 🗣️ Interview Script → 💻 Code → ⚠️ Pitfalls → 🆚 vs. → 🎯 Tricky Qs → ⚡ Remember → 🔗 Follow-ups

---

<a id="q1"></a>
## Q1. Explain OOP principles — 4 Pillars and SOLID

### 📝 One-Liner
OOP has **4 pillars** (Encapsulation, Inheritance, Polymorphism, Abstraction) and **5 SOLID principles** (SRP, OCP, LSP, ISP, DIP) — together they guide writing maintainable, extensible, and testable object-oriented code.

### 🔑 Quick Answer

**4 Pillars:**
1. **Encapsulation** — Bundle data + methods, hide internals via access modifiers. Fields `private`, expose controlled access via getters/setters or methods.
2. **Inheritance** — Child class reuses parent's fields/methods (`extends`). Promotes code reuse. "IS-A" relationship.
3. **Polymorphism** — Same method name, different behavior. **Compile-time** (method overloading) + **Runtime** (method overriding via inheritance/interfaces).
4. **Abstraction** — Hide complex implementation, expose only what's needed (`abstract class` / `interface`). User knows *what* not *how*.

**SOLID:**
1. **S**ingle Responsibility — A class should have only one reason to change.
2. **O**pen/Closed — Open for extension, closed for modification.
3. **L**iskov Substitution — Subtype must be substitutable for its parent without breaking behavior.
4. **I**nterface Segregation — Many specific interfaces > one fat interface.
5. **D**ependency Inversion — Depend on abstractions, not concrete implementations.

*(4 pillars = OOP ka foundation; SOLID = code ko maintainable banane ke rules)*

### 📖 How It Works
```
4 Pillars in Action:

Encapsulation:
  private double balance;  // hidden
  public void deposit(double amount) {  // controlled access
    if (amount > 0) this.balance += amount;
  }

Inheritance:
  Animal → Dog, Cat  (Dog IS-A Animal)
  Dog inherits eat(), sleep() from Animal

Polymorphism:
  Compile-time: add(int, int) vs add(double, double)  (overloading)
  Runtime:      animal.speak() → Dog says "Bark", Cat says "Meow"  (overriding)

Abstraction:
  interface PaymentGateway { void charge(Order order); }
  // Caller doesn't know if it's Stripe, PayPal, or Razorpay

SOLID in Action:

  SRP: UserService handles users, EmailService handles emails (not one class doing both)
  OCP: Add new PaymentMethod without modifying existing code (Strategy pattern)
  LSP: Square extends Rectangle should still work wherever Rectangle is used
  ISP: Printer interface split into Printable, Scannable, Faxable
  DIP: Controller depends on Service interface, not ServiceImpl
```

### 🗣️ Interview Script
"OOP has four pillars. Encapsulation bundles data and methods together and controls access through visibility modifiers — I keep fields private and expose behavior through methods, so internal representation can change without breaking callers. Inheritance enables code reuse through IS-A relationships — a Dog extends Animal and inherits common behavior. Polymorphism means the same method call can behave differently — at compile time through overloading (different parameter types) and at runtime through overriding (subclass provides its own implementation). Abstraction hides complexity behind interfaces or abstract classes — my service depends on a PaymentGateway interface, not on Stripe or PayPal directly.

For SOLID: Single Responsibility means each class has one job — UserService manages users, not users AND emails. Open/Closed means I can add new functionality by extending, not modifying existing code — this is where the Strategy pattern shines. Liskov Substitution says if I replace a parent with its subclass, the program should still work correctly — the classic violation is Square extending Rectangle. Interface Segregation means I shouldn't force classes to implement methods they don't use — split fat interfaces into focused ones. Dependency Inversion means high-level modules depend on abstractions, not concrete implementations — this is the foundation of Spring's DI."

### 💻 Code Example

```java
// === ENCAPSULATION ===
public class BankAccount {
    private double balance;  // hidden state

    public void deposit(double amount) {
        if (amount <= 0) throw new IllegalArgumentException("Amount must be positive");
        this.balance += amount;
    }

    public double getBalance() {
        return this.balance;  // read-only access
    }
    // No setBalance() — balance can only change through business methods
}

// === INHERITANCE + POLYMORPHISM ===
abstract class Shape {
    abstract double area();  // abstraction — what, not how
}

class Circle extends Shape {
    private final double radius;
    Circle(double radius) { this.radius = radius; }

    @Override
    double area() { return Math.PI * radius * radius; }  // runtime polymorphism
}

class Rectangle extends Shape {
    private final double width, height;
    Rectangle(double w, double h) { this.width = w; this.height = h; }

    @Override
    double area() { return width * height; }
}

// Polymorphism in action — same method, different behavior
List<Shape> shapes = List.of(new Circle(5), new Rectangle(3, 4));
shapes.forEach(s -> System.out.println(s.area()));  // 78.54, 12.0

// === SOLID PRINCIPLES ===

// SRP — each class has one responsibility
class UserService {
    User createUser(UserRequest req) { ... }  // only user logic
}
class EmailService {
    void sendWelcomeEmail(User user) { ... }  // only email logic
}

// OCP — open for extension, closed for modification
interface DiscountStrategy {
    double apply(double price);
}
class FlatDiscount implements DiscountStrategy {
    public double apply(double price) { return price - 100; }
}
class PercentDiscount implements DiscountStrategy {
    public double apply(double price) { return price * 0.9; }
}
// Add new discount types without changing existing code

// LSP — subtypes must be substitute-able
// ❌ Violation: Square overrides setWidth to also set height → breaks Rectangle behavior
// ✅ Fix: don't extend Rectangle; use Shape interface instead

// ISP — focused interfaces
interface Printable { void print(); }
interface Scannable { void scan(); }
// Instead of: interface Machine { void print(); void scan(); void fax(); }
// A simple printer implements only Printable

// DIP — depend on abstractions
@Service
class OrderService {
    private final PaymentGateway gateway;  // depends on interface
    OrderService(PaymentGateway gateway) { this.gateway = gateway; }
    // Spring injects StripeGateway or PayPalGateway — OrderService doesn't know or care
}
```

### ⚠️ Pitfalls / Gotchas
- **Inheritance abuse** — prefer composition over inheritance. "HAS-A" (composition) is more flexible than "IS-A" (inheritance) *(jab bhi doubt ho, composition use karo)*
- **LSP violation** — Square extending Rectangle breaks substitutability; use interfaces instead
- **God class** — violates SRP; if a class has 20+ methods or 500+ lines, it probably needs splitting
- **Empty interface methods** — violates ISP; if you're writing `throw new UnsupportedOperationException()`, the interface is too fat

### 🆚 Key Comparisons

| Concept | Overloading | Overriding |
|---------|-------------|-----------|
| **Type** | Compile-time polymorphism | Runtime polymorphism |
| **Where** | Same class | Parent → child |
| **Signature** | Different params | Same signature |
| **Return type** | Can differ | Must be same or covariant |
| **`@Override`** | Not used | Required (best practice) |

| Concept | Abstract Class | Interface |
|---------|---------------|-----------|
| **Methods** | Abstract + concrete | Abstract (+ default since Java 8) |
| **Fields** | Instance fields | Only constants (static final) |
| **Constructor** | Yes | No |
| **Inheritance** | Single (`extends`) | Multiple (`implements`) |
| **Use when** | Shared state + partial implementation | Contract/capability |

### 🎯 Tricky Interview Qs

**Q: Can you achieve polymorphism without inheritance?**
Yes — through interfaces. A class implementing `Comparable` achieves polymorphism without extending another class. Also, lambdas provide polymorphic behavior through functional interfaces.

**Q: Why is composition preferred over inheritance?**
Inheritance is rigid (can't change at runtime), breaks encapsulation (subclass depends on parent internals), and leads to fragile base class problem. Composition is flexible — swap components at runtime, no tight coupling.

**Q: Give a real-world example of Dependency Inversion in Spring.**
`@Service` class depends on `@Repository` interface. Spring injects the JPA implementation. If you switch to MongoDB, only the implementation changes — the service layer is untouched.

### ⚡ Remember
- **4 Pillars**: Encapsulation (hide), Inheritance (reuse), Polymorphism (many forms), Abstraction (simplify)
- **SOLID**: SRP (one job) → OCP (extend, don't modify) → LSP (substitute safely) → ISP (thin interfaces) → DIP (depend on abstractions)
- Composition > Inheritance for most cases
- Spring framework is built on DIP — `@Autowired` injects abstractions
