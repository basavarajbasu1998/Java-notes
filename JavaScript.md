# JavaScript (ES2023) — Interview Deep Dive for a Java Full-Stack Developer

> Every code block ending with an `// ---- output ----` comment block was executed with `node v22.12` and the output pasted verbatim. Blocks marked `// browser` need a DOM and were not executed. Timer-based outputs use generous margins so they are stable.
> Sections: 1 Mental model · 2 Engine basics · 3 Objects & types · 4 Async (event loop, promises) · 5 Implement-from-scratch · 6 Modules & errors & memory · 7 Browser (DOM, fetch, CORS, storage, security, perf) · 8 TypeScript & tooling · 9 Node.js · 10 Production stories · 11 Interview Q&A (56) · 12 Cheat sheet

---

## 1. 60-second mental model

**JavaScript = one thread, one call stack, run-to-completion, plus queues.** The engine (V8) executes synchronous code on the call stack. Anything slow (timers, network, disk, DOM events) is handed to the host (browser/Node) which later drops a *callback* into a queue. The **event loop** moves a callback onto the stack only when the stack is empty. **Microtasks** (promise reactions, `queueMicrotask`) drain completely before the next **macrotask** (timer, I/O, UI event).

**Analogy (restaurant with ONE chef):** the chef (call stack) cooks one dish at a time and never leaves a dish half-done. Waiters (Web APIs / libuv) handle slow things: oven timers, deliveries. When something is ready the waiter puts a ticket on a rail. There are two rails: a **VIP rail** (microtasks) — the chef clears every VIP ticket before touching the normal rail — and the **normal rail** (macrotasks). A chef who spends 10 s chopping one onion (a heavy sync loop) freezes the whole restaurant: no tickets, no service, no screen repaint.

```
 your code ──► CALL STACK ◄──── event loop: "stack empty? take next job"
                  │                     ▲            ▲
        hands off │                     │ 1st        │ 2nd (one per turn)
                  ▼                     │            │
       Web APIs / libuv (timers,   MICROTASK      MACROTASK (TASK) QUEUE
       fetch, DOM events, fs)      QUEUE          setTimeout, I/O, UI events
       run OUTSIDE the JS thread   .then, await,  MessageChannel, setImmediate
                                   queueMicrotask
```

**Java contrast (you will be asked):**

| | Java (Servlet/Spring MVC) | JavaScript / Node.js |
|---|---|---|
| Concurrency | Many threads, shared memory, locks | One thread for JS, async I/O, no data races on JS objects |
| Blocking I/O | Thread blocks, pool scales it | Blocking = freezes everyone; use async |
| CPU-bound | Fine (more threads/cores) | Bad on main thread → worker_threads / child process |
| Types | Static, compile time | Dynamic (add TypeScript) |
| Inheritance | Class-based | Prototype-based (`class` is syntax sugar) |
| Equivalent of `CompletableFuture` | `thenApply/thenCompose` | `Promise.then` / `async-await` |
| Closures | Lambdas capture *effectively final* values | Closures capture *variables* (mutable, live) |

---

## 2. Engine basics

### 2.1 Execution context & call stack

Each function call creates an **execution context** with: a *variable environment* (params, `var`, function declarations), a *lexical environment* (`let`/`const`/class, block scopes), an `[[OuterEnv]]` reference (the scope chain) and a `this` binding. Two phases: **creation** (hoist declarations, bind `this`) then **execution** (run line by line).

```
Trace: function a(){ b() }  function b(){ c() }  function c(){ throw new Error() }  a()

 push global ─► push a ─► push b ─► push c ─► throws
 CALL STACK (grows up)          stack trace printed = the stack at the throw:
 |   c()   |                     Error
 |   b()   |                        at c
 |   a()   |                        at b
 | global  |                        at a
 +---------+                        at <anonymous>   (global)
 Infinite recursion -> "RangeError: Maximum call stack size exceeded" (stack overflow, same idea as Java StackOverflowError).
```

Tail-call optimisation is in the spec but only Safari implements it — never rely on it; convert deep recursion to loops or trampolines.

### 2.2 Hoisting and the Temporal Dead Zone (TDZ)

"Hoisting" = declarations are registered when the context is *created*, before execution. What differs is the **initial value**:

| Declaration | Hoisted? | Initial value before its line | Scope |
|---|---|---|---|
| `var` | yes | `undefined` | function |
| `let` / `const` / `class` | yes (binding exists) | **uninitialised → TDZ** (`ReferenceError`) | block |
| function declaration | yes, fully | the function itself | function (block in strict) |
| function expression / arrow (`var f = ...`) | only the `var f` | `undefined` → `TypeError: f is not a function` | as var |

```js
console.log(typeof a, a);            // var: hoisted + initialised to undefined
var a = 1;

try { console.log(b); } catch (e) { console.log(e.name + ': ' + e.message); }   // let: TDZ
let b = 2;

try { console.log(typeof c); } catch (e) { console.log('typeof in TDZ ->', e.name); } // typeof is NOT safe in TDZ
const c = 3;

console.log(f1());                   // function declaration: fully hoisted
function f1() { return 'f1 works'; }

try { f2(); } catch (e) { console.log(e.name + ': ' + e.message); } // var f2 is undefined at this point
var f2 = function () { return 'f2'; };

try { f3(); } catch (e) { console.log(e.name + ': ' + e.message); }
const f3 = () => 'f3';

// function declaration wins over var during hoisting; the assignment runs later
console.log(typeof g);
var g = 5;
function g() {}
console.log(typeof g);

// TDZ is temporal (time), not positional
const t = () => x;      // refers to x lexically before its declaration - fine, called later
let x = 10;
console.log(t());
// ---- output (node v22.12, verified) ----
// undefined undefined
// ReferenceError: Cannot access 'b' before initialization
// typeof in TDZ -> ReferenceError
// f1 works
// TypeError: f2 is not a function
// ReferenceError: Cannot access 'f3' before initialization
// function
// number
// 10
```
Points interviewers probe: (1) `let` *is* hoisted — proof: the TDZ error message says "before initialization", not "not defined" (an inner `let x` shadows an outer `x` for the whole block). (2) `typeof` is NOT safe in TDZ, but is safe for undeclared names. (3) TDZ is about *time*, not position: `t()` above works because it runs after `x` initialised.

### 2.3 Scope chain (lexical scoping)

Identifier lookup walks `current scope → outer → … → global`. The chain is fixed where the function is **written** (lexical/static), not where it is called (unlike `this`).

```js
const g = 'global';
function outer() {
  const o = 'outer';
  function inner() {
    const i = 'inner';
    console.log(i, o, g);          // walks the scope chain: inner -> outer -> global
    try { console.log(nope); } catch (e) { console.log(e.message); }
  }
  inner();
}
outer();

// lexical (static) scoping: where the function is WRITTEN, not where it is called
const name = 'global-name';
function show() { return name; }
function caller() { const name = 'caller-name'; return show(); }
console.log(caller());

{ var v = 'var leaks'; let l = 'let stays'; }   // block scope
console.log(v, typeof l);

let z = 1;                                       // shadowing
{ let z = 2; console.log(z); }
console.log(z);
try { eval('let q = 1; { var q = 2; }'); } catch (e) { console.log(e.name + ': ' + e.message); }

function leak() { leaked = 'oops'; }             // implicit global in sloppy mode
leak();
console.log(globalThis.leaked);
(function () { 'use strict'; try { leaked2 = 1; } catch (e) { console.log(e.name + ': ' + e.message); } })();
// ---- output (node v22.12, verified) ----
// inner outer global
// nope is not defined
// global-name
// var leaks undefined
// 2
// 1
// SyntaxError: Identifier 'q' has already been declared
// oops
// ReferenceError: leaked2 is not defined
```
### 2.4 Closures in depth

A **closure = function + the environment (variables) it was created in**. The function keeps a reference to the *variables* (not copies), so it sees later mutations, and those variables live as long as the function is reachable.

```js
// 1. private state
function makeCounter() {
  let n = 0;
  return { inc: () => ++n, dec: () => --n, get value() { return n; } };
}
const c1 = makeCounter(), c2 = makeCounter();
c1.inc(); c1.inc(); c2.inc();
console.log(c1.value, c2.value, c1.n);   // separate closures, n is unreachable from outside

// 2. classic loop bug
const fnsVar = [];
for (var i = 0; i < 3; i++) fnsVar.push(() => i);
console.log(fnsVar.map(f => f()));       // one shared i

const fnsLet = [];
for (let j = 0; j < 3; j++) fnsLet.push(() => j);
console.log(fnsLet.map(f => f()));       // fresh binding per iteration

const fnsIife = [];                      // pre-ES6 fix: IIFE
for (var k = 0; k < 3; k++) (function (k) { fnsIife.push(() => k); })(k);
console.log(fnsIife.map(f => f()));

// 3. module pattern (IIFE)
const Bank = (function () {
  let balance = 0;                        // private
  function validate(x) { if (x <= 0) throw new RangeError('amount must be > 0'); }
  return {
    deposit(x) { validate(x); balance += x; return balance; },
    get balance() { return balance; },
  };
})();
console.log(Bank.deposit(50), Bank.balance, Bank.validate);

// 4. once() - closures as building blocks
const once = fn => { let done = false, r; return (...a) => done ? r : (done = true, r = fn(...a)); };
const init = once(() => { console.log('init runs'); return 42; });
console.log(init(), init());
// ---- output (node v22.12, verified) ----
// 2 1 undefined
// [ 3, 3, 3 ]
// [ 0, 1, 2 ]
// [ 0, 1, 2 ]
// 50 50 undefined
// init runs
// 42 42
```
**The `var`-vs-`let` loop, traced.** `for (var i…)` creates ONE binding shared by all callbacks; when timers fire, the loop is over and `i === 3`. `for (let i…)` creates a **fresh binding per iteration** (the spec copies the value into a new environment each turn), so each closure sees its own value.

```js
for (var m = 0; m < 3; m++) setTimeout(() => process.stdout.write('var:' + m + ' '), 0);
for (let n = 0; n < 3; n++) setTimeout(() => process.stdout.write('let:' + n + ' '), 0);
setTimeout(() => console.log(), 5);
// ---- output (node v22.12, verified) ----
// var:3 var:3 var:3 let:0 let:1 let:2
```
```
var i:  env{ i:3 } ◄── cb0, cb1, cb2       let i: env0{i:0} ◄─ cb0   env1{i:1} ◄─ cb1   env2{i:2} ◄─ cb2
```

**Memory implications.** A closure retains its whole *scope object* (engines optimise by keeping only variables actually referenced, but a single `eval`/nested closure that references a big variable keeps it alive). Leaks happen when a long-lived closure (event handler, timer, cache, module-level array) references a large object you thought was garbage. See §6.5 for a measured demo.

**Where closures are used in real code:** private state/module pattern, factories (`makeValidator(rules)`), memoize/debounce/once, currying, React hooks (`useState` state lives in a closure over the fiber), event handlers capturing props (stale-closure bugs), iterators/generators.

**Java comparison:** Java lambdas capture *effectively final* locals (a copy); JS closures capture the *variable* itself, so counters and shared mutable state work — and stale-closure bugs exist.

### 2.5 `this` — the five rules

`this` is decided **at call time** by *how* the function is called (except arrow functions, which take `this` lexically from where they are defined). Priority, highest first:

1. **`new`** — `this` = the fresh object (`new Foo()`). Beats `bind`.
2. **Explicit** — `f.call(obj)`, `f.apply(obj)`, `f.bind(obj)()`. (`bind` result cannot be re-bound.)
3. **Implicit** — `obj.method()` → `this = obj` (whatever is left of the dot *at the call site*).
4. **Default** — plain call `f()` → `undefined` in strict mode/modules/classes, `globalThis` in sloppy mode.
5. **Arrow** — no own `this`; uses the enclosing one. `call/apply/bind` cannot change it; it cannot be a constructor.

```js
'use strict';
const user = {
  name: 'Asha',
  regular() { return this && this.name; },
  arrow: () => typeof this,                // module-level this in CJS = module.exports ({})
  delayedArrow() { setTimeout(() => console.log('timeout arrow:', this.name), 0); },
};
console.log(user.regular());               // implicit binding
const detached = user.regular;
console.log(detached());                   // lost this (strict => undefined)
console.log(user.arrow());
user.delayedArrow();

function greet(greeting, punct) { return `${greeting}, ${this.name}${punct}`; }
console.log(greet.call({ name: 'Ravi' }, 'Hi', '!'));
console.log(greet.apply({ name: 'Meera' }, ['Hello', '?']));
const bound = greet.bind({ name: 'Kiran' }, 'Namaste');
console.log(bound('.'));
console.log(bound.call({ name: 'ignored' }, '~'));     // bind wins over call
console.log(new (function F() { this.x = 1; })().x);   // new binding

function P() { this.v = 'from new'; }                  // new beats bind
const B = P.bind({ v: 'from bind' });
console.log(new B().v);

class Btn {                                            // extracted methods
  constructor() { this.label = 'OK'; this.handleArrowField = () => this.label; }
  handle() { return this?.label; }
}
const btn = new Btn();
const { handle, handleArrowField } = btn;
console.log(handle(), handleArrowField());

const arr = () => typeof this;                         // arrows ignore call/bind
console.log(arr.call({}), arr.bind({})());
// ---- output (node v22.12, verified) ----
// Asha
// undefined
// object
// Hi, Ravi!
// Hello, Meera?
// Namaste, Kiran.
// Namaste, Kiran~
// 1
// from new
// undefined OK
// object object
// timeout arrow: Asha
```
The important interview trap is **lost `this`**: `const { handle } = btn`, `setTimeout(obj.method, 0)`, `arr.forEach(this.method)`, `onClick={this.handle}`. Fixes: arrow class field, `.bind(this)`, wrapper arrow `() => this.handle()`.

**Implementing `call` / `apply` / `bind`** (they are asked constantly):

```js
Function.prototype.myCall = function (ctx, ...args) {
  ctx = ctx == null ? globalThis : Object(ctx);
  const key = Symbol('fn');
  ctx[key] = this;
  const res = ctx[key](...args);
  delete ctx[key];
  return res;
};
Function.prototype.myApply = function (ctx, args = []) { return this.myCall(ctx, ...args); };
Function.prototype.myBind = function (ctx, ...preset) {
  const fn = this;
  function bound(...rest) {
    return fn.apply(this instanceof bound ? this : ctx, [...preset, ...rest]); // new => this is instance
  }
  if (fn.prototype) bound.prototype = Object.create(fn.prototype);
  return bound;
};
function greet(greeting, punct) { return `${greeting}, ${this.name}${punct}`; }
console.log(greet.myCall({ name: 'A' }, 'Yo', '!'), greet.myApply({ name: 'B' }, ['Sup', '.']));
const mb = greet.myBind({ name: 'C' }, 'Hey');
console.log(mb('...'));
function Pt(x, y) { this.x = x; this.y = y; }
const BP = Pt.myBind(null, 1);
const p = new BP(2);
console.log(p.x, p.y, p instanceof Pt);
// ---- output (node v22.12, verified) ----
// Yo, A! Sup, B.
// Hey, C...
// 1 2 true
```
Key details: `call` temporarily attaches the function to the context under a **Symbol** key (no collisions); `bind` must support `new` (`this instanceof bound`) and partial application; `apply` takes an array-like.

---

## 3. Objects, prototypes, and types

### 3.1 Prototype chain, `new`, `class`

Every object has an internal `[[Prototype]]` link (`Object.getPrototypeOf(o)`, legacy `o.__proto__`). Property lookup walks the chain until `null`. Functions also have a `.prototype` property — the object that becomes the `[[Prototype]]` of instances made with `new`. **`.prototype` (on functions) ≠ `[[Prototype]]` (on every object).**

```
 a  ──[[Prototype]]──►  Animal.prototype ──► Object.prototype ──► null
 {name}                 { speak, constructor:Animal }   {toString, hasOwnProperty…}
 Animal (function) ──[[Prototype]]──► Function.prototype ──► Object.prototype
 Dog.prototype = Object.create(Animal.prototype)   →  dog ─► Dog.prototype ─► Animal.prototype ─► Object.prototype
```

**`new F(args)` — the 4 steps:**
1. Create an empty object `obj` whose `[[Prototype]]` is `F.prototype`.
2. Call `F` with `this = obj`.
3. If `F` returns an **object**, that object is the result (a returned primitive is ignored).
4. Otherwise the result is `obj`.

```js
function Animal(name) { this.name = name; }
Animal.prototype.speak = function () { return this.name + ' makes a sound'; };
const a = new Animal('Generic');
console.log(Object.getPrototypeOf(a) === Animal.prototype, a.__proto__ === Animal.prototype);
console.log(Animal.prototype.constructor === Animal, a.hasOwnProperty('speak'), 'speak' in a);
console.log(Object.getPrototypeOf(Animal.prototype) === Object.prototype, Object.getPrototypeOf(Object.prototype));
console.log(Object.getPrototypeOf(Animal) === Function.prototype);

// what `new` does - hand-written
function myNew(Ctor, ...args) {
  const obj = Object.create(Ctor.prototype);      // 1 create + link [[Prototype]]
  const res = Ctor.apply(obj, args);              // 2 run with this = obj
  return (res !== null && (typeof res === 'object' || typeof res === 'function')) ? res : obj; // 3 override rule
}
const b = myNew(Animal, 'Bob');
console.log(b.speak(), b instanceof Animal);
function Weird() { this.a = 1; return { z: 'returned object wins' }; }
function Prim() { this.a = 1; return 5; }
console.log(new Weird(), new Prim());

// classical inheritance with functions
function Dog(name) { Animal.call(this, name); }
Dog.prototype = Object.create(Animal.prototype);
Dog.prototype.constructor = Dog;
Dog.prototype.speak = function () { return Animal.prototype.speak.call(this) + ' (woof)'; };
console.log(new Dog('Rex').speak());

// class syntax = sugar over the same machinery
class Shape {
  #id = Math.floor(1.9);                // private field
  static count = 0;
  static #secret = 'static private';
  constructor(name) { this.name = name; Shape.count++; }
  get id() { return this.#id; }
  set label(v) { this._label = v.trim(); }
  area() { return 0; }
  #priv() { return 'private method'; }
  callPriv() { return this.#priv(); }
  static peek() { return Shape.#secret; }
  static isShape(o) { return #id in o; }   // ergonomic brand check (ES2022)
  toString() { return `Shape(${this.name})`; }
}
class Circle extends Shape {
  constructor(r) { super('circle'); this.r = r; }
  area() { return +(Math.PI * this.r ** 2).toFixed(2); }
}
const c = new Circle(2);
c.label = '  hi ';
console.log(c.area(), c.id, c._label, c.callPriv(), Shape.peek(), Shape.count, `${c}`);
console.log(Shape.isShape(c), Shape.isShape({}), typeof Shape, Object.keys(c));
try { new Function('o', 'return o.#id')({}); } catch (e) { console.log(e.name); }   // private names must be declared in a class body
try { Circle(); } catch (e) { console.log(e.message); }        // classes cannot be called without new
try { new (class A extends Object { constructor() { this.x = 1; super(); } })(); } catch (e) { console.log(e.name); }
console.log(Object.getOwnPropertyNames(Shape.prototype));      // methods are non-enumerable
const proto = { hi() { return 'hi ' + this.n; } };
const o = Object.create(proto, { n: { value: 'x', enumerable: true } });
console.log(o.hi(), Object.getPrototypeOf(o) === proto, Object.create(null).toString);
// ---- output (node v22.12, verified) ----
// true true
// true false true
// true null
// true
// Bob makes a sound true
// { z: 'returned object wins' } Prim { a: 1 }
// Rex makes a sound (woof)
// 12.57 1 hi private method static private 1 Shape(circle)
// true false function [ 'name', 'r', '_label' ]
// SyntaxError
// Class constructor Circle cannot be invoked without 'new'
// ReferenceError
// [ 'constructor', 'id', 'label', 'area', 'callPriv', 'toString' ]
// hi x true undefined
```
Reading the output: classes are functions (`typeof Shape === 'function'`) and cannot be called without `new`; class methods live on the prototype and are **non-enumerable** (so `Object.keys` shows only own fields); `#id` is truly private (a syntax error outside the class; `#id in o` is the brand check); in a derived class you must call `super()` before touching `this`; `static` members live on the constructor; private fields are per-instance and *not* inherited through the prototype chain (invisible to `Object.keys`, `JSON.stringify`; note `this.#x` on a `Proxy` receiver throws).

