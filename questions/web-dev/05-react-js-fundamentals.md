# 🌐 React & JavaScript — Core Fundamentals

> Consolidated from multiple unnamed frontend interview experiences. Covers JavaScript fundamentals (closures, event loop, promises, ES6+) and React core concepts (hooks, state management, lifecycle, optimization) — the essential knowledge for any React developer interview.

> **Questions**: Q1–Q20 | **Difficulty**: Intermediate to Advanced

---

<a id="q1"></a>
## Q1. Closures in JavaScript — what, why, and real-world usage

### 📝 One-Liner
A closure is a function that **remembers and accesses** variables from its outer (lexical) scope even after the outer function has finished executing. Every function in JS creates a closure.

### 🔑 Quick Answer
When a function is defined inside another function, it "closes over" the outer function's variables. The inner function retains a reference to the outer scope — not a copy. This is how data privacy, factory functions, memoization, and module patterns work in JavaScript.

(Closure ek aisa function hai jo apne bahar ke scope ke variables ko yaad rakhta hai — chahe outer function execute ho chuka ho. Ye JavaScript ka fundamental concept hai.)

### 💻 Code
```javascript
// Basic closure
function counter() {
    let count = 0;  // Private variable
    return {
        increment: () => ++count,
        decrement: () => --count,
        getCount: () => count
    };
}
const c = counter();
c.increment(); // 1
c.increment(); // 2
c.getCount();  // 2
// count is NOT accessible from outside — encapsulated

// Closure in loops — classic gotcha
for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100); // 3, 3, 3 (var is shared)
}
for (let i = 0; i < 3; i++) {
    setTimeout(() => console.log(i), 100); // 0, 1, 2 (let creates new scope per iteration)
}

// Real-world: Debounce function
function debounce(fn, delay) {
    let timerId;  // Closed over
    return function (...args) {
        clearTimeout(timerId);
        timerId = setTimeout(() => fn.apply(this, args), delay);
    };
}
```

### 🗣️ Interview Script
"A closure gives a function access to its outer scope's variables even after the outer function returns. The classic example is a counter with a private count variable. Closures are fundamental to patterns like debounce/throttle, module pattern for data privacy, and callback-based async code. The common gotcha is the var-in-loop problem — since var is function-scoped, all closures share the same variable. Using let fixes this because let creates a new binding per iteration."

### ⚠️ Pitfalls
- **Stale closures in React**: `useEffect` callback captures stale state if dependency array is wrong
- **Memory leaks**: Closures prevent garbage collection of outer scope — large objects in outer scope stay alive
- **var in loops**: All iterations share the same variable (use `let` or IIFE)

### ⚡ Remember
> Closure = function + its lexical environment | Used in: debounce, throttle, memoize, module pattern, React hooks | Stale closure in React = wrong dependency array | `var` in loops = shared variable, `let` = new binding per iteration | Enables data privacy (private variables)

---

<a id="q2"></a>
## Q2. Event Loop — how asynchronous JavaScript works

### 📝 One-Liner
The Event Loop is the mechanism that allows single-threaded JavaScript to handle async operations. It continuously checks: (1) Execute **call stack** (synchronous code). (2) Process all **microtasks** (Promises, queueMicrotask). (3) Process one **macrotask** (setTimeout, setInterval, I/O).

### 🔑 Quick Answer
**Execution order**: Call Stack → Microtask Queue (Promises, `queueMicrotask`, `MutationObserver`) → Macrotask Queue (setTimeout, setInterval, `setImmediate`, I/O callbacks). Microtasks ALWAYS run before the next macrotask.

### 💻 Code
```javascript
console.log('1 — sync');

setTimeout(() => console.log('2 — macrotask'), 0);

Promise.resolve().then(() => console.log('3 — microtask'));

queueMicrotask(() => console.log('4 — microtask'));

console.log('5 — sync');

// Output: 1 → 5 → 3 → 4 → 2
// Sync first, then ALL microtasks, then macrotask
```

