# 🏢 EY (Ernst & Young) — React.js Frontend Engineer Interview Experience (2 Technical Rounds)

> Two technical rounds: Round 1 focused on JavaScript fundamentals, Promises, and coding. Round 2 focused on React optimization, Redux, CSS layout, and live coding (todo app).

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

## Technical Round 1: JavaScript Fundamentals + Coding

---

<a id="q1"></a>
## Q1. What will be the output? (this keyword + setTimeout + Promise)

### 📝 One-Liner
Tests understanding of `this` binding in object methods vs extracted functions, and microtask (Promise) vs macrotask (setTimeout) execution order.

### 💻 Code
```javascript
// Output question 1: this binding
const obj = { name: "A", getName() { return this.name; } };
const fn = obj.getName;
console.log(fn()); // undefined (or error in strict mode)
// fn is called without context → this = window (non-strict) → window.name = undefined

// Output question 2: Event loop ordering
console.log("Start");
setTimeout(() => console.log("Timeout"), 0);
Promise.resolve().then(() => console.log("Promise"));
console.log("End");
// Output: Start → End → Promise → Timeout
// Microtask queue (Promise.then) runs BEFORE macrotask queue (setTimeout)
```

### ⚡ Remember
> Extracted method loses `this` context | Microtasks (Promise) before macrotasks (setTimeout) | Sync code runs first, then microtasks, then macrotasks

---

<a id="q2"></a>
## Q2. What is a Promise in JavaScript? How do you resolve a Promise?

### 📝 One-Liner
A Promise is an object representing the **eventual completion** (or failure) of an async operation — it has 3 states: pending → fulfilled (resolved) or rejected.

### 💻 Code
```javascript
// Creating and resolving a Promise
const fetchData = () => new Promise((resolve, reject) => {
  setTimeout(() => {
    const success = true;
    if (success) resolve({ id: 1, name: "EY" }); // fulfilled
    else reject(new Error("API failed"));          // rejected
  }, 1000);
});

// Consuming
fetchData()
  .then(data => console.log(data))    // { id: 1, name: "EY" }
  .catch(err => console.error(err))
  .finally(() => console.log("Done"));

// async/await (syntactic sugar)
async function getData() {
  try {
    const data = await fetchData();
    console.log(data);
  } catch (err) {
    console.error(err);
  }
}
```

### ⚡ Remember
> 3 states: pending → fulfilled/rejected | `resolve()` to fulfill, `reject()` to fail | `.then()` for success, `.catch()` for error | `async/await` = cleaner Promise syntax

---

<a id="q3"></a>
## Q3. Write a Promise method that resolves something

### 💻 Code
```javascript
// Simple Promise that resolves after delay
const delay = (ms, value) => new Promise(resolve => setTimeout(() => resolve(value), ms));

// Promise.all — wait for all
const results = await Promise.all([
  delay(1000, "First"),
  delay(500, "Second"),
  delay(1500, "Third")
]);
// results: ["First", "Second", "Third"] — resolves when ALL complete

// Promise.race — first to settle wins
const fastest = await Promise.race([delay(1000, "Slow"), delay(100, "Fast")]);
// fastest: "Fast"

// Promise.allSettled — get all results (success + failure)
const outcomes = await Promise.allSettled([
  Promise.resolve("OK"),
  Promise.reject("Fail"),
  Promise.resolve("Also OK")
]);
// [{ status: "fulfilled", value: "OK" }, { status: "rejected", reason: "Fail" }, ...]
```

### ⚡ Remember
> `Promise.all` = all succeed or first failure | `Promise.race` = first to settle | `Promise.allSettled` = all results regardless | `Promise.any` = first success

---

<a id="q4"></a>
## Q4. Write a program to return an array of sorted unique elements

### 💻 Code
```javascript
// Input: [2,1,3,4,2,5,4,6,7,4]  →  Output: [1,2,3,4,5,6,7]

// Method 1: Set + Sort
const sortedUnique = arr => [...new Set(arr)].sort((a, b) => a - b);

// Method 2: Filter + Sort
const sortedUnique2 = arr =>
  arr.filter((v, i, a) => a.indexOf(v) === i).sort((a, b) => a - b);

// Method 3: Reduce
const sortedUnique3 = arr =>
  arr.reduce((acc, v) => acc.includes(v) ? acc : [...acc, v], []).sort((a, b) => a - b);

console.log(sortedUnique([2,1,3,4,2,5,4,6,7,4])); // [1,2,3,4,5,6,7]
```

### ⚡ Remember
> `new Set()` removes duplicates | `.sort((a,b) => a-b)` for numeric sort (default sort is lexicographic!) | Method 1 is cleanest and fastest (O(n log n))

---

<a id="q5"></a>
## Q5. Create a todo list app — input field + add button, display list, each item has delete button

