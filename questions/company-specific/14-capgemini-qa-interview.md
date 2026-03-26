# 🏢 Capgemini — QA / Automation Testing Interview Experience (L1 Round)

> L1 round covering core automation concepts, Selenium/framework design, Java collections, API testing, Cucumber BDD, real-time debugging scenarios, and OOP in automation frameworks.

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

<a id="q1"></a>
## Q1. Introduction and project explanation — Automation framework walkthrough

### 🗣️ Interview Script
"I work on a POM-based Selenium automation framework using Java, TestNG, and Maven. The framework follows Page Object Model where each page has a corresponding class with locators and actions. We use a Data-Driven approach with Excel/JSON for test data, Extent Reports for reporting, and Jenkins for CI/CD execution. The framework covers web UI tests, API tests (RestAssured), and database validation. We run ~500 test cases across 3 environments with parallel execution using TestNG XML."

### ⚡ Remember
> Structure: POM + TestNG + Maven | Data-driven (Excel/JSON) | Reporting (Extent/Allure) | CI/CD integration | Be specific about YOUR framework, not generic

---

<a id="q2"></a>
## Q2. POJO classes and their usage in automation

### 📝 One-Liner
POJO (Plain Old Java Object) classes are simple Java classes with private fields + getters/setters — used in automation for **test data models**, **API request/response mapping**, and **configuration objects**.

### 💻 Code
```java
// POJO for API response mapping
public class UserResponse {
    private int id;
    private String name;
    private String email;
    // getters, setters, constructor

    // RestAssured usage
    // UserResponse user = given().get("/api/users/1").as(UserResponse.class);
}

// POJO for test data
public class LoginData {
    private String username;
    private String password;
    private boolean expectedResult;
    // Used with DataProvider or JSON deserialization
}
```

### ⚡ Remember
> POJO = simple data container | Used for API response deserialization (RestAssured/Jackson) | Test data modeling | Configuration objects | Keeps code clean and type-safe

---

<a id="q3"></a>
## Q3. Page Factory — object initialization

### 📝 One-Liner
Page Factory uses `@FindBy` annotations to declare locators and `PageFactory.initElements(driver, this)` to lazily initialize web elements — cleaner than `driver.findElement()` calls scattered in code.

### 💻 Code
```java
public class LoginPage {
    @FindBy(id = "username")
    private WebElement usernameInput;

    @FindBy(id = "password")
    private WebElement passwordInput;

    @FindBy(css = "button[type='submit']")
    private WebElement loginButton;

    public LoginPage(WebDriver driver) {
        PageFactory.initElements(driver, this); // Initialize elements
    }

    public void login(String user, String pass) {
        usernameInput.sendKeys(user);
        passwordInput.sendKeys(pass);
        loginButton.click();
    }
}
```

### ⚡ Remember
> `@FindBy` annotations for locators | `PageFactory.initElements()` in constructor | Elements lazily initialized | Cleaner than `driver.findElement()` | Each page = separate class (POM)

---

<a id="q4"></a>
## Q4. Handling scrolling in Selenium

### 💻 Code
```java
// JavaScript scroll
JavascriptExecutor js = (JavascriptExecutor) driver;
js.executeScript("window.scrollBy(0, 500)");                  // Scroll down 500px
js.executeScript("window.scrollTo(0, document.body.scrollHeight)"); // Scroll to bottom
js.executeScript("arguments[0].scrollIntoView(true);", element); // Scroll to element

// Actions class scroll
Actions actions = new Actions(driver);
actions.scrollToElement(element).perform();      // Selenium 4
actions.scrollByAmount(0, 500).perform();        // Selenium 4
```

### ⚡ Remember
> `scrollIntoView` for specific element | `scrollTo(0, scrollHeight)` for bottom | Selenium 4: `Actions.scrollToElement()` | JavascriptExecutor is the reliable fallback

---

<a id="q5"></a>
## Q5. Handling non-select dropdowns

### 📝 One-Liner
Non-select dropdowns (custom divs/spans) can't use Selenium's `Select` class — click the dropdown trigger, wait for options to appear, then click the desired option by text/attribute.

### 💻 Code
```java
// Click dropdown trigger
driver.findElement(By.cssSelector(".dropdown-toggle")).click();

// Wait for options to be visible
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(10));
wait.until(ExpectedConditions.visibilityOfElementLocated(By.cssSelector(".dropdown-menu")));

// Select option by text
List<WebElement> options = driver.findElements(By.cssSelector(".dropdown-menu li"));
options.stream()
    .filter(opt -> opt.getText().equals("India"))
    .findFirst()
    .ifPresent(WebElement::click);
```

### ⚡ Remember
> Custom dropdown ≠ `<select>` → can't use Select class | Click trigger → wait → click option | Use explicit waits (not Thread.sleep) | Stream API for finding option by text

