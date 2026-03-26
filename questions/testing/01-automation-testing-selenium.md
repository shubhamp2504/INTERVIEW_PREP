# 🧪 Automation Testing — Selenium, API, CI/CD & Framework Design

> Consolidated from unnamed automation testing interview experiences (5+ YOE). Covers framework architecture, Selenium advanced scenarios, API testing, CI/CD integration, Playwright, and real-world debugging challenges.

> **Questions**: Q1–Q18 | **Difficulty**: Intermediate to Advanced

---

<a id="q1"></a>
## Q1. Automation Framework Architecture — design decisions

### 📝 One-Liner
A well-designed automation framework should be **modular** (POM), **data-driven** (external test data), **configurable** (env switching), **reportable** (Extent/Allure), and **CI-integrated** (Jenkins/GitHub Actions).

### 🔑 Quick Answer
**Key layers**: (1) **Page Objects** — locators + actions per page. (2) **Test Layer** — TestNG/JUnit test classes. (3) **Utilities** — waits, screenshots, logging, config reader. (4) **Data Layer** — Excel/JSON/DB for test data. (5) **Reporting** — Extent Reports/Allure. (6) **Config** — properties files for URLs, browser, env. (7) **Runner** — TestNG XML / Maven Surefire for execution.

```
framework/
├── src/main/java/
│   ├── pages/          ← Page Objects (LoginPage, DashboardPage)
│   ├── utils/          ← WebDriver factory, waits, screenshot, config
│   └── constants/      ← Locators, URLs, timeouts
├── src/test/java/
│   ├── tests/          ← TestNG test classes
│   ├── dataproviders/  ← Excel/JSON data readers
│   └── listeners/      ← TestNG listeners (screenshot on failure)
├── src/test/resources/
│   ├── testdata/       ← Excel, JSON test data files
│   ├── config.properties ← env, browser, URL configs
│   └── testng.xml      ← Suite configuration
├── reports/            ← Generated test reports
└── pom.xml             ← Maven dependencies
```

### 🗣️ Interview Script
"My framework follows POM with TestNG and Maven. Each page has a class with private locators and public action methods. A BasePage class provides reusable methods like click, type, and wait wrappers. Test data comes from Excel using Apache POI, with DataProviders feeding parameterized tests. We use config.properties for environment switching (dev/QA/staging). Extent Reports generate HTML reports with screenshots on failure via a TestNG listener. Jenkins triggers nightly regression runs on a Selenium Grid, with results published to a Slack channel."

### ⚡ Remember
> **POM** = page per class | **Data-Driven** = external data (Excel/JSON) | **Config-driven** = environment props | **Reporting** = Extent/Allure (screenshots on failure) | **CI/CD** = Jenkins/GitHub Actions | Keep tests independent — no shared state between tests

---

<a id="q2"></a>
## Q2. CI/CD integration for test automation — Jenkins pipeline

### 📝 One-Liner
Automate test execution in CI/CD pipelines: Jenkins/GitHub Actions triggers Maven test runs on code push/merge/schedule, runs on Selenium Grid (or Docker), publishes reports, and gates deployment on test results.

### 💻 Code
```groovy
// Jenkinsfile — declarative pipeline
pipeline {
    agent any
    tools { maven 'Maven-3.9' }
    parameters {
        choice(name: 'ENV', choices: ['dev', 'qa', 'staging'], description: 'Test environment')
        choice(name: 'SUITE', choices: ['smoke', 'regression'], description: 'Test suite')
    }
    stages {
        stage('Checkout') { steps { git 'https://github.com/org/automation.git' } }
        stage('Run Tests') {
            steps {
                sh "mvn clean test -Denv=${params.ENV} -Dsuite=${params.SUITE} -Dbrowser=chrome-headless"
            }
        }
        stage('Publish Report') {
            steps {
                publishHTML(target: [reportDir: 'reports', reportFiles: 'TestReport.html', reportName: 'Test Report'])
            }
            post {
                always { archiveArtifacts artifacts: 'reports/**', fingerprint: true }
                failure { slackSend channel: '#qa-alerts', message: "Tests FAILED: ${env.BUILD_URL}" }
            }
        }
    }
}
```

