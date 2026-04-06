# 🧱 OOP in Real Projects — Practical Examples (Q1–Q5)

> **Source**: "OOP in Real Projects" LinkedIn post (2026)  
> **Coverage**: How OOP principles (Encapsulation, Abstraction, Inheritance, Polymorphism, SOLID) manifest in real automation and backend projects — not textbook definitions but actual usage  
> **Cross-refs**: Textbook OOP → [oops-patterns/01-oop-principles.md](01-oop-principles.md)

---

<a id="q1"></a>
## Q1. How is Encapsulation used in real projects? (POM, Private Fields, Getters)

### 📝 One-Liner
Encapsulation hides internal implementation details — in Page Object Model, page locators are `private` and exposed only through meaningful action methods.

### 🔑 Quick Answer
In automation testing (Selenium), the **Page Object Model (POM)** uses encapsulation extensively:
- **Locators** declared as `private` fields → no test class can directly access `driver.findElement()`
- **Action methods** (public) expose behavior: `loginPage.enterUsername("admin")` not `loginPage.usernameField.sendKeys("admin")`
- **Benefits**: If locator changes from ID to XPath, only the Page class changes. Tests don't break.

In backend services:
- **Entity fields** are `private` → accessed via getters/setters
- **DTOs** hide internal representation → API response doesn't expose database column names
- **Service internals** hidden from controllers → controller calls `orderService.placeOrder(dto)`, not individual DB queries

*(Private fields rakhte ho, getter/setter expose karte ho — taaki koi bhi directly modify na kar sake)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Encapsulation in Page Object Model
public class LoginPage {
    private WebDriver driver;
    
    // Private locators — hidden from tests
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton = By.xpath("//button[@type='submit']");
    
    // Public behavior methods — this is the "contract"
    public void enterUsername(String username) {
        driver.findElement(usernameField).sendKeys(username);
    }
    
    public void enterPassword(String password) {
        driver.findElement(passwordField).sendKeys(password);
    }
    
    public DashboardPage clickLogin() {
        driver.findElement(loginButton).click();
        return new DashboardPage(driver);
    }
}

// ✅ Test uses behavior, not internals
@Test
void testLogin() {
    loginPage.enterUsername("admin");
    loginPage.enterPassword("pass123");
    DashboardPage dashboard = loginPage.clickLogin();
    assertTrue(dashboard.isWelcomeVisible());
}
```

### 🗣️ Answering Approach
"In my automation project, Encapsulation was core to the Page Object Model. All locators were private — test classes never directly called driver.findElement(). Instead, public action methods like enterUsername() and clickLogin() exposed meaningful behavior. When the frontend team changed the login button from ID-based to class-based, I only changed one line in LoginPage.java — all 50 test cases that used login continued working without modification. In backend, I apply the same principle: entity fields are private, DTOs hide database structure, and service internals are never exposed to controllers."

---

<a id="q2"></a>
## Q2. How is Abstraction used in real projects? (Utility Methods, Helper Classes)

### 📝 One-Liner
Abstraction hides the "how" and exposes the "what" — utility/helper classes provide simple method calls that hide complex multi-step operations.

### 🔑 Quick Answer
**In automation:**
- `WaitUtils.waitForElementVisible(locator)` — hides WebDriverWait, FluentWait, polling config
- `ExcelReader.getData(sheet, row, col)` — hides FileInputStream, Workbook creation, sheet handling
- `BrowserFactory.getDriver("chrome")` — hides ChromeOptions, WebDriverManager, capabilities setup

**In backend:**
- `emailService.sendOtp(user)` — hides SMTP config, template rendering, retry logic
- `paymentGateway.charge(order)` — hides API calls, signature generation, error mapping

*(Complex implementation chhupao, simple method dikhao — yahi abstraction hai)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Abstraction — complex operation behind simple interface
public class WaitUtils {
    // User sees: waitForElement(locator) — simple
    // Hidden: WebDriverWait + FluentWait + polling + exception handling
    public static WebElement waitForElement(By locator) {
        return new WebDriverWait(DriverManager.getDriver(), Duration.ofSeconds(10))
            .pollingEvery(Duration.ofMillis(500))
            .ignoring(StaleElementReferenceException.class)
            .until(ExpectedConditions.visibilityOfElementLocated(locator));
    }
}

// ✅ Backend abstraction — payment gateway
public interface PaymentGateway {
    PaymentResult charge(Order order);  // simple contract
}

// Hidden implementation complexity
public class StripePaymentGateway implements PaymentGateway {
    @Override
    public PaymentResult charge(Order order) {
        // 1. Build Stripe PaymentIntent
        // 2. Add idempotency key
        // 3. Make API call with retry
        // 4. Map Stripe response to domain object
        // 5. Handle decline codes
        return result;
    }
}
```