### 🗣️ Interview Script
"JavaScript runs on a single thread with an event loop. Synchronous code runs first on the call stack. When async operations complete, their callbacks go to task queues. The event loop checks: first the call stack (must be empty), then drains ALL microtasks (Promises), then picks ONE macrotask (setTimeout). This cycle repeats. That's why a Promise.then always runs before a setTimeout — even if the timeout is 0."

### ⚡ Remember
> **Single-threaded + event loop** = async without threads | **Order**: sync → microtasks → macrotask | **Microtasks**: Promise callbacks, queueMicrotask | **Macrotasks**: setTimeout, setInterval, I/O | Microtask queue drains completely before next macrotask

---

<a id="q3"></a>
## Q3. Promise methods — all, allSettled, race, any

### 📝 One-Liner
**Promise.all** = ALL must resolve (fails fast on first rejection). **Promise.allSettled** = waits for ALL regardless of outcome. **Promise.race** = first to settle (resolve or reject). **Promise.any** = first to resolve (ignores rejections until all fail).

### 💻 Code
```javascript
const p1 = Promise.resolve("A");
const p2 = Promise.reject("Error");
const p3 = new Promise(resolve => setTimeout(() => resolve("C"), 100));

// Promise.all — fails fast
Promise.all([p1, p2, p3]).catch(e => console.log(e)); // "Error" (fails on p2)

// Promise.allSettled — always completes
Promise.allSettled([p1, p2, p3]).then(results => console.log(results));
// [{ status: "fulfilled", value: "A" },
//  { status: "rejected", reason: "Error" },
//  { status: "fulfilled", value: "C" }]

// Promise.race — first to settle (resolve OR reject)
Promise.race([p1, p2, p3]).then(v => console.log(v)); // "A" (p1 resolves first)

// Promise.any — first to resolve (ignores rejections)
Promise.any([p2, p3]).then(v => console.log(v)); // "C" (ignores p2 rejection)
```

### ⚡ Remember
> **all** = AND (all must pass) | **allSettled** = report all results | **race** = first to finish (any outcome) | **any** = first SUCCESS | Use `all` for parallel independent API calls | Use `allSettled` when you need all results regardless | Use `race` for timeouts

---

<a id="q4"></a>
## Q4. this keyword binding rules in JavaScript

### 📝 One-Liner
**4 rules** in order of precedence: (1) `new` binding (constructor). (2) Explicit binding (`call/apply/bind`). (3) Implicit binding (object method — `this` = the object before dot). (4) Default binding (global/undefined in strict mode). Arrow functions **don't have** their own `this` — they inherit from enclosing scope.

### 💻 Code
```javascript
// Implicit binding — this = object before dot
const obj = { name: "Alice", greet() { console.log(this.name); } };
obj.greet(); // "Alice"

// Lost binding
const greet = obj.greet;
greet(); // undefined (default binding: window/global)

// Explicit binding
greet.call({ name: "Bob" }); // "Bob"

// Arrow function — inherits this from enclosing scope
const arrow = {
    name: "Charlie",
    greet: () => console.log(this.name) // `this` is outer scope, NOT arrow
};
arrow.greet(); // undefined (this = window in non-strict)

// React class components — need bind or arrow
class Component {
    handleClick = () => { /* arrow auto-binds this */ };
}
```

### ⚡ Remember
> **new > explicit > implicit > default** | Arrow functions = no own `this` | Method shorthand in objects uses implicit binding | Class: arrow fields auto-bind `this` | `.bind()` returns new function with fixed `this` | `.call()` invokes immediately

---

<a id="q5"></a>
## Q5. Virtual DOM — how React updates the UI efficiently

### 📝 One-Liner
Virtual DOM is a lightweight **in-memory JavaScript representation** of the real DOM. When state changes, React creates a new Virtual DOM tree, **diffs** it with the previous one (reconciliation), and applies **only the minimal changes** (patches) to the real DOM.

### 🔑 Quick Answer
**Process**: (1) State/props change triggers re-render. (2) React creates new Virtual DOM tree. (3) **Diffing algorithm** compares new vs old VDOM (O(n) with heuristics). (4) Calculates minimal change set. (5) **Batches** and applies DOM updates. This is fast because: JS object operations are cheap, real DOM operations are expensive, and batching reduces reflows/repaints.

