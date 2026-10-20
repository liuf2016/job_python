# Session 5 — Type Hints, Exceptions, Testing, Logging & Project Hygiene (Intermediate/Advanced)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 5.1 Type hints (PEP 484 and friends)
Hints are **not enforced at runtime**; they power static checkers (mypy, pyright), IDEs, and libraries (Pydantic, FastAPI, dataclasses).

```python
from collections.abc import Iterable, Callable, Sequence, Mapping
from typing import Any, Optional, Union, Literal, TypeVar, Generic, Protocol, TypedDict, Final, ClassVar, overload

def mean(xs: Iterable[float]) -> float: ...
def find(users: Sequence["User"], id: int) -> "User | None": ...    # 3.10+: X | Y, X | None
def apply(fn: Callable[[int], str], x: int) -> str: ...
Mode = Literal["r", "w"]
```
- Modern syntax (3.9+): `list[int]`, `dict[str, int]`, `tuple[int, ...]`; (3.10+) `int | None`; (3.12+) `type Alias = ...` and `def f[T](x: T) -> T`.
- Use abstract types for parameters (`Sequence`, `Mapping`, `Iterable`), concrete for returns.
- `Any` disables checking; `object` means "anything, but you can't use it unchecked".
- **Generics**:
  ```python
  T = TypeVar("T")
  def first(xs: Sequence[T]) -> T: return xs[0]
  class Stack(Generic[T]):
      def __init__(self) -> None: self._items: list[T] = []
      def push(self, x: T) -> None: self._items.append(x)
      def pop(self) -> T: return self._items.pop()
  ```
- **Protocol** (structural subtyping):
  ```python
  class SupportsClose(Protocol):
      def close(self) -> None: ...
  def shutdown(r: SupportsClose) -> None: r.close()   # any object with close() qualifies
  ```
- `TypedDict` (dict with known keys), `NamedTuple`, `Final`, `Literal`, `Self`, `ParamSpec` (typing decorators), `@overload`, `TypeGuard`, `Annotated` (metadata used by FastAPI/Pydantic).
- Variance (advanced): `list[Dog]` is **not** a `list[Animal]` (invariant, because lists are mutable); `Sequence[Dog]` is a `Sequence[Animal]` (covariant).
- Gradual typing: adopt incrementally; `mypy --strict` for new code; `# type: ignore[code]` sparingly.

### 5.2 Exceptions
```python
try:
    value = int(s)
except ValueError as e:                 # catch specific exceptions
    raise ConfigError(f"bad port: {s!r}") from e    # chain: preserves cause
else:
    use(value)                          # runs only if no exception
finally:
    cleanup()                           # always runs
```
- Hierarchy: `BaseException` → (`KeyboardInterrupt`, `SystemExit`, `GeneratorExit`) and `Exception` → everything normal. Catch `Exception`, never bare `except:`.
- **EAFP** ("easier to ask forgiveness than permission": try/except) vs **LBYL** (check first). Python favors EAFP; avoids TOCTOU races.
- Custom exceptions: subclass `Exception`, group under a package base (`class AppError(Exception)`), add context attributes.
- `raise` (bare) re-raises preserving traceback; `raise ... from None` hides context.
- `ExceptionGroup` and `except*` (3.11+) for concurrent errors.
- Don't use exceptions for ordinary control flow in hot loops (slower than `if`), but a successful `try` is near free.
- `assert` is for invariants/dev checks; it's stripped with `-O` — don't validate user input with it.
- `traceback`, `logging.exception()` (logs stack), `contextlib.suppress`.

### 5.3 Logging
```python
import logging
logger = logging.getLogger(__name__)          # module-level logger; never print() in libs

logging.basicConfig(level=logging.INFO,
                    format="%(asctime)s %(levelname)s %(name)s %(message)s")
logger.info("user %s logged in", user_id)     # lazy %-formatting: no cost if level disabled
try: ...
except Exception: logger.exception("failed to process %s", job_id)   # includes traceback
```
Levels: DEBUG < INFO < WARNING < ERROR < CRITICAL. Libraries add `NullHandler` and don't configure handlers; applications configure once at the entry point (dictConfig). Structured (JSON) logs + correlation IDs in services.

### 5.4 Testing with pytest
```python
# test_calc.py
import pytest
from calc import divide

def test_divide():
    assert divide(6, 3) == 2

def test_divide_by_zero():
    with pytest.raises(ZeroDivisionError, match="division"):
        divide(1, 0)

@pytest.mark.parametrize("a,b,expected", [(4, 2, 2), (9, 3, 3), (0, 5, 0)])
def test_many(a, b, expected):
    assert divide(a, b) == expected

@pytest.fixture
def db(tmp_path):                       # fixtures: setup/teardown + dependency injection
    conn = connect(tmp_path / "t.db")
    yield conn
    conn.close()
```
- Fixture scopes: `function` (default), `class`, `module`, `session`. `conftest.py` shares fixtures. Built-ins: `tmp_path`, `monkeypatch`, `capsys`, `caplog`.
- **Mocking**: `unittest.mock.patch("module.where_its_used.name")` — patch where the name is *looked up*, not where it's defined.
  ```python
  from unittest.mock import patch, MagicMock
  @patch("app.service.requests.get")
  def test_fetch(mock_get):
      mock_get.return_value = MagicMock(status_code=200, json=lambda: {"ok": True})
      assert fetch() == {"ok": True}
      mock_get.assert_called_once_with("https://x", timeout=5)
  ```