### ⚡ Remember
> **Triggers**: on push, on merge request, scheduled (nightly regression) | **Parameterized**: environment + suite selection | **Headless** browsers in CI (no UI) | **Reports**: publish + archive | **Notifications**: Slack/email on failure | Gate deployments on test pass rate

---

<a id="q3"></a>
## Q3. API testing approach — RestAssured framework design

### 📝 One-Liner
API testing validates **status codes, response body, headers, response time, and schema**. Design: RequestSpecification for reusable config, POJOs for request/response mapping, TestNG DataProvider for data-driven tests, and JSON Schema validation.

### 💻 Code
```java
// Reusable request spec
RequestSpecification baseSpec = new RequestSpecBuilder()
    .setBaseUri(ConfigReader.get("api.baseUrl"))
    .setContentType(ContentType.JSON)
    .addHeader("Authorization", "Bearer " + TokenManager.getToken())
    .addFilter(new RequestLoggingFilter())
    .addFilter(new ResponseLoggingFilter())
    .build();

// POST + validate
@Test
public void createUser() {
    UserRequest payload = new UserRequest("Alice", "alice@test.com");
    UserResponse response = given()
        .spec(baseSpec)
        .body(payload)
    .when()
        .post("/users")
    .then()
        .statusCode(201)
        .body("name", equalTo("Alice"))
        .body("id", notNullValue())
        .body(matchesJsonSchemaInClasspath("schemas/user-response.json"))
        .time(lessThan(2000L)) // Response time < 2s
    .extract().as(UserResponse.class);

    // Use response in next test
    assertNotNull(response.getId());
}
```

### ⚡ Remember
> **RequestSpecification** for reusable config | **POJO** for type-safe request/response | **JSON Schema validation** with `json-schema-validator` | **BDD style**: given-when-then | Validate: status + body + headers + time + schema | Logging filters for debugging

---

<a id="q4"></a>
## Q4. How to validate third-party API responses

### 📝 One-Liner
Validate: (1) **Contract** — response schema matches expected structure. (2) **Data integrity** — values are correct types and ranges. (3) **Error handling** — 4xx/5xx responses handled gracefully. (4) **Performance** — response time within SLA. (5) **Edge cases** — null fields, empty arrays, special characters.

### 🔑 Quick Answer
**Validation strategy**: (1) JSON Schema validation for structure. (2) Field-level assertions (types, ranges, enums). (3) Negative testing (invalid inputs, missing fields). (4) Rate limit testing (429 handling). (5) Timeout testing (slow responses). (6) Mock/stub the API for flaky external tests (WireMock). (7) Contract testing with Pact for consumer-driven validation.

### ⚡ Remember
> JSON Schema = structural validation | WireMock/MockServer for mocking external APIs in tests | Pact for consumer-driven contract testing | Always test error scenarios (4xx, 5xx, timeouts) | Don't couple tests to live third-party APIs (fragile)

---

<a id="q5"></a>
## Q5. Git rebase vs merge, stash, and branching strategies

### 📝 One-Liner
**Merge** = preserves history (merge commit). **Rebase** = linear history (replays commits on top). **Stash** = temporarily save uncommitted changes. Use rebase for feature branches (clean history), merge for main branches (preserve context).

### 💻 Code
```bash
# Rebase — linear history
git checkout feature-branch
git rebase main              # Replay feature commits on top of main
git checkout main
git merge feature-branch     # Fast-forward merge (linear)

# Merge — merge commit
git checkout main
git merge feature-branch     # Creates merge commit

# Stash — save WIP changes
git stash                     # Save uncommitted changes
git stash list                # List stashes
git stash pop                 # Apply + delete latest stash
git stash apply stash@{1}    # Apply specific stash (keep in list)

# Interactive rebase — squash commits
git rebase -i HEAD~3          # Squash/edit last 3 commits
```