### 🗣️ Interview Script
"React maintains a virtual representation of the DOM in memory. When state changes, React creates a new VDOM tree and runs a diffing algorithm comparing it with the previous tree. The key heuristics are: elements of different types produce different trees, and the developer can hint stability with keys. React then calculates the minimum set of mutations needed and batches them into a single DOM update, avoiding expensive per-change reflows."

### ⚡ Remember
> Virtual DOM = JS object tree, not real DOM | Diffing = O(n) with two heuristics | Keys help identify which items changed in lists | Reconciliation = diff + patch | React 18: automatic batching of state updates | Fiber architecture enables interruptible rendering

---

<a id="q6"></a>
## Q6. React Hooks — useState, useEffect, useRef, useMemo, useCallback

### 📝 One-Liner
Hooks let functional components use state and side effects. **useState** = state management. **useEffect** = side effects (API calls, subscriptions). **useRef** = mutable ref without re-render. **useMemo** = memoize computed value. **useCallback** = memoize function reference.

### 💻 Code
```jsx
function UserProfile({ userId }) {
    // useState — reactive state
    const [user, setUser] = useState(null);
    const [loading, setLoading] = useState(true);

    // useRef — persists across renders, no re-render on change
    const renderCount = useRef(0);
    renderCount.current++;

    // useEffect — side effect (API call)
    useEffect(() => {
        setLoading(true);
        fetch(`/api/users/${userId}`)
            .then(res => res.json())
            .then(data => { setUser(data); setLoading(false); });
        return () => { /* cleanup: cancel request, unsubscribe */ };
    }, [userId]); // Runs when userId changes

    // useMemo — memoize expensive computation
    const fullName = useMemo(() => {
        return user ? `${user.firstName} ${user.lastName}`.toUpperCase() : '';
    }, [user]);

    // useCallback — memoize function (for child components)
    const handleSave = useCallback(() => {
        saveUser(user);
    }, [user]);

    if (loading) return <Spinner />;
    return <UserCard name={fullName} onSave={handleSave} />;
}
```

### ⚡ Remember
> **useState**: `[value, setter]` — triggers re-render | **useEffect**: side effects + cleanup return | **useRef**: `.current` — no re-render | **useMemo**: memoize value, `[deps]` | **useCallback**: memoize function, `[deps]` | Rules: only call at top level, only in React functions

---

<a id="q7"></a>
## Q7. useEffect — dependency array behavior and cleanup

### 📝 One-Liner
`useEffect(callback, deps)`: **No deps** = runs after every render. **Empty []** = runs once on mount. **[a, b]** = runs when a or b changes. **Cleanup function** = returned function runs before next effect and on unmount.

### 💻 Code
```jsx
// Runs after EVERY render
useEffect(() => { console.log("Every render"); });

// Runs ONCE on mount
useEffect(() => {
    const subscription = api.subscribe();
    return () => subscription.unsubscribe(); // Cleanup on unmount
}, []);

// Runs when `query` changes
useEffect(() => {
    const controller = new AbortController();
    fetch(`/search?q=${query}`, { signal: controller.signal })
        .then(res => res.json())
        .then(setResults);
    return () => controller.abort(); // Cancel previous request
}, [query]);
```

### ⚡ Remember
> **No deps** = every render | **[]** = mount only | **[deps]** = when deps change | Always return cleanup for: subscriptions, timers, fetch requests | React StrictMode calls effects twice in dev (to catch bugs) | Don't lie about dependencies

---

<a id="q8"></a>
## Q8. State management — useState vs useReducer vs Context vs Redux

### 📝 One-Liner
**useState** = simple local state. **useReducer** = complex local state with defined transitions. **Context** = share state without prop drilling (theme, auth). **Redux** = global state with middleware, DevTools, predictable updates for large apps.

### 🔑 Quick Answer
| Solution | Scope | Complexity | Best For |
|----------|-------|------------|----------|
| useState | Component | Simple | Counters, toggles, form fields |
| useReducer | Component | Medium | Complex objects, state machines |
| Context API | Subtree | Simple | Theme, auth, locale (infrequent updates) |
| Redux (RTK) | Global | Complex | Large apps, frequent updates, middleware |
| Zustand | Global | Simple | Medium apps wanting Redux-like without boilerplate |

