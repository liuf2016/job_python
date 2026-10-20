# Session 3 — Python Internals: Objects, Memory, Mutability, GIL, Gotchas (Advanced)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 3.1 Names, objects, references
Variables are **names bound to objects**, not boxes holding values. Each object has identity (`id`), type, value.

```python
a = [1, 2]
b = a              # same object, two names
b.append(3)
print(a)           # Output: [1, 2, 3]
print(a is b)      # True: identity.   a == b: equality of value

def f(x): x.append(1); x = [9]    # mutating affects caller; rebinding doesn't
```
Python is **pass-by-object-reference** ("pass by assignment"): the callee gets a new name bound to the same object.

### 3.2 Mutable vs immutable
- Immutable: `int float str bytes tuple frozenset bool None`. Mutable: `list dict set bytearray` and most user objects.
- A tuple is immutable but can contain mutable items: `t = ([],); t[0].append(1)` works. `t[0] += [2]` raises TypeError **but still mutates** (augmented assignment extends in place, then fails on rebinding).
- `x += y` on lists calls `__iadd__` (in-place); on tuples/ints/strs makes a new object.
- `is` for singletons (`None`, `True`). Never `is` for ints/strings: small-int cache (-5..256) and string interning are CPython implementation details.

```python
a = 256; b = 256; print(a is b)     # True (cached)
a = 257; b = 257                     # in the REPL: False; in one module/compiled unit: may be True
```

### 3.3 Copying
```python
import copy
a = [[1], [2]]
s = a.copy()          # or a[:], list(a): SHALLOW  – inner lists shared
d = copy.deepcopy(a)  # recursive; handles cycles via memo
a[0].append(9); print(s[0], d[0])    # [1, 9] [1]
[[0]*3]*3             # BUG: 3 references to the same row
[[0]*3 for _ in range(3)]   # correct
```

### 3.4 Hashing, dicts, sets
- Hashable ⇒ immutable-ish: `__hash__` stable over the lifetime, and `a == b ⇒ hash(a) == hash(b)`.
- `dict`/`set` are hash tables: average O(1) get/set/delete/`in`; worst case O(n) with collisions. Dicts preserve insertion order (guaranteed since 3.7).
- Mutating a dict while iterating → `RuntimeError`. Modifying keys that affect their hash → lost entries.
- `list.__contains__` is O(n); convert to `set` for repeated membership tests.
- Dict key equality: `1 == 1.0 == True` → `{1: 'a', 1.0: 'b', True: 'c'}` is `{1: 'c'}`.

### 3.5 Memory management
1. **Reference counting** (primary): object freed immediately when count hits 0. `sys.getrefcount(x)`.
2. **Cyclic garbage collector** (`gc` module): detects reference cycles that refcounting can't free; generational (gen 0/1/2).
3. **pymalloc**: arena/pool allocator for small objects (≤512 bytes). Memory may not be returned to the OS even after freeing.
4. `weakref`: reference that doesn't increase refcount (caches, observers, avoiding leaks).
5. `__del__` is unreliable timing; prefer context managers/`weakref.finalize`.

Common leaks: unbounded caches/globals, closures/`lru_cache` on methods holding `self`, lingering references in tracebacks (`except ... as e` is cleared at block end), event listeners never removed.

Tools: `tracemalloc`, `sys.getsizeof` (shallow), `objgraph`, `memory_profiler`.

### 3.6 The GIL (Global Interpreter Lock)
- A mutex in CPython so only **one thread executes Python bytecode at a time**. Protects interpreter internals/refcounts.
- Consequences: CPU-bound threads don't run in parallel; I/O-bound threads do overlap (GIL released during blocking I/O and in many C extensions like NumPy).
- Doesn't make your code thread-safe: `x += 1` is multiple bytecodes; race conditions still occur.
- Bypass: `multiprocessing`, C extensions releasing the GIL, subinterpreters (3.12+ per-interpreter GIL), and the **free-threaded build (3.13+ experimental, PEP 703)**.
- Details in Session 4.

### 3.7 Execution model
Source → bytecode (`.pyc` in `__pycache__`) → executed by the CPython VM (stack machine). `dis.dis(fn)` shows bytecode. Compilation happens at import; `def` and `class` statements execute at runtime (defaults evaluated at `def` time; class bodies run once, top to bottom).
Modules are cached in `sys.modules`; second `import` is a lookup. Circular imports: import inside the function or restructure.

### 3.8 Classic gotchas cheat-sheet

