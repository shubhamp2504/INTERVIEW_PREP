# 🏢 Accenture — Multi-Role Interview Experiences

> Three distinct Accenture interview experiences covering **Snowflake Data Warehouse** (11 Qs), **React.js Developer** (13 Qs), and **Selenium Java Automation** (20 Qs — 44 total questions across 3 roles).

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

# Section A — Snowflake Data Warehouse (11 Questions)

> Role: Data Engineer / Snowflake Developer | Focus: Snowflake architecture, table types, warehousing concepts, SQL queries

---

<a id="q1"></a>
## Q1. What is Snowpipe?

### 📝 One-Liner
Snowpipe is Snowflake's continuous **serverless data ingestion** service that automatically loads data from staged files (S3/Azure Blob/GCS) into tables within seconds of file arrival — no manual COPY INTO needed.

### 🔑 Quick Answer
Snowpipe uses **event notifications** (S3 SQS, Azure Event Grid) or **REST API calls** to detect new files in a stage and automatically loads them using a defined COPY INTO statement in a pipe definition. It's serverless — Snowflake manages compute. Billing is per-second based on actual data loaded, not warehouse runtime.

(Snowpipe ek continuous serverless service hai jo automatically files ko stage se table mein load karta hai — file arrive hote hi seconds mein. Manual load ki zaroorat nahi.)

### 💻 Code
```sql
-- Create a pipe for auto-ingestion
CREATE OR REPLACE PIPE my_pipe
  AUTO_INGEST = TRUE
AS
  COPY INTO my_table
  FROM @my_s3_stage
  FILE_FORMAT = (TYPE = 'CSV' SKIP_HEADER = 1);

-- Check pipe status
SELECT SYSTEM$PIPE_STATUS('my_pipe');

-- Check load history
SELECT * FROM TABLE(INFORMATION_SCHEMA.COPY_HISTORY(
  TABLE_NAME => 'MY_TABLE', START_TIME => DATEADD(HOUR, -1, CURRENT_TIMESTAMP())
));
```

### ⚡ Remember
> AUTO_INGEST = TRUE → event-driven | Serverless (no warehouse needed) | Near real-time (~seconds) | Pay per-second of compute | Alternative to scheduled COPY INTO | Best for streaming/continuous loads

---

<a id="q2"></a>
## Q2. Types of tables in Snowflake

### 📝 One-Liner
Three types: **Permanent** (default, full Time Travel + Fail-safe), **Transient** (Time Travel only, no Fail-safe — cheaper), **Temporary** (session-scoped, auto-dropped, no Fail-safe).

### 🔑 Quick Answer
| Feature | Permanent | Transient | Temporary |
|---------|-----------|-----------|-----------|
| Time Travel | Up to 90 days | Up to 1 day | Up to 1 day |
| Fail-safe | 7 days | ❌ None | ❌ None |
| Scope | Persists forever | Persists forever | Session only |
| Cost | Highest (storage for TT + FS) | Medium | Lowest |
| Use Case | Production/critical data | ETL staging, intermediate | Session-specific scratch data |

(Permanent = full protection, Transient = kam protection lekin cheaper, Temporary = sirf session tak.)

### ⚡ Remember
> **Permanent** = default, most expensive (TT + Fail-safe) | **Transient** = no Fail-safe, cheaper staging tables | **Temporary** = session-scoped, auto-dropped | Choose based on data criticality and cost

---

<a id="q3"></a>
## Q3. Difference between Permanent and Transient tables

### 📝 One-Liner
Permanent tables have **7-day Fail-safe** (Snowflake-managed disaster recovery) after Time Travel expires — Transient tables have **no Fail-safe**, saving storage costs but losing that safety net.

### ⚡ Remember
> Permanent = Time Travel + Fail-safe (7 days) | Transient = Time Travel only (max 1 day), no Fail-safe | Use Transient for staging/intermediate tables where data loss is acceptable | Permanent for critical production data

---

<a id="q4"></a>
## Q4. Standard View vs Secure View in Snowflake

### 📝 One-Liner
**Standard View** = regular SQL view (definition visible via SHOW VIEWS). **Secure View** = hides view definition and optimizer details — query plan not exposed to users, data access controlled.

