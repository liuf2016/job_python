# Session 1 — Python Data Model, Classes & Dataclasses (Intermediate)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 1.1 The data model: dunder methods
Python syntax maps to special methods. Implement them and your objects behave like built-ins.

```python
class Vec:
    def __init__(self, x, y):
        self.x, self.y = x, y

    def __repr__(self):                  # unambiguous, for developers; used by REPL/logging
        return f"Vec({self.x!r}, {self.y!r})"

    def __str__(self):                   # readable, for users; falls back to __repr__
        return f"({self.x}, {self.y})"

    def __eq__(self, other):
        if not isinstance(other, Vec):
            return NotImplemented        # let Python try the reflected op / fall back
        return (self.x, self.y) == (other.x, other.y)

    def __hash__(self):                  # must be defined if __eq__ is, to be hashable
        return hash((self.x, self.y))

    def __add__(self, other):
        return Vec(self.x + other.x, self.y + other.y)

    def __mul__(self, k):                # Vec * 3
        return Vec(self.x * k, self.y * k)
    __rmul__ = __mul__                   # 3 * Vec

    def __bool__(self):                  # truthiness; else __len__; else always True
        return bool(self.x or self.y)

    def __abs__(self):
        return (self.x**2 + self.y**2) ** 0.5

print(Vec(1, 2) + Vec(3, 4))   # Output: (4, 6)
print(3 * Vec(1, 2))           # Output: (3, 6)
print({Vec(1, 2), Vec(1, 2)})  # Output: {Vec(1, 2)}
```

Key rules:
- Defining `__eq__` without `__hash__` sets `__hash__ = None` (unhashable). Equal objects **must** have equal hashes.
- Return `NotImplemented` (not raise) from binary ops for unsupported types.
- Container protocol: `__len__`, `__getitem__`, `__iter__`, `__contains__`. With only `__getitem__` (ints from 0) you can already iterate.
- `__enter__/__exit__` (context manager), `__call__` (callable objects), `__getattr__` (called only when normal lookup *fails*), `__getattribute__` (called on *every* access).

### 1.2 Attributes, class vs instance, three method kinds

```python
class Counter:
    created = 0                      # class attribute, shared

    def __init__(self, start=0):
        self.n = start               # instance attribute
        Counter.created += 1

    def inc(self):                   # instance method: gets self
        self.n += 1

    @classmethod
    def from_str(cls, s):            # alternative constructor; works with subclasses
        return cls(int(s))

    @staticmethod
    def is_valid(s):                 # no self/cls; just a namespaced function
        return s.isdigit()
```

**Gotcha — mutable class attribute:**
```python
class Bad:
    items = []                       # shared across ALL instances
    def add(self, x): self.items.append(x)

a, b = Bad(), Bad()
a.add(1)
print(b.items)                       # Output: [1]  (surprise)
```
Fix: create the list in `__init__`.

**Attribute lookup order** (instance): data descriptors on the type → instance `__dict__` → non-data descriptors / class attributes (walking the MRO) → `__getattr__`.

### 1.3 Properties (managed attributes)

```python
class Temp:
    def __init__(self, c): self.celsius = c        # goes through the setter

    @property
    def celsius(self): return self._c

    @celsius.setter
    def celsius(self, v):
        if v < -273.15: raise ValueError("below absolute zero")
        self._c = v

    @property
    def fahrenheit(self): return self._c * 9/5 + 32   # computed, read-only
```
Start with plain attributes; add `@property` later without breaking callers. No Java-style getters/setters up front.

### 1.4 Inheritance, `super()`, and MRO

```python
class A:
    def hi(self): print("A")
class B(A):
    def hi(self): print("B"); super().hi()
class C(A):
    def hi(self): print("C"); super().hi()
class D(B, C):
    def hi(self): print("D"); super().hi()

D().hi()               # Output: D B C A
print(D.__mro__)       # (D, B, C, A, object)
```
- `super()` follows the **MRO** of the *instance's* class, not "the parent". That's why B's `super()` reaches C.
- MRO uses **C3 linearization**: children before parents, left-to-right order of bases preserved.
- Cooperative multiple inheritance: every method calls `super()`, and accepts `**kwargs`.
- Mixins: small classes adding one behavior (`JsonMixin`), placed *left* of the main base.
- Prefer **composition** ("has-a") when the relationship isn't truly "is-a".

### 1.5 Abstract base classes and Protocols

```python
from abc import ABC, abstractmethod

class Repo(ABC):
    @abstractmethod
    def get(self, id: int): ...

Repo()                        # TypeError: Can't instantiate abstract class
```
`typing.Protocol` gives **structural** typing ("duck typing, checked statically") — see Session 5.

### 1.6 Dataclasses

