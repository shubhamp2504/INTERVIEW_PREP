# 🌐 React — Scenario-Based Interview Questions

> Advanced scenario-based React questions commonly asked in 3-5 YOE frontend interviews. Each question presents a real-world problem with architecture, implementation approach, and production considerations.

> **Questions**: Q1–Q10 | **Difficulty**: Advanced

---

<a id="q1"></a>
## Q1. Implement infinite scroll with API pagination

### 📝 One-Liner
Infinite scroll loads more data as the user scrolls near the bottom — use **Intersection Observer API** to detect when a sentinel element enters the viewport, then trigger the next API page fetch.

### 💻 Code
```jsx
function InfiniteList() {
    const [items, setItems] = useState([]);
    const [page, setPage] = useState(1);
    const [hasMore, setHasMore] = useState(true);
    const [loading, setLoading] = useState(false);
    const observerRef = useRef();

    useEffect(() => {
        setLoading(true);
        fetch(`/api/items?page=${page}&limit=20`)
            .then(res => res.json())
            .then(data => {
                setItems(prev => [...prev, ...data.items]);
                setHasMore(data.hasNext);
                setLoading(false);
            });
    }, [page]);

    // Intersection Observer on last element
    const lastItemRef = useCallback(node => {
        if (loading) return;
        if (observerRef.current) observerRef.current.disconnect();
        observerRef.current = new IntersectionObserver(entries => {
            if (entries[0].isIntersecting && hasMore) {
                setPage(prev => prev + 1);
            }
        });
        if (node) observerRef.current.observe(node);
    }, [loading, hasMore]);

    return (
        <ul>
            {items.map((item, idx) => (
                <li key={item.id} ref={idx === items.length - 1 ? lastItemRef : null}>
                    {item.name}
                </li>
            ))}
            {loading && <li>Loading...</li>}
        </ul>
    );
}
```

### ⚡ Remember
> **Intersection Observer** is preferred over scroll event listeners (better performance) | Attach ref to last item | Track `hasMore` flag to stop fetching | Disconnect previous observer on re-render | Consider `react-window` for virtualization of very long lists

---

<a id="q2"></a>
## Q2. Auto-logout after inactivity timeout

### 📝 One-Liner
Track user activity (mouse, keyboard, touch events) and reset a timer on each interaction. When the timer expires, clear auth tokens and redirect to login. Use a single timer + event listeners for efficiency.

### 💻 Code
```jsx
function useAutoLogout(timeoutMs = 15 * 60 * 1000) { // 15 minutes
    const timerRef = useRef();

    const logout = useCallback(() => {
        localStorage.removeItem('token');
        window.location.href = '/login';
    }, []);

    const resetTimer = useCallback(() => {
        clearTimeout(timerRef.current);
        timerRef.current = setTimeout(logout, timeoutMs);
    }, [logout, timeoutMs]);

    useEffect(() => {
        const events = ['mousedown', 'keydown', 'touchstart', 'scroll'];
        events.forEach(event => window.addEventListener(event, resetTimer));
        resetTimer(); // Start initial timer

        return () => {
            clearTimeout(timerRef.current);
            events.forEach(event => window.removeEventListener(event, resetTimer));
        };
    }, [resetTimer]);
}

// Usage in App
function App() {
    useAutoLogout(15 * 60 * 1000); // 15 min
    return <Router>...</Router>;
}
```

### ⚡ Remember
> Track multiple events (mouse, key, touch, scroll) | Single timer — reset on activity | Clear tokens + redirect on logout | Consider warning modal 1-2 min before logout | Cross-tab sync with BroadcastChannel or localStorage events

---

<a id="q3"></a>
## Q3. Internationalization (i18n) — multi-language React app

### 📝 One-Liner
Use **react-i18next** library with translation JSON files per locale. Wrap app with `I18nextProvider`, use `useTranslation` hook to get translated strings. Support RTL layouts, date/number formatting, and dynamic language switching.

### 💻 Code
```jsx
// i18n.js — configuration
import i18n from 'i18next';
import { initReactI18next } from 'react-i18next';

i18n.use(initReactI18next).init({
    resources: {
        en: { translation: { welcome: "Welcome, {{name}}!", save: "Save" } },
        hi: { translation: { welcome: "स्वागत है, {{name}}!", save: "सेव करें" } }
    },
    lng: 'en',
    fallbackLng: 'en',
    interpolation: { escapeValue: false }
});

// Component usage
function Header() {
    const { t, i18n } = useTranslation();
    return (
        <div>
            <h1>{t('welcome', { name: 'Shubham' })}</h1>
            <button onClick={() => i18n.changeLanguage('hi')}>हिंदी</button>
            <button onClick={() => i18n.changeLanguage('en')}>English</button>
        </div>
    );
}
```