### 🔑 Quick Answer
| Feature | Standard View | Secure View |
|---------|--------------|-------------|
| Definition visible | Yes (SHOW VIEWS) | No (hidden) |
| Query optimizer bypass | No | Yes (optimizer can't push predicates through) |
| Performance | Better (optimizer can optimize fully) | Slightly slower (limited optimization) |
| Use case | Internal team | Multi-tenant, shared data, data marketplace |

```sql
CREATE OR REPLACE SECURE VIEW customer_view AS
SELECT id, name, email FROM customers WHERE region = CURRENT_ROLE();
```

### ⚡ Remember
> Secure View = `CREATE SECURE VIEW` | Hides SQL definition + optimizer details | Use for data sharing / multi-tenant | Slight performance cost | Standard View for internal use

---

<a id="q5"></a>
## Q5. How to recover a dropped table?

### 💻 Code
```sql
-- Drop table (goes to Time Travel)
DROP TABLE my_table;

-- Recover using UNDROP (within Time Travel period)
UNDROP TABLE my_table;

-- If Time Travel expired, cannot recover Transient/Temporary
-- Permanent tables: Snowflake support may recover from Fail-safe (7 days)

-- Clone from a point in time (alternative)
CREATE TABLE my_table_recovered CLONE my_table
  AT(TIMESTAMP => '2024-01-15 10:00:00'::TIMESTAMP);
```

### ⚡ Remember
> **UNDROP TABLE** within Time Travel window | Permanent tables: Fail-safe as last resort (contact Snowflake support) | `CLONE AT TIMESTAMP` for point-in-time recovery | Transient/Temporary = no Fail-safe recovery

---

<a id="q6"></a>
## Q6. What is a Virtual Warehouse in Snowflake?

### 📝 One-Liner
A Virtual Warehouse is Snowflake's **compute cluster** (collection of compute nodes) that executes queries and DML operations — it's completely separate from storage, enabling independent scaling of compute and storage.

### 🔑 Quick Answer
Key features: (1) **Sizes**: X-Small to 6X-Large (each size doubles compute + cost). (2) **Auto-suspend**: Shuts down after idle period (saves credits). (3) **Auto-resume**: Starts automatically on query execution. (4) **Multi-cluster**: Scales out horizontally for concurrency. (5) **No data stored** — pure compute, reads from shared storage layer.

(Virtual Warehouse ek compute cluster hai — storage se completely alag. Query run karna hai toh warehouse chahiye, data store karna hai toh storage.)

### ⚡ Remember
> Warehouse = compute only (no data) | Separate from storage | Auto-suspend + Auto-resume for cost savings | Size up for complex queries, scale out for concurrency | Pay per-second while running

---

<a id="q7"></a>
## Q7. Scaling Up vs Scaling Out in Snowflake

### 📝 One-Liner
**Scaling Up** = increase warehouse size (X-Small → Large) for **complex/slow queries**. **Scaling Out** = add more clusters (multi-cluster warehouse) for **more concurrent users**.

### 🔑 Quick Answer
| | Scaling Up | Scaling Out |
|---|-----------|-------------|
| What changes | Warehouse size (more nodes per cluster) | Number of clusters |
| Solves | Slow/complex queries | Concurrency (queue waits) |
| Cost | 2× per size increase | Credits × number of clusters |
| How to | `ALTER WAREHOUSE SET WAREHOUSE_SIZE = 'LARGE'` | `ALTER WAREHOUSE SET MAX_CLUSTER_COUNT = 5` |
| Best for | ETL, large scans, complex joins | Peak hours, many users |

### ⚡ Remember
> **Up** = bigger warehouse for complex queries | **Out** = more clusters for more users | Multi-cluster = auto-scales out based on queue depth | Scale up for ETL/heavy workloads, scale out for dashboards/BI concurrency

---

<a id="q8"></a>
## Q8. Unique Snowflake data types

### 📝 One-Liner
Snowflake supports **VARIANT** (semi-structured: JSON/XML/Avro), **OBJECT** (key-value pairs), **ARRAY** (ordered collection), **GEOGRAPHY/GEOMETRY** (geospatial) — VARIANT is the most distinctive and frequently used.

### 💻 Code
```sql
-- VARIANT — semi-structured data
CREATE TABLE events (id INT, data VARIANT);
INSERT INTO events SELECT 1, PARSE_JSON('{"name": "click", "page": "/home", "meta": {"browser": "Chrome"}}');

-- Query nested JSON
SELECT data:name::STRING, data:meta.browser::STRING FROM events;
-- "click", "Chrome"

-- ARRAY
SELECT ARRAY_CONSTRUCT(1, 2, 3); -- [1, 2, 3]

-- OBJECT
SELECT OBJECT_CONSTRUCT('key1', 'value1', 'key2', 'value2');
```

### ⚡ Remember
> **VARIANT** = JSON/XML/Avro storage (most important) | Colon notation `data:field` for access | **ARRAY** and **OBJECT** for structured semi-data | `PARSE_JSON()`, `FLATTEN()` for processing | No ETL needed — load semi-structured data directly

---

<a id="q9"></a>
## Q9. SQL — Find second highest salary

### 💻 Code
```sql
-- Method 1: DENSE_RANK
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) WHERE rnk = 2;

-- Method 2: LIMIT/OFFSET
SELECT DISTINCT salary FROM employees ORDER BY salary DESC LIMIT 1 OFFSET 1;

-- Method 3: Subquery
SELECT MAX(salary) FROM employees WHERE salary < (SELECT MAX(salary) FROM employees);

-- Nth highest (parameterized)
SELECT salary FROM (
    SELECT salary, DENSE_RANK() OVER (ORDER BY salary DESC) AS rnk
    FROM employees
) WHERE rnk = :n;
```

### ⚡ Remember
> `DENSE_RANK()` = handles ties correctly | `LIMIT 1 OFFSET 1` = simpler but misses ties | Subquery approach = classic but less flexible | Use DENSE_RANK for production queries

---

<a id="q10"></a>
## Q10. SQL — Find departments with more than 5 employees

### 💻 Code
```sql
SELECT d.department_name, COUNT(e.employee_id) AS emp_count
FROM departments d
JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_name
HAVING COUNT(e.employee_id) > 5
ORDER BY emp_count DESC;
```

### ⚡ Remember
> GROUP BY + HAVING for aggregate filters | HAVING operates AFTER grouping (WHERE operates before) | JOIN to get department name from separate table

---

<a id="q11"></a>
## Q11. SQL — Find employees earning more than department average

### 💻 Code
```sql
-- Window function approach (Snowflake-optimized)
SELECT employee_name, department_id, salary, dept_avg
FROM (
    SELECT *, AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
    FROM employees
)
WHERE salary > dept_avg;

-- Correlated subquery approach
SELECT e.employee_name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary) FROM employees WHERE department_id = e.department_id
);
```

### ⚡ Remember
> Window function = `AVG() OVER (PARTITION BY dept)` — efficient, one pass | Correlated subquery = runs per row (slower) | Window function is preferred in Snowflake

---

# Section B — React.js Developer (13 Questions)

> Role: React.js Developer | Focus: JavaScript fundamentals, React concepts, state management, optimization

---

<a id="q12"></a>
## Q12. Difference between var, let, and const

### 📝 One-Liner
**var** = function-scoped, hoisted (initialized as undefined), can be redeclared. **let** = block-scoped, hoisted but in TDZ (Temporal Dead Zone). **const** = block-scoped, must be initialized, can't be reassigned (but objects/arrays can be mutated).

### 💻 Code
```javascript
// var — function-scoped, hoisted
console.log(x); // undefined (hoisted)
var x = 10;
var x = 20; // Redeclaration OK

// let — block-scoped, TDZ
// console.log(y); // ReferenceError: TDZ
let y = 10;
// let y = 20; // SyntaxError: already declared

// const — block-scoped, immutable binding
const z = 10;
// z = 20; // TypeError: assignment to constant
const arr = [1, 2];
arr.push(3); // OK — mutating object, not reassigning

// Block scope difference
if (true) {
    var a = 1;   // Leaks out of block
    let b = 2;   // Stays in block
    const c = 3; // Stays in block
}
console.log(a); // 1
// console.log(b); // ReferenceError
```

### ⚡ Remember
> **var** = function scope, hoisted, avoid in modern JS | **let** = block scope, reassignable | **const** = block scope, not reassignable (but mutable objects) | Default to `const`, use `let` when reassignment needed, never `var`

---

<a id="q13"></a>
## Q13. Hoisting and Temporal Dead Zone (TDZ)

### 📝 One-Liner
**Hoisting**: JS moves declarations to top of scope before execution — var is initialized as `undefined`, let/const are hoisted but NOT initialized (entering TDZ). **TDZ**: The zone between scope entry and declaration where accessing let/const throws `ReferenceError`.

### 💻 Code
```javascript
// var hoisting — initialized as undefined
console.log(a); // undefined
var a = 5;

// let/const TDZ — hoisted but not initialized
console.log(b); // ReferenceError: Cannot access 'b' before initialization
let b = 10;

// Function hoisting — entire function is hoisted
greet(); // "Hello!" — works
function greet() { console.log("Hello!"); }

// Function expression — NOT hoisted
// sayHi(); // TypeError: sayHi is not a function
var sayHi = function() { console.log("Hi!"); };
```

### ⚡ Remember
> **var** = hoisted + initialized as `undefined` | **let/const** = hoisted but in TDZ | **Function declarations** = fully hoisted | **Function expressions** = variable hoisting rules apply | TDZ prevents using variables before declaration

---

<a id="q14"></a>
## Q14. Promises vs async/await

### 📝 One-Liner
**Promises** = `.then()/.catch()` chaining for async operations. **async/await** = syntactic sugar over Promises — makes async code look synchronous, easier to read and debug.

### 💻 Code
```javascript
// Promise chaining
fetchUser(1)
    .then(user => fetchOrders(user.id))
    .then(orders => fetchDetails(orders[0].id))
    .then(details => console.log(details))
    .catch(err => console.error(err));

// async/await — same logic, cleaner
async function getOrderDetails() {
    try {
        const user = await fetchUser(1);
        const orders = await fetchOrders(user.id);
        const details = await fetchDetails(orders[0].id);
        console.log(details);
    } catch (err) {
        console.error(err);
    }
}

// Parallel execution with async/await
async function loadAll() {
    const [users, posts] = await Promise.all([
        fetchUsers(),
        fetchPosts()
    ]);
}
```

### ⚡ Remember
> async/await = syntactic sugar over Promises | try/catch for error handling | `Promise.all()` for parallel execution | await only inside async functions | Both return Promises under the hood

---

<a id="q15"></a>
## Q15. Destructuring in JavaScript

### 💻 Code
```javascript
// Object destructuring
const user = { name: "Alice", age: 25, city: "Mumbai" };
const { name, age, city: location } = user; // Rename: city → location

// Default values
const { role = "user" } = user; // "user" (doesn't exist in object)

// Nested destructuring
const response = { data: { users: [{ id: 1 }] } };
const { data: { users: [{ id }] } } = response; // id = 1

// Array destructuring
const [first, , third] = [1, 2, 3]; // Skip second element

// Function parameter destructuring
function greet({ name, age }) {
    console.log(`${name} is ${age}`);
}
greet(user);

// Rest operator
const { name: n, ...rest } = user; // rest = { age: 25, city: "Mumbai" }
```

### ⚡ Remember
> Object: `{ key }` extracts by name | Array: `[a, b]` extracts by position | Rename: `{ key: newName }` | Default: `{ key = default }` | Rest: `{ ...rest }` | Use in function params for clean APIs

---

<a id="q16"></a>
## Q16. Class Components vs Functional Components

### 📝 One-Liner
**Class Components** = ES6 class + `render()` + lifecycle methods (componentDidMount, etc.) + `this.state`. **Functional Components** = plain functions + Hooks (useState, useEffect) — simpler, no `this`, better performance, industry standard since React 16.8.

### 💻 Code
```jsx
// Class Component (legacy)
class Counter extends React.Component {
    state = { count: 0 };
    componentDidMount() { console.log("Mounted"); }
    render() {
        return <button onClick={() => this.setState({ count: this.state.count + 1 })}>
            {this.state.count}
        </button>;
    }
}

// Functional Component (modern)
function Counter() {
    const [count, setCount] = useState(0);
    useEffect(() => { console.log("Mounted"); }, []);
    return <button onClick={() => setCount(c => c + 1)}>{count}</button>;
}
```

### ⚡ Remember
> Functional + Hooks = modern standard | No `this` keyword needed | `useState` replaces `this.state` | `useEffect` replaces lifecycle methods | Class components still work but not recommended for new code

---

<a id="q17"></a>
## Q17. Prop drilling and solutions

### 📝 One-Liner
Prop drilling = passing props through multiple intermediate components that don't use them — leads to tightly coupled, hard-to-maintain code. Solutions: **Context API**, **Redux**, **Zustand**, **component composition**.

### 💻 Code
```jsx
// Problem: prop drilling
<App user={user}>
    <Header user={user}>           {/* doesn't use user, just passes */}
        <NavBar user={user}>       {/* doesn't use user, just passes */}
            <UserMenu user={user}> {/* actually uses user */}
            </UserMenu>
        </NavBar>
    </Header>
</App>

// Solution: Context API
const UserContext = createContext();
function App() {
    return (
        <UserContext.Provider value={user}>
            <Header /> {/* No props needed */}
        </UserContext.Provider>
    );
}
function UserMenu() {
    const user = useContext(UserContext); // Direct access
}
```

### ⚡ Remember
> Prop drilling = passing through components that don't need the data | Context API for global/shared state | Redux for complex state logic | Component composition (children pattern) for simple cases | Avoid over-using Context for frequently-changing data

---

<a id="q18"></a>
## Q18. Redux vs Context API — when to use which

### 📝 One-Liner
**Context API** = simple shared state (theme, auth, locale) — built-in, no extra library. **Redux** = complex state with frequent updates, middleware (async), DevTools debugging, time-travel — for large apps with predictable state management needs.

### 🔑 Quick Answer
| Feature | Context API | Redux (RTK) |
|---------|-------------|-------------|
| Setup | Minimal (built-in) | createSlice + configureStore |
| Performance | Re-renders all consumers on change | Selective re-renders with selectors |
| Middleware | None | Thunk, Saga, RTK Query |
| DevTools | Limited | Full time-travel debugging |
| Best for | Theme, auth, locale | Shopping cart, dashboard, complex forms |

### ⚡ Remember
> Context = simple, infrequent updates | Redux = complex, frequent updates with middleware | RTK Query = built-in data fetching | Don't use Redux for simple state | Don't use Context for frequently changing data (re-render perf)

---

<a id="q19"></a>
## Q19. React performance optimization techniques

### 📝 One-Liner
Key techniques: **React.memo** (skip re-renders), **useMemo/useCallback** (memoize values/functions), **lazy loading** (code splitting), **virtualization** (render only visible items), **proper key props**, **avoiding inline objects/functions**.

### 💻 Code
```jsx
// React.memo — skip re-render if props unchanged
const ExpensiveList = React.memo(({ items }) => {
    return items.map(item => <ListItem key={item.id} {...item} />);
});

// useMemo — memoize expensive computation
const sortedData = useMemo(() => {
    return data.sort((a, b) => a.name.localeCompare(b.name));
}, [data]);

// useCallback — memoize function reference
const handleClick = useCallback((id) => {
    setItems(prev => prev.filter(item => item.id !== id));
}, []);

// Lazy loading with Suspense
const Dashboard = React.lazy(() => import('./Dashboard'));
<Suspense fallback={<Spinner />}><Dashboard /></Suspense>
```

### ⚡ Remember
> `React.memo` = component-level memoization | `useMemo` = value memoization | `useCallback` = function memoization | Lazy loading = code splitting | Don't premature optimize — measure first with React DevTools Profiler

---

<a id="q20"></a>
## Q20. useReducer hook — when and why

### 📝 One-Liner
`useReducer` is an alternative to `useState` for **complex state logic** — especially when next state depends on previous state, state has multiple sub-values, or state transitions follow defined rules (like Redux pattern).

### 💻 Code
```jsx
const initialState = { count: 0, step: 1 };

function reducer(state, action) {
    switch (action.type) {
        case 'increment': return { ...state, count: state.count + state.step };
        case 'decrement': return { ...state, count: state.count - state.step };
        case 'setStep':   return { ...state, step: action.payload };
        case 'reset':     return initialState;
        default: throw new Error(`Unknown action: ${action.type}`);
    }
}

function Counter() {
    const [state, dispatch] = useReducer(reducer, initialState);
    return (
        <>
            <p>Count: {state.count}</p>
            <button onClick={() => dispatch({ type: 'increment' })}>+</button>
            <button onClick={() => dispatch({ type: 'reset' })}>Reset</button>
        </>
    );
}
```

### ⚡ Remember
> `useReducer(reducer, initialState)` → `[state, dispatch]` | Use when: complex state transitions, multiple sub-values, next-state-depends-on-prev | Reducer is pure function | `dispatch({ type, payload })` | Combine with Context for mini-Redux

---

<a id="q21"></a>
## Q21. Refs in React — useRef hook

### 📝 One-Liner
`useRef` creates a mutable ref object (`{ current: value }`) that persists across re-renders WITHOUT causing re-renders when changed — used for DOM access, storing mutable values, and preserving values between renders.

### 💻 Code
```jsx
function TextInput() {
    const inputRef = useRef(null);
    const renderCount = useRef(0);

    useEffect(() => { renderCount.current++; }); // No re-render

    const focusInput = () => inputRef.current.focus();

    return (
        <>
            <input ref={inputRef} />
            <button onClick={focusInput}>Focus</button>
            <p>Renders: {renderCount.current}</p>
        </>
    );
}
```

### ⚡ Remember
> `useRef` = mutable container, `.current` holds value | Doesn't trigger re-render | DOM access: `ref={myRef}` on JSX | Store interval IDs, previous values, render counts | Not for state that affects UI (use useState)

---

<a id="q22"></a>
## Q22. React Router — client-side routing

### 💻 Code
```jsx
import { BrowserRouter, Routes, Route, Link, useParams, Navigate } from 'react-router-dom';

function App() {
    return (
        <BrowserRouter>
            <nav>
                <Link to="/">Home</Link>
                <Link to="/users">Users</Link>
            </nav>
            <Routes>
                <Route path="/" element={<Home />} />
                <Route path="/users" element={<Users />} />
                <Route path="/users/:id" element={<UserDetail />} />
                <Route path="/admin" element={
                    isAuth ? <Admin /> : <Navigate to="/login" />
                } />
                <Route path="*" element={<NotFound />} />
            </Routes>
        </BrowserRouter>
    );
}

function UserDetail() {
    const { id } = useParams(); // Dynamic route param
    return <p>User ID: {id}</p>;
}
```

### ⚡ Remember
> React Router v6: `<Routes>` + `<Route>` | `useParams()` for dynamic segments | `<Navigate>` for redirects | `<Link>` for navigation (not `<a>`) | Protected routes with conditional rendering | `path="*"` for 404 catch-all

---

<a id="q23"></a>
## Q23. Build a counter program (coding task)

### 💻 Code
```jsx
function Counter() {
    const [count, setCount] = useState(0);

    return (
        <div style={{ textAlign: 'center', padding: '2rem' }}>
            <h1>Counter: {count}</h1>
            <button onClick={() => setCount(c => c + 1)}>Increment</button>
            <button onClick={() => setCount(c => c - 1)}>Decrement</button>
            <button onClick={() => setCount(0)}>Reset</button>
        </div>
    );
}
```

### ⚡ Remember
> Functional updater: `setCount(c => c + 1)` | Not `setCount(count + 1)` (stale closure risk) | Show increment, decrement, reset buttons | Production: add min/max bounds, step size

---

<a id="q24"></a>
## Q24. Lazy loading and code splitting in React

### 📝 One-Liner
`React.lazy()` + `Suspense` enables **code splitting** — splits the bundle into smaller chunks that load on-demand, reducing initial load time. Each lazy-loaded component becomes a separate JS chunk.

### 💻 Code
```jsx
// Lazy load heavy components
const Dashboard = React.lazy(() => import('./Dashboard'));
const Settings = React.lazy(() => import('./Settings'));

function App() {
    return (
        <Suspense fallback={<LoadingSpinner />}>
            <Routes>
                <Route path="/dashboard" element={<Dashboard />} />
                <Route path="/settings" element={<Settings />} />
            </Routes>
        </Suspense>
    );
}
```

### ⚡ Remember
> `React.lazy(() => import('./Component'))` | Wrap in `<Suspense fallback={...}>` | Route-level splitting is most impactful | Reduces initial bundle size | Named exports need intermediate re-export

---

# Section C — Selenium Java Automation (20 Questions)

> Role: Selenium Java Automation Engineer | Focus: Selenium WebDriver, Java OOP, TestNG, framework design, real-time scenarios

---

<a id="q25"></a>
## Q25. Types of locators in Selenium

### 📝 One-Liner
8 locator types: **id** (fastest, unique), **name**, **className**, **tagName**, **linkText**, **partialLinkText**, **cssSelector** (recommended), **xpath** (most flexible). Priority: id > css > xpath.

### ⚡ Remember
> **id** = fastest, most reliable | **CSS Selector** = fast, readable, recommended | **XPath** = most powerful/flexible but slower | Avoid: className (multiple values), tagName (too generic) | Use `data-testid` attributes for test-specific locators

---

<a id="q26"></a>
## Q26. XPath vs CSS Selector

### 📝 One-Liner
**CSS Selector** = faster execution, cleaner syntax, can't traverse up (parent). **XPath** = can traverse any direction (parent/sibling), supports text matching, more verbose.

### 💻 Code
```java
// CSS — faster, cleaner
driver.findElement(By.cssSelector("#login .btn-primary"));
driver.findElement(By.cssSelector("input[type='email']"));
driver.findElement(By.cssSelector("div.card > h3"));

// XPath — more flexible
driver.findElement(By.xpath("//button[text()='Submit']"));        // Text match
driver.findElement(By.xpath("//input[@id='email']/parent::div")); // Parent traverse
driver.findElement(By.xpath("//div[contains(@class, 'active')]")); // Contains
driver.findElement(By.xpath("//table//tr[3]/td[2]"));             // Table cell
```

### ⚡ Remember
> CSS = performance + readability | XPath = flexibility + text matching | Use CSS by default | Use XPath when: need text match, parent traversal, complex conditions | `contains()`, `starts-with()` = XPath functions

---

<a id="q27"></a>
## Q27. Implicit, Explicit, and Fluent Waits

### 💻 Code
```java
// Implicit Wait — global, applies to all findElement calls
driver.manage().timeouts().implicitlyWait(Duration.ofSeconds(10));

// Explicit Wait — specific condition for specific element
WebDriverWait wait = new WebDriverWait(driver, Duration.ofSeconds(15));
WebElement element = wait.until(ExpectedConditions.elementToBeClickable(By.id("submit")));

// Fluent Wait — polling interval + ignore exceptions
Wait<WebDriver> fluentWait = new FluentWait<>(driver)
    .withTimeout(Duration.ofSeconds(30))
    .pollingEvery(Duration.ofSeconds(2))
    .ignoring(NoSuchElementException.class);
WebElement el = fluentWait.until(d -> d.findElement(By.id("dynamic-element")));
```

### ⚡ Remember
> **Implicit** = global timeout for all findElements | **Explicit** = condition-based, specific element | **Fluent** = explicit + custom polling + ignored exceptions | Don't mix implicit + explicit (unpredictable) | Prefer explicit waits

---

<a id="q28"></a>
## Q28. Page Object Model (POM) in Selenium

### 📝 One-Liner
POM is a design pattern where each web page has a corresponding Java class containing **locators** (private) and **action methods** (public). Separates test logic from page interaction logic — improves maintainability.

### ⚡ Remember
> Each page = one class | Locators = private fields | Actions = public methods | Return next page object for fluent chaining | Base page class for common methods | Locator change → update one class, all tests still work

---

<a id="q29"></a>
## Q29. How to handle frames in Selenium

### 💻 Code
```java
// Switch by index
driver.switchTo().frame(0);

// Switch by name or ID
driver.switchTo().frame("frameName");

// Switch by WebElement
WebElement frameElement = driver.findElement(By.cssSelector("iframe.content"));
driver.switchTo().frame(frameElement);

// Nested frames — switch sequentially
driver.switchTo().frame("outerFrame");
driver.switchTo().frame("innerFrame");

// Switch back to parent frame
driver.switchTo().parentFrame();

// Switch back to main page (default content)
driver.switchTo().defaultContent();
```

### ⚡ Remember
> Must switch to frame before interacting with its elements | `parentFrame()` = one level up | `defaultContent()` = back to main page | Wait for frame before switching | Nested frames = switch sequentially

---

<a id="q30"></a>
## Q30. Generating Extent Reports

### 💻 Code
```java
// Setup in @BeforeTest
ExtentReports extent = new ExtentReports();
ExtentSparkReporter spark = new ExtentSparkReporter("reports/TestReport.html");
spark.config().setTheme(Theme.DARK);
spark.config().setDocumentTitle("Automation Report");
extent.attachReporter(spark);

// In test methods
ExtentTest test = extent.createTest("Login Test");
test.log(Status.INFO, "Navigating to login page");
test.pass("Login successful");
// On failure:
test.fail("Login failed", MediaEntityBuilder.createScreenCaptureFromPath("screenshot.png").build());

// In @AfterTest
extent.flush();
```

### ⚡ Remember
> ExtentReports v5 with SparkReporter | `flush()` required to write report | Attach screenshots on failure | Integrate with TestNG Listener for automatic logging | `createTest()` per test method

---

<a id="q31"></a>
## Q31. StaleElementReferenceException — cause and fix

### 📝 One-Liner
`StaleElementReferenceException` occurs when a previously found element is no longer attached to the DOM — due to page refresh, AJAX update, or navigation. The reference becomes "stale" (outdated).

### 💻 Code
```java
// Problem
WebElement button = driver.findElement(By.id("submit"));
// ... page refreshes / AJAX updates DOM ...
button.click(); // StaleElementReferenceException!

// Fix 1: Re-find the element
driver.findElement(By.id("submit")).click();

// Fix 2: Explicit wait for staleness to resolve
new WebDriverWait(driver, Duration.ofSeconds(10))
    .until(ExpectedConditions.refreshed(
        ExpectedConditions.elementToBeClickable(By.id("submit"))
    )).click();

// Fix 3: Retry mechanism
public void clickWithRetry(By locator, int maxRetries) {
    for (int i = 0; i < maxRetries; i++) {
        try {
            driver.findElement(locator).click();
            return;
        } catch (StaleElementReferenceException e) {
            if (i == maxRetries - 1) throw e;
        }
    }
}
```

### ⚡ Remember
> Cause: DOM changed after element was found | Fix: re-find element | `ExpectedConditions.refreshed()` waits for fresh reference | Retry pattern for AJAX-heavy pages | Don't store element references for long periods

---

<a id="q32"></a>
## Q32. == vs .equals() in Java

### 📝 One-Liner
**==** compares references (memory addresses) for objects, values for primitives. **.equals()** compares content/value (when overridden). For Strings: `==` may work due to String Pool but `.equals()` is correct.

### 💻 Code
```java
String a = "hello";
String b = "hello";
String c = new String("hello");

System.out.println(a == b);       // true (same pool reference)
System.out.println(a == c);       // false (different objects)
System.out.println(a.equals(c));  // true (same content)
```

### ⚡ Remember
> Always use `.equals()` for object comparison | `==` for primitives only | String Pool makes `==` misleading for Strings | Override `.equals()` + `.hashCode()` together in custom classes

---

<a id="q33"></a>
## Q33. OOP concepts briefly (for automation context)

### ⚡ Remember
> **Encapsulation** = private locators in POM | **Inheritance** = BasePage class | **Polymorphism** = method overloading (multiple click methods)/overriding | **Abstraction** = abstract BasePage methods | See Q13 in Section A for detailed code example

---

<a id="q34"></a>
## Q34. Method overloading vs overriding in Java

### 📝 One-Liner
**Overloading** = same method name, different parameters (compile-time polymorphism). **Overriding** = subclass redefines parent method with same signature (runtime polymorphism, `@Override`).

### 💻 Code
```java
// Overloading — same class, different params
public void click(By locator) { driver.findElement(locator).click(); }
public void click(WebElement element) { element.click(); }
public void click(By locator, int timeout) { /* wait then click */ }

// Overriding — subclass redefines parent method
class BasePage {
    public void navigate() { driver.get(baseUrl); }
}
class LoginPage extends BasePage {
    @Override
    public void navigate() { driver.get(baseUrl + "/login"); }
}
```

### ⚡ Remember
> **Overloading** = compile-time, same class, different params | **Overriding** = runtime, subclass, same signature + `@Override` | Overloading in automation: different click/type methods | Overriding: page-specific behavior

---

<a id="q35"></a>
## Q35. Types of exceptions in Java — checked vs unchecked

### 📝 One-Liner
**Checked** = compile-time, must handle (IOException, SQLException) — extends Exception. **Unchecked** = runtime, optional handling (NullPointerException, ArrayIndexOutOfBounds) — extends RuntimeException.

### ⚡ Remember
> **Checked** = compiler forces try/catch or throws | **Unchecked** = RuntimeException subclasses | Selenium: most exceptions are unchecked (NoSuchElementException, StaleElement) | IOException = checked (file operations)

---

<a id="q36"></a>
## Q36. List vs Set in Java

### 📝 One-Liner
**List** = ordered, allows duplicates, index-based access (ArrayList, LinkedList). **Set** = unordered (or sorted), no duplicates (HashSet, TreeSet, LinkedHashSet).

### ⚡ Remember
> **ArrayList** = ordered + duplicates (test data, element lists) | **HashSet** = unique values (deduplicate test data) | **LinkedHashSet** = unique + insertion order | **TreeSet** = unique + sorted | List for sequences, Set for uniqueness

---

<a id="q37"></a>
## Q37. Scenario — Click button by color attribute

### 💻 Code
```java
// Find button by background color CSS property
List<WebElement> buttons = driver.findElements(By.tagName("button"));
for (WebElement btn : buttons) {
    String bgColor = btn.getCssValue("background-color");
    if (bgColor.equals("rgba(255, 0, 0, 1)")) { // Red
        btn.click();
        break;
    }
}

// Or by style attribute with CSS selector
driver.findElement(By.cssSelector("button[style*='background-color: red']")).click();

// Or by class name if color-mapped to class
driver.findElement(By.cssSelector("button.btn-danger")).click();
```

### ⚡ Remember
> `getCssValue("background-color")` returns rgba format | CSS selector `[style*='...']` for inline styles | Prefer class-based selectors (`.btn-danger`) over computed styles | Computed color values may vary by browser

---

<a id="q38"></a>
## Q38. Handling dynamic elements (changing locators)

### 📝 One-Liner
Dynamic elements change IDs/classes on each load. Solutions: **partial attribute match** (contains/starts-with), **relative XPath**, **parent-child traversal**, **data attributes**, **explicit waits**.

### 💻 Code
```java
// XPath — contains/starts-with for dynamic IDs
// ID changes: "btn_12345" → "btn_67890"
driver.findElement(By.xpath("//button[starts-with(@id, 'btn_')]"));
driver.findElement(By.xpath("//button[contains(@id, 'btn')]"));

// CSS — partial attribute match
driver.findElement(By.cssSelector("button[id^='btn_']"));   // starts with
driver.findElement(By.cssSelector("button[id*='btn']"));    // contains
driver.findElement(By.cssSelector("button[id$='_submit']")); // ends with

// Relative locator — find near a stable element
driver.findElement(By.xpath("//label[text()='Email']/following-sibling::input"));

// Explicit wait for dynamic element to appear
new WebDriverWait(driver, Duration.ofSeconds(10))
    .until(ExpectedConditions.presenceOfElementLocated(By.xpath("//button[contains(@class, 'submit')]")));
```

### ⚡ Remember
> `contains()` / `starts-with()` for dynamic IDs | CSS: `^=` (starts), `$=` (ends), `*=` (contains) | Use parent/sibling traversal for stable context | Add `data-testid` attributes (best long-term solution) | Wait for element to stabilize

---

<a id="q39"></a>
## Q39. Handling multiple browser windows

### 📝 One-Liner
Use `getWindowHandles()` to get all window handles, iterate to find the new window, switch using `switchTo().window(handle)`. Same approach as Q14 — see detailed code there.

### ⚡ Remember
> `getWindowHandle()` = current | `getWindowHandles()` = all (Set) | Store parent before clicking | Switch → perform actions → close → switch back | See Q14 for full code example

---

<a id="q40"></a>
## Q40. Automate a login page scenario

### 💻 Code
```java
// Page Object
public class LoginPage {
    private WebDriver driver;
    private By usernameField = By.id("username");
    private By passwordField = By.id("password");
    private By loginButton = By.cssSelector("button[type='submit']");
    private By errorMessage = By.className("error-msg");

    public LoginPage(WebDriver driver) { this.driver = driver; }

    public DashboardPage login(String user, String pass) {
        driver.findElement(usernameField).sendKeys(user);
        driver.findElement(passwordField).sendKeys(pass);
        driver.findElement(loginButton).click();
        return new DashboardPage(driver);
    }

    public String getErrorMessage() {
        return new WebDriverWait(driver, Duration.ofSeconds(5))
            .until(ExpectedConditions.visibilityOfElementLocated(errorMessage))
            .getText();
    }
}

// Test
@Test
public void testValidLogin() {
    LoginPage loginPage = new LoginPage(driver);
    DashboardPage dashboard = loginPage.login("admin", "admin123");
    Assert.assertTrue(dashboard.isDisplayed());
}

@Test
public void testInvalidLogin() {
    LoginPage loginPage = new LoginPage(driver);
    loginPage.login("invalid", "wrong");
    Assert.assertEquals(loginPage.getErrorMessage(), "Invalid credentials");
}
```

### ⚡ Remember
> POM pattern: LoginPage class with locators + methods | Return next page object (DashboardPage) | Both positive + negative test cases | Explicit waits for elements | Assert expected outcome

---

<a id="q41"></a>
## Q41. Check if a string is a palindrome

### 💻 Code
```java
public boolean isPalindrome(String s) {
    String clean = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    return clean.equals(new StringBuilder(clean).reverse().toString());
}

// Without StringBuilder — two-pointer approach (interview preferred)
public boolean isPalindromeTwoPointer(String s) {
    String clean = s.toLowerCase().replaceAll("[^a-z0-9]", "");
    int left = 0, right = clean.length() - 1;
    while (left < right) {
        if (clean.charAt(left++) != clean.charAt(right--)) return false;
    }
    return true;
}
```

### ⚡ Remember
> Clean string first (lowercase + remove non-alphanumeric) | Two-pointer = O(n) time, O(1) space | StringBuilder.reverse = simpler, O(n) space | Handle edge cases: null, empty, single char

---

<a id="q42"></a>
## Q42. TestNG annotations — execution order

### 📝 One-Liner
`@BeforeSuite` → `@BeforeTest` → `@BeforeClass` → `@BeforeMethod` → `@Test` → `@AfterMethod` → `@AfterClass` → `@AfterTest` → `@AfterSuite`. Each "Before" has a corresponding "After".

### ⚡ Remember
> Suite > Test > Class > Method (hierarchy) | `@BeforeMethod` = runs before EACH @Test method | `@BeforeClass` = once per class | `@DataProvider` for parameterized tests | `@Test(priority = 1)` for ordering | `dependsOnMethods` for dependencies

---

<a id="q43"></a>
## Q43. Desired Capabilities in Selenium

### 📝 One-Liner
Desired Capabilities (now **Options** classes in Selenium 4) configure browser properties — headless mode, proxy, download path, incognito, SSL handling. Used to customize WebDriver behavior.

### 💻 Code
```java
// Selenium 4 — Options classes (replaces DesiredCapabilities)
ChromeOptions options = new ChromeOptions();
options.addArguments("--headless=new");
options.addArguments("--disable-gpu");
options.addArguments("--incognito");
options.addArguments("--window-size=1920,1080");

// Download path
HashMap<String, Object> prefs = new HashMap<>();
prefs.put("download.default_directory", "/path/to/downloads");
options.setExperimentalOption("prefs", prefs);

// SSL handling
options.setAcceptInsecureCerts(true);

WebDriver driver = new ChromeDriver(options);
```

### ⚡ Remember
> Selenium 4: Use `ChromeOptions`/`FirefoxOptions` (not DesiredCapabilities) | `--headless=new` for headless execution | `setAcceptInsecureCerts(true)` for SSL | Prefs for download directory | Options are browser-specific

---

<a id="q44"></a>
## Q44. Handling date picker in web applications

### 💻 Code
```java
// Method 1: Direct input (if editable)
WebElement dateInput = driver.findElement(By.id("datepicker"));
dateInput.clear();
dateInput.sendKeys("15/03/2025");
dateInput.sendKeys(Keys.ENTER);

// Method 2: Clear with JS (for readonly fields)
JavascriptExecutor js = (JavascriptExecutor) driver;
js.executeScript("arguments[0].removeAttribute('readonly')", dateInput);
js.executeScript("arguments[0].value = '2025-03-15'", dateInput);

// Method 3: Navigate calendar UI
driver.findElement(By.cssSelector(".calendar-trigger")).click(); // Open calendar
// Navigate to month/year
while (!driver.findElement(By.cssSelector(".month-label")).getText().equals("March 2025")) {
    driver.findElement(By.cssSelector(".next-month")).click();
}
driver.findElement(By.xpath("//td[@data-day='15']")).click(); // Select day
```

### ⚡ Remember
> Try `sendKeys()` first (simplest) | Remove readonly with JS if needed | For custom calendars: navigate month → click day | Prefer JS `value` setting over UI navigation when possible | Handle different date formats