```python
from dataclasses import dataclass, field

@dataclass(frozen=True, order=True, slots=True)
class Item:
    name: str
    price: float = 0.0
    tags: list[str] = field(default_factory=list)   # NEVER `= []` (ValueError)

    def __post_init__(self):                          # validation hook
        if self.price < 0: raise ValueError("price")
```
- Generates `__init__`, `__repr__`, `__eq__` (and `__lt__` etc. with `order=True`).
- `frozen=True` → immutable + hashable (with `eq=True`). `slots=True` (3.10+) → no `__dict__`, less memory.
- Alternatives: `NamedTuple` (immutable, tuple-like), `TypedDict` (typed dict shape only), Pydantic (runtime validation).

### 1.7 `__slots__` & descriptors (going deeper)
- `__slots__ = ("x","y")` removes per-instance `__dict__` → ~40–50% less memory per object; no dynamic attributes; weakrefs need `"__weakref__"` slot.
- A **descriptor** is an object with `__get__`/`__set__`/`__delete__` on a class. `property`, methods, `classmethod`, `staticmethod`, and `__slots__` are all descriptors. Functions are descriptors: `obj.method` binds `self` via `function.__get__`.

---

## Part 2 — Homework / Exercises (not timed)

1. **Vector2D**: add `__sub__`, `__neg__`, `__iter__` (so `x, y = v` works), `__len__`... decide whether `__len__` makes sense (it shouldn't — explain why).
2. **Money**: frozen dataclass with `amount: Decimal`, `currency: str`; `__add__` raising `ValueError` on currency mismatch; sortable within same currency.
3. **Predict the MRO** for `class X(A, B)`, `class Y(B, A)`, `class Z(X, Y)` with `A`, `B` plain classes. (Hint: try it; explain the error.)
4. **Sequence**: write `Playlist` supporting `len()`, indexing, slicing, `in`, iteration, and `reversed()` using as few dunders as possible.
5. **Validated attribute**: write a descriptor class `Positive` used as `class P: x = Positive()` that rejects negatives.
6. **Singleton / registry**: implement two ways — via `__new__`, and via a module-level instance. Which is more Pythonic?

<details><summary>Solution sketches</summary>

1. `__len__` implies a sequence length/truthiness; a 2D vector has fixed arity, `len(v)==2` is arguably OK but misleading — define `__iter__` only.
2. ```python
   @dataclass(frozen=True, order=True)
   class Money:
       amount: Decimal; currency: str
       def __add__(self, o):
           if self.currency != o.currency: raise ValueError("currency mismatch")
           return Money(self.amount + o.amount, self.currency)
   ```
   Note `order=True` compares across currencies too — override `__lt__` if that matters.
3. `X(A,B)` → (X,A,B,object); `Y(B,A)` → (Y,B,A,object); `Z(X,Y)` → `TypeError: Cannot create a consistent MRO` (X wants A before B, Y wants B before A).
4. Implement `__len__` and `__getitem__`; `in`, iteration, `reversed()` all work (slicing passes a `slice` object; delegate to the underlying list).
5. ```python
   class Positive:
       def __set_name__(self, owner, name): self.name = "_" + name
       def __get__(self, obj, owner=None): return self if obj is None else getattr(obj, self.name)
       def __set__(self, obj, v):
           if v < 0: raise ValueError
           setattr(obj, self.name, v)
   ```
6. Module-level instance (modules are singletons) is simplest and most Pythonic.
</details>

---

## Part 3 — Quiz (step through; answer aloud first)

**Q1.** What does this print?
```python
class A:
    x = 1
class B(A): pass
B.x = 2
print(A.x, B.x)
a = A(); a.x = 5
print(A.x, a.x)
```
**Q2.** Why is this a bug?
```python
class P:
    def __init__(self, tags=[]): self.tags = tags
```
**Q3.** Difference between `__repr__` and `__str__`? Which one does `print([obj])` use for the element?

**Q4.** A class defines `__eq__` only. Can its instances go in a set? Why?

**Q5.** What is `D.__mro__` for `class A: ...; class B(A): ...; class C(A): ...; class D(B, C): ...`?

**Q6.** `@classmethod` vs `@staticmethod` — when to choose each?

**Q7.** What error does `@dataclass class X: items: list = []` raise, and why?

**Q8.** Does `super().__init__()` always call the direct parent's `__init__`?

<details><summary>Answers</summary>

1. `1 2`, then `1 5`. Assigning via `B.x`/`a.x` creates/overwrites on that object, not on `A`.
2. Default evaluated once at function definition; every instance shares the same list. Use `tags=None` then `self.tags = tags if tags is not None else []`.
3. `__repr__`: unambiguous, ideally `eval`-able, for developers. `__str__`: readable for end users. Containers use `repr()` of elements → `__repr__`.
4. No: defining `__eq__` sets `__hash__` to `None` → `TypeError: unhashable type`. Define `__hash__` consistent with `__eq__`.
5. `(D, B, C, A, object)`.
6. `classmethod` when you need the class (alternative constructors, works with subclasses). `staticmethod` when it needs neither — often better as a module function.
7. `ValueError: mutable default <class 'list'> is not allowed: use default_factory` — shared-state hazard; dataclass blocks list/dict/set defaults.
8. No — it follows the MRO of the instance's class; in multiple inheritance it may call a sibling.
</details>

---

## Part 4 — Interview Questions

**I1. Explain Python's data model. Why do dunder methods exist?**
<details><summary>Model answer</summary>
Python's syntax and built-ins delegate to special methods: `len(x)`→`x.__len__()`, `a+b`→`a.__add__(b)`, `for` → `__iter__`/`__next__`, `with` → `__enter__/__exit__`. By implementing them, user types integrate with the language (sorting, hashing, iteration, context management). Example: making a `Vec` support `+`, `abs`, `==`, and use as dict keys. Gotcha: `__eq__`/`__hash__` must agree. Follow-ups: <em>Why `NotImplemented`?</em> (lets Python try the other operand's reflected method).
</details>

