# Session 9 — JavaScript Core (Basic level, for a Python developer)

**Time:** Concepts 15 · Quiz 10 · Interview 15
Run examples in Node (`node file.js`, or `node` REPL) or the browser console. Each concept notes the **Python contrast**.

---

## Part 1 — Concepts

### 9.1 Variables, types, equality
```js
const PI = 3.14;        // block-scoped, cannot be REASSIGNED (object contents can still mutate)
let count = 0;          // block-scoped, reassignable
var old = 1;            // function-scoped, hoisted → avoid
```
- Primitives: `number` (one IEEE-754 double type; no int/float split), `bigint` (`10n`), `string`, `boolean`, `undefined`, `null`, `symbol`. Everything else is an object (arrays, functions, dates…).
- `typeof null === "object"` (historic bug); use `x === null`. `Array.isArray(x)` for arrays.
- `NaN !== NaN`; use `Number.isNaN(x)`. `0.1 + 0.2 !== 0.3` (same as Python).
- **Equality**: always `===`/`!==` (strict). `==` coerces: `"1" == 1`, `null == undefined`, `[] == false` → true.
- **Falsy**: `false, 0, -0, 0n, "", null, undefined, NaN`. Note `[]` and `{}` are **truthy** (unlike Python!).
- `null` (explicit empty) vs `undefined` (not assigned). Optional chaining/nullish: `user?.address?.city ?? "unknown"` (`??` only for null/undefined; `||` replaces any falsy including `0` and `""`).
- Template strings: `` `Hello ${name}` `` (Python f-strings).

### 9.2 Functions
```js
function add(a, b = 0) { return a + b; }           // declaration (hoisted)
const mul = (a, b) => a * b;                        // arrow function
const sum = (...nums) => nums.reduce((x, y) => x + y, 0);   // rest params (*args)
const { name, age = 30, ...rest } = person;         // destructuring (+ defaults)
const [first, , third] = arr;
const merged = { ...a, ...b };  const copy = [...arr, 4];    // spread
```
- Functions are first-class. Missing args are `undefined`; extra args ignored (no TypeError like Python).
- Arrow functions: concise, no own `this`/`arguments`, can't be constructors.
- Python contrast: no keyword arguments — emulate with an options object: `connect({ host, port = 5432 })`.

### 9.3 Scope, hoisting, closures
- `let/const` live in the **temporal dead zone** until declared (ReferenceError if used early); `var` declarations hoist as `undefined`; function declarations hoist fully.
- Closures work like Python's, but with block scoping the classic loop bug differs:
  ```js
  for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));   // 3 3 3
  for (let i = 0; i < 3; i++) setTimeout(() => console.log(i));   // 0 1 2 (new binding per iteration)
  ```
- Modules (ES modules) have their own scope: `export const x`, `export default fn`, `import { x } from "./m.js"`. CommonJS (`require`/`module.exports`) is the older Node style.

### 9.4 Objects, arrays, and prototypes
```js
const user = { name: "Ana", greet() { return `Hi ${this.name}`; } };
Object.keys(user); Object.entries(user); Object.assign({}, user);
structuredClone(obj)             // deep copy (like copy.deepcopy)
```
- Objects are like dicts with string/symbol keys (use `Map` for arbitrary keys & insertion-order/size; `Set` for uniques).
- Arrays: `push/pop/shift/unshift/splice/slice`, and the functional trio:
  ```js
  const evens = nums.filter(n => n % 2 === 0);
  const squares = nums.map(n => n ** 2);
  const total = nums.reduce((acc, n) => acc + n, 0);
  nums.some(...); nums.every(...); nums.find(...); nums.includes(3); nums.flatMap(...)
  [...nums].sort((a, b) => a - b);   // default sort is LEXICOGRAPHIC: [10, 9, 1].sort() → [1, 10, 9]; sort mutates!
  ```
