# Session 2 — Functions, Closures, Decorators, Iterators, Generators, Context Managers (Intermediate)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 2.1 Functions are first-class objects

```python
def greet(name, /, greeting="Hi", *, punct="!"):   # / = positional-only, * = keyword-only
    return f"{greeting} {name}{punct}"

def f(*args, **kwargs):          # args: tuple, kwargs: dict
    ...

f(*[1, 2], **{"a": 3})           # unpacking at call site
```
- Functions have attributes (`f.__name__`, `f.__doc__`, `f.__defaults__`), can be passed, returned, stored.
- **Default values are evaluated once** at `def` time → mutable default trap (`def f(x, acc=[])`). Use `None` sentinel.
- Argument binding order: positional → `*args` → keyword-only → `**kwargs`.

### 2.2 Scope: LEGB, closures, `nonlocal`
Lookup: **L**ocal → **E**nclosing → **G**lobal → **B**uilt-in.

```python
def make_counter():
    n = 0
    def inc():
        nonlocal n          # rebind enclosing variable (reading needs no keyword)
        n += 1
        return n
    return inc

c = make_counter()
print(c(), c())             # Output: 1 2
```
A closure keeps *variables* (cells), not values:

```python
funcs = [lambda: i for i in range(3)]
print([f() for f in funcs])                # Output: [2, 2, 2]   late binding
funcs = [lambda i=i: i for i in range(3)]  # fix: bind at definition via default arg
print([f() for f in funcs])                # Output: [0, 1, 2]
```
Assigning to a name anywhere in a function makes it local for the *whole* function → `UnboundLocalError` if read first.

### 2.3 Decorators
A decorator is a callable taking a function and returning a replacement. `@d` is sugar for `f = d(f)`.

```python
import functools, time

def timed(fn):
    @functools.wraps(fn)                 # preserve __name__, __doc__, __wrapped__
    def wrapper(*args, **kwargs):
        t = time.perf_counter()
        try:
            return fn(*args, **kwargs)
        finally:
            print(f"{fn.__name__}: {time.perf_counter()-t:.4f}s")
    return wrapper

def retry(times=3, exceptions=(Exception,)):    # decorator WITH arguments = factory
    def deco(fn):
        @functools.wraps(fn)
        def wrapper(*a, **kw):
            for attempt in range(1, times + 1):
                try:
                    return fn(*a, **kw)
                except exceptions:
                    if attempt == times: raise
        return wrapper
    return deco

@retry(times=5, exceptions=(ConnectionError,))
@timed
def fetch(): ...
```
Stacking applies bottom-up: `fetch = retry(...)(timed(fetch))`.
Useful built-ins: `functools.lru_cache`/`cache`, `functools.partial`, `functools.singledispatch`, `@contextmanager`, `@property`, `@dataclass`.
Class-based decorators implement `__call__`. Decorating methods: wrapper receives `self` as first of `*args`.

### 2.4 Iterators and the iteration protocol
- **Iterable**: has `__iter__()` returning an iterator. **Iterator**: has `__next__()` (raises `StopIteration` when done) and `__iter__()` returning itself.
- `for x in obj` ≡ `it = iter(obj)`; loop `next(it)` until `StopIteration`.
- Iterators are **single-pass**; iterables like lists can be iterated repeatedly.

```python
class Countdown:
    def __init__(self, n): self.n = n
    def __iter__(self): return self
    def __next__(self):
        if self.n <= 0: raise StopIteration
        self.n -= 1
        return self.n + 1
print(list(Countdown(3)))      # Output: [3, 2, 1]
```

### 2.5 Generators
A function with `yield` returns a generator (an iterator) that **suspends** state between values: lazy, O(1) memory.