**Class fields gotcha:** `handle = () => {...}` creates a new function **per instance** (memory) but fixes `this`; a prototype method is shared but can lose `this`.

### 3.2 Property descriptors, `freeze` vs `seal`, getters/setters

Each property has attributes: `value`/`get`/`set`, `writable`, `enumerable`, `configurable`. Literals create all-true; `defineProperty` defaults to all-false.

```js
const o = {};
Object.defineProperty(o, 'id', { value: 7, writable: false, enumerable: false, configurable: false });
o.id = 9; console.log(o.id, Object.keys(o), JSON.stringify(o));
console.log(Object.getOwnPropertyDescriptor(o, 'id'));
console.log(Object.getOwnPropertyDescriptor({ x: 1 }, 'x'));
(function () { 'use strict'; try { o.id = 1; } catch (e) { console.log(e.message); } })();

const acc = { _t: 20, get f() { return this._t * 9 / 5 + 32; }, set f(v) { this._t = (v - 32) * 5 / 9; } };
acc.f = 212; console.log(acc._t, acc.f);

const cfg = { db: { host: 'x' }, port: 1 };
Object.freeze(cfg);
cfg.port = 2; cfg.extra = 1; delete cfg.port; cfg.db.host = 'mutated';   // silent failures in sloppy mode
console.log(cfg, Object.isFrozen(cfg), Object.isFrozen(cfg.db));

const s = Object.seal({ a: 1 });
s.a = 2; s.b = 3; delete s.a;
console.log(s, Object.isSealed(s));

const ne = Object.preventExtensions({ a: 1 });
ne.b = 1; delete ne.a; console.log(ne);

function deepFreeze(x) { Object.values(x).forEach(v => typeof v === 'object' && v !== null && deepFreeze(v)); return Object.freeze(x); }
const d = deepFreeze({ a: { b: { c: 1 } } }); d.a.b.c = 99; console.log(d.a.b.c);
const arr = [1]; arr.push(2); console.log(arr);                      // const != immutable
try { eval('const k = 1; k = 2;'); } catch (e) { console.log(e.name + ': ' + e.message); }
// ---- output (node v22.12, verified) ----
// 7 [] {}
// { value: 7, writable: false, enumerable: false, configurable: false }
// { value: 1, writable: true, enumerable: true, configurable: true }
// Cannot assign to read only property 'id' of object '#<Object>'
// 100 212
// { db: { host: 'mutated' }, port: 1 } true false
// { a: 2 } true
// {}
// 1
// [ 1, 2 ]
// TypeError: Assignment to constant variable.
```
| | add props | delete props | change values | reconfigure |
|---|---|---|---|---|
| `preventExtensions` | no | yes | yes | yes |
| `seal` | no | no | **yes** | no |
| `freeze` | no | no | no (**shallow!**) | no |

`const` prevents *rebinding* the variable; `Object.freeze` prevents *mutating* the object (one level). Failures are silent in sloppy mode and throw `TypeError` in strict mode/modules/classes. For immutable updates in state (React/Redux) use spread, `structuredClone`, or `toSorted/with/toSpliced`.

### 3.3 Symbols, iterators, generators

`Symbol()` creates a unique, non-string key (hidden from `for-in`, `Object.keys`, `JSON.stringify`). *Well-known symbols* customise language behaviour: `Symbol.iterator`, `Symbol.asyncIterator`, `Symbol.toPrimitive`, `Symbol.hasInstance`, `Symbol.toStringTag`.

**Iterator protocol:** an object with `next()` returning `{value, done}`. **Iterable:** has `[Symbol.iterator]()` returning an iterator. `for-of`, spread, destructuring, `Array.from`, `Map/Set` constructors all consume iterables. A **generator** function (`function*`) returns an object that is both iterator and iterable; `yield` pauses, and `next(v)` sends a value back *into* the paused `yield`. It is lazy, so infinite sequences are fine.

```js
const s1 = Symbol('id'), s2 = Symbol('id');
console.log(s1 === s2, s1.toString(), s1.description, Symbol.for('app') === Symbol.for('app'));
const obj = { [s1]: 1, a: 2 };
console.log(Object.keys(obj), JSON.stringify(obj), Object.getOwnPropertySymbols(obj));

class Range {                                   // custom iterable
  constructor(a, b) { this.a = a; this.b = b; }
  [Symbol.iterator]() {
    let cur = this.a; const end = this.b;
    return { next: () => cur <= end ? { value: cur++, done: false } : { value: undefined, done: true } };
  }
}
console.log([...new Range(1, 5)], Math.max(...new Range(1, 5)));
const [first, , third] = new Range(10, 20); console.log(first, third);

function* fib() { let [a, b] = [0, 1]; for (;;) { yield a; [a, b] = [b, a + b]; } }
const take = (n, it) => { const r = []; for (const v of it) { if (r.length >= n) break; r.push(v); } return r; };
console.log(take(10, fib()));

function* conv() { const x = yield 'q1'; console.log('got', x); const y = yield 'q2'; return x + y; }
const g = conv();
console.log(g.next('ignored'), g.next(10), g.next(5), g.next());

function* inner() { yield 1; yield 2; return 'r'; }
function* outer() { const r = yield* inner(); yield r; }
console.log([...outer()]);

function* withCleanup() { try { yield 1; yield 2; } finally { console.log('cleanup'); } }
for (const v of withCleanup()) { console.log(v); break; }      // break calls return() -> finally runs

async function* ticker() { for (let i = 0; i < 3; i++) { await null; yield i; } }
(async () => { const r = []; for await (const t of ticker()) r.push(t); console.log('async gen', r); })();
console.log(typeof [][Symbol.iterator], typeof {}[Symbol.iterator]);
// ---- output (node v22.12, verified) ----
// false Symbol(id) id true
// [ 'a' ] {"a":2} [ Symbol(id) ]
// [ 1, 2, 3, 4, 5 ] 5
// 10 12
// [
//   0, 1,  1,  2,  3,
//   5, 8, 13, 21, 34
// ]
// got 10
// { value: 'q1', done: false } { value: 'q2', done: false } { value: 15, done: true } { value: undefined, done: true }
// [ 1, 2, 'r' ]
// 1
// cleanup
// function undefined
// async gen [ 0, 1, 2 ]
```
Note the order `got 10` appears **before** the line with the `next()` results: the arguments are evaluated first. `break` out of `for-of` calls the iterator's `return()`, which runs `finally` blocks (resource cleanup, like try-with-resources). Async generators + `for await` model streams/pagination.

### 3.4 Destructuring, spread, rest, optional chaining, nullish

```js
const { a, b: { c = 5, ...restB } = {}, ...rest } = { a: 1, b: { d: 2, e: 3 }, x: 9, y: 8 };
console.log(a, c, restB, rest);
const [p, q = 'dflt', ...others] = [1, undefined, 3, 4];
console.log(p, q, others);
let m = 1, n = 2; [m, n] = [n, m]; console.log(m, n);
function f({ host = 'localhost', port = 80 } = {}, ...more) { return `${host}:${port}/${more.length}`; }
console.log(f(), f({ port: 8080 }, 1, 2));
console.log({ ...{ a: 1, b: 2 }, ...{ b: 3 } }, [...'héy'], Math.max(...[1, 5, 3]));
const key = 'dyn'; const o = { [key + '1']: 1 }; console.log(o);
const { z = 'd1' } = { z: undefined }; const { w = 'd2' } = { w: null }; console.log(z, w);   // default only for undefined
const orig = { n: { v: 1 } }; const cp = { ...orig }; cp.n.v = 2; console.log(orig.n.v);        // spread is SHALLOW
console.log(structuredClone({ d: new Date(0), s: new Set([1]), m: new Map([[1, 2]]) }));
try { structuredClone({ f() {} }); } catch (e) { console.log(e.name); }
console.log(0 || 'x', 0 ?? 'x', '' ?? 'x', null ?? 'x', undefined?.foo, ({}).a?.b.c.d);
let l = null; l ??= 5; let t = 0; t ||= 7; let u = 1; u &&= 9; console.log(l, t, u);
// ---- output (node v22.12, verified) ----
// 1 5 { d: 2, e: 3 } { x: 9, y: 8 }
// 1 dflt [ 3, 4 ]
// 2 1
// localhost:80/0 localhost:8080/2
// { a: 1, b: 3 } [ 'h', 'é', 'y' ] 5
// { dyn1: 1 }
// d1 null
// 2
// { d: 1970-01-01T00:00:00.000Z, s: Set(1) { 1 }, m: Map(1) { 1 => 2 } }
// DataCloneError
// x 0  x undefined undefined
// 5 7 9
```
Rules: defaults apply only for `undefined` (not `null`); spread copies **own enumerable** props and is **shallow**; `??` falls through only on `null/undefined` (unlike `||` which also rejects `0`, `''`, `false`); `structuredClone` handles Date/Map/Set/cycles but throws on functions and drops prototypes/class identity.

### 3.5 Map, Set, WeakMap, WeakSet

| | Object | Map |
|---|---|---|
| Key types | string/symbol | **any** (objects, NaN) |
| Order | integer keys first (ascending), then insertion | insertion |
| Size | `Object.keys(o).length` | `.size` |
| Perf for frequent add/delete | ok | better |
| Prototype pollution risk | yes (`__proto__`, `toString`) | no |