- `for…of` iterates values; `for…in` iterates enumerable keys (avoid for arrays). `forEach` can't be `await`ed in sequence or `break`.
- **Prototypal inheritance**: objects link to a prototype object; property lookup walks the chain. `class` is syntax sugar over prototypes:
  ```js
  class Animal {
    #secret = 1;                          // private field
    constructor(name) { this.name = name; }
    speak() { return `${this.name} makes a sound`; }
    static create(n) { return new Animal(n); }
  }
  class Dog extends Animal { speak() { return super.speak() + " (woof)"; } }
  ```
- Immutability is shallow: `Object.freeze(obj)`; `const` ≠ immutable.
- JSON: `JSON.stringify(obj)` / `JSON.parse(str)` (dates become strings; `undefined`/functions dropped).

### 9.5 `this`
Determined by **how a function is called**, not where it is defined (except arrow functions, which inherit it lexically).
```js
const obj = { n: 1, f() { return this.n; } };
obj.f();              // 1   (method call)
const g = obj.f; g(); // undefined (this lost; in strict mode `this` is undefined)
obj.f.call({ n: 2 }); // 2   (call/apply/bind set this explicitly)
```
Callbacks lose `this` → use arrow functions or `.bind(this)`. Python contrast: Python's `self` is explicit and bound at attribute access.

### 9.6 The event loop & async programming
JavaScript is **single-threaded** with an **event loop** — conceptually like asyncio.
- Call stack runs synchronous code; async work (timers, I/O, network) is handled by the runtime; completed callbacks are queued.
- **Microtask queue** (promise callbacks, `queueMicrotask`) runs fully after the current stack, **before** the next **macrotask** (`setTimeout`, I/O events).
```js
console.log("A");
setTimeout(() => console.log("B"), 0);
Promise.resolve().then(() => console.log("C"));
console.log("D");
// A D C B
```
- **Promises** represent future values: pending → fulfilled/rejected.
  ```js
  fetch(url).then(r => r.json()).then(data => ...).catch(err => ...).finally(...);
  ```
- **async/await** (sugar over promises; like Python's `async/await`, but a promise starts running as soon as it's created — no "coroutine object never awaited" laziness):
  ```js
  async function load(ids) {
    try {
      const results = await Promise.all(ids.map(id => fetchUser(id)));    // concurrent
      return results;
    } catch (e) { console.error(e); throw e; }
  }
  ```
- Combinators: `Promise.all` (fails fast), `allSettled` (all outcomes), `race`, `any`. Sequential `for…of` with `await` vs parallel `Promise.all(map(...))`.
- Unhandled rejections crash Node (≥15). Always handle or `catch`.
- Node specifics: `process.env`, `fs/promises`, `npm`, `package.json`; Node has worker threads for CPU work (like multiprocessing).

### 9.7 Browser/DOM and tooling (just enough)
- DOM: `document.querySelector`, `addEventListener`, `element.textContent`; events bubble; event delegation. `fetch` for HTTP. `localStorage`.
- Tooling: npm/pnpm, ESLint, Prettier, Vite/webpack, Jest/Vitest. Frameworks (React, Vue, etc.) build on these ideas: components, state, virtual DOM.

### 9.8 Python ↔ JavaScript quick map
| Python | JavaScript |
|---|---|
| `None` | `null` / `undefined` |
| `True/False` | `true/false` |
| `list` / `dict` / `set` | `Array` / `Object` or `Map` / `Set` |
| `len(x)` | `x.length`, `x.size` (Map/Set) |
| `f"{x}"` | `` `${x}` `` |
| list comprehension | `.filter().map()` |
| `x in d` | `"k" in obj` / `map.has(k)` |
| `try/except/finally` | `try/catch/finally` |
| `async/await` + asyncio | `async/await` + Promises + event loop |
| `**kwargs` | options object + destructuring |
| `self` explicit | `this` implicit (rules!) |
| `is` / `==` | `===` (identity for objects) / avoid `==` |
| `import x` | `import x from "x"` |
| `pip`, `venv`, `pyproject` | `npm`, `node_modules`, `package.json` |