```python
# 1. Mutable default argument
def f(x, l=[]): ...
# 2. Late-binding closures in loops (see Session 2)
# 3. Modifying a list while iterating
a = [1, 2, 3, 4]
for x in a:
    if x % 2 == 0: a.remove(x)         # skips elements → [1, 3]
# 4. is vs ==
# 5. Chained comparison
1 < 2 < 3        # True; (1 < 2) < 3 evaluated differently only if you add parens
# 6. Tuple with one element
t = (1)          # int!  vs (1,)
# 7. Rounding
round(2.5)       # 2  (banker's rounding);  0.1 + 0.2 == 0.3  → False  (use Decimal / math.isclose)
# 8. Sort stability: sorted() is stable → sort by multiple keys in passes or tuple key
# 9. String concatenation in loop is O(n²) in worst case; use "".join(parts)
# 10. except: / except Exception: swallowing; bare except catches KeyboardInterrupt
# 11. Falsy values: 0, 0.0, "", [], {}, set(), None, False  → `if x:` vs `if x is not None:`
# 12. dict.get(k, default) evaluates default eagerly; use setdefault/defaultdict for expensive defaults
```

### 3.9 Integers, strings, bytes
- `int` is arbitrary precision. `float` is IEEE-754 double. `Decimal` for money, `Fraction` for exact rationals.
- `str` is immutable Unicode (code points); `bytes` is raw bytes. `.encode("utf-8")` / `.decode("utf-8")`. Always specify encoding on `open()`.
- Strings: CPython uses compact representations (1/2/4 bytes per char); slicing copies; `in` is substring search.

### 3.10 Performance mental model
- Function calls and attribute lookups are relatively expensive; local variable access is fastest.
- Built-ins and C-implemented methods (`sum`, `sorted`, `str.join`) beat hand-written loops.
- Big-O of common ops: list append O(1) amortized, `insert(0, x)` O(n), `pop(0)` O(n) (use `deque`), `x in list` O(n), `x in set/dict` O(1) avg, sort O(n log n), slicing O(k).

---

## Part 2 — Homework / Exercises (not timed)

1. Predict, then verify with `id()`: what happens to `id(x)` before/after `x += [1]` for a list vs `x += (1,)` for a tuple vs `x += 1` for int?
2. Create a reference cycle (`a.ref = b; b.ref = a`), `del` both, and show `gc.collect()` returns > 0; then rewrite with `weakref` to avoid the cycle.
3. Use `tracemalloc` to compare memory of 1M instances of a normal class vs a `__slots__` class.
4. Implement a simple `deepcopy`-like function for nested lists/dicts handling cycles using a memo dict.
5. Demonstrate the GIL: time a CPU-bound function (`sum(i*i for i in range(10**7))`) run twice sequentially vs in two threads; then via `ProcessPoolExecutor`.
6. Explain the output without running:
   ```python
   a = [1, 2, 3]
   b = a
   a += [4]
   a = a + [5]
   print(a, b)
   ```
7. `dis.dis` the function `def f(x): x += 1` and `def g(x): x = x + 1`... what's the difference for lists?

<details><summary>Solutions</summary>

1. List: same id (in-place `__iadd__`). Tuple: new object (and `(1)` is an int, TypeError — use `(1,)`). Int: new object.
2. `weakref.ref(b)` or `weakref.proxy` for the back-reference.
3. Typically ~2× smaller with slots (e.g., ~100 MB vs ~50 MB for 1M small objects).
5. Threads ≈ same total time as sequential (maybe slower); processes ≈ half the time on ≥2 cores.
6. `[1,2,3,4,5] [1,2,3,4]` — `+=` mutated the shared object (b sees `4`); then `a = a + [5]` rebinds `a` to a new list.
7. `INPLACE_ADD`/`BINARY_OP (+=)` vs `BINARY_OP (+)`; for mutable types `+=` can mutate in place.
</details>

---

## Part 3 — Quiz

**Q1.** Output?
```python
a = [1, 2, 3]; b = a; b = b + [4]; print(a)
```
**Q2.** Output?
```python
d = {}
d[1] = "int"; d[1.0] = "float"; d[True] = "bool"
print(d, len(d))
```
**Q3.** Why does `[[0]*3]*3` produce a bug?

**Q4.** `t = ([1], [2]); t[0] += [3]` — what happens?

**Q5.** True/False: with the GIL, `counter += 1` from multiple threads is safe.

**Q6.** How does CPython free memory? Why a cyclic GC in addition to refcounting?

**Q7.** `0.1 + 0.2 == 0.3`? What should you use instead?

**Q8.** What is the complexity of `list.pop(0)`, `x in list`, `x in set`, `dict[key]`, `sorted(list)`?

**Q9.** What's the output?
```python
def f(): 
    try: return "try"
    finally: return "finally"
print(f())
```

<details><summary>Answers</summary>

