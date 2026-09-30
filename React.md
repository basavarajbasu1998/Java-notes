# React (18 / 19) - Interview Deep Dive

> Audience: 5-year Java full-stack dev. Backend is your strength, React is the secondary skill, but interviews still probe it hard.
> Plain-JS logic in this note (mini hooks list, queue arithmetic, reducer, debounce, refresh-queue) was **run in Node 22** and outputs quoted are real.
> Anything that needs the real React runtime (StrictMode double render, batching counts) is written from knowledge and marked **[expected behaviour]**.

**Contents**
1. 60-second mental model
2. Internals: JSX, elements, render vs commit, reconciliation, keys, Fiber, batching, snapshots/stale closures, hooks list, effect timeline
3. useState and useReducer in practice
4. Effects in depth (timing, deps, cleanup, StrictMode, fetching, You Might Not Need an Effect)
5. Other hooks: refs, memo, context, transitions, useId, useSyncExternalStore
6. Custom hooks (code)
7. Error boundaries, portals, refs, forms, composition
8. State management decision guide (Redux Toolkit, Zustand, Jotai, TanStack Query)
9. Routing (React Router v6)
10. Performance playbook
11. Rendering strategies + Next.js
12. React 19 features
13. Testing, TypeScript, accessibility, security
14. Full auth flow with Spring Boot JWT
15. Full CRUD feature (Orders) + Vite proxy + Docker/nginx
16. Common bugs / production stories
17. 25 "implement it" tasks
18. 50+ interview questions
19. Cheat sheet

---

## 1. 60-second mental model

```
UI = f(state, props)        -- a component is a PURE function that DESCRIBES the UI
```
* A **component** is a function returning a description (JSX -> plain objects called *elements*).
* When state/props change React **calls your function again** (a *render*), gets a new description, **diffs** it with the previous one (*reconciliation*) and applies the minimal DOM edits (*commit*).
* State is a **snapshot per render**: inside one render `count` never changes, no matter how many times you call `setCount`.
* Effects (`useEffect`) are how you **synchronize with the outside world** (network, timers, subscriptions, DOM APIs) after React has committed. They are not "lifecycle methods".

**Analogy (restaurant):** props = the order slip, state = the kitchen's memory between orders, render = the chef re-writing the *recipe card* (cheap, on paper, no side effects), commit = the waiter actually changing the table (expensive, real world), effect = phoning the supplier after the table is set. React is the manager comparing the old recipe card with the new one so waiters touch only what differs.

**Java analogy:** a component is like a `Function<Props, ViewModel>`; hooks are like fields on a per-instance object stored *positionally* (index 0, 1, 2 ...) - which is why call order must never change.

The 8 sentences that answer 80% of interviews:
1. Render must be pure; side effects go in event handlers or effects.
2. Re-render = call the function again, not "touch the DOM".
3. Children re-render when the parent re-renders (unless `memo` and same props by `Object.is`).
4. `setState` schedules a render; the current closure keeps the old value.
5. Keys = identity. Same key + same type at the same position = same instance (state preserved).
6. Effects run after paint; dependencies must list everything reactive you read.
7. Server state (API data) is not client state - use TanStack Query/RTK Query.
8. Measure before memoizing.

---

## 2. Deep internals

### 2.1 JSX is syntax sugar

```jsx
const el = <Button color="red" onClick={go}>Save</Button>;
```
Classic transform (React <= 16 / `runtime: classic`):
```js
const el = React.createElement(Button, { color: "red", onClick: go }, "Save");
```
Automatic runtime (React 17+, default in Vite/CRA/Next - no need to `import React`):
```js
import { jsx as _jsx } from "react/jsx-runtime";
const el = _jsx(Button, { color: "red", onClick: go, children: "Save" });
```
(`jsxs` is used when there are multiple static children; `key` is passed as the 3rd arg.)

The result is a **React element** - a plain immutable object roughly:
```js
{ $$typeof: Symbol(react.element), type: Button, key: null, ref: null, props: { color:"red", onClick:go, children:"Save" } }
```
`$$typeof` (a Symbol) is why JSON coming from a server cannot be smuggled in as an element (XSS defence).

### 2.2 Element vs component vs instance

| Term | What it is | Lifetime |
|---|---|---|
| **Component** | Function (or class) `Button` - the *recipe* | Defined once |
| **Element** | Plain object `{type, props, key}` describing what to render | Created on **every render**, cheap, immutable |
| **Instance** | The mounted thing React keeps (a **Fiber** node holding state, hooks, DOM node) | From mount to unmount |

`<Button/>` does **not** call `Button`. React calls it later, during render. `Button()` called manually breaks the hooks rules (no fiber).

Component names must be Capitalized: `<button>` = host element `"button"` (string type), `<Button>` = component reference.

### 2.3 Render phase vs commit phase (the pipeline)

```
 trigger                render phase (pure, may be paused/restarted/thrown away)      commit phase (sync, cannot be interrupted)
 -------                -----------------------------------------------------      --------------------------------------
 setState / root.render  call component fns -> new elements -> reconcile children      1. before-mutation (getSnapshotBeforeUpdate)
 context change          -> build "work-in-progress" fiber tree, mark effects/flags    2. MUTATION: DOM insert/update/delete,
 parent re-render                                                                         detach refs, run layout-effect CLEANUPS
                                                                                        3. swap current <-> workInProgress tree
                                                                                        4. LAYOUT: attach refs, run useLayoutEffect
                                                                                           (and componentDidMount/Update)
                                                                                     -> browser PAINTS
                                                                                     5. PASSIVE: useEffect cleanups then effects
```
Consequences:
* **Render can run more than once** for one commit (StrictMode dev, Concurrent rendering discarding work, Suspense). So render must be a pure calculation: no `fetch`, no mutating outside variables, no `Math.random()`/`Date.now()` in output that must be stable, no writing refs.
* DOM is only touched in commit. "Component rendered" does **not** mean "DOM changed". If the diff is empty, DOM is untouched (React does not repaint the whole tree).
* Layout effects block paint; passive effects (`useEffect`) normally run after paint. Nuance: if the render was caused by a **discrete user event** (click, keypress), React flushes passive effects synchronously at the end of the commit (before the next event is processed), so don't assume "always after paint" for exact-timing hacks.

### 2.4 Reconciliation (diffing) - the rules

Diffing two arbitrary trees is O(n^3). React uses two heuristics to get O(n):
1. Elements of **different type** produce different trees.
2. The developer hints stable children with **`key`**.

Algorithm, per position in the tree:
```
compare old fiber vs new element at same position
 ├─ different `type` (div -> span, <A/> -> <B/>)  => UNMOUNT old subtree (state lost, effects cleaned), MOUNT new
 ├─ same host type (both <div>)                   => keep DOM node, diff props/attributes, recurse into children
 └─ same component type                           => keep instance (state preserved), re-render with new props, recurse
children lists:
 ├─ no keys      => matched by INDEX
 └─ with keys    => matched by KEY (moves detected, insert/delete only what changed)
```
**Position matters, not just type.** Example:
```jsx
{isAdmin ? <Panel /> : <Panel />}         // same type, same slot => state PRESERVED (surprise)
{isAdmin ? <Panel key="a" /> : <Panel key="b" />}  // different key => remount (state reset)
{cond && <Form />}                          // when false slot holds `false` - siblings' positions are stable
```
Defining a component **inside** another component makes a **new type every render** => full remount every render (lost input focus, lost state). Classic bug:
```jsx
function Page() {
  function Row() { return <input/>; }   // NEW function identity each Page render
  return <Row />;                        // remounts each time -> input loses focus on each keystroke
}
```
Fix: define at module level, or pass data as props.

**Resetting state on purpose:** `<Profile key={userId} />` - changing key remounts and clears state (better than an effect that resets state).

### 2.5 Keys and the index-as-key bug (worked example)

```jsx
function List() {
  const [items, setItems] = useState([{id:'a',t:'Apple'},{id:'b',t:'Banana'},{id:'c',t:'Cherry'}]);
  return (<>
    <button onClick={() => setItems(items.slice(1))}>Remove first</button>
    {items.map((it, i) => <Row key={i} label={it.t} />)}   // BUG: key = index
  </>);
}
function Row({ label }) {
  const [text, setText] = useState("");        // uncontrolled-ish local state
  return <div>{label}: <input value={text} onChange={e => setText(e.target.value)} /></div>;
}
```
Steps:
1. Type "hello" in the **Apple** row's input (Row key 0 holds state `"hello"`).
2. Click "Remove first". New list is [Banana, Cherry] with keys **[0, 1]** (verified: index keys before `[0,1,2]`, after `[0,1]`; id keys after `['b','c']`).
3. React matches by key: key 0 instance (state "hello") is **reused** and now receives `label="Banana"`. Key 2 instance is unmounted.
4. Result: Banana row shows "hello". State stuck to the *slot*, not to the item. Also with reordering/inserting at top, all rows re-render with shifted props and inputs/focus/animations misbehave.
Fix: `key={it.id}`. Then key 'a' is unmounted, 'b' and 'c' preserved with their own state.

Key rules: unique **among siblings** only; stable across renders (never `Math.random()` - remounts every time); index is acceptable only for static lists never reordered/filtered and without local state; keys are not passed as props (`props.key` is undefined). Missing key => console warning and index fallback.

### 2.6 Virtual DOM - the myths

* "Virtual DOM makes React fast" - **half myth**. The VDOM is overhead vs hand-tuned DOM code; the value is a *declarative model* with acceptable (not optimal) performance and the ability to schedule work. Svelte/Solid skip VDOM and can be faster.
* "React re-renders the entire DOM" - no; it re-runs component functions and commits only diffs.
* "Re-render = DOM update" - no. Re-render = function call + diff. Often result is identical, nothing touches the DOM.
* "Shadow DOM = Virtual DOM" - unrelated (Shadow DOM is a browser feature for encapsulation).
* Today the term React itself uses is the **element tree / Fiber tree**; the "virtual DOM" is folklore.

### 2.7 Fiber architecture (conceptual)

Before React 16 (the "stack reconciler") rendering was a recursive, synchronous call stack; a big tree blocked the main thread (jank). **Fiber** (16) re-implemented reconciliation as a linked data structure that can be paused.