### ⚡ Remember
> **Rebase** for clean feature branch history | **Merge** for main/develop branches | Never rebase shared/public branches | **Stash** for context switching | **Squash** commits before merging (clean PR) | **Cherry-pick** for selective commit application

---

<a id="q6"></a>
## Q6. Playwright hooks and test lifecycle

### 📝 One-Liner
Playwright provides **beforeAll** (once per file), **beforeEach** (before each test), **afterEach** (after each test), **afterAll** (once per file) hooks. Use fixtures for reusable setup (page, context, authenticated user).

### 💻 Code
```javascript
// Playwright test with hooks
const { test, expect } = require('@playwright/test');

test.describe('Login Tests', () => {
    test.beforeAll(async () => {
        // One-time setup: seed database, start server
    });

    test.beforeEach(async ({ page }) => {
        await page.goto('https://app.example.com/login');
    });

    test.afterEach(async ({ page }, testInfo) => {
        if (testInfo.status !== testInfo.expectedStatus) {
            await page.screenshot({ path: `screenshots/${testInfo.title}.png` });
        }
    });

    test('valid login', async ({ page }) => {
        await page.fill('#username', 'admin');
        await page.fill('#password', 'admin123');
        await page.click('button[type="submit"]');
        await expect(page).toHaveURL('/dashboard');
    });

    test('invalid login shows error', async ({ page }) => {
        await page.fill('#username', 'invalid');
        await page.fill('#password', 'wrong');
        await page.click('button[type="submit"]');
        await expect(page.locator('.error')).toBeVisible();
    });
});
```

### ⚡ Remember
> **Playwright** = auto-wait, multi-browser, fast | Fixtures: `{ page, context, browser }` auto-provided | `test.describe` for grouping | Screenshot on failure in `afterEach` | `expect(locator)` has auto-retry assertions | Parallel by default

---

<a id="q7"></a>
## Q7. Handling date pickers in automated tests

### 📝 One-Liner
Three approaches: (1) **Direct input** — sendKeys if the field accepts keyboard input. (2) **JavaScript injection** — set value via JS for readonly fields. (3) **Calendar UI navigation** — click through month/year and select the date. Prefer direct input for reliability.

### ⚡ Remember
> Try sendKeys first (simplest, most reliable) | JS: `executeScript("arguments[0].value = '2025-03-15'", element)` | Calendar UI: navigate month → click day (fragile, last resort) | Handle different formats (MM/DD/YYYY vs DD/MM/YYYY) | See company-specific/14 Q44 for detailed code

---

<a id="q8"></a>
## Q8. Scrolling strategies in Selenium

### 📝 One-Liner
**JavascriptExecutor** for programmatic scrolling (most reliable). **Actions.scrollToElement()** in Selenium 4. **scrollIntoView** for specific elements. Choose based on what needs to be in view before interaction.

### ⚡ Remember
> `scrollIntoView(true)` = scroll element to top | `scrollIntoView(false)` = scroll to bottom | `window.scrollTo(0, document.body.scrollHeight)` = page bottom | Selenium 4: `Actions.scrollToElement()` | Always scroll before interacting with off-screen elements

---

<a id="q9"></a>
## Q9. Handling 401 Unauthorized and OAuth token refresh

### 📝 One-Liner
When API returns 401: (1) Check if access token expired. (2) Use refresh token to get new access token. (3) Retry original request with new token. Implement as an **interceptor/retry mechanism** in your test framework.