### 🗣️ Answering Approach
"Abstraction hides complexity behind simple interfaces. In my automation framework, I created utility classes: WaitUtils hides the complexity of WebDriverWait with polling, timeouts, and exception handling behind a single waitForElement() call. ExcelReader hides Apache POI's file streams, workbook creation, and cell type handling. In backend, my PaymentGateway interface exposes just charge() — the Stripe implementation behind it handles API calls, idempotency keys, retries, and error mapping. Users of the interface never see that complexity."

---

<a id="q3"></a>
## Q3. How is Inheritance used in real projects? (BaseTest, DriverFactory, Base Entities)

### 📝 One-Liner
Base classes provide shared setup/teardown, configuration, and common utilities — concrete classes extend them with specific behavior.

### 🔑 Quick Answer
**In automation:**
- `BaseTest` → setup/teardown (browser init, screenshots on failure, report logging)
- All test classes extend `BaseTest` → reuse setup without duplication
- `BasePage` → common page operations (wait, scroll, screenshot)

**In backend (Spring Boot):**
- `BaseEntity` → `id`, `createdAt`, `updatedAt`, `version` fields shared across all entities
- `BaseRepository<T>` → common query methods
- `BaseController` → standard response wrapping, error handling

*(Ek base class banao, common cheezein rakh do — sub classes sirf apna specific kaam karein)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Automation — BaseTest with shared lifecycle
public class BaseTest {
    protected WebDriver driver;
    
    @BeforeEach
    void setUp() {
        driver = DriverFactory.createDriver("chrome");
        driver.manage().window().maximize();
    }
    
    @AfterEach
    void tearDown(TestInfo testInfo) {
        if (testInfo.getTags().contains("failed")) {
            ScreenshotUtils.capture(driver, testInfo.getDisplayName());
        }
        driver.quit();
    }
}

// Every test class gets free setup/teardown
public class LoginTest extends BaseTest {
    @Test void testValidLogin() { /* uses this.driver directly */ }
}

// ✅ Backend — BaseEntity
@MappedSuperclass
public abstract class BaseEntity {
    @Id @GeneratedValue private Long id;
    @CreatedDate private LocalDateTime createdAt;
    @LastModifiedDate private LocalDateTime updatedAt;
    @Version private Long version;  // optimistic locking
}

public class Order extends BaseEntity {
    private String orderNumber;
    private BigDecimal total;
    // id, createdAt, updatedAt, version inherited automatically
}
```

### 🗣️ Answering Approach
"In my automation framework, every test class extends BaseTest which handles browser initialization, window maximize, and screenshot capture on failure. When I added a new test class, I didn't duplicate any setup code — just extended BaseTest and focused on test logic. In Spring Boot, I use @MappedSuperclass for a BaseEntity with id, createdAt, updatedAt, and version fields. Every entity inherits these audit fields automatically. This eliminates duplication across 20+ entity classes."

---

<a id="q4"></a>
## Q4. How is Polymorphism used in real projects? (Method Overloading, Platform Handling)

### 📝 One-Liner
Same method name handles different scenarios — method overloading for different parameter types, and runtime polymorphism for platform-specific behavior (Chrome/Firefox/Safari, Android/iOS).

### 🔑 Quick Answer
**Compile-time (Overloading):**
- `click(By locator)` → simple click
- `click(By locator, int timeoutSec)` → click with custom wait
- `click(WebElement element)` → click a pre-found element

**Runtime (Overriding / Interface):**
- `DriverFactory.createDriver("chrome")` → returns ChromeDriver
- `DriverFactory.createDriver("firefox")` → returns FirefoxDriver
- Both return `WebDriver` type → calling code doesn't change

*(Ek hi method naam, alag behavior — compile time pe overloading, runtime pe overriding. Same interface, platform change karo — code same rahega)*

### 📖 How It Works (Detailed Explanation)

```java
// ✅ Runtime polymorphism — DriverFactory
public class DriverFactory {
    public static WebDriver createDriver(String browser) {
        return switch (browser.toLowerCase()) {
            case "chrome"  -> new ChromeDriver(chromeOptions());
            case "firefox" -> new FirefoxDriver(firefoxOptions());
            case "safari"  -> new SafariDriver();
            default -> throw new IllegalArgumentException("Unsupported: " + browser);
        };
    }
}
// Test code: WebDriver driver = DriverFactory.createDriver(config.getBrowser());
// Works identically regardless of browser — polymorphism!