### 💻 Code
```jsx
import { useState } from 'react';

function TodoApp() {
  const [input, setInput] = useState('');
  const [todos, setTodos] = useState([]);

  const addTodo = () => {
    const trimmed = input.trim();
    if (!trimmed) return;
    setTodos(prev => [...prev, { id: Date.now(), text: trimmed }]);
    setInput('');
  };

  const deleteTodo = (id) => {
    setTodos(prev => prev.filter(todo => todo.id !== id));
  };

  return (
    <div>
      <input value={input} onChange={e => setInput(e.target.value)}
             onKeyDown={e => e.key === 'Enter' && addTodo()} />
      <button onClick={addTodo}>Add</button>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            {todo.text}
            <button onClick={() => deleteTodo(todo.id)}>Delete</button>
          </li>
        ))}
      </ul>
    </div>
  );
}
```

### ⚡ Remember
> `useState` for list + input | `Date.now()` for unique IDs | Filter to delete | `key` prop on list items | Handle Enter key for UX

---

## Technical Round 2: React Optimization + Redux + Live Coding

---

<a id="q6"></a>
## Q6. How do you optimize a React application?

### 📝 One-Liner
**Prevent unnecessary re-renders** (React.memo, useMemo, useCallback), **reduce bundle size** (code splitting, tree shaking, lazy loading), **optimize rendering** (virtualization for large lists), and **cache data** (React Query, SWR).

### 🔑 Quick Answer
**Key techniques**: (1) `React.memo` for pure components. (2) `useMemo` for expensive calculations. (3) `useCallback` for stable function references. (4) `React.lazy` + `Suspense` for code splitting. (5) Virtualization (`react-window`) for long lists. (6) Image optimization (lazy load, WebP, CDN). (7) Debounce expensive event handlers. (8) Use production builds (minification, tree shaking).

### ⚡ Remember
> Memo/useMemo/useCallback prevent re-renders | Lazy loading splits bundles | Virtualization for 1000+ item lists | React Profiler to identify bottlenecks

---

<a id="q7"></a>
## Q7. Flexbox vs CSS Grid — which for e-commerce product layout (3 columns, multiple rows)?

### 📝 One-Liner
**CSS Grid** is ideal for 2D layouts (rows AND columns simultaneously) — perfect for product grids. **Flexbox** is for 1D layouts (row OR column). For a 3-column product grid like Amazon, Grid gives cleaner code with `grid-template-columns: repeat(3, 1fr)`.

### 💻 Code
```css
/* ✅ CSS Grid — cleaner for product grid */
.product-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);     /* 3 equal columns */
  gap: 16px;
}
/* Responsive */
@media (max-width: 768px) {
  .product-grid { grid-template-columns: repeat(2, 1fr); }
}
@media (max-width: 480px) {
  .product-grid { grid-template-columns: 1fr; }
}

/* Flexbox alternative — works but needs wrapping */
.product-flex {
  display: flex;
  flex-wrap: wrap;
  gap: 16px;
}
.product-card { flex: 0 0 calc(33.33% - 12px); }
```

### ⚡ Remember
> **Grid** = 2D (rows + columns) → product layouts, dashboards | **Flexbox** = 1D (row OR column) → navbars, card rows | Grid is cleaner for equal-column layouts

---

<a id="q8"></a>
## Q8. Redux / Redux Toolkit — how to subscribe to store, what is RTK Query

### 📝 One-Liner
**Redux Toolkit** simplifies Redux with `createSlice` (reducer + actions), `configureStore`, and **RTK Query** (built-in data fetching + caching layer that replaces manual axios/fetch calls with auto-managed loading/error/cache states).

### 💻 Code
```javascript
// Redux Toolkit — store subscription
import { useSelector, useDispatch } from 'react-redux';

function Counter() {
  const count = useSelector(state => state.counter.value); // Subscribe
  const dispatch = useDispatch();
  return <button onClick={() => dispatch(increment())}>Count: {count}</button>;
}

// RTK Query — built-in API management
import { createApi, fetchBaseQuery } from '@reduxjs/toolkit/query/react';

const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: '/api' }),
  endpoints: (builder) => ({
    getUsers: builder.query({ query: () => '/users' }),
    addUser: builder.mutation({
      query: (user) => ({ url: '/users', method: 'POST', body: user })
    })
  })
});

// Auto-generated hooks!
const { data, isLoading, error } = api.useGetUsersQuery();
// No manual useState, useEffect, loading state management!
```

### 🆚 vs.
| Feature | fetch/axios | RTK Query |
|---------|------------ |-----------|
| Loading state | Manual useState | Auto `isLoading` |
| Caching | Manual | Built-in + auto-invalidation |
| Deduplication | None | Automatic |
| Polling | Manual setInterval | `pollingInterval` option |
| Code | ~20 lines per API | ~5 lines per endpoint |

### ⚡ Remember
> `useSelector` to subscribe | `useDispatch` to dispatch | RTK Query = auto-managed fetch + cache + loading | Replaces axios boilerplate | Auto-generated hooks per endpoint

---

<a id="q9"></a>
## Q9. How to make an app responsive

### 📝 One-Liner
Use **CSS media queries** for breakpoints, **relative units** (rem, %, vw/vh), **CSS Grid/Flexbox** for fluid layouts, **mobile-first** approach (min-width queries), and **viewport meta tag** for device scaling.