**I2. How does method resolution and `super()` work with multiple inheritance?**
<details><summary>Model answer</summary>
C3 linearization gives a deterministic MRO: each class before its bases, preserving the left-to-right order of the bases. `super()` returns a proxy that looks up the *next class in the instance type's MRO after the current class*. Therefore all classes in a diamond are called exactly once if each cooperatively calls `super()`. Practical advice: keep hierarchies shallow, use mixins with `**kwargs` forwarding, or prefer composition. Follow-up: <em>diamond problem?</em> → solved by MRO; no duplicate base call.
</details>

**I3. `@property` — what is it, and how does it work under the hood?**
<details><summary>Model answer</summary>
A built-in data descriptor: a class attribute with `__get__/__set__/__delete__`. Attribute access on an instance triggers it because data descriptors take precedence over instance `__dict__`. Use it for validation, computed attributes, lazy values, and for evolving a public attribute into logic without changing the API. Compare `functools.cached_property` (non-data descriptor; caches into instance `__dict__`; not thread-safe in the sense of may compute twice in older versions, and needs a `__dict__`).
</details>

**I4. Dataclass vs NamedTuple vs plain class vs Pydantic model?**
<details><summary>Model answer</summary>
Plain class: full control, most boilerplate. `dataclass`: generated `__init__/__repr__/__eq__`, mutable by default, `frozen`/`slots` options, no runtime type validation. `NamedTuple`: immutable, tuple behaviors (unpacking, indexable), tiny. Pydantic: parses and validates/coerces data at runtime (API boundaries, config). Rule: internal domain objects → dataclass; untrusted external data → Pydantic.
</details>

**I5. Composition vs inheritance — which and why?**
<details><summary>Model answer</summary>
Inheritance couples subclass to base implementation (fragile base class problem) and exposes the whole base interface; use for genuine "is-a" and framework extension points. Composition injects collaborators ("has-a"), easier to test (swap fakes), and combine behaviors at runtime. In Python, duck typing/Protocols lower the need for inheritance for polymorphism. Example: a `Notifier` taking an injected `sender` object rather than subclassing `EmailSender`.
</details>

**I6. What's the difference between `__new__` and `__init__`? When would you override `__new__`?**
<details><summary>Model answer</summary>
`__new__(cls, ...)` creates and returns the instance (static method); `__init__(self, ...)` initializes it afterward, only if `__new__` returned an instance of `cls`. Override `__new__` for immutable subclasses (`int`, `str`, `tuple`), singletons/flyweights, or custom instance creation. Otherwise leave it alone.
</details>

**I7. What are `__slots__` and when would you use them?**
<details><summary>Model answer</summary>
Replace the per-instance `__dict__` with fixed storage. Saves memory and speeds attribute access slightly; useful when creating millions of small objects. Trade-offs: no dynamic attributes, inheritance subtleties (subclass without `__slots__` regains `__dict__`), weakref needs explicit slot.
</details>

**I8 (Advanced). What is a metaclass? When have you seen one used?**
<details><summary>Model answer</summary>
A class whose instances are classes (`type` is the default). `class Foo(metaclass=Meta)` runs `Meta.__new__/__init__` when the class is created. Used by ORMs (Django models), `ABCMeta`, enum, plugin registries. Modern alternatives often suffice: `__init_subclass__` for registration/validation, class decorators. "If you aren't sure you need a metaclass, you don't."
</details>

**Live-coding warm-up:** Implement an `LRUCache(capacity)` class with `get`/`put` O(1) using `OrderedDict` (then discuss a manual doubly-linked-list + dict version).