A **fiber** = a unit of work = a JS object per component instance / host node:
```
Fiber { type, key, stateNode(DOM node|instance), memoizedState(= hooks linked list),
        memoizedProps, pendingProps, updateQueue, flags(effects), lanes(priority),
        child, sibling, return(parent), alternate(the other tree's twin) }
```
Two trees (double buffering): **current** (what's on screen) and **workInProgress** (being built). Each fiber's `alternate` points to its twin. On commit the root pointer flips: WIP becomes current.

Work loop (conceptual):
```
performUnitOfWork(fiber):
   beginWork(fiber)   -> call component, reconcile children, return first child
   if no child: completeWork(fiber) -> create/prepare DOM node (off-screen), bubble flags/lanes; go to sibling, else return to parent
loop: while (work && !shouldYield()) work = performUnitOfWork(work)      // concurrent mode
```
* `shouldYield()` checks the ~5 ms time slice; if the browser needs the thread (input, paint) React **yields** and continues later. Legacy sync rendering never yields.
* **Lanes** = bit-flag priority model. Updates from a click are `SyncLane`/discrete (urgent); updates in `startTransition` get a `TransitionLane` (interruptible, low). A higher-priority update can **interrupt** and restart a low-priority render; the abandoned work-in-progress is thrown away (another reason render must be pure).
* Concurrent rendering is **opt-in by feature** (createRoot + `useTransition`, `useDeferredValue`, Suspense). `createRoot` alone does not make everything time-sliced; urgent updates still render synchronously.
* Commit is always synchronous and atomic (no half-updated UI ever visible).

Concurrent features (concept): `useTransition`/`startTransition` mark updates as non-urgent; `useDeferredValue` gives a lagging copy of a value; Suspense boundaries let subtrees "wait" without blocking siblings; selective/streaming hydration in SSR.

### 2.8 Batching

**React 17 and earlier:** batched only inside React event handlers. In `setTimeout`, promises, native handlers each `setState` re-rendered immediately.
**React 18 (createRoot):** **automatic batching everywhere** - timeouts, promises, native events, all updates in the same tick coalesce into one render.

```jsx
function onClick() {
  setA(1); setB(2);           // 1 render (both React 17 and 18, inside a React handler)
}
setTimeout(() => { setA(1); setB(2); }, 0);
// React 17: 2 renders. React 18 with createRoot: 1 render.   [expected behaviour]
```
Opt out with `flushSync` (react-dom) - forces synchronous DOM update, e.g. to measure/scroll right after a state change:
```jsx
import { flushSync } from "react-dom";
flushSync(() => setTodos(next));      // DOM updated here
listRef.current.lastChild.scrollIntoView();
```
Use rarely; hurts performance. Not allowed inside lifecycle/effects rendering.

### 2.9 State is a snapshot (asynchronous-looking updates)

`setCount(n)` does not change the variable; it **enqueues an update and schedules a render**. The next render re-runs the function; `useState` returns the new value.

```jsx
function Counter() {
  const [count, setCount] = useState(0);
  function handle() {
    setCount(count + 1);
    setCount(count + 1);
    setCount(count + 1);
    console.log(count);          // logs 0 (snapshot), not 1
  }
  // After click: count === 1 (all three enqueue "set to 0+1")          [verified arithmetic: 1]
}
```
Functional updates apply in sequence to the latest queued value:
```jsx
setCount(c => c + 1); setCount(c => c + 1); setCount(c => c + 1);   // -> 3   [verified]
```
Queue processing rule (replaying the queue from the last committed state):
* `setCount(5); setCount(c => c + 1); setCount(42);` from 0 => **42** (verified)
* `setCount(5); setCount(c => c + 1); setCount(42); setCount(c => c * 2);` => **84** (verified)
* Each item is either "replace with value" or "apply function to the running result".

Use functional update when the new state depends on the previous one (and always inside async callbacks/intervals). Use the value form when it doesn't.

Bail-out: if the new state is `Object.is`-equal to the current, React may skip re-rendering (it might still call the component once more before bailing, but won't render children or run effects). `Object.is(NaN,NaN)` is true, `Object.is(0,-0)` false, `Object.is({}, {})` false (verified) - which is why mutating then setting the same reference does nothing.

### 2.10 Stale closures - worked examples

**Example A - interval reads stale value**
```jsx
function Timer() {
  const [n, setN] = useState(0);
  useEffect(() => {
    const id = setInterval(() => setN(n + 1), 1000);   // captures n = 0 forever (deps [])
    return () => clearInterval(id);
  }, []);
  return n;     // sticks at 1
}
```
The effect ran once with closure `n = 0`; every tick does `setN(0+1)`. Fixes: (1) `setN(c => c + 1)` (best, deps stay `[]`), (2) add `n` to deps (re-creates the interval each tick), (3) ref holding latest value.

**Example B - handler after await**
```jsx
async function save() {
  await api.save(form);
  alert(`Saved ${form.name}`);     // `form` is the snapshot from when the click happened, even if user kept typing
}
```
That is often *desired*. If you need the latest, read a ref.

**Example C - delayed log**
```jsx
function handleClick() { setCount(count + 1); setTimeout(() => console.log(count), 3000); }
// click at count=0 then quickly count becomes 5 by other clicks: the timeout still logs 0
```
**Latest-ref pattern** (when you truly need the latest inside a long-lived callback):
```jsx
const latest = useRef(value);
useEffect(() => { latest.current = value; });   // update after commit
useEffect(() => { const id = setInterval(() => console.log(latest.current), 1000); return () => clearInterval(id); }, []);
```
(React 19.2 adds `useEffectEvent` for exactly this "non-reactive logic in effects" case - it is only callable from effects and is not a dependency.)

### 2.11 What triggers a re-render - and what doesn't

Triggers: (1) `setState`/`dispatch` with a different value; (2) parent re-rendered (default: **all children re-render**, regardless of props); (3) a consumed **context** value changed (`Object.is` on provider `value`); (4) `useSyncExternalStore` store changed; (5) key/type change remounts; (6) `forceUpdate` equivalents (class), root `render`.

**Does NOT trigger:** mutating a `useRef().current`; assigning to a module/local `let` variable; mutating state objects in place (`arr.push`); `setState` with an identical value; changing DOM directly.

```jsx
let clicks = 0;                      // module variable
function Bad() {
  let local = 0;                     // reset to 0 every render
  const r = useRef(0);
  return <button onClick={() => { clicks++; local++; r.current++; }}>{clicks}/{local}/{r.current}</button>;
  // clicking changes 3 things but NOTHING re-renders; the button text stays 0/0/0 until something else renders
}
```
Re-render cascade: a render of `<App>` re-renders every descendant unless wrapped in `React.memo` (props shallow-compare via `Object.is`) **or** the child element is the same reference (e.g. passed as `children` from a parent that did not re-render):
```jsx
function Slow({children}) { const [v,setV]=useState(0); return <div onMouseMove={e=>setV(e.clientX)}>{v}{children}</div>; }
<Slow><ExpensiveTree/></Slow>   // ExpensiveTree element was created by the PARENT of Slow; when Slow's state changes
                                // that element reference is identical => React skips re-rendering it. "State colocation via children".
```

### 2.12 Hooks internals: the linked list and the Rules of Hooks

Conceptually each function-component fiber has `memoizedState` = **head of a singly linked list**, one node per hook call, in call order:
```
fiber.memoizedState
   |
   v
 [hook#1 useState  ] --next--> [hook#2 useEffect] --next--> [hook#3 useRef] --next--> [hook#4 useMemo] --> null
  {memoizedState:'Ann',         {memoizedState: effect,     {current: ...}            {[value,deps]}
   queue:{pending updates}}      deps:[id]}
```
* **Mount:** each `useX()` call creates a node and appends it.
* **Update:** the dispatcher walks the *same list by position*: 1st call reads node 1, 2nd call node 2 ...  There is **no name or key** - identity is call order.
* Hence the **Rules of Hooks**: call hooks only at the top level (no `if`, loops, early `return` before a hook, nested functions), and only in components/custom hooks. Break order and every later hook reads the wrong node.

Verified mini-simulation (Node 22, `hooks.js`):
```js
let fiber = null;
function component(fn) { return { fn, hooks: null, tail: null, cursor: null, mounted: false }; }
function useState(initial) {
  const f = fiber; let hook;
  if (!f.mounted) {                                   // MOUNT: append node
    hook = { state: initial, next: null };
    if (!f.hooks) f.hooks = hook; else f.tail.next = hook;
    f.tail = hook;
  } else {                                            // UPDATE: next node BY POSITION
    hook = f.cursor ? f.cursor.next : f.hooks;
    if (!hook) throw new Error("Rendered more hooks than during the previous render");
  }
  f.cursor = hook;
  const set = v => { hook.state = typeof v === "function" ? v(hook.state) : v; };
  return [hook.state, set];
}
function render(f) { fiber = f; f.cursor = null; const out = f.fn(); f.mounted = true; fiber = null; return out; }
```
Outputs actually produced:
```
good component  render twice           -> [ 'name', 42 ] [ 'name', 42 ]
conditional hook (flag false -> true)  -> mount { a:'A', b:'-', c:'C' } ; next render: ERR: Rendered more hooks than during the previous render
same hook COUNT but different branch   -> [ 'first-slot', 'only-when-flag' ] both times (2nd render reads the OLD slot value: silently wrong!)
```
Takeaway: a conditional hook either throws (count changed) or - worse - silently reads another hook's state (count same). That is the real reason for the rule (and for the `eslint-plugin-react-hooks` `rules-of-hooks` lint). `use()` (React 19) is the one API that *may* be called conditionally because it is not stored in the list.

`useState` update path: `dispatchSetState` puts an update object on the hook's **circular queue**, marks the fiber with a **lane**, schedules a render. On the next render the hook processes the queue from its base state (that's the queue arithmetic in 2.9). `useReducer` is the general form; `useState` is a reducer `(s, a) => typeof a === 'function' ? a(s) : a`.

Lazy init: `useState(() => expensive())` runs initializer only on mount; `useState(expensive())` runs it every render (result ignored after mount).

### 2.13 Effect timeline

```
Mount:      render -> commit DOM -> [layout: useLayoutEffect setup, refs attached] -> PAINT -> useEffect setup
Update:     render -> commit DOM -> [layout: OLD layout cleanup, NEW layout setup]  -> PAINT -> OLD effect cleanup -> NEW effect setup
                                  (only for effects whose deps changed)
Unmount:    layout cleanups, DOM removal, then effect cleanups (cleanup for all effects of that component)
Strict Dev: mount -> setup -> (simulated unmount) cleanup -> setup again        (see 4.4)
```
Order rules: within one component effects run **in declaration order**; children's effects run **before parents'** (bottom-up), both for setup. For an update with changed deps: **all cleanups of that commit run before any setup**.

Example:
```jsx
function Child() { useEffect(() => { console.log("child effect"); return () => console.log("child cleanup"); }); return null; }
function Parent() { useEffect(() => { console.log("parent effect"); return () => console.log("parent cleanup"); }); return <Child/>; }
// mount (production): child effect, parent effect
// re-render Parent (no deps => both re-run): child cleanup, parent cleanup, child effect, parent effect   [expected behaviour]
```

---

## 3. useState and useReducer in practice

```jsx
const [user, setUser] = useState({ name: "Ann", address: { city: "Pune" } });
setUser(u => ({ ...u, address: { ...u.address, city: "Goa" } }));   // immutable nested update
```
Immutable update cheat-sheet: add `[...a, x]`; remove `a.filter(...)`; replace `a.map(...)`; object `{...o, k: v}`; nested spread each level (or **Immer**). Never `a.sort()` / `a.reverse()` / `a.push()` on state (mutate) - use `[...a].sort()` or `a.toSorted()` (ES2023).

`useReducer` when: many related fields, next state depends on several actions, transitions must be testable/centralised.
```jsx
function reducer(state, action) {
  switch (action.type) {
    case "add":    return { ...state, items: [...state.items, { id: action.id, text: action.text, done: false }] };
    case "toggle": return { ...state, items: state.items.map(t => t.id === action.id ? { ...t, done: !t.done } : t) };
    case "remove": return { ...state, items: state.items.filter(t => t.id !== action.id) };
    default: throw new Error("unknown action " + action.type);
  }
}
const [state, dispatch] = useReducer(reducer, { items: [] });
dispatch({ type: "add", id: 1, text: "learn hooks" });
```
Verified in Node: add(1), add(2), toggle(1), remove(2) from `{items:[]}` => `{"items":[{"id":1,"text":"a","done":true}]}`, the original state object untouched (`items.length` 0) and the new state is a different reference (`true`).

Reducer rules: pure, no side effects, no mutation, returns new state for real changes and the **same reference** for no-ops (lets React bail out). In StrictMode React calls reducers twice to catch impurity.

`dispatch` identity is stable (safe to omit from deps / pass to memoised children) - a big reason to use reducer + context (2 contexts: state and dispatch).

Derived data: do not store what you can compute (`const total = items.reduce(...)` during render, not `useState` + effect).

---

## 4. Effects in depth

### 4.1 What an effect is
`useEffect(setup, deps?)` - after the commit React runs `setup`; the optional returned function is `cleanup`. The model: **"synchronise this piece of the outside world with the current props/state"**, not "on mount" / "on update".

### 4.2 Dependency array semantics
| deps | runs |
|---|---|
| omitted | after **every** render |
| `[]` | after mount only (and cleanup on unmount) |
| `[a, b]` | after mount, and after any render where `Object.is` says `a` or `b` changed |

Rules:
* Deps = **every reactive value** (props, state, variables/functions derived from them) used inside. Not optional - the lint rule `react-hooks/exhaustive-deps` is the truth-teller. Lying about deps => stale closures.
* Objects/arrays/functions created in render are **new each render** => effect runs every render if listed. Fix: move inside the effect, hoist out of the component, depend on primitives (`user.id` not `user`), or memoise (`useMemo/useCallback`) only if needed.
* Refs (`ref.current`), setState functions, dispatch are stable, need not be listed. Module-level values aren't reactive.
* The array isn't "watch these"; it's "these are the reactive values my code reads".

### 4.3 Cleanup
Cleanup runs (1) before the next setup when deps changed, (2) on unmount. It sees the **closure of the render that created it** (the old values) - that's exactly what you want to unsubscribe the old thing.
```jsx
useEffect(() => {
  const conn = createConnection(roomId);
  conn.connect();
  return () => conn.disconnect();        // disconnects the OLD roomId's connection
}, [roomId]);
```

### 4.4 StrictMode double-invocation (development only)
`<React.StrictMode>` in dev (React 18+): components/reducers/initialisers render **twice** (second result discarded, logs dimmed), and on **mount** every effect runs **setup -> cleanup -> setup** ("simulated unmount/remount"). React 19 also double-invokes ref callbacks and, since 19, reuses the memoised results from the first render for the second one for `useMemo`/`useCallback`.

Why: React wants components resilient to being unmounted and remounted with preserved state (needed for Offscreen/Activity, fast refresh, future reuse of state). If your effect is a correct *synchronisation* with a proper cleanup, running it twice is harmless; if not, the double run shows the bug now rather than in production.

```jsx
useEffect(() => { console.log("connect"); return () => console.log("disconnect"); }, []);
// dev (StrictMode):  connect, disconnect, connect      production: connect       [expected behaviour]
```
Wrong reaction: removing StrictMode or a `useRef` "didRun" hack. Right reaction: add cleanup (abort fetch, unsubscribe, clear timers). For "run once per app load" things (analytics init, auth bootstrap) put them at module level or in an event handler, not in an effect.

### 4.5 useLayoutEffect and useInsertionEffect
* `useLayoutEffect` - runs **synchronously after DOM mutation, before paint**. Use to **measure** layout (`getBoundingClientRect`) and re-position (tooltips) so the user never sees the wrong frame. Blocks paint - keep short. Warns on the server (SSR) because no layout exists.
* `useInsertionEffect` - runs **before any DOM mutation/layout effects**; only for **CSS-in-JS libraries** injecting `<style>` rules. App code shouldn't use it; cannot schedule updates or read refs.
* Order: insertion -> (DOM mutation) -> layout -> paint -> passive.

```jsx
function Tooltip({ anchorRef, children }) {
  const ref = useRef(null); const [pos, setPos] = useState({ top: 0, left: 0 });
  useLayoutEffect(() => {
    const a = anchorRef.current.getBoundingClientRect();
    const h = ref.current.getBoundingClientRect().height;
    setPos({ top: a.top - h, left: a.left });    // re-render happens BEFORE paint -> no flicker
  }, [anchorRef]);
  return <div ref={ref} style={{ position: "fixed", ...pos }}>{children}</div>;
}
```

### 4.6 Data fetching in effects - pitfalls
1. **Race condition:** type "a", "ab" quickly; responses may arrive out of order and the older overwrites the newer.
2. **No cleanup:** setState after unmount / after a newer request.
3. **StrictMode double fetch** in dev (harmless with cleanup; still wasteful).
4. **Waterfalls:** parent fetches, renders child, child fetches, ... each round trip serialised. Fix: hoist fetches / route loaders / prefetch / parallel `Promise.all` / server components.
5. No caching, dedup, retries, background refresh, pagination state - all hand-rolled. Use TanStack Query.
6. Error/loading state combinatorics.

Correct hand-written version (AbortController + ignore flag):
```jsx
function useUser(id) {
  const [state, setState] = useState({ data: null, error: null, loading: true });
  useEffect(() => {
    const ctrl = new AbortController();
    setState(s => ({ ...s, loading: true, error: null }));
    fetch(`/api/users/${id}`, { signal: ctrl.signal })
      .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then(data => setState({ data, error: null, loading: false }))
      .catch(err => {
        if (err.name === "AbortError") return;          // expected on cleanup, ignore
        setState({ data: null, error: err, loading: false });
      });
    return () => ctrl.abort();                            // cancels in-flight request for the OLD id
  }, [id]);
  return state;
}
```
The **ignore flag** alternative (when you can't abort, e.g. a library promise): `let ignore = false; ... .then(d => { if (!ignore) setX(d); }); return () => { ignore = true; };`. Abort also saves bandwidth and server work (your Spring Boot side sees a cancelled request).

Note `fetch` only rejects on network failure - HTTP 500 resolves; always check `r.ok`.

### 4.7 "You Might Not Need an Effect" - rules
| Situation | Don't | Do |
|---|---|---|
| Derived value | `useEffect(() => setFull(first+last))` | `const full = first + " " + last` in render |
| Expensive derived | effect + state | `useMemo` (or just compute) |
| Reset state when prop changes | effect calling `setX(initial)` | `key={prop}` on the component |
| Adjust part of state on prop change | effect | compute during render, or store id and derive |
| User-caused action (submit, buy) | effect watching `submitted` state | do it in the **event handler** |
| Notify parent of change | effect calling `onChange` | call `onChange` in the same handler that sets state |
| Chain of effects setting each other's state | multiple effects | compute in one handler / reducer |
| Subscribe to external store | effect + state | `useSyncExternalStore` |
| Fetch on mount | raw effect | TanStack Query / router loader / RSC |
| App init once | effect with ref guard | module-level / entry file |

Test: "Is it there because the **component was displayed** (effect) or because **the user did something** (handler)?"

---

## 5. The other hooks

### 5.1 useRef
```jsx
const inputRef = useRef(null);            // DOM ref
<input ref={inputRef} />;   inputRef.current.focus();      // after mount (effect/handler), never during render

const renders = useRef(0); renders.current++;               // mutable box; changing does NOT re-render
```
Use for: DOM nodes, timer ids, previous values, "latest value" mirrors, instance-like mutable data. Don't read/write `ref.current` during render (except lazy init) - it breaks purity/concurrency.

Previous value:
```jsx
function usePrevious(value) {
  const ref = useRef();
  useEffect(() => { ref.current = value; });     // runs AFTER render, so during render ref still holds the old one
  return ref.current;
}
```
Callback refs: `ref={node => {...}}` (called with node on attach, with null on detach; **React 19 lets it return a cleanup function**, then it is *not* called with null).

### 5.2 useMemo / useCallback / React.memo
* `useMemo(() => compute(a,b), [a,b])` caches a **value**; `useCallback(fn, deps)` == `useMemo(() => fn, deps)` caches a **function identity**. Both are *performance hints*, React may discard cache (semantic guarantee only "won't break if dropped").
* They only matter for **referential equality** where identity is consumed: (1) props of `React.memo` children, (2) dependency arrays of hooks, (3) context provider values, (4) genuinely expensive calculation (> ~1 ms measured, e.g. sorting 10k rows).
* `React.memo(Comp)` - skip re-render if all props are `Object.is`-equal. Useless if you pass a fresh `{}`, `[]`, `() => {}` each render - hence `useCallback`/`useMemo` for those props. Custom comparator as 2nd arg (rare, bug-prone).
```jsx
const Row = React.memo(function Row({ order, onSelect }) { /* expensive */ });
function Table({ orders }) {
  const [sel, setSel] = useState(null);
  const onSelect = useCallback(id => setSel(id), []);      // stable identity
  return orders.map(o => <Row key={o.id} order={o} onSelect={onSelect} />);
}
```
**When they harm:** memo everywhere adds comparison cost + memory + code noise; memoising cheap things is slower than recomputing; wrong deps give stale values; `children` prop `<Row><b/></Row>` defeats `memo` (new element each render).
**React Compiler** (stable 1.0, opt-in build plugin `babel-plugin-react-compiler`): auto-memoises components/hooks at build time by analysing code following the Rules of React, making most manual `useMemo/useCallback/memo` unnecessary. Still valid to write them for escape hatches (effect deps semantics).

### 5.3 useContext and the re-render problem
```jsx
const ThemeCtx = createContext("light");
function ThemeProvider({ children }) {
  const [theme, setTheme] = useState("light");
  const value = useMemo(() => ({ theme, toggle: () => setTheme(t => t === "light" ? "dark" : "light") }), [theme]);
  return <ThemeCtx.Provider value={value}>{children}</ThemeCtx.Provider>;   // React 19: <ThemeCtx value={value}>
}
const useTheme = () => { const c = useContext(ThemeCtx); if (!c) throw new Error("outside provider"); return c; };
```
Problem: **every consumer re-renders whenever `value` identity changes** - even if it only uses one field. `React.memo` on the consumer does not stop it (context bypasses props). A provider that re-renders (parent state change) with `value={{a,b}}` inline creates a new object every time -> all consumers re-render.
Mitigations:
1. `useMemo` the value (or keep value stable).
2. **Split contexts** by change frequency / read vs write: `StateCtx` + `DispatchCtx`; `UserCtx` vs `ThemeCtx`.
3. Put provider **low** in the tree; pass `children` so provider's own re-render doesn't re-render the subtree.
4. Extract consumers into small memoised components.
5. Selector-based stores (Zustand, Redux `useSelector`, `use-context-selector`, Jotai atoms) re-render only if selected slice changes.
Context is good for low-frequency, app-wide values (theme, locale, auth user); bad as a high-frequency store (form keystrokes, mouse position).

### 5.4 Concurrent-era hooks
**useTransition** - mark a state update non-urgent so the UI stays responsive:
```jsx
const [isPending, startTransition] = useTransition();
function onTab(next) { startTransition(() => setTab(next)); }   // keep old tab visible, interruptible render, isPending for spinner
```
Only wrap the state *setters*; async callbacks inside transitions in 18 were not tracked (in 19, async functions are supported as "Actions": `startTransition(async () => {...})`, but state updates after `await` must be wrapped again).
**useDeferredValue** - a lagging copy: `const deferredQuery = useDeferredValue(query);` input stays instantly responsive; heavy list renders with deferred value and is interruptible. Pair with `memo` on the list. (Like debounce, but no fixed delay; render-level, does not reduce network calls.)
**Suspense** - `<Suspense fallback={<Spinner/>}>` shows fallback while a child suspends (lazy component, `use(promise)`, Suspense-enabled data library/RSC). Nest boundaries to control granularity; combine with transitions to avoid replacing already-visible content with a fallback.
**Code splitting:**
```jsx
const Reports = lazy(() => import("./Reports"));     // default export required
<Suspense fallback={<Skeleton/>}><Reports/></Suspense>
```
Split at routes first; then heavy widgets (charts, editors).

### 5.5 useId, useSyncExternalStore, others
* `useId()` - stable unique id **consistent between server and client** (SSR-safe); for `htmlFor`/`aria-*` linking. Not for list keys; don't use for CSS selectors (contains colons).
* `useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?)` - the correct way to read external mutable stores (Redux/Zustand internals, `navigator.onLine`, `matchMedia`) safely under concurrent rendering (no tearing). `getSnapshot` must return a cached/stable value (a new object each call = infinite loop).
```jsx
function useOnline() {
  return useSyncExternalStore(
    cb => { window.addEventListener("online", cb); window.addEventListener("offline", cb);
            return () => { window.removeEventListener("online", cb); window.removeEventListener("offline", cb); }; },
    () => navigator.onLine,
    () => true);                          // server snapshot
}
```
* `useImperativeHandle(ref, () => ({ focus() {...} }), [])` - expose a limited imperative API to parents (rare).
* `useDebugValue` - label in DevTools for custom hooks.

---

## 6. Custom hooks (design + code)

A custom hook = function starting with `use` that calls other hooks. Shares **logic**, never **state** (each call gets its own state). Design rules: name what it *does* (`useOnlineStatus`, not `useMount`); return the minimum (value, or `[value, setter]`, or object); accept stable inputs or document that; clean up everything; don't hide unrelated effects.

```jsx
// useDebounce - value debouncing (debounce logic verified in Node: d('a'); d('ab'); d('abc') within window => one call with 'abc')
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const id = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(id);            // each new value cancels the previous timer
  }, [value, delay]);
  return debounced;
}
```
```jsx
// useFetch - with abort + refetch + race safety
function useFetch(url, options) {
  const [state, setState] = useState({ data: null, error: null, loading: !!url });
  const [tick, setTick] = useState(0);
  const optsRef = useRef(options); optsRef.current = options;          // avoid making `options` a dependency
  useEffect(() => {
    if (!url) return;
    const ctrl = new AbortController();
    setState(s => ({ ...s, loading: true, error: null }));
    fetch(url, { ...optsRef.current, signal: ctrl.signal })
      .then(r => { if (!r.ok) throw new Error(`HTTP ${r.status}`); return r.json(); })
      .then(data => setState({ data, error: null, loading: false }))
      .catch(e => { if (e.name !== "AbortError") setState({ data: null, error: e, loading: false }); });
    return () => ctrl.abort();
  }, [url, tick]);
  const refetch = useCallback(() => setTick(t => t + 1), []);
  return { ...state, refetch };
}
```
```jsx
// useLocalStorage - SSR-safe, lazy init, functional updates, sync across tabs
function useLocalStorage(key, initial) {
  const [value, setValue] = useState(() => {
    try { const raw = localStorage.getItem(key); return raw !== null ? JSON.parse(raw) : (initial instanceof Function ? initial() : initial); }
    catch { return initial instanceof Function ? initial() : initial; }
  });
  useEffect(() => {
    try { localStorage.setItem(key, JSON.stringify(value)); } catch { /* quota / private mode */ }
  }, [key, value]);
  useEffect(() => {
    const onStorage = e => { if (e.key === key && e.newValue !== null) setValue(JSON.parse(e.newValue)); };
    window.addEventListener("storage", onStorage);            // fires in OTHER tabs only
    return () => window.removeEventListener("storage", onStorage);
  }, [key]);
  return [value, setValue];
}
```
```jsx
// useEventListener, useOnClickOutside, usePrevious (5.1), useToggle
function useOnClickOutside(ref, handler) {
  const h = useRef(handler); h.current = handler;
  useEffect(() => {
    const l = e => { if (ref.current && !ref.current.contains(e.target)) h.current(e); };
    document.addEventListener("mousedown", l);
    return () => document.removeEventListener("mousedown", l);
  }, [ref]);
}
```

---

## 7. Error boundaries, portals, refs, forms, composition

### 7.1 Error boundaries
Catch errors thrown **during render, in lifecycle/effects of descendants, and in constructors** - render a fallback instead of unmounting the whole app. They do **not** catch: event-handler errors (use try/catch), async code (promises/timeouts - unless you `throw` inside a state setter/`use`), SSR errors, errors in the boundary itself. Still **class-only** API (`getDerivedStateFromError`, `componentDidCatch`); use `react-error-boundary` in function-component code bases.
```jsx
class ErrorBoundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) { return { error }; }         // render phase: switch to fallback
  componentDidCatch(error, info) { reportToSentry(error, info.componentStack); }   // commit phase: side effects/logging
  render() { return this.state.error ? this.props.fallback : this.props.children; }
}
// react-error-boundary:
<ErrorBoundary FallbackComponent={Fallback} onReset={() => queryClient.clear()} resetKeys={[orderId]}>
  <OrderDetails />
</ErrorBoundary>
```
Place at route level + around risky widgets. React 19 adds root options `onCaughtError`, `onUncaughtError`, `onRecoverableError` for central logging.

### 7.2 Portals
```jsx
import { createPortal } from "react-dom";
function Modal({ open, onClose, children }) {
  if (!open) return null;
  return createPortal(
    <div className="backdrop" onClick={onClose}><div role="dialog" aria-modal="true" onClick={e => e.stopPropagation()}>{children}</div></div>,
    document.body);
}
```
Renders DOM elsewhere (escaping `overflow:hidden`, `z-index`, transforms) but stays in the **React tree**: context works, and **events bubble through the React tree** to the logical parent (not the DOM parent).

### 7.3 forwardRef and ref as prop
React 18: function components can't receive `ref` as a prop; wrap with `forwardRef`.
```jsx
const FancyInput = forwardRef(function FancyInput(props, ref) { return <input ref={ref} {...props} />; });
```
React 19: `ref` is a **regular prop** for function components; `forwardRef` still works but is deprecated (codemod available), will be removed later.
```jsx
function FancyInput({ ref, ...props }) { return <input ref={ref} {...props} />; }
```

### 7.4 Controlled vs uncontrolled inputs
| | Controlled | Uncontrolled |
|---|---|---|
| Source of truth | React state (`value` + `onChange`) | The DOM (read via `ref` / `FormData` on submit) |
| Re-render per keystroke | yes | no |
| Instant validation / formatting / dependent fields | easy | harder |
| Perf on big forms | can lag | great |
| `defaultValue` | n/a | initial only |
```jsx
<input value={name} onChange={e => setName(e.target.value)} />          // controlled
<input defaultValue="Ann" ref={ref} />                                    // uncontrolled
<input type="file" />                                                     // always uncontrolled
```
Gotchas: `value` without `onChange` => read-only + warning; `value={undefined}` -> `value="x"` switches uncontrolled->controlled (warning): init with `""`. Uncontrolled + FormData: `new FormData(e.currentTarget)`; React 19 form `action` uses this.

### 7.5 Forms: controlled vs React Hook Form + zod
React Hook Form (RHF) registers uncontrolled inputs via refs => minimal re-renders, built-in dirty/touched/errors, resolvers for zod/yup.
```jsx
import { useForm } from "react-hook-form";
import { zodResolver } from "@hookform/resolvers/zod";
import { z } from "zod";

const schema = z.object({
  customerEmail: z.string().email("Invalid email"),
  quantity: z.coerce.number().int().min(1, "Min 1").max(100),
  notes: z.string().max(200).optional(),
});
type FormValues = z.infer<typeof schema>;      // TypeScript

function OrderForm({ onSubmit }: { onSubmit: (v: FormValues) => Promise<void> }) {
  const { register, handleSubmit, setError, formState: { errors, isSubmitting } } =
    useForm<FormValues>({ resolver: zodResolver(schema), defaultValues: { quantity: 1 } });
  const submit = handleSubmit(async values => {
    try { await onSubmit(values); }
    catch (e) { setError("root", { message: "Server rejected the order" }); }     // map server errors
  });
  return (
    <form onSubmit={submit} noValidate>
      <label htmlFor="email">Email</label>
      <input id="email" {...register("customerEmail")} aria-invalid={!!errors.customerEmail} />
      {errors.customerEmail && <p role="alert">{errors.customerEmail.message}</p>}
      <input type="number" {...register("quantity")} />
      {errors.quantity && <p role="alert">{errors.quantity.message}</p>}
      {errors.root && <p role="alert">{errors.root.message}</p>}
      <button disabled={isSubmitting}>Create</button>
    </form>
  );
}
```
Use controlled state for tiny forms and inputs needing live derived UI; RHF for anything with many fields or validation. `Controller`/`useController` wrap controlled UI libs (MUI Select). **Client validation is UX only** - the Spring Boot `@Valid` bean validation is the authority; map its 400 field errors back with `setError`.

### 7.6 Composition patterns
| Pattern | Idea | Use when | Downsides |
|---|---|---|---|
| **children / slots** | Parent passes JSX: `<Card header={<H/>}>body</Card>` | Layout containers, avoid prop drilling | none; default choice |
| **Render props** | Prop is a function returning JSX: `<Mouse>{pos => <Dot {...pos}/>}</Mouse>` | Share stateful logic + let caller decide markup (pre-hooks); still good for virtualised lists / headless UI | nesting ("callback hell"); mostly replaced by hooks |
| **Compound components** | Parent + subcomponents share implicit state via context: `<Tabs><Tabs.List/><Tabs.Panel/></Tabs>` | Flexible design-system components (Tabs, Accordion, Select) | needs context; children order/structure contract |
| **HOC** | `withAuth(Component)` returns wrapped component | Cross-cutting concerns in legacy code, libs (`connect`, `withRouter`) | wrapper hell, prop collisions, static hoisting, ref forwarding; prefer hooks |
| **Custom hook** | logic reuse | 90% of "share behaviour" cases | logic only |
| **Container/presentational** | split data and UI | testability | hooks made it mostly unnecessary |
Compound example skeleton:
```jsx
const TabsCtx = createContext(null);
function Tabs({ defaultValue, children }) { const [v, setV] = useState(defaultValue); return <TabsCtx.Provider value={{ v, setV }}>{children}</TabsCtx.Provider>; }
Tabs.Tab   = ({ value, children }) => { const c = useContext(TabsCtx); return <button role="tab" aria-selected={c.v === value} onClick={() => c.setV(value)}>{children}</button>; };
Tabs.Panel = ({ value, children }) => { const c = useContext(TabsCtx); return c.v === value ? <div role="tabpanel">{children}</div> : null; };
```

---

## 8. State management decision guide

```
Is it used by ONE component?                  -> useState / useReducer (local)
Used by a few siblings?                       -> lift to nearest common parent (props)
Passed through many layers, changes rarely?   -> Context (theme, locale, auth user)
Complex client state, many actors, need devtools/time-travel, large team? -> Redux Toolkit
Simple global client state, minimal boilerplate, selectors? -> Zustand (or Jotai for atomic/derived state)
Data that lives on the SERVER (lists, details)?  -> TanStack Query / RTK Query (NOT Redux/Context)
URL-representable state (filters, page, tab)?    -> the URL (search params) - shareable, back-button friendly
Form state?                                      -> React Hook Form / local
```
Most "we need Redux" is really "we need a server-state cache".

### 8.1 Redux Toolkit (RTK)
Redux = single store, state changed only by dispatching **actions** handled by pure **reducers**; RTK removes the boilerplate.
```js
// store/ordersSlice.js
import { createSlice, createAsyncThunk, configureStore } from "@reduxjs/toolkit";

export const fetchOrders = createAsyncThunk("orders/fetch", async (_, { rejectWithValue }) => {
  const r = await fetch("/api/orders");
  if (!r.ok) return rejectWithValue(await r.text());
  return r.json();
});
const ordersSlice = createSlice({
  name: "orders",
  initialState: { items: [], status: "idle", error: null },
  reducers: {
    orderAdded(state, action) { state.items.push(action.payload); },          // "mutation" is safe: Immer produces immutable result
    orderRemoved(state, action) { state.items = state.items.filter(o => o.id !== action.payload); },
  },
  extraReducers: b => b
    .addCase(fetchOrders.pending,   s => { s.status = "loading"; })
    .addCase(fetchOrders.fulfilled, (s, a) => { s.status = "succeeded"; s.items = a.payload; })
    .addCase(fetchOrders.rejected,  (s, a) => { s.status = "failed"; s.error = a.payload ?? a.error.message; }),
});
export const { orderAdded, orderRemoved } = ordersSlice.actions;
export const store = configureStore({ reducer: { orders: ordersSlice.reducer } });   // thunk + devtools + immutability checks included
// component:  const items = useSelector(s => s.orders.items);   const dispatch = useDispatch();  <Provider store={store}>
```
* **Immer** records your "mutations" on a draft `Proxy` and produces a new immutable object with structural sharing. Only mutate the draft *or* return a new value, never both. Works only inside `createSlice`/`produce`.
* **Thunk** = function `(dispatch, getState) => ...` for async logic. `createAsyncThunk` auto-dispatches pending/fulfilled/rejected.
* **Selectors** (`useSelector(s => s.orders.items)`) - re-render only if selected value changes by `===`; selecting a fresh array/object each time (`.filter()`) re-renders always -> `createSelector` (reselect) memoises.
* **RTK Query** = data fetching/caching layer in RTK:
```js
export const api = createApi({
  baseQuery: fetchBaseQuery({ baseUrl: "/api", prepareHeaders: h => { /* add token */ return h; } }),
  tagTypes: ["Order"],
  endpoints: b => ({
    getOrders:   b.query({ query: () => "orders", providesTags: ["Order"] }),
    addOrder:    b.mutation({ query: body => ({ url: "orders", method: "POST", body }), invalidatesTags: ["Order"] }),
  }),
});
export const { useGetOrdersQuery, useAddOrderMutation } = api;    // add api.reducer + api.middleware to the store
```
### 8.2 Zustand and Jotai (concise)
```js
import { create } from "zustand";
const useCart = create(set => ({ items: [], add: i => set(s => ({ items: [...s.items, i] })), clear: () => set({ items: [] }) }));
const count = useCart(s => s.items.length);        // selector => re-render only when count changes
```
No provider, tiny, hook-based, selectors built in. **Jotai**: state as small **atoms** (`atom(0)`, derived `atom(get => get(a) * 2)`), fine-grained subscriptions - good for many independent pieces / derived graphs. **Redux** wins on ecosystem/devtools/conventions/large teams; Zustand on simplicity.

### 8.3 TanStack Query (React Query v5)
Server state is **remote, shared, async, can go stale**. The library gives: caching by **query key**, dedup, background refetch, retries, pagination, invalidation, optimistic updates, devtools.
```jsx
const { data, isPending, isError, error, isFetching, refetch } =
  useQuery({ queryKey: ["orders", { page, status }], queryFn: ({ signal }) => api.get("/orders", { params: { page, status }, signal }).then(r => r.data),
             staleTime: 30_000, placeholderData: keepPreviousData });
```
Concepts:
* **Query key** = dependency array; change it => new cache entry/fetch. Include every variable used by `queryFn`.
* **`staleTime`** (default **0**): how long data is "fresh" - fresh data is served from cache with **no** refetch. **`gcTime`** (v5 name; v4 `cacheTime`; default **5 min**): how long an **unused** (no observers) query stays in cache before garbage-collected. They are different clocks: staleTime = "when to refetch", gcTime = "when to forget".
* Stale data is still shown instantly and refetched in background on mount, window refocus, reconnect (`refetchOnWindowFocus`).
* `isPending` = no data yet; `isFetching` = any in-flight request (incl. background); v5 `isLoading = isPending && isFetching`. `isError`, `status`. v5 has only the object signature and no `onSuccess/onError` on `useQuery`.
* **Retries:** default **3** retries with exponential backoff for queries (mutations 0). Don't retry 4xx: `retry: (n, err) => err.status >= 500 && n < 3`.
* **Mutations + invalidation:**
```jsx
const qc = useQueryClient();
const create = useMutation({
  mutationFn: body => api.post("/orders", body).then(r => r.data),
  onSuccess: () => qc.invalidateQueries({ queryKey: ["orders"] }),     // marks stale + refetches active ones
});
create.mutate(values);     // create.isPending / create.error / mutateAsync for await
```
* **Optimistic update with rollback:**
```jsx
useMutation({
  mutationFn: id => api.post(`/orders/${id}/like`),
  onMutate: async id => {
    await qc.cancelQueries({ queryKey: ["orders"] });                    // stop in-flight refetch overwriting us
    const prev = qc.getQueryData(["orders"]);
    qc.setQueryData(["orders"], old => old.map(o => o.id === id ? { ...o, liked: true } : o));
    return { prev };                                                     // context for rollback
  },
  onError: (_e, _id, ctx) => qc.setQueryData(["orders"], ctx.prev),
  onSettled: () => qc.invalidateQueries({ queryKey: ["orders"] }),       // reconcile with server truth
});
```
(v5 also offers a simpler UI-only optimistic approach using `mutation.variables` while pending.)
* Dependent queries: `enabled: !!userId`. Prefetch: `qc.prefetchQuery`. Infinite scroll: `useInfiniteQuery` (`getNextPageParam`). Suspense: `useSuspenseQuery`.

---

## 9. Routing (React Router v6.4+ / data routers; v7 is the same model merged with Remix)

```jsx
import { createBrowserRouter, RouterProvider, Outlet, Navigate, Link, NavLink, useParams, useNavigate, useSearchParams, redirect } from "react-router-dom";

const router = createBrowserRouter([
  { path: "/login", element: <Login /> },
  {
    element: <RequireAuth />,                        // layout route = protected wrapper (renders <Outlet/>)
    children: [
      { path: "/", element: <Layout />,              // nested routes render inside <Outlet/> of Layout
        children: [
          { index: true, element: <Navigate to="orders" replace /> },
          { path: "orders", element: <OrdersPage />,
            loader: ordersLoader },                   // runs BEFORE render, in parallel for nested matches
          { path: "orders/new", element: <NewOrder /> },
          { path: "orders/:id", element: <OrderDetail />, errorElement: <RouteError /> },
          { path: "reports", lazy: () => import("./reports.route") },   // route-level code splitting
        ] },
    ],
  },
  { path: "*", element: <NotFound /> },
]);
root.render(<RouterProvider router={router} />);
```
* **Nested routes** + `<Outlet/>`: URL segments map to component tree; layouts persist across child navigation.
* **Params/query:** `useParams()` gives strings; `useSearchParams()` for `?page=2`.
* **Navigation:** `<Link>` (no reload), `useNavigate()`; `navigate(-1)`, `replace: true` for redirects so Back doesn't loop.
* **Loaders/actions** (data routers): fetch before navigation completes (avoids render-then-fetch waterfalls), `useLoaderData()`; `redirect("/login")` from a loader is the best place to protect data routes. With TanStack Query: loader calls `queryClient.ensureQueryData(...)`.
* **Protected route (component form):**
```jsx
function RequireAuth() {
  const { user, ready } = useAuth();
  const location = useLocation();
  if (!ready) return <FullPageSpinner />;                                        // token refresh on page load still pending
  if (!user) return <Navigate to="/login" replace state={{ from: location }} />;  // remember target
  return <Outlet />;
}
// after login:  navigate(location.state?.from?.pathname ?? "/", { replace: true });
```
Client-side route guarding is **UX only**; the Spring Security backend must enforce authorisation on every endpoint.
* SPA hosting needs the server to return `index.html` for unknown paths (see 15.5).
* Lazy routes: `lazy: () => import(...)` or `React.lazy` + `Suspense`.

---

## 10. Performance playbook

Order of work: **measure -> find the actual bottleneck -> fix the cheapest big thing -> re-measure.**
1. **React DevTools Profiler** (record interaction; flamegraph shows what rendered, "why did this render" with *Record why each component rendered* enabled; grey = not rendered). Check dev build vs production build - dev is 2-10x slower and StrictMode doubles renders.
2. **why-did-you-render** library (dev only) logs avoidable re-renders (same props by value).
3. **Avoid unnecessary renders (cheapest first):**
   * Colocate state (move `useState` down into the small component that needs it).
   * Pass expensive subtree as `children` (2.11).
   * Split context; use selectors.
   * `React.memo` + stable props (`useCallback`/`useMemo`) for hot list rows / heavy components.
   * Don't create components inside components; avoid new object/array/function props only where they meet `memo`.
   * Debounce/`useDeferredValue` for typing-driven heavy work.
4. **Expensive computations:** `useMemo`, move to web worker, or precompute on server (Spring Boot pagination/aggregation).
5. **Long lists:** pagination or **virtualization** (`react-window`, `@tanstack/react-virtual`) render only visible rows (~20 DOM nodes for 10,000 items).
6. **Code splitting:** route-level `lazy`, dynamic `import()` for heavy libs (charts, editors, moment), `Suspense` fallback. Prefer small libs (dayjs, date-fns), import per-function.
7. **Bundle analysis:** `vite-bundle-visualizer` / `rollup-plugin-visualizer`, `webpack-bundle-analyzer`, `source-map-explorer`; check duplicated deps, tree-shaking, `sideEffects: false`.
8. **Images:** modern formats (WebP/AVIF), correct sizes, `srcset`, `loading="lazy"`, `width/height` set (avoid CLS), CDN; `next/image` does it automatically.
9. **Network:** HTTP/2, gzip/brotli at nginx, cache headers for hashed assets (`immutable`), API response shape/pagination, prefetch on hover, TanStack Query caching.
10. **Web Vitals** (field metrics, from `web-vitals` lib / Lighthouse / CrUX): **LCP** (largest content paint, < 2.5 s), **INP** (interaction to next paint, < 200 ms; replaced FID in 2024), **CLS** (layout shift, < 0.1); also TTFB, FCP. Typical React fixes: LCP - SSR/preload hero image, split JS; INP - break up long tasks, `useTransition`, less render work per interaction; CLS - reserve space for images/async content.
11. Keys + stable identity; avoid rendering hidden tabs (or lazy-mount).

---

## 11. Rendering strategies and Next.js

| Strategy | HTML produced | Pros | Cons | Fit |
|---|---|---|---|---|
| **CSR** (Vite SPA) | Empty shell; JS builds UI in browser | Simple hosting (static + nginx), rich app feel | slow first paint, SEO weaker, big bundle | Internal dashboards, behind login |
| **SSR** | Server renders HTML per request; then **hydration** attaches handlers | Fast first content, SEO, personalised | Server cost/latency (TTFB), hydration mismatch risk | Public dynamic pages |
| **SSG** | HTML at build time | Fastest, CDN-cacheable, cheap | Stale until rebuild | Docs, marketing, blog |
| **ISR** | SSG + revalidate in background after N seconds/on demand | Static speed + freshness | Eventual consistency | Catalogs, product pages |

**Hydration:** browser shows server HTML, then React attaches to the existing DOM; the client render must produce the **same** output (else hydration mismatch warning): avoid `Date.now()`, `Math.random()`, `window` checks in render; use `useId`, `useEffect` for browser-only data.

**Next.js App Router (13.4+ / 14 / 15):** file-based (`app/orders/page.tsx`, `layout.tsx`, `loading.tsx`, `error.tsx`, `route.ts`).
* **Server Components (default):** run only on the server (or at build), can be `async` and `await db.query()`/`fetch`, ship **zero JS** for themselves, can use secrets. Cannot use state, effects, event handlers, browser APIs.
* **Client Components:** file starts with `"use client"` - marks the **boundary**; that module and everything it imports become client bundle. They are still SSR'd for first HTML, then hydrated. Server components can render client components and pass **serialisable props** (not functions); client components can render server components only via `children`/props.
* **Streaming:** `loading.tsx`/`<Suspense>` sends shell HTML immediately and streams slower parts as they resolve (selective hydration).
* **Caching/data:** `fetch` caching semantics changed between versions (15 made fetch uncached by default) - check the version's docs; ISR via `revalidate` / `revalidatePath` / tags.
* **Server Actions:** `"use server"` async functions callable from forms/handlers - Next creates an RPC endpoint; validate & authorise inside like any public endpoint.
```tsx
// app/orders/page.tsx  (Server Component)
export default async function Page() {
  const orders = await fetch(`${process.env.API}/orders`, { cache: "no-store" }).then(r => r.json());
  return <OrderList initial={orders} />;          // OrderList has "use client" if it needs interactivity
}
// actions.ts
"use server";
export async function createOrder(formData: FormData) { /* validate, auth, call Spring Boot, revalidatePath("/orders") */ }
```
When to choose Next vs Vite SPA: public SEO pages / need SSR -> Next; authenticated internal app with Spring Boot API -> Vite SPA is simpler (one more server otherwise: Node BFF).

---

## 12. React 19 features

1. **Actions:** async functions used in transitions (`startTransition(async () => {...})`) / form `action` - React tracks pending state, errors, optimistic updates, and **resets uncontrolled form** fields after a successful action.
2. **`<form action={fn}>`:** `fn(formData)` called on submit (no `preventDefault`); works with server actions in frameworks.
3. **`useActionState(action, initialState)`** -> `[state, formAction, isPending]`; `action(prevState, formData)` returns next state.
```jsx
function CreateOrder() {
  const [state, formAction, isPending] = useActionState(async (prev, formData) => {
    const res = await fetch("/api/orders", { method: "POST", headers: {"Content-Type":"application/json"},
                                             body: JSON.stringify({ sku: formData.get("sku") }) });
    return res.ok ? { ok: true } : { ok: false, error: await res.text() };
  }, { ok: null });
  return (<form action={formAction}>
    <input name="sku" required />
    <button disabled={isPending}>Create</button>
    {state.error && <p role="alert">{state.error}</p>}
  </form>);
}
```
4. **`useFormStatus()`** (from `react-dom`): child of a `<form>` reads `{ pending, data, method, action }` - design-system submit buttons.
5. **`useOptimistic(state, reducer?)`** -> `[optimisticState, addOptimistic]`: show expected result immediately while an action runs; reverts to real state when the action finishes.
```jsx
const [optLikes, addLike] = useOptimistic(likes, (cur, delta) => cur + delta);
async function like() { startTransition(async () => { addLike(1); await api.like(id); setLikes(l => l + 1); }); }
```
6. **`use(resource)`:** reads a **Promise** (suspends until resolved; use with Suspense and a promise created *outside* render/cached - a promise created inline in render is new every render = infinite suspend) or a **Context** (can be called conditionally, after early return - unlike `useContext`). Not stored in the hooks list.
7. **Ref changes:** `ref` as prop; **ref callback cleanup** functions; `forwardRef` deprecated path.
8. **Context as provider:** `<ThemeCtx value={x}>` instead of `.Provider`.
9. **Document metadata:** `<title>`, `<meta>`, `<link>` rendered in components are hoisted to `<head>`; also stylesheet/script preload APIs (`preload`, `preinit`), `<link rel="stylesheet" precedence>`.
10. **Server Components** and **Server Actions** stable (for frameworks/bundlers that support them), better hydration error messages, custom elements support, `useDeferredValue(value, initialValue)`, cleanup of `ref`s, `ReactDOM.createRoot` options (`onCaughtError`...).
11. 19.x: `<Activity>` (hide UI but preserve state, deprioritised), `useEffectEvent`, React Compiler 1.0 (separate package).

---

## 13. Testing, TypeScript, accessibility, security

### 13.1 Testing philosophy
Runner: **Jest** or **Vitest** (Vite-native, Jest-compatible API) + **React Testing Library (RTL)** + `@testing-library/user-event` + `jest-dom` matchers. Principle (Kent C. Dodds): *"The more your tests resemble the way your software is used, the more confidence they give you."* Test **behaviour** (what the user sees/does), not implementation (state names, internal methods, hook calls, snapshot of huge trees).

Query priority (most to least preferred): `getByRole` (with `name`) > `getByLabelText` > `getByPlaceholderText` > `getByText` > `getByDisplayValue` > `getByAltText/Title` > `getByTestId` (last resort). Variants: `getBy` (throws if none, sync, must exist), `queryBy` (returns null - for asserting absence), `findBy` (async, waits up to 1 s - for things appearing after async work), `*AllBy`.
```jsx
import { render, screen } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { setupServer } from "msw/node";
import { http, HttpResponse } from "msw";

const server = setupServer(
  http.get("/api/orders", () => HttpResponse.json([{ id: 1, customer: "Ann", total: 100 }])),
  http.post("/api/orders", async ({ request }) => HttpResponse.json({ id: 2, ...(await request.json()) }, { status: 201 })),
);
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers());
afterAll(() => server.close());

test("lists orders and shows error on failure", async () => {
  const user = userEvent.setup();
  renderWithProviders(<OrdersPage />);                          // wraps QueryClientProvider (retry: false!) + Router
  expect(screen.getByText(/loading/i)).toBeInTheDocument();
  expect(await screen.findByText("Ann")).toBeInTheDocument();    // findBy = wait for async UI

  server.use(http.get("/api/orders", () => new HttpResponse(null, { status: 500 })));   // per-test override
  await user.click(screen.getByRole("button", { name: /refresh/i }));
  expect(await screen.findByRole("alert")).toHaveTextContent(/failed/i);
});
```
* **`userEvent` vs `fireEvent`:** userEvent simulates the real sequence (focus, keydown, input, keyup, click), `fireEvent` dispatches one synthetic event. Prefer `userEvent.setup()` and `await`.
* **MSW (Mock Service Worker)** intercepts at network level - the code under test uses real `fetch/axios`; same handlers reusable in browser dev and Storybook. Better than `jest.mock("axios")` (couples to implementation).
* Async: `await screen.findBy...` or `waitFor(() => expect(...))`; never `setTimeout` sleeps; "not wrapped in act(...)" warning means state updated after the test finished asserting - await the UI.
* Testing hooks: `renderHook(() => useDebounce(v, 300))` with fake timers (`vi.useFakeTimers()` / `jest.useFakeTimers()`, `act(() => vi.advanceTimersByTime(300))`).
* Test pyramid: many unit/component (RTL), some integration (page + MSW), few **e2e** (Cypress or Playwright: real browser, whole stack, critical journeys: login, checkout). Playwright: multi-browser, auto-wait, parallel, traces; Cypress: great DX, runs in-browser, single tab historically.
* Snapshot tests: fine for small stable output, bad as a default (noisy, rubber-stamped).

### 13.2 TypeScript with React
```tsx
type ButtonProps = { variant?: "primary" | "ghost"; onClick?: React.MouseEventHandler<HTMLButtonElement>; children: React.ReactNode };
function Button({ variant = "primary", ...rest }: ButtonProps) { return <button className={variant} {...rest} />; }
// extend native props:  type InputProps = React.ComponentPropsWithoutRef<"input"> & { label: string };

const [order, setOrder] = useState<Order | null>(null);                 // generic state, union with null
const inputRef = useRef<HTMLInputElement>(null);                        // DOM ref: null until mounted
const onChange = (e: React.ChangeEvent<HTMLInputElement>) => setName(e.target.value);
const onSubmit = (e: React.FormEvent<HTMLFormElement>) => { e.preventDefault(); };
type Action = { type: "add"; text: string } | { type: "toggle"; id: number };   // discriminated union for reducers

// Generic component
type ListProps<T> = { items: T[]; getKey: (t: T) => string | number; render: (t: T) => React.ReactNode };
function List<T>({ items, getKey, render }: ListProps<T>) { return <ul>{items.map(i => <li key={getKey(i)}>{render(i)}</li>)}</ul>; }
<List items={orders} getKey={o => o.id} render={o => o.customer} />        // T inferred as Order

// Context with non-null guard, custom hook generics
const Ctx = createContext<AuthState | undefined>(undefined);
```
Tips: prefer plain function components + props type over `React.FC` (implicit children removed in 18 types); `ReactNode` for anything renderable, `ReactElement` for a single element, `ElementType` for "component or tag"; use `satisfies`, avoid `any`; infer types from zod (`z.infer`); share API DTO types from OpenAPI (`openapi-typescript`) generated from your Spring Boot springdoc spec.

### 13.3 Accessibility (a11y)
* Semantic HTML first: `<button>` not `<div onClick>`; `<nav>`, `<main>`, headings in order; `<label htmlFor>` for every control.
* Keyboard: everything operable with Tab/Enter/Space/Esc; visible focus; logical tab order; never `tabIndex > 0`.
* **Modals:** `role="dialog"`, `aria-modal`, label via `aria-labelledby`, **focus trap**, return focus to trigger on close, Esc closes (or use Radix/Headless UI/`<dialog>`).
* ARIA only when HTML can't express it; first rule of ARIA: don't use ARIA if native works. `aria-live="polite"` / `role="alert"` for toasts and async errors; `aria-expanded`, `aria-controls` for accordions; `aria-invalid` + `aria-describedby` for errors.
* Alt text, colour contrast >= 4.5:1, don't rely on colour only, respect `prefers-reduced-motion`.
* Route change: move focus/announce new page title (SPA gotcha).
* Tools: `eslint-plugin-jsx-a11y`, axe DevTools / `jest-axe`, Lighthouse, screen reader test (NVDA/VoiceOver). RTL's role queries double as a11y checks.

### 13.4 Security
* **XSS:** React escapes interpolated values (`{userInput}`) - safe by default. Dangers: `dangerouslySetInnerHTML` (sanitize with **DOMPurify**, never raw), `href={userUrl}` with `javascript:` scheme (validate protocol), `eval`/`new Function`, injecting into `<script>`/style, server-side templating of initial state (`JSON.stringify` inside `<script>` must escape `<`), third-party scripts. Add a **Content-Security-Policy** header (from nginx/Spring).
* **Token storage:** `localStorage` -> readable by any XSS => token theft. **HttpOnly, Secure, SameSite cookie** -> JS can't read it but browser sends it automatically => **CSRF** risk (mitigate with `SameSite=Lax/Strict`, CSRF token/double-submit, custom header check, CORS locked down). Recommended pattern for SPA + Spring: short-lived **access token in memory**, long-lived **refresh token in HttpOnly SameSite cookie** scoped to `/api/auth`; restore session on reload via `/auth/refresh`. (Section 14.)
* **CSRF specifics:** Spring Security's CSRF protection is on by default for cookie/session apps; for a stateless Bearer-token API it's usually disabled - but if the refresh endpoint uses a cookie it should be protected (SameSite + require `POST` + custom header/Origin check).
* **CORS** is a *browser* policy, not security for your API (curl ignores it). Configure allowed origins exactly (no `*` with credentials).
* **Dependency risk:** supply-chain attacks (typosquats, compromised maintainers, postinstall scripts): lockfile committed, `npm ci`, `npm audit`/Dependabot/Renovate, pin & review new deps, avoid tiny needless packages, SBOM, `--ignore-scripts` in CI where possible.
* Secrets: anything in `VITE_*` / `NEXT_PUBLIC_*` env is **public** in the bundle - never put API secrets in the frontend.
* Authorization is enforced on the server; hiding buttons/routes is not security. Validate all input server-side (`@Valid`). Avoid open redirects after login (`from` must be an internal path). Use `rel="noopener noreferrer"` for `target=_blank` (modern browsers imply noopener).

---

## 14. Authentication with a Spring Boot JWT backend (full working sketch)

### 14.1 Design
```
                          access token (JWT, 5-15 min)  -> in JS MEMORY, sent as  Authorization: Bearer
 React SPA  <----------   refresh token (opaque/JWT, days) -> HttpOnly; Secure; SameSite=Strict; Path=/api/auth  cookie
    |  1 POST /api/auth/login {email,password}          -> 200 {accessToken, user}  + Set-Cookie refresh
    |  2 GET  /api/orders   Authorization: Bearer <A>   -> 200
    |  3 ... A expires -> 401
    |  4 POST /api/auth/refresh  (cookie sent automatically, withCredentials) -> 200 {accessToken}   (refresh rotation: new cookie)
    |  5 retry the failed request with new A
    |  6 refresh fails (401/403) -> clear auth, redirect /login
    |  7 POST /api/auth/logout -> server revokes refresh token + clears cookie; SPA drops memory token
 Page reload: memory is wiped -> bootstrap calls /auth/refresh once (cookie) before rendering protected routes.
```
Trade-off table:
| Storage | XSS | CSRF | Survives reload | Verdict |
|---|---|---|---|---|
| localStorage | token stolen | immune | yes | simplest, weakest |
| memory (variable) | harder to steal (still XSS can call API) | immune | no (needs refresh) | good for access token |
| HttpOnly cookie | can't read | needs SameSite/CSRF defence | yes | good for refresh token |

Note: XSS defeats any client-side scheme (attacker script can just call your API as the user), so **prevent XSS first**.

### 14.2 Spring Boot side (contract only - your Spring notes have the filter chain)
* `POST /api/auth/login` -> validate, return `{accessToken, user}` and `Set-Cookie: refresh=...; HttpOnly; Secure; SameSite=Strict; Path=/api/auth; Max-Age=...`.
* `POST /api/auth/refresh` -> read cookie, verify/rotate (store hashed refresh token id in DB; detect reuse -> revoke family), return new access token.
* `POST /api/auth/logout` -> revoke + expire cookie.
* CORS: `allowedOrigins("https://app.example.com")`, `allowCredentials(true)`, expose needed headers. In dev use Vite proxy to stay same-origin (15.4).

### 14.3 Frontend code
```js
// api/client.js
import axios from "axios";

let accessToken = null;                       // in memory only
let onAuthFailure = () => {};                 // set by AuthProvider (logout + redirect)
export const setAccessToken = t => { accessToken = t; };
export const setOnAuthFailure = fn => { onAuthFailure = fn; };

export const api = axios.create({ baseURL: "/api", withCredentials: true });   // withCredentials so refresh cookie is sent

api.interceptors.request.use(cfg => {
  if (accessToken) cfg.headers.Authorization = `Bearer ${accessToken}`;
  return cfg;
});

// Single-flight refresh: all concurrent 401s share ONE refresh call, then retry.
let refreshPromise = null;
function refreshAccessToken() {
  refreshPromise ??= axios.post("/api/auth/refresh", null, { withCredentials: true })   // bare axios: bypass our interceptors (no loop)
    .then(r => { setAccessToken(r.data.accessToken); return r.data.accessToken; })
    .finally(() => { refreshPromise = null; });
  return refreshPromise;
}

api.interceptors.response.use(
  res => res,
  async error => {
    const original = error.config;
    const status = error.response?.status;
    const isAuthCall = original?.url?.includes("/auth/");
    if (status === 401 && !original._retry && !isAuthCall) {
      original._retry = true;                              // retry at most once per request
      try {
        const token = await refreshAccessToken();
        original.headers.Authorization = `Bearer ${token}`;
        return api(original);                              // replay
      } catch (e) {
        setAccessToken(null);
        onAuthFailure();                                   // -> navigate("/login")
        return Promise.reject(e);
      }
    }
    return Promise.reject(error);
  });
```
The refresh-queue idea was validated in Node with a fake API: 4 concurrent calls with an expired token -> `[200, 200, 200, 200]` and `refreshCount = 1` (using the same `refreshing ??= refresh().finally(() => refreshing = null)` single-flight pattern). Without it you'd get 4 refresh calls, and with **refresh-token rotation** calls 2-4 would fail (token already used) and log the user out - a classic production bug.

```jsx
// auth/AuthProvider.jsx
const AuthCtx = createContext(null);
export function AuthProvider({ children }) {
  const [user, setUser] = useState(null);
  const [ready, setReady] = useState(false);
  const navigate = useNavigate();

  const logout = useCallback(async () => {
    try { await api.post("/auth/logout"); } catch {}
    setAccessToken(null); setUser(null); queryClient.clear();        // wipe cached private data!
    navigate("/login", { replace: true });
  }, [navigate]);

  useEffect(() => { setOnAuthFailure(() => { setUser(null); navigate("/login", { replace: true }); }); }, [navigate]);

  useEffect(() => {                                                   // bootstrap session on page load
    let ignore = false;
    refreshAccessToken().then(() => api.get("/auth/me")).then(r => { if (!ignore) setUser(r.data); })
      .catch(() => {}).finally(() => { if (!ignore) setReady(true); });
    return () => { ignore = true; };
  }, []);
  // NOTE: StrictMode runs this twice in dev; single-flight refreshPromise makes that harmless (with rotation, unguarded double refresh would break).

  const login = useCallback(async (email, password) => {
    const { data } = await api.post("/auth/login", { email, password });
    setAccessToken(data.accessToken); setUser(data.user);
  }, []);

  const value = useMemo(() => ({ user, ready, login, logout }), [user, ready, login, logout]);
  return <AuthCtx.Provider value={value}>{children}</AuthCtx.Provider>;
}
export const useAuth = () => useContext(AuthCtx);
```
(`refreshAccessToken` must be exported from `client.js`.) Login page: call `login`, catch 401 -> show "Invalid credentials", then `navigate(from, {replace:true})`. `RequireAuth` is in section 9. Role-based UI: `user.roles.includes("ADMIN")` for display only.

Multi-tab: logout in one tab - use `BroadcastChannel`/`storage` event to log out others. Token expiry proactive refresh: schedule with `exp` claim (optional; interceptor is enough).

---

## 15. Complete CRUD feature: Orders (React Query + React Router + zod + Spring Boot)

### 15.1 API contract (Spring Boot)
```
GET    /api/orders?page=0&size=10&status=NEW   -> 200 { content:[Order], totalPages, totalElements, number }
GET    /api/orders/{id}                        -> 200 Order | 404
POST   /api/orders  {customerEmail, quantity, notes}  -> 201 Order | 400 { message, fieldErrors:{ customerEmail:"must be a valid email" } }
PUT    /api/orders/{id}                        -> 200 Order | 404 | 409 (version conflict)
DELETE /api/orders/{id}                        -> 204
```
### 15.2 Data layer
```js
// features/orders/ordersApi.js
import { api } from "../../api/client";
export const listOrders  = (params, signal) => api.get("/orders", { params, signal }).then(r => r.data);
export const getOrder    = (id, signal)     => api.get(`/orders/${id}`, { signal }).then(r => r.data);
export const createOrder = body             => api.post("/orders", body).then(r => r.data);
export const deleteOrder = id               => api.delete(`/orders/${id}`);

// normalise server errors once
export function toFormErrors(error) {
  const d = error?.response?.data;
  return { message: d?.message ?? error.message ?? "Unexpected error", fieldErrors: d?.fieldErrors ?? {} };
}
```
```js
// features/orders/hooks.js
import { useQuery, useMutation, useQueryClient, keepPreviousData } from "@tanstack/react-query";
export const orderKeys = { all: ["orders"], list: p => ["orders", "list", p], detail: id => ["orders", "detail", id] };

export const useOrders = params => useQuery({
  queryKey: orderKeys.list(params),
  queryFn: ({ signal }) => listOrders(params, signal),     // axios supports AbortSignal -> cancels on unmount/key change
  placeholderData: keepPreviousData,                        // no flicker between pages
  staleTime: 15_000,
});
export const useCreateOrder = () => {
  const qc = useQueryClient();
  return useMutation({ mutationFn: createOrder, onSuccess: () => qc.invalidateQueries({ queryKey: orderKeys.all }) });
};
export const useDeleteOrder = () => {
  const qc = useQueryClient();
  return useMutation({ mutationFn: deleteOrder, onSuccess: () => qc.invalidateQueries({ queryKey: orderKeys.all }) });
};
```
### 15.3 Screens
```jsx
// main.jsx
const queryClient = new QueryClient({ defaultOptions: { queries: { retry: (n, e) => (e.response?.status ?? 500) >= 500 && n < 2, refetchOnWindowFocus: false } } });
createRoot(document.getElementById("root")).render(
  <StrictMode><QueryClientProvider client={queryClient}><RouterProvider router={router} /></QueryClientProvider></StrictMode>);
// AuthProvider uses useNavigate => render it inside the router, e.g. as the element of the root layout route.

// OrdersPage.jsx
function OrdersPage() {
  const [sp, setSp] = useSearchParams();
  const page = Number(sp.get("page") ?? 0), status = sp.get("status") ?? "";
  const { data, isPending, isError, error, isFetching, refetch } = useOrders({ page, size: 10, status });
  const del = useDeleteOrder();

  if (isPending) return <Skeleton rows={10} />;
  if (isError) return <ErrorState message={toFormErrors(error).message} onRetry={refetch} />;
  return (
    <section aria-busy={isFetching}>
      <header><h1>Orders</h1><Link to="/orders/new">New order</Link></header>
      <select value={status} onChange={e => setSp({ status: e.target.value, page: 0 })} aria-label="Status filter">
        <option value="">All</option><option>NEW</option><option>SHIPPED</option>
      </select>
      {data.content.length === 0 ? <p>No orders yet.</p> : (
        <table>
          <tbody>{data.content.map(o => (
            <tr key={o.id}>
              <td><Link to={`/orders/${o.id}`}>{o.customerEmail}</Link></td><td>{o.quantity}</td>
              <td><button disabled={del.isPending && del.variables === o.id}
                          onClick={() => window.confirm("Delete?") && del.mutate(o.id)}>Delete</button></td>
            </tr>))}
          </tbody>
        </table>)}
      {del.isError && <p role="alert">{toFormErrors(del.error).message}</p>}
      <Pagination page={page} totalPages={data.totalPages} onChange={p => setSp({ status, page: p })} />
    </section>);
}

// NewOrder.jsx  (RHF + zod + server error mapping)
function NewOrder() {
  const navigate = useNavigate();
  const create = useCreateOrder();
  const { register, handleSubmit, setError, formState: { errors } } = useForm({ resolver: zodResolver(schema), defaultValues: { quantity: 1 } });
  const onSubmit = handleSubmit(values => create.mutate(values, {
    onSuccess: o => navigate(`/orders/${o.id}`),
    onError: err => {
      const { message, fieldErrors } = toFormErrors(err);
      Object.entries(fieldErrors).forEach(([f, m]) => setError(f, { type: "server", message: m }));
      if (!Object.keys(fieldErrors).length) setError("root", { message });
    },
  }));
  return (
    <form onSubmit={onSubmit} noValidate>
      <input aria-label="Customer email" {...register("customerEmail")} />{errors.customerEmail && <p role="alert">{errors.customerEmail.message}</p>}
      <input aria-label="Quantity" type="number" {...register("quantity")} />{errors.quantity && <p role="alert">{errors.quantity.message}</p>}
      {errors.root && <p role="alert">{errors.root.message}</p>}
      <button disabled={create.isPending}>{create.isPending ? "Saving..." : "Create"}</button>
    </form>);
}
```
Coverage checklist (interviewers look for it): loading, empty, error+retry, pending mutation (disable button, prevent double submit), field-level server errors, cache invalidation after mutation, URL-driven pagination/filter, cancellation, auth errors handled centrally by the interceptor, 404 page for missing order (`isError` with `error.response.status === 404`).

### 15.4 Dev server: CORS vs proxy (Vite)
```js
// vite.config.js
export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: { "/api": { target: "http://localhost:8080", changeOrigin: true } },   // browser sees same origin -> no CORS, cookies "just work"
  },
});
```
Frontend calls `/api/...` relative. Production: nginx serves the SPA and reverse-proxies `/api` to Spring Boot => same origin again. If instead the SPA calls `https://api.x.com` cross-origin: Spring `CorsConfigurationSource` with exact origin, `allowCredentials(true)`, and axios `withCredentials`; cookies then need `SameSite=None; Secure`. Same-origin proxy is simpler and safer.

### 15.5 Build + Docker/nginx
```dockerfile
# Dockerfile (multi-stage)
FROM node:22-alpine AS build
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build                      # -> /app/dist  (hashed filenames)

FROM nginx:1.27-alpine
COPY nginx.conf /etc/nginx/conf.d/default.conf
COPY --from=build /app/dist /usr/share/nginx/html
EXPOSE 80
```
```nginx
server {
  listen 80;
  root /usr/share/nginx/html;
  index index.html;

  location /api/ { proxy_pass http://backend:8080; proxy_set_header Host $host; proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for; proxy_set_header X-Forwarded-Proto $scheme; }

  location /assets/ { add_header Cache-Control "public, max-age=31536000, immutable"; }    # content-hashed files

  location / {
    try_files $uri $uri/ /index.html;          # SPA FALLBACK: /orders/42 refresh must return index.html, else 404
    add_header Cache-Control "no-cache";       # index.html must always revalidate so new hashes are picked up
  }
  gzip on; gzip_types text/css application/javascript application/json image/svg+xml;
}
```
Why the fallback: client-side routes don't exist as files; a hard refresh or deep link asks nginx for `/orders/42` -> 404 unless rewritten to `index.html`. Build-time env: `VITE_API_URL` is **baked in at build**; for runtime config use a `config.js` loaded before the bundle or nginx `envsubst`. Same idea for S3+CloudFront: 404/403 -> `/index.html` with 200.

---

## 16. Common bugs and production stories (symptom -> diagnosis -> fix)

| # | Symptom | Diagnosis | Fix |
|---|---|---|---|
| 1 | Page freezes / "Maximum update depth exceeded" | `useEffect(() => setX(...))` with no deps or with dep that the effect itself changes (e.g. `[items]` and `setItems([...items, y])`); or `onClick={setX(1)}` (called during render) | Correct deps, functional update, derive instead of effect; pass `() => setX(1)` |
| 2 | Effect fires endlessly / API hammered | dependency is a new object/array/function each render (`const opts = {}` in body) | Move inside effect, hoist, depend on primitives, `useMemo` |
| 3 | Counter "sticks" at 1 in interval / handler shows old value | Stale closure (2.10) | Functional update; correct deps; latest-ref; `useEffectEvent` |
| 4 | UI doesn't update after `arr.push(x); setArr(arr)` | Same reference -> `Object.is` bail-out; also mutation breaks memo/selectors | `setArr(a => [...a, x])` |
| 5 | Inputs lose focus on every keystroke | Component defined inside component, or `key` changes (random key) each render | Hoist component; stable keys |
| 6 | Wrong text stays in a row after delete/reorder | `key={index}` (2.5) | stable id keys |
| 7 | Old search results replace newer ones | Race: out-of-order responses | AbortController/ignore flag/`queryKey` |
| 8 | Works in prod, "double call" or broken in dev | StrictMode remount exposes missing cleanup / non-idempotent effect | Add cleanup; move one-time init out of effect |
| 9 | "Rendered more hooks than during the previous render" | Hook after conditional return / in `if` (2.12) | Move hooks above returns |
| 10 | Memoised child still re-renders | Unstable props (inline fn/object), `children`, context change | `useCallback/useMemo`, move context read |
| 11 | Whole app slow when typing in one field | State too high; context value with fast-changing data | Colocate state; split context; `useDeferredValue` |
| 12 | Hydration mismatch (Next) | Date/random/localStorage in render | `useEffect`+state, `suppressHydrationWarning` for known text, stable ids |
| 13 | Users randomly logged out under load | Parallel 401s each trigger refresh; rotation invalidates | Single-flight refresh (14.3) |
| 14 | Data of user A visible after user B logs in | Query cache not cleared on logout | `queryClient.clear()`; include user id in keys |
| 15 | `Cannot read properties of undefined` on first render | Data not loaded yet | Handle `isPending`, optional chaining, default values, Suspense |
| 16 | State update on unmounted component warning (old React)/lost updates | async without cleanup | abort/ignore flag |
| 17 | `useEffect` with `async` function warning | Returns a Promise, not cleanup | inner `async function run(){}; run();` |
| 18 | Deep link 404 in production | No SPA fallback | `try_files ... /index.html` |
| 19 | CORS error only in browser | Missing `Access-Control-Allow-Origin`/credentials mismatch, or preflight blocked by security filter order | Proxy in dev; Spring `CorsConfigurationSource` + `http.cors()`, permit `OPTIONS` |
| 20 | Stale UI after mutation | Forgot invalidate / wrong key | `invalidateQueries` with key prefix |
| 21 | `setState` in `useEffect` cause double render flicker | Should be derived or layout effect | derive; `useLayoutEffect` if measuring |
| 22 | Form field reset on parent re-render | `defaultValue` vs controlled confusion / key change | consistent control mode |

Production story A: "Dashboard fires 300 API calls per minute after release." Diagnosis: Network tab shows requests every render; effect dep `filters` object rebuilt inline in parent. Fix: `useMemo` filters / URL params as source and query key with primitives; add TanStack Query dedup. Postmortem: lint rule `exhaustive-deps` set to error.
Production story B: "Search box shows results for a previous word." Race condition; fixed with `AbortController` plus query key.
Production story C: "Bundle 4 MB, LCP 6 s on 3G." Analyzer showed `moment` locales + whole lodash + charting library on login page; fixed with route-level lazy loading, `dayjs`, `lodash-es` per-method imports, brotli; LCP 2.3 s.

---

## 17. Twenty-five "implement it" tasks

Each: requirement, then the key implementation. Talk through edge cases aloud (cleanup, keys, a11y, stale closures) - that is what is graded.

**T1. useDebounce** - see section 6. Edge: cleanup clears timer; delay change resets timer; debounced *callback* variant needs a ref to latest fn.

**T2. useFetch** - section 6 (abort, refetch, `r.ok`). Edge: url null (skip), options in ref, no data flash on url change (keep previous `data`? decide and state it).

**T3. Infinite scroll (IntersectionObserver)**
```jsx
function useInfinite(fetchPage) {                      // fetchPage(cursor) -> { items, next }
  const [items, setItems] = useState([]); const [cursor, setCursor] = useState(0);
  const [loading, setLoading] = useState(false); const [done, setDone] = useState(false);
  useEffect(() => {
    if (done) return; let ignore = false; setLoading(true);
    fetchPage(cursor).then(({ items: more, next }) => {
      if (ignore) return; setItems(p => [...p, ...more]); if (next == null) setDone(true); setLoading(false); });
    return () => { ignore = true; };
  }, [cursor]);                                         // fetchPage assumed stable (or use ref)
  const observer = useRef(null);
  const sentinel = useCallback(node => {                // callback ref: re-attach when node changes
    observer.current?.disconnect();
    if (!node || loading || done) return;
    observer.current = new IntersectionObserver(([e]) => e.isIntersecting && setCursor(c => c + 1), { rootMargin: "200px" });
    observer.current.observe(node);
  }, [loading, done]);
  return { items, loading, done, sentinel };
}
// <ul>{items.map(i => <li key={i.id}>{i.name}</li>)}</ul><div ref={sentinel} />
```
Production choice: `useInfiniteQuery` + same sentinel; combine with virtualization for very long lists.

**T4. Modal via portal** - section 7.2. Add: Esc key listener (effect w/ cleanup), focus trap, restore focus, scroll lock (`document.body.style.overflow`), `aria-labelledby`.
```jsx
useEffect(() => {
  if (!open) return;
  const prev = document.activeElement; const onKey = e => e.key === "Escape" && onClose();
  document.addEventListener("keydown", onKey); document.body.style.overflow = "hidden";
  return () => { document.removeEventListener("keydown", onKey); document.body.style.overflow = ""; prev?.focus(); };
}, [open, onClose]);
```

**T5. Accordion** (single open)
```jsx
function Accordion({ items }) {
  const [open, setOpen] = useState(null);
  const base = useId();
  return items.map((it, i) => {
    const isOpen = open === i;
    return (<div key={it.id}>
      <h3><button id={`${base}-h${i}`} aria-expanded={isOpen} aria-controls={`${base}-p${i}`} onClick={() => setOpen(isOpen ? null : i)}>{it.title}</button></h3>
      <div id={`${base}-p${i}`} role="region" aria-labelledby={`${base}-h${i}`} hidden={!isOpen}>{it.body}</div>
    </div>);
  });
}
```
Multi-open: state `Set`/array of ids (new Set each update - immutability).

**T6. Tabs** - compound components in 7.6; add arrow-key roving `tabIndex`, `role="tablist"`.

**T7. Todo with reducer** - reducer in section 3 plus `useReducer`, filter (`all|active|done` derived not stored), `localStorage` persistence via `useEffect([state])`, `key={t.id}`, `crypto.randomUUID()` for ids **generated in the handler** (not in the reducer: reducers must be pure).

**T8. Autocomplete with race handling**
```jsx
function Autocomplete() {
  const [q, setQ] = useState(""); const dq = useDebounce(q, 250);
  const [opts, setOpts] = useState([]); const [active, setActive] = useState(-1);
  useEffect(() => {
    if (dq.trim().length < 2) { setOpts([]); return; }
    const ctrl = new AbortController();
    fetch(`/api/products?search=${encodeURIComponent(dq)}`, { signal: ctrl.signal })
      .then(r => r.json()).then(setOpts).catch(e => e.name !== "AbortError" && console.error(e));
    return () => ctrl.abort();                        // newest query wins; older aborted
  }, [dq]);
  const onKeyDown = e => {
    if (e.key === "ArrowDown") setActive(a => Math.min(a + 1, opts.length - 1));
    if (e.key === "ArrowUp") setActive(a => Math.max(a - 1, 0));
    if (e.key === "Enter" && active >= 0) { setQ(opts[active].name); setOpts([]); }
    if (e.key === "Escape") setOpts([]);
  };
  return (<div><input role="combobox" aria-expanded={opts.length > 0} aria-controls="opts" aria-activedescendant={active >= 0 ? `o${active}` : undefined}
                      value={q} onChange={e => { setQ(e.target.value); setActive(-1); }} onKeyDown={onKeyDown} />
    <ul id="opts" role="listbox">{opts.map((o, i) => <li key={o.id} id={`o${i}`} role="option" aria-selected={i === active} onMouseDown={() => setQ(o.name)}>{o.name}</li>)}</ul></div>);
}
```
Discuss: debounce reduces requests, abort handles ordering; `onMouseDown` (not click) to beat input blur; cache with TanStack Query (`enabled: dq.length >= 2`).

**T9. Star rating**
```jsx
function StarRating({ value, onChange, max = 5 }) {
  const [hover, setHover] = useState(0);
  return (<div role="radiogroup" aria-label="Rating">
    {Array.from({ length: max }, (_, i) => i + 1).map(n => (
      <button key={n} role="radio" aria-checked={value === n} aria-label={`${n} star${n > 1 ? "s" : ""}`}
              onMouseEnter={() => setHover(n)} onMouseLeave={() => setHover(0)} onClick={() => onChange(n)}>
        {(hover || value) >= n ? "★" : "☆"}</button>))}
  </div>);
}
```
Controlled component (value/onChange from parent).

**T10. Pagination**
```jsx
function usePagination(total, size, initial = 1) {
  const pages = Math.max(1, Math.ceil(total / size)); const [page, setPage] = useState(initial);
  const safe = Math.min(page, pages);                                   // clamp if total shrinks
  return { page: safe, pages, next: () => setPage(p => Math.min(p + 1, pages)), prev: () => setPage(p => Math.max(p - 1, 1)),
           go: n => setPage(Math.min(Math.max(n, 1), pages)), from: (safe - 1) * size, to: Math.min(safe * size, total) };
}
```
Prefer server-side pagination (Spring `Pageable`) and page in URL (`useSearchParams`); windowed page numbers with ellipsis: show first, last, current +- 1.

**T11. Toast system** (context + reducer + portal)
```jsx
const ToastCtx = createContext(null);
function ToastProvider({ children }) {
  const [toasts, setToasts] = useState([]);
  const remove = useCallback(id => setToasts(t => t.filter(x => x.id !== id)), []);
  const push = useCallback((msg, type = "info", ms = 4000) => {
    const id = crypto.randomUUID();
    setToasts(t => [...t, { id, msg, type }]);
    setTimeout(() => remove(id), ms);                                  // (production: track timers, pause on hover, clear on unmount)
  }, [remove]);
  const api = useMemo(() => ({ push }), [push]);                       // stable => consumers don't re-render on each toast
  return (<ToastCtx.Provider value={api}>{children}
    {createPortal(<div aria-live="polite" role="status" className="toasts">
      {toasts.map(t => <div key={t.id} className={`toast ${t.type}`}>{t.msg}<button aria-label="Dismiss" onClick={() => remove(t.id)}>x</button></div>)}
    </div>, document.body)}</ToastCtx.Provider>);
}
export const useToast = () => useContext(ToastCtx);
```
Key insight: keep the *API* in context and the *list* only in the provider so pushing a toast doesn't re-render consumers.

**T12. Theme context** - section 5.3; add `prefers-color-scheme` initial, persist via `useLocalStorage`, apply `document.documentElement.dataset.theme = theme` in an effect (or `useLayoutEffect`/inline script to prevent flash).

**T13. Protected route** - section 9 (`RequireAuth` + `Outlet`, `state.from`, `ready` gate, role variant `<RequireRole roles={["ADMIN"]}/>` rendering 403 page).

**T14. Optimistic like button (React 19 + Query variants)**
```jsx
function LikeButton({ id, liked, count }) {
  const qc = useQueryClient();
  const m = useMutation({
    mutationFn: () => api.post(`/posts/${id}/like`),
    onMutate: async () => { await qc.cancelQueries({ queryKey: ["post", id] }); const prev = qc.getQueryData(["post", id]);
      qc.setQueryData(["post", id], p => ({ ...p, liked: !p.liked, count: p.count + (p.liked ? -1 : 1) })); return { prev }; },
    onError: (_e, _v, ctx) => qc.setQueryData(["post", id], ctx.prev),
    onSettled: () => qc.invalidateQueries({ queryKey: ["post", id] }),
  });
  return <button aria-pressed={liked} onClick={() => m.mutate()}>{liked ? "Unlike" : "Like"} ({count})</button>;
}
```
Edge: rapid double clicks (disable while pending or queue), rollback flicker, idempotent server endpoint (PUT/DELETE `/like` is better than toggle POST).

**T15. Virtualized list (simplified)**
```jsx
function VirtualList({ items, rowHeight = 40, height = 400, overscan = 5 }) {
  const [top, setTop] = useState(0);
  const start = Math.max(0, Math.floor(top / rowHeight) - overscan);
  const end = Math.min(items.length, Math.ceil((top + height) / rowHeight) + overscan);
  return (<div style={{ height, overflow: "auto" }} onScroll={e => setTop(e.currentTarget.scrollTop)}>
    <div style={{ height: items.length * rowHeight, position: "relative" }}>            {/* spacer gives correct scrollbar */}
      {items.slice(start, end).map((it, i) => (
        <div key={it.id} style={{ position: "absolute", top: (start + i) * rowHeight, height: rowHeight, left: 0, right: 0 }}>{it.label}</div>))}
    </div></div>);
}
```
Discuss: only ~(height/rowHeight + 2*overscan) nodes; variable heights need measurement (libs); `will-change`, throttle via `requestAnimationFrame`.

**T16. usePrevious / useToggle / useInterval**
```jsx
function useInterval(cb, delay) {                          // Dan Abramov's pattern: latest callback via ref, timer via delay
  const saved = useRef(cb); useEffect(() => { saved.current = cb; });
  useEffect(() => { if (delay == null) return; const id = setInterval(() => saved.current(), delay); return () => clearInterval(id); }, [delay]);
}
```

**T17. useOnClickOutside / dropdown** - section 6 hook + `Escape` + `aria-haspopup`, portal for overflow.

**T18. Countdown / stopwatch** - `useInterval` with functional updates; drift: compute from `Date.now()` target instead of counting ticks.

**T19. Search with URL state** - `useSearchParams` as the single source; `const q = sp.get("q") ?? ""`; `setSp(prev => { prev.set("q", v); return prev; }, { replace: true })`; debounce writing to URL; TanStack Query keyed by `q`.

**T20. Multi-step form wizard** - `useReducer` with `{step, data}`, per-step zod schemas, back/next preserve data, `key={step}` reset, save draft to `sessionStorage`.

**T21. Debounced save (autosave) with status** - `useDebounce(form, 1000)` -> effect posts if changed (compare JSON against last saved ref); states: idle/saving/saved/error; flush on `beforeunload`/unmount.

**T22. Undo/redo** - reducer state `{past, present, future}`; `set` pushes present to `past`, clears `future`; `undo` pops from past; pure and easily unit-tested.
```js
function undoable(reducer) {
  return (s, a) => {
    if (a.type === "UNDO") return s.past.length ? { past: s.past.slice(0, -1), present: s.past.at(-1), future: [s.present, ...s.future] } : s;
    if (a.type === "REDO") return s.future.length ? { past: [...s.past, s.present], present: s.future[0], future: s.future.slice(1) } : s;
    const present = reducer(s.present, a);
    return present === s.present ? s : { past: [...s.past, s.present], present, future: [] };
  };
}
```

**T23. Data table: sort + filter + memoised derived rows** - `useMemo(() => [...rows].filter(...).sort(...), [rows, filter, sortKey, dir])`; never sort state in place; `aria-sort` on headers; stable sort tie-breaker by id.

**T24. Error boundary + retry** - `react-error-boundary` with `resetKeys` and `QueryErrorResetBoundary` for Query (`useQueryErrorResetBoundary`).

**T25. Online/offline banner + `useSyncExternalStore`** - section 5.5 `useOnline`; render banner, disable mutations, queue via Query's `networkMode`.

---

## 18. Interview questions (E = easy, M = medium, H = hard)

### A. Fundamentals

**Q1 (E). What is React, and what problem does it solve?**
A: A declarative UI library: describe UI as a function of state; React reconciles and updates the DOM. Solves manual DOM sync, component reuse, unidirectional data flow.
*Follow-ups:* library vs framework? (No router/data layer built in; Next/Remix are frameworks.) -> Declarative vs imperative? (say what, not how; jQuery example.)
*Wrong answer:* "React is a framework that uses virtual DOM to be faster than the DOM."

**Q2 (E). What is JSX? Does the browser understand it?**
A: Syntax extension compiled (Babel/SWC/esbuild) to `jsx()`/`createElement` calls returning element objects. Browser doesn't understand it. Expressions in `{}`, `className`, `htmlFor`, camelCase events, must return a single root (or Fragment).
*Follow-ups:* Do you need `import React`? (Not with automatic runtime, 17+.) -> How does `<Foo/>` differ from `<foo/>`? (Capitalised = component reference; lowercase = host tag string.)

**Q3 (E). Props vs state?**
A: Props: inputs from parent, read-only for the child. State: memory owned by the component, changed via setter, triggers re-render. Props flow down; events/callbacks flow up.
*Follow-up:* Can a child modify props? (No; call a parent callback.) -> Can state be derived from props? (Prefer computing during render; copying props to state causes stale copies unless deliberately "initial" - name it `initialX`.)

**Q4 (E). Why are keys needed in lists? What makes a good key?**
A: Identity for reconciliation between renders (2.5). Stable, unique among siblings, from data (DB id).
*Follow-ups:* Why not index? (Show the state-sticking example.) -> When is index OK? (static, never reordered/filtered, no state.) -> What if key = Math.random()? (Remounts every render: state loss, perf, focus loss.)

**Q5 (E). Explain the Virtual DOM.**
A: Lightweight object tree (elements/fibers) describing UI; React diffs new vs previous and commits minimal DOM operations, batching them. Value is declarative programming model and enabling scheduling, not "always faster than DOM".
*Follow-up:* Is re-render the same as DOM update? (No.) -> Why do Svelte/Solid skip it? (compile-time fine-grained updates.)

**Q6 (E). Controlled vs uncontrolled components?** See 7.4. Follow-up: `defaultValue` vs `value`; file input; RHF is uncontrolled-based; when would you choose uncontrolled? (large forms, perf, simple submit-only.)

**Q7 (E). Class vs function components? Lifecycle mapping?**
A: Function + hooks is the standard; classes still work and are needed for error boundaries. `componentDidMount` ~ `useEffect(..., [])`, `componentDidUpdate` ~ effect with deps, `componentWillUnmount` ~ cleanup. But effects are about synchronisation, not lifecycle moments.

**Q8 (E). What are the Rules of Hooks and why do they exist?**
A: Top-level only, only in components/custom hooks. Because hooks are stored per fiber in a linked list matched by call order (2.12) - changing order reads the wrong slot.
*Follow-ups:* What error do you get? ("Rendered more hooks than during the previous render", or silent wrong state.) -> Can you conditionally call `use()`? (Yes, React 19.) -> How to conditionally run an effect? (condition *inside* the effect.)

**Q9 (E). What does `useState` return and is the setter synchronous?**
A: `[state, setter]`. Setter enqueues an update and schedules a render; the current variable doesn't change (snapshot).

**Q10 (E). What is lifting state up / prop drilling? Solutions?**
A: Move state to the common ancestor. Drilling solutions: composition (`children`), context, state library. Don't reach for context first.

### B. Rendering and reconciliation

**Q11 (M). Output prediction - how many renders and what logs?**
```jsx
function App() {
  const [n, setN] = useState(0);
  console.log("render", n);
  return <button onClick={() => { setN(n + 1); setN(n + 1); setN(n + 1); console.log("clicked", n); }}>{n}</button>;
}
```
A (one click, production/React 18): logs `clicked 0`, then `render 1` - **one** render, n becomes **1** (batching + snapshot). In StrictMode dev each render logs twice (the second dimmed). With `setN(x => x + 1)` x3 result is 3 and still one render.
*Follow-ups:* What if the three calls are in `setTimeout`? (React 18: still 1 render; 17: 3.) -> How would `flushSync` change it? (each flush renders synchronously.)

**Q12 (M). What happens on `setState` with the same value?**
A: `Object.is` equal -> React bails out (may call the component once more but doesn't render children/commit/run effects). Explains why `arr.push(); setArr(arr)` does nothing.
*Follow-up:* NaN? (`Object.is(NaN,NaN)` true - no re-render; `0` vs `-0` different.)

**Q13 (M). Parent re-renders. Does the child? How to prevent?**
A: Yes by default even with unchanged props. Prevent: `React.memo` (+ stable props), children-as-props pattern, colocate state, React Compiler.
*Follow-ups:* Does `memo` help if you pass an inline arrow? (No.) -> Does `memo` stop context updates? (No.) -> Does memo make sense on cheap components? (No; comparison overhead.)

**Q14 (M). Output prediction - render/effect order**
```jsx
function Child() { console.log("child render"); useEffect(() => console.log("child effect"), []); return null; }
function Parent() { console.log("parent render"); useEffect(() => console.log("parent effect"), []); return <Child />; }
```
A: `parent render`, `child render`, `child effect`, `parent effect` (render top-down, effects bottom-up). StrictMode dev: renders twice and effects setup/cleanup/setup.
*Follow-up:* With a layout effect in each? (child layout, parent layout - before paint - then passive effects.)

**Q15 (M). Explain reconciliation rules and what triggers remount.**
A: Type change at a position, key change, or parent unmount. Same type+key at same position preserves state even if props differ. Trick: `<Chat key={contactId}/>` resets. (2.4.)
*Follow-up:* Why is a component defined inside another a bug? (new type identity every render.)

**Q16 (M). Render phase vs commit phase; what must be pure?**
A: 2.3. Render may be repeated/discarded; commit is sync and applies DOM + layout/passive effects. Side effects in render (fetch, mutate, `ref.current =`) break under StrictMode/concurrent rendering.
*Follow-up:* When do refs get set? (commit; layout phase.)

**Q17 (H). Explain Fiber. Why was it introduced?**
A: Rewrite to make rendering incremental: fiber = unit of work with parent/child/sibling pointers, priority lanes, alternate tree (double buffering). Enables yielding to the browser, interrupting low-priority renders (transitions), Suspense, error boundaries, streaming SSR. Commit stays synchronous.
*Follow-ups:* What are lanes? (bitmask priorities.) -> Does React 18 always time-slice? (Only for transition/deferred/Suspense work; urgent updates are sync.) -> What happens to interrupted work? (discarded and restarted - hence purity.)

**Q18 (H). What is automatic batching and what changed in 18?**
A: 2.8. All updates in same tick batched (timeouts, promises, native handlers) when using `createRoot`; legacy `ReactDOM.render` keeps old behaviour. `flushSync` opts out.
*Follow-up:* Does batching mean updates are async? (They're deferred to end of the event/microtask; the render is scheduled.) -> Can two different state variables render inconsistent intermediate UI? (No; single commit.)

**Q19 (H). Why does React call components twice in StrictMode? Does it happen in production?**
A: Dev-only. Double render finds impure renders; effect mount/unmount/mount finds missing cleanup; 19 double-invokes ref callbacks. Not in production. Fix the code, don't remove StrictMode.

**Q20 (H). Output prediction - queue arithmetic**
```jsx
const [n, setN] = useState(0);
function h() { setN(5); setN(x => x + 1); setN(42); setN(x => x * 2); }
```
A: **84** (replace 5 -> 6 -> replace 42 -> 84); one render. Without the last call: 42. (Verified with a queue replay in Node.)
*Follow-up:* What if `setN(n + 1)` then `setN(x => x + 1)` with n=0? (1 then 2.)

### C. Effects

**Q21 (M). `useEffect` vs `useLayoutEffect`?** 4.5. Layout: sync after DOM mutation before paint (measure/position, avoid flicker); effect: after paint, non-blocking. SSR warns for layout effects.
*Follow-up:* What's `useInsertionEffect`? (CSS-in-JS style injection before layout.)

**Q22 (M). Dependency array semantics? What if you omit a dependency?**
A: 4.2. Omitting gives stale closure. Lint rule enforces. Fix by functional updates, moving values in, `useRef`/`useEffectEvent`, not by silencing lint.
*Follow-ups:* Function in deps changing every render? (`useCallback` or move into effect.) -> Object deps? (depend on primitives.)

**Q23 (M). Output: what logs (dev, StrictMode) for mount?**
```jsx
useEffect(() => { console.log("setup"); return () => console.log("cleanup"); }, []);
```
A: `setup`, `cleanup`, `setup` [expected behaviour]; production just `setup`.
*Follow-up:* How do you make "fetch once" robust? (abort in cleanup; or Query.)

**Q24 (M). Race conditions in data fetching - how do you prevent them?** 4.6: AbortController/ignore flag in cleanup, or Query keys. *Follow-up:* Difference between aborting and ignoring? (abort cancels network; ignore only drops result.)

**Q25 (M). Why is "fetch in useEffect" not ideal for production?**
A: waterfalls, no cache/dedup/retry/refetch/pagination, race handling boilerplate, StrictMode double fetch, SSR doesn't run effects. Use Query / loaders / RSC.

**Q26 (M). Interval counter stuck at 1 - why and fix?** 2.10 Example A. Fixes: functional update (best), deps, ref. *Wrong answer:* "wrap in useCallback".

**Q27 (H). Cleanup order with multiple effects and components?** 2.13: setup children-first in declaration order; on update all cleanups (of changed effects) before any setups; unmount runs cleanups. Cleanup closes over the *previous* render's values.

**Q28 (M). You Might Not Need an Effect - give three cases.** 4.7 (derived state, reset with key, event-handler logic).
*Follow-up:* Why is `useEffect(() => setFiltered(items.filter(...)), [items])` bad? (extra render, stale frame, more state; compute in render/`useMemo`.)

**Q29 (M). Output prediction - infinite loop?**
```jsx
const [list, setList] = useState([]);
useEffect(() => { setList([...list, 1]); }, [list]);
```
A: Yes, infinite: each effect run sets a new array => `list` changes => effect re-runs. Fix: run once with `[]` + functional update, or don't use an effect. An effect with no deps calling `setCount(c=>c+1)` loops the same way (React eventually logs "Maximum update depth exceeded").

**Q30 (H). How does `useEffect` timing relate to paint? Is it always after paint?**
A: Normally after paint (scheduled after commit). Exceptions: if the update was caused by a discrete event like a click, React flushes passive effects synchronously at the end of commit (so state derived from effects is ready before the next interaction); layout effects are always before paint. Never rely on exact scheduling; use layout effect for visual measurements.

### D. Hooks and performance

**Q31 (M). `useMemo` vs `useCallback` vs `React.memo`?** 5.2. *Follow-ups:* When do they hurt? -> Is `useMemo` a semantic guarantee? (No, a hint.) -> What is React Compiler doing? (auto-memoisation.) *Wrong answer:* "useCallback makes function faster / prevents creation" (it still creates the inline closure; it preserves identity).

**Q32 (M). `useRef` uses? Difference to state?** 5.1. Mutable, no re-render, persists across renders, DOM access, prev value. *Follow-up:* Can you read `ref.current` in render? (Avoid; only for lazy init.)

**Q33 (M). Context re-render problem and mitigations.** 5.3. *Follow-ups:* Does `memo` help consumer? (No.) -> Redux vs Context for high-frequency data? (selector subscription vs whole-value.) -> Difference between context and props for perf? (context skips intermediate memoised parents; still re-renders all consumers.)

**Q34 (M). `useReducer` vs `useState`?** Complex transitions, related fields, stable dispatch, testable reducer. Not "Redux in React" necessarily.

**Q35 (M). `useTransition` vs `useDeferredValue` vs debounce?**
A: Transition: you own the setter, mark update non-urgent, `isPending`. Deferred: you receive a value, get a lagging copy. Debounce: time-based, reduces *calls* (network); transitions reduce blocking of *rendering*. They can be combined.

**Q36 (M). What is Suspense? How does `React.lazy` work?** 5.4. Component throws a promise (conceptually) -> nearest boundary shows fallback -> re-render when resolved. lazy needs default export; wrap in Suspense; split per route.

**Q37 (H). What is `useSyncExternalStore` for? Why not `useEffect`+`useState`?**
A: Avoid **tearing** (different components reading different snapshots in a concurrent render) and race between subscribe and first render; supports SSR snapshot. Snapshot must be referentially stable.

**Q38 (M). `useId` purpose?** SSR/CSR consistent unique ids for label/aria; not for keys.

**Q39 (H). How would you diagnose and fix a slow React screen?** 10: Profile in production build, find hot component/why rendered, colocate state, memo + stable props, virtualise, split, defer, reduce work per render, check network/waterfalls; verify with Web Vitals. *Follow-up:* What did you measure? (commit duration, render count, INP.)

**Q40 (H). `useCallback` with an unstable dependency - is it useful?** Only if deps stable; otherwise recreated each render and memoisation is pointless. Use ref/latest pattern or reducer/dispatch for stability.

### E. Architecture, data, routing

**Q41 (M). State management choice for a large app?** 8 decision guide. *Follow-ups:* Why not put API data in Redux? (You'd rebuild caching/invalidation; use RTK Query/Query.) -> How does Immer let you "mutate"? (draft proxy.) -> Selectors and memoisation? (`createSelector`.)

**Q42 (M). Redux Toolkit flow: dispatch -> ?** Component dispatches action -> middleware (thunk) -> reducer (Immer) -> new state -> subscribers/`useSelector` compare selected slice -> re-render only changed. *Follow-up:* thunk vs saga? (thunk simple async; saga generator-based orchestration, rarely needed now.)

**Q43 (M). `staleTime` vs `gcTime` in TanStack Query?** 8.3: fresh window (no refetch) vs cache retention after unobserved. Default 0 and 5 min. *Follow-ups:* `invalidateQueries` vs `refetch`? (invalidate marks stale & refetches active observers, respects key prefixes; refetch is direct.) -> `isPending` vs `isFetching`? -> How do you do optimistic updates? (onMutate/rollback/onSettled.)

**Q44 (M). Explain nested routes, loaders and protected routes in React Router.** 9. *Follow-up:* How do you avoid flashing the login page on reload? (`ready` flag while session refresh runs.) -> Is route guarding secure? (No, UX only.)

**Q45 (H). Design token handling in a SPA with Spring Boot JWT.** 14. Memory access token + HttpOnly refresh cookie, interceptor single-flight refresh, bootstrap `/refresh`, logout revocation, clear caches. *Follow-ups:* What if refresh token stolen? (rotation + reuse detection, short TTL, bind to device.) -> Why not localStorage? (XSS.) -> Why cookie needs CSRF care? (auto-sent.) -> How do 401 vs 403 differ in the interceptor? (401 -> refresh; 403 -> forbidden UI, no refresh.)

**Q46 (H). Multiple requests get 401 at the same time. What happens with naive interceptors and how do you fix?** 14.3: N refresh calls, rotation invalidation => forced logout. Fix: shared promise / queue, `_retry` flag, exclude auth endpoints from refresh logic.

**Q47 (M). CORS error in React dev - causes and fixes?** Browser enforces cross-origin policy; preflight for non-simple requests (Authorization header/JSON); server must answer `OPTIONS`. Fix with dev proxy; in prod same-origin via nginx or exact-origin CORS. Not solved by `no-cors` mode or disabling browser security.

**Q48 (M). SSR vs SSG vs ISR vs CSR? When Next.js?** 11. *Follow-up:* What is hydration and a mismatch? -> Server vs Client Components? (`"use client"` boundary; server comps ship no JS, can't use hooks/state; props must be serialisable.) -> Server Actions security? (public endpoints: validate + authorise.)

**Q49 (H). What is hydration and why can it be slow? What are streaming/selective hydration?** Attaching handlers to server HTML by re-running components client-side; cost scales with JS. Streaming sends HTML chunks per Suspense boundary; selective hydration hydrates boundaries as they arrive/are interacted with. RSC reduces the JS needing hydration.

**Q50 (M). Testing React components - what and how?** 13.1. Behaviour not implementation, role queries, userEvent, `findBy`, MSW, avoid testing internals/snapshots-only. *Follow-ups:* `getBy` vs `queryBy` vs `findBy`? -> Test a custom hook? (`renderHook`.) -> Unit vs e2e balance?

**Q51 (M). How do you test a debounce with timers?** fake timers + `act` + advance; assert single call with last value; restore real timers.

### F. React 19 / advanced

**Q52 (M). What's new in React 19?** 12: Actions, `useActionState`, `useOptimistic`, `useFormStatus`, `use`, ref as prop + ref cleanup, `<Context>` as provider, metadata tags, stable Server Components/Actions, error reporting hooks, `forwardRef` deprecation path. *Follow-up:* `use` vs `useContext`? (conditional call allowed; also reads promises.) -> Where should the promise passed to `use` come from? (cached/created outside render, e.g. loader/Server Component.)

**Q53 (H). How does `useOptimistic` differ from a Query optimistic update?** Local, transient state layered over "real" state for the duration of an action, auto-reverts. Query modifies cache and needs manual rollback but shares across components/pages.

**Q54 (H). Error boundaries - what do they not catch, and how do you handle those?** Event handlers, async, SSR, self. Use try/catch, global `window.onerror`/`unhandledrejection`, Query `onError`, `throw` inside `setState` callback to route to boundary.

**Q55 (M). Portals - why do events still bubble to React parent?** React synthetic events follow the React tree, not the DOM tree. Context also flows through portals.

**Q56 (M). forwardRef and ref as prop?** 7.3. *Follow-up:* `useImperativeHandle` when? (expose narrow API like `focus()`, `scrollTo()`; prefer declarative props.)

**Q57 (M). Compound components vs render props vs HOC vs hooks?** 7.6.

**Q58 (H). Design a reusable Table component API.** Generic `Table<T>`, columns config w/ `accessor`, `render`, `sortable`; controlled sort/page via props (works with server-side), a11y (`aria-sort`), virtualization slot, memoised rows, stable keys, headless hook (`useTable`) + presentational layer (TanStack Table). Discuss extensibility vs YAGNI.

**Q59 (H). How would you migrate a large class-component/Redux app?** Incremental: hooks for new code, wrap classes (no rewrite big-bang), RTK migration slice by slice (`legacy redux` interoperable), replace fetch thunks with RTK Query/Query, codemods, tests first, upgrade to createRoot with StrictMode enabled gradually.

**Q60 (H). Explain React Compiler and when manual memoisation is still needed.** Build-time auto-memoisation of components/hooks respecting Rules of React; removes most `memo/useMemo/useCallback`. Still: code violating rules is skipped, effect-dependency semantics can rely on stable identity, third-party libs expecting stable refs, escape hatches.

**Q61 (M). Accessibility - how do you make a modal accessible?** T4/13.3.

**Q62 (M). XSS in React?** 13.4: escaped by default; `dangerouslySetInnerHTML` + DOMPurify; `javascript:` URLs; CSP; token storage trade-offs.

**Q63 (H). Output prediction - key/state**
```jsx
{step === 1 ? <Input label="Name"/> : <Input label="Email"/>}
```
where `Input` holds its own `useState("")` and step toggles 1 -> 2. A: Same type, same position, no key => **state preserved**: typed name still appears under the Email label. Fix with `key`.

**Q64 (H). Output prediction - stale closure**
```jsx
const [c, setC] = useState(0);
useEffect(() => { const id = setInterval(() => console.log(c), 1000); return () => clearInterval(id); }, []);
// user clicks to increase c to 3
```
A: keeps logging `0` (closure from first render). Adding `[c]` re-creates interval on each change; latest-ref avoids re-creation.

**Q65 (H). Output prediction - object identity**
```jsx
const [user, setUser] = useState({ name: "A" });
function rename() { user.name = "B"; setUser(user); }
```
A: no re-render (same reference; bail-out). If something else later re-renders, UI suddenly shows "B" - mutation bug; also breaks `memo`/effects/selectors. Fix: `setUser(u => ({...u, name: "B"}))`.

**Q66 (M). Difference between `useEffect(fn)` with no deps and code in the render body?** Effect runs after commit (side effects allowed, cleanup, after paint) - render body runs during render and must be pure.

**Q67 (M). How do you handle forms with server-side validation errors from Spring Boot?** Map `fieldErrors` from 400 body to RHF `setError`, show root error, disable submit while pending, keep client zod for UX only. 15.3.

### Common wrong answers (quick list)
* "Virtual DOM is faster than the real DOM" (it is overhead; benefit is declarative model + minimal updates).
* "`setState` is asynchronous" (it schedules; the value you hold is a snapshot; there is no promise).
* "`useEffect` is componentDidMount" (synchronisation model, runs on deps changes, double in StrictMode dev).
* "`useCallback` prevents re-renders" (only with `memo`/deps consumers).
* "Use index as key, it works" (state mismatch).
* "Store the JWT in localStorage, it's fine" (XSS).
* "Context replaces Redux" (no selectors, re-render of all consumers, no devtools/middleware).
* "Re-render means DOM updated".
* "Class components are deprecated" (not deprecated; still needed for error boundaries, still supported).
* "Server Components are SSR" (different: RSC never hydrate; SSR renders client components to HTML too).

---

## 19. One-page cheat sheet

**Mental model:** UI = f(state). Render (pure, may repeat) -> Commit (DOM, layout effects) -> Paint -> Passive effects.
**Re-render when:** state changes (`!Object.is`), parent renders, context value changes, external store changes. **Not when:** ref/variable mutation, same-value set.
**State:** snapshot per render; functional update for "based on previous"; batching everywhere in 18; never mutate; derive don't store; reset with `key`.
**Hooks list:** per-fiber linked list by call order => top level only.
**Effect:** `useEffect(setup, deps)` after paint; cleanup before next setup/unmount; deps = all reactive values; StrictMode dev: setup-cleanup-setup; layout effect = pre-paint measure; insertion = CSS-in-JS.
**Fetching:** abort/ignore flag; avoid waterfalls; use TanStack Query (`queryKey`, `staleTime` 0 default, `gcTime` 5 min, retry 3, `invalidateQueries`, optimistic = onMutate/rollback/onSettled; v5: `isPending`).
**Perf:** measure (Profiler) -> colocate state -> children pattern -> memo+stable props -> split context -> virtualise -> lazy/Suspense -> analyzer -> Web Vitals (LCP 2.5 s, INP 200 ms, CLS 0.1). React Compiler auto-memoises.
**Keys:** stable unique ids; type/key change = remount; components defined in components = remount every render.
**Context:** low-frequency values; memoise value; split state/dispatch; selectors for hot data.
**Concurrent:** `useTransition` (own setter), `useDeferredValue` (own value), Suspense, lazy, streaming SSR. Interrupted renders are discarded.
**React 19:** Actions, `useActionState`, `useOptimistic`, `useFormStatus`, `use()`, ref as prop + ref cleanup, `<Ctx value>`, metadata hoisting, RSC/server actions stable; 19.2 `useEffectEvent`, `<Activity>`.
**Routing (v6.4+):** `createBrowserRouter`, nested + `<Outlet/>`, loaders, `Navigate` in `RequireAuth`, lazy routes, `useSearchParams` for filters; SPA fallback `try_files ... /index.html`.
**Auth:** access token in memory + refresh in HttpOnly SameSite cookie; interceptor with single-flight refresh + `_retry`; bootstrap via `/refresh`; clear query cache on logout; server enforces authz.
**Forms:** RHF + zod; server `@Valid` errors -> `setError`; disable submit while pending.
**Testing:** RTL by role, userEvent, `findBy` for async, MSW for network, Vitest/Jest; few Playwright/Cypress e2e.
**Security:** don't `dangerouslySetInnerHTML` unsanitised (DOMPurify), validate URLs, CSP, no secrets in `VITE_*`, audit deps, CORS != security.
**Next.js:** Server Components default (no hooks, zero client JS), `"use client"` boundary, streaming with Suspense, Server Actions = public endpoints, SSG/ISR/SSR/CSR trade-offs.
**Common bugs:** effect loops (unstable deps), stale closure, mutating state, index keys, missing cleanup/race, hooks in conditions, parallel refresh, missing SPA fallback, uncleared cache on logout.
**Say in interviews:** "It depends - measure first", "render must be pure", "server state is not client state", "the backend is the security boundary".
