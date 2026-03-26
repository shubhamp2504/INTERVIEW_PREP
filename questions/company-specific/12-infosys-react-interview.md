# 🏢 Infosys — Frontend React Interview Experience

> Practical interview focused on building a React Grid Matrix (n × n), dynamic rendering, state management, accessibility, and DevTools debugging. Emphasis on implementation thinking over theory.

> 📝 One-Liner → 🔑 Quick Answer → 💻 Code → ⚡ Remember

---

<a id="q1"></a>
## Q1. Build a React Grid Matrix (n × n) based on user input (e.g., 3×3, 4×4)

### 📝 One-Liner
Accept a number `n` from user input → dynamically render an n×n grid of cells using nested `Array.from()` or `map` → style with CSS Grid → handle cell click for interactivity.

### 💻 Code
```jsx
import { useState } from 'react';

function GridMatrix() {
  const [size, setSize] = useState(3);
  const [activeCells, setActiveCells] = useState(new Set());

  const toggleCell = (row, col) => {
    const key = `${row}-${col}`;
    setActiveCells(prev => {
      const next = new Set(prev);
      next.has(key) ? next.delete(key) : next.add(key);
      return next;
    });
  };

  return (
    <div>
      <label>
        Grid size:
        <input type="number" min="1" max="10" value={size}
               onChange={e => {
                 setSize(Number(e.target.value));
                 setActiveCells(new Set());
               }} />
      </label>

      <div className="grid" style={{
        display: 'grid',
        gridTemplateColumns: `repeat(${size}, 50px)`,
        gap: '4px',
        marginTop: '16px'
      }}>
        {Array.from({ length: size }, (_, row) =>
          Array.from({ length: size }, (_, col) => {
            const key = `${row}-${col}`;
            return (
              <div key={key}
                   role="button" tabIndex={0}
                   aria-label={`Cell ${row + 1}, ${col + 1}`}
                   className={`cell ${activeCells.has(key) ? 'active' : ''}`}
                   onClick={() => toggleCell(row, col)}
                   onKeyDown={e => e.key === 'Enter' && toggleCell(row, col)}
                   style={{
                     width: 50, height: 50,
                     border: '1px solid #ccc',
                     backgroundColor: activeCells.has(key) ? '#4CAF50' : '#fff',
                     cursor: 'pointer',
                     display: 'flex', alignItems: 'center', justifyContent: 'center'
                   }}>
                {row * size + col + 1}
              </div>
            );
          })
        )}
      </div>

      <p>Active cells: {activeCells.size}</p>
    </div>
  );
}
```

### ⚡ Remember
> `Array.from({ length: n })` for dynamic rows/cols | CSS Grid for layout | Set for tracking active cells | `role="button"` + `tabIndex` for accessibility | Key should be unique and stable

---

<a id="q2"></a>
## Q2. Handling dynamic rendering & state management

### 📝 One-Liner
Use `useState` for local UI state (grid size, active cells), derive rendering from state (not DOM manipulation), and re-render automatically when state changes — React's declarative model handles DOM updates.

### 🔑 Quick Answer
**Principles**: (1) State drives UI — change state, React re-renders. (2) Never manipulate DOM directly (`document.getElementById`). (3) Derived state — compute from existing state instead of adding new state. (4) Lift state up when siblings need to share. (5) For complex state — `useReducer` over multiple `useState`.

### ⚡ Remember
> State → UI (declarative) | Don't store derived data as state | `useReducer` for complex state transitions | Lift state to lowest common ancestor | Controlled components for forms

---

<a id="q3"></a>
## Q3. Basics of Accessibility (a11y) in web applications

### 📝 One-Liner
Accessibility ensures web apps are usable by everyone — use **semantic HTML** (button, nav, main), **ARIA attributes** (role, aria-label), **keyboard navigation** (tabIndex, onKeyDown), **color contrast** (4.5:1 ratio), and **screen reader support** (alt text).

### 🔑 Quick Answer
**Quick wins**: (1) Semantic HTML — `<button>` not `<div onClick>`. (2) Alt text on images. (3) Label on inputs (`<label htmlFor>`). (4) Keyboard handlers — Enter/Space for interactive elements. (5) Focus management — visible focus indicator, skip-to-content link. (6) ARIA — `role`, `aria-live` for dynamic content, `aria-expanded` for accordions. (7) Color contrast — 4.5:1 for normal text, 3:1 for large. (8) Test with screen reader (NVDA/VoiceOver).

### ⚡ Remember
> Semantic HTML first, ARIA second | `<button>` not `<div onClick>` | Alt text on every image | Visible focus indicators | Test with keyboard-only navigation | Chrome Lighthouse audits a11y

---

<a id="q4"></a>
## Q4. Usage of Browser DevTools for debugging

### 📝 One-Liner
Chrome DevTools: **Elements** (DOM inspection, CSS editing), **Console** (JS execution, errors), **Network** (API calls, timing, size), **Performance** (flame chart, bottlenecks), **React DevTools** (component tree, props, state, profiler).

### 🔑 Quick Answer
**Debugging workflow**: (1) Console tab — check errors, `console.log`, breakpoints. (2) Network tab — verify API calls, check response/status/timing. (3) Sources tab — set breakpoints, step through code. (4) Performance tab — record and find slow renders (flame chart). (5) React DevTools — inspect component hierarchy, state changes, re-render causes. (6) Lighthouse — automated performance/a11y/SEO audit.

### ⚡ Remember
> Console for errors | Network for API issues | Sources for breakpoints | Performance for bottlenecks | React DevTools Profiler for re-render analysis | Lighthouse for automated audits

---

<a id="q5"></a>
## Q5. Core React concepts like hooks & performance

### 📝 One-Liner
**Essential hooks**: `useState` (local state), `useEffect` (side effects), `useContext` (shared state), `useMemo` (cached value), `useCallback` (cached function), `useRef` (mutable ref, DOM access). Performance: avoid unnecessary re-renders with memoization.

### 🔑 Quick Answer
**Hook rules**: (1) Only call at top level (no conditions/loops). (2) Only in React functions. **Performance hooks**: `useMemo` for expensive computations, `useCallback` for stable function refs (with React.memo on children), `useTransition` (React 18) for non-urgent updates. **Profiling**: React DevTools Profiler highlights which components re-rendered and why.

### ⚡ Remember
> `useState` + `useEffect` = 80% of hooks usage | `useRef` for DOM access + persist values without re-render | `useMemo`/`useCallback` only when profiling shows need | Hook rules: top level, React functions only
