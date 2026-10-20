# Session 6 — Standard Library Power Tools, Performance & Coding-Interview Patterns (Intermediate/Advanced)

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 6.1 `collections` essentials
```python
from collections import Counter, defaultdict, deque, OrderedDict, namedtuple, ChainMap

Counter("abracadabra").most_common(2)      # [('a', 5), ('b', 2)]
Counter(a) - Counter(b)                    # multiset arithmetic

graph = defaultdict(list); graph[u].append(v)     # no KeyError, auto-create

q = deque(maxlen=3); q.append(1); q.appendleft(0); q.popleft()   # O(1) both ends
```
| Structure | Use for | Key ops |
|---|---|---|
| `list` | ordered, indexed | append O(1)*, insert/pop(0) O(n) |
| `deque` | queues, sliding window | O(1) at both ends, O(n) middle index |
| `dict` | mapping | O(1) avg |
| `set` | membership, dedupe, set algebra (`& | - ^`) | O(1) avg |
| `heapq` | priority queue (min-heap) | push/pop O(log n), heapify O(n) |
| `bisect` | sorted list search/insert | O(log n) search, O(n) insert |

### 6.2 `itertools` & `functools`
```python
from itertools import chain, islice, groupby, product, permutations, combinations, accumulate, zip_longest, count, cycle, pairwise, batched
list(pairwise([1, 2, 4, 7]))                  # [(1,2),(2,4),(4,7)]
list(accumulate([1, 2, 3, 4]))                # [1,3,6,10] running sum
for k, g in groupby(sorted(rows, key=f), key=f): ...   # input must be SORTED by the key
list(combinations("ABC", 2))                  # AB AC BC
```
`functools`: `lru_cache/cache`, `partial`, `reduce`, `cached_property`, `singledispatch`, `total_ordering`.

### 6.3 `heapq` and priority queues
```python
import heapq
h = []; heapq.heappush(h, (priority, item)); heapq.heappop(h)
heapq.nlargest(3, data)  # k largest in O(n log k)
# max-heap trick: push negatives. Tie-breaking with non-comparable items: add a counter (priority, count, item).
```
Top-K, merge K sorted lists (`heapq.merge`), Dijkstra, scheduling.

### 6.4 Sorting
`sorted(xs, key=lambda r: (r.dept, -r.salary))` — stable Timsort O(n log n), O(n) on nearly sorted. `key` is called once per element. `list.sort()` in place returns `None`. `functools.cmp_to_key` for comparator style.

### 6.5 Strings, files, paths, text
```python
from pathlib import Path
for p in Path("logs").glob("**/*.log"):
    text = p.read_text(encoding="utf-8")
import re
m = re.search(r"(?P<year>\d{4})-(?P<mon>\d{2})", s); m["year"]
re.findall / re.sub / re.compile  # compile for reuse; raw strings r""; beware catastrophic backtracking
import json, csv; json.dumps(obj, default=str); csv.DictReader(f)
```
`str.join`, `split`, `partition`, `startswith(tuple)`, `str.format_map`; large files: iterate lines, don't `read()`.

### 6.6 Date/time & numbers
- `datetime.now(timezone.utc)`; store UTC, convert at the edges; `zoneinfo.ZoneInfo("America/New_York")`. Naive vs aware datetimes cannot be compared (TypeError).
- `Decimal("0.1")` from strings, not floats; `fractions.Fraction`; `statistics`, `random` (not for security — use `secrets`), `math.isclose`.

### 6.7 Profiling and performance
```python
import timeit, cProfile, pstats
timeit.timeit("sum(range(1000))", number=10_000)
cProfile.run("main()", "out.prof"); pstats.Stats("out.prof").sort_stats("cumtime").print_stats(10)
```
Tools: `cProfile` (functions), `line_profiler` (lines), `py-spy` (sampling a live process, no code change), `tracemalloc` (memory), `scalene`.
Playbook: **measure → find the hot spot → pick algorithmic win → rely on built-ins/C → vectorize → parallelize → native**.
Typical wins: `set` for lookups; `"".join`; local variable binding in hot loops; avoid repeated `list.pop(0)`/`insert(0)`; generator instead of list; `dict`-based indexing instead of nested scan; batch DB/API calls; caching.

### 6.8 Complexity cheat-sheet
| Op | Complexity |
|---|---|
| `list[i]`, `len`, `append`, `pop()` | O(1) |
| `list.insert(0,x)`, `pop(0)`, `x in list`, `list.index` | O(n) |
| `dict/set` get/set/in | O(1) avg |
| `sorted`, `sort` | O(n log n) |
| `heappush/pop` | O(log n) |
| `bisect` | O(log n) |
| string `+=` in loop | O(n²) worst case |
| slicing `a[i:j]` | O(j−i) |