---

## Part 2 — Homework / Exercises (not timed)

1. Rewrite a Python script (word frequency counter) in Node using `Map`, `split`, `reduce`.
2. Predict outputs, then run:
   ```js
   console.log([] + [], [] + {}, 1 + "2", "3" - 1, [10, 9, 1].sort(), 0 == "", null == 0, null >= 0);
   ```
3. Implement `debounce(fn, ms)` and `memoize(fn)` using closures.
4. Convert a callback-based function (`setTimeout`) to a Promise (`sleep(ms)`), then write `retry(fn, times)` with `async/await`.
5. Fetch 5 URLs concurrently with `Promise.all`; then sequentially with a `for…of` loop; then limit concurrency to 2.
6. Explain the output order:
   ```js
   console.log(1);
   setTimeout(() => console.log(2));
   Promise.resolve().then(() => console.log(3));
   (async () => { console.log(4); await null; console.log(5); })();
   console.log(6);
   ```
7. Fix the bug: `class Counter { constructor(){ this.n=0; } inc(){ this.n++; } }` then `button.addEventListener("click", counter.inc)`.

<details><summary>Solutions</summary>

2. `"" "[object Object]" "12" 2 [1, 10, 9] true false true`.
3. ```js
   const debounce = (fn, ms) => { let t; return (...a) => { clearTimeout(t); t = setTimeout(() => fn(...a), ms); }; };
   const memoize = fn => { const c = new Map(); return x => c.has(x) ? c.get(x) : (c.set(x, fn(x)), c.get(x)); };
   ```
4. `const sleep = ms => new Promise(r => setTimeout(r, ms));`
6. `1 4 6 3 5 2` (sync first; microtasks 3 then 5 in order queued; macrotask 2 last).
7. `this` is the button, not counter → `counter.inc.bind(counter)` or `() => counter.inc()` or define `inc = () => {...}` as a class field.
</details>

---

## Part 3 — Quiz

**Q1.** `let` vs `const` vs `var`?

**Q2.** Output?
```js
console.log(typeof null, typeof undefined, typeof [], typeof (() => 1));
```
**Q3.** `[1, 2, 3] == "1,2,3"` → ? Why avoid `==`?

**Q4.** `0 || "x"`, `0 ?? "x"`, `"" || "y"` → ?

**Q5.** What does `[1, 10, 2].sort()` return?

**Q6.** Output order?
```js
setTimeout(() => console.log("t"), 0);
Promise.resolve().then(() => console.log("p"));
console.log("s");
```
**Q7.** What's the value of `this` inside an arrow function?

**Q8.** `Promise.all` vs `Promise.allSettled`?

**Q9.** Does `const arr = []; arr.push(1)` throw?

**Q10.** What's printed?
```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i), 0);
```
**Q11.** What does `await` do inside `array.forEach(async x => { await ... })`?

<details><summary>Answers</summary>

1. `var`: function-scoped, hoisted (initialized undefined), re-declarable. `let`: block-scoped, TDZ, reassignable. `const`: block-scoped, binding can't be reassigned (object stays mutable).
2. `object undefined object function`.
3. `true` (array coerced to string). `==` applies surprising coercions; `===` doesn't.
4. `"x"`, `0`, `"y"`.
5. `[1, 10, 2]` (string comparison). Use `(a, b) => a - b`.
6. `s p t` — sync, then microtask, then macrotask.
7. Lexically inherited from the enclosing scope (it has no own `this`).
8. `all` rejects as soon as one rejects (fail-fast); `allSettled` waits for all and reports `{status, value|reason}` for each.
9. No — `const` prevents reassigning the variable, not mutating the array.
10. `3 3 3` (single function-scoped `i`). With `let` → `0 1 2`.
11. `forEach` ignores the returned promises: iterations start concurrently and the outer code does not wait. Use `for…of` with `await` (sequential) or `await Promise.all(arr.map(async …))` (parallel).
</details>

