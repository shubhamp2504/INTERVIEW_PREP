# 🏢 CGI — Frontend Interview Experience Round 1 (React + JavaScript + Performance)

> Round 1 focused heavily on performance, React internals, state management, real-world scenarios, and JavaScript coding. Deep technical round for experienced frontend developers.

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

## Section A: Core Frontend & Performance

---

<a id="q1"></a>
## Q1. How do you optimize performance of a web application?

### 📝 One-Liner
Bundle optimization (code splitting, tree shaking, lazy loading) + rendering optimization (virtualization, memoization) + network optimization (caching, CDN, compression) + image optimization (WebP, lazy load) + monitoring (Lighthouse, Web Vitals).

### 🔑 Quick Answer
**Frontend checklist**: (1) Code splitting — `React.lazy` for routes. (2) Tree shaking — ESM imports, remove dead code. (3) GZIP/Brotli compression. (4) CDN for static assets. (5) Image optimization — WebP, responsive `srcset`, lazy load. (6) Virtual lists for 1000+ items. (7) `React.memo` / `useMemo` / `useCallback`. (8) Service Worker for offline cache. (9) HTTP/2 for parallel requests. (10) Critical CSS inlined, non-critical deferred.

### ⚡ Remember
> Split → Cache → Compress → Lazy load → Virtualize → Memo → Measure with Lighthouse

---

<a id="q2"></a>
## Q2. What is lazy loading?

### 📝 One-Liner
Load resources **on demand** rather than upfront — images load when scrolled into viewport, route components load when navigated to. Reduces initial bundle size and improves First Contentful Paint.

### 💻 Code
```jsx
// Route-level lazy loading
const Settings = React.lazy(() => import('./Settings'));
<Suspense fallback={<Spinner />}><Settings /></Suspense>

// Image lazy loading (native)
<img src="photo.jpg" loading="lazy" alt="Photo" />

// Intersection Observer (custom)
const observer = new IntersectionObserver(entries => {
  entries.forEach(entry => {
    if (entry.isIntersecting) {
      entry.target.src = entry.target.dataset.src; // Load actual image
      observer.unobserve(entry.target);
    }
  });
});
```

### ⚡ Remember
> `React.lazy` for components | `loading="lazy"` for images | Intersection Observer for custom triggers | Reduces initial payload by 40-60%

---

<a id="q3"></a>
## Q3. How to cancel previous API requests?

### 📝 One-Liner
Use `AbortController` — create a controller per request, call `controller.abort()` when a new request starts or component unmounts. With axios, pass `signal` in config. With fetch, pass `signal` option.

### 💻 Code
```javascript
// AbortController with fetch
const controller = new AbortController();
fetch('/api/search?q=react', { signal: controller.signal })
  .then(res => res.json())
  .catch(err => { if (err.name === 'AbortError') console.log('Cancelled'); });
controller.abort(); // Cancel!

// In React — cancel on new search or unmount
function Search() {
  const [query, setQuery] = useState('');
  useEffect(() => {
    const controller = new AbortController();
    fetch(`/api/search?q=${query}`, { signal: controller.signal })
      .then(r => r.json()).then(setResults).catch(() => {});
    return () => controller.abort(); // Cancel on re-run or unmount
  }, [query]);
}
```

### ⚡ Remember
> `AbortController` for fetch | `CancelToken` (deprecated) / `signal` for axios | Cancel in useEffect cleanup | Prevents race conditions in search-as-you-type

---

<a id="q4"></a>
## Q4. Difference between SSR and CSR?

### 🆚 vs.
| Feature | SSR (Server-Side Rendering) | CSR (Client-Side Rendering) |
|---------|----------------------------|----------------------------|
| HTML generated | Server (Node.js) | Browser (JavaScript) |
| Initial load | Fast (HTML ready) | Slow (JS download → render) |
| SEO | ✅ Excellent | ❌ Poor (empty HTML initially) |
| Interactivity | Slower (hydration needed) | Faster (already in JS) |
| Server load | Higher | Lower |
| Framework | Next.js, Nuxt.js | React SPA, Angular SPA |
| Best for | Content sites, e-commerce | Dashboards, admin panels |