### 💻 Code
```java
public class TokenManager {
    private static String accessToken;
    private static String refreshToken;

    public static String getToken() {
        if (isTokenExpired(accessToken)) {
            refreshAccessToken();
        }
        return accessToken;
    }

    private static void refreshAccessToken() {
        Response response = given()
            .contentType(ContentType.JSON)
            .body(Map.of("refresh_token", refreshToken, "grant_type", "refresh_token"))
        .when()
            .post("/auth/token")
        .then()
            .statusCode(200)
        .extract().response();

        accessToken = response.jsonPath().getString("access_token");
        refreshToken = response.jsonPath().getString("refresh_token");
    }

    private static boolean isTokenExpired(String token) {
        // Decode JWT and check exp claim
        // Or simply check if last request returned 401
        return token == null || /* JWT exp check */;
    }
}
```

### ⚡ Remember
> Check token expiry BEFORE request (proactive) | Intercept 401 → refresh → retry (reactive) | Store tokens securely | Refresh token has longer TTL than access token | Test both: valid token, expired token, invalid refresh token | OAuth 2.0 flow: Authorization Code (web), Client Credentials (API-to-API)

---

<a id="q10"></a>
## Q10. Top challenges in test automation and solutions

### 🔑 Quick Answer
| Challenge | Solution |
|-----------|----------|
| Flaky tests | Explicit waits, stable locators, isolated test data |
| Dynamic elements | Contains/starts-with selectors, data-testid, relative locators |
| Test data management | Factory pattern, API setup/teardown, isolated per test |
| Environment differences | Config-driven, containerized (Docker) |
| Slow test execution | Parallel execution, headless browser, selective suites |
| Maintenance cost | POM pattern, DRY utilities, review locator strategy |
| CI/CD integration | Docker Selenium Grid, retry mechanisms, proper reporting |
| Cross-browser issues | Selenium Grid, BrowserStack/Sauce Labs service |

### ⚡ Remember
> Flaky tests = #1 problem (fix root cause, not retry) | Maintenance cost = design well upfront (POM, reusable utils) | Test data isolation = each test creates its own data | Parallel execution = TestNG parallel + Selenium Grid | Docker = consistent environment

---

<a id="q11"></a>
## Q11. Building an API automation framework from scratch

### 📝 One-Liner
Stack: **RestAssured** + **TestNG** + **Maven** + **POJO models** + **Allure/Extent Reports**. Structure: base request specs, endpoint constants, POJO for request/response, utility classes for auth/data, TestNG suites for organization.

### 🔑 Quick Answer
**Implementation steps**: (1) Maven project with RestAssured + TestNG dependencies. (2) `BaseTest` class with `RequestSpecification` setup. (3) Constants/Enum for endpoints. (4) POJOs for request/response payloads (Jackson deserialization). (5) `TokenManager` for authentication. (6) Data-driven tests with TestNG DataProvider + JSON files. (7) JSON Schema validation for contract testing. (8) Allure/Extent reporting. (9) Jenkinsfile for CI integration. (10) Logging with Log4j2.

### ⚡ Remember
> RestAssured + TestNG + Maven = standard stack | POJO models for type safety | JSON Schema for contract validation | Separate config per environment | Logging for debugging | Start simple, add complexity as needed

---

<a id="q12"></a>
## Q12. Parallel test execution — configuration and pitfalls

### 💻 Code
```xml
<!-- testng.xml — parallel execution -->
<suite name="Regression" parallel="tests" thread-count="4">
    <test name="ChromeTests">
        <parameter name="browser" value="chrome"/>
        <classes>
            <class name="com.tests.LoginTest"/>
            <class name="com.tests.SearchTest"/>
        </classes>
    </test>
    <test name="FirefoxTests">
        <parameter name="browser" value="firefox"/>
        <classes>
            <class name="com.tests.LoginTest"/>
        </classes>
    </test>
</suite>
```
```java
// Thread-safe WebDriver management
public class DriverManager {
    private static final ThreadLocal<WebDriver> driver = new ThreadLocal<>();

    public static WebDriver getDriver() { return driver.get(); }
    public static void setDriver(WebDriver d) { driver.set(d); }
    public static void quit() {
        if (driver.get() != null) { driver.get().quit(); driver.remove(); }
    }
}
```