```js
const m = new Map([[1, 'a']]);
const k = { id: 1 };
m.set(k, 'obj').set(NaN, 'nan');
console.log(m.get(k), m.get({ id: 1 }), m.get(NaN), m.size, [...m.keys()]);
const o = {}; o[k] = 1; o[{}] = 2; console.log(Object.keys(o));   // objects stringify to "[object Object]"
const s = new Set([1, 2, 2, 3, NaN, NaN, '1']); console.log(s, s.size);
console.log([...new Set([3, 1, 3, 2, 1])]);
const A = new Set([1, 2, 3]), B = new Set([2, 3, 4]);
console.log([...A].filter(x => B.has(x)), [...A].filter(x => !B.has(x)), [...new Set([...A, ...B])]);
const mm = new Map(); mm.set('b', 1).set('a', 2); console.log([...mm.keys()]);
console.log(Object.keys({ 2: 'x', 1: 'y', b: 1, a: 2 }));   // integer-like keys come first, ascending!
const arr = [3, 1, 2];
console.log(arr.toSorted(), arr, arr.at(-1), arr.findLast(x => x < 3), arr.with(0, 9), Object.groupBy(arr, x => x % 2 ? 'odd' : 'even'));
const wm = new WeakMap(); let el = {}; wm.set(el, 'meta'); console.log(wm.get(el), wm.has({}), typeof wm.size);
try { wm.set('str', 1); } catch (e) { console.log(e.name); }
const priv = new WeakMap();
class Acct { constructor(b) { priv.set(this, { b }); } get bal() { return priv.get(this).b; } }
console.log(new Acct(5).bal);
// ---- output (node v22.12, verified) ----
// obj undefined nan 3 [ 1, { id: 1 }, NaN ]
// [ '[object Object]' ]
// Set(5) { 1, 2, 3, NaN, '1' } 5
// [ 3, 1, 2 ]
// [ 2, 3 ] [ 1 ] [ 1, 2, 3, 4 ]
// [ 'b', 'a' ]
// [ '1', '2', 'b', 'a' ]
// [ 1, 2, 3 ] [ 3, 1, 2 ] 2 2 [ 9, 1, 2 ] [Object: null prototype] { odd: [ 3, 1 ], even: [ 2 ] }
// meta false undefined
// TypeError
// 5
```
`WeakMap/WeakSet`: keys must be objects, **held weakly** (don't prevent GC), not iterable, no `size`. Use: private data per object, caches keyed by DOM nodes/objects (auto-cleanup), tracking "seen" objects. Java analogue: `WeakHashMap`.

### 3.6 Type coercion & equality

Types: 7 primitives (`string number bigint boolean undefined null symbol`) + objects. Primitives are immutable and compared by value; objects by reference.

**`==` (Abstract Equality) algorithm, simplified:**
1. Same type → behaves like `===` (except `NaN != NaN`).
2. `null == undefined` → true; `null`/`undefined` equal **nothing else** (so `null == 0` is false).
3. number vs string → string converted to number.
4. boolean vs anything → boolean converted to **number** first (`true→1`) then re-compare.
5. object vs primitive → object `ToPrimitive` (calls `valueOf` then `toString`; hint “default”→number, Date→string), then compare.
6. bigint vs number → compared mathematically.

Worked traces: `[] == false` → `false→0` → `[] == 0` → `ToPrimitive([])=""` → `"" == 0` → `0 == 0` → **true**. `[] == ![]` → `![]` is `false` (arrays are truthy) → same as above → true. `null >= 0` is true because relational operators convert with ToNumber (`null→0`) while `==` has the special rule.

`+` operator: if either operand (after ToPrimitive) is a string → concatenation; else numeric addition. `-`, `*`, `/` always numeric.

```js
const rows = [
  ['[] + {}', [] + {}], ['[] + []', [] + []], ['({}) + []', ({}) + []], ['1 + "2"', 1 + '2'], ['"3" - 1', '3' - 1],
  ['"3" * "4"', '3' * '4'], ['true + 1', true + 1], ['null + 1', null + 1], ['undefined + 1', undefined + 1],
  ['+""', +''], ['+" 12 "', +' 12 '], ['+"12px"', +'12px'], ['+[]', +[]], ['+{}', +{}], ['+[5]', +[5]], ['+[1,2]', +[1, 2]],
  ['[] == false', [] == false], ['[] == ![]', [] == ![]], ['null == undefined', null == undefined], ['null == 0', null == 0],
  ['null >= 0', null >= 0], ['undefined == 0', undefined == 0], ['NaN == NaN', NaN == NaN], ['"" == 0', '' == 0], ['"0" == false', '0' == false],
  ['"1" == [1]', '1' == [1]], ['[1,2] == "1,2"', [1, 2] == '1,2'], ['typeof null', typeof null], ['typeof NaN', typeof NaN],
  ['typeof undeclared', typeof undeclaredVar], ['typeof function', typeof function () {}], ['typeof class', typeof class {}],
  ['typeof []', typeof []], ['Object.is(NaN,NaN)', Object.is(NaN, NaN)], ['Object.is(0,-0)', Object.is(0, -0)], ['0 === -0', 0 === -0],
  ['[10,9,1].sort()', [10, 9, 1].sort()], ['["1","2","3"].map(parseInt)', ['1', '2', '3'].map(parseInt)],
  ['Number.isNaN("x"), isNaN("x")', [Number.isNaN('x'), isNaN('x')]], ['[] instanceof Object', [] instanceof Object],
  ['`${[1,2]}`', `${[1, 2]}`], ['String({})', String({})],
  ['!!"false", !!0n, !![]', [!!'false', !!0n, !![]]], ['1 < 2 < 3, 3 > 2 > 1', [1 < 2 < 3, 3 > 2 > 1]], ['"b"+"a"+ +"a"+"a"', 'b' + 'a' + +'a' + 'a'],
];
for (const [k, v] of rows) console.log(k.padEnd(32), '->', typeof v === 'string' ? JSON.stringify(v) : v);
const o = { valueOf() { return 42; }, toString() { return 'str'; } };
console.log(o + 1, `${o}`, o * 2, o == 42, String(o));
const d = { [Symbol.toPrimitive](hint) { return hint === 'number' ? 1 : hint === 'string' ? 'S' : 'default'; } };
console.log(+d, `${d}`, d + '');
console.log(new Date(0) + 1 === new Date(0).toString() + '1');   // Date's default hint is string
// ---- output (node v22.12, verified) ----
// [] + {}                          -> "[object Object]"
// [] + []                          -> ""
// ({}) + []                        -> "[object Object]"
// 1 + "2"                          -> "12"
// "3" - 1                          -> 2
// "3" * "4"                        -> 12
// true + 1                         -> 2
// null + 1                         -> 1
// undefined + 1                    -> NaN
// +""                              -> 0
// +" 12 "                          -> 12
// +"12px"                          -> NaN
// +[]                              -> 0
// +{}                              -> NaN
// +[5]                             -> 5
// +[1,2]                           -> NaN
// [] == false                      -> true
// [] == ![]                        -> true
// null == undefined                -> true
// null == 0                        -> false
// null >= 0                        -> true
// undefined == 0                   -> false
// NaN == NaN                       -> false
// "" == 0                          -> true
// "0" == false                     -> true
// "1" == [1]                       -> true
// [1,2] == "1,2"                   -> true
// typeof null                      -> "object"
// typeof NaN                       -> "number"
// typeof undeclared                -> "undefined"
// typeof function                  -> "function"
// typeof class                     -> "function"
// typeof []                        -> "object"
// Object.is(NaN,NaN)               -> true
// Object.is(0,-0)                  -> false
// 0 === -0                         -> true
// [10,9,1].sort()                  -> [ 1, 10, 9 ]
// ["1","2","3"].map(parseInt)      -> [ 1, NaN, NaN ]
// Number.isNaN("x"), isNaN("x")    -> [ false, true ]
// [] instanceof Object             -> true
// `${[1,2]}`                       -> "1,2"
// String({})                       -> "[object Object]"
// !!"false", !!0n, !![]            -> [ true, false, true ]
// 1 < 2 < 3, 3 > 2 > 1             -> [ true, false ]
// "b"+"a"+ +"a"+"a"                -> "baNaNa"
// 43 str 84 true str
// 1 S default
// true
```
**Truthiness — falsy values (8):** `false, 0, -0, 0n, "", null, undefined, NaN`. Everything else is truthy, including `"0"`, `"false"`, `[]`, `{}`.
**Which check to use:** `Number.isNaN(x)` (not global `isNaN`, which coerces), `Object.is(a,b)` for NaN/−0 distinctions, `Array.isArray`, `x == null` (deliberate "null or undefined" test — the one accepted use of `==`), `typeof x === 'undefined'` for possibly undeclared globals.

### 3.7 Numbers, precision, BigInt

All numbers are IEEE-754 doubles (53-bit mantissa): integers exact up to `2**53-1`; `0.1` has no finite binary representation.

```js
console.log(0.1 + 0.2, 0.1 + 0.2 === 0.3, Math.abs(0.1 + 0.2 - 0.3) < Number.EPSILON);
console.log((0.1 + 0.2).toFixed(2), +(0.1 + 0.2).toFixed(2), 1.005 * 1000, Math.round(1.005 * 100) / 100, +(1.005).toFixed(2));
console.log(Number.MAX_SAFE_INTEGER, 2 ** 53 === 2 ** 53 + 1, Number.MAX_VALUE, 1 / 0, -1 / 0, 0 / 0);
console.log(9007199254740993n + 2n, 2n ** 64n, typeof 1n, 5n / 2n, 1n == 1, 1n === 1);
try { 1n + 1; } catch (e) { console.log(e.name + ': ' + e.message); }
console.log(BigInt(Number.MAX_SAFE_INTEGER) + 2n, Number(2n ** 60n), parseInt('08'), parseFloat('3.14abc'), (255).toString(16), (0.5).toString(2));
const cents = Math.round(19.99 * 100); console.log(cents, (cents * 3) / 100);   // money: integer minor units
console.log(Number('1_000'), 1_000, 0b101, 0o17, 0xff, 5 % -3, -5 % 3, ((-5 % 3) + 3) % 3);
console.log(new Intl.NumberFormat('en-IN', { style: 'currency', currency: 'INR' }).format(1234567.891));
// ---- output (node v22.12, verified) ----
// 0.30000000000000004 false true
// 0.30 0.3 1004.9999999999999 1 1
// 9007199254740991 true 1.7976931348623157e+308 Infinity -Infinity NaN
// 9007199254740995n 18446744073709551616n bigint 2n true false
// TypeError: Cannot mix BigInt and other types, use explicit conversions
// 9007199254740993n 1152921504606847000 8 3.14 ff 0.1
// 1999 59.97
// NaN 1000 5 15 255 2 -2 1
// ₹12,34,567.89
```
Rules: never compare floats with `===` (use an epsilon); do money in **integer minor units** (paise/cents) or a decimal library; `BigInt` cannot mix with `Number` and is not supported by `JSON.stringify` (send as string) and `Math.*`; use it for IDs > 2^53 (Twitter snowflakes, DB `BIGINT` — Java `long` IDs arriving in JSON silently lose precision unless serialised as strings!). `toFixed` returns a *string* and rounds using the binary value (`1.005.toFixed(2) === "1.00"`).

---

## 4. Asynchronous JavaScript

### 4.1 The event loop in depth

**Actors:** (1) **call stack**; (2) **heap**; (3) **host APIs** — browser: timers, `fetch`, DOM events, rendering; Node: libuv (timers, epoll/IOCP network, a 4-thread pool for fs/crypto/dns); (4) **macrotask (task) queues** — timers, I/O callbacks, UI events, `MessageChannel`, `setImmediate` (Node); (5) **microtask queue** — promise reactions, `queueMicrotask`, `MutationObserver` (+ Node's `process.nextTick` queue, which is separate and even higher priority).

**Browser loop (HTML spec), one turn:**
```
 ┌─► 1. pick ONE task from a task queue (script, click, timer, message, network callback)
 │   2. run it to completion on the call stack
 │   3. MICROTASK CHECKPOINT: run ALL microtasks (including ones added meanwhile) until queue is empty
 │   4. if a rendering opportunity (~every 16.7ms @60Hz, and the tab is visible):
 │        a. resize / scroll events   b. requestAnimationFrame callbacks (then microtasks after each)
 │        c. style recalculation      d. layout (reflow)   e. paint / composite   f. IntersectionObserver
 │   5. if idle: requestIdleCallback
 └───┘ repeat
```
Consequences: (a) **a long task blocks paint and input** (INP/jank); (b) microtasks run before rendering, so an endless microtask chain freezes the page even worse than a timer loop; (c) `setTimeout(fn, 0)` is *not* immediate — it is clamped to ≥1 ms (≥4 ms when nested > 5 deep, and browsers throttle background tabs to ≥1 s); (d) **`rAF` runs right before paint** — the correct place for visual updates; a 0 ms timer may run before or after the next paint.

**Node loop (libuv) phases:**
```
   ┌──────────────────────────┐
┌─►│ timers        (setTimeout / setInterval whose time has come)
│  │ pending callbacks         (some deferred system errors)
│  │ idle, prepare             (internal)
│  │ poll          (wait for + run I/O callbacks; blocks here if nothing else scheduled)
│  │ check         (setImmediate callbacks)
│  │ close callbacks           (socket.on('close'))
└──┴──────────────────────────┘
 Between EVERY callback (Node >= 11): 1) drain process.nextTick queue  2) drain promise microtasks (repeat until both empty)
```
From the main script, `setTimeout(0)` vs `setImmediate` order is **not deterministic** (depends on process speed); inside an I/O callback `setImmediate` always wins (poll → check comes before timers).

```js
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout (timers phase)'), 0);
  setImmediate(() => console.log('immediate (check phase)'));
});
// Node phases: timers -> pending -> idle/prepare -> poll (I/O) -> check (setImmediate) -> close callbacks
// between EVERY callback: process.nextTick queue, then microtasks.
// ---- output (node v22.12, verified) ----
// immediate (check phase)
// timeout (timers phase)
```
**`await` desugared.** `await x` ≈ `Promise.resolve(x).then(resumeFunction)`: the function is suspended (like a generator) and its continuation is queued as a *microtask* when `x` settles. Since V8 7.2 (ES2019 spec change), `await p` on a native promise costs exactly **one** microtask tick (it used to be three). `await nonPromise` also costs one tick. `await thenable` costs an extra job to call `.then`. `return somePromise` in an async function costs ~3 ticks; `return await somePromise` ~1 tick more than a plain value but keeps `try/catch` effective.

```js
// What `await` means: a generator driven by promises (this is how async/await was first transpiled)
function run(genFn) {
  return new Promise((resolve, reject) => {
    const it = genFn();
    const step = (method, arg) => {
      let r;
      try { r = it[method](arg); } catch (e) { return reject(e); }
      if (r.done) return resolve(r.value);
      Promise.resolve(r.value).then(v => step('next', v), e => step('throw', e));   // await == Promise.resolve(x).then(resume)
    };
    step('next');
  });
}
const later = (v, ms = 5) => new Promise(r => setTimeout(r, ms, v));
run(function* () {                       // async function () {
  const a = yield later(1);              //   const a = await later(1);
  try { yield Promise.reject(new Error('x')); } catch (e) { console.log('caught', e.message); }
  const b = yield later(a + 1);
  return a + b;                          //   return a + b; }
}).then(v => console.log('result', v));
console.log('run() returned immediately');
// ---- output (node v22.12, verified) ----
// run() returned immediately
// caught x
// result 3
```
**Ordering rules to memorise:**
1. Sync code first. 2. `process.nextTick` queue (Node only; CJS main script). 3. Microtasks FIFO, *including those enqueued by microtasks*. 4. Then one macrotask; go to 3. 5. Microtask enqueue order = order in which promises *settle and have handlers* — a `.then` on an already-resolved promise is queued immediately; on a pending one, when it resolves. 6. `.then(cb)` returning a promise adds **2 extra ticks**; `new Promise(res => res(promise))` also adds 2. 7. ES module top-level code runs inside a microtask job, so `Promise.then` beats `nextTick` there (see `p15m` below).

### 4.2 Output-prediction puzzles (all verified)

Try to predict before reading the output. Trace notes follow each.

**Puzzle 1 — sync, timer, promise, queueMicrotask, async body**
```js
console.log('1 sync');
setTimeout(() => console.log('2 timeout'), 0);
Promise.resolve().then(() => console.log('3 promise'));
queueMicrotask(() => console.log('4 queueMicrotask'));
(async () => { console.log('5 async body runs sync'); await null; console.log('6 after await'); })();
console.log('7 sync end');
// ---- output (node v22.12, verified) ----
// 1 sync
// 5 async body runs sync
// 7 sync end
// 3 promise
// 4 queueMicrotask
// 6 after await
// 2 timeout
```
Trace: sync = 1, 5 (an `async` function body runs synchronously until the first `await`), 7. Microtasks in enqueue order: `.then` (3), `queueMicrotask` (4), the `await null` continuation (6). Then the timer (2).

**Puzzle 2 — microtasks vs timers, nested scheduling**
```js
setTimeout(() => {
  console.log('A timeout1');
  Promise.resolve().then(() => console.log('B micro inside timeout1'));
}, 0);
setTimeout(() => console.log('C timeout2'), 0);
Promise.resolve().then(() => {
  console.log('D micro1');
  setTimeout(() => console.log('E timeout from micro'), 0);
  Promise.resolve().then(() => console.log('F micro from micro'));
});
console.log('G sync');
// ---- output (node v22.12, verified) ----
// G sync
// D micro1
// F micro from micro
// A timeout1
// B micro inside timeout1
// C timeout2
// E timeout from micro
```
Trace: G. Microtask D runs, queues timer E and microtask F → F runs in the same drain (before any timer). Timers in order: A; after A's callback the microtask checkpoint runs B *before* timer C (microtasks drain after **each** task since Node 11 — older Node ran all timers first); then C; E was created later so it is last.

**Puzzle 3 — the classic async/await interview question**
```js
async function async1() {
  console.log('async1 start');
  await async2();
  console.log('async1 end');
}
async function async2() { console.log('async2'); }
console.log('script start');
setTimeout(() => console.log('setTimeout'), 0);
async1();
new Promise(resolve => { console.log('promise1'); resolve(); }).then(() => console.log('promise2'));
console.log('script end');
// ---- output (node v22.12, verified) ----
// script start
// async1 start
// async2
// promise1
// script end
// async1 end
// promise2
// setTimeout
```
Trace: `async1` runs until `await async2()`; `async2` logs synchronously and returns an already-resolved promise; the continuation is queued (1 tick) *before* `promise2`'s `.then` is registered (which happens when `new Promise(...)` executes, later) → `async1 end` then `promise2`. In pre-2019 engines this printed `promise2` first — mention that the answer changed with the spec.

**Puzzle 4 — two promise chains interleave**
```js
Promise.resolve().then(() => console.log('a1')).then(() => console.log('a2')).then(() => console.log('a3'));
Promise.resolve().then(() => console.log('b1')).then(() => console.log('b2')).then(() => console.log('b3'));
// ---- output (node v22.12, verified) ----
// a1
// b1
// a2
// b2
// a3
// b3
```
Each `.then` link queues the next only after the previous ran, so equal-length chains alternate: a1 b1 a2 b2 a3 b3.

**Puzzle 5 — returning a promise from `then` costs 2 extra ticks**
```js
// returning a promise from .then costs extra microtask ticks
Promise.resolve().then(() => { console.log('p1-1'); return Promise.resolve('x'); }).then(() => console.log('p1-2'));
Promise.resolve().then(() => console.log('p2-1')).then(() => console.log('p2-2')).then(() => console.log('p2-3')).then(() => console.log('p2-4'));
// ---- output (node v22.12, verified) ----
// p1-1
// p2-1
// p2-2
// p2-3
// p1-2
// p2-4
```
Chain 1's callback returns a promise, so resolving the derived promise needs `NewPromiseResolveThenableJob` + a `.then` reaction: `p1-2` lands after `p2-3`.

**Puzzle 6 — Node: nextTick, promise, queueMicrotask, timer, immediate**
```js
const fs = require('fs');
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
process.nextTick(() => console.log('nextTick'));
Promise.resolve().then(() => console.log('promise'));
queueMicrotask(() => console.log('queueMicrotask'));
console.log('sync');
fs.readFile(__filename, () => {
  // inside an I/O callback the order is deterministic: check phase (immediate) precedes timers
  setTimeout(() => console.log('io: timeout'), 0);
  setImmediate(() => console.log('io: immediate'));
  process.nextTick(() => console.log('io: nextTick'));
});
// ---- output (node v22.12, verified) ----
// sync
// nextTick
// promise
// queueMicrotask
// timeout
// immediate
// io: nextTick
// io: immediate
// io: timeout
```
Trace: sync; `nextTick` queue; microtasks (promise, queueMicrotask, in registration order); then the loop: timers phase (the `0`→1 ms timer had probably elapsed, hence `timeout` before `immediate`, but **this pair can flip** on a fast run); inside the `readFile` callback: `nextTick` first, then `immediate` (check phase) before `timeout` (next loop's timers phase) — deterministic.

**Puzzle 7 — `await` a non-promise vs a thenable**
```js
async function f() {
  console.log('f1');
  await 1;                                   // non-promise: wrapped, resumes after 1 microtask tick
  console.log('f2');
  await { then(res) { console.log('thenable.then called'); res('t'); } };  // thenable: extra job to call then
  console.log('f3');
}
f();
Promise.resolve().then(() => console.log('m1')).then(() => console.log('m2')).then(() => console.log('m3')).then(() => console.log('m4'));
// ---- output (node v22.12, verified) ----
// f1
// f2
// m1
// thenable.then called
// m2
// f3
// m3
// m4
```
`await 1` resumes after one tick (`f2` before `m1`). `await thenable` first queues a job that calls `thenable.then` (`m1` runs first), then the resume is queued after resolve, so `f3` comes after `m2`.

**Puzzle 8 — promise executor is synchronous; settle-once**
```js
console.log('before');
const p = new Promise((resolve, reject) => {
  console.log('executor runs synchronously');
  resolve('ok');
  reject(new Error('ignored: already settled'));
  console.log('code after resolve() still runs');
});
p.then(v => console.log('then', v));
console.log('after');
// ---- output (node v22.12, verified) ----
// before
// executor runs synchronously
// code after resolve() still runs
// after
// then ok
```
Only handlers are async. The second `reject` is ignored (a promise settles once), but code after `resolve()` still runs.

**Puzzle 9 — error propagation, `catch`, `finally`, `then(a, b)`**
```js
Promise.reject(new Error('boom'))
  .then(v => console.log('skipped 1'))
  .then(v => console.log('skipped 2'))
  .catch(e => { console.log('caught', e.message); return 'recovered'; })
  .then(v => console.log('then after catch:', v))
  .finally(() => { console.log('finally (no args)'); return 'ignored'; })
  .then(v => console.log('value through finally:', v));

// then(onOk, onErr): onErr does NOT catch errors thrown in onOk
Promise.resolve(1).then(() => { throw new Error('in onOk'); }, e => console.log('never')).catch(e => console.log('caught later:', e.message));
// ---- output (node v22.12, verified) ----
// caught later: in onOk
// caught boom
// then after catch: recovered
// finally (no args)
// value through finally: undefined
```
`catch` returns a *fulfilled* promise (recovery) unless it throws again. `finally` receives no argument and its return value is ignored (unless it throws/rejects) — the previous value passes through, hence `undefined` here because the previous `then` returned `undefined`. `then(onOk, onErr)`: `onErr` does **not** handle errors thrown by `onOk` in the same call.

**Puzzle 10 — async functions always return promises; unhandled rejections**
```js
process.on('unhandledRejection', r => console.log('unhandledRejection event:', r.message));
async function ok() { return 1; }
async function bad() { throw new Error('bad'); }
console.log(ok() instanceof Promise, bad().catch(() => {}) instanceof Promise);
bad().catch(e => console.log('caught:', e.message));
try { bad(); } catch (e) { console.log('never printed - rejection is async'); }   // nobody handles this one
(async () => {
  try { await bad(); } catch (e) { console.log('await + try/catch works:', e.message); }
  try { return bad(); } catch (e) { console.log('not reached: returned without await'); }
})().catch(e => console.log('outer caught:', e.message));
(async () => {
  try { return await bad(); } catch (e) { console.log('return await is caught inside:', e.message); }
})();
// ---- output (node v22.12, verified) ----
// true true
// caught: bad
// await + try/catch works: bad
// return await is caught inside: bad
// outer caught: bad
// unhandledRejection event: bad
```
`try { bad(); }` cannot catch (the function returned a rejected promise; nobody awaited it) → `unhandledRejection` event (crash in Node ≥ 15 without a handler). `return bad()` inside `try` in an async function **escapes** the `try` (the promise is returned, not awaited); `return await bad()` is caught inside — the one good use of `return await`.

**Puzzle 11 — `forEach(async …)` trap vs `for…of` vs `Promise.all(map)`**
```js
const wait = (ms, v) => new Promise(r => setTimeout(() => r(v), ms));
(async () => {
  console.log('--- forEach');
  [1, 2, 3].forEach(async n => { await wait(10 * (4 - n)); console.log('forEach done', n); });
  console.log('after forEach (did not wait)');
  await wait(60);
  console.log('--- for...of (sequential)');
  for (const n of [1, 2, 3]) { await wait(5); console.log('for-of done', n); }
  console.log('--- map + Promise.all (parallel, ordered result)');
  const r = await Promise.all([3, 2, 1].map(async n => { await wait(n * 5); console.log('finished', n); return n * 10; }));
  console.log(r);
})();
// ---- output (node v22.12, verified) ----
// --- forEach
// after forEach (did not wait)
// forEach done 3
// forEach done 2
// forEach done 1
// --- for...of (sequential)
// for-of done 1
// for-of done 2
// for-of done 3
// --- map + Promise.all (parallel, ordered result)
// finished 1
// finished 2
// finished 3
// [ 30, 20, 10 ]
```
`forEach` ignores the returned promises: it returns immediately, there is nothing to await, errors become unhandled rejections. Use `for…of` with `await` for sequential work, `Promise.all(arr.map(async …))` for parallel.

**Puzzle 12 — microtask chains run to exhaustion before timers**
```js
setTimeout(() => console.log('timeout'), 0);
let n = 0;
(function tick() { if (n++ < 3) { console.log('micro', n); queueMicrotask(tick); } })();
console.log('sync');
// microtasks queued by microtasks all run before the next macrotask; an infinite chain would starve timers and rendering
// ---- output (node v22.12, verified) ----
// micro 1
// sync
// micro 2
// micro 3
// timeout
```
`micro 1` prints during the sync call; subsequent microtasks run before `timeout`. An unbounded `queueMicrotask` loop never lets timers/rendering run.

**Puzzle 13 — sequential vs parallel awaits (timings rounded)**
```js
const sleep = (ms, v) => new Promise(r => setTimeout(() => r(v), ms));
const t0 = Date.now(); const el = () => Math.round((Date.now() - t0) / 50) * 50;
(async () => {
  await sleep(100); await sleep(100);
  console.log('sequential ~', el(), 'ms');
  const t1 = Date.now();
  await Promise.all([sleep(100), sleep(100)]);
  console.log('parallel ~', Math.round((Date.now() - t1) / 50) * 50, 'ms');
  const t2 = Date.now();
  const a = sleep(100, 'a'), b = sleep(100, 'b');          // start both, then await
  console.log(await a, await b, '~', Math.round((Date.now() - t2) / 50) * 50, 'ms');
})();
// ---- output (node v22.12, verified) ----
// sequential ~ 200 ms
// parallel ~ 100 ms
// a b ~ 100 ms
```
Awaiting in sequence adds durations (200); `Promise.all` overlaps (100); *starting* both promises before awaiting also overlaps.

**Puzzle 14 — nextTick vs promise reactions (CommonJS)**
```js
Promise.resolve().then(() => {
  console.log('promise1');
  process.nextTick(() => console.log('nextTick inside promise'));
}).then(() => console.log('promise2'));
process.nextTick(() => {
  console.log('nextTick1');
  Promise.resolve().then(() => console.log('promise inside nextTick'));
});
process.nextTick(() => console.log('nextTick2'));
// Node drains the whole nextTick queue, then the whole microtask queue, and repeats until both are empty
// ---- output (node v22.12, verified) ----
// nextTick1
// nextTick2
// promise1
// promise inside nextTick
// promise2
// nextTick inside promise
```
Node drains **all** `nextTick` callbacks, then all microtasks, and repeats while either has work: `nextTick1, nextTick2` → `promise1` → (microtask queued from nextTick1) `promise inside nextTick` → `promise2` → the `nextTick` scheduled from a promise callback only runs when the microtask queue is empty. In an **ES module** the order flips:
```js
// same as p15 but as an ES module: the module itself runs inside a microtask job
Promise.resolve().then(() => console.log('promise1'));
process.nextTick(() => console.log('nextTick1'));
console.log('sync');
// ---- output (node v22.12, verified) ----
// sync
// promise1
// nextTick1
```
**Puzzle 15 — combinators: race, any, allSettled, all**
```js
const d = (ms, v, fail) => new Promise((res, rej) => setTimeout(() => fail ? rej(new Error(v)) : res(v), ms));
(async () => {
  console.log('race:', await Promise.race([d(30, 'slow'), d(10, 'fast')]));
  try { await Promise.race([d(30, 'slow'), d(10, 'fast-fail', true)]); } catch (e) { console.log('race rejects first:', e.message); }
  console.log('any:', await Promise.any([d(12, 'e1', true), d(25, 'winner'), d(60, 'late')]));
  try { await Promise.any([d(5, 'x', true), d(6, 'y', true)]); } catch (e) { console.log(e.constructor.name, e.errors.map(x => x.message)); }
  const settled = await Promise.allSettled([d(5, 'ok'), d(6, 'no', true), 3]);
  console.log('allSettled:', settled.map(s => s.status === 'fulfilled' ? s.value : 'rejected:' + s.reason.message));
  try { await Promise.all([d(20, 'a'), d(5, 'fails first', true)]); } catch (e) { console.log('all fails fast:', e.message); }
})();
// ---- output (node v22.12, verified) ----
// race: fast
// race rejects first: fast-fail
// any: winner
// AggregateError [ 'x', 'y' ]
// allSettled: [ 'ok', 'rejected:no', 3 ]
// all fails fast: fails first
```
**Puzzle 16 — resolving a promise with a promise**
```js
const p = Promise.resolve('inner');
const q = new Promise(res => res(p));            // resolving with a promise adopts its state (costs extra ticks)
q.then(v => console.log('q', v));
Promise.resolve().then(() => console.log('t1')).then(() => console.log('t2')).then(() => console.log('t3'));
// ---- output (node v22.12, verified) ----
// t1
// t2
// q inner
// t3
```
`res(p)` schedules a job to adopt `p`'s state: `q` takes 2 extra ticks, so it lands after `t2`.

**Puzzle 17 — nested async functions**
```js
async function a() { console.log('a1'); await b(); console.log('a2'); }
async function b() { console.log('b1'); await c(); console.log('b2'); }
async function c() { console.log('c1'); }
a().then(() => console.log('a done'));
Promise.resolve().then(() => console.log('p1')).then(() => console.log('p2')).then(() => console.log('p3'));
console.log('sync end');
// ---- output (node v22.12, verified) ----
// a1
// b1
// c1
// sync end
// b2
// p1
// a2
// p2
// a done
// p3
```
Trace: `c` resolved → `b`'s continuation queued (J1); `a` awaits `b`'s pending promise; `p1` queued (J2). Drain: J1 prints `b2` and resolves `b`'s promise → queues `a`'s continuation (J3); J2 prints `p1`, queues `p2`; J3 `a2` and resolves `a`'s promise → queues `a done`; `p2`; `a done`; `p3`.


### 4.3 Promises

**States:** `pending → fulfilled(value)` or `rejected(reason)`; settled is final. `.then(onOk, onErr)` always returns a **new promise**: the callback's return value fulfils it; a *thrown* error rejects it; a returned promise/thenable is adopted. Missing handlers pass the value/error through (this is how `.catch` at the end catches everything above). Handlers run **asynchronously** (microtask) even if the promise is already settled — never "Zalgo" (sometimes sync, sometimes async).

```js
const delay = (ms, v) => new Promise(r => setTimeout(r, ms, v));
// 1. states: pending -> fulfilled | rejected (settled = immutable)
const p = new Promise(res => setTimeout(() => res('done'), 5));
console.log(p);
p.then(() => console.log(p));
// 2. each then() returns a NEW promise; return value becomes next value; thrown error becomes rejection
Promise.resolve(1)
  .then(x => x + 1)
  .then(x => { console.log('x =', x); return delay(5, x * 10); })      // returned promise is awaited
  .then(x => { console.log('after delay', x); throw new TypeError('oops'); })
  .catch(e => { console.log('caught', e.name); return 'fallback'; })
  .then(x => console.log('resumed with', x));
// 3. forgot to return -> next step gets undefined and does not wait
Promise.resolve().then(() => { delay(5, 'lost'); }).then(v => console.log('forgot return ->', v));
// 4. anti-pattern: wrapping a promise in new Promise; and nesting instead of chaining
const bad = id => new Promise((resolve, reject) => { delay(1, id).then(resolve).catch(reject); });   // pointless wrapper
const good = id => delay(1, id);
// 5. promisify a callback API correctly
const fs = require('fs');
const readP = f => new Promise((res, rej) => fs.readFile(f, 'utf8', (err, data) => err ? rej(err) : res(data)));
readP('/definitely/missing').catch(e => console.log('promisified error:', e.code));
require('util').promisify(fs.stat)(__filename).then(s => console.log('util.promisify ok:', s.isFile()));
// ---- output (node v22.12, verified) ----
// Promise { <pending> }
// x = 2
// forgot return -> undefined
// Promise { 'done' }
// promisified error: ENOENT
// util.promisify ok: true
// after delay 20
// caught TypeError
// resumed with fallback
```
**Combinators:**

| | resolves when | rejects when | notes |
|---|---|---|---|
| `Promise.all` | all fulfil (array in input order) | **first** rejection (fail-fast; others keep running!) | `[]` resolves immediately |
| `Promise.allSettled` | all settle | never | `{status, value/reason}` |
| `Promise.race` | first **settles** (either way) | first settles as rejection | timeouts; `race([])` never settles |
| `Promise.any` | first **fulfils** | all reject → `AggregateError` | fastest mirror wins |

**Anti-patterns:** (1) *Explicit construction*: `new Promise(res => api().then(res))` around something already a promise (loses rejections unless you pass `reject`). (2) Forgetting `return` inside `.then` (next step gets `undefined` and doesn't wait). (3) Nesting `.then` (pyramid) instead of flat chaining. (4) `.then(a).catch(b)` vs `.then(a, b)` — the first also catches errors from `a`. (5) `async` function with `return await` in non-try paths (pointless). (6) Swallowing errors: `.catch(() => {})`. (7) Mixing callbacks and promises; promisify once with `util.promisify`.

**Unhandled rejections:** a rejected promise with no handler by the end of the microtask drain fires `unhandledrejection` (browser, on `window`) / `process.on('unhandledRejection')`. **Node ≥ 15: default = crash with exit code 1** (like an uncaught exception). Attach handlers synchronously after creating a promise; a *late* handler triggers `rejectionHandled`.

```js
process.on('unhandledRejection', (reason, promise) => console.log('unhandledRejection:', reason.message));
process.on('rejectionHandled', () => console.log('rejectionHandled (late handler attached)'));
const late = Promise.reject(new Error('nobody listening'));
setTimeout(() => late.catch(() => console.log('late catch attached')), 20);
new Promise((_, rej) => rej(new Error('second'))).then(() => {});      // then() derives a promise that ALSO rejects unhandled
// Node >= 15 default: unhandled rejection with no handler => process crashes (exit code 1)
// ---- output (node v22.12, verified) ----
// unhandledRejection: nobody listening
// unhandledRejection: second
// late catch attached
// rejectionHandled (late handler attached)
```
Default (no handler) Node 22 behaviour for `Promise.reject(new Error('crash me'))`:
```text
[stderr] ...unhandled2.js:1  Promise.reject(new Error('crash me'));  ^
Error: crash me  at Object.<anonymous> ... Node.js v22.12.0     (process exits, code 1; later timers never run)
```

### 4.4 async / await pitfalls

1. `await` in a loop = **sequential**; independent calls → `Promise.all`. Limit concurrency with a pool (below), never `Promise.all` over 10 000 items hitting one API.
2. `forEach(async)` doesn't wait (the `forEach(async)` puzzle).
3. `return promise` inside `try` bypasses `catch`; use `return await` there.
4. `Promise.all` fail-fast doesn't cancel the rest — cancel with `AbortController`.
5. `await` only pauses its own async function; the caller continues synchronously after the first `await`.
6. Top-level `await` works in ES modules only.
7. Errors thrown before the first `await` in an `async` function still become rejections (never sync throws) — so no `try/catch` around the *call* without `await`.

```js
const wait = (ms, v) => new Promise(r => setTimeout(() => r(v), ms));
const t = () => Math.round((Date.now() - t0) / 100) * 100; let t0;
async function seqBad() { t0 = Date.now(); const a = await wait(100, 1); const b = await wait(100, 2); return [a + b, t()]; }
async function par() { t0 = Date.now(); const [a, b] = await Promise.all([wait(100, 1), wait(100, 2)]); return [a + b, t()]; }
async function limited(items, limit, worker) {          // concurrency limiter (pool of N workers)
  const results = []; let i = 0;
  await Promise.all(Array.from({ length: limit }, async () => {
    while (i < items.length) { const idx = i++; results[idx] = await worker(items[idx]); }
  }));
  return results;
}
(async () => {
  console.log('sequential', await seqBad());
  console.log('parallel  ', await par());
  t0 = Date.now();
  const r = await limited([1, 2, 3, 4, 5, 6], 2, async n => { await wait(100); return n * n; });
  console.log('pool of 2 ->', r, '~', t(), 'ms (3 waves of 100ms)');
  // timeout wrapper + AbortSignal.timeout
  const withTimeout = (p, ms) => Promise.race([p, new Promise((_, rej) => setTimeout(() => rej(new Error('timeout')), ms))]);
  try { await withTimeout(wait(300), 20); } catch (e) { console.log(e.message); }
  // error handling: await inside try/catch vs unawaited promise
  async function f() { try { return wait(5).then(() => { throw new Error('late'); }); } catch { return 'never'; } }
  try { await f(); } catch (e) { console.log('escaped the try inside f:', e.message); }
  // top-level style: sequential dependent vs independent
  const [u, o] = await Promise.all([wait(5, 'user'), wait(5, 'orders')]);
  console.log(u, o);
  // async iteration with reduce (sequential accumulate)
  const total = await [1, 2, 3].reduce(async (accP, n) => (await accP) + n, Promise.resolve(0));
  console.log('reduce total', total);
})();
// ---- output (node v22.12, verified) ----
// sequential [ 3, 200 ]
// parallel   [ 3, 100 ]
// pool of 2 -> [ 1, 4, 9, 16, 25, 36 ] ~ 300 ms (3 waves of 100ms)
// timeout
// escaped the try inside f: late
// user orders
// reduce total 6
```
---

## 5. Implement-from-scratch (all executed and tested)

### 5.1 Mini Promise (Promises/A+ core: async handlers, chaining, adoption, pass-through)
```js
class MyPromise {
  static #PENDING = 0; static #FULFILLED = 1; static #REJECTED = 2;
  #state = MyPromise.#PENDING; #value; #handlers = [];
  constructor(executor) {
    const resolve = v => this.#settle(v, true), reject = e => this.#settle(e, false);
    let called = false;                       // resolve/reject may only take effect once
    const once = fn => x => { if (!called) { called = true; fn(x); } };
    try { executor(once(resolve), once(reject)); } catch (e) { once(reject)(e); }
  }
  #settle(v, ok) {
    if (this.#state !== MyPromise.#PENDING) return;
    if (ok && v && (typeof v === 'object' || typeof v === 'function')) {      // adopt thenables
      let then; try { then = v.then; } catch (e) { return this.#finish(e, false); }
      if (typeof then === 'function') {
        let called = false;
        try { then.call(v, x => { if (!called) { called = true; this.#state = MyPromise.#PENDING; this.#settle(x, true); } },
                            e => { if (!called) { called = true; this.#state = MyPromise.#PENDING; this.#settle(e, false); } }); }
        catch (e) { if (!called) this.#finish(e, false); }
        return;
      }
    }
    this.#finish(v, ok);
  }
  #finish(v, ok) {
    this.#state = ok ? MyPromise.#FULFILLED : MyPromise.#REJECTED; this.#value = v;
    queueMicrotask(() => this.#handlers.forEach(h => this.#run(h)));
  }
  #run({ onOk, onErr, resolve, reject }) {
    const cb = this.#state === MyPromise.#FULFILLED ? onOk : onErr;
    if (typeof cb !== 'function') return this.#state === MyPromise.#FULFILLED ? resolve(this.#value) : reject(this.#value); // pass-through
    try { resolve(cb(this.#value)); } catch (e) { reject(e); }
  }
  then(onOk, onErr) {
    return new MyPromise((resolve, reject) => {
      const h = { onOk, onErr, resolve, reject };
      if (this.#state === MyPromise.#PENDING) this.#handlers.push(h);
      else queueMicrotask(() => this.#run(h));
    });
  }
  catch(onErr) { return this.then(undefined, onErr); }
  finally(fn) { return this.then(v => MyPromise.resolve(fn()).then(() => v), e => MyPromise.resolve(fn()).then(() => { throw e; })); }
  static resolve(v) { return v instanceof MyPromise ? v : new MyPromise(r => r(v)); }
  static reject(e) { return new MyPromise((_, r) => r(e)); }
}
console.log('1 sync');
new MyPromise(res => setTimeout(() => res('A'), 10))
  .then(v => { console.log('2', v); return new MyPromise(r => r(v + 'B')); })      // returns a promise -> adopted
  .then(v => { console.log('3', v); throw new Error('E'); })
  .then(() => console.log('skipped'))
  .catch(e => { console.log('4 caught', e.message); return 'C'; })
  .finally(() => console.log('5 finally'))
  .then(v => console.log('6', v));
MyPromise.resolve(1).then(v => console.log('7 microtask, after sync code:', v));
console.log('8 sync end');
// ---- output (node v22.12, verified) ----
// 1 sync
// 8 sync end
// 7 microtask, after sync code: 1
// 2 A
// 3 AB
// 4 caught E
// 5 finally
// 6 C
```
Interview points: state machine + handler queue; `then` returns a new promise; handlers deferred via `queueMicrotask`; resolve with thenable → adopt; `resolve/reject` once-only; executor throw → reject.

### 5.2 debounce & throttle
Debounce = run after `ms` of *silence* (search box, resize end, autosave). Throttle = at most once per `ms` (scroll, mousemove, rate limits). Both are closures over a timer.
```js
function debounce(fn, ms, { leading = false } = {}) {
  let t, lastArgs, lastThis;
  function debounced(...args) {
    lastArgs = args; lastThis = this;
    const callNow = leading && !t;
    clearTimeout(t);
    t = setTimeout(() => { t = null; if (!leading) fn.apply(lastThis, lastArgs); }, ms);
    if (callNow) fn.apply(this, args);
  }
  debounced.cancel = () => { clearTimeout(t); t = null; };
  debounced.flush = () => { if (t) { clearTimeout(t); t = null; fn.apply(lastThis, lastArgs); } };
  return debounced;
}
function throttle(fn, ms) {                  // leading + trailing
  let last = 0, t, lastArgs;
  return function (...args) {
    const now = Date.now(), remaining = ms - (now - last);
    lastArgs = args;
    if (remaining <= 0) { clearTimeout(t); t = null; last = now; fn.apply(this, args); }
    else if (!t) t = setTimeout(() => { last = Date.now(); t = null; fn.apply(this, lastArgs); }, remaining);
  };
}
const sleep = ms => new Promise(r => setTimeout(r, ms));
(async () => {
  const log = []; const d = debounce(x => log.push('debounced:' + x), 30);
  d(1); await sleep(10); d(2); await sleep(10); d(3); await sleep(50); d(4); await sleep(50);
  console.log(log);
  const l2 = []; const dl = debounce(x => l2.push('lead:' + x), 30, { leading: true });
  dl(1); dl(2); dl(3); await sleep(50); dl(4); await sleep(50); console.log(l2);
  const l3 = []; const th = throttle(x => l3.push(x), 40);
  for (let i = 1; i <= 10; i++) { th(i); await sleep(10); }     // ~100ms of calls
  await sleep(60);
  console.log('throttled calls:', l3.length, 'first', l3[0], 'last', l3.at(-1), '(10 calls over ~100ms+ -> far fewer executions)');
})();
// ---- output (node v22.12, verified) ----
// [ 'debounced:3', 'debounced:4' ]
// [ 'lead:1', 'lead:4' ]
// throttled calls: 5 first 1 last 10 (10 calls over ~100ms+ -> far fewer executions)
```
### 5.3 curry, compose/pipe, memoize, partial
```js
// curry (respects arity, supports placeholder-free partials)
const curry = fn => function curried(...args) {
  return args.length >= fn.length ? fn.apply(this, args) : (...more) => curried.apply(this, [...args, ...more]);
};
const add3 = (a, b, c) => a + b + c;
const c = curry(add3);
console.log(c(1)(2)(3), c(1, 2)(3), c(1)(2, 3), c(1, 2, 3));
// compose / pipe
const compose = (...fns) => x => fns.reduceRight((v, f) => f(v), x);
const pipe = (...fns) => x => fns.reduce((v, f) => f(v), x);
const inc = x => x + 1, dbl = x => x * 2;
console.log(compose(inc, dbl)(5), pipe(inc, dbl)(5));         // inc(dbl(5)) = 11, dbl(inc(5)) = 12
const pipeAsync = (...fns) => x => fns.reduce((p, f) => p.then(f), Promise.resolve(x));
pipeAsync(async x => x + 1, x => x * 3)(1).then(v => console.log('pipeAsync', v));
// memoize (key strategy matters!)
function memoize(fn, keyFn = (...a) => JSON.stringify(a)) {
  const cache = new Map();
  const m = function (...args) {
    const k = keyFn(...args);
    if (cache.has(k)) return cache.get(k);
    const v = fn.apply(this, args); cache.set(k, v); return v;
  };
  m.cache = cache; return m;
}
let calls = 0;
const slowSq = memoize(n => { calls++; return n * n; });
slowSq(4); slowSq(4); slowSq(5);
console.log('calls', calls, 'cache size', slowSq.cache.size);
const fib = memoize(n => n < 2 ? n : fib(n - 1) + fib(n - 2));
console.log(fib(50));
// once, partial
const partial = (fn, ...preset) => (...rest) => fn(...preset, ...rest);
console.log(partial(Math.max, 10)(3, 20));
// ---- output (node v22.12, verified) ----
// 6 6 6 6
// 11 12
// calls 2 cache size 2
// 12586269025
// 20
// pipeAsync 6
```
Memoize caveats: cache key (`JSON.stringify` breaks for functions/cycles/`undefined`), unbounded memory (use LRU or `WeakMap` for object keys), only for **pure** functions.

### 5.4 deepClone (circular refs, Date, RegExp, Map, Set, symbols, prototypes)
```js
function deepClone(value, seen = new WeakMap()) {
  if (value === null || typeof value !== 'object') return value;          // primitives + functions (returned as-is)
  if (seen.has(value)) return seen.get(value);                            // circular refs
  if (value instanceof Date) return new Date(value);
  if (value instanceof RegExp) return new RegExp(value.source, value.flags);
  if (value instanceof Map) { const m = new Map(); seen.set(value, m); value.forEach((v, k) => m.set(deepClone(k, seen), deepClone(v, seen))); return m; }
  if (value instanceof Set) { const s = new Set(); seen.set(value, s); value.forEach(v => s.add(deepClone(v, seen))); return s; }
  const out = Array.isArray(value) ? [] : Object.create(Object.getPrototypeOf(value));
  seen.set(value, out);
  for (const key of Reflect.ownKeys(value)) out[key] = deepClone(value[key], seen);   // includes symbol keys
  return out;
}
const src = { n: 1, d: new Date(0), re: /a+/gi, arr: [1, { x: 2 }], m: new Map([['k', { v: 1 }]]), s: new Set([1]), [Symbol('s')]: 'sym', u: undefined };
src.self = src;
const cl = deepClone(src);
console.log(cl !== src, cl.arr[1] !== src.arr[1], cl.self === cl, cl.d instanceof Date, cl.m.get('k') !== src.m.get('k'), cl.re.flags);
console.log(JSON.stringify({ d: new Date(0), u: undefined, f() {}, n: NaN, i: Infinity }), JSON.parse(JSON.stringify({ d: new Date(0) })).d);
try { JSON.stringify(src); } catch (e) { console.log(e.name); }
// ---- output (node v22.12, verified) ----
// true true true true true gi
// {"d":"1970-01-01T00:00:00.000Z","n":null,"i":null} 1970-01-01T00:00:00.000Z
// TypeError
```
`JSON.parse(JSON.stringify(x))` loses `undefined`, functions, symbols, `Date` (→ string), `NaN/Infinity` (→ null), Map/Set, and throws on cycles. Prefer `structuredClone` (built-in) unless you need functions/prototypes.

### 5.5 Array polyfills (map, filter, reduce) and flatten
```js
Array.prototype.myMap = function (cb, thisArg) {
  if (typeof cb !== 'function') throw new TypeError(cb + ' is not a function');
  const out = new Array(this.length);
  for (let i = 0; i < this.length; i++) if (i in this) out[i] = cb.call(thisArg, this[i], i, this);   // skips holes
  return out;
};
Array.prototype.myFilter = function (cb, thisArg) { const out = []; for (let i = 0; i < this.length; i++) if (i in this && cb.call(thisArg, this[i], i, this)) out.push(this[i]); return out; };
Array.prototype.myReduce = function (cb, ...init) {
  let i = 0, acc;
  if (init.length) acc = init[0];
  else { while (i < this.length && !(i in this)) i++; if (i >= this.length) throw new TypeError('Reduce of empty array with no initial value'); acc = this[i++]; }
  for (; i < this.length; i++) if (i in this) acc = cb(acc, this[i], i, this);
  return acc;
};
console.log([1, 2, 3].myMap(x => x * 2), [1, , 3].myMap(x => x * 2), [1, 2, 3, 4].myFilter(x => x % 2 === 0));
console.log([1, 2, 3].myReduce((a, b) => a + b), [[1], [2]].myReduce((a, b) => a.concat(b), []));
try { [].myReduce((a, b) => a + b); } catch (e) { console.log(e.message); }
// flatten: recursive, iterative, and reduce forms
const flat = (a, depth = Infinity) => depth < 1 ? a.slice() : a.reduce((acc, x) => Array.isArray(x) ? acc.concat(flat(x, depth - 1)) : (acc.push(x), acc), []);
const flatIter = a => { const st = [...a], out = []; while (st.length) { const x = st.pop(); Array.isArray(x) ? st.push(...x) : out.push(x); } return out.reverse(); };
const nested = [1, [2, [3, [4, [5]]]]];
console.log(flat(nested), flat(nested, 1), flatIter(nested), nested.flat(Infinity));
// group / chunk / unique helpers often asked
const chunk = (a, n) => Array.from({ length: Math.ceil(a.length / n) }, (_, i) => a.slice(i * n, i * n + n));
console.log(chunk([1, 2, 3, 4, 5], 2), [...new Set([1, 1, 2])]);
// ---- output (node v22.12, verified) ----
// [ 2, 4, 6 ] [ 2, <1 empty item>, 6 ] [ 2, 4 ]
// 6 [ 1, 2 ]
// Reduce of empty array with no initial value
// [ 1, 2, 3, 4, 5 ] [ 1, 2, [ 3, [ 4, [Array] ] ] ] [ 1, 2, 3, 4, 5 ] [ 1, 2, 3, 4, 5 ]
// [ [ 1, 2 ], [ 3, 4 ], [ 5 ] ] [ 1, 2 ]
```
Detail: spec `map` skips holes (`i in this`), passes `(item, index, array)`, and honours `thisArg`; `reduce` without an initial value on an empty array throws `TypeError`.

### 5.6 EventEmitter and pub-sub
```js
class EventEmitter {
  #h = new Map();
  on(evt, fn) { (this.#h.get(evt) ?? this.#h.set(evt, []).get(evt)).push(fn); return this; }
  once(evt, fn) { const w = (...a) => { this.off(evt, w); fn(...a); }; w.orig = fn; return this.on(evt, w); }
  off(evt, fn) { const l = this.#h.get(evt); if (l) this.#h.set(evt, l.filter(f => f !== fn && f.orig !== fn)); return this; }
  emit(evt, ...args) { const l = this.#h.get(evt); if (!l?.length) return false; [...l].forEach(f => f.apply(this, args)); return true; }  // copy: safe if handlers unsubscribe
}
const e = new EventEmitter();
const h = x => console.log('h', x);
e.on('a', h).once('a', x => console.log('once', x));
e.emit('a', 1); e.emit('a', 2); e.off('a', h); console.log(e.emit('a', 3));
// Pub-sub with topic wildcards and unsubscribe handle
class PubSub {
  #subs = new Map(); #id = 0;
  subscribe(topic, fn) { const id = ++this.#id; (this.#subs.get(topic) ?? this.#subs.set(topic, new Map()).get(topic)).set(id, fn); return () => this.#subs.get(topic)?.delete(id); }
  publish(topic, data) { for (const [t, subs] of this.#subs) if (t === topic || t === '*') subs.forEach(fn => fn(data, topic)); }
}
const ps = new PubSub();
const unsub = ps.subscribe('order', d => console.log('order handler', d));
ps.subscribe('*', (d, t) => console.log('audit', t, d));
ps.publish('order', { id: 1 }); unsub(); ps.publish('order', { id: 2 });
// ---- output (node v22.12, verified) ----
// h 1
// once 1
// h 2
// false
// order handler { id: 1 }
// audit order { id: 1 }
// audit order { id: 2 }
```
Edge cases interviewers add: `once`, `off` of a `once` handler (the `orig` trick), removal during emit (iterate a copy), error event semantics, memory leaks from never unsubscribing (Node warns at >10 listeners).

### 5.7 Promise.all / allSettled / race / any polyfills
```js
Promise.myAll = iterable => new Promise((resolve, reject) => {
  const items = [...iterable]; const results = new Array(items.length); let left = items.length;
  if (!left) return resolve([]);
  items.forEach((p, i) => Promise.resolve(p).then(v => { results[i] = v; if (--left === 0) resolve(results); }, reject));  // order preserved by index
});
Promise.myAllSettled = it => Promise.myAll([...it].map(p => Promise.resolve(p).then(value => ({ status: 'fulfilled', value }), reason => ({ status: 'rejected', reason }))));
Promise.myRace = it => new Promise((res, rej) => [...it].forEach(p => Promise.resolve(p).then(res, rej)));
Promise.myAny = it => new Promise((res, rej) => {
  const items = [...it], errs = []; let left = items.length; if (!left) return rej(new AggregateError([], 'All promises were rejected'));
  items.forEach((p, i) => Promise.resolve(p).then(res, e => { errs[i] = e; if (--left === 0) rej(new AggregateError(errs, 'All promises were rejected')); }));
});
const d = (ms, v, fail) => new Promise((r, j) => setTimeout(() => fail ? j(new Error(v)) : r(v), ms));
(async () => {
  console.log(await Promise.myAll([d(20, 'a'), 'plain', d(5, 'c')]), await Promise.myAll([]));
  try { await Promise.myAll([d(20, 'a'), d(5, 'boom', true)]); } catch (e) { console.log('rejected:', e.message); }
  console.log((await Promise.myAllSettled([d(5, 'x'), d(5, 'y', true)])).map(r => r.status));
  console.log(await Promise.myRace([d(20, 'slow'), d(5, 'fast')]), await Promise.myAny([d(5, 'e', true), d(10, 'ok')]));
})();
// ---- output (node v22.12, verified) ----
// [ 'a', 'plain', 'c' ] []
// rejected: boom
// [ 'fulfilled', 'rejected' ]
// fast ok
```
Details: normalise with `Promise.resolve(p)` (accepts plain values), preserve **index order**, empty-input edge case, count down instead of re-checking the array.

### 5.8 LRU cache
```js
class LRUCache {
  #cap; #map = new Map();                 // Map preserves insertion order: first key = least recently used
  constructor(cap) { this.#cap = cap; }
  get(k) {
    if (!this.#map.has(k)) return undefined;
    const v = this.#map.get(k); this.#map.delete(k); this.#map.set(k, v);   // refresh recency
    return v;
  }
  put(k, v) {
    if (this.#map.has(k)) this.#map.delete(k);
    else if (this.#map.size >= this.#cap) this.#map.delete(this.#map.keys().next().value);  // evict LRU
    this.#map.set(k, v);
  }
  get keys() { return [...this.#map.keys()]; }
}
const lru = new LRUCache(2);
lru.put('a', 1); lru.put('b', 2); lru.get('a'); lru.put('c', 3);   // b is evicted
console.log(lru.keys, lru.get('b'), lru.get('a'));
// Same in Java: LinkedHashMap(cap, .75f, true) + removeEldestEntry(). The hand-rolled version is HashMap + doubly linked list, O(1) get/put.
// ---- output (node v22.12, verified) ----
// [ 'a', 'c' ] undefined 1
```
### 5.9 Retry with exponential backoff + jitter
```js
const sleep = ms => new Promise(r => setTimeout(r, ms));
async function retry(fn, { retries = 3, base = 10, factor = 2, max = 1000, jitter = true, shouldRetry = () => true, onRetry = () => {} } = {}) {
  for (let attempt = 0; ; attempt++) {
    try { return await fn(attempt); }
    catch (err) {
      if (attempt >= retries || !shouldRetry(err)) throw err;
      let delay = Math.min(max, base * factor ** attempt);
      if (jitter) delay = Math.random() * delay;             // "full jitter" avoids thundering herd
      onRetry(err, attempt + 1, Math.round(delay));
      await sleep(delay);
    }
  }
}
let n = 0;
retry(async () => { if (++n < 3) throw Object.assign(new Error('flaky ' + n), { status: 503 }); return 'success on attempt ' + n; },
  { onRetry: (e, a) => console.log('retry', a, 'after', e.message) }).then(console.log);
retry(async () => { throw Object.assign(new Error('bad request'), { status: 400 }); }, { shouldRetry: e => e.status >= 500 })
  .catch(e => console.log('not retried:', e.message));
// Also: only retry idempotent operations (GET, PUT with idempotency key) - never blind-retry a payment POST.
// ---- output (node v22.12, verified) ----
// retry 1 after flaky 1
// not retried: bad request
// retry 2 after flaky 2
// success on attempt 3
```
---

## 6. Modules, errors, memory

### 6.1 ESM vs CommonJS

| | CommonJS (`require`) | ES Modules (`import`) |
|---|---|---|
| Loading | **Synchronous**, at runtime (can be inside `if`) | Static syntax, analysed before execution; async loading; `import()` for dynamic |
| Exports | **Copy** of `module.exports` (value snapshot for primitives) | **Live bindings** (read-only views of the exporter's variable) |
| `this` at top level | `module.exports` | `undefined` |
| `__dirname`, `require` | yes | no (use `import.meta.url`, `createRequire`) |
| Top-level `await` | no | yes |
| Strict mode | opt-in | always |
| Tree-shaking | hard (dynamic) | possible (static structure) |
| Cycles | partial exports at time of cycle | bindings in TDZ until evaluated |
| Node selection | `.js` (default) / `.cjs` | `.mjs` or `"type": "module"` |

```js
import hello, { count, inc } from './_esm_counter.mjs';
import * as ns from './_esm_counter.mjs';
console.log(count); inc(); console.log(count, ns.count, '<- live binding, importer sees the update');
try { count = 5; } catch (e) { console.log(e.name + ':', e.message); }     // imports are read-only views
console.log(hello(), Object.keys(ns), typeof require, typeof this, import.meta.url.startsWith('file:'));
const dyn = await import('./_esm_counter.mjs');                              // top-level await + dynamic import
console.log(dyn.default === hello, dyn === ns);
// ---- output (node v22.12, verified) ----
// 0
// 1 1 <- live binding, importer sees the update
// TypeError: Assignment to constant variable.
// default export [ 'count', 'default', 'inc' ] undefined undefined true
// true true
```
Circular imports. CJS gives the second module a **partially filled `exports`** object:
```js
exports.loaded = false;
const b = require('./_cjs_b');
console.log('in a: b.done =', b.done);
exports.loaded = true;
```
```js
exports.done = false;
const a = require('./_cjs_a');
console.log('in b: a.loaded =', a.loaded, '(a only partially executed!)');
exports.done = true;
```
```js
require('./_cjs_a');
const a1 = require('./_cjs_a');
console.log('main: a.loaded =', a1.loaded, '| cached module, no re-run');
// CJS exports are copies of primitive values at require time; module.exports is an object
const counter = { n: 0, inc() { this.n++; } };
console.log(typeof require, typeof module, typeof exports, exports === module.exports, __filename.endsWith('.js'));
// ---- output (node v22.12, verified) ----
// in b: a.loaded = false (a only partially executed!)
// in a: b.done = true
// main: a.loaded = true | cached module, no re-run
// function object object true true
```
ESM: modules are evaluated depth-first; in a cycle, an export that is a `const/let` is still in the **TDZ** when the other module touches it, while function declarations are already usable:
```js
import { b } from './_esm_b.mjs';
export const a = 'A';
export function useB() { return 'a uses ' + b; }
console.log('a evaluating; b =', b);
```
```js
import { a, useB } from './_esm_a.mjs';
export const b = 'B';
try { console.log(a); } catch (e) { console.log('b evaluating; touching a ->', e.name); }   // a is in TDZ
export function callA() { return typeof useB; }
console.log('b evaluating; useB is', typeof useB, '(function declarations are hoisted across the cycle)');
```
```text
b evaluating; touching a -> ReferenceError
b evaluating; useB is function (function declarations are hoisted across the cycle)
a evaluating; b = B
```
Fix cycles by extracting shared code into a third module, injecting dependencies, or using lazy access inside functions (not at module top level). Bundlers/Jest may treat interop differently (`default` vs named, `__esModule`).

### 6.2 Error handling

- `throw` any value, but always throw `Error` objects (stack trace). Subclass: `class ValidationError extends Error { constructor(m){ super(m); this.name='ValidationError'; } }`; `new Error('msg', { cause: err })` (ES2022) chains causes.
- `try/catch/finally`; `finally` runs even after `return` (and can override it — Q20 in drills). Optional catch binding `catch {}`.
- Sync errors: `try/catch`. Promise errors: `.catch` / `await` in `try`. Callback errors: error-first convention `(err, data)`. Event emitters: `'error'` event (unhandled = throws).
- Global nets: browser `window.onerror`, `addEventListener('unhandledrejection')`; Node `process.on('uncaughtException')` (log, then **exit** — state may be corrupt) and `unhandledRejection`.
- React: error boundaries only catch render errors, not async/event-handler errors.
- Don't use exceptions for control flow; validate at boundaries; report to Sentry with a release/source-map.

### 6.3 Garbage collection basics

V8 uses **generational mark-and-sweep** (young generation "scavenger" for short-lived objects; old generation mark-sweep-compact, incremental/concurrent to limit pauses). An object is collected when it is **unreachable from roots** (globals, current stack, active closures, DOM). **No reference counting** (cycles are no problem). Compare to Java: same reachability idea; no manual `System.gc()` — `global.gc()` exists only with `--expose-gc`.

### 6.4 Memory leaks in browsers — the usual suspects

| Leak | Why | Fix |
|---|---|---|
| **Detached DOM nodes** | node removed from document but referenced by a JS variable/array/closure | null the reference; don't cache nodes in globals; `WeakRef`/`WeakMap` |
| **Event listeners never removed** | listener closure holds elements/state | `removeEventListener`, `AbortController` signal (`addEventListener(t, fn, { signal })`), `{ once: true }`; in React return cleanup from `useEffect` |
| **Timers** | `setInterval` callback keeps its closure alive forever | `clearInterval` in teardown |
| **Closures** | long-lived closure references big data | narrow what you capture; null out |
| **Unbounded caches/globals** | Map that only grows | LRU / TTL / `WeakMap` |
| **Accidental globals** | `x = 1` without declaration (sloppy) | `'use strict'`, ESLint `no-undef` |
| **Console.log of objects in DevTools** | dev-only retention | remove in prod |
| **SPA route changes** | old page's subscriptions (RxJS, websockets, observers) not torn down | unsubscribe / disconnect |

How to find: Chrome DevTools → **Memory** tab → take **heap snapshots** before/after an action, compare (**Comparison view**, filter "Detached"); *Performance monitor* (JS heap, DOM nodes, listeners); *Allocation timeline*. A saw-tooth heap = healthy; a staircase that never drops = leak.

### 6.5 Measured demo (Node `--expose-gc`)
```js
const { spawnSync } = require('child_process');
if (process.argv[2] !== 'child') { spawnSync(process.execPath, ['--expose-gc', __filename, 'child'], { stdio: 'inherit' }); return; }
(async () => {
  const strongCache = new Map(), weakCache = new WeakMap(); const refs = [];
  (() => {
    for (let i = 0; i < 3; i++) {
      const obj = { payload: new Array(1000).fill(i) };      // stands in for a DOM node / big object
      strongCache.set(obj, 'meta'); weakCache.set(obj, 'meta'); refs.push(new WeakRef(obj));
    }
  })();
  await new Promise(r => setTimeout(r, 10));                 // WeakRef targets are kept alive until the end of the current job
  gc();
  console.log('objects still alive (Map key holds them):', refs.filter(r => r.deref()).length);
  strongCache.clear(); await new Promise(r => setTimeout(r, 10)); gc();
  console.log('after clearing the Map, still alive:', refs.filter(r => r.deref()).length, '(WeakMap never prevented collection)');
  // closure leak: big array kept alive by a long-lived closure
  const handlers = [];
  function attach() { const big = new Array(1e5).fill('x'); handlers.push(() => big.length); }
  const before = process.memoryUsage().heapUsed; for (let i = 0; i < 20; i++) attach(); gc();
  console.log('heap grew by > 10MB because closures retain `big`:', process.memoryUsage().heapUsed - before > 10e6);
  handlers.length = 0; gc();
  console.log('after dropping handlers, heap back to near baseline:', process.memoryUsage().heapUsed - before < 5e6);
})();
// ---- output (node v22.12, verified) ----
// objects still alive (Map key holds them): 3
// after clearing the Map, still alive: 0 (WeakMap never prevented collection)
// heap grew by > 10MB because closures retain `big`: true
// after dropping handlers, heap back to near baseline: true
```
---

## 7. Browser: DOM, network, storage, security, performance

### 7.1 DOM events: capture, target, bubble, delegation

```
 click on <button> inside <li> inside <ul>:      window → document → html → body → ul → li → button
   1. CAPTURING phase (top → target)   listeners with { capture: true }
   2. TARGET phase
   3. BUBBLING phase  (target → top)   default for addEventListener / onclick
```
```js
// browser (not executed)
ul.addEventListener('click', e => {                 // event DELEGATION: 1 listener instead of N
  const li = e.target.closest('li[data-id]');       // e.target = actual clicked node, e.currentTarget = ul
  if (!li || !ul.contains(li)) return;
  remove(li.dataset.id);
});
form.addEventListener('submit', e => { e.preventDefault(); /* stop native submit; event still bubbles */ });
child.addEventListener('click', e => e.stopPropagation()); // stop bubbling to parents (other listeners on the SAME node still run)
// e.stopImmediatePropagation() also stops the remaining listeners on this node
window.addEventListener('scroll', onScroll, { passive: true });   // promise never to call preventDefault -> browser scrolls without waiting
el.addEventListener('click', fn, { once: true, signal: abortController.signal }); // auto-remove, or abort() to remove many at once
```
- **`preventDefault`** cancels the browser's default action (link navigation, form submit, checkbox toggle); **`stopPropagation`** stops the event travelling further up/down. They are independent; overusing `stopPropagation` breaks analytics/delegated handlers.
- **Delegation** wins for dynamic lists (works for elements added later), uses less memory. Doesn't work for events that don't bubble (`focus`, `blur`, `mouseenter/leave` — use `focusin/focusout`, `mouseover/out`).
- **Passive listeners**: touch/wheel listeners on `window/document/body` are passive by default in Chrome; needed for smooth scrolling (INP).
- Inline `onclick` sets one handler (overwrites); `addEventListener` stacks handlers. `DOMContentLoaded` = DOM parsed (deferred scripts done); `load` = all resources (images, css) loaded.
- Layout thrash: reading `offsetHeight` after writing styles in a loop forces synchronous reflow each iteration; batch reads then writes (or `requestAnimationFrame`).

### 7.2 Fetch API

`fetch` rejects **only on network failure/abort/CORS block** — HTTP 404/500 *resolve* with `res.ok === false`. You must check `res.ok`, and `res.json()` can itself throw (empty body/HTML error page). Body can be read once (`res.clone()`). No timeout by default → `AbortSignal.timeout(ms)` or `AbortController`. Verified against a local server:

```js
const http = require('http');
const server = http.createServer((req, res) => {
  if (req.url === '/ok') { res.setHeader('content-type', 'application/json'); return res.end(JSON.stringify({ hello: 'world' })); }
  if (req.url === '/slow') return setTimeout(() => res.end('late'), 500);
  if (req.url === '/badjson') return res.end('<html>not json</html>');
  res.statusCode = req.url === '/500' ? 500 : 404; res.end('nope');
});
async function http_(url, { timeout = 100, ...opts } = {}) {         // production-style wrapper
  const res = await fetch(url, { ...opts, signal: AbortSignal.timeout(timeout) });
  if (!res.ok) throw Object.assign(new Error(`HTTP ${res.status}`), { status: res.status });
  return res.json();
}
server.listen(0, async () => {
  const base = `http://127.0.0.1:${server.address().port}`;
  const r404 = await fetch(base + '/missing');
  console.log('404 did NOT reject: ok =', r404.ok, 'status =', r404.status);
  console.log(await http_(base + '/ok'));
  for (const path of ['/500', '/slow', '/badjson']) {
    try { await http_(base + path); } catch (e) { console.log(path, '->', e.name, '|', e.message.slice(0, 40)); }
  }
  try { await fetch('http://127.0.0.1:1/'); } catch (e) { console.log('network failure rejects:', e.name, '|', e.message, '| cause:', e.cause?.constructor.name); }
  const ac = new AbortController();
  const p = fetch(base + '/slow', { signal: ac.signal }).catch(e => console.log('manual abort:', e.name));
  ac.abort(); await p;
  server.close();
});
// ---- output (node v22.12, verified) ----
// 404 did NOT reject: ok = false status = 404
// { hello: 'world' }
// /500 -> Error | HTTP 500
// /slow -> TimeoutError | The operation was aborted due to timeout
// /badjson -> SyntaxError | Unexpected token '<', "<html>not "... is
// network failure rejects: TypeError | fetch failed | cause: Error
// manual abort: AbortError
```
```js
// browser: typical wrapper with credentials + JSON + abort
const ac = new AbortController();
const res = await fetch('/api/orders', {
  method: 'POST', credentials: 'include',            // send cookies (default 'same-origin')
  headers: { 'Content-Type': 'application/json' },   // NOTE: this header makes it a non-simple request -> preflight cross-origin
  body: JSON.stringify(order), signal: ac.signal,
});
```
Axios vs fetch: axios rejects on non-2xx, has interceptors, timeouts, automatic JSON, request cancellation; fetch is native (no dependency), streams the body.

### 7.3 CORS in depth

**Same-Origin Policy**: origin = **scheme + host + port**. The browser lets a page *send* cross-origin requests but blocks *reading* the response unless the server opts in. CORS is a browser-enforced, server-configured relaxation. `curl`/Postman/server-to-server calls are not affected — so CORS is not server protection.

```
Simple request (GET/HEAD/POST + only safelisted headers + Content-Type in {text/plain, multipart/form-data, x-www-form-urlencoded}):
  Browser ──GET /data  Origin: https://app.com──────────────► API
          ◄─200  Access-Control-Allow-Origin: https://app.com  (else browser hides the response from JS)

Preflighted request (PUT/DELETE/PATCH, custom headers e.g. Authorization, or Content-Type: application/json):
  Browser ──OPTIONS /data
            Origin: https://app.com
            Access-Control-Request-Method: PUT
            Access-Control-Request-Headers: authorization,content-type ─► API
          ◄─204  Access-Control-Allow-Origin: https://app.com
                 Access-Control-Allow-Methods: GET,PUT,DELETE
                 Access-Control-Allow-Headers: authorization,content-type
                 Access-Control-Max-Age: 600            (preflight result cached)
  Browser ──PUT /data (the real request) ──► API ◄─200 + Access-Control-Allow-Origin
```
**Credentials** (cookies/HTTP auth): client `credentials: 'include'`; server must send `Access-Control-Allow-Credentials: true` **and** an explicit origin (not `*`), explicit allowed headers/methods (not `*`), plus `Vary: Origin`. Cross-site cookies also need `SameSite=None; Secure`. To read non-safelisted response headers (e.g. `X-Total-Count`, `Location`) server sends `Access-Control-Expose-Headers`.

**Why the server must fix it:** the *browser* enforces the check by looking at the *response headers the server sends*; front-end code cannot legitimately change them. `mode: 'no-cors'` only yields an opaque, unreadable response. Workarounds that do work: a same-origin reverse proxy/API gateway or BFF (nginx `location /api`, Vite/CRA dev `proxy`) so the browser sees one origin.

**Spring (Java) specifics:** `@CrossOrigin(origins = "https://app.com")` or a global `WebMvcConfigurer#addCorsMappings`; with **Spring Security** you must enable `http.cors(...)` with a `CorsConfigurationSource` so the CORS filter runs *before* authentication — otherwise the preflight (which carries no `Authorization` header) gets **401/403** and the browser reports a CORS error. Common bugs: `allowedOrigins("*")` + `allowCredentials(true)` (illegal → use `allowedOriginPatterns`), duplicate `Access-Control-Allow-Origin` headers (gateway *and* service both add → browser rejects), redirect on the OPTIONS request, missing `OPTIONS` in allowed methods, error responses (500) without CORS headers hiding the real error behind "CORS error".

### 7.4 Browser storage and auth token trade-offs

| | Cookie | localStorage | sessionStorage | IndexedDB |
|---|---|---|---|---|
| Capacity | ~4 KB each | ~5 MB | ~5 MB | hundreds MB+ (quota) |
| Sent to server automatically | **yes** (every matching request) | no | no | no |
| Lifetime | Expires/Max-Age or session | forever | tab lifetime | forever |
| API | string, `document.cookie` / `Set-Cookie` | sync, strings | sync, strings | async, structured (objects, blobs), indexes, transactions |
| Readable by JS | not if `HttpOnly` | yes (XSS-stealable) | yes | yes |
| Scope | domain+path | origin | origin + tab | origin |
| Use for | session/refresh tokens | UI prefs, non-sensitive cache | wizard state | offline data, large caches (PWA) |

**Cookie attributes:** `HttpOnly` (invisible to JS → XSS can't read it), `Secure` (HTTPS only), `SameSite=Strict|Lax|None` (cross-site sending policy; `Lax` = default in modern browsers: sent on top-level GET navigations, not on cross-site POST/`fetch`; `None` requires `Secure`), `Domain`/`Path`, `Max-Age`/`Expires`, prefixes `__Host-`/`__Secure-`.

`Set-Cookie: sid=abc123; HttpOnly; Secure; SameSite=Lax; Path=/; Max-Age=3600`

**XSS vs CSRF**

| | XSS (Cross-Site Scripting) | CSRF (Cross-Site Request Forgery) |
|---|---|---|
| Attacker | Runs **their JS in your origin** | Tricks the victim's browser into **sending an authenticated request** |
| Abuses | Trust the user has in the site | Trust the site has in the browser (cookies auto-sent) |
| Steals tokens? | Yes if in localStorage/JS-readable | No (attacker can't read the response) |
| Defences | Output encoding, CSP, sanitise HTML, HttpOnly, avoid `innerHTML` | `SameSite` cookies, CSRF tokens (synchroniser/double-submit), check `Origin`, custom header + CORS, re-auth for sensitive ops |
| If you use Bearer header from JS | vulnerable via token theft | immune (header not auto-sent) |

**Where to store the JWT (the standard answer):** localStorage is simplest but any XSS (including a compromised npm/third-party script) exfiltrates it. Best-practice combo: short-lived **access token in memory** (JS variable) + **refresh token in `HttpOnly; Secure; SameSite` cookie** with rotation; or a **BFF/session cookie** so the browser never sees tokens. If tokens are in cookies you need CSRF protection (SameSite + token). Spring Security enables CSRF by default for session-cookie apps; stateless JWT APIs often disable it *only because* they use an Authorization header.

### 7.5 Web security basics

- **XSS types:** *Stored* (payload saved in DB, shown to all: comments), *Reflected* (payload in URL/query echoed by server), *DOM-based* (client JS writes untrusted data into `innerHTML`, `document.write`, `eval`, `location`). **Defence in depth:** contextual output encoding (React/Angular escape by default; escape hatches: `dangerouslySetInnerHTML`, `bypassSecurityTrust*`, `v-html`), `textContent` over `innerHTML`, sanitise rich text with **DOMPurify** (server allow-lists too), **CSP**, Trusted Types, `HttpOnly` cookies, validate `href` schemes (`javascript:`).
- **CSP** (`Content-Security-Policy` header) whitelists where scripts/styles/images/connections may come from: `default-src 'self'; script-src 'self' 'nonce-r4nd0m'; object-src 'none'; frame-ancestors 'none'; base-uri 'self'`. Blocks inline scripts unless nonce/hash; use `Report-Only` first. Avoid `'unsafe-inline'`/`'unsafe-eval'`.
- **Other:** clickjacking (`frame-ancestors`/`X-Frame-Options`), HSTS + HTTPS, `rel="noopener"` on `target=_blank`, Subresource Integrity for CDN scripts, prototype pollution (`__proto__` in merge/parse → use `Object.create(null)`/`Map`, patched libs), ReDoS, open redirect, `npm audit`/lockfile/`--ignore-scripts` for supply chain, never trust client-side validation (server re-validates — Bean Validation), secrets are never safe in the bundle (`VITE_*`/`REACT_APP_*` are public).

### 7.6 Performance

**Critical rendering path:** HTML → DOM; CSS → CSSOM (CSS is **render-blocking**); DOM+CSSOM → render tree → **layout** (geometry) → **paint** → **composite** (GPU layers). Synchronous `<script>` is **parser-blocking** (it must wait for CSS and blocks HTML parsing).

| Script tag | Download | Execution | Order kept | Use |
|---|---|---|---|---|
| `<script>` | blocks parser | immediately, blocks parser | yes | rare |
| `<script async>` | parallel | **as soon as ready** (may interrupt parsing) | no | analytics, independent scripts |
| `<script defer>` | parallel | **after HTML parsed**, before `DOMContentLoaded` | **yes** | app bundles (default choice) |
| `<script type="module">` | parallel | deferred by default | yes | modern apps |

**Reflow vs repaint:** *reflow* (layout) recalculates geometry — triggered by width/height/margin/font/DOM changes and by reading layout values (`offsetTop`) after writes; *repaint* redraws pixels (color, shadow); *composite-only* (`transform`, `opacity`) skips both and is GPU-friendly → animate these.

**Levers:** code-splitting + lazy routes (`import()`, `React.lazy`); **tree-shaking** (static ESM + `sideEffects:false`, import `lodash-es`/per-method, avoid moment.js); minify + brotli/gzip; image formats (AVIF/WebP), `srcset`, `loading="lazy"`, explicit `width/height` (prevents CLS); `<link rel="preconnect|preload|prefetch">`; fonts `font-display: swap`; **HTTP caching**: hashed filenames + `Cache-Control: public, max-age=31536000, immutable`; `index.html` with `no-cache` (revalidate via ETag → 304) — else users see stale apps; `no-store` = never cache (sensitive data); CDN; HTTP/2/3; virtualise long lists; debounce/throttle handlers; move CPU work to Web Workers; avoid long tasks (> 50 ms) — split with `scheduler.yield()`/`setTimeout`/`requestIdleCallback`.

**Core Web Vitals (75th percentile targets):** **LCP** Largest Contentful Paint ≤ **2.5 s** (hero image/text render; fix: preload hero, server speed, CDN); **INP** Interaction to Next Paint ≤ **200 ms** (replaced FID in Mar 2024; fix: shorter tasks, less JS, avoid layout thrash); **CLS** Cumulative Layout Shift ≤ **0.1** (reserve space for images/ads, no late-inserted content above). Measure with Lighthouse (lab), CrUX / `web-vitals` library (field).

### 7.7 Web Workers, Service Workers, PWA

- **Web Worker**: separate thread with its own event loop, no DOM; communicate with `postMessage` (structured clone copy; `Transferable`s like `ArrayBuffer` move without copy; `SharedArrayBuffer` + `Atomics` truly share). Use for parsing big JSON, image processing, crypto. Node analogue: `worker_threads` (demo in §9).
- **Service Worker**: programmable network proxy between page and network, runs off-page, HTTPS only, event-driven (`install`, `activate`, `fetch`). Enables offline caching strategies (cache-first, network-first, stale-while-revalidate), push notifications, background sync. Lifecycle gotcha: a new SW waits until old tabs close (`skipWaiting`/`clients.claim`); a buggy cached SW can serve a broken app — version caches, ship a kill switch.
- **PWA** = HTTPS + web app manifest (name, icons, `display: standalone`) + service worker → installable, offline-capable.

---

## 8. TypeScript & tooling

### 8.1 TypeScript essentials (not executed here — needs `tsc`)
TypeScript = JavaScript + a **static type system that is erased at compile time** (no runtime cost, no runtime checks — validate external data with Zod/Joi/Bean-Validation-equivalent). Benefits: refactor safety, IDE autocomplete, contracts with your Java DTOs (generate types from OpenAPI), catches `undefined` bugs (`strict`, `strictNullChecks`).
```ts
interface User { id: number; name: string; email?: string }            // interface: object shapes, extendable, declaration merging
type Id = number | string;                                              // type alias: unions, tuples, mapped/conditional types
type Result<T> = { ok: true; data: T } | { ok: false; error: string };  // discriminated union

function handle<T>(r: Result<T>) {                                      // narrowing on the discriminant
  if (r.ok) return r.data; else throw new Error(r.error);
}
function first<T>(xs: T[]): T | undefined { return xs[0]; }              // generics: reusable + type-safe
function len(x: string | string[]) { return typeof x === 'string' ? x.length : x.length; }  // typeof/instanceof/in/custom guards

type Partial<User>            // all optional          type Required<User>     // all required
type Pick<User,'id'|'name'>   // subset                type Omit<User,'email'> // remove
type Readonly<User>           // shallow readonly      type Record<string, User>
type ReturnType<typeof fn>    // extract types         const x = obj as const; // literal + readonly
enum vs union: prefer  type Role = 'admin' | 'user'  (no runtime object)
unknown (must narrow before use) vs any (turns checking off) vs never (unreachable)
```
**`interface` vs `type`:** both describe object shapes; `interface` supports `extends` + declaration merging and gives better error messages; `type` can express unions, intersections, tuples, primitives, mapped/conditional types. Pick `interface` for public object contracts, `type` for everything else — consistency matters more. TS is **structurally typed** (duck typing) — unlike Java's nominal typing: any object with the right shape is assignable. Generics are erased (no reified generics, like Java).

### 8.2 Tooling
- **Package managers:** npm (default), yarn, **pnpm** (content-addressable global store + symlinks: fast, disk-efficient, strict — no phantom dependencies). `package-lock.json`/`yarn.lock`/`pnpm-lock.yaml` pin the **exact resolved tree** → commit it; CI uses `npm ci` (clean, exact, fails if lock and package.json disagree) not `npm install`. `dependencies` (runtime) vs `devDependencies` (build/test) vs `peerDependencies` (host must provide, e.g. React for a component library). Compare Maven/Gradle: `pom.xml` ↔ `package.json`, local repo ↔ `node_modules`.
- **Semver** `MAJOR.MINOR.PATCH`: breaking / feature / fix. Ranges: `^1.2.3` = `>=1.2.3 <2.0.0` (minor+patch updates); `~1.2.3` = `>=1.2.3 <1.3.0`; `^0.2.3` = `>=0.2.3 <0.3.0` (0.x: minor is breaking!); exact `1.2.3`; `*`/`latest` = dangerous. Lockfile stops surprise upgrades; `npm audit`, Dependabot/Renovate keep them fresh.
- **Bundlers:** **webpack** — bundles everything, loaders + plugins, huge ecosystem, slow cold start. **Vite** — dev server serves **native ES modules on demand** (esbuild pre-bundles deps) → near-instant start & HMR; production build uses **Rollup**. Others: esbuild/SWC (Go/Rust speed), Turbopack, Rspack. Bundlers do: module resolution, transpile (Babel/SWC: JSX/TS/modern syntax), minify, tree-shake, code-split, hash filenames, env injection.
- **Quality:** ESLint (static analysis: `no-undef`, `eqeqeq`, hooks rules), Prettier (formatting), TypeScript (types), Husky + lint-staged (pre-commit).
- **Testing:** **Jest** (runner + assertions + mocks + jsdom, `describe/it/expect`, `jest.fn()`, `jest.mock`, snapshot) / **Vitest** (Jest-compatible API, Vite-native, faster); React Testing Library (test behaviour via roles/text, not implementation); Cypress/Playwright (E2E); MSW (mock network). Pyramid: many unit, some integration, few E2E. Async tests: `await`/return the promise, use fake timers (`jest.useFakeTimers()`) for debounce/retry.

---

## 9. Node.js essentials

**Model:** JS on V8 + **libuv** (event loop, async I/O). One main thread runs your JS. Network I/O uses OS async primitives (epoll/kqueue/IOCP) — no thread per connection. File system, `crypto.pbkdf2/scrypt/randomBytes`, `zlib`, `dns.lookup` are blocking syscalls, so libuv runs them on a **thread pool (default 4, `UV_THREADPOOL_SIZE` up to 1024)** and posts the callback back to the loop. Your JS code itself is never parallel.

```
 requests ─► [ event loop thread: JS handlers ] ──net I/O──► OS (epoll) ──► callback queued
                     │ fs/crypto/zlib/dns                    
                     ▼                                       
              libuv thread pool (4) ─► result callback ─► poll phase
 CPU-heavy JS (JSON.parse of 100MB, sync crypto, regex backtracking) BLOCKS every request.
```
**When Node shines:** I/O-bound, many concurrent connections (APIs, gateways/BFF, websockets, SSR, tooling). **Poor for:** heavy CPU (use `worker_threads`, child processes, or a Java/Go/Python service).

**Java threading comparison:**

| | Spring MVC + Tomcat | Node/Express | Spring WebFlux/Netty | Java 21 virtual threads |
|---|---|---|---|---|
| Model | thread per request (default 200 worker threads) | 1 event-loop thread | few event-loop threads | thread per request, cheap threads |
| Blocking call | ties up a thread (pool exhaustion) | freezes all requests | must not block (or offload) | parks the virtual thread, cheap |
| Programming style | imperative | callbacks/promises/async-await | Mono/Flux reactive | imperative (like sync code) |
| CPU parallelism | natural | worker_threads / cluster | schedulers | natural |
| Shared state | needs synchronisation | no data races in JS code (but async interleaving/races between awaits still exist: check-then-act across an `await`) | reactive ops | needs synchronisation |

**Scaling Node across cores:** `cluster` module (fork N worker processes sharing a port; each has its own memory — no shared state, use Redis) or run N containers behind a load balancer/PM2; `worker_threads` for CPU tasks inside one process (message passing; `SharedArrayBuffer` optional).

```js
const { Worker, isMainThread, parentPort, workerData } = require('worker_threads');
if (isMainThread) {
  const t = Date.now();
  const w = new Worker(__filename, { workerData: { n: 30 } });
  let ticks = 0; const iv = setInterval(() => ticks++, 5);            // main loop stays responsive
  w.on('message', v => { clearInterval(iv); console.log('worker result fib(30) =', v, '| main thread kept ticking:', ticks > 0); });
  w.on('error', console.error);
} else {
  const fib = n => n < 2 ? n : fib(n - 1) + fib(n - 2);
  parentPort.postMessage(fib(workerData.n));                            // data is structured-cloned (copied), not shared, unless SharedArrayBuffer
}
// ---- output (node v22.12, verified) ----
// worker result fib(30) = 832040 | main thread kept ticking: true
```
**Streams** process data in chunks (constant memory) with **backpressure**: `write()` returns `false` when the internal buffer exceeds `highWaterMark` → wait for `'drain'`. Types: Readable, Writable, Duplex, Transform. Always use `stream.pipeline` (handles error propagation and cleanup; plain `.pipe` doesn't). Typical: `fs.createReadStream` → `zlib.createGzip()` → `res` for big downloads; never `fs.readFile` a 2 GB file.

```js
const { Readable, Transform, Writable, pipeline } = require('stream');
const { pipeline: pipeP } = require('stream/promises');
(async () => {
  const src = Readable.from(['a', 'b', 'c', 'd']);
  const upper = new Transform({ transform(chunk, enc, cb) { cb(null, chunk.toString().toUpperCase()); } });
  const out = [];
  const sink = new Writable({ highWaterMark: 1, write(chunk, enc, cb) { setTimeout(() => { out.push(chunk.toString()); cb(); }, 5); } });
  await pipeP(src, upper, sink);                     // pipeline handles errors + backpressure + cleanup
  console.log('piped:', out.join(''));
  const big = new Writable({ highWaterMark: 4, write(c, e, cb) { setImmediate(cb); } });
  console.log('write returns:', big.write('ab'), big.write('cdef'), '(false = buffer full: stop writing until "drain")');
  for await (const chunk of Readable.from(['x', 'y'])) process.stdout.write('[' + chunk + ']');
  console.log();
})();
// ---- output (node v22.12, verified) ----
// piped: ABCD
// write returns: true false (false = buffer full: stop writing until "drain")
// [x][y]
```
**Express middleware chain:** a middleware is `(req, res, next)`; each either ends the response or calls `next()`; `next(err)` skips to error middleware `(err, req, res, next)` (**4 params** — arity identifies it). Order matters (body parser → auth → routes → 404 → error handler). Express 4 does **not** catch rejected promises from `async` handlers (Express 5 does) — wrap or use `express-async-errors`. Analogy: Servlet `Filter` chain / Spring `HandlerInterceptor`. A from-scratch model (note "logger end" prints after downstream — the onion/stack model, like `chain.doFilter` then post-processing):

```js
class App {
  #stack = [];
  use(fn) { this.#stack.push(fn); return this; }
  handle(req) {
    let i = 0;
    const next = err => {
      const fn = this.#stack[i++];
      if (!fn) return err ? console.log('default error handler:', err.message) : console.log('404 no middleware ended the response');
      try {
        if (err) { if (fn.length === 3) return fn(err, req, next); return next(err); }   // error middleware has 3 params (err, req, next); express uses 4 with res
        if (fn.length === 3) return next(err);
        return fn(req, next);
      } catch (e) { next(e); }
    };
    next();
  }
}
const app = new App();
app.use((req, next) => { console.log('1 logger start', req.url); next(); console.log('1 logger end (after downstream)'); });
app.use((req, next) => { req.user = 'asha'; next(); });
app.use((req, next) => { if (req.url === '/boom') throw new Error('kaboom'); console.log('3 handler for', req.user); });
app.use((err, req, next) => { console.log('4 error middleware:', err.message); next(); });
app.handle({ url: '/hello' }); console.log('---');
app.handle({ url: '/boom' });
// ---- output (node v22.12, verified) ----
// 1 logger start /hello
// 3 handler for asha
// 1 logger end (after downstream)
// ---
// 1 logger start /boom
// 4 error middleware: kaboom
// 404 no middleware ended the response
// 1 logger end (after downstream)
```
**Other Node facts:** `process.nextTick` vs `setImmediate` (see §4); `EventEmitter` is the base of streams/servers; `require` caches modules (singleton pattern); graceful shutdown = handle `SIGTERM`, stop accepting connections, `server.close()`, drain, exit (Kubernetes); never `process.exit()` mid-flight; use `AsyncLocalStorage` for request-scoped context (like Java ThreadLocal/MDC — which can't work across async callbacks without it); `NODE_ENV=production`; log JSON (pino); health/readiness endpoints; `--max-old-space-size` for heap limit.

---

## 10. Production stories (what interviewers love to hear)

1. **Java `Long` ID corrupted in the browser.** API returned `{"id": 1234567890123456789}`; JS `JSON.parse` rounded it to `1234567890123456800` → wrong record opened. *Fix:* serialise 64-bit IDs as strings (`@JsonSerialize(using = ToStringSerializer.class)`), or parse with BigInt-aware reviver.
2. **"CORS error" that was really a 401.** Spring Security rejected the `OPTIONS` preflight (no `Authorization`) before the CORS filter ran. *Fix:* `http.cors(withDefaults())` + `CorsConfigurationSource`; also ensure error responses carry CORS headers.
3. **`forEach(async)` lost writes.** Bulk save loop used `forEach(async …)`, then redirected; requests were cancelled by navigation. *Fix:* `await Promise.all(items.map(save))` (or a pool for rate limits).
4. **Stale closure.** `setInterval(() => setCount(count + 1), 1000)` stayed at 1 (captured `count` of first render). *Fix:* functional updater `setCount(c => c + 1)`, or a ref, and clear the interval in the effect cleanup.
5. **SPA memory leak.** Each route mount added `window.addEventListener('resize', …)` and a websocket without cleanup; heap snapshot comparison showed thousands of detached DOM nodes. *Fix:* cleanup in `useEffect` return / `AbortController` signal.
6. **JWT stolen through XSS** via a compromised third-party chat widget reading `localStorage`. *Fix:* HttpOnly refresh cookie + in-memory access token, strict CSP with nonces, SRI, remove the vendor script.
7. **Users stuck on the old version.** `index.html` served with `max-age=31536000`. *Fix:* `no-cache` on HTML, `immutable` on hashed assets, versioned service-worker.
8. **Wrong day for dates.** `new Date('2024-03-10')` is parsed as **UTC** midnight (date-only ISO), but `new Date('2024-03-10T00:00')` is local → in negative-offset zones the date shows as March 9. *Fix:* keep date-only values as strings, use `Intl.DateTimeFormat` with explicit `timeZone`, or Temporal/date-fns-tz; send instants as ISO-8601 with offset/`Instant`.
9. **`sort()` on prices.** `[100, 20, 3].sort()` → `[100, 20, 3]` (string comparison). *Fix:* `(a, b) => a - b`.
10. **Node service crashed after Node upgrade.** Unhandled promise rejection (previously a warning) terminates the process from Node 15. *Fix:* handle every promise; global `unhandledRejection` logging as a net; fix root causes.
11. **Event loop blocked.** A `JSON.parse` of a 60 MB payload + catastrophic regex (ReDoS) made health checks time out and Kubernetes restarted pods. *Fix:* stream parsing, size limits, safe regex (RE2), worker thread.
12. **Double payments.** Users double-clicked "Pay". *Fix:* disable button while pending, **idempotency key** header validated server-side, debounce; never blind-retry non-idempotent POSTs.
13. **5 MB bundle.** `import _ from 'lodash'` + moment locales. *Fix:* `lodash-es` named imports, `date-fns`, bundle analyzer, route-level code-splitting → LCP 4.8 s → 2.1 s.
14. **`localStorage` throws** in Safari private mode/when full/when JSON is corrupt → white screen. *Fix:* wrap in try/catch, fallback to memory, version stored schemas.

---

## 11. Interview questions (60) — graded, with follow-up chains and common wrong answers

**Legend:** [E] easy, [M] medium, [H] hard. *FU* = follow-up chain (each step is what a good interviewer asks next). *Wrong* = answers that lose points.

### 11.0 Output-prediction drills (predict first; outputs verified)
```js
var a1 = 1; function f1() { console.log('Q1', a1); var a1 = 2; } f1();
console.log('Q2', typeof typeof 1, typeof [] + typeof {}, typeof (() => {}));
let x3 = { a: 1 }; let y3 = x3; y3.a = 2; x3 = { a: 3 }; console.log('Q3', y3.a, x3.a);
const arr4 = [1, 2, 3]; arr4[6] = 7; console.log('Q4', arr4.length, arr4, 3 in arr4, arr4.map(x => x * 2));
function r5() { return
  { ok: true }; }
console.log('Q5', r5());
console.log('Q6', [] + null + 1, [1, 2] + [3], {} + 1, [] * 2, [3] * [4]);
console.log('Q7', parseInt(null, 36), parseInt('0x1f'), parseInt(0.0000005), Number(''), Number(' '), Number(null), Number(undefined), Number([]), Number(['7']));
(function () { console.log('Q8', arguments.length, typeof arguments, Array.isArray(arguments)); })(1, 2, 3);
console.log('Q9', new Array(3).map((_, i) => i), Array.from({ length: 3 }, (_, i) => i), [...Array(3).keys()], Array(3).fill().map((_, i) => i));
console.log('Q10', [3, 20, 100].sort(), [3, 20, 100].sort((a, b) => a - b), ['b', 'a', 'C'].sort(), ['é', 'e', 'z'].sort(), ['é', 'e', 'z'].sort((a, b) => a.localeCompare(b)));
function d11(a, b = a + 1, c = () => a + b) { a = 10; return [a, b, c()]; } console.log('Q11', d11(1));
function A12() {} A12.prototype.x = 1; const a12 = new A12(); A12.prototype = { x: 2 };
console.log('Q12', a12.x, new A12().x, a12 instanceof A12, new A12() instanceof A12);
console.log('Q13', '10' < '9', 10 < 9, '10' < 9, 'b' > 'a', 'B' > 'a', null < 1, undefined < 1, NaN < 1);
console.log('Q14', [1, 2, 3].includes(NaN), [NaN].includes(NaN), [NaN].indexOf(NaN), [0].includes(-0));
const o15 = { a: 1, get b() { return this.a + 1; }, c: () => typeof this };
console.log('Q15', o15.b, o15.c(), JSON.stringify(o15), Object.keys(o15));
console.log('Q16', JSON.stringify({ a: [undefined, function () {}, Symbol()], b: undefined, c: null, d: new Date(0), e: NaN }));
console.log('Q17', 1 + 2 + '3', '1' + 2 + 3, 1 + +'2', '5' - - '2', '5' + - '2', [] + [] === '', 0 == '', 0 == '0', '' == '0');
const s18 = 'hello'; s18[0] = 'J'; console.log('Q18', s18, s18.at(-1), 'abc'.split('').reverse().join(''), [...'a😀'].length, 'a😀'.length);
let i19 = 0; const r19 = [i19++, i19++, ++i19, i19--]; console.log('Q19', r19, i19);
console.log('Q20', (() => { try { return 'try'; } finally { console.log('finally runs first'); } })(), (() => { try { throw 1; } catch { return 'catch'; } finally { return 'finally overrides'; } })());
var v21 = 'global'; const o21 = { v21: 'obj', f() { return function () { return typeof this.v21; }; } }; console.log('Q21', o21.f()());
console.log('Q22', [1, 2, 3, 4].reduce((a, b) => a + b) / 4, [[1, 2], [3]].flat().length, Math.max(), Math.min(), Math.max([]), Math.max([1, 2]), Math.max(...[1, 2]));
label: for (let i = 0; i < 3; i++) { for (let j = 0; j < 3; j++) { if (j === 1) continue label; if (i === 2) break label; console.log('Q23', i, j); } }
const fns24 = {}; for (const k of ['a', 'b']) fns24[k] = () => k; console.log('Q24', fns24.a(), fns24.b());
console.log('Q25', Number.MAX_SAFE_INTEGER + 2, 0.1 * 3, 9999999999999999, 2 ** 31 | 0, ~~-5.7, -5.7 | 0, Math.trunc(-5.7), Math.round(-2.5), Math.round(2.5), 1 << 31, 5 >>> 1, -1 >>> 0);
console.log('Q26', [1, 2, 3].indexOf(2), [1, [2]].toString(), String([null, undefined, 1]), Array.isArray(Array.prototype), typeof Array.prototype);
const g27 = { valueOf: () => 1 }; console.log('Q27', g27 + g27, `${g27}`, g27 > 0, [g27] + '');
console.log('Q28', 'abc'.replace('b', '$&$&'), 'a-b-c'.replace('-', '+'), 'a-b-c'.replaceAll('-', '+'), 'a-b-c'.split('-', 2), ' x '.trim().length);
console.log('Q29', new Date(2024, 0, 31).getMonth(), new Date('2024-03-10').getUTCDate(), new Date(NaN).getTime(), typeof Date.now());
console.log('Q30', Object.entries({ b: 1, a: 2 }).map(([k, v]) => k + v).join(), Object.fromEntries([['x', 1], ['y', 2]]), Object.assign({}, 'ab', [9]), { ...'hi' });
// ---- output (node v22.12, verified) ----
// Q1 undefined
// Q2 string objectobject function
// Q3 2 3
// Q4 7 [ 1, 2, 3, <3 empty items>, 7 ] false [ 2, 4, 6, <3 empty items>, 14 ]
// Q5 undefined
// Q6 null1 1,23 [object Object]1 0 12
// Q7 1112745 31 5 0 0 0 NaN 0 7
// Q8 3 object false
// Q9 [ <3 empty items> ] [ 0, 1, 2 ] [ 0, 1, 2 ] [ 0, 1, 2 ]
// Q10 [ 100, 20, 3 ] [ 3, 20, 100 ] [ 'C', 'a', 'b' ] [ 'e', 'z', 'é' ] [ 'e', 'é', 'z' ]
// Q11 [ 10, 2, 12 ]
// Q12 1 2 false true
// Q13 true false false true false true false false
// Q14 false true -1 true
// Q15 2 object {"a":1,"b":2} [ 'a', 'b', 'c' ]
// Q16 {"a":[null,null,null],"c":null,"d":"1970-01-01T00:00:00.000Z","e":null}
// Q17 33 123 3 7 5-2 true true true false
// Q18 hello o cba 2 3
// Q19 [ 0, 1, 3, 3 ] 2
// finally runs first
// Q20 try finally overrides
// Q21 undefined
// Q22 2.5 3 -Infinity Infinity 0 NaN 2
// Q23 0 0
// Q23 1 0
// Q24 a b
// Q25 9007199254740992 0.30000000000000004 10000000000000000 -2147483648 -5 -5 -5 -2 3 -2147483648 2 4294967295
// Q26 1 1,2 ,,1 true object
// Q27 2 [object Object] true [object Object]
// Q28 abbc a+b-c a+b+c [ 'a', 'b' ] 1
// Q29 0 10 NaN number
// Q30 b1,a2 { x: 1, y: 2 } { '0': 9, '1': 'b' } { '0': 'h', '1': 'i' }
```
**Why (the ones that trip people):** Q1 the inner `var a1` is hoisted and shadows the outer → `undefined`. Q5 ASI inserts `;` after `return` → `undefined`. Q6 `[] + null + 1` = `"" + "null" + 1`; `{}` after `+` is an expression, not a block. Q7 `parseInt(null, 36)` parses the string `"null"` in base 36; `parseInt(0.0000005)` stringifies to `"5e-7"` → 5. Q9 `new Array(3)` has holes, `map` skips holes. Q10 default `sort` compares UTF-16 strings, uppercase before lowercase, `é` after `z`; use `localeCompare`. Q11 body assignment `a = 10` updates the same binding the closure `c` sees (no separate body scope, since there's no `var a`). Q12 replacing `prototype` does not affect existing instances (their `[[Prototype]]` link is fixed at creation). Q13 both strings → lexicographic; one number → numeric. `null < 1` true (null→0) but `undefined < 1` false (NaN). Q14 `includes` uses SameValueZero (finds NaN, treats -0 = 0), `indexOf` uses `===`. Q17 `'5' - - '2'` = 7, `'5' + - '2'` = `"5-2"`. Q18 strings are immutable (assignment silently ignored in sloppy mode) and `length` counts UTF-16 code units (😀 = 2). Q19 postfix returns the old value. Q20 `finally` runs before the value is delivered, and a `return` in `finally` overrides the earlier `return`/`throw`. Q21 detached function called plainly → `this` = `globalThis`, and top-level `var` in a **CommonJS module is not a global property** (`'undefined'` here; in a browser `<script>` it would be `'string'`). Q22 `Math.max()` = -Infinity; `Math.max([])` = 0 (`[]→""→0`), `Math.max([1,2])` = NaN. Q25 `Math.round(-2.5)` = -2 (rounds toward +∞); `|0` truncates toward zero; `>>> 0` gives unsigned. Q28 `$&` inserts the match in a replacement string. Q29 `new Date(2024, 0, 31).getMonth()` = 0 (months are 0-based); `'2024-03-10'` parses as UTC.

### 11.1 Language fundamentals

**1. [E] `var` vs `let` vs `const`?** `var`: function-scoped, hoisted & initialised to `undefined`, re-declarable, becomes a property of `globalThis` in scripts. `let`/`const`: block-scoped, hoisted but in the TDZ, no re-declaration; `const` forbids rebinding, not mutation. Default to `const`, `let` when reassigning, never `var`. *FU:* Is `let` hoisted? (yes—TDZ proves it) → TDZ + `typeof`? (throws) → `const` object mutation? (allowed; `freeze` to prevent) → global `var` vs `let` on `window`? *Wrong:* "let is not hoisted".

**2. [E] `==` vs `===`; when is `==` acceptable?** `===` no coercion; `==` applies the Abstract Equality algorithm (§3.6). Use `===` always; the accepted exception is `x == null` (null or undefined). *FU:* `[] == false`? (true) → `NaN === NaN`? (false; `Number.isNaN`, `Object.is`) → `null == 0` vs `null >= 0`? (false / true).

**3. [E] What is hoisting?** Declarations are processed when the execution context is created: functions fully, `var` as `undefined`, `let/const/class` uninitialised (TDZ). Only the declaration is hoisted, not the assignment. *FU:* function expression vs declaration hoisting → predict Q1 → TDZ example with default params `function f(a = b, b)` (ReferenceError). *Wrong:* "code is moved to the top".

**4. [M] `null` vs `undefined`; why `typeof null === 'object'`?** `undefined` = uninitialised/missing; `null` = intentional absence. `typeof null` is a 1995 bug (type tag 0 = object) kept for compatibility. Test null with `=== null`. `JSON.stringify` drops `undefined` properties but keeps `null`. *FU:* default parameters trigger on which? (only `undefined`) → optional chaining short-circuits on? (both) → `??` vs `||`.

**5. [E] Primitives vs objects; pass by value or reference?** Primitives are immutable, copied by value; objects are handled through references. Functions receive a **copy of the reference** ("call by sharing"): mutating a param's properties is visible outside, reassigning the param is not (Q3). Java is the same (always pass-by-value of references). *FU:* how to avoid mutating arguments? (copy, `structuredClone`, immutability) → `const` array push?

**6. [M] Shallow vs deep copy — options?** Shallow: spread, `Object.assign`, `slice`. Deep: `structuredClone` (Date/Map/Set/cycles; not functions/DOM/class prototypes), JSON round trip (loses undefined, functions, Date→string, NaN→null, no cycles), hand-written recursive (§5.4), `lodash.cloneDeep`. Better: avoid deep copies via immutable updates of only the changed path. *FU:* implement deepClone with cycles → why WeakMap for `seen`? *Wrong:* "spread is a deep copy".

**7. [E] What is a closure and give real uses?** Function + persistent reference to its lexical environment; keeps variables alive after the outer function returns. Uses: private state, factories, memoize/debounce/once, currying, module pattern, callbacks capturing context, React hooks. *FU:* memory implications → how does a closure leak? → stale closure bug in `setInterval`/`useEffect` and the fix (functional update, refs, deps) → Java lambda difference (effectively final).

**8. [M] Predict the `var` loop with `setTimeout`; three fixes.** (§2.4) `3 3 3` vs `0 1 2`. Fixes: `let`; IIFE; `setTimeout(fn, 0, i)` third argument / `bind`. *FU:* why does `let` create a per-iteration binding? → what if the loop body mutates `i` in a closure? *Wrong:* "because of async" (the reason is one shared binding).

**9. [M] Explain `this` binding rules and priority.** new > explicit (bind/call/apply) > implicit (obj.f()) > default (undefined strict/global sloppy); arrows are lexical. *FU:* `setTimeout(obj.method)` → what is `this` inside a DOM event handler? (`currentTarget`, for non-arrow) → `this` in a class static method → `this` in a getter (the receiver).

**10. [M] Arrow vs regular functions?** Arrow: lexical `this`, no own `arguments`, not constructible (`new` throws), no `prototype`, cannot be generators, can't be rebound, implicit return. Don't use arrows as object methods that need `this` or where dynamic `this` is wanted (jQuery handlers, Mocha). *FU:* `arguments` alternative → rest params. *Wrong:* "arrow functions are just shorter syntax".

**11. [M] `call` vs `apply` vs `bind`; implement `bind`.** `call(ctx, a, b)` invokes now; `apply(ctx, [a, b])`; `bind` returns a new function with fixed `this` (+ partial args), invoked later. §2.5 implements all three with `new` support. *FU:* can you `bind` an already bound function? (`this` is fixed to first bind) → `bind` + `new`? (`new` wins) → `Math.max.apply(null, arr)` today? (spread).

**12. [M] Prototype chain; `__proto__` vs `prototype`; is JS class-based?** Objects link to a prototype object; lookups walk the chain; `prototype` is a property of constructor functions used as the `[[Prototype]]` of instances; `__proto__` is the legacy accessor for `[[Prototype]]` (use `Object.getPrototypeOf`). `class` is syntactic sugar over prototypes (plus semantics: TDZ, strict, non-callable, private fields). *FU:* `Object.create(null)` uses? (dictionaries without prototype pollution) → `hasOwnProperty` vs `in` → `instanceof` mechanics (walks chain checking `C.prototype`; customisable by `Symbol.hasInstance`) → prototype pollution attack.

**13. [M] What does `new` do? What if the constructor returns something?** Four steps (§3.1); a returned object overrides `this`, a primitive is ignored. `new.target` detects calls without `new`. *FU:* write `myNew` → what if you forget `new` on a function constructor (sloppy: pollutes global!, strict: TypeError) → why can't arrow functions be constructors?

**14. [M] Class vs constructor function; private fields.** Class: strict mode, non-hoisted (TDZ), methods non-enumerable, can't call without `new`, `super`, static blocks, `#private` fields enforced by the engine (vs `_name` convention, vs closures/WeakMap). Inheritance = `extends` sets up prototype chains of both instance and constructor. *FU:* `super()` before `this`? → static vs instance → getters/setters → mixins.

**15. [M] `Object.freeze` vs `seal` vs `preventExtensions` vs `const`.** §3.2 table. Freeze is shallow → `deepFreeze`. `const` = binding immutability. Strict mode makes violations throw. *FU:* how does immer/immutable updates avoid full copies? (structural sharing).

**16. [H] Property descriptors & accessors; where are they used in frameworks?** `writable/enumerable/configurable/get/set`; `Object.defineProperty` powered Vue 2 reactivity (getters/setters per property; can't detect property add/delete), replaced by `Proxy` in Vue 3 (intercepts everything). `Proxy`/`Reflect` allow validation, logging, revocable proxies; costs performance. *FU:* enumerable and `for-in` → `Object.keys` vs `Reflect.ownKeys` (includes symbols, non-enumerables).

**17. [M] Symbols and iterators — make an object iterable.** Implement `[Symbol.iterator]()` returning `{ next() }` or use a generator method `*[Symbol.iterator]() { ... }`. Consumers: for-of, spread, destructuring, `Array.from`, `Promise.all`, `new Map/Set`. *FU:* async iterators → why not `for-in` on arrays (inherited props, string indexes, order) → `Object.entries` vs `for-in`.

**18. [M] Generators — what for?** Lazy sequences, pausing/resuming, two-way communication (`next(v)`), custom iteration, coroutines (redux-saga, `co`; async/await is built on the same idea — §4.1 desugar). *FU:* what does `return()` do in `for-of` break → `yield*` delegation → infinite generators and `take`.

**19. [M] `Map`/`Set` vs object/array; `WeakMap` use cases.** §3.5. WeakMap: keys held weakly, not enumerable — per-object private data, DOM-node metadata caches. *FU:* why can't WeakMap have `size`/iteration? (GC non-determinism) → `WeakRef`/`FinalizationRegistry` caveats → dedupe an array with Set (O(n)) → object keys ordering surprise (integer keys first).

**20. [M] `map`/`filter`/`reduce`/`forEach`/`find`/`some`/`every`; default `sort`.** `map` transforms (new array), `filter` subsets, `reduce` folds (always give the initial value), `forEach` side effects (can't break/await), `some/every` short-circuit. `sort` is **in place**, default string order, stable since ES2019; `toSorted` (ES2023) copies. *FU:* `[1,2,3].map(parseInt)` (`[1,NaN,NaN]`) → mutating vs non-mutating array methods list → implement `reduce` (§5.5) → Java Stream comparison (streams are lazy; JS array methods are eager and allocate intermediates).

**21. [M] Spread vs rest; destructuring defaults; `??` vs `||`; optional chaining.** Same `...`: spread expands (call/array/object literal), rest collects (params/destructuring). Defaults only for `undefined`. `?.` short-circuits on null/undefined (also for calls `f?.()` and index `a?.[0]`). *FU:* nested destructuring with defaults (`const { a: { b = 1 } = {} } = obj`) → object spread order and overriding.

### 11.2 Coercion and numbers

**22. [M] Predict `[] + {}`, `{} + []`, `[] == ![]`, `1 < 2 < 3`, `'b'+'a'+ +'a'+'a'`.** Verified in §3.6 output: `"[object Object]"`, `"[object Object]"` (as an expression; as a statement `{}` is an empty block → `+[]` = 0), true, true, `"baNaNa"`. Explain via ToPrimitive: arrays → `""`/joined strings, objects → `"[object Object]"`. *FU:* `valueOf` vs `toString` order (number hint: valueOf first; string hint: toString first; default = number except Date/Symbol) → `Symbol.toPrimitive`. *Wrong:* "JS is random" — it is fully specified.

**23. [H] Walk through `[] == ![]` step by step.** `![]` → `false` (object truthy); `[] == false` → boolean→number: `[] == 0` → object→primitive: `""` → `"" == 0` → `0 == 0` → true (§3.6). *FU:* `null == false`? (false: null equals only undefined) → `NaN == NaN`.

**24. [M] Why is `0.1 + 0.2 !== 0.3`; how do you handle money; when BigInt?** IEEE-754 binary doubles can't represent 0.1 exactly. Compare with epsilon; money in integer minor units (paise) or decimal.js/Big; BigInt for integers > 2^53 (IDs) — not mixable with Number, not JSON-serialisable. Java parallel: `BigDecimal`. *FU:* `toFixed` returns a string and `1.005.toFixed(2)` = "1.00" (§3.7) → `Number.MAX_SAFE_INTEGER` → `Intl.NumberFormat` for currency (Indian grouping `en-IN`).

### 11.3 Async — event loop and promises

**25. [E] Is JavaScript single-threaded? How does async work then?** One thread runs JS; the host does I/O/timers concurrently (browser Web APIs / libuv), then queues callbacks; the event loop pushes them onto the stack when empty. *FU:* what blocks the loop? → how do Web Workers fit? → explain to a Java dev who asks "where is the thread pool?".

**26. [M] Explain the event loop, macrotasks vs microtasks, and rendering.** §4.1 diagram: one task → all microtasks → (maybe) render (rAF → style → layout → paint) → next task. Microtasks: promise reactions, `queueMicrotask`, MutationObserver; macrotasks: timers, I/O, UI events, `postMessage`. *FU:* can microtasks starve rendering? (yes) → where does `requestAnimationFrame` run? → why `setTimeout(0)` ≥ 1–4 ms? → `requestIdleCallback`. *Wrong:* "promises are async because they run on another thread".

**27. [H] Predict Puzzle 3 (`async1/async2`) and explain why the answer changed in 2019.** Output in §4.2: `async1 end` before `promise2`. Old spec wrapped the awaited native promise in an extra promise (3 ticks); the fast path (V8 7.2) uses 1 tick. *FU:* what if `async2` were `return new Promise(...)` pending? → `await` a thenable? (Puzzle 7 behaviour) → Node vs browser difference? (none for this).

**28. [H] Why does `return promise` in `.then` (or resolving with a promise) take 2 extra ticks?** The resolve function sees a thenable and queues `NewPromiseResolveThenableJob`, which calls `promise.then(resolveDerived)`, whose reaction queues another job → 2 extra ticks (Puzzles 5 and 16 in §4.2). Practical rule: never depend on cross-chain ordering; sequence explicitly with `await`.

**29. [M] `nextTick` vs `setImmediate` vs `setTimeout(0)` vs promises in Node.** Order: sync → `nextTick` queue → promise microtasks → timers phase → poll → `setImmediate` (check). From the main module, timeout vs immediate is non-deterministic; inside an I/O callback immediate always first. `nextTick` recursion starves I/O. ESM main: promise before nextTick (Puzzle 14 output). *FU:* which is the fairer yielding primitive? (`setImmediate`) → browsers have no `setImmediate` (use `MessageChannel`/`scheduler.postTask`).

**30. [E] Promise states and chaining.** pending → fulfilled/rejected; `then` returns a new promise; returned value/promise flows on, thrown error becomes rejection; `catch` recovers; `finally` passes through. *FU:* what does `.then(a).catch(b)` vs `.then(a, b)` do? → forgetting `return` → multiple `.then` on the same promise (fan-out; each gets the value).

**31. [M] `Promise.all` vs `allSettled` vs `race` vs `any`.** Table §4.3; `all` fail-fast without cancelling others; `any` → `AggregateError`. *FU:* implement `Promise.all` (§5.7) → timeout via `race` → limit concurrency (§4.4) → which one for "load dashboard widgets, show whatever works"? (`allSettled`).

**32. [M] Promise anti-patterns.** Explicit constructor wrapping, missing `return`, nested pyramids, swallowing errors, `async` without `await`, `.then` inside `async` mixing, unhandled rejections, `await` in loops for independent work. *Wrong:* `new Promise` inside `async` function returns.

**33. [M] async/await error handling and `return await`.** `try/catch` around `await`; without `await`, rejection escapes (Puzzle 10). `return await` inside `try` to catch; otherwise plain `return` is fine (stack traces are slightly better with `await`). Centralise with wrapper `to(promise)` returning `[err, data]`, or a top-level handler. *FU:* error in a `.then` callback vs `await` → `Promise.all` with try/catch → error `cause`.

**34. [M] `forEach(async)` trap; sequential vs parallel; bounded concurrency.** §4.2 puzzle + §4.4 pool. *FU:* order of results in `Promise.all(map)` (input order, not completion) → how to stream results as they complete? (`for await` over a merged async iterator) → rate-limit 100 rps.

**35. [H] Implement a mini Promise / what does Promises/A+ require?** State machine, `then` returns new promise, handlers async (microtask), handler results resolved via the "resolution procedure" (thenables adopted, cycle → TypeError), exactly-once settle, pass-through of missing handlers (§5.1). *FU:* `finally` semantics → static `resolve/all` → `queueMicrotask` vs `setTimeout` for scheduling (spec says any async mechanism; microtask matches native).

**36. [M] Debounce vs throttle — when and how?** §5.2. Debounce: after inactivity (search-as-you-type, validation, autosave, resize end). Throttle: max rate (scroll, drag, telemetry). Options: leading/trailing, cancel/flush; `requestAnimationFrame` throttle for visuals. *FU:* how to handle `this`/args → React: debounce inside `useMemo/useRef` (a new debounced fn per render breaks it) → `useDeferredValue`.

**37. [M] Unhandled promise rejections — what happens?** Browser: `unhandledrejection` event; Node ≥ 15: crash exit 1 unless handled (§4.3). Always attach handlers or `await` in try; global listener for logging. *FU:* `rejectionHandled` (late handler) → `process.on('uncaughtException')` — log and exit.

**38. [H] How do you cancel async work?** Promises can't be cancelled; you ignore results or cancel the underlying operation: `AbortController.signal` to `fetch`, listeners, streams, timers (`AbortSignal.timeout`). Pattern for typeahead: abort previous request on each keystroke, or ignore stale responses via a request id, otherwise **out-of-order responses** show wrong results. *FU:* `fetch` abort error name (`AbortError`; timeout → `TimeoutError`) → React `useEffect` cleanup aborting.

**39. [M] Implement retry with exponential backoff.** §5.9: attempts, base·2^n capped, jitter (avoid thundering herd), retry only on retryable errors (network, 5xx, 429 with `Retry-After`), idempotency; circuit breaker for persistent failure. *FU:* Java analogues (Resilience4j / Spring Retry) → full jitter vs equal jitter.

**40. [H] Design: typeahead search box.** Debounce ~250 ms; min 2 chars; `AbortController` to cancel stale request; cache by query (LRU); show loading/empty/error states; keyboard navigation + ARIA `combobox`; escape/`encodeURIComponent`; guard against out-of-order responses; server-side rate limits; highlight matches with safe escaping (no `innerHTML`). *FU:* how would you test it? (fake timers + MSW).

### 11.4 Modules, browser, security, performance

**41. [M] CommonJS vs ES modules; live bindings; circular dependencies.** §6.1 table + demos. *FU:* why ESM enables tree-shaking (static imports/exports) → `import()` dynamic for code splitting → `require` in ESM? (`createRequire`) → `"type": "module"`, `.mjs/.cjs` → dual-package hazard.

**42. [M] Event bubbling, capturing, delegation, `preventDefault` vs `stopPropagation`.** §7.1. *FU:* `target` vs `currentTarget` → non-bubbling events → passive listeners → delegating events for dynamically added rows → memory: remove listeners.

**43. [M] `fetch` doesn't reject on 404 — how do you handle errors and timeouts?** Check `res.ok`; wrap in helper that throws; `AbortSignal.timeout`; parse errors; retry only idempotent; distinguish network vs HTTP vs parse errors (§7.2 verified). *FU:* fetch vs axios → streaming a response body (`res.body.getReader()`) → upload progress (XHR/`fetch` duplex).

**44. [H] Explain CORS: preflight, credentials, and how to fix it.** §7.3. Say: SOP blocks *reads*; server opts in with `Access-Control-Allow-*`; simple vs preflighted; credentials need explicit origin + `Allow-Credentials: true`; fix on the server/gateway (or same-origin proxy); Spring Security ordering; `Vary: Origin`. *FU:* why does `no-cors` not help? → does CORS protect the API from attackers? (no—non-browser clients) → CSRF vs CORS relationship (simple POST can still be sent) → `Access-Control-Max-Age`. *Wrong:* "add `Access-Control-Allow-Origin` in the React app/`mode: 'no-cors'`".

**45. [M] Cookies vs localStorage vs sessionStorage vs IndexedDB — and where do you store a JWT?** §7.4 table & recommendation (memory + HttpOnly refresh cookie / BFF). *FU:* `SameSite` values → `HttpOnly` vs `Secure` → third-party cookie phase-out → what does a logout do for stateless JWT? (revocation list/short TTL).

**46. [M] XSS vs CSRF; how do you prevent each; what is CSP?** §7.4–7.5. *FU:* is React XSS-proof? (escapes by default; `dangerouslySetInnerHTML`, `href="javascript:"`, third-party scripts, SSR injection) → how does `SameSite=Lax` change CSRF? → Spring's CSRF token flow.

**47. [M] `defer` vs `async`; what is the critical rendering path?** §7.6 table; CSS blocks render, JS blocks parser; put `defer` scripts in head, inline critical CSS, preload key resources. *FU:* `DOMContentLoaded` vs `load` → where does `type="module"` fit?

**48. [M] Core Web Vitals and how to improve each.** LCP ≤ 2.5 s (preload hero, optimise/serve AVIF via CDN, SSR/streaming, reduce render-blocking), INP ≤ 200 ms (break long tasks, defer non-critical JS, avoid layout thrash, virtualise, workers), CLS ≤ 0.1 (dimensions on media, reserve space, `font-display`, no top insertions). *FU:* lab vs field data → what replaced FID? (INP) → how do you monitor in prod? (RUM, `web-vitals`).

**49. [M] Reflow vs repaint; how to animate smoothly?** §7.6; animate `transform`/`opacity` (compositor), avoid layout properties, `will-change` sparingly, batch DOM reads/writes, `requestAnimationFrame`. (Details in CSS.md.) *FU:* layout thrashing example.

**50. [M] Common causes of memory leaks in SPAs; how do you find them?** §6.4 table + DevTools heap snapshot comparison. *FU:* difference between a leak and high memory use → `WeakMap` fix example → timers/`useEffect` cleanup.

**51. [M] Web Workers vs Service Workers; what is a PWA?** §7.7. *FU:* what can't a worker access (DOM) → transferables → SW update problem → caching strategies.

**52. [M] TypeScript: `interface` vs `type`, generics, unions & narrowing, utility types, why use it?** §8.1. *FU:* `unknown` vs `any` → structural vs nominal typing → does TS validate API responses at runtime? (no → Zod) → `strict` flags worth enabling → `as` casts are lies.

**53. [M] semver, lockfile, `npm ci`; dependencies types; pnpm.** §8.2. *FU:* what does `^0.2.3` allow? (`<0.3.0`) → how do you handle a vulnerable transitive dependency? (`overrides`/`resolutions`, audit, upgrade) → monorepo workspaces.

**54. [M] Vite vs webpack; what do bundlers do?** §8.2. *FU:* HMR mechanism → why dev server is fast (no bundling, native ESM, esbuild) → code splitting/tree-shaking prerequisites → source maps in production (upload to Sentry, don't expose).

### 11.5 Node.js and system-level

**55. [H] How can Node be "single-threaded" yet handle thousands of connections? What is the libuv thread pool?** §9: non-blocking sockets via OS event notification; blocking-by-nature tasks (fs, crypto, zlib, dns.lookup) go to a pool of 4 threads (tunable). JS itself never runs in parallel; CPU-bound code blocks everything. *FU:* how to use multiple cores? (cluster / worker_threads / containers) → how do you detect event-loop lag? (`perf_hooks.monitorEventLoopDelay`) → what happens with a slow DB query? (async → fine; sync loop → blocks).

**56. [H] Node vs Java (Spring MVC / WebFlux / virtual threads) — when would you pick which?** §9 table. Node: I/O-heavy, real-time, BFF, shared TS types with front-end. Java: CPU-bound, rich enterprise ecosystem, transactional systems, strong typing, mature JVM tooling/observability. Virtual threads (Java 21) give thread-per-request simplicity with Node-like scalability for blocking I/O. *FU:* race conditions in Node? (yes across awaits — check-then-act) → shared state across cluster workers (no; Redis).

**57. [M] Express middleware and error handling.** §9: chain via `next()`, order, 4-arg error handler, async handler pitfall in Express 4, `helmet`, `cors`, rate-limit, body-size limits. *FU:* how does it compare to Servlet filters? → how would you implement `use()` and `next()`? (§9 code) → structured logging and request IDs (`AsyncLocalStorage`).

**58. [M] Streams and backpressure — why?** Constant memory, start sending before all data is read; `pipeline` for errors; `write()` false → wait for `drain`; async iteration `for await`. *FU:* what goes wrong with `readFile` + `res.send` for large files (heap OOM) → `highWaterMark` → object mode.

**59. [H] Explain stale closures and how you would debug a "state doesn't update in my interval/handler" bug.** A closure captures the *variable of that render/invocation*; a timer created once keeps seeing old values. Fix by functional updates, refs (`useRef` latest value), re-subscribing with correct deps, or reading from mutable store. Debug: log values inside callbacks, ESLint `react-hooks/exhaustive-deps`. *FU:* why do `useCallback` dependencies matter? → is `var` in a loop the same problem? (shared binding vs stale snapshot — opposite mechanisms).

**60. [H] You get "Maximum call stack exceeded" / a frozen tab / a Node service with growing RSS — what do you check?** Recursion depth (convert to iteration/trampoline); long tasks blocking the loop (Performance panel flame chart, `--cpu-prof`); leak (heap snapshots, `--heapsnapshot-signal`, count listeners/timers, unbounded caches); inspect `process.memoryUsage()`, event-loop delay, open handles (`why-is-node-running`). *FU:* what is the difference between RSS and heapUsed? → how would you alert on it?

---

## 12. One-page cheat sheet

```
EVENT LOOP   sync → (nextTick) → ALL microtasks (then/await/queueMicrotask) → ONE macrotask → microtasks → render(rAF→style→layout→paint) → ...
             then-callback returning a promise = +2 ticks; await native promise = 1 tick; timers ≥1ms; Node: timers→poll→check(setImmediate)
HOISTING     function ✔ full | var → undefined | let/const/class → TDZ ReferenceError | fn-expression var → TypeError
THIS         new > call/apply/bind > obj.f() > plain (undefined strict) ; arrow = lexical, can't be rebound/new
PROTOTYPE    obj.[[Proto]] → F.prototype → Object.prototype → null ; class = sugar ; new = create+link, run, return object ?? this
EQUALITY     null==undefined only ; NaN!=NaN (Object.is) ; [] == false ; use === ; x == null idiom ; typeof null 'object'
NUMBERS      0.1+0.2 ≠ 0.3 ; ints exact ≤ 2^53-1 ; money in minor units ; BigInt can't mix ; Java long IDs → send as strings
FALSY        false 0 -0 0n "" null undefined NaN
ARRAYS       sort() = string order & in place ; toSorted/with/at/findLast (ES2023) ; map skips holes ; parseInt in map bug
PROMISES     all: fail-fast | allSettled: never rejects | race: first settle | any: first fulfil (AggregateError)
             then returns NEW promise ; catch recovers ; finally passes value ; unhandled rejection kills Node ≥15
ASYNC        await in loop = sequential ; forEach(async) doesn't wait ; return await inside try ; Promise.all + pool for concurrency
FETCH        rejects only on network/abort ; check res.ok ; AbortSignal.timeout ; credentials: 'include' for cookies
CORS         SOP blocks reads ; preflight OPTIONS for non-simple ; server sends ACAO/ACAM/ACAH/Max-Age ; credentials → explicit origin + ACAC:true + Vary
COOKIES      HttpOnly (no JS) Secure SameSite=Lax|Strict|None ; JWT: memory access token + HttpOnly refresh cookie
XSS/CSRF     XSS: encode/sanitise/CSP/no innerHTML ; CSRF: SameSite + token + Origin check
PERF         defer > async ; CSS blocks render ; animate transform/opacity ; LCP 2.5s INP 200ms CLS 0.1 ; hashed assets immutable, html no-cache
MEMORY       leaks: detached DOM, listeners, timers, closures, unbounded caches, globals ; heap snapshot diff
MODULES      ESM static live-binding async TLA | CJS sync copy dynamic ; cycles: CJS partial, ESM TDZ
NODE         1 JS thread + libuv pool(4) for fs/crypto/zlib/dns ; CPU → worker_threads ; cluster = processes ; pipeline() for streams ; graceful SIGTERM
TS           types erased ; interface(extend/merge) vs type(union) ; unknown>any ; Partial/Pick/Omit/Record/ReturnType ; validate at runtime (Zod)
TOOLS        npm ci in CI ; ^1.2.3 = <2.0.0 ; ^0.2.3 = <0.3.0 ; commit lockfile ; Vite=native ESM dev + Rollup build ; Jest/Vitest
HANDWRITTEN  call/apply/bind, new, debounce, throttle, curry, compose, memoize, deepClone(WeakMap), Promise, all/any/race, LRU(Map), emitter, retry+jitter, flatten
```
