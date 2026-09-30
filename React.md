# React

### Core idea
UI = **function of state**: `UI = f(state)`. When state changes, React re-runs the component and updates only what changed (via **Virtual DOM diffing**).

```
State/props change → Component re-renders (new virtual DOM tree)
   → React diffs old vs new tree (reconciliation)
   → Applies minimal real DOM updates (commit) → browser paints
```
`key` prop on list items lets React match items between renders (use stable ids, not array index).

### Component + hooks (with real order-page example)
```jsx
import { useState, useEffect } from "react";

function OrderPage() {
  const [orders, setOrders] = useState([]);       // state
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {                                // runs AFTER render
    let cancelled = false;
    fetch("/api/orders", { headers: { Authorization: `Bearer ${token}` } })
      .then(r => { if (!r.ok) throw new Error(r.status); return r.json(); })
      .then(data => { if (!cancelled) setOrders(data); })
      .catch(setError)
      .finally(() => setLoading(false));
    return () => { cancelled = true; };            // cleanup on unmount
  }, []);                                          // [] = run once on mount

  if (loading) return <p>Loading…</p>;
  if (error) return <p>Error: {String(error)}</p>;
  return (
    <ul>{orders.map(o => <li key={o.id}>{o.id} – ₹{o.total}</li>)}</ul>
  );
}
```

### `useEffect` dependency flow
```
Render → commit to DOM → useEffect runs
   deps [] → only first time         deps [id] → when id changes
   no deps → after EVERY render      cleanup function → runs before next effect / unmount
```

### Hooks summary
| Hook | Purpose |
|---|---|
| `useState` | local state; setter triggers re-render; use functional update `setN(n => n+1)` |
| `useEffect` | side-effects (API call, subscription, timer) |
| `useRef` | mutable value/DOM ref that does NOT cause re-render |
| `useMemo` / `useCallback` | cache computed value / function between renders (optimisation) |
| `useContext` | read shared data (theme, user) without prop drilling |
| `useReducer` | complex state transitions (Redux-like) |
| custom hook | reuse logic: `useFetch(url)` |

Rules of hooks: only call at top level (no loops/conditions), only in components/custom hooks.

### Props vs state; lifting state; controlled forms
```jsx
function Child({ product, onAdd }) {             // props: read-only, passed by parent
  return <button onClick={() => onAdd(product.id)}>Add</button>;
}
function Parent() {
  const [cart, setCart] = useState([]);           // state lives in the common parent
  return <Child product={{id:1}} onAdd={id => setCart(c => [...c, id])} />;
}
// controlled input: value comes from state
<input value={name} onChange={e => setName(e.target.value)} />
```
Never mutate state directly (`cart.push(x)` ✘ → `setCart([...cart, x])` ✔).

### State management & routing
Local → `useState`; shared → Context / **Redux Toolkit** / Zustand; server data → **React Query** (caching, refetch). Routing: `react-router-dom` (`<Route path="/orders/:id">`, `useParams`, `useNavigate`, protected routes redirecting if no token).

### Auth flow in React (ties to your Spring Security notes)
```
Login form → POST /auth/login → receive JWT (best: HttpOnly cookie; else memory)
   → axios interceptor adds Authorization header to every request
   → 401 response → try refresh token → else redirect to /login
```
Storing JWT in `localStorage` is vulnerable to XSS; HttpOnly cookie is vulnerable to CSRF unless SameSite/CSRF token.

### Performance & other Qs
`React.memo`, lazy loading (`React.lazy` + `Suspense`), code splitting, list virtualisation, avoid inline object props. Class vs function components (use functions). Real vs virtual DOM. Controlled vs uncontrolled. SSR/Next.js vs CSR. Testing: Jest + React Testing Library.