### ⚡ Remember
> **ThreadLocal** = must for parallel execution (one driver per thread) | `parallel="tests|classes|methods"` in testng.xml | Isolated test data per thread | Thread-safe reporting | Don't share state between parallel tests | Selenium Grid for distributed execution

---

<a id="q13"></a>
## Q13. TestNG DataProvider — data-driven testing

### 💻 Code
```java
@DataProvider(name = "loginData")
public Object[][] loginTestData() {
    return new Object[][] {
        { "admin", "admin123", true },
        { "user", "user123", true },
        { "invalid", "wrong", false },
        { "", "", false }
    };
}

@Test(dataProvider = "loginData")
public void testLogin(String username, String password, boolean expectedSuccess) {
    LoginPage loginPage = new LoginPage(driver);
    loginPage.login(username, password);
    if (expectedSuccess) {
        Assert.assertTrue(new DashboardPage(driver).isDisplayed());
    } else {
        Assert.assertTrue(loginPage.isErrorDisplayed());
    }
}

// External data source (Excel)
@DataProvider(name = "excelData")
public Object[][] readFromExcel() {
    return ExcelUtils.readData("testdata/login.xlsx", "Sheet1");
}
```

### ⚡ Remember
> `@DataProvider` returns `Object[][]` | `@Test(dataProvider = "name")` to link | External data: Excel (Apache POI), JSON (Jackson), CSV | Each row = one test execution | Name DataProviders descriptively | `@DataProvider(parallel = true)` for parallel data runs

---

<a id="q14"></a>
## Q14. Coding: Count character frequency in a string

### 💻 Code
```java
// Java — character frequency
public Map<Character, Integer> charFrequency(String str) {
    Map<Character, Integer> freq = new LinkedHashMap<>();
    for (char c : str.toCharArray()) {
        freq.merge(c, 1, Integer::sum);
    }
    return freq;
}

// Streams approach
public Map<Character, Long> charFrequencyStream(String str) {
    return str.chars()
        .mapToObj(c -> (char) c)
        .collect(Collectors.groupingBy(c -> c, LinkedHashMap::new, Collectors.counting()));
}

// "hello" → {h=1, e=1, l=2, o=1}
```

### ⚡ Remember
> `Map.merge()` = cleanest imperative approach | `Collectors.groupingBy()` for streams | `LinkedHashMap` preserves insertion order | Handle case-sensitivity (toLowerCase first if needed)

---

<a id="q15"></a>
## Q15. Coding: Reverse a string without using built-in methods

### 💻 Code
```java
// Two-pointer approach
public String reverse(String str) {
    char[] chars = str.toCharArray();
    int left = 0, right = chars.length - 1;
    while (left < right) {
        char temp = chars[left];
        chars[left++] = chars[right];
        chars[right--] = temp;
    }
    return new String(chars);
}

// Recursive approach
public String reverseRecursive(String str) {
    if (str.length() <= 1) return str;
    return reverseRecursive(str.substring(1)) + str.charAt(0);
}
```

### ⚡ Remember
> Two-pointer = O(n) time, O(n) space (char array) | Recursive = O(n²) due to String concatenation | `StringBuilder.reverse()` in production | Interviewers want to see algorithm thinking, not built-in methods

---

<a id="q16"></a>
## Q16. Coding: Find duplicate elements in an array