### ⚡ Remember
> **SSR** = server sends ready HTML (SEO, fast FCP) | **CSR** = server sends skeleton, JS renders (SPA) | **SSG** = pre-build at compile time (static sites) | Next.js supports all three

---

<a id="q5"></a>
## Q5. What is WebSocket?

### 📝 One-Liner
WebSocket is a **full-duplex, persistent** communication protocol (ws://) — unlike HTTP (request-response), WebSocket keeps the connection open for **bidirectional** real-time data exchange (chat, live feeds, gaming).

### 🔑 Quick Answer
**HTTP**: Client asks → Server responds → Connection closes. **WebSocket**: Client connects → Upgrade handshake → Both sides can send messages anytime → Connection stays open. **Use cases**: Chat apps, live notifications, stock tickers, collaborative editing, multiplayer games.

### ⚡ Remember
> Full-duplex (bidirectional) | Persistent connection (no polling) | Upgrade from HTTP handshake | Use for real-time: chat, live data, gaming | SSE for server→client only

---

<a id="q6"></a>
## Q6. What is a Service Worker?

### 📝 One-Liner
A Service Worker is a **JavaScript proxy** that runs in the background (separate thread), intercepting network requests for **offline caching**, push notifications, and background sync — the foundation of Progressive Web Apps (PWA).

### 🔑 Quick Answer
**Lifecycle**: Register → Install (cache assets) → Activate (clean old caches) → Fetch (intercept requests, serve from cache or network). **Cache strategies**: Cache-first (offline-first), Network-first (fresh data), Stale-while-revalidate (fast + fresh). Cannot access DOM directly.

### ⚡ Remember
> Background thread (no DOM access) | Intercepts fetch requests | Cache strategies: cache-first, network-first, stale-while-revalidate | Enables offline + push notifications | HTTPS required

---

<a id="q7"></a>
## Q7. How do you prevent XSS and CSRF attacks?

### 📝 One-Liner
**XSS**: Sanitize user input, use textContent over innerHTML, Content-Security-Policy headers, React auto-escapes JSX. **CSRF**: Use anti-CSRF tokens (SameSite cookies), verify Origin/Referer headers, don't use GET for mutations.

### 🔑 Quick Answer
**XSS prevention**: (1) React escapes JSX by default (safe from most XSS). (2) Never use `dangerouslySetInnerHTML` with user input. (3) Sanitize with DOMPurify if HTML rendering needed. (4) Content-Security-Policy header blocks inline scripts. **CSRF prevention**: (1) `SameSite=Strict` on cookies. (2) Anti-CSRF token in forms. (3) Verify `Origin` header on server. (4) Use POST/PUT/DELETE for mutations (never GET).

### ⚡ Remember
> React auto-escapes = XSS safe by default | Avoid `dangerouslySetInnerHTML` | CSP headers | SameSite cookies for CSRF | CSRF tokens for forms | Never trust user input

---

## Section B: React & State Management

---

<a id="q8"></a>
## Q8. Explain implementation of Context API

### 💻 Code
```jsx
// 1. Create context
const ThemeContext = React.createContext('light');

// 2. Provider wraps app
function App() {
  const [theme, setTheme] = useState('light');
  return (
    <ThemeContext.Provider value={{ theme, setTheme }}>
      <Header />
      <Main />
    </ThemeContext.Provider>
  );
}

// 3. Consume with useContext
function Header() {
  const { theme, setTheme } = useContext(ThemeContext);
  return <button onClick={() => setTheme(t => t === 'light' ? 'dark' : 'light')}>
    Theme: {theme}
  </button>;
}
```

### ⚡ Remember
> `createContext` → `Provider` wraps tree → `useContext` consumes | All consumers re-render when value changes | Split contexts for unrelated state | Use memo to reduce re-renders

---

<a id="q9"></a>
## Q9. Explain implementation of Redux Toolkit

### 💻 Code
```javascript
// 1. Create slice (reducer + actions)
import { createSlice, configureStore } from '@reduxjs/toolkit';

const counterSlice = createSlice({
  name: 'counter',
  initialState: { value: 0 },
  reducers: {
    increment: state => { state.value += 1; },  // Immer allows "mutation"
    decrement: state => { state.value -= 1; },
    addBy: (state, action) => { state.value += action.payload; }
  }
});

// 2. Configure store
const store = configureStore({ reducer: { counter: counterSlice.reducer } });

// 3. Use in component
function Counter() {
  const count = useSelector(state => state.counter.value);
  const dispatch = useDispatch();
  return (
    <>
      <span>{count}</span>
      <button onClick={() => dispatch(counterSlice.actions.increment())}>+</button>
    </>
  );
}
```

### ⚡ Remember
> `createSlice` = reducer + actions | `configureStore` = store setup | `useSelector` to read | `useDispatch` to update | Immer under the hood (write "mutations")

---

<a id="q10"></a>
## Q10. Difference between Redux and Redux Toolkit?

### 🆚 vs.
| Feature | Redux (vanilla) | Redux Toolkit |
|---------|----------------|---------------|
| Boilerplate | ❌ Action types, creators, reducers | ✅ `createSlice` (all-in-one) |
| Immutability | Manual spread operator | ✅ Immer (write mutations) |
| Store setup | Manual combineReducers + middleware | ✅ `configureStore` |
| Async | Manual thunk/saga setup | ✅ `createAsyncThunk` built-in |
| DevTools | Manual config | ✅ Auto-included |
| API calls | Not included | ✅ RTK Query |

### ⚡ Remember
> RTK = Redux best practices in a box | 60-80% less boilerplate | Immer for immutability | `createAsyncThunk` for async | RTK Query for API management

---

<a id="q11"></a>
## Q11. What is useEffect?

### 📝 One-Liner
`useEffect` runs **side effects** after render — API calls, subscriptions, DOM manipulation. Dependencies array controls when it re-runs: `[]` = mount only, `[dep]` = when dep changes, none = every render.

### 💻 Code
```jsx
useEffect(() => {
  const fetchData = async () => {
    const res = await fetch('/api/users');
    setUsers(await res.json());
  };
  fetchData();
  return () => { /* cleanup: unsubscribe, cancel requests */ };
}, []); // [] = run once on mount
```

### ⚡ Remember
> Side effects after render | `[]` = mount only | `[dep]` = when dep changes | Return cleanup function | Don't call setters directly (infinite loop) | Use async function inside, not on useEffect directly

---

<a id="q12"></a>
## Q12. What are Error Boundaries? Explain implementation

### 📝 One-Liner
Error Boundaries are **class components** that catch JavaScript errors in their child component tree, log the error, and display a **fallback UI** instead of crashing the entire app.

### 💻 Code
```jsx
class ErrorBoundary extends React.Component {
  state = { hasError: false, error: null };

  static getDerivedStateFromError(error) {
    return { hasError: true, error };
  }

  componentDidCatch(error, errorInfo) {
    logErrorToService(error, errorInfo); // Send to Sentry/DataDog
  }

  render() {
    if (this.state.hasError) {
      return <div>Something went wrong. <button onClick={() =>
        this.setState({ hasError: false })}>Retry</button></div>;
    }
    return this.props.children;
  }
}

// Usage
<ErrorBoundary>
  <RiskyComponent />
</ErrorBoundary>
```

### ⚡ Remember
> Class component only (no hook equivalent yet) | Catches render errors, not event handlers | Wrap at route/section level | Log to monitoring service | Does NOT catch: event handlers, async code, SSR, errors in itself

---

<a id="q13"></a>
## Q13. How do you handle errors in React applications?

### 📝 One-Liner
**Render errors** → Error Boundaries. **Event handler errors** → try-catch. **Async/API errors** → try-catch in async functions, `.catch()` on promises. **Global** → `window.onerror` + error logging service (Sentry).

### ⚡ Remember
> Error Boundaries for render | try-catch for events + async | Global error handler for uncaught | Log to Sentry/DataDog | Show user-friendly fallback UI

---

<a id="q14"></a>
## Q14. What is reconciliation in React?

### 📝 One-Liner
Reconciliation is React's **diffing algorithm** that compares the previous and new Virtual DOM trees to determine the **minimum set of changes** needed to update the real DOM — it runs in O(n) using heuristics.

### 🔑 Quick Answer
**Two heuristics**: (1) Different element types → destroy old tree, build new (e.g., `<div>` → `<span>`). (2) Same type → compare attributes, update changed ones, recurse on children. **Keys**: For lists, `key` prop helps React identify which items moved, added, or removed — without keys, React re-renders entire list.

### ⚡ Remember
> O(n) algorithm (not O(n³) tree diff) | Same type → update attributes | Different type → replace subtree | `key` prop essential for lists | Stable keys (ID, not index) for dynamic lists

---

<a id="q15"></a>
## Q15. Difference between React 16 and React 18?

### 🆚 vs.
| Feature | React 16 | React 18 |
|---------|----------|----------|
| Rendering | Synchronous | ✅ Concurrent rendering |
| Batching | Only in event handlers | ✅ Automatic batching everywhere |
| Suspense | Basic (lazy only) | ✅ Full (data fetching, SSR) |
| Root API | `ReactDOM.render()` | ✅ `createRoot()` |
| Transitions | ❌ | ✅ `useTransition`, `useDeferredValue` |
| Strict Mode | Less strict | ✅ Double-invokes effects (dev) |
| Streaming SSR | ❌ | ✅ `renderToPipeableStream` |

### ⚡ Remember
> React 18 = concurrent rendering | Auto-batching (fewer re-renders) | `useTransition` for non-urgent updates | `createRoot` replaces `ReactDOM.render` | Streaming SSR for faster TTFB

---

<a id="q16"></a>
## Q16. What is state scheduling in React?

### 📝 One-Liner
React **batches** state updates and schedules re-renders — calling `setState` doesn't immediately update state. React 18's concurrent mode adds **priority-based scheduling**: urgent updates (typing) are processed before non-urgent (search results).

### 🔑 Quick Answer
**Batching**: Multiple `setState` calls in one event handler = single re-render. **React 18 auto-batching**: Works in setTimeout, promises, native event handlers too (not just React events). **Transitions**: `useTransition` marks updates as non-urgent → React renders urgent updates first, defers non-urgent.

### ⚡ Remember
> State updates are batched (not immediate) | React 18 auto-batches everywhere | `useTransition` for low-priority updates | `flushSync()` to force immediate update (rare)

---

## Section C: JavaScript & Coding

---

<a id="q17"></a>
## Q17. What are closures?

### 📝 One-Liner
A closure is a function that **remembers** variables from its outer (lexical) scope even after the outer function has finished executing — the inner function "closes over" those variables.

### 💻 Code
```javascript
function createCounter() {
  let count = 0; // Enclosed variable
  return {
    increment: () => ++count,
    getCount: () => count
  };
}
const counter = createCounter();
counter.increment(); // 1
counter.increment(); // 2
counter.getCount();  // 2 — "count" still accessible via closure!

// Practical use: debounce
function debounce(fn, delay) {
  let timer; // Closed over
  return (...args) => {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delay);
  };
}
```

### ⚡ Remember
> Function + its lexical environment = closure | Used in: counters, debounce, memoization, module pattern, React hooks | Common gotcha: closures in loops (use `let` not `var`)

---

<a id="q18"></a>
## Q18. Implement throttle

### 💻 Code
```javascript
function throttle(fn, limit) {
  let inThrottle = false;
  return function(...args) {
    if (!inThrottle) {
      fn.apply(this, args);
      inThrottle = true;
      setTimeout(() => inThrottle = false, limit);
    }
  };
}

// Usage: scroll handler fires at most once per 200ms
window.addEventListener('scroll', throttle(() => {
  console.log('Scrolled at', Date.now());
}, 200));

// Debounce vs Throttle:
// Debounce: waits until user STOPS, then fires ONCE (search input)
// Throttle: fires at REGULAR intervals while action continues (scroll, resize)
```

### ⚡ Remember
> Throttle = max once per interval (scroll, resize) | Debounce = wait until idle (search input) | Both use closures | Throttle: `if (!inThrottle)` | Debounce: `clearTimeout + setTimeout`

---

<a id="q19"></a>
## Q19. Difference between fetch and axios?

### 🆚 vs.
| Feature | fetch | axios |
|---------|-------|-------|
| Built-in | ✅ Browser native | ❌ npm package (~15KB) |
| JSON parsing | Manual `.json()` | ✅ Automatic |
| Error handling | No throw on 4xx/5xx | ✅ Throws on non-2xx |
| Interceptors | ❌ | ✅ Request/response |
| Cancel | AbortController | AbortController (or signal) |
| Timeout | ❌ Manual | ✅ `timeout` option |
| Progress | ❌ | ✅ `onUploadProgress` |
| Browser support | Modern only | ✅ IE11+ with polyfill |

### ⚡ Remember
> fetch = lightweight, native, manual error handling | axios = feature-rich, auto JSON, interceptors | RTK Query/TanStack Query replace both for React apps

---

<a id="q20"></a>
## Q20. Write code to find frequency of elements in an array

### 💻 Code
```javascript
const arr = [1, 2, 2, 3, 3, 3, 4, 4, 4, 4];

// Method 1: reduce
const freq = arr.reduce((acc, val) => {
  acc[val] = (acc[val] || 0) + 1;
  return acc;
}, {});
// { 1: 1, 2: 2, 3: 3, 4: 4 }

// Method 2: Map
const freqMap = new Map();
arr.forEach(v => freqMap.set(v, (freqMap.get(v) || 0) + 1));

// Method 3: One-liner
const freq3 = arr.reduce((a, v) => ({ ...a, [v]: (a[v] || 0) + 1 }), {});
```

### ⚡ Remember
> `reduce` is the standard approach | `Map` preserves insertion order | `Map` handles non-string keys better | O(n) time, O(k) space where k = unique elements

---

## Section D: Practical / Scenario-Based

---

<a id="q21"></a>
## Q21. Why migrate from Angular to React? What challenges did you face?

### 📝 One-Liner
**Why**: Smaller bundle, faster rendering (VDOM), larger ecosystem, simpler mental model (components + hooks vs modules + decorators), better hiring pool. **Challenges**: Rewriting component logic, state management migration (NgRx → Redux), routing differences, testing setup changes, team retraining.

### ⚡ Remember
> Don't migrate for hype — have concrete reasons (performance, hiring, ecosystem) | Gradual migration > big bang rewrite | Keep both running during transition | Shared design system helps

---

<a id="q22"></a>
## Q22. How to send data from parent to child component?

### 💻 Code
```jsx
// Parent → Child via props
function Parent() {
  const [user, setUser] = useState({ name: 'Alice', age: 30 });
  return <Child user={user} onUpdate={setUser} />;
}

function Child({ user, onUpdate }) {
  return <div>{user.name} <button onClick={() => onUpdate({...user, age: 31})}>Birthday</button></div>;
}
```

### ⚡ Remember
> Props flow DOWN (parent → child) | Events flow UP (child → parent via callback props) | For deep nesting → Context API or state management

---

<a id="q23"></a>
## Q23. What is prop drilling?

### 📝 One-Liner
Prop drilling is passing props through **multiple intermediate components** that don't use them, just to reach a deeply nested child — makes code verbose and hard to maintain.

### 🔑 Quick Answer
**Problem**: App → Layout → Sidebar → Menu → MenuItem — passing `user` through 4 levels when only MenuItem needs it. **Solutions**: (1) **Context API** — provide at top, consume where needed. (2) **Redux/Zustand** — global state store. (3) **Component composition** — pass components as children instead of data. (4) **Custom hooks** — encapsulate shared logic.

### ⚡ Remember
> Prop drilling = passing through intermediaries | Solution: Context for simple shared state, Redux for complex | Component composition (children prop) is underused but powerful | Max 2-3 levels of prop passing is OK