### ⚡ Remember
> Start with **useState** | Upgrade to **useReducer** for complex transitions | **Context** for infrequent global state | **Redux/Zustand** for frequent global state with DevTools | Don't use Redux for simple apps | Don't use Context for frequently changing data (re-render perf)

---

<a id="q9"></a>
## Q9. React.memo, useMemo, useCallback — when to use

### 📝 One-Liner
**React.memo** = HOC that skips re-rendering a component if props haven't changed. **useMemo** = memoize a computed value. **useCallback** = memoize a function reference. Use them together: React.memo on child + useCallback for handler props = skip unnecessary child re-renders.

### 💻 Code
```jsx
// Child won't re-render if items and onDelete haven't changed
const TodoList = React.memo(({ items, onDelete }) => {
    console.log("TodoList rendered");
    return items.map(item => (
        <li key={item.id}>
            {item.text}
            <button onClick={() => onDelete(item.id)}>X</button>
        </li>
    ));
});

// Parent
function App() {
    const [todos, setTodos] = useState([]);
    const [filter, setFilter] = useState('');

    // Without useCallback, new function on every render → TodoList re-renders
    const handleDelete = useCallback((id) => {
        setTodos(prev => prev.filter(t => t.id !== id));
    }, []);

    // Without useMemo, filtered recalculated on every render
    const filteredTodos = useMemo(() => {
        return todos.filter(t => t.text.includes(filter));
    }, [todos, filter]);

    return <TodoList items={filteredTodos} onDelete={handleDelete} />;
}
```

### ⚡ Remember
> **React.memo** = component level | **useMemo** = value level | **useCallback** = function level | Use together for maximum effect | Don't premature optimize — measure first with React Profiler | Memoization has memory cost

---

<a id="q10"></a>
## Q10. Code splitting and lazy loading in React

### 📝 One-Liner
**Code splitting** breaks the bundle into smaller chunks loaded on demand. `React.lazy()` + `Suspense` enables component-level splitting. Route-based splitting has the biggest impact — each route becomes a separate chunk.

### 💻 Code
```jsx
// Route-based code splitting
const Home = React.lazy(() => import('./pages/Home'));
const Dashboard = React.lazy(() => import('./pages/Dashboard'));
const Settings = React.lazy(() => import('./pages/Settings'));

function App() {
    return (
        <BrowserRouter>
            <Suspense fallback={<LoadingSpinner />}>
                <Routes>
                    <Route path="/" element={<Home />} />
                    <Route path="/dashboard" element={<Dashboard />} />
                    <Route path="/settings" element={<Settings />} />
                </Routes>
            </Suspense>
        </BrowserRouter>
    );
}

// Named export lazy loading (needs re-export)
// utils/lazyImport.js
export function lazyImport(factory, name) {
    return React.lazy(() =>
        factory().then(module => ({ default: module[name] }))
    );
}
const { UserProfile } = lazyImport(() => import('./components'), 'UserProfile');
```

### ⚡ Remember
> `React.lazy()` only works with default exports (or use wrapper) | Route-level splitting = biggest impact | `<Suspense fallback>` required | Webpack creates separate chunks automatically | Preload: `import(/* webpackPrefetch: true */ './Heavy')`

---

<a id="q11"></a>
## Q11. Error Boundaries in React

### 📝 One-Liner
Error Boundaries are **class components** that catch JavaScript errors in their child component tree during rendering, lifecycle methods, and constructors — preventing the entire app from crashing. They use `componentDidCatch` and `static getDerivedStateFromError`.

### 💻 Code
```jsx
class ErrorBoundary extends React.Component {
    state = { hasError: false, error: null };

    static getDerivedStateFromError(error) {
        return { hasError: true, error };
    }

    componentDidCatch(error, errorInfo) {
        // Log to error reporting service (Sentry, DataDog)
        logErrorToService(error, errorInfo.componentStack);
    }

    render() {
        if (this.state.hasError) {
            return (
                <div>
                    <h2>Something went wrong</h2>
                    <button onClick={() => this.setState({ hasError: false })}>
                        Try Again
                    </button>
                </div>
            );
        }
        return this.props.children;
    }
}

// Usage — wrap sections of your app
<ErrorBoundary>
    <Dashboard />
</ErrorBoundary>
<ErrorBoundary>
    <Sidebar />
</ErrorBoundary>
```