### 💻 Code
```java
// Using HashSet
public List<Integer> findDuplicates(int[] arr) {
    Set<Integer> seen = new HashSet<>();
    Set<Integer> duplicates = new LinkedHashSet<>();
    for (int num : arr) {
        if (!seen.add(num)) {
            duplicates.add(num);
        }
    }
    return new ArrayList<>(duplicates);
}

// Streams approach
public List<Integer> findDuplicatesStream(int[] arr) {
    return Arrays.stream(arr).boxed()
        .collect(Collectors.groupingBy(n -> n, Collectors.counting()))
        .entrySet().stream()
        .filter(e -> e.getValue() > 1)
        .map(Map.Entry::getKey)
        .collect(Collectors.toList());
}
```

### ⚡ Remember
> `HashSet.add()` returns false if already present | O(n) time, O(n) space | Streams: `groupingBy + counting + filter` | For sorted arrays: compare adjacent elements | Common follow-up: find first non-repeating element

---

<a id="q17"></a>
## Q17. Handling Shadow DOM elements in Selenium

### 📝 One-Liner
Shadow DOM encapsulates component internals — regular locators can't find elements inside a shadow root. In **Selenium 4**: use `getShadowRoot()` method. In Selenium 3: use JavascriptExecutor to pierce the shadow DOM.

### 💻 Code
```java
// Selenium 4 — built-in support
WebElement shadowHost = driver.findElement(By.cssSelector("my-component"));
SearchContext shadowRoot = shadowHost.getShadowRoot();
WebElement innerElement = shadowRoot.findElement(By.cssSelector(".inner-button"));
innerElement.click();

// Selenium 3 — JavaScript approach
WebElement innerElement = (WebElement) ((JavascriptExecutor) driver)
    .executeScript(
        "return document.querySelector('my-component').shadowRoot.querySelector('.inner-button')"
    );
innerElement.click();
```

### ⚡ Remember
> Selenium 4: `shadowHost.getShadowRoot()` (clean API) | Selenium 3: JS `shadowRoot.querySelector()` | Shadow DOM = web components encapsulation | Can't use XPath inside shadow DOM (CSS only) | Nested shadows: chain `getShadowRoot()` calls

---

<a id="q18"></a>
## Q18. BDD with Cucumber — end-to-end framework integration

### 📝 One-Liner
**Cucumber** enables BDD (Behavior-Driven Development) — write tests in Gherkin (Given/When/Then), map to Java step definitions, and execute with JUnit/TestNG runner. Non-technical stakeholders can read and validate scenarios.

### 💻 Code
```gherkin
# login.feature
Feature: User Login
  As a user, I want to login so that I can access my dashboard

  @smoke
  Scenario: Successful login with valid credentials
    Given I am on the login page
    When I enter username "admin" and password "admin123"
    And I click the login button
    Then I should be redirected to the dashboard
    And I should see welcome message "Welcome, admin"

  @regression
  Scenario Outline: Login with multiple credentials
    Given I am on the login page
    When I enter username "<user>" and password "<pass>"
    Then I should see "<result>"
    Examples:
      | user    | pass     | result    |
      | admin   | admin123 | Dashboard |
      | invalid | wrong    | Error     |
```
```java
// Step definitions
public class LoginSteps {
    LoginPage loginPage;
    DashboardPage dashboardPage;

    @Given("I am on the login page")
    public void navigateToLogin() {
        loginPage = new LoginPage(DriverManager.getDriver());
        loginPage.open();
    }

    @When("I enter username {string} and password {string}")
    public void enterCredentials(String user, String pass) {
        loginPage.enterUsername(user);
        loginPage.enterPassword(pass);
    }

    @Then("I should be redirected to the dashboard")
    public void verifyDashboard() {
        dashboardPage = new DashboardPage(DriverManager.getDriver());
        Assert.assertTrue(dashboardPage.isDisplayed());
    }
}
```

### ⚡ Remember
> Gherkin: Given (precondition) / When (action) / Then (assertion) | `@CucumberOptions(glue, features, plugin)` | Tags: `@smoke`, `@regression` for selective execution | Scenario Outline + Examples = parameterized BDD | Step reuse across features | Hooks (`@Before`, `@After`) for setup/teardown