1. `[1, 2, 3]` — `b + [4]` creates a new list and rebinds `b`.
2. `{1: 'bool'} 1` — equal & same hash; key stays the first (`1`), value is last assigned.
3. Outer list holds 3 references to the same inner list; mutate one row → all change.
4. The list is mutated (`[1, 3]`), then `TypeError: 'tuple' object does not support item assignment`. Both happen.
5. False. `+=` isn't atomic (load, add, store); a thread switch between them loses updates. Use `threading.Lock`/`queue`/atomic ops.
6. Refcount reaches 0 → immediate dealloc. Cycles (A↔B) never hit 0, so the generational GC periodically finds unreachable cycles.
7. `False` (binary floating-point). Use `math.isclose` or `decimal.Decimal`.
8. O(n), O(n), O(1) average, O(1) average, O(n log n) (Timsort; O(n) on already sorted).
9. `"finally"` — a `return` in `finally` overrides (and swallows exceptions too). Avoid.
</details>

---

## Part 4 — Interview Questions

**I1. Explain how Python passes arguments. Is it pass-by-value or by-reference?**
<details><summary>Model answer</summary>
Neither exactly: "pass by object reference/assignment". The parameter name is bound to the same object as the argument. Mutating the object (append) is visible to the caller; rebinding the local name is not. Show the `def f(x): x.append(1); x = []` example. Immutable arguments can't be changed in place, so they *appear* pass-by-value.
</details>

**I2. What is the GIL, why does it exist, and how do you work around it?**
<details><summary>Model answer</summary>
CPython's global lock allowing one thread to run bytecode at a time; it exists because reference counting isn't thread-safe and a single lock is simple and fast for single-threaded code and C-extension compatibility. Effect: threads don't speed up CPU-bound code but help I/O-bound code (GIL is released while waiting). Workarounds: `multiprocessing`/`ProcessPoolExecutor`, native extensions (NumPy, Cython, Rust via PyO3) that release the GIL, `asyncio` for I/O concurrency, and — on newer versions — free-threaded builds (3.13+ experimental) and per-interpreter GIL (3.12+). Not all implementations have it (PyPy does; Jython/IronPython don't).
</details>

**I3. How does Python manage memory? Have you debugged a memory leak?**
<details><summary>Model answer</summary>
Refcounting + generational cyclic GC + pymalloc for small objects. Leak story template: "Memory grew in a long-running service; I used `tracemalloc` snapshots (compare before/after) / `objgraph` to find growing types; cause was an unbounded dict cache / `lru_cache` on a method / listener list. Fixed with bounded LRU/TTL, weakrefs, or clearing references." Mention `gc.get_objects`, `gc.collect`, and that memory not returned to OS isn't necessarily a leak (fragmentation/arenas).
</details>

**I4. Shallow vs deep copy? When does each matter?**
<details><summary>Model answer</summary>
Shallow copies the container but shares the contained objects; deep copy recursively copies (with a memo for cycles/shared refs). Matters with nested mutable structures (config dicts, matrices, default templates). Deep copy cost is high; consider immutable data structures or explicit copy of only what changes. Customizable via `__copy__`/`__deepcopy__`.
</details>

**I5. Why must dict keys be hashable? What happens with a custom class?**
<details><summary>Model answer</summary>
Dict stores entries by `hash(key)` → slot, then compares with `==` on collisions. The hash must be stable and consistent with equality, hence immutable (or at least not mutated). Custom classes hash by identity by default; if you define `__eq__`, define `__hash__` on the same immutable fields, or `frozen=True` dataclass. Mention open-addressing and resize at ~2/3 load; average O(1).
</details>

**I6. `is` vs `==`?**
<details><summary>Model answer</summary>
`==` calls `__eq__` (value equality); `is` compares identity (same object). Use `is` for `None`/singletons only. Integer/string caching makes `is` unreliable for values.
</details>

**I7. What happens when you `import` a module? How do you break a circular import?**
<details><summary>Model answer</summary>
Check `sys.modules`; if absent, locate via finders (`sys.path`), create the module, execute its code top to bottom, cache it. Circular: A imports B while B imports A and A is partially initialized → `ImportError`/`AttributeError`. Fix by moving shared code to a third module, importing inside functions, importing the module (not names) and referencing attributes lazily, or using dependency injection.
</details>

**I8 (Advanced). How would you speed up a slow Python program?**
<details><summary>Model answer</summary>
Measure first: `cProfile`/`py-spy`/`line_profiler`. Then: better algorithm/data structure (set/dict, avoid O(n²)), reduce work (caching, early exits), use built-ins and comprehensions, vectorize (NumPy/pandas), batch I/O / DB queries (N+1), concurrency (async/threads for I/O, processes for CPU), native extensions (Cython/Numba/Rust), and only last micro-optimizations. State that you validate with benchmarks (`timeit`, `pytest-benchmark`).
</details>

**Live-coding warm-up:** Implement a bounded `TTLCache` using a dict and `time.monotonic()`; discuss memory growth and thread safety.