### ⚡ Remember
> Must be class component (no hooks equivalent yet) | Catches: rendering errors, lifecycle errors | Does NOT catch: event handlers, async code, SSR | Wrap around major sections (not individual components) | Log errors to monitoring service | Use `react-error-boundary` library for functional API

---

<a id="q12"></a>
## Q12. SSR vs CSR vs SSG — rendering strategies

### 📝 One-Liner
**CSR** (Client-Side Rendering) = JS renders in browser (React default, slow FCP). **SSR** (Server-Side Rendering) = HTML generated on server per request (fast FCP, SEO-friendly). **SSG** (Static Site Generation) = HTML generated at build time (fastest, limited to static content).

### 🔑 Quick Answer
| | CSR | SSR | SSG |
|---|-----|-----|-----|
| HTML generated | Client (browser) | Server (per request) | Build time |
| First paint | Slow (download + parse JS) | Fast (HTML ready) | Fastest (pre-built) |
| SEO | Poor (empty HTML) | Good (full HTML) | Best (static HTML) |
| Server load | Low | High (per request) | None (CDN) |
| Dynamic content | Yes | Yes | Limited (rebuild needed) |
| Framework | React (CRA) | Next.js (getServerSideProps) | Next.js (getStaticProps) |

### ⚡ Remember
> **CSR** = SPA default, poor SEO | **SSR** = Next.js, good SEO, server cost | **SSG** = fastest, CDN-cached, static content | **ISR** (Incremental Static Regeneration) = SSG + periodic rebuilds | Use SSR for dynamic SEO-critical pages, SSG for blogs/marketing

---

<a id="q13"></a>
## Q13. Debounce vs Throttle — implementation and use cases

### 📝 One-Liner
**Debounce** = delays execution until pause in activity (search input — wait for user to stop typing). **Throttle** = limits execution to once per interval (scroll handler — max once per 200ms).

### 💻 Code
```javascript
// Debounce — executes AFTER delay since last call
function debounce(fn, delay) {
    let timer;
    return function (...args) {
        clearTimeout(timer);
        timer = setTimeout(() => fn.apply(this, args), delay);
    };
}

// Throttle — executes AT MOST once per interval
function throttle(fn, interval) {
    let lastTime = 0;
    return function (...args) {
        const now = Date.now();
        if (now - lastTime >= interval) {
            lastTime = now;
            fn.apply(this, args);
        }
    };
}

// Usage in React
function SearchBar() {
    const debouncedSearch = useMemo(
        () => debounce((query) => fetchResults(query), 300),
        []
    );
    return <input onChange={(e) => debouncedSearch(e.target.value)} />;
}
```

### ⚡ Remember
> **Debounce** = wait until quiet (search, resize, auto-save) | **Throttle** = limit rate (scroll, mousemove, resize) | Debounce resets timer on each call | Throttle skips calls within interval | In React: wrap in useMemo or create outside component

---

<a id="q14"></a>
## Q14. Higher-Order Functions and Array methods

### 📝 One-Liner
Higher-Order Functions take a function as argument or return a function. Key array HOFs: **map** (transform), **filter** (select), **reduce** (accumulate), **forEach** (iterate), **find** (first match), **some/every** (boolean test).

### 💻 Code
```javascript
const users = [
    { name: "Alice", age: 25, role: "admin" },
    { name: "Bob", age: 30, role: "user" },
    { name: "Charlie", age: 20, role: "user" }
];

// map — transform
const names = users.map(u => u.name); // ["Alice", "Bob", "Charlie"]

// filter — select
const adults = users.filter(u => u.age >= 25); // [Alice, Bob]

// reduce — accumulate
const totalAge = users.reduce((sum, u) => sum + u.age, 0); // 75

// find — first match
const admin = users.find(u => u.role === "admin"); // Alice

// Chaining
const adultNames = users
    .filter(u => u.age >= 25)
    .map(u => u.name)
    .sort(); // ["Alice", "Bob"]

// some/every
users.some(u => u.age < 21);  // true (Charlie is 20)
users.every(u => u.age > 18); // true (all > 18)
```

