# Session 4 — Concurrency: Threads, Processes, asyncio (Advanced)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 4.1 Vocabulary
- **Concurrency**: managing multiple tasks making progress (interleaving). **Parallelism**: tasks literally running at the same time (multiple cores).
- **I/O-bound**: waiting on network/disk/DB → threads or asyncio. **CPU-bound**: computing → processes (or native code).
- Decision table:

| Workload | Best tool |
|---|---|
| Many network calls, moderate (≤ hundreds) | `ThreadPoolExecutor` |
| Thousands of concurrent connections | `asyncio` |
| Heavy number crunching | `ProcessPoolExecutor` / NumPy / native |
| Blocking legacy library inside async app | `asyncio.to_thread` / `run_in_executor` |

### 4.2 Threading
```python
from concurrent.futures import ThreadPoolExecutor, as_completed
import requests

def fetch(url):
    return url, requests.get(url, timeout=5).status_code

with ThreadPoolExecutor(max_workers=10) as pool:
    futures = [pool.submit(fetch, u) for u in urls]
    for fut in as_completed(futures):
        try: print(fut.result())
        except Exception as e: print("fail", e)
    # or: results = list(pool.map(fetch, urls))   # preserves input order
```
Shared state needs synchronization:

```python
import threading
lock = threading.Lock()
counter = 0
def work():
    global counter
    for _ in range(100_000):
        with lock:            # without the lock: lost updates (race condition)
            counter += 1
```
Primitives: `Lock`, `RLock` (re-entrant), `Semaphore` (limit concurrency), `Event` (signal), `Condition`, `Barrier`, `queue.Queue` (thread-safe producer/consumer — **prefer sharing via queues**).
Hazards: race conditions, **deadlock** (lock ordering; acquire in a consistent order, use timeouts), starvation. Daemon threads die at exit without cleanup.
Because of the GIL (Session 3), threads don't speed up CPU-bound Python.

### 4.3 Multiprocessing
```python
from concurrent.futures import ProcessPoolExecutor

def heavy(n): return sum(i*i for i in range(n))

if __name__ == "__main__":                   # REQUIRED on spawn platforms (macOS/Windows)
    with ProcessPoolExecutor() as pool:
        print(list(pool.map(heavy, [10**7]*4)))
```
- Each process has its own interpreter and memory → true parallelism, no GIL contention.
- Arguments/results are **pickled** → must be picklable (no lambdas/local functions); overhead for big data. Use `shared_memory`, memory-mapped files, or Arrow for large arrays.
- Start methods: `fork` (Linux default pre-3.14; fast, unsafe with threads), `spawn` (macOS/Windows; safe, slower), `forkserver`.
- IPC: `Queue`, `Pipe`, `Manager`. Process crash isolation is a plus.

### 4.4 asyncio: cooperative single-threaded concurrency
An **event loop** runs coroutines; each `await` yields control while waiting on I/O.

```python
import asyncio, aiohttp

async def fetch(session, url):
    async with session.get(url) as r:
        return await r.text()

async def main(urls):
    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(*(fetch(session, u) for u in urls),
                                       return_exceptions=True)
    return results

asyncio.run(main(urls))
```
Core ideas:
- `async def` creates a coroutine function; calling it returns a coroutine **object** (nothing runs until awaited/scheduled). Forgetting `await` → "coroutine was never awaited" warning.
- `await` only works inside `async def`. `asyncio.run()` is the entry point.
- **Tasks**: `asyncio.create_task(coro)` schedules concurrently. Keep a reference to tasks (weak refs in loop → can be garbage collected!).
- **Structured concurrency (3.11+)**: `async with asyncio.TaskGroup() as tg: tg.create_task(...)` — if one fails, siblings are cancelled and errors raised as `ExceptionGroup` (`except*`).
- `asyncio.gather` (collect results) · `wait_for` / `asyncio.timeout()` (timeouts) · `Semaphore` (limit concurrency) · `Queue` (producers/consumers) · `Lock`/`Event` (async versions).
- **Never block the loop**: `time.sleep`, `requests.get`, heavy CPU inside a coroutine stalls everything. Use `await asyncio.sleep`, async libraries (aiohttp, httpx, asyncpg), or `await asyncio.to_thread(blocking_fn)`.
- Cancellation: `task.cancel()` raises `CancelledError` inside the coroutine at the next `await`; always `finally`-cleanup and re-raise.
- Async iterators/generators: `async for`, `async with`, `__aiter__/__anext__`.

```python
sem = asyncio.Semaphore(20)
async def limited(session, url):
    async with sem:
        return await fetch(session, url)
```

### 4.5 threads vs asyncio vs processes: trade-offs
| | Threads | asyncio | Processes |
|---|---|---|---|
| Parallel CPU | no (GIL) | no | yes |
| Memory | low-medium | very low | high |
| Switching | OS preemptive (races anywhere) | cooperative (only at `await`) | OS |
| Shared state | easy, risky | easy, safe between awaits | needs IPC |
| Ecosystem | any blocking lib | needs async libs | picklable data |

