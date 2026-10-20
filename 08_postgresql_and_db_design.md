# Session 8 — PostgreSQL & Database Design, Indexes, Transactions, Python Access

**Time:** Concepts 15 · Quiz 10 · Interview 15

---

## Part 1 — Concepts

### 8.1 Relational model & keys
- **Table** (relation), **row** (tuple), **column** (attribute). Schema enforces structure; constraints enforce integrity.
- **Primary key**: unique + NOT NULL row identity. Surrogate (`bigint GENERATED ALWAYS AS IDENTITY`, `uuid`) vs natural key (email, ISBN) — surrogate keys are stable; still add a UNIQUE constraint on natural keys.
- **Foreign key**: `REFERENCES parent(id) ON DELETE CASCADE | SET NULL | RESTRICT`. Postgres does **not** auto-index FK columns — add an index yourself (joins and cascading deletes need it).
- Constraints: `NOT NULL`, `UNIQUE`, `CHECK (price >= 0)`, `DEFAULT`, `EXCLUDE` (e.g., no overlapping reservations).
- Relationships: 1-to-many (FK on the "many" side), many-to-many (junction table with composite PK), 1-to-1 (FK + UNIQUE).

```sql
CREATE TABLE students (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, email text UNIQUE NOT NULL);
CREATE TABLE courses  (id bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY, title text NOT NULL);
CREATE TABLE enrollments (
  student_id bigint REFERENCES students(id) ON DELETE CASCADE,
  course_id  bigint REFERENCES courses(id),
  enrolled_at timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (student_id, course_id)
);
```

### 8.2 Normalization (and when to stop)
- **1NF**: atomic values, no repeating groups. **2NF**: no partial dependency on part of a composite key. **3NF**: no transitive dependency (non-key columns depend on "the key, the whole key, and nothing but the key"). BCNF: stricter 3NF.
- Goal: avoid update/insert/delete anomalies and redundancy. 
- **Denormalize deliberately** for read performance (reporting tables, caching counts, JSONB snapshots) — with a plan for keeping copies consistent.
- OLTP (many small transactions → normalized) vs OLAP (analytics → star schema, columnar warehouses).

### 8.3 Data types worth knowing (Postgres)
`integer/bigint`, `numeric(p,s)` (exact; **use for money**, never float), `text` (preferred over `varchar(n)`; no perf difference), `boolean`, `date`, `timestamptz` (**store timestamps with time zone**; stored as UTC), `uuid`, `jsonb` (binary JSON, indexable), arrays, `enum`, `inet`, range types (`tstzrange`), `tsvector` (full-text).

### 8.4 Indexes
Without an index, a filter requires a **sequential scan** (read the whole table). An index is a separate structure (default **B-tree**) letting Postgres find rows in O(log n).

```sql
CREATE INDEX idx_orders_customer ON orders (customer_id);
CREATE INDEX idx_orders_cust_date ON orders (customer_id, created_at DESC);   -- composite
CREATE UNIQUE INDEX ux_users_email ON users (lower(email));                    -- expression index
CREATE INDEX idx_active ON orders (created_at) WHERE status = 'active';        -- partial index
CREATE INDEX idx_orders_cover ON orders (customer_id) INCLUDE (amount);        -- covering (index-only scan)
CREATE INDEX idx_doc ON docs USING gin (payload jsonb_path_ops);               -- GIN for jsonb/arrays/full-text
```
Rules of thumb:
- Index columns used in `WHERE`, `JOIN`, `ORDER BY`. Composite index order matters: **leftmost-prefix** — `(a, b)` helps `WHERE a=?` and `WHERE a=? AND b=?`, not `WHERE b=?`. Equality columns first, then range/sort.
- Index types: **B-tree** (equality, range, sort), **Hash** (equality only), **GIN** (jsonb, arrays, FTS), **GiST** (geometry, ranges), **BRIN** (huge, naturally ordered tables, tiny).
- Costs: slower writes, more disk, bloat. Don't index low-selectivity columns (e.g., boolean) unless partial.
- Index not used? Functions on the column (`WHERE lower(email) = …` needs an expression index), type mismatches, leading wildcards (`LIKE '%x'`), tiny tables (planner prefers seq scan), stale stats (`ANALYZE`).
- `CREATE INDEX CONCURRENTLY` to avoid blocking writes in production.

