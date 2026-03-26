# 🏢 Bajaj Allianz — Full-Stack Interview Experience (Backend + CI/CD + React)

> Mixed interview covering backend API design, CI/CD pipelines, React fundamentals, and real-world debugging scenarios. Focus on practical experience over theory.

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

## Section A: Backend

---

<a id="q1"></a>
## Q1. How do you design scalable APIs?

### 📝 One-Liner
Stateless REST endpoints + pagination + caching (Redis) + rate limiting + async processing for heavy operations + API versioning + GZIP compression + horizontal scaling behind load balancer.

### 🔑 Quick Answer
**Key principles**: (1) **Stateless** — no server sessions, JWT for auth. (2) **Pagination** — `?page=0&size=20` or cursor-based for large datasets. (3) **Caching** — Redis for hot data, `Cache-Control` headers. (4) **Rate limiting** — protect against abuse (429 Too Many Requests). (5) **Async** — offload heavy tasks to message queues. (6) **GZIP** — reduce payload size by 70%. (7) **Connection pooling** — HikariCP for DB. (8) **HATEOAS** — self-documenting responses. (9) **Versioning** — URI (`/v1/`) or header-based. (10) **Monitoring** — latency p99, error rates, throughput.

### ⚡ Remember
> Stateless + Paginate + Cache + Rate limit + Async offload + GZIP + Connection pool + Monitor | Design for 10× current load

---

<a id="q2"></a>
## Q2. Microservices vs Monolith — which one did you use and why?

### 📝 One-Liner
**Monolith** = single deployable unit (simpler, faster dev for small teams). **Microservices** = independent services per domain (scalable, team autonomy, tech diversity). Choose based on team size, complexity, and scaling needs.

### 🆚 vs.
| Aspect | Monolith | Microservices |
|--------|----------|---------------|
| Deployment | Single unit | Independent per service |
| Scaling | Scale everything | Scale per service |
| Team size | 1-10 devs | 10+ with domain teams |
| Complexity | Lower initially | Higher (network, observability) |
| Data | Shared DB | DB per service |
| Testing | Easier (one process) | Harder (integration tests) |
| Start with | ✅ New projects | ❌ Don't start here |
| Migration | — | When monolith becomes bottleneck |

### ⚡ Remember
> Start monolith, migrate to microservices when needed | Microservices = organizational scaling, not just tech | Each service owns its data | Conway's Law: architecture mirrors team structure

---

<a id="q3"></a>
## Q3. How do you optimize API performance?

### 📝 One-Liner
Profile first (find the bottleneck), then: optimize DB queries (indexes, N+1 fix), add caching (Redis/Caffeine), enable GZIP, use connection pooling, paginate responses, and make I/O calls async.

### 🔑 Quick Answer
**Common bottlenecks + fixes**: (1) **Slow queries** → add indexes, fix N+1 with JOIN FETCH, use projections. (2) **No caching** → Redis for repeated reads, HTTP cache headers. (3) **Large payloads** → pagination, field filtering (`?fields=id,name`), GZIP. (4) **Blocking I/O** → CompletableFuture, WebClient for parallel calls. (5) **Connection exhaustion** → tune HikariCP pool size. (6) **Serialization** → Jackson view filters, DTO projections.

### ⚡ Remember
> Profile before optimizing | 80% of latency is usually DB | Index + Cache + Paginate solves most | Async parallel calls for multiple downstream services

---

## Section B: CI/CD & DevOps

---

<a id="q4"></a>
## Q4. Explain your CI/CD pipeline

### 📝 One-Liner
Code Push → Build (Maven/Gradle) → Unit Tests → Code Quality (SonarQube) → Docker Build → Push to Registry → Deploy to Dev → Integration Tests → Deploy to Staging → Manual Approval → Deploy to Prod.

### 📖 How It Works
```
Developer pushes to Git
  → Webhook triggers Jenkins/GitHub Actions
    → Stage 1: Checkout + Build (mvn clean package)
    → Stage 2: Unit Tests (JUnit, 80% coverage gate)
    → Stage 3: SonarQube Analysis (quality gates)
    → Stage 4: Docker Build + Push to ECR
    → Stage 5: Deploy to Dev (auto)
    → Stage 6: Integration Tests (Postman/RestAssured)
    → Stage 7: Deploy to Staging (auto)
    → Stage 8: Deploy to Prod (manual approval gate)
  → Slack notification on success/failure
```