### ⚡ Remember
> **map** = transform (returns new array, same length) | **filter** = select (returns subset) | **reduce** = accumulate to single value | **find** = first match (or undefined) | Chain for readable data transformations | Prefer these over `for` loops in React

---

<a id="q15"></a>
## Q15. ES6+ features essential for React development

### 📝 One-Liner
Key ES6+ features: **Arrow functions** (concise + lexical `this`), **Destructuring** (objects/arrays), **Spread/Rest** (`...`), **Template literals**, **Modules** (import/export), **Optional chaining** (`?.`), **Nullish coalescing** (`??`).

### 💻 Code
```javascript
// Arrow functions
const greet = (name) => `Hello, ${name}`;

// Destructuring + Default
const { name, role = "user" } = props;
const [first, ...rest] = [1, 2, 3, 4]; // first=1, rest=[2,3,4]

// Spread operator
const newObj = { ...oldObj, updatedField: "new" }; // Immutable update
const merged = [...arr1, ...arr2];

// Optional chaining
const city = user?.address?.city ?? "Unknown"; // ?.  +  ?? (nullish coalescing)

// Dynamic property
const key = "name";
const obj = { [key]: "Alice" }; // { name: "Alice" }

// Modules
export const helper = () => {};
import { helper } from './utils';
export default function App() {}
```

### ⚡ Remember
> **Arrow functions** = lexical `this`, concise syntax | **Spread** = immutability in React state updates | **Optional chaining** `?.` = safe nested access | **Nullish coalescing** `??` = only null/undefined (not 0 or '') | **Destructuring** = clean prop/state access

---

<a id="q16"></a>
## Q16. Fetch API vs Axios — comparison and usage

### 📝 One-Liner
**Fetch** = native browser API, returns Promises, doesn't auto-reject on HTTP errors. **Axios** = third-party library, auto-JSON parsing, request/response interceptors, timeout support, better error handling.

### 💻 Code
```javascript
// Fetch — native
const response = await fetch('/api/users', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ name: 'Alice' })
});
if (!response.ok) throw new Error(`HTTP ${response.status}`); // Manual error check!
const data = await response.json(); // Manual JSON parse

// Axios — library
const { data } = await axios.post('/api/users', { name: 'Alice' });
// Auto: JSON parse, error on 4xx/5xx, interceptors

// Axios interceptors (global auth, error handling)
axios.interceptors.request.use(config => {
    config.headers.Authorization = `Bearer ${getToken()}`;
    return config;
});
axios.interceptors.response.use(
    response => response,
    error => {
        if (error.response?.status === 401) redirectToLogin();
        return Promise.reject(error);
    }
);
```

### ⚡ Remember
> **Fetch**: native, manual error handling, manual JSON parse | **Axios**: auto JSON, auto error on 4xx/5xx, interceptors, cancel tokens | Fetch doesn't reject on 404/500 (must check `response.ok`) | Axios for large apps (interceptors), Fetch for simple or no-lib projects

---

<a id="q17"></a>
## Q17. Critical Rendering Path and Web Performance

### 📝 One-Liner
The Critical Rendering Path is the sequence browsers follow to convert HTML/CSS/JS into pixels: **HTML → DOM**, **CSS → CSSOM**, **DOM + CSSOM → Render Tree**, **Layout** (positions), **Paint** (pixels), **Composite** (layers). Optimizing this path = faster First Contentful Paint.

### 🔑 Quick Answer
**Optimization strategies**: (1) Minimize critical resources (inline critical CSS, defer non-critical JS). (2) Minimize critical path length (reduce network round trips). (3) Reduce critical bytes (minify, compress, tree-shake). (4) Use `async`/`defer` on script tags. (5) Preload critical assets (`<link rel="preload">`). (6) Lazy load below-fold images and components.