asyncio's advantage: race conditions only occur across `await` points → easier reasoning, scales to 10k+ connections.

### 4.6 Practical patterns
- **Producer/consumer** with `queue.Queue` / `asyncio.Queue` and sentinel (`None`) or `join()`/`task_done()`.
- **Retries with backoff + jitter**; **timeouts everywhere**; **bounded concurrency** (semaphores, pools) to avoid overwhelming downstream.
- **Idempotency** when retrying.
- Web servers: Gunicorn/uvicorn workers (processes) × async handlers; FastAPI `async def` vs `def` endpoints (sync ones run in a threadpool).
- Distributed work: Celery/RQ/Dramatiq task queues for background jobs.

---

## Part 2 — Homework / Exercises (not timed)

1. Reproduce a race condition: 4 threads incrementing a shared counter 1M times each; observe wrong total; fix with `Lock`; then fix with a design that avoids shared state (each thread returns its count, main sums).
2. Fetch 50 URLs (use `https://httpbin.org/delay/1`) sequentially, with `ThreadPoolExecutor(10)`, and with asyncio + `httpx.AsyncClient` + `Semaphore(10)`. Compare timings.
3. CPU-bound benchmark: compute primes under 5M with a sequential loop, `ThreadPoolExecutor`, `ProcessPoolExecutor`. Explain results.
4. Build an async producer/consumer pipeline: 1 producer, 3 consumers, `asyncio.Queue(maxsize=10)`; consumers process with `await asyncio.sleep(random)`; clean shutdown using sentinels.
5. Write `async def with_timeout_and_retry(coro_fn, retries, timeout)` using `asyncio.timeout`.
6. Create a deadlock with two locks acquired in opposite order in two threads; fix it by ordering.
7. Fix the bug:
   ```python
   async def main():
       results = [fetch(u) for u in urls]     # what is wrong?
       print(results)
   ```

<details><summary>Solutions</summary>

1. Final count < 4,000,000 without lock. Lock gives correct but serialized. Best: `sum(pool.map(count_chunk, ...))`.
2. Sequential ≈ 50 s; threads(10) ≈ 5–6 s; asyncio(10) ≈ 5–6 s; raise concurrency to see asyncio scale further.
3. Threads ≈ sequential (GIL); processes ≈ N× faster (up to cores), less after pickling/startup overhead.
4. Put N sentinels (one per consumer) or use `await queue.join()` then cancel consumers.
5. ```python
   async def run(coro_fn, retries=3, timeout=2):
       for i in range(retries):
           try:
               async with asyncio.timeout(timeout):
                   return await coro_fn()
           except TimeoutError:
               if i == retries - 1: raise
               await asyncio.sleep(2 ** i)
   ```
6. Thread A: lock1→lock2; B: lock2→lock1 → deadlock. Fix: always lock in a global order (e.g., by `id`).
7. Missing `await`/`gather`: `results = await asyncio.gather(*(fetch(u) for u in urls))`; the original list holds un-run coroutine objects.
</details>

---

## Part 3 — Quiz

**Q1.** You must download 1,000 files. Which tool(s) and why?

**Q2.** You must resize 10,000 images (CPU-heavy, using Pillow). Which tool?

**Q3.** What does this do?
```python
import time, asyncio
async def main():
    time.sleep(2)
    await asyncio.sleep(0)
```
What's wrong if 100 such tasks are run concurrently?

**Q4.** What's the difference between `asyncio.gather` and `asyncio.TaskGroup`?

**Q5.** Why does multiprocessing code need `if __name__ == "__main__":`?

**Q6.** Calling `fetch(url)` where `fetch` is `async def` — what's returned?

**Q7.** Name two ways threads can share data safely.

**Q8.** What's a deadlock, and what four conditions cause it (Coffman)? Name a prevention technique.

**Q9.** In asyncio, can two coroutines race on a shared variable? When?

<details><summary>Answers</summary>

1. I/O-bound: `ThreadPoolExecutor` (simple) or asyncio + httpx/aiohttp with a semaphore (scales better). Add timeouts, retries, and bounded concurrency.
2. `ProcessPoolExecutor` (CPU-bound; or batch in chunks to reduce pickling). Pillow releases the GIL in places, but processes are the safe choice.
3. `time.sleep` blocks the entire event loop for 2 s (tasks run sequentially → 200 s total instead of 2 s). Use `await asyncio.sleep(2)` or `to_thread`.
4. `gather` returns results in order; on first exception (without `return_exceptions`) others keep running un-cancelled. `TaskGroup` (3.11+) cancels siblings on failure and aggregates errors in `ExceptionGroup`; clear scope & lifetimes.
5. With `spawn`/`forkserver` the child re-imports the main module; without the guard, it would recursively spawn processes (RuntimeError).
6. A coroutine object; nothing runs until awaited or wrapped in a task.
7. `queue.Queue`; `Lock`/`RLock` around critical sections; immutable data; thread-local storage; `Event`/`Condition` for signaling.
8. Threads waiting on each other cyclically. Mutual exclusion, hold-and-wait, no preemption, circular wait. Prevent by consistent lock ordering, timeouts on `acquire`, or avoiding nested locks.
9. Yes, but only across `await` points (e.g., read value, `await`, write back). Without an `await` in between, code runs atomically w.r.t. other coroutines.
</details>