---

<a id="q6"></a>
## Q6. Debugging failed test cases and flaky test cases

### 📝 One-Liner
**Failed**: Check screenshot + logs → identify root cause (locator changed, data issue, env issue). **Flaky**: Add explicit waits, stabilize locators, isolate test data, check for race conditions, retry mechanism.

### 🔑 Quick Answer
**Debugging steps**: (1) Check screenshot/video (if captured). (2) Check logs — exact error and stack trace. (3) Reproduce manually. (4) Check if locator changed (DOM updates). (5) Check test data (stale/shared data). (6) Check environment (API down, slow network). **Flaky test fixes**: (1) Replace `Thread.sleep` with explicit waits. (2) Use stable locators (ID/data-testid over XPath). (3) Isolate test data. (4) Add retry annotation. (5) Run in isolation to confirm.

### ⚡ Remember
> Screenshot + logs first | Reproduce manually | Flaky = timing issue (add waits), data issue (isolate), or env issue | Never ignore flaky tests — fix root cause | Retry is a band-aid, not a solution

---

<a id="q7"></a>
## Q7. Approach when 15 out of 100 test cases fail

### 🗣️ Interview Script
"First, I'd categorize the failures: (1) Is there a common pattern? — Same page, same API, same data? If 10 failures are on the login page, it might be an environment issue, not 10 separate bugs. (2) Check infrastructure — is the test environment stable? (3) Separate genuine bugs from automation issues (stale locators, timing). (4) Prioritize by business impact — critical flow failures first. (5) Log defects for genuine bugs. (6) Fix automation issues immediately. (7) Rerun after fixes to confirm stability. (8) Report to team with categorized analysis."

### ⚡ Remember
> Categorize failures (pattern?) | Infrastructure check | Bugs vs automation issues | Prioritize by business impact | Fix + rerun + report | Root cause analysis prevents recurrence

---

<a id="q8"></a>
## Q8. Cucumber concepts — Data Table, Scenario Outline, Glue

### 📝 One-Liner
**Scenario Outline** = parameterized scenario with Examples table (runs once per row). **Data Table** = inline table within a step for structured data. **Glue** = package path connecting feature files to step definitions.

### 💻 Code
```gherkin
# Scenario Outline — parameterized
Scenario Outline: Login with multiple users
  Given I am on the login page
  When I enter "<username>" and "<password>"
  Then I should see "<result>"
  Examples:
    | username | password | result  |
    | admin    | admin123 | Dashboard |
    | invalid  | wrong    | Error   |

# Data Table — structured data in a step
Scenario: Create users
  Given the following users exist:
    | name  | email          | role  |
    | Alice | alice@test.com | Admin |
    | Bob   | bob@test.com   | User  |
```
```java
// Glue — connects features to step definitions
@CucumberOptions(
    features = "src/test/resources/features",
    glue = "com.project.stepdefinitions",  // Package with step defs
    plugin = {"pretty", "html:target/report.html"}
)
public class TestRunner extends AbstractTestNGCucumberTests {}
```

### ⚡ Remember
> **Scenario Outline** = same steps, different data (Examples table) | **Data Table** = structured data within one step | **Glue** = package linking features → step defs | Use tags for selective execution (`@smoke`, `@regression`)

---

<a id="q9"></a>
## Q9. API testing basics — Request Specification

### 💻 Code
```java
// RestAssured — Request Specification (reusable config)
RequestSpecification requestSpec = new RequestSpecBuilder()
    .setBaseUri("https://api.example.com")
    .setContentType(ContentType.JSON)
    .addHeader("Authorization", "Bearer " + token)
    .build();

// Usage
given()
    .spec(requestSpec)
    .body(userPayload)
.when()
    .post("/users")
.then()
    .statusCode(201)
    .body("name", equalTo("Alice"))
    .body("id", notNullValue());
```

### ⚡ Remember
> `RequestSpecBuilder` for reusable config | BDD style: given-when-then | Validate status + body + headers | JSONPath assertions | Use specification for base URL, auth, content type

---

<a id="q10"></a>
## Q10. Java Collections — key concepts for automation

### 📝 One-Liner
**List** (ordered, duplicates allowed — ArrayList, LinkedList), **Set** (unique elements — HashSet, TreeSet), **Map** (key-value — HashMap, LinkedHashMap). In automation: List for ordered data, Map for key-value configs, Set for unique test data.

### ⚡ Remember
> `ArrayList` for most lists | `HashMap` for key-value (test data, config) | `LinkedHashMap` to preserve insertion order | `Set` to remove duplicates | Use streams for filtering/transforming

---

<a id="q11"></a>
## Q11. Managing object locators centrally — framework design