### 8.5 EXPLAIN
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE customer_id = 42;
```
Read bottom-up/inside-out. Look for: node type (`Seq Scan`, `Index Scan`, `Index Only Scan`, `Bitmap Heap Scan`), join algorithm (`Nested Loop`, `Hash Join`, `Merge Join`), estimated vs actual rows (big mismatches ⇒ stale stats), `Rows Removed by Filter`, `Sort` spilling to disk, loops. `ANALYZE` actually runs the query (wrap `UPDATE/DELETE` in a transaction you roll back!).

### 8.6 Transactions & ACID
- **Atomicity** (all or nothing), **Consistency** (constraints hold), **Isolation** (concurrent transactions don't see each other's partial work), **Durability** (committed data survives crashes — WAL).
```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;      -- or ROLLBACK;
```
- **MVCC**: Postgres keeps multiple row versions; readers don't block writers and vice versa. Old versions (dead tuples) are cleaned by **VACUUM** (autovacuum). Long-running transactions block cleanup → bloat.
- **Isolation levels** (Postgres): `READ COMMITTED` (default; each statement sees data committed before it started), `REPEATABLE READ` (snapshot for whole transaction; serialization failures possible), `SERIALIZABLE` (as if serial; retry on failure). (Postgres' `READ UNCOMMITTED` behaves as `READ COMMITTED`.)
- Anomalies: dirty read, non-repeatable read, phantom read, lost update, write skew.
- **Locking**: `SELECT … FOR UPDATE` (lock rows), `FOR UPDATE SKIP LOCKED` (job queues), advisory locks. Avoid lost updates with atomic `UPDATE x = x + 1`, optimistic locking (version column), or row locks. Deadlocks: lock rows in a consistent order.
- Keep transactions short; don't hold them open during network calls.

### 8.7 Postgres features likely to come up
- **JSONB**: `payload->>'name'`, `payload @> '{"a":1}'`, GIN index. Use for flexible attributes, not as a replacement for relational design.
- **CTEs, window functions, `RETURNING`, `ON CONFLICT`** (Session 7).
- **Views** (saved query) and **materialized views** (stored result; `REFRESH MATERIALIZED VIEW [CONCURRENTLY]`).
- **Partitioning** (range/list/hash) for huge tables/time-series; **replication** (streaming physical, logical), **read replicas** (replica lag!), **pgBouncer** connection pooling (each Postgres connection is a process → keep connections limited).
- Full-text search (`tsvector`, `to_tsquery`), extensions (`pg_trgm`, `postgis`, `pgvector`, `uuid-ossp`), `LISTEN/NOTIFY`.
- Maintenance: `VACUUM`, `ANALYZE`, `pg_stat_statements` (find slow queries), `pg_stat_activity`.
- Migrations: schema changes versioned (Alembic, Django migrations, Flyway). Safe migrations: add nullable column, backfill in batches, then add constraint (`NOT VALID` → `VALIDATE`); avoid long exclusive locks.

### 8.8 Using PostgreSQL from Python
```python
import psycopg                                   # psycopg 3
with psycopg.connect("postgresql://user:pw@localhost/db") as conn:   # commits on exit, rolls back on error
    with conn.cursor() as cur:
        cur.execute("SELECT id, name FROM employees WHERE dept_id = %s", (dept_id,))   # parameterized!
        rows = cur.fetchall()