- Prefer fakes/in-memory implementations and dependency injection to deep mocking; mock only boundaries (network, clock, DB).
- Test pyramid: many unit tests, some integration (real DB in Docker/testcontainers), few end-to-end. Property-based testing with **hypothesis**. Coverage with `pytest-cov` (a metric, not a goal).
- Async tests: `pytest-asyncio` / `anyio`. API tests: FastAPI `TestClient`.
- Good tests: deterministic, isolated, fast, one behavior per test, readable names (arrange–act–assert).

### 5.5 Packaging, environments, tooling
- **Virtual environments**: `python -m venv .venv`; tools: `uv`, `poetry`, `pipenv`, `pip-tools`. Pin dependencies (lock file) for apps; use ranges for libraries.
- `pyproject.toml` (PEP 621) holds metadata, dependencies, and tool config; build with `python -m build`; publish wheels to PyPI/private index.
- Project layout: `src/pkg/`, `tests/`, `pyproject.toml`, `README.md`. `__init__.py` marks packages (namespace packages don't need it). Run modules with `python -m pkg.module`.
- Quality tooling: **ruff** (lint + format), black, isort, mypy/pyright, pre-commit, CI (GitHub Actions).
- 12-factor config: environment variables / `pydantic-settings`, never commit secrets.
- Docker: slim base image, multi-stage builds, non-root user, `PYTHONUNBUFFERED=1`.

### 5.6 Idioms checklist ("Pythonic")
`enumerate`, `zip`, unpacking (`a, *rest = xs`), f-strings (`f"{x!r:>10}"`), `pathlib.Path`, `with` for resources, `dict.get/setdefault`, `defaultdict`/`Counter`, `any/all`, `sorted(key=...)`, truthiness, `__name__ == "__main__"`, `argparse`/`typer`, `json`, `datetime` with timezone-aware objects (`datetime.now(timezone.utc)`), `enum.Enum`, avoid global state.

---

## Part 2 — Homework / Exercises (not timed)

1. Add full type hints to a small module (e.g., your own CSV parser); run `mypy --strict` and fix errors. Note which errors were real bugs.
2. Write a `Protocol` `Notifier` with `send(msg: str) -> None`, two implementations (`EmailNotifier`, `FakeNotifier`), and a service function typed to accept the Protocol. Test using the fake.
3. Custom exception hierarchy for an order system: `OrderError` → `OutOfStock`, `PaymentFailed(code)`. Show chaining with `from`.
4. Write pytest tests for a `fizzbuzz(n)` using `parametrize`, and for a function calling `requests.get` using `patch` — then rewrite it with a fake HTTP client injected.
5. Configure `logging` via `dictConfig` with console + rotating file handler; add a JSON formatter.
6. Create a package with `pyproject.toml`, `src/` layout, an entry-point script (`[project.scripts]`), build a wheel, install it into a fresh venv.
7. Spot the bugs:
   ```python
   def load(path):
       try:
           f = open(path)
           data = json.load(f)
       except:
           print("error")
       return data
   ```

<details><summary>Solutions</summary>

7. Bare `except` hides all errors (even KeyboardInterrupt); file never closed; `data` unbound after failure (`UnboundLocalError` masks the real error); `print` rather than logging; no encoding. Fix: `with open(path, encoding="utf-8") as f: return json.load(f)` and catch `(OSError, json.JSONDecodeError)` where you can handle them, otherwise let it propagate (maybe wrapped with `raise ConfigError(...) from e`).
4. Fake: `class FakeHttp: def get(self, url, timeout): return FakeResp()` and pass into the function — no patching required.
</details>

---

## Part 3 — Quiz

**Q1.** Are type hints enforced at runtime? What enforces them?

**Q2.** `list[Dog]` passed to `def f(xs: list[Animal])` — does mypy accept it? Why/why not? What would you use instead?

**Q3.** What's the difference between `except Exception` and a bare `except:`?

**Q4.** What does `raise NewError() from e` do vs `raise NewError()` inside an `except` block?

**Q5.** Where should you patch: `patch("requests.get")` or `patch("myapp.client.requests.get")`? Which target is "where used"?

**Q6.** What's the output order?
```python
def f():
    try:
        print("a"); return "r"
    finally:
        print("b")
print(f())
```
**Q7.** Why use `logger.info("x=%s", x)` instead of an f-string?

**Q8.** Name 3 pytest features that make tests concise and DRY.

**Q9.** What's an `Optional[int]` equivalent in modern syntax? Does `Optional` mean "optional argument"?

**Q10.** `assert` vs raising `ValueError` for validating user input?

<details><summary>Answers</summary>

1. No. Static checkers (mypy/pyright) enforce; Pydantic/FastAPI/`beartype` use them at runtime for validation.
2. Rejected: `list` is invariant (f could append a `Cat`). Use `Sequence[Animal]` (covariant, read-only) or a TypeVar bound.
3. `except Exception` skips `KeyboardInterrupt`, `SystemExit`, `GeneratorExit` (BaseException-only). Bare catches everything — including Ctrl-C.
4. `from e` sets `__cause__` explicitly ("direct cause of"); without it, implicit `__context__` ("during handling of …") is shown. `from None` suppresses.
5. `patch("myapp.client.requests.get")` — patch the lookup location in the module under test (or patch `myapp.client.requests` object). Patching the source module works only if the code looks it up at call time.
6. `a`, `b`, `r`. `finally` runs before the function returns.
7. Lazy formatting: string is built only if the record is emitted; also lets handlers group by message template (Sentry), avoids formatting errors on bad args.
8. Fixtures, `parametrize`, plain `assert` with introspection, `monkeypatch`, `tmp_path`, `raises`, marks, `conftest.py`.
9. `int | None`. No — it means value may be `None`; optional args are those with defaults.
10. `assert` can be disabled with `-O` and signals programmer error. Use explicit exceptions for external input.
</details>

---

## Part 4 — Interview Questions

**I1. How do you use type hints in a real project? Benefits and limits?**
<details><summary>Model answer</summary>
Annotate public APIs first, run mypy/pyright in CI (strict for new modules, gradual for legacy), use `Protocol` for seams, `TypedDict`/dataclasses/Pydantic for data shapes. Benefits: catches bugs early, self-documenting, better IDE refactoring, enables FastAPI/Pydantic. Limits: not enforced at runtime, `Any` leaks, third-party stubs gaps, complex generics can become unreadable. Cite a bug type-checking caught (e.g., `None` not handled).
</details>

**I2. EAFP vs LBYL?**
<details><summary>Model answer</summary>
EAFP: attempt the operation and handle exceptions (`try: d[k] except KeyError`). Fits Python's duck typing, avoids races between check and use (file existence). LBYL: check preconditions first; fine when failure is common and the check is cheap or when exceptions would hide bugs. Use `dict.get`/`in` when absence is normal.
</details>

**I3. How do you design error handling for a service?**
<details><summary>Model answer</summary>
Define an app exception hierarchy; catch low-level errors at boundaries and translate (DB error → `RepositoryError`) with `from e`; handle only what you can recover from; let others propagate to a top-level handler (HTTP middleware) which logs with stack trace + correlation ID and returns a sanitized response; retry only idempotent/transient failures with backoff; never swallow silently; use `finally`/context managers for cleanup; distinguish client errors (4xx) from server errors (5xx).
</details>

**I4. How do you test code that calls external services or the database?**
<details><summary>Model answer</summary>
Isolate I/O behind interfaces; inject dependencies; unit-test logic with fakes; mock at boundaries only (time, HTTP) with `patch`/`responses`/`respx`; integration-test against a real Postgres (Docker/testcontainers, transaction rollback per test via fixtures); contract tests for APIs; keep E2E few. Use factories for data, freeze time (`freezegun`/`time-machine`), seed randomness. Mention flaky-test hygiene.
</details>

**I5. What is dependency injection in Python and why is it useful?**
<details><summary>Model answer</summary>
Passing collaborators (clients, repos, clocks) into functions/classes instead of constructing/importing them inside. Increases testability and flexibility. Done with plain constructor/arg passing; FastAPI uses `Depends`. No framework needed typically.
</details>

**I6. What goes in `pyproject.toml`, and how do you manage dependencies reproducibly?**
<details><summary>Model answer</summary>
Project metadata, dependencies/optional extras, entry points, build backend, tool configs (ruff, mypy, pytest). Reproducibility via lock files (`uv.lock`, `poetry.lock`, or pip-compile `requirements.txt` with hashes), pinned Python version, a venv per project, and building the Docker image from the lock file. Separate dev/test dependency groups.
</details>

**I7. How do you structure and configure logging in an application?**
<details><summary>Model answer</summary>
Module-level `getLogger(__name__)`; configure once at startup (`dictConfig`), levels per environment, JSON formatter for aggregation, include request/trace IDs (contextvars + filter), avoid logging secrets/PII, use `logger.exception` in handlers, don't log-and-raise repeatedly (duplicates), sample or rate-limit noisy logs.
</details>

**I8 (Advanced). Explain `Protocol` vs `ABC`, and `TypeVar` bounds.**
<details><summary>Model answer</summary>
`ABC`: nominal — classes must inherit; enforces abstract methods at instantiation (runtime). `Protocol`: structural — matches by shape, checked statically (runtime check only with `@runtime_checkable`, shallow). Use Protocol to decouple from implementations and type third-party classes. `TypeVar("T", bound=Comparable)` restricts generics to types satisfying the bound, preserving the specific type through the function (`def max_of(a: T, b: T) -> T`).
</details>

**Live-coding warm-up:** Write a retry decorator fully typed with `ParamSpec`/`TypeVar` so type checkers preserve the wrapped signature; then a pytest test for it using a flaky stub that fails twice.