### ⚡ Remember
> **react-i18next** = most popular React i18n library | JSON files per language | `useTranslation()` hook in components | Interpolation: `{{variable}}` | Lazy load translations for large apps | RTL layout: `dir="rtl"` based on locale

---

<a id="q4"></a>
## Q4. Large file upload with progress tracking and chunking

### 📝 One-Liner
For large files (100MB+), use **chunked upload** — split file into smaller chunks (5-10MB), upload each with progress tracking, support resume on failure. Backend reassembles chunks.

### 💻 Code
```jsx
function FileUploader() {
    const [progress, setProgress] = useState(0);

    async function uploadFile(file) {
        const CHUNK_SIZE = 5 * 1024 * 1024; // 5MB
        const totalChunks = Math.ceil(file.size / CHUNK_SIZE);
        const uploadId = crypto.randomUUID();

        for (let i = 0; i < totalChunks; i++) {
            const start = i * CHUNK_SIZE;
            const chunk = file.slice(start, start + CHUNK_SIZE);

            const formData = new FormData();
            formData.append('chunk', chunk);
            formData.append('chunkIndex', i);
            formData.append('totalChunks', totalChunks);
            formData.append('uploadId', uploadId);

            await fetch('/api/upload/chunk', { method: 'POST', body: formData });
            setProgress(Math.round(((i + 1) / totalChunks) * 100));
        }
        // Finalize — tell server to merge chunks
        await fetch(`/api/upload/complete/${uploadId}`, { method: 'POST' });
    }

    return (
        <div>
            <input type="file" onChange={e => uploadFile(e.target.files[0])} />
            <progress value={progress} max={100} />
            <span>{progress}%</span>
        </div>
    );
}
```

### ⚡ Remember
> **Chunked upload** for large files | `File.slice()` to split | Track progress per chunk | Resumable: skip already-uploaded chunks | Server merges chunks with uploadId | XHR has `onUploadProgress` for byte-level tracking

---

<a id="q5"></a>
## Q5. SEO optimization for React SPA

### 📝 One-Liner
React SPAs have poor SEO by default (empty HTML). Solutions: **SSR with Next.js** (best), **pre-rendering** (react-snap), **dynamic meta tags** (react-helmet), proper use of semantic HTML, structured data (JSON-LD).

### 🔑 Quick Answer
**SEO checklist for React**: (1) **SSR/SSG** with Next.js — full HTML for crawlers. (2) **Meta tags**: react-helmet-async for title, description, OG tags. (3) **Semantic HTML**: Use h1-h6, main, nav, article, section. (4) **Structured data**: JSON-LD for rich snippets. (5) **Sitemap + robots.txt**. (6) **Core Web Vitals**: LCP < 2.5s, FID < 100ms, CLS < 0.1. (7) **Canonical URLs** for duplicate content.

### ⚡ Remember
> **CSR = bad SEO** (empty HTML, Googlebot may not wait for JS) | **Next.js SSR/SSG = best SEO** | **react-helmet-async** for dynamic meta tags in CSR apps | Semantic HTML matters | Core Web Vitals = ranking factor | Pre-rendering as middle ground

---

<a id="q6"></a>
## Q6. Fix stale closures in React hooks

### 📝 One-Liner
Stale closures happen when a callback captures an old state/props value because the dependency array of `useEffect`/`useCallback` is incorrect. Fix: add correct dependencies, use functional updater (`setState(prev => ...)`), or use `useRef` for mutable current value.

### 💻 Code
```jsx
// BUG: Stale closure — count is always 0
function Counter() {
    const [count, setCount] = useState(0);
    useEffect(() => {
        const id = setInterval(() => {
            console.log(count); // Always 0 — stale!
            setCount(count + 1); // Always sets to 1
        }, 1000);
        return () => clearInterval(id);
    }, []); // Empty deps — closure captures initial count (0)
}

// FIX 1: Functional updater (no dependency needed)
setCount(prev => prev + 1); // Always uses latest value

// FIX 2: Add dependency (re-creates interval each time — less efficient)
useEffect(() => { /* ... */ }, [count]);

// FIX 3: useRef for current value
const countRef = useRef(count);
countRef.current = count; // Sync ref with state
useEffect(() => {
    const id = setInterval(() => {
        console.log(countRef.current); // Always current
    }, 1000);
    return () => clearInterval(id);
}, []);
```

### ⚡ Remember
> Stale closure = callback uses old value | **Fix 1**: functional updater `setState(prev => ...)` (best) | **Fix 2**: correct dependency array | **Fix 3**: `useRef` for reading current value in long-lived callbacks | ESLint `exhaustive-deps` rule catches this

---

<a id="q7"></a>
## Q7. Reduce bundle size — practical strategies

### 📝 One-Liner
Run `webpack-bundle-analyzer` to identify large dependencies, then: (1) Replace heavy libs (moment→dayjs, lodash→lodash-es), (2) Code split routes, (3) Tree shake with ES modules, (4) Lazy load non-critical components, (5) Use CDN for large libraries.