### ⚡ Remember
> Pipeline as code (Jenkinsfile/workflow YAML) | Quality gates before deploy | Manual approval for production | Rollback strategy ready | Monitoring post-deploy

---

<a id="q5"></a>
## Q5. How do you handle deployment failures?

### 📝 One-Liner
**Immediate**: Automated rollback to last known good version. **Strategy**: Use blue-green or canary deployments so failures affect minimal users. **Post-mortem**: Root cause analysis, add test coverage, update runbooks.

### 🔑 Quick Answer
**Strategies**: (1) **Blue-Green** — two identical environments; swap traffic on deploy; instant rollback = swap back. (2) **Canary** — route 5% traffic to new version; if errors spike, rollback; if stable, gradually increase. (3) **Rolling** — replace instances one by one; health checks gate progression. (4) **Automated rollback** — monitoring alerts trigger auto-rollback (response time > threshold or error rate > 1%).

### ⚡ Remember
> Blue-green = instant rollback | Canary = gradual rollout (safest) | Health checks at every stage | Automated rollback on error spike | Post-mortem for every failure

---

<a id="q6"></a>
## Q6. What is Blue-Green deployment?

### 📝 One-Liner
Run **two identical environments** (Blue = current, Green = new version). Deploy to Green, test, then switch the load balancer to route traffic from Blue to Green. Instant rollback = switch back to Blue.

### 📖 How It Works
```
Before deploy:
  Load Balancer → Blue (v1.0) ← 100% traffic
                  Green (idle)

Deploy new version:
  Load Balancer → Blue (v1.0) ← 100% traffic
                  Green (v2.0) ← deploy + smoke test

Switch traffic:
  Load Balancer → Green (v2.0) ← 100% traffic
                  Blue (v1.0) ← standby (rollback target)

If issues: Switch back to Blue instantly!
```

### ⚡ Remember
> Two environments: Blue (current) + Green (new) | LB switch = zero downtime | Instant rollback | Requires 2× infrastructure cost | Simpler than canary, less granular

---

## Section C: React

---

<a id="q7"></a>
## Q7. What is Virtual DOM and how does it improve performance?

### 📝 One-Liner
The Virtual DOM is a **lightweight JavaScript copy** of the real DOM. When state changes, React creates a new VDOM, **diffs** it against the previous one (reconciliation), and applies only the **minimal changes** to the real DOM.

### 📖 How It Works
```
State Change → New Virtual DOM tree created
  ↓
Diffing (Reconciliation Algorithm):
  Old VDOM vs New VDOM → finds minimal differences
  ↓
Batch Update:
  Only changed elements updated in Real DOM
  (Instead of re-rendering entire page)

Why faster:
  JS object manipulation (VDOM): ~1ms
  Real DOM manipulation: ~10-100ms (triggers reflow/repaint)
  10-100× faster to diff in memory, then apply minimal changes!
```

### ⚡ Remember
> VDOM = JS object tree (cheap to create/diff) | Reconciliation = O(n) diffing | Batch updates to real DOM | `key` prop helps reconciliation identify moved elements

---

<a id="q8"></a>
## Q8. Difference between state and props

### 🆚 vs.
| Feature | State | Props |
|---------|-------|-------|
| Owned by | Component itself | Parent component |
| Mutable | ✅ via `setState`/`useState` | ❌ Read-only |
| Triggers re-render | ✅ Yes | ✅ Yes (when parent passes new) |
| Scope | Local to component | Passed from parent to child |
| Default | Component defined | Parent defined |
| Use case | User input, toggles, form data | Configuration, callbacks, data |

### ⚡ Remember
> **State** = internal, mutable, owned | **Props** = external, read-only, passed down | "Props down, events up"

---

<a id="q9"></a>
## Q9. useMemo & useCallback — explain with examples

### 📝 One-Liner
`useMemo` caches a **computed value**; `useCallback` caches a **function reference** — both prevent unnecessary recalculations and re-renders when dependencies haven't changed.

### 💻 Code
```jsx
// useMemo — expensive calculation
const filtered = useMemo(() =>
  products.filter(p => p.price > minPrice).sort((a, b) => a.price - b.price),
  [products, minPrice] // Only recalculates when these change
);

// useCallback — stable function reference
const handleDelete = useCallback((id) => {
  setItems(prev => prev.filter(item => item.id !== id));
}, []); // Same function reference across renders

// React.memo — skip re-render if props unchanged
const ProductCard = React.memo(({ product, onDelete }) => {
  return <div>{product.name} <button onClick={() => onDelete(product.id)}>×</button></div>;
});
```