---

## Part 4 — Interview Questions

**I1. Threads vs processes vs asyncio — how do you choose?**
<details><summary>Model answer</summary>
Classify the bottleneck. I/O-bound with a manageable number of concurrent operations → threads (simple, works with blocking libs). Very high concurrency or when you control the stack → asyncio (lightweight tasks, deterministic switch points). CPU-bound → processes (bypass GIL) or push work to native libs; in a web service, scale with multiple worker processes. Mention hybrid: async app + `to_thread` for blocking calls; process pool for CPU parts. Always measure; avoid unbounded concurrency; set timeouts.
</details>

**I2. How does the event loop work?**
<details><summary>Model answer</summary>
Single thread; maintains a queue of ready callbacks and a selector (epoll/kqueue) for I/O readiness plus a timer heap. Each iteration: poll I/O (with timeout until next timer), move ready callbacks, run them. A coroutine runs until it `await`s something not ready; the loop then switches to another task. When the awaited future completes it schedules the task to resume. If any code blocks, the whole loop stalls — hence "never block the loop."
</details>

**I3. What is a race condition? Show one and fix it.**
<details><summary>Model answer</summary>
Outcome depends on unsynchronized interleavings of operations on shared state. `counter += 1` compiles to load/add/store; two threads interleave and lose updates. Fixes: `Lock` around the critical section, atomic data structures (`queue.Queue`, `collections.deque` append/pop are atomic), message passing, thread confinement, or immutability. Also mention check-then-act (TOCTOU) and DB-level equivalents (use `SELECT ... FOR UPDATE`, unique constraints, atomic `UPDATE ... SET x = x + 1`).
</details>

**I4. Explain `async`/`await`. What happens if you call a coroutine function without `await`?**
<details><summary>Model answer</summary>
`async def` defines a coroutine function; calling it creates a coroutine object (no execution). `await` suspends the current coroutine until the awaitable completes, letting the loop run others. Without await you get an unexecuted coroutine and a `RuntimeWarning: coroutine ... was never awaited`; results are lost. To run concurrently: `create_task`, `gather`, or `TaskGroup`.
</details>

**I5. How do you cancel tasks and handle timeouts safely in asyncio?**
<details><summary>Model answer</summary>
`task.cancel()` injects `CancelledError` at the next `await`; code should clean up in `finally` and re-raise it (don't swallow). Use `asyncio.timeout(sec)` (3.11+) or `wait_for`. With `TaskGroup`, a failure cancels siblings. Shield critical sections with `asyncio.shield` if needed. Ensure resources use `async with`.
</details>

**I6. How would you design a web scraper that fetches 100k pages politely and reliably?**
<details><summary>Model answer</summary>
asyncio + httpx/aiohttp with a shared client (connection pooling), `Semaphore`/per-host limits, rate limiting, `robots.txt`, timeouts, retries with exponential backoff + jitter for 429/5xx, dedupe via set/Bloom filter, work queue with bounded size (backpressure), persisting progress (so it's resumable), structured logging/metrics, graceful shutdown on signals, parse in a process pool if CPU-heavy, store results in batches.
</details>

**I7. How is data shared between processes? Cost?**
<details><summary>Model answer</summary>
Serialized (pickle) over pipes/queues; `multiprocessing.shared_memory`/`mmap` for zero-copy buffers; `Manager` proxies (slow); external stores (Redis, DB). Pickling cost dominates for large objects → chunk work, pass file paths/IDs instead of data, or use shared memory/Arrow.
</details>

**I8 (Advanced). What's changing with the GIL in newer Python versions?**
<details><summary>Model answer</summary>
PEP 684 (3.12): per-interpreter GIL — subinterpreters can run in parallel (`concurrent.interpreters` arrives in later versions). PEP 703: free-threaded build (no GIL) shipped as experimental in 3.13 and increasingly supported afterward; single-thread performance cost has been shrinking but extension modules need to be thread-safe, and libraries must audit shared state. Practical stance: don't depend on it yet in production unless your dependencies declare support; continue to use processes/async.
</details>

**Live-coding warm-up:** Write an async rate limiter (token bucket) allowing N requests per second; then use it with `gather` over 100 URLs.