```
- **Never** build SQL via f-strings with user input → **SQL injection**. Use placeholders (`%s` psycopg, `$1` asyncpg, `:name` SQLAlchemy `text()`); identifiers need `psycopg.sql.Identifier`.
- Connection pooling (`psycopg_pool`, SQLAlchemy engine pool, pgBouncer). Async: `asyncpg`, `psycopg` async, SQLAlchemy 2.0 async.
- **SQLAlchemy 2.0** (Core + ORM):
  ```python
  from sqlalchemy import select, create_engine
  from sqlalchemy.orm import Session, DeclarativeBase, Mapped, mapped_column, relationship, selectinload

  engine = create_engine("postgresql+psycopg://user:pw@localhost/db", pool_size=5)
  with Session(engine) as s:
      stmt = select(Employee).where(Employee.dept_id == 3).options(selectinload(Employee.projects))
      emps = s.scalars(stmt).all()
      s.commit()
  ```
- **N+1 problem**: one query for the parents + one per child. Fix with joins/eager loading (`selectinload`, `joinedload`; Django `select_related`/`prefetch_related`) or a manual `WHERE id = ANY(%s)` batch.
- Bulk operations: `executemany`/`COPY` (fastest) instead of row-by-row inserts; commit in batches.
- Handle `serialization_failure`/`deadlock_detected` with retry logic; set statement timeouts.
- ORMs vs raw SQL: ORM for CRUD productivity & migrations; raw SQL/Core for complex analytics — know the SQL the ORM generates (`echo=True`).

---

## Part 2 — Homework / Exercises (not timed)

1. Design a schema for a **library** (books, authors, members, loans) in 3NF with PKs, FKs, constraints. Draw ER diagram (text OK). Identify the many-to-many relationships.
2. Populate `orders` with 1M rows using `generate_series`; time `SELECT * FROM orders WHERE customer_id = 123` before/after an index; read the `EXPLAIN ANALYZE` output both times.
3. Show that a composite index `(a, b)` is used for `WHERE a=1` and `WHERE a=1 AND b=2` but not `WHERE b=2` (use EXPLAIN).
4. Demonstrate a lost update using two `psql` sessions and `READ COMMITTED`; fix with `SELECT … FOR UPDATE`, then with an atomic `UPDATE`.
5. Demonstrate a deadlock with two sessions updating two rows in opposite order.
6. In Python, write: (a) an unsafe f-string query and demonstrate injection with `' OR '1'='1`; (b) the parameterized version.
7. Reproduce an N+1 with SQLAlchemy (`echo=True`), then fix with `selectinload`; count queries.
8. Write an idempotent upsert for a `daily_stats(day, metric, value)` table with a composite unique key; batch-load 10k rows with `COPY` (psycopg `cursor.copy`).

<details><summary>Solution hints</summary>

2. Before: `Seq Scan … Rows Removed by Filter: ~1M`, tens of ms; after: `Index Scan` or `Bitmap Heap Scan`, < 1 ms.
4. Session A: `BEGIN; SELECT balance …;` Session B same; both write `balance = old - 10` → one update lost. With `FOR UPDATE`, B waits until A commits and then reads the new value.
7. Expect `1 + N` SELECT statements before, `2` after.
</details>

---

## Part 3 — Quiz

**Q1.** Which normal form removes transitive dependencies?

**Q2.** Does Postgres automatically index foreign key columns? Primary keys? Unique constraints?

**Q3.** You have an index on `(last_name, first_name)`. Which of these use it efficiently? (a) `WHERE last_name='X'` (b) `WHERE first_name='Y'` (c) `WHERE last_name='X' AND first_name='Y'` (d) `ORDER BY last_name, first_name`

**Q4.** `WHERE lower(email) = 'a@b.com'` is slow even though `email` is indexed. Why, and the fix?

**Q5.** What's the Postgres default isolation level, and what can still happen at that level?

**Q6.** What does `VACUUM` do and why is it needed in Postgres?

**Q7.** Why is `float` a bad type for currency? What do you use?

**Q8.** What is SQL injection and the primary defense?

**Q9.** What is the N+1 query problem?

**Q10.** `EXPLAIN` vs `EXPLAIN ANALYZE`?

**Q11.** `SELECT ... FOR UPDATE SKIP LOCKED` — use case?

<details><summary>Answers</summary>

1. Third normal form (3NF).
2. PK and UNIQUE: yes (implicit unique index). FK: **no** — create it manually.
3. (a) yes, (b) no (not leftmost prefix; maybe a full index scan), (c) yes, (d) yes.
4. The expression isn't the indexed column. Create `CREATE INDEX ON users (lower(email))`, or use `citext`.
5. `READ COMMITTED`: non-repeatable reads, phantoms, lost updates (read-modify-write) and write skew are possible. Use higher isolation or explicit locks/atomic updates.
6. Reclaims space from dead row versions (MVCC leaves old versions behind), updates visibility map & stats help index-only scans, prevents transaction ID wraparound. Autovacuum does it routinely.
7. Binary floating-point can't represent decimals exactly (0.1+0.2≠0.3). Use `numeric(p,s)` / integer cents; `Decimal` in Python.
8. Attacker input is interpreted as SQL. Defense: parameterized queries/prepared statements (never string-format), least-privilege DB roles, validate identifiers via allowlist, ORM bindings.
9. 1 query for N parent rows, then N queries for children. Fix with joins/eager loading/batch `IN`.
10. `EXPLAIN` shows the plan with estimates; `ANALYZE` executes the query and reports actual times and row counts.
11. Job queues: multiple workers claim distinct unlocked rows without blocking each other.
</details>

---

## Part 4 — Interview Questions

**I1. Walk me through designing a schema for an e-commerce system.**
<details><summary>Model answer</summary>
Entities: `customers`, `products`, `orders`, `order_items` (junction: order_id, product_id, qty, unit_price **copied at purchase time**), `addresses`, `payments`, `inventory`. Use surrogate PKs, FKs with indexes, `numeric` money, `timestamptz`, status enums/check constraints, unique constraints (email, SKU). Normalized to 3NF; deliberately snapshot price and shipping address on the order (history shouldn't change). Consider soft delete vs archive, audit columns (`created_at/updated_at`), indexes for access patterns (orders by customer & date), partitioning orders by time later, and idempotency keys for payments.
</details>

**I2. How do indexes work, and how do you decide what to index?**
<details><summary>Model answer</summary>
B-tree: balanced tree of sorted keys pointing at heap tuples; O(log n) lookups, range scans, sorted output. Decide from real queries: `pg_stat_statements` + `EXPLAIN ANALYZE`; index selective columns in WHERE/JOIN/ORDER BY; composite indexes with equality columns first; partial and covering indexes for hot paths. Costs: writes, space, bloat, planner complexity. Remove unused indexes (`pg_stat_user_indexes`). Add `CONCURRENTLY` in prod.
</details>

**I3. A query is slow in production. What do you do?**
<details><summary>Model answer</summary>
Get the exact query and parameters (logs/`pg_stat_statements`), run `EXPLAIN (ANALYZE, BUFFERS)`, compare estimated vs actual rows, check for seq scans on big tables, missing/unused index, non-sargable predicates, bad join order, sorts spilling to disk, N+1 from the app. Remedies: add/adjust index, rewrite query, `ANALYZE`, reduce selected columns, pagination, caching/materialized view, partitioning; verify improvement and watch write overhead; check locks (`pg_locks`) if latency is wait-based.
</details>

**I4. Explain ACID and isolation levels. What is MVCC?**
<details><summary>Model answer</summary>
(See 8.6.) Emphasize MVCC: each transaction sees a snapshot; writers create new row versions; readers don't block. Consequence: autovacuum and long transactions. Isolation: READ COMMITTED default; REPEATABLE READ gives snapshot isolation; SERIALIZABLE uses SSI and may abort — design retries. Show a lost update example and the atomic-update fix.
</details>

**I5. SQL vs NoSQL — when would you choose which?**
<details><summary>Model answer</summary>
Relational: structured data, relationships, transactions, ad-hoc queries/joins, strong consistency — default choice, and Postgres with JSONB covers a lot of "document" needs. NoSQL: key-value (Redis, DynamoDB) for massive scale/low-latency access by key, document stores for flexible schema, wide-column for huge write throughput, graph for relationship traversal, search engines for text relevance. Choose based on access patterns, consistency needs, scale, and team expertise — avoid premature distribution.
</details>

**I6. How do you prevent SQL injection and handle connections safely in Python?**
<details><summary>Model answer</summary>
Parameterized queries always; never concatenate user input; ORM bindings; allowlist for dynamic identifiers/ORDER BY; least privilege; secrets from environment/secret manager; connection pooling with sane limits; context managers for commits/rollbacks; timeouts; retry on transient errors; TLS.
</details>

**I7. What is an ORM good and bad at? How do you avoid its pitfalls?**
<details><summary>Model answer</summary>
Good: CRUD productivity, mapping, migrations, portability, relationship handling, session/unit-of-work. Bad: hidden queries (N+1), inefficient bulk ops, awkward complex analytics, leaky abstractions. Mitigate: inspect generated SQL, eager loading, bulk APIs/`COPY`, drop to Core/raw SQL for reports, keep sessions short, avoid lazy loading outside session scope.
</details>

**I8. Explain how you would do a zero-downtime schema migration (e.g., rename a column).**
<details><summary>Model answer</summary>
Expand/contract: (1) add new column (nullable); (2) deploy code writing both (or trigger sync); (3) backfill in small batches; (4) switch reads to new column; (5) stop writing old; (6) drop old. Use `CREATE INDEX CONCURRENTLY`, `NOT VALID` constraints then `VALIDATE`, short lock timeouts, and feature flags. Never hold `ACCESS EXCLUSIVE` locks long.
</details>

**Whiteboard drill:** Design tables + indexes for "users follow users, with a feed of their posts sorted by time." Discuss fan-out on read vs write, index `(follower_id, followee_id)`, `(author_id, created_at DESC)`, keyset pagination.