---

## Part 4 — Interview Questions

**I1. Explain the JavaScript event loop.**
<details><summary>Model answer</summary>
Single-threaded: one call stack. Async operations (timers, I/O, network) are delegated to the host (browser/libuv); when they finish, callbacks are queued. The event loop moves tasks from queues to the stack when it's empty. Microtasks (promise reactions, `queueMicrotask`) drain completely after each task, before the next macrotask (timers, I/O). So a long synchronous computation blocks everything (UI freeze/server stall) — offload to Web Workers/worker threads. Compare: Python's asyncio has the same cooperative model.
</details>

**I2. What is a closure? Give a use and a pitfall.**
<details><summary>Model answer</summary>
A function plus its lexical environment; it keeps access to outer variables after the outer function returns. Uses: private state (`makeCounter`), factories, memoize/debounce, callbacks. Pitfalls: `var` in loops, memory retention of large captured scopes. Same concept as Python closures.
</details>

**I3. How does `this` work? How do you control it?**
<details><summary>Model answer</summary>
Set at call time: method call → the object; plain function call → `undefined` (strict) or global; `new` → new object; `call/apply/bind` → explicit; arrow functions → inherited lexically. Lost when extracting methods or passing as callbacks; fix with arrow functions, `bind`, or class-field arrows. In classes and React-style code prefer arrows/explicit binding.
</details>

**I4. Promises vs async/await vs callbacks?**
<details><summary>Model answer</summary>
Callbacks → "callback hell" and inversion of control; Promises → chaining and error propagation, composability (`all`, `race`); async/await → sequential-looking code using promises under the hood, with try/catch. Mention parallelism pitfalls (awaiting in a loop serializes), unhandled rejections, and error handling at the boundary. Contrast: JS promises are eager; Python coroutines are lazy.
</details>

**I5. Explain prototypal inheritance and how `class` relates.**
<details><summary>Model answer</summary>
Each object has an internal `[[Prototype]]` link; property lookups climb the chain until found or `null`. `class` syntax sets up constructor functions and prototype methods, `extends` links prototypes; `super` calls the parent. No multiple inheritance (use mixins/composition). Python contrast: class-based with MRO.
</details>

**I6. `==` vs `===`; truthy/falsy gotchas?**
<details><summary>Model answer</summary>
`===` compares without coercion; `==` coerces types by complex rules. Use `===` always (exception: `x == null` to check both null/undefined, which some style guides allow). Falsy list; `[]`/`{}` truthy; `NaN` falsy; `||` vs `??`.
</details>

**I7. How do you handle errors in async code?**
<details><summary>Model answer</summary>
`try/catch` around `await`, `.catch()` on promise chains, `Promise.allSettled` for partial failure, top-level handlers (`process.on("unhandledRejection")`, `window.onunhandledrejection`) as a safety net, custom Error subclasses, rethrow with `cause`: `new Error("msg", { cause: e })`. Don't swallow errors; avoid `async` functions in `forEach`.
</details>

**I8. ES modules vs CommonJS? What is npm/package.json?**
<details><summary>Model answer</summary>
ESM: `import/export`, static analysis (tree shaking), async loading, strict mode; the standard. CommonJS: `require/module.exports`, synchronous, Node legacy. `package.json`: metadata, dependencies (`dependencies` vs `devDependencies`), scripts, `"type": "module"`; lock file (`package-lock.json`) for reproducibility; semver ranges (`^1.2.3`).
</details>

**Live-coding warm-up:** (1) `debounce`, (2) flatten a nested array without `flat()`, (3) `Promise.all` implemented by hand, (4) group an array of objects by key using `reduce`.