### 6.9 Coding-interview patterns (with Python idioms)
1. **Hash map / counting** — two-sum, anagram groups, first unique char.
   ```python
   def two_sum(nums, target):
       seen = {}
       for i, x in enumerate(nums):
           if target - x in seen: return seen[target - x], i
           seen[x] = i
   ```
2. **Two pointers / sliding window** — longest substring without repeats:
   ```python
   def longest_unique(s):
       last, start, best = {}, 0, 0
       for i, ch in enumerate(s):
           if last.get(ch, -1) >= start: start = last[ch] + 1
           last[ch] = i; best = max(best, i - start + 1)
       return best
   ```
3. **Stack** — valid parentheses, next greater element (monotonic stack).
4. **Heap** — top-K frequent: `heapq.nlargest(k, Counter(xs).items(), key=lambda kv: kv[1])`.
5. **Binary search** — `bisect_left`; "binary search on answer" for monotonic predicates.
6. **BFS/DFS** — graphs/grids; BFS with `deque` for shortest path in unweighted graphs; iterative DFS to avoid recursion limit (default ~1000: `sys.setrecursionlimit` carefully).
   ```python
   def bfs(graph, s):
       dist, q = {s: 0}, deque([s])
       while q:
           u = q.popleft()
           for v in graph[u]:
               if v not in dist: dist[v] = dist[u] + 1; q.append(v)
       return dist
   ```
7. **Dynamic programming** — define state, recurrence, base case; memoize with `@cache`, then bottom-up.
   ```python
   @cache
   def climb(n): return n if n <= 2 else climb(n-1) + climb(n-2)
   ```
8. **Intervals** — sort by start, merge. 9. **Linked list / tree** — recursion, iterative with explicit stack; `ListNode` pointer reversal. 10. **Backtracking** — subsets, permutations (or use `itertools`, but know the manual version).

How to run a coding interview: clarify inputs/edge cases → give brute force + complexity → optimize → code cleanly → test with examples and edges (empty, one element, duplicates, negatives, large) → state final complexity.

---

## Part 2 — Homework / Exercises (not timed)

1. Implement and test: `group_anagrams`, `top_k_frequent`, `merge_intervals`, `valid_parentheses`, `LRU cache`, `binary_search` (with `bisect`), `min_window_substring`.
2. Given a CSV of 1M rows `(user_id, ts, amount)`, compute per-user totals and the top 10 users with only the stdlib, streaming (`csv.reader`, `defaultdict(float)`, `heapq.nlargest`). Compare with pandas.
3. Profile a deliberately slow function (nested loops finding duplicate pairs) with `cProfile`; fix via a set; report speedup.
4. Implement Dijkstra with `heapq` for a weighted graph; handle stale heap entries.
5. Parse log lines like `2024-05-01 12:03:55 ERROR [db] timeout after 30s` with a named-group regex and aggregate errors per module with `Counter`.
6. Write a generic `batched(iterable, n)` (pre-3.12) and a sliding-window max with `deque` in O(n).

<details><summary>Solution sketches</summary>

1. `group_anagrams`: key = `tuple(sorted(w))` or 26-count tuple into `defaultdict(list)`. `merge_intervals`: sort by start; extend last if `start <= last_end`.
4. Skip when popped `d > dist[u]`:
   ```python
   def dijkstra(g, s):
       dist = {s: 0}; h = [(0, s)]
       while h:
           d, u = heapq.heappop(h)
           if d > dist.get(u, inf): continue
           for v, w in g[u]:
               nd = d + w
               if nd < dist.get(v, inf): dist[v] = nd; heapq.heappush(h, (nd, v))
       return dist
   ```
6. Sliding window max: deque of indices with decreasing values; pop back while smaller than the new element; pop front when out of window.
</details>

---

## Part 3 — Quiz

**Q1.** Complexity of `x in some_list` vs `x in some_set` for n = 10⁶, and why?

**Q2.** What does `groupby` require of its input?

**Q3.** Which structure for a FIFO queue with frequent `popleft`? Why not a list?

**Q4.** Output?
```python
from collections import Counter
print(Counter("aabbbc").most_common(1))
```
**Q5.** `heapq` is a min-heap. How do you get a max-heap? How do you heap non-comparable payloads?

**Q6.** Complexity: `s = ""; for w in words: s += w` vs `"".join(words)`?

**Q7.** `sorted(["b", "A", "c"])` → ? How to sort case-insensitively?

**Q8.** Why use `datetime.now(timezone.utc)` rather than `datetime.utcnow()`?

**Q9.** Time and space complexity of your sliding-window "longest substring without repeats" solution?

**Q10.** `a.sort()` returns what?

<details><summary>Answers</summary>