### ⚡ Remember
> **CRP**: DOM → CSSOM → Render Tree → Layout → Paint → Composite | CSS is render-blocking (must parse before render) | JS is parser-blocking (blocks DOM construction) | `defer` = download parallel, execute after parse | `async` = download parallel, execute immediately | Critical CSS inline, rest async load

---

<a id="q18"></a>
## Q18. XSS and CSRF prevention in web applications

### 📝 One-Liner
**XSS** (Cross-Site Scripting) = attacker injects malicious script into the page. **CSRF** (Cross-Site Request Forgery) = attacker makes authenticated user perform unwanted actions. React auto-escapes JSX — but `dangerouslySetInnerHTML` bypasses protection.

### 🔑 Quick Answer
| Attack | Prevention |
|--------|-----------|
| XSS (Stored) | Sanitize user input, CSP headers, React auto-escaping |
| XSS (Reflected) | Validate/encode URL params, sanitize before render |
| XSS (DOM) | Avoid `dangerouslySetInnerHTML`, sanitize with DOMPurify |
| CSRF | CSRF tokens, SameSite cookies, verify Origin header |

### 💻 Code
```jsx
// React auto-escapes — SAFE
<p>{userInput}</p> // "<script>alert('XSS')</script>" → rendered as text

// DANGEROUS — bypasses protection
<div dangerouslySetInnerHTML={{ __html: userInput }} /> // DON'T use with user input

// Safe approach with DOMPurify
import DOMPurify from 'dompurify';
<div dangerouslySetInnerHTML={{ __html: DOMPurify.sanitize(userInput) }} />
```

### ⚡ Remember
> React JSX auto-escapes by default (great XSS protection) | Never `dangerouslySetInnerHTML` with unsanitized input | Use DOMPurify if HTML rendering needed | CSRF: SameSite cookies + CSRF tokens | Set CSP headers | HttpOnly cookies prevent JS access to tokens

---

<a id="q19"></a>
## Q19. GraphQL vs REST — key differences

### 📝 One-Liner
**REST** = fixed endpoints returning fixed data shapes (over/under-fetching). **GraphQL** = single endpoint, client specifies exact data needed in query (no over-fetching, typed schema).

### 🔑 Quick Answer
| Feature | REST | GraphQL |
|---------|------|---------|
| Endpoints | Multiple (/users, /posts) | Single (/graphql) |
| Data fetching | Server-defined response | Client-defined query |
| Over-fetching | Common | Eliminated |
| Under-fetching | Multiple requests needed | Single query |
| Caching | HTTP caching (simple) | Complex (Apollo Client) |
| Versioning | URL versioning (/v1/users) | Schema evolution (no versions) |
| Learning curve | Low | Higher |

### ⚡ Remember
> **REST** = simple, cacheable, standard | **GraphQL** = flexible, typed, single endpoint | Use REST for simple CRUD, public APIs | Use GraphQL for complex UIs with varied data needs | Apollo Client + GraphQL = powerful React data layer

---

<a id="q20"></a>
## Q20. Webpack, Tree Shaking, and Bundle Optimization

### 📝 One-Liner
**Webpack** = module bundler (bundles JS/CSS/assets). **Tree shaking** = dead code elimination (removes unused exports). **Bundle optimization** = code splitting + minification + compression + lazy loading to reduce initial load time.

### 🔑 Quick Answer
**Key optimizations**: (1) **Tree shaking**: Use ES modules (import/export), avoid side effects in modules, mark sideEffects in package.json. (2) **Code splitting**: Dynamic `import()` + React.lazy. (3) **Minification**: TerserPlugin for JS, CssMinimizerPlugin for CSS. (4) **Compression**: gzip/Brotli. (5) **Analyze**: `webpack-bundle-analyzer` to find large dependencies. (6) **External**: Load large libraries from CDN.

### ⚡ Remember
> Tree shaking needs ES modules (`import/export`, not `require`) | `sideEffects: false` in package.json enables aggressive tree shaking | Analyze bundle with `webpack-bundle-analyzer` | Replace heavy libs (moment → dayjs, lodash → lodash-es) | Vite = faster dev alternative to Webpack