### 📝 One-Liner
Centralize locators using: (1) **Page Object classes** (recommended — locators with methods). (2) **Properties/YAML files** (external — change without recompilation). (3) **Constants class** (simple, strongly typed). POM approach is preferred for maintainability.

### ⚡ Remember
> POM = locators in page classes (best practice) | External files (properties/YAML) for cross-platform | Never hardcode locators in test methods | Use data-testid attributes for stable locators | Locator strategy: ID > CSS > XPath

---

<a id="q12"></a>
## Q12. What are Listeners in TestNG?

### 📝 One-Liner
TestNG Listeners intercept test events — `ITestListener` (test start/pass/fail/skip), `ISuiteListener` (suite start/finish), `IInvokedMethodListener` (before/after each method). Used for screenshots on failure, custom logging, and real-time reporting.

### 💻 Code
```java
public class TestListener implements ITestListener {
    @Override
    public void onTestFailure(ITestResult result) {
        // Take screenshot on failure
        TakesScreenshot ts = (TakesScreenshot) driver;
        File src = ts.getScreenshotAs(OutputType.FILE);
        Files.copy(src.toPath(), Path.of("screenshots/" + result.getName() + ".png"));
        // Attach to report
    }

    @Override
    public void onTestSuccess(ITestResult result) {
        log.info("PASSED: {}", result.getName());
    }
}

// Register in TestNG XML or annotation
@Listeners(TestListener.class)
public class LoginTest { /* ... */ }
```

### ⚡ Remember
> `ITestListener` = most used (pass/fail/skip events) | Screenshot on failure is #1 use case | Register via annotation or testng.xml | `ISuiteListener` for setup/teardown at suite level | Custom logging + reporting integration

---

<a id="q13"></a>
## Q13. Applying OOP concepts in automation frameworks

### 📝 One-Liner
**Encapsulation**: private locators + public methods in Page Objects. **Inheritance**: BasePage with common methods (click, type, wait), pages extend it. **Polymorphism**: same method name for different page behaviors. **Abstraction**: interfaces/abstract classes for page contracts.

### 💻 Code
```java
// Abstraction + Inheritance
public abstract class BasePage {
    protected WebDriver driver;
    protected WebDriverWait wait;

    // Encapsulated reusable methods
    protected void click(By locator) {
        wait.until(ExpectedConditions.elementToBeClickable(locator)).click();
    }
    protected void type(By locator, String text) {
        WebElement el = wait.until(ExpectedConditions.visibilityOfElementLocated(locator));
        el.clear();
        el.sendKeys(text);
    }
    protected abstract String getPageTitle(); // Abstract contract
}

// LoginPage extends BasePage (Inheritance)
public class LoginPage extends BasePage {
    private By username = By.id("user");  // Encapsulation
    private By password = By.id("pass");

    public void login(String user, String pass) {
        type(username, user);  // Reused from BasePage
        type(password, pass);
        click(By.id("submit"));
    }

    @Override
    protected String getPageTitle() { return "Login"; }
}
```

### ⚡ Remember
> **POM = Encapsulation** (private locators, public actions) | **BasePage = Inheritance** (reusable methods) | **Abstract methods = Abstraction** (page contracts) | **Interface** for cross-cutting (Navigable, Searchable) | OOP makes framework maintainable and extensible

---

<a id="q14"></a>
## Q14. Write code to handle multiple windows

### 💻 Code
```java
// Store parent window
String parentWindow = driver.getWindowHandle();

// Click link that opens new window
driver.findElement(By.linkText("Open New Window")).click();

// Switch to new window
Set<String> allWindows = driver.getWindowHandles();
for (String window : allWindows) {
    if (!window.equals(parentWindow)) {
        driver.switchTo().window(window);
        break;
    }
}

// Perform actions on new window
System.out.println("New window title: " + driver.getTitle());

// Close new window and switch back
driver.close();
driver.switchTo().window(parentWindow);
```

### ⚡ Remember
> `getWindowHandle()` = current window | `getWindowHandles()` = all windows (Set) | `switchTo().window(handle)` to switch | Close child with `close()`, switch back to parent | Always store parent handle before clicking

---

<a id="q15"></a>
## Q15. Finding palindrome words from a given sentence

### 💻 Code
```java
public List<String> findPalindromes(String sentence) {
    return Arrays.stream(sentence.split("\\s+"))
        .filter(word -> {
            String clean = word.toLowerCase().replaceAll("[^a-z]", "");
            return clean.length() > 1 && clean.equals(new StringBuilder(clean).reverse().toString());
        })
        .collect(Collectors.toList());
}

// "madam went to racecar event at noon"
// → ["madam", "racecar", "noon"]
```

### ⚡ Remember
> Split sentence by spaces | Clean non-alpha chars | Reverse and compare | Filter single-char words | Stream API for clean code | Common coding question in automation interviews