### 🔑 Quick Answer
**Action plan**: (1) **Analyze**: `npx webpack-bundle-analyzer build/stats.json`. (2) **Replace**: moment.js (330KB) → dayjs (2KB), lodash → individual imports or lodash-es. (3) **Split**: Route-level code splitting with React.lazy. (4) **Tree shake**: Use ES module imports (`import { debounce } from 'lodash-es'`). (5) **Externalize**: Large libs (React, chart.js) from CDN. (6) **Compress**: Enable gzip/Brotli on server. (7) **Purge CSS**: Remove unused styles (PurgeCSS).

### ⚡ Remember
> Always analyze bundle first (data-driven optimization) | Replace heavy libs with lighter alternatives | Tree shaking needs ES modules | Code splitting = biggest impact for initial load | gzip saves 60-80% transfer size | Target < 200KB initial JS for fast loading

---

<a id="q8"></a>
## Q8. Role-based routing and protected routes

### 💻 Code
```jsx
// ProtectedRoute component
function ProtectedRoute({ children, allowedRoles }) {
    const { user, isAuthenticated } = useAuth();

    if (!isAuthenticated) return <Navigate to="/login" />;
    if (allowedRoles && !allowedRoles.includes(user.role)) {
        return <Navigate to="/unauthorized" />;
    }
    return children;
}

// Route configuration
function App() {
    return (
        <Routes>
            <Route path="/login" element={<Login />} />
            <Route path="/dashboard" element={
                <ProtectedRoute allowedRoles={['admin', 'user']}>
                    <Dashboard />
                </ProtectedRoute>
            } />
            <Route path="/admin" element={
                <ProtectedRoute allowedRoles={['admin']}>
                    <AdminPanel />
                </ProtectedRoute>
            } />
            <Route path="/unauthorized" element={<Unauthorized />} />
        </Routes>
    );
}
```

### ⚡ Remember
> Client-side route protection = UX only (server must also validate!) | Check auth status + user role | Redirect to login if unauthenticated | Redirect to /unauthorized if wrong role | Store user role in JWT/context | Server-side API authorization is the real security

---

<a id="q9"></a>
## Q9. Page transitions and animations in React

### 📝 One-Liner
Use **Framer Motion** (most popular), **react-transition-group**, or CSS transitions for page animations. Wrap routes with `AnimatePresence` for enter/exit animations on navigation.

### 💻 Code
```jsx
import { motion, AnimatePresence } from 'framer-motion';
import { useLocation } from 'react-router-dom';

function AnimatedRoutes() {
    const location = useLocation();
    return (
        <AnimatePresence mode="wait">
            <motion.div
                key={location.pathname}
                initial={{ opacity: 0, x: 50 }}
                animate={{ opacity: 1, x: 0 }}
                exit={{ opacity: 0, x: -50 }}
                transition={{ duration: 0.3 }}
            >
                <Routes location={location}>
                    <Route path="/" element={<Home />} />
                    <Route path="/about" element={<About />} />
                </Routes>
            </motion.div>
        </AnimatePresence>
    );
}
```

### ⚡ Remember
> **Framer Motion** = most popular React animation library | `AnimatePresence` + `motion.div` for enter/exit | `key={location.pathname}` triggers animation on route change | CSS transitions for simple hover/toggle | Prefer `transform` and `opacity` for 60fps (GPU-accelerated)

---

<a id="q10"></a>
## Q10. API request throttling and rate limiting on frontend

### 📝 One-Liner
Prevent excessive API calls through: **debounce** (search inputs), **throttle** (scroll/resize handlers), **request deduplication** (SWR/React Query cache), **queue with concurrency limit**, and **abort stale requests** (AbortController).

### 💻 Code
```jsx
// AbortController — cancel stale requests
function useSearch(query) {
    const [results, setResults] = useState([]);
    const controllerRef = useRef();

    useEffect(() => {
        if (!query) return;

        // Abort previous request
        controllerRef.current?.abort();
        controllerRef.current = new AbortController();

        const debounceTimer = setTimeout(async () => {
            try {
                const res = await fetch(`/api/search?q=${encodeURIComponent(query)}`, {
                    signal: controllerRef.current.signal
                });
                const data = await res.json();
                setResults(data);
            } catch (e) {
                if (e.name !== 'AbortError') console.error(e);
            }
        }, 300); // Debounce 300ms

        return () => {
            clearTimeout(debounceTimer);
            controllerRef.current?.abort();
        };
    }, [query]);

    return results;
}
```

### ⚡ Remember
> **Debounce + AbortController** = best combo for search | **React Query/SWR** has built-in deduplication + caching | Cancel stale requests to avoid race conditions | Ignore `AbortError` in catch | Server-side rate limiting is still needed (429 status handling)