```python
def read_chunks(f, size=1024):
    while chunk := f.read(size):
        yield chunk

def fib():
    a, b = 0, 1
    while True:
        yield a
        a, b = b, a + b

from itertools import islice
print(list(islice(fib(), 8)))  # Output: [0, 1, 1, 2, 3, 5, 8, 13]

squares = (x*x for x in range(10**9))   # generator expression: no memory blowup
print(sum(x for x in range(10)))        # Output: 45
```
- `yield from sub` delegates to a sub-iterator (and forwards `send/throw`, returns sub's `return` value).
- `gen.send(v)`, `gen.throw()`, `gen.close()` make generators coroutines (the basis of old-style async).
- Pipelines: chain generators (`lines → filtered → parsed`) to process huge files streaming.
- Generator exhausted → second `list(gen)` yields `[]`.

### 2.6 Context managers

```python
class Timer:
    def __enter__(self): self.t = time.perf_counter(); return self
    def __exit__(self, exc_type, exc, tb):
        self.dt = time.perf_counter() - self.t
        return False        # True would SUPPRESS the exception

from contextlib import contextmanager, suppress, ExitStack

@contextmanager
def cd(path):
    old = os.getcwd(); os.chdir(path)
    try:
        yield
    finally:                 # always restore
        os.chdir(old)

with suppress(FileNotFoundError):
    os.remove("x")
```
Use for files, locks, DB transactions, temp state, `pytest.raises`. Multiple: `with open(a) as f, open(b) as g:`.

### 2.7 Comprehensions & functional tools

```python
[x*x for x in range(10) if x % 2]            # list
{k: v for k, v in pairs}                     # dict
{x % 3 for x in range(10)}                   # set
```
- Comprehensions have their own scope (loop variable doesn't leak in Py3).
- Prefer comprehension over `map/filter` + `lambda`; use `any`/`all`, `sorted(key=...)`, `zip`, `enumerate`.
- Walrus `:=` assignment expression: `if (n := len(a)) > 10:`.

---

## Part 2 — Homework / Exercises (not timed)

1. Write `@memoize` (dict cache keyed by args) and compare with `functools.lru_cache`. Handle unhashable args gracefully.
2. Write `@retry(times, delay, backoff)` with exponential backoff and a logged warning per failure.
3. Write `@validate_types` that checks arguments against `__annotations__` (simple types only).
4. Generator pipeline: given a 5 GB log file path, yield only lines containing `ERROR`, parse timestamp and message into tuples, and produce counts per hour — without loading the file into memory.
5. Implement `chunked(iterable, n)` generator; then `flatten(nested)` using `yield from` (recursive, skipping strings).
6. Write a context manager (both class-based and `@contextmanager`) that temporarily sets an environment variable.
7. Predict the output:
   ```python
   def gen():
       x = yield 1
       print("got", x)
       y = yield 2
       print("got", y)
   g = gen(); print(next(g)); print(g.send("a")); 
   ```

<details><summary>Solution sketches</summary>

1. ```python
   def memoize(fn):
       cache = {}
       @functools.wraps(fn)
       def w(*a):
           try:
               return cache[a]
           except KeyError:
               cache[a] = r = fn(*a); return r
           except TypeError:         # unhashable
               return fn(*a)
       return w
   ```
   (Careful: the `except TypeError` also catches errors raised in the lookup only; real implementation separates hashing from calling.)
4. `with open(p) as f: errs = (l for l in f if "ERROR" in l); ...; Counter(hour(t) for t, _ in parsed)`.
5. ```python
   def chunked(it, n):
       it = iter(it)
       while batch := list(islice(it, n)): yield batch
   def flatten(x):
       for e in x:
           if isinstance(e, (list, tuple)): yield from flatten(e)
           else: yield e
   ```
6. `@contextmanager def env(k, v): old = os.environ.get(k); os.environ[k]=v; try: yield; finally: (restore or del)`.
7. Prints `1`, then `got a` then `2`. `send("a")` resumes at the first yield with value "a".
</details>

---

## Part 3 — Quiz

**Q1.** Output?
```python
def f(a, acc=[]):
    acc.append(a); return acc
print(f(1), f(2))
```
**Q2.** Output?
```python
x = 10
def g():
    print(x)
    x = 5
g()
```
**Q3.** Why use `functools.wraps`?

**Q4.** What is the output order?
```python
@A
@B
def f(): ...
```
Which decorator wraps which, and which runs first when `f()` is called?

**Q5.** Output?
```python
g = (i for i in range(3))
print(list(g), list(g))
```
**Q6.** Difference between iterable and iterator? Is a list an iterator?

**Q7.** What does returning `True` from `__exit__` do?

**Q8.** `[lambda: i for i in range(3)]` — what do the lambdas return and why?

**Q9.** What's wrong with: `def f(): return [x for x in huge_file]` vs generator? When is each right?

<details><summary>Answers</summary>

1. `[1, 2] [1, 2]` — same list object.
2. `UnboundLocalError` — `x = 5` makes `x` local for the whole function.
3. Copies `__name__`, `__doc__`, `__module__`, `__wrapped__` onto the wrapper — keeps debugging, introspection, and tools (pytest, Sphinx, `inspect.signature`) correct.
4. `f = A(B(f))`. B wraps the original, A wraps B's result. Calling `f()` enters A's wrapper first, then B's.
5. `[0, 1, 2] []` — generator exhausted.
6. Iterable: can produce an iterator via `iter()`. Iterator: implements `__next__`, one-shot. A list is iterable, not an iterator (`next([1])` → TypeError).
7. Suppresses the exception raised inside the `with` block (swallows it). Dangerous unless intentional.
8. All return `2` — closure captures the variable `i`, evaluated at call time. Fix with `lambda i=i: i`.
9. The list materializes everything (memory O(n)); the generator is lazy O(1). Use a list if you need random access, `len`, or multiple passes.
</details>

---

## Part 4 — Interview Questions

**I1. What is a closure? Give a practical use.**
<details><summary>Model answer</summary>
A function that retains references to variables from its enclosing scope after that scope has returned (stored in `__closure__` cells). Uses: factories (`make_multiplier(n)`), decorators with state, callbacks, partial application, simple encapsulation instead of a class. Gotcha: late binding in loops; need `nonlocal` to rebind.
</details>

**I2. Walk me through writing a decorator that takes arguments.**
<details><summary>Model answer</summary>
Three layers: outer factory takes the config and returns the decorator; decorator takes the function and returns the wrapper; wrapper takes `*args, **kwargs`, adds behavior, calls the function. Use `@functools.wraps`. Be able to write `retry(times)` or `rate_limit` on the whiteboard. Mention async: if the wrapped function is a coroutine function, wrapper must be `async def` and `await` it.
</details>

**I3. Generators vs lists — when and why? How do they work internally?**
<details><summary>Model answer</summary>
A generator function returns an object holding a suspended frame; each `next()` runs until the next `yield`, preserving locals. Benefits: lazy evaluation, constant memory, infinite streams, pipeline composition, early exit. Costs: single-pass, no `len`/indexing, harder to debug. Use for ETL on big files, DB cursor streaming, paginated API reads. Mention `yield from` and that generator expressions are the compact form.
</details>

**I4. Explain the iterator protocol. How would you make a class iterable?**
<details><summary>Model answer</summary>
Implement `__iter__` returning an iterator; the iterator implements `__next__` raising `StopIteration`. Easiest: make `__iter__` a generator method (`yield` inside). Separate iterable from iterator when you want multiple independent traversals (iterable returns a *new* iterator each time).
</details>

**I5. What is `lru_cache`, and what are the pitfalls?**
<details><summary>Model answer</summary>
Memoization decorator with a bounded LRU cache keyed by args (must be hashable), thread-safe for its own bookkeeping. `maxsize=None` = unbounded (`@cache`). Pitfalls: caches on methods hold `self` references (memory leaks; use `cached_property` or per-instance caches), mutable return values are shared, no TTL, stale data for impure functions. `cache_info()` and `cache_clear()` for introspection.
</details>

**I6. How do context managers work and why prefer them to try/finally?**
<details><summary>Model answer</summary>
`with` calls `__enter__`, binds its return value, runs the block, then calls `__exit__(exc_type, exc, tb)` regardless of outcome; truthy return suppresses the exception. It packages setup/teardown, making resource handling reusable and hard to forget. `@contextmanager` converts a generator into one. Examples: files, locks, DB transactions (`with conn:`), `tempfile`, `pytest.raises`, `decimal.localcontext`.
</details>

**I7. Difference between `*args/**kwargs` use cases; positional-only and keyword-only parameters—why do they exist?**
<details><summary>Model answer</summary>
`*args` collects extras positionally, `**kwargs` collects extra keywords; used for forwarding and flexible APIs. `/` makes params positional-only (lets you rename them without breaking callers; mirrors C builtins); `*` forces keyword-only (clarity for flags: `connect(host, *, timeout=5)`). Boolean flags should usually be keyword-only.
</details>

**I8 (Advanced). What's the difference between `yield` in a generator and `await` in a coroutine? How are they related?**
<details><summary>Model answer</summary>
Both suspend execution and resume later. Generators produce values to a consumer driving `next()`; coroutines (`async def`) are driven by an event loop and `await` suspends until an awaitable completes. Historically asyncio coroutines were generator-based (`yield from`); native coroutines are a distinct type but share the suspension mechanism. See Session 4.
</details>

**Live-coding warm-up:** Write a `@debounce`/`@rate_limited(calls, period)` decorator; then a generator `sliding_window(iterable, n)` using `collections.deque(maxlen=n)`.
