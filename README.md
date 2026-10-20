# Python Job Interview Review Course

11 sessions (7 Python, 2 SQL/DB, 2 TS/JS), 30–45 min each. Run through the whole set (or the relevant subset) before every Python interview.

| # | File | Topic | Level |
|---|------|-------|-------|
| 1 | `01_python_data_model_oop.md` | Data model, classes, inheritance, dataclasses | Intermediate |
| 2 | `02_python_functions_iterators_generators.md` | Closures, decorators, iterators, generators, context managers | Intermediate |
| 3 | `03_python_internals_memory.md` | Objects, mutability, references, hashing, GIL, GC, gotchas | Advanced |
| 4 | `04_python_concurrency.md` | Threads, processes, asyncio, GIL implications | Advanced |
| 5 | `05_python_typing_testing_errors.md` | Type hints, exceptions, pytest/mock, logging, packaging | Intermediate/Advanced |
| 6 | `06_python_stdlib_performance_algorithms.md` | collections, itertools, heapq, complexity, profiling, coding patterns | Intermediate/Advanced |
| 7 | `07_sql_fundamentals.md` | SELECT, joins, aggregation, CTEs, window functions, NULL | Fundamental |
| 8 | `08_postgresql_and_db_design.md` | Keys, normalization, indexes, EXPLAIN, transactions/MVCC, Python DB access | Fundamental |
| 9 | `09_javascript_core.md` | Scope, closures, `this`, prototypes, event loop, promises | Basic |
| 10 | `10_typescript_basics.md` | Types, unions, generics, narrowing, utility types | Basic |
| 11 | `11_python_web_frameworks_fastapi_flask_django.md` | FastAPI, Flask, Django, project structure (routers/blueprints), microservices, MCP servers | Intermediate |

## Session format and timing (30–45 min)

Every file has the same four parts:

1. **Concepts** — guided reading with examples (~15 min). Read the code, *predict output before the "Output" comment*, then move on.
2. **Homework / Exercises** — NOT timed. Do these on your own beforehand (or after); solutions are in collapsed `<details>` blocks.
3. **Quiz** — step through the questions (~8–10 min). Answer out loud first, then expand the answer.
4. **Interview questions** — (~10–15 min). Practice answering in 60–90 seconds each; model answers and likely follow-ups are included.

Suggested pacing: Concepts 15 → Quiz 10 → Interview 15 = 40 min. Short on time? Skip "Going deeper" notes.

## Using these files

- Best viewed in VS Code Markdown preview or GitHub (collapsed answers use `<details>`).
- Run code with Python 3.11+ (`python3 --version`). SQL examples target PostgreSQL 15+. TS examples need `npm i -g typescript tsx` (or use https://www.typescriptlang.org/play).
- Suggested schedule before an interview: Day −3: sessions 1–3. Day −2: sessions 4–6. For API/backend roles add session 11 (FastAPI/Flask/Django). Day −1: 7–8 (or 9–10 if the role is full-stack), then skim the interview sections of whichever you're weakest on.
- Track weak spots: put a ✗ next to any quiz question you missed, and re-read those next time.

## Interview delivery tips (apply to every session)

- Answer in three beats: **definition → why it matters / trade-off → concrete example or gotcha**.
- Say complexity (Big-O) unprompted for any data-structure or algorithm answer.
- When unsure, reason aloud from first principles; interviewers grade the reasoning.
- Name the real-world failure mode ("this bites you when…"). It signals production experience.
# job_python