1. O(n) vs O(1) average: list scans linearly; set hashes the value.
2. Sorted by the grouping key (it groups only *consecutive* equal keys).
3. `collections.deque` — O(1) `popleft`; `list.pop(0)` shifts all elements, O(n).
4. `[('b', 3)]`
5. Push negated priorities (or wrap in a class with reversed `__lt__`). For payloads: `(priority, counter, payload)` so ties never compare payloads.
6. Repeated concatenation can be O(n²) (CPython sometimes optimizes in-place for refcount 1, but don't rely on it); join is O(n).
7. `['A', 'b', 'c']` (uppercase sorts before lowercase). `sorted(xs, key=str.casefold)`.
8. `utcnow()` returns a *naive* datetime (and is deprecated since 3.12); aware datetimes prevent mixing/timezone bugs.
9. O(n) time, O(min(n, alphabet)) space.
10. `None` (in-place). `sorted()` returns a new list.
</details>

---

## Part 4 — Interview Questions

**I1. Walk through solving "top K frequent elements." Give two or three approaches.**
<details><summary>Model answer</summary>
Count with `Counter` (O(n)). (a) Sort items by count: O(u log u). (b) Heap of size k: `heapq.nlargest(k, ...)` O(u log k) — good when k ≪ u. (c) Bucket sort by frequency: O(n). Edge cases: ties, k > unique count, empty input. Then discuss streaming variants (Count-Min Sketch / Space-Saving) if data doesn't fit in memory.
</details>

**I2. How would you find duplicates in a 50 GB file on a machine with 8 GB RAM?**
<details><summary>Model answer</summary>
Can't hold a set of everything. Options: hash-partition lines into N bucket files by `hash(line) % N` (so duplicates land in the same bucket), then dedupe each bucket in memory; or external sort + adjacent compare (`sort -u`); or Bloom filter for approximate pre-filter; or store line hashes (8–16 bytes each) instead of lines then verify collisions. Mention streaming reads, chunk sizes, and I/O cost.
</details>

**I3. Python list vs tuple vs deque vs array vs numpy array?**
<details><summary>Model answer</summary>
List: dynamic array of pointers, heterogeneous, mutable. Tuple: immutable, hashable if elements are, slightly smaller/faster; used as records/keys. Deque: doubly-linked blocks, O(1) ends. `array.array`: typed compact numeric storage. NumPy: contiguous typed n-dim arrays with vectorized C operations — orders of magnitude faster for numeric work, but fixed dtype.
</details>

**I4. Explain how you'd profile and speed up a slow endpoint/script.**
<details><summary>Model answer</summary>
Reproduce with realistic data; profile (cProfile + snakeviz, or `py-spy` in prod); examine wall vs CPU time to see whether it's I/O (DB, HTTP) or compute. Typical: fix N+1 queries and add indexes, cache, batch, algorithmic improvements, move invariant work out of loops, use generators/streaming, vectorize or parallelize, then micro-opts. Verify gains with benchmarks and guard with a performance test/regression metric.
</details>

**I5. Implement an LRU cache. What data structures and why?**
<details><summary>Model answer</summary>
Hash map for O(1) lookup + doubly linked list for O(1) recency updates and eviction (head = most recent). In Python, `OrderedDict` with `move_to_end` and `popitem(last=False)` does it. Mention thread safety (lock), TTL variants, `functools.lru_cache` for function memoization, and why `list` is wrong (O(n)).
</details>

**I6. Recursion in Python — limits and alternatives?**
<details><summary>Model answer</summary>
Default limit ~1000 frames, no tail-call optimization. Convert to iteration with explicit stack/queue, use memoization to avoid exponential recursion, or `lru_cache`. Raising the limit risks segfaults on deep stacks. Bottom-up DP avoids stack depth entirely.
</details>

**I7. Regex: pitfalls and when not to use?**
<details><summary>Model answer</summary>
Greedy vs lazy quantifiers, catastrophic backtracking with nested quantifiers (ReDoS), anchors and multiline flags, unreadable patterns (use `re.VERBOSE`, named groups). Don't parse HTML/JSON/CSV/nested structures with regex; use proper parsers.
</details>

**I8 (Advanced). How do `heapq.merge`, generators, and `itertools` help process data larger than memory?**
<details><summary>Model answer</summary>
They operate lazily on iterators, keeping O(1) or O(k) memory: `heapq.merge(*sorted_streams)` lazily merges pre-sorted inputs (external merge sort), `islice` slices streams, `groupby` aggregates sorted streams, `chain` concatenates files. Combined with chunked reads and generators you build streaming ETL pipelines with bounded memory.
</details>

**Live-coding warm-up (30 min drill):** (1) `two_sum`, (2) `valid_parentheses`, (3) `merge_intervals`, (4) `LRU cache`, (5) BFS shortest path on a grid. Narrate complexity out loud.
