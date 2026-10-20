# Session 10 — TypeScript Basics (for a Python developer)

**Time:** Concepts 15 · Quiz 10 · Interview 15
Try everything in the TS Playground (typescriptlang.org/play) or locally: `npm i -D typescript tsx` → `npx tsc --init` → `npx tsx file.ts`.
Prerequisite: Session 9 (JavaScript). Mental model: **TypeScript = JavaScript + a static type system (like mypy for JS), erased at compile time.**

---

## Part 1 — Concepts

### 10.1 What TypeScript is (and isn't)
- A superset of JavaScript; the compiler (`tsc`) type-checks and emits plain JS. **Types don't exist at runtime** (same as Python hints, but TS types are checked at build time by default).
- No runtime validation of external data (API JSON) → use a validator (zod, valibot, io-ts) or type guards at boundaries.
- `tsconfig.json`: `"strict": true` (turns on `strictNullChecks`, `noImplicitAny`, etc. — always enable), `target`, `module`, `outDir`, `noUncheckedIndexedAccess`.
- Structural typing: compatibility based on shape, not declared name (like Python's `Protocol`).

### 10.2 Basic types & annotations
```ts
let n: number = 1;               // also bigint, string, boolean, null, undefined, symbol
let ids: number[] = [1, 2];      // or Array<number>
let pair: [string, number] = ["a", 1];      // tuple
const u: string | undefined = maybe();      // union
let anything: any;               // opt-out of typing (avoid)
let safe: unknown;               // like `object` – must narrow before use (preferred over any)
function fail(msg: string): never { throw new Error(msg); }   // never returns
function log(msg: string): void { console.log(msg); }
```
- Type inference: annotate function params/returns at API boundaries; let inference handle locals.
- `const x = "a"` infers literal type `"a"`; `let x = "a"` infers `string`. `as const` freezes to literal/readonly: `const dirs = ["N", "S"] as const;`.

### 10.3 Objects: interfaces vs type aliases
```ts
interface User {
  readonly id: number;
  name: string;
  email?: string;             // optional → string | undefined
}
type Point = { x: number; y: number };
type ID = string | number;
type Callback = (err: Error | null, data?: string) => void;

interface Admin extends User { perms: string[] }
type Staff = User & { dept: string };       // intersection
```
- `interface`: extendable, mergeable (declaration merging), good for object shapes/classes. `type`: can alias unions, tuples, primitives, mapped/conditional types. Teams pick one convention; common rule: `interface` for public object shapes, `type` for everything else.
- Excess property check on object literals; index signatures `{ [key: string]: number }` ≈ `dict[str, int]`; `Record<string, number>` is the same idea.
- Python parallels: `TypedDict` ≈ `interface`; `dataclass` ≈ class with fields; `Optional[X]` ≈ `X | undefined` (`null` and `undefined` are distinct!).

### 10.4 Unions, narrowing, discriminated unions
```ts
function show(x: string | number) {
  if (typeof x === "string") return x.toUpperCase();   // narrowed to string
  return x.toFixed(2);                                  // narrowed to number
}
// other narrowing tools: instanceof, `in`, truthiness, equality, Array.isArray, user-defined guards
function isUser(v: unknown): v is User { return typeof v === "object" && v !== null && "id" in v; }

// Discriminated (tagged) union — TS's algebraic data type
type Shape =
  | { kind: "circle"; r: number }
  | { kind: "rect"; w: number; h: number };

function area(s: Shape): number {
  switch (s.kind) {
    case "circle": return Math.PI * s.r ** 2;
    case "rect":   return s.w * s.h;
    default: { const _exhaustive: never = s; return _exhaustive; }   // compile error if a case is missed
  }
}
```
Literal types and unions replace many enums: `type Status = "idle" | "loading" | "error";`. TS `enum` exists, but string-literal unions are usually simpler (no runtime object, better tree-shaking).

### 10.5 Functions
```ts
function greet(name: string, greeting = "Hi", ...rest: string[]): string { ... }
const add = (a: number, b: number): number => a + b;
type Mapper<T, U> = (x: T) => U;
function overload(x: string): string;            // overload signatures
function overload(x: number): number;
function overload(x: any) { return x; }
function opts({ host, port = 5432 }: { host: string; port?: number }) { ... }
```
Optional param `b?: number`; callbacks' parameter bivariance details are advanced; `this` parameter typing available.

### 10.6 Generics
```ts
function first<T>(xs: T[]): T | undefined { return xs[0]; }
function pick<T, K extends keyof T>(obj: T, key: K): T[K] { return obj[key]; }   // constraints

interface Box<T> { value: T }
class Stack<T> {
  private items: T[] = [];
  push(x: T) { this.items.push(x); }
  pop(): T | undefined { return this.items.pop(); }
}
type ApiResponse<T> = { data: T; error?: string };
```
Python parallel: `TypeVar` + `Generic[T]`. `T extends X` ≈ bound TypeVar. Defaults: `<T = string>`.

### 10.7 Utility types & type operators
```ts
Partial<User>               // all props optional (PATCH payloads)
Required<User>; Readonly<User>
Pick<User, "id" | "name">; Omit<User, "email">
Record<"a" | "b", number>   // { a: number; b: number }
ReturnType<typeof fn>; Parameters<typeof fn>; Awaited<Promise<string>>
NonNullable<string | null>; Exclude<"a"|"b", "a">; Extract<...>
keyof User                  // "id" | "name" | "email"
typeof value                // type of a value
type Name = User["name"]    // indexed access
// Mapped type
type Flags<T> = { [K in keyof T]: boolean };
// Conditional type
type ElementOf<T> = T extends (infer U)[] ? U : never;
```
You rarely write fancy conditional types in application code; know the utilities.

### 10.8 Classes
```ts
class Account {
  static count = 0;
  #secret = 1;                              // JS private (runtime)
  constructor(public readonly id: number, private balance = 0) {}  // parameter properties
  deposit(n: number): this { this.balance += n; return this; }
  get total() { return this.balance; }
}
abstract class Repo<T> { abstract find(id: number): Promise<T | undefined>; }
class PgRepo implements Repository { ... }      // `implements` checks structural compatibility
```
`private` is compile-time only; `#private` is truly private at runtime. Prefer composition and plain objects/functions in TS—classes are optional here.

### 10.9 Async & modules in TS
```ts
async function getUser(id: number): Promise<User> {
  const res = await fetch(`/api/users/${id}`);
  if (!res.ok) throw new Error(res.statusText);
  return (await res.json()) as User;       // `as` is an assertion, NOT a runtime check!
}
import type { User } from "./types";       // type-only import (erased)
```
- Type assertions (`as`) override the compiler — dangerous; prefer guards/validators. Non-null assertion `x!` is the same smell. `satisfies` operator checks a value against a type without widening.
- `.d.ts` declaration files and `@types/*` packages provide types for JS libraries.
- Unions with `null`: with `strictNullChecks`, the compiler forces handling: `user?.name ?? "anon"`.

### 10.10 TS tooling
`tsc --noEmit` in CI, ESLint (`typescript-eslint`), Prettier, Vitest/Jest, `ts-node`/`tsx`, bundlers (Vite, esbuild). Frameworks (React + TS, NestJS, Next.js) rely on these types; Python devs often meet TS via React or Node APIs.

### 10.11 Python ↔ TypeScript map
| Python | TypeScript |
|---|---|
| `Optional[int]` | `number \| undefined` (or `\| null`) |
| `Union[int, str]` | `number \| string` |
| `Literal["a","b"]` | `"a" \| "b"` |
| `TypedDict` | `interface` / `type` |
| `Protocol` | `interface` (structural by default) |
| `TypeVar` / `Generic[T]` | `<T>` |
| `Callable[[int], str]` | `(x: number) => string` |
| `list[int]`, `dict[str,int]` | `number[]`, `Record<string, number>` |
| `Any` | `any` (unsafe) / `unknown` (safe) |
| `NoReturn` | `never` |
| mypy | `tsc` (strict) |
| Pydantic | zod |
| `match` on tagged data | discriminated union + `switch` |

---

## Part 2 — Homework / Exercises (not timed)

1. Convert a small JS utility (group-by, debounce) into strict TS with generics: `groupBy<T, K extends string | number>(items: T[], key: (x: T) => K): Record<K, T[]>`.
2. Model a payment: `type Payment = { method: "card"; last4: string } | { method: "paypal"; email: string } | { method: "cash" }`. Write `describe(p: Payment): string` with an exhaustiveness check; add a new variant and watch the compiler complain.
3. Write a type guard `isApiError(x: unknown): x is ApiError` and use it in a `catch (e: unknown)` block.
4. Given `interface User { id: number; name: string; email: string; password: string }`, derive with utility types: `PublicUser` (no password), `UserUpdate` (all optional except id), `UserKeys`.
5. Write a typed `fetchJson<T>(url: string): Promise<T>` and then replace the unchecked `as T` with a zod schema parse; explain the difference.
6. Implement `Stack<T>` and `Result<T, E> = { ok: true; value: T } | { ok: false; error: E }` and a function returning it.
7. Predict which lines error under `strict`:
   ```ts
   let a: string = null;                      // (1)
   function f(x) { return x; }                // (2)
   const o: { n: number } = { n: 1, m: 2 };   // (3)
   const el = document.getElementById("x"); el.value;   // (4)
   const xs: number[] = [1]; const y: string = xs[0];   // (5)
   ```

<details><summary>Solutions</summary>

4. `type PublicUser = Omit<User, "password">; type UserUpdate = Partial<Omit<User, "id">> & Pick<User, "id">; type UserKeys = keyof User;`
5. `as T` silences the compiler (no runtime check); schema `.parse()` validates at runtime and *infers* `T` (`z.infer<typeof Schema>`), so type and runtime check cannot drift.
7. All five error: (1) null not assignable (strictNullChecks); (2) implicit `any` parameter; (3) excess property `m`; (4) `el` possibly `null` (and `HTMLElement` has no `value`); (5) number not assignable to string.
</details>

---

## Part 3 — Quiz

**Q1.** Are TypeScript types present at runtime?

**Q2.** `any` vs `unknown`?

**Q3.** `interface` vs `type` — name two differences.

**Q4.** What does this print and why does the compiler complain (or not)?
```ts
function f(x: string | number) { return x.toFixed(); }
```
**Q5.** What is a discriminated union and why use one?

**Q6.** `Partial<T>`, `Pick<T,K>`, `Omit<T,K>`, `Record<K,V>` — one line each.

**Q7.** What does `as User` do to the result of `res.json()`? Is it safe?

**Q8.** What does `strict: true` enable that matters most?

**Q9.** `let x = "a"` vs `const x = "a"` — inferred types?

**Q10.** What does `never` represent? Give a use.

**Q11.** What's `keyof typeof obj`?

<details><summary>Answers</summary>

1. No — erased on compile. Runtime validation needs libraries or guards.
2. `any` disables checking (anything goes, infects code); `unknown` accepts any value but forces narrowing before use.
3. `interface` supports declaration merging and `extends`; `type` can alias unions, tuples, primitives, and use mapped/conditional types; class `implements` works with both for object shapes.
4. Compile error: `toFixed` doesn't exist on `string`. Narrow first: `typeof x === "number" ? x.toFixed() : x`. (Without checking, at runtime a string would throw.)
5. A union whose members share a literal "tag" property (`kind`). Narrowing by the tag gives precise types and exhaustiveness checks via `never`; models state machines/API results cleanly.
6. Partial: all props optional. Pick: subset of props. Omit: all but some props. Record: object type with keys K and values V.
7. It's a type assertion: tells the compiler to trust you; no runtime check. Unsafe if the server shape differs — validate with zod or a guard.
8. `strictNullChecks` (null/undefined handling), `noImplicitAny`, `strictFunctionTypes`, `strictPropertyInitialization`, etc.
9. `let` → `string`; `const` → literal type `"a"`.
10. The type of values that never occur: functions that throw/never return, and the leftover type after exhaustive narrowing (compile-time exhaustiveness checks).
11. The union of the object's key names as string literals (`typeof obj` gets its type; `keyof` its keys).
</details>

---

## Part 4 — Interview Questions

**I1. Why use TypeScript over JavaScript? Any downsides?**
<details><summary>Model answer</summary>
Catches whole classes of bugs at compile time (null/undefined, wrong props, typos), better editor autocompletion/refactoring, self-documenting contracts, safer large-team changes. Downsides: build step, learning curve for advanced types, types can drift from runtime reality (no validation), `any`/assertions leak, slower initial development, third-party typings of varying quality. Mitigate with strict mode, runtime validation at boundaries, and lint rules banning `any`.
</details>

**I2. `any`, `unknown`, `never` — explain.**
<details><summary>Model answer</summary>
`any`: top type that also disables checks (escape hatch). `unknown`: type-safe top type; must narrow before use — use for untrusted input and `catch (e: unknown)`. `never`: bottom type; no values — exhaustive checks, throwing functions. Relationship: everything is assignable to `unknown`; `never` is assignable to everything.
</details>

**I3. What is structural typing? Compare with Python.**
<details><summary>Model answer</summary>
Types are compatible if their shapes match, regardless of declared names. Like duck typing checked statically (Python's `Protocol`). Contrast with nominal typing (Java, Python's ABC inheritance). Consequence: you can pass any object with the required properties; excess-property checks apply only to fresh object literals.
</details>

**I4. How do you type API responses safely?**
<details><summary>Model answer</summary>
Define response types (or generate from OpenAPI via `openapi-typescript`), but treat network data as `unknown` and validate at the edge with zod (parse → typed value). Use a `Result`/discriminated-union for success/error states; wrap `fetch` in a typed client; handle `res.ok` and errors; avoid raw `as`.
</details>

**I5. Explain generics with a practical example.**
<details><summary>Model answer</summary>
Generics parametrize types so functions/containers work for many types while preserving relationships: `function first<T>(xs: T[]): T | undefined`, `Promise<T>`, `ApiResponse<T>`. Constraints (`T extends { id: number }`) allow property access; `keyof` constraints enable safe property getters (`pick<T, K extends keyof T>`). Mirrors Python `TypeVar`.
</details>

**I6. How would you model UI/request state (loading/success/error) in TS?**
<details><summary>Model answer</summary>
Discriminated union: `type State<T> = {status:"idle"} | {status:"loading"} | {status:"success"; data:T} | {status:"error"; error:string}`. Impossible states become unrepresentable (no `data` while loading), and `switch` with a `never` default guarantees all states are handled.
</details>

**I7. What's the difference between `type assertion` and `type guard`? What is `satisfies`?**
<details><summary>Model answer</summary>
Assertion (`as T`) overrides the compiler without checking; a guard (`v is T` function, `typeof`, `in`, `instanceof`) performs a real runtime check which the compiler uses to narrow. `satisfies` validates that an expression conforms to a type while preserving its more specific inferred type (useful for config objects).
</details>

**I8. As a Python dev, how do TS and Python typing differ?**
<details><summary>Model answer</summary>
TS typing is mandatory-by-compiler (build fails) and pervasive in the ecosystem; Python typing is optional and external to the interpreter. TS is structural by default; Python uses nominal classes plus Protocols. TS has richer type-level programming (mapped/conditional/template literal types), union/intersection types are first-class, narrowing is flow-sensitive in both (mypy/pyright similar). Both erase types at runtime unless you add a validator (zod ↔ Pydantic).
</details>

**Live-coding warm-up:** (1) Write `groupBy<T, K extends PropertyKey>` fully typed. (2) Create a typed event emitter `on<K extends keyof Events>(event: K, cb: (payload: Events[K]) => void)`. (3) Implement `Result<T, E>` helpers `ok`, `err`, and `mapResult`.

---

## Wrap-up — what to say when asked "How strong is your TypeScript/JavaScript?"
"My primary language is Python; I'm comfortable reading and writing JavaScript/TypeScript for APIs and tooling — I understand the event loop and promises, closures and `this`, and I use strict TypeScript with unions, generics, and utility types, validating external data with zod. I'd ramp up quickly on a framework like React or NestJS."
