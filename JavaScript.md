# JavaScript

### Variables & scope
`var` (function-scoped, hoisted, avoid) · `let` (block, reassignable) · `const` (block, no reassign — object contents can still change).
`==` (type coercion) vs `===` (strict, always use).

### Closures
A function that **remembers variables from where it was created**.
```js
function counter() {
  let count = 0;                 // private
  return () => ++count;
}
const c = counter();
c(); c();   // 1, 2  — count survives because the inner function closes over it
```

### `this`, arrow functions
Regular function: `this` depends on **how it's called**. Arrow function: `this` taken from **surrounding scope** (use in callbacks/React).

### Event loop ⭐⭐⭐ (most asked)
JavaScript is **single-threaded**; async work is handed to the browser and results return through queues.
```
Call Stack (runs code)                Web APIs (timers, fetch, DOM events – outside JS thread)
     ▲                                          │ when done
     │ event loop picks when stack is empty     ▼
     ├──── Microtask queue (Promises .then, async/await, queueMicrotask)  ← runs FIRST, fully
     └──── Macrotask queue (setTimeout, setInterval, DOM events, I/O)      ← one task per turn
```
```js
console.log("1");
setTimeout(() => console.log("2"), 0);       // macrotask
Promise.resolve().then(() => console.log("3")); // microtask
console.log("4");
// Output: 1 4 3 2
```

### Promises & async/await
```js
// callback hell → Promise → async/await
async function placeOrder(order) {
  try {
    const res = await fetch("/api/orders", {
      method: "POST",
      headers: { "Content-Type": "application/json", Authorization: `Bearer ${token}` },
      body: JSON.stringify(order),
    });
    if (!res.ok) throw new Error(`HTTP ${res.status}`);   // fetch does NOT reject on 404/500!
    return await res.json();
  } catch (e) { console.error(e); throw e; }
}
await Promise.all([getUser(), getOrders()]);   // parallel, fails fast
await Promise.allSettled([...]);               // wait for all, never rejects
```

### Modern essentials
```js
const { name, age } = user;                 // destructuring
const merged = { ...a, ...b };              // spread
const copy = [...list, 4];                  // immutable update
list.map(x => x * 2).filter(x => x > 5).reduce((s, x) => s + x, 0);
user?.address?.city ?? "unknown";           // optional chaining + nullish coalescing
import { add } from "./math.js";  export default App;   // ES modules
```
Other Qs: hoisting, prototype chain / prototypal inheritance, `call/apply/bind`, debounce vs throttle, `null` vs `undefined`, deep vs shallow copy (`structuredClone`), `localStorage` vs `sessionStorage` vs cookies, `==` gotchas, `typeof null === "object"`.

```js
function debounce(fn, ms) {           // run only after user STOPS typing
  let t;
  return (...args) => { clearTimeout(t); t = setTimeout(() => fn(...args), ms); };
}
```