### ⚡ Remember
> `useMemo(fn, deps)` = cached VALUE | `useCallback(fn, deps)` = cached FUNCTION | Use with `React.memo` on children | Profile first — don't prematurely optimize

---

<a id="q10"></a>
## Q10. Lazy loading & code splitting in React

### 📝 One-Liner
`React.lazy()` dynamically imports components → Webpack creates separate bundles per lazy component → loaded on demand when component renders → reduces initial bundle size by 40-60%.

### 💻 Code
```jsx
const HeavyChart = React.lazy(() => import('./HeavyChart'));

function Dashboard() {
  return (
    <Suspense fallback={<Skeleton />}>
      <HeavyChart data={data} />
    </Suspense>
  );
}
```

### ⚡ Remember
> `React.lazy` + `Suspense` | Best at route level | Reduces initial load | Webpack auto-chunks | Preload critical routes with `prefetch`

---

<a id="q11"></a>
## Q11. Redux vs Context API — when to use which?

### 🆚 vs.
| Feature | Context API | Redux |
|---------|-------------|-------|
| Complexity | Simple (built-in) | Complex (actions, reducers, store) |
| Size | 0 KB (React built-in) | ~15 KB (redux + react-redux) |
| DevTools | ❌ | ✅ Redux DevTools (time travel) |
| Middleware | ❌ | ✅ Thunk, Saga, RTK Query |
| Performance | ❌ Re-renders all consumers | ✅ Selective subscriptions |
| Best for | Theme, locale, auth | Complex state, many consumers |
| Learning curve | Low | Higher (but RTK simplifies) |

### ⚡ Remember
> Context = simple shared state (theme, auth) | Redux = complex state with devtools + middleware | Context re-renders ALL consumers on change | Redux `useSelector` = granular subscriptions

---

## Section D: Real Scenarios

---

<a id="q12"></a>
## Q12. API is slow — what will you do?

### 📝 One-Liner
**Profile** → find bottleneck → **DB** (slow query → add index, fix N+1) → **Network** (add caching layer, CDN) → **Code** (async operations, connection pooling) → **Infra** (scale up/out, tune JVM).

### 🔑 Quick Answer
**Investigation order**: (1) Check APM (response time breakdown). (2) DB query analysis (EXPLAIN ANALYZE). (3) External API calls (timeout, retries). (4) Memory/CPU metrics. (5) Connection pool saturation. **Common fixes**: Add Redis cache, add DB indexes, parallelize independent calls, paginate large responses, add circuit breaker for slow dependencies.

### ⚡ Remember
> Profile before fixing | 80% is usually DB | EXPLAIN ANALYZE is your friend | Cache + Index + Async = 90% of fixes | Monitor after every change

---

<a id="q13"></a>
## Q13. Build failed in CI — how will you debug?

### 📝 One-Liner
Check CI logs → identify failure stage (build/test/deploy) → reproduce locally → fix → push fix → verify pipeline passes.

### 🔑 Quick Answer
**Steps**: (1) Read CI logs — find exact error (compilation, test failure, dependency issue). (2) Check if it's a flaky test (re-run once). (3) Reproduce locally with same command (`mvn clean test`). (4) Common causes: dependency conflict, environment variable missing, test ordering issue, disk space, timeout. (5) Fix + push. (6) Add test to prevent regression. (7) If environment issue — check Docker image, Node/Java version, cache invalidation.

### ⚡ Remember
> Read logs first (90% answer is there) | Reproduce locally | Check: dependency versions, env vars, Docker image | Flaky tests = address root cause, don't just re-run | Post-fix: add monitoring/alerting

---

<a id="q14"></a>
## Q14. How do you optimize React performance in production?

### 📝 One-Liner
Code splitting (lazy routes) + memoization (React.memo, useMemo, useCallback) + virtualization (react-window for long lists) + image optimization (lazy load, WebP) + production build + CDN for static assets.

### ⚡ Remember
> Lazy routes | Memo for expensive renders | Virtualize long lists | Compress images | Production build (`npm run build`) | CDN for assets | Lighthouse audit for metrics