// ✅ Backend — Strategy Pattern (runtime polymorphism)
public interface NotificationSender {
    void send(User user, String message);
}

@Component("email") 
class EmailSender implements NotificationSender {
    public void send(User user, String message) { /* SMTP logic */ }
}

@Component("sms")
class SmsSender implements NotificationSender {
    public void send(User user, String message) { /* Twilio logic */ }
}

// Controller: same method, different behavior based on channel
NotificationSender sender = senderMap.get(channel); // "email" or "sms"
sender.send(user, message); // polymorphic call
```

### 🗣️ Answering Approach
"Polymorphism shows up in two ways in my projects. Compile-time: I overload the click() method — one version takes a locator, another takes a locator plus timeout, another takes a pre-found WebElement. The caller picks the right version based on the situation. Runtime: DriverFactory returns WebDriver, but the actual object could be ChromeDriver, FirefoxDriver, or SafariDriver. The entire test suite runs without knowing which browser is underneath — I just change a config property. In backend, the Strategy pattern is pure polymorphism: NotificationSender interface with EmailSender and SmsSender — the calling code is identical."

---

<a id="q5"></a>
## Q5. Real-world SOLID principles implementation — give a concrete project example

### 📝 One-Liner
A test framework or microservice where each SOLID principle is demonstrated through actual design decisions, not textbook definitions.

### 🔑 Quick Answer

| Principle | Real Example |
|-----------|-------------|
| **S** — Single Responsibility | LoginPage only handles login UI. Assertions are in a separate AssertionUtils class. |
| **O** — Open/Closed | Add new browsers by adding a class (SafariDriver), not modifying DriverFactory's existing code. Strategy pattern for new payment methods. |
| **L** — Liskov Substitution | Every WebDriver subtype (Chrome, Firefox) works wherever WebDriver is expected. No special handling needed. |
| **I** — Interface Segregation | Separate `Clickable`, `Scrollable`, `Draggable` interfaces instead of one fat `PageElement` interface. |
| **D** — Dependency Inversion | Tests depend on `WebDriver` interface, not `ChromeDriver` concrete class. Services depend on `Repository` interface, not `JpaRepository` implementation. |

### 📖 How It Works (Detailed Explanation)

```java
// ✅ S: Single Responsibility
class LoginPage { /* only login UI actions */ }
class LoginValidator { /* only validation logic */ }
class LoginReporter { /* only report generation */ }

// ✅ O: Open/Closed — add OTP login without modifying existing code
interface LoginStrategy { void login(Credentials creds); }
class PasswordLogin implements LoginStrategy { /* existing */ }
class OtpLogin implements LoginStrategy { /* new — no existing code changed */ }

// ✅ L: Liskov — any WebDriver subtype works
void runTest(WebDriver driver) { // works with ANY implementation
    driver.get("https://example.com");
    driver.findElement(By.id("login")).click();
}

// ✅ I: Interface Segregation
interface Clickable { void click(); }
interface Scrollable { void scrollTo(); }
class Button implements Clickable { /* only click */ }
class ScrollPanel implements Clickable, Scrollable { /* both */ }

// ✅ D: Dependency Inversion
class OrderService {
    private final OrderRepository repo; // depends on interface, NOT JpaOrderRepository
    OrderService(OrderRepository repo) { this.repo = repo; }
}
```

### 🗣️ Answering Approach
"Let me walk through each principle with a real example from my project. Single Responsibility: LoginPage class only handles login UI interactions — assertions are in a separate utility. Open/Closed: when we needed OTP-based login, I created a new OtpLoginStrategy class without modifying the existing PasswordLogin — Strategy pattern. Liskov: all WebDriver subtypes are interchangeable — tests work identically on Chrome, Firefox, or Safari. Interface Segregation: instead of one huge PageElement interface, I have Clickable, Scrollable, Draggable — a Button only implements Clickable. Dependency Inversion: services depend on Repository interfaces, not JPA implementations — this makes testing with mocks trivial."

### ⚡ Remember
- Cross-ref: [OOP Principles (textbook definitions) → oops-patterns/01-oop-principles.md](01-oop-principles.md)
- Cross-ref: [Strategy Pattern for login OTP/CAPTCHA → spring/11 Q2](../languages/java/spring/11-springboot-scenario-interviews.md#q2)
- Cross-ref: [Design Patterns in practice → core/18-design-patterns-java.md](../languages/java/core/18-design-patterns-java.md)