### 🔑 Quick Answer
**Mobile-first**: Start with mobile styles, add `@media (min-width: 768px)` for tablet, `(min-width: 1024px)` for desktop. Use `rem` over `px`, `%` or `fr` for widths, `max-width` on containers. Flexbox for horizontal alignment, Grid for layouts. Images: `max-width: 100%`, `srcset` for responsive images. Test with Chrome DevTools device emulation.

### ⚡ Remember
> Mobile-first (`min-width`) | rem > px | Flexbox/Grid for layout | `max-width: 100%` for images | Viewport meta tag essential | Test on real devices

---

<a id="q10"></a>
## Q10. React.memo, useCallback, useMemo — how to prevent re-creation of functions passed to child

### 📝 One-Liner
Wrap callback functions with `useCallback(fn, [deps])` to maintain stable reference across re-renders → prevents `React.memo` child from re-rendering when parent re-renders but callback hasn't changed.

### 💻 Code
```jsx
// ❌ Without useCallback — child re-renders every time
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = () => console.log("clicked"); // New function each render!
  return <MemoChild onClick={handleClick} />;        // Re-renders despite memo!
}

// ✅ With useCallback — stable reference
function Parent() {
  const [count, setCount] = useState(0);
  const handleClick = useCallback(() => console.log("clicked"), []); // Stable!
  return <MemoChild onClick={handleClick} />; // Does NOT re-render on count change
}

const MemoChild = React.memo(({ onClick }) => {
  console.log("Child rendered");
  return <button onClick={onClick}>Click</button>;
});

// useMemo for expensive calculations
const expensiveResult = useMemo(() => computeHeavy(data), [data]);
```

### 🆚 vs.
| Hook | Returns | Use Case |
|------|---------|----------|
| `useMemo` | Memoized value | Expensive computations |
| `useCallback` | Memoized function | Stable callbacks for children |
| `React.memo` | Memoized component | Prevent re-render if props unchanged |

### ⚡ Remember
> `useCallback` = stable function reference | Pair with `React.memo` on child | `useMemo` = cached computation result | Only optimize when profiler shows need | Premature optimization is the root of all evil

---

<a id="q11"></a>
## Q11. How did you implement code splitting in your project?

### 📝 One-Liner
Use `React.lazy()` + `Suspense` for route-level code splitting — each route loads its bundle on demand, reducing initial load time. Webpack automatically creates separate chunks.

### 💻 Code
```jsx
import { lazy, Suspense } from 'react';
import { Routes, Route } from 'react-router-dom';

// Lazy-loaded routes — separate bundles
const Dashboard = lazy(() => import('./pages/Dashboard'));
const Settings = lazy(() => import('./pages/Settings'));
const Reports = lazy(() => import('./pages/Reports'));

function App() {
  return (
    <Suspense fallback={<LoadingSpinner />}>
      <Routes>
        <Route path="/" element={<Dashboard />} />
        <Route path="/settings" element={<Settings />} />
        <Route path="/reports" element={<Reports />} />
      </Routes>
    </Suspense>
  );
}
// Webpack will create: main.js, Dashboard.chunk.js, Settings.chunk.js, Reports.chunk.js
```

### ⚡ Remember
> `React.lazy` + `Suspense` = route-level splitting | Webpack auto-creates chunks | Reduces initial bundle by 40-60% | Add fallback UI (spinner) while loading

---

<a id="q12"></a>
## Q12. Enhanced todo app — if input is number, multiply by 5 then display; if string, display as-is. Evaluate production readiness.

### 💻 Code
```jsx
function TodoApp() {
  const [input, setInput] = useState('');
  const [todos, setTodos] = useState([]);

  const addTodo = () => {
    const trimmed = input.trim();
    if (!trimmed) return;

    // Number check — multiply by 5
    const display = isNaN(trimmed) ? trimmed : String(Number(trimmed) * 5);

    setTodos(prev => [...prev, { id: Date.now(), text: display, original: trimmed }]);
    setInput('');
  };

  const deleteTodo = id => setTodos(prev => prev.filter(t => t.id !== id));

  return (
    <div>
      <input value={input} onChange={e => setInput(e.target.value)}
             onKeyDown={e => e.key === 'Enter' && addTodo()}
             placeholder="Enter item..." />
      <button onClick={addTodo} disabled={!input.trim()}>Add</button>
      <ul>
        {todos.map(todo => (
          <li key={todo.id}>
            {todo.text}
            <button onClick={() => deleteTodo(todo.id)}>×</button>
          </li>
        ))}
      </ul>
      {todos.length === 0 && <p>No items yet</p>}
    </div>
  );
}

// Production readiness concerns:
// ❌ No input sanitization (XSS if rendering HTML)
// ❌ Date.now() IDs can collide in rapid adds → use uuid
// ❌ No persistence (refresh loses data) → localStorage or API
// ❌ No error boundaries
// ❌ No accessibility (aria labels, keyboard nav)
// ❌ No input max length validation
```

### ⚡ Remember
> `isNaN()` to check number | Always trim input | Disable button when empty | Handle edge cases (empty, whitespace, special chars) | Production needs: persistence, a11y, error handling, sanitization
