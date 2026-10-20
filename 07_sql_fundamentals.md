# Session 7 — SQL Fundamentals (PostgreSQL flavor)

**Time:** Concepts 15 · Quiz 10 · Interview 15

Sample schema used throughout (create it to follow along: `psql` or any Postgres, e.g. `docker run -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres:16`):

```sql
CREATE TABLE departments (id serial PRIMARY KEY, name text NOT NULL);
CREATE TABLE employees (
  id serial PRIMARY KEY,
  name text NOT NULL,
  dept_id int REFERENCES departments(id),
  manager_id int REFERENCES employees(id),
  salary numeric(10,2) NOT NULL,
  hired_on date NOT NULL
);
CREATE TABLE orders (
  id serial PRIMARY KEY,
  customer_id int NOT NULL,
  amount numeric(10,2) NOT NULL,
  status text NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
```

---

## Part 1 — Concepts

### 7.1 Logical order of execution (the key to most "why doesn't this work?" questions)
Written: `SELECT … FROM … WHERE … GROUP BY … HAVING … ORDER BY … LIMIT`
Evaluated: **FROM/JOIN → WHERE → GROUP BY → HAVING → window functions → SELECT (aliases born here) → DISTINCT → ORDER BY → LIMIT/OFFSET**.
Consequences: you can't use a `SELECT` alias in `WHERE` (can in `ORDER BY`, and in Postgres `GROUP BY`); can't filter on an aggregate in `WHERE` (use `HAVING`); can't filter on a window function directly (wrap in a subquery/CTE).

### 7.2 Basics
```sql
SELECT name, salary FROM employees
WHERE dept_id = 3 AND salary BETWEEN 50000 AND 90000
  AND name ILIKE 'a%'                 -- ILIKE = case-insensitive (Postgres)
  AND hired_on >= DATE '2023-01-01'
ORDER BY salary DESC, name
LIMIT 10 OFFSET 20;                   -- OFFSET gets slow on deep pages → keyset pagination
```
`IN (…)`, `NOT IN`, `EXISTS`, `IS NULL`, `DISTINCT`, `DISTINCT ON (col)` (Postgres: first row per group), `CASE WHEN … THEN … ELSE … END`, `COALESCE(a, b)`, `NULLIF(a, b)`, casts `x::int`, `CAST(x AS int)`.

### 7.3 NULL — three-valued logic
- `NULL` = unknown. `NULL = NULL` → NULL (not true). Use `IS NULL`, `IS DISTINCT FROM`.
- `WHERE` keeps only rows where the condition is **TRUE** (unknown is dropped).
- `NOT IN (subquery containing NULL)` returns no rows → prefer `NOT EXISTS`.
- Aggregates ignore NULLs (except `COUNT(*)`); `AVG` over NULLs excludes them; `SUM` of no rows is NULL (use `COALESCE(SUM(x), 0)`).
- `ORDER BY`: NULLs sort last in ASC by default in Postgres (`NULLS FIRST/LAST` to control).

### 7.4 Joins
```sql
-- INNER: only matching rows
SELECT e.name, d.name FROM employees e JOIN departments d ON d.id = e.dept_id;
-- LEFT: all employees, dept columns NULL if no match
SELECT e.name, d.name FROM employees e LEFT JOIN departments d ON d.id = e.dept_id;
-- Anti-join: employees without a department
SELECT e.* FROM employees e LEFT JOIN departments d ON d.id = e.dept_id WHERE d.id IS NULL;
--   or: WHERE NOT EXISTS (SELECT 1 FROM departments d WHERE d.id = e.dept_id)
-- Self-join: employee and manager
SELECT e.name AS emp, m.name AS mgr FROM employees e LEFT JOIN employees m ON m.id = e.manager_id;
```
- Types: `INNER`, `LEFT`, `RIGHT`, `FULL OUTER`, `CROSS` (Cartesian), plus `LATERAL` (subquery per row).
- **Gotcha:** filter on the right table in `WHERE` after a `LEFT JOIN` turns it into an inner join. Put such conditions in `ON`.
- Join explosion: joining 1-to-many tables twice multiplies rows → aggregate before joining, or use `COUNT(DISTINCT)` carefully.

### 7.5 Aggregation
```sql
SELECT dept_id, COUNT(*) AS n, AVG(salary) AS avg_sal, MAX(salary)
FROM employees
GROUP BY dept_id
HAVING COUNT(*) >= 5
ORDER BY avg_sal DESC;
```
- Every non-aggregated `SELECT` column must be in `GROUP BY` (or functionally dependent on the PK).
- `COUNT(*)` counts rows; `COUNT(col)` non-NULLs; `COUNT(DISTINCT col)`.
- Conditional aggregation / pivot: `SUM(CASE WHEN status = 'paid' THEN amount END)`, or `COUNT(*) FILTER (WHERE status = 'paid')`.
- `GROUP BY ROLLUP/CUBE/GROUPING SETS` for subtotals.

### 7.6 Subqueries and CTEs
```sql
-- Scalar / IN / EXISTS
SELECT * FROM employees WHERE salary > (SELECT AVG(salary) FROM employees);
-- Correlated subquery
SELECT * FROM employees e WHERE salary > (SELECT AVG(salary) FROM employees WHERE dept_id = e.dept_id);

-- CTE: named, readable steps
WITH dept_avg AS (
  SELECT dept_id, AVG(salary) AS avg_sal FROM employees GROUP BY dept_id
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e JOIN dept_avg d USING (dept_id)
WHERE e.salary > d.avg_sal;

-- Recursive CTE: org chart / tree traversal
WITH RECURSIVE chain AS (
  SELECT id, name, manager_id, 1 AS depth FROM employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, c.depth + 1
  FROM employees e JOIN chain c ON e.manager_id = c.id
)
SELECT * FROM chain;
```
Since Postgres 12, non-recursive CTEs referenced once are inlined (optimizer can push predicates); use `MATERIALIZED` to force a barrier.

### 7.7 Window functions
Compute across related rows **without collapsing** them.
```sql
SELECT name, dept_id, salary,
       RANK()       OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rnk,
       ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) AS rn,
       salary - AVG(salary) OVER (PARTITION BY dept_id)              AS diff_from_avg,
       SUM(salary)  OVER (ORDER BY hired_on ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total,
       LAG(salary)  OVER (PARTITION BY dept_id ORDER BY hired_on)    AS prev_hire_salary
FROM employees;
```
- `ROW_NUMBER` (unique 1,2,3), `RANK` (ties share, gaps: 1,1,3), `DENSE_RANK` (1,1,2), `NTILE(n)`, `LAG/LEAD`, `FIRST_VALUE/LAST_VALUE`, aggregates `OVER`.
- **Top-N per group** pattern:
  ```sql
  SELECT * FROM (
    SELECT *, ROW_NUMBER() OVER (PARTITION BY dept_id ORDER BY salary DESC) rn FROM employees
  ) t WHERE rn <= 3;
  ```
- Default frame with `ORDER BY` is `RANGE UNBOUNDED PRECEDING … CURRENT ROW` (ties included) — a classic `LAST_VALUE` gotcha.

### 7.8 Set operations & DML
- `UNION` (dedupes, sorts/hashes) vs `UNION ALL` (fast, keeps duplicates); `INTERSECT`, `EXCEPT`.
- `INSERT … VALUES`, `INSERT … SELECT`, **upsert**: `INSERT … ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name`.
- `UPDATE … FROM`, `DELETE … USING`, `RETURNING *` (Postgres), `TRUNCATE`.
- Always run `UPDATE/DELETE` with a `WHERE`; test with `SELECT` first; wrap in a transaction.

### 7.9 Common query patterns
- Duplicates: `SELECT email, COUNT(*) FROM users GROUP BY email HAVING COUNT(*) > 1;`
- Delete duplicates keeping the lowest id: `DELETE FROM users a USING users b WHERE a.email = b.email AND a.id > b.id;`
- Nth highest salary: `SELECT DISTINCT salary FROM employees ORDER BY salary DESC OFFSET 1 LIMIT 1;` (or `DENSE_RANK`).
- Running totals / month-over-month change: window `SUM() OVER` / `LAG`.
- Gaps and islands: `value - ROW_NUMBER() OVER (ORDER BY value)` as group key.
- Date bucketing: `date_trunc('month', created_at)`.

---

## Part 2 — Homework / Exercises (not timed)

Load some data (a few `INSERT`s or `generate_series`) and write queries for:

1. Employees earning more than their manager.
2. Departments with no employees (two ways).
3. Average salary per department, only departments with > 3 employees, sorted descending.
4. Second-highest salary per department (handle ties via `DENSE_RANK`).
5. For each customer, their most recent order (three ways: `DISTINCT ON`, `ROW_NUMBER`, correlated subquery).
6. Monthly revenue and percent change from the previous month.
7. Running total of order amount per customer ordered by date.
8. Customers who placed orders in January but not in February.
9. Find consecutive-day login streaks (gaps and islands).
10. Upsert a row into a table with a unique constraint and return the resulting row.

<details><summary>Solution sketches</summary>

1. `SELECT e.name FROM employees e JOIN employees m ON m.id = e.manager_id WHERE e.salary > m.salary;`
2. `LEFT JOIN … WHERE e.id IS NULL` / `NOT EXISTS`.
5. `SELECT DISTINCT ON (customer_id) * FROM orders ORDER BY customer_id, created_at DESC;`
6. ```sql
   WITH m AS (SELECT date_trunc('month', created_at) mon, SUM(amount) rev FROM orders GROUP BY 1)
   SELECT mon, rev, ROUND(100.0*(rev - LAG(rev) OVER (ORDER BY mon)) / NULLIF(LAG(rev) OVER (ORDER BY mon),0), 1) pct FROM m;
   ```
8. `EXCEPT` of the two sets of customer IDs, or `NOT EXISTS`.
9. `date - (ROW_NUMBER() OVER (PARTITION BY user_id ORDER BY date))::int` constant within a streak → `GROUP BY user_id, that_key`.
10. `INSERT … ON CONFLICT (key) DO UPDATE SET … RETURNING *;`
</details>

---

## Part 3 — Quiz

**Q1.** Why does this fail? Fix it.
```sql
SELECT dept_id, COUNT(*) AS n FROM employees WHERE n > 5 GROUP BY dept_id;
```
**Q2.** `SELECT * FROM t WHERE x NOT IN (SELECT y FROM u)` returns zero rows although many x don't appear in u. Likely cause?

**Q3.** Difference between `WHERE` and `HAVING`?

**Q4.** Results of `RANK()`, `DENSE_RANK()`, `ROW_NUMBER()` for salaries 100, 100, 90?

**Q5.** `LEFT JOIN b ON … WHERE b.status = 'x'` — what's the problem?

**Q6.** `COUNT(*)` vs `COUNT(col)` vs `COUNT(DISTINCT col)`?

**Q7.** `UNION` vs `UNION ALL` — which is faster and why?

**Q8.** What does `SELECT NULL = NULL` return? How to compare nullable values for equality?

**Q9.** How do you get the top 3 salaries per department?

**Q10.** `SELECT AVG(bonus)` where half the bonuses are NULL — what's averaged?

<details><summary>Answers</summary>

1. `n` is an alias from SELECT, and aggregates can't be in `WHERE` (evaluated before grouping). Use `HAVING COUNT(*) > 5`.
2. `u.y` contains a NULL → `x NOT IN (…, NULL)` is never true. Use `NOT EXISTS` or filter `WHERE y IS NOT NULL`.
3. `WHERE` filters rows before grouping (no aggregates); `HAVING` filters groups after aggregation.
4. RANK: 1,1,3. DENSE_RANK: 1,1,2. ROW_NUMBER: 1,2,3 (arbitrary among ties unless ordered further).
5. The filter on `b` discards the NULL-extended rows → behaves like INNER JOIN. Move the condition to `ON`.
6. All rows; non-NULL values of col; distinct non-NULL values.
7. `UNION ALL` — no duplicate elimination (no sort/hash step).
8. NULL (unknown). Use `IS NOT DISTINCT FROM` or `a = b OR (a IS NULL AND b IS NULL)`.
9. `ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)` in a subquery, filter `rn <= 3` (use `DENSE_RANK` to include ties).
10. Only non-NULL bonuses (the average of the non-NULL half). Use `AVG(COALESCE(bonus,0))` if NULL means zero.
</details>

---

## Part 4 — Interview Questions

**I1. Explain the different JOIN types with an example of when you'd use each.**
<details><summary>Model answer</summary>
INNER: only matching rows (orders with customers). LEFT: keep all rows from the left, NULLs when no match (customers and their orders, including those with none — and, with `WHERE right.id IS NULL`, customers with no orders = anti-join). RIGHT: mirror of left (rarely used). FULL: keep unmatched from both (reconciliation between two systems). CROSS: all combinations (calendar × product grid). Mention self-joins and `LATERAL`. Add the ON-vs-WHERE gotcha for outer joins.
</details>

**I2. Window functions vs GROUP BY?**
<details><summary>Model answer</summary>
`GROUP BY` collapses rows to one per group. Window functions compute over a set of related rows but keep every row — enabling rankings, running totals, moving averages, previous/next row comparisons, and shares of total in the same result. Evaluated after `WHERE/GROUP BY/HAVING`, so you filter on them via a subquery/CTE. Give the top-N-per-group example.
</details>

**I3. CTE vs subquery vs temp table?**
<details><summary>Model answer</summary>
CTE: readability, reuse within a statement, recursion. In modern Postgres (12+) simple CTEs are inlined, so no performance penalty by default; `MATERIALIZED` forces a single evaluation. Subquery: fine for simple filters. Temp table: persists across statements in a session, can be indexed/analyzed — good for large intermediate results reused many times. Choose by clarity first, measure with `EXPLAIN ANALYZE`.
</details>

**I4. How do you find and delete duplicate rows?**
<details><summary>Model answer</summary>
Identify via `GROUP BY … HAVING COUNT(*) > 1`. Delete keeping one: `DELETE FROM t a USING t b WHERE a.key = b.key AND a.id > b.id` or `ROW_NUMBER() OVER (PARTITION BY key ORDER BY id)` and delete `rn > 1` by `ctid`/id. Then add a UNIQUE constraint so it can't recur. Do it in a transaction and take a backup/count first.
</details>

**I5. What is the difference between `DELETE`, `TRUNCATE`, and `DROP`?**
<details><summary>Model answer</summary>
`DELETE`: row-by-row DML with `WHERE`, fires row triggers, MVCC (dead tuples need vacuum), transactional. `TRUNCATE`: removes all rows fast (deallocates storage), takes a strong lock, no `WHERE`, resets sequences with `RESTART IDENTITY`; transactional in Postgres. `DROP`: removes the table definition itself.
</details>

**I6. How does `NULL` behave in comparisons, aggregates, and constraints?**
<details><summary>Model answer</summary>
Three-valued logic; comparisons with NULL yield NULL; `WHERE` treats it as false; aggregates skip NULLs except `COUNT(*)`; `UNIQUE` allows multiple NULLs by default in Postgres (`NULLS NOT DISTINCT` in 15+); `NOT IN` + NULL trap; `COALESCE` to default. Give a bug you've seen.
</details>

**I7. Write a query: "For each department, the employee(s) with the highest salary."**
<details><summary>Model answer</summary>
```sql
SELECT * FROM (
  SELECT e.*, RANK() OVER (PARTITION BY dept_id ORDER BY salary DESC) r FROM employees e
) t WHERE r = 1;
```
Alternatives: join to `(SELECT dept_id, MAX(salary) …)` subquery; or `DISTINCT ON` if only one is needed. Discuss ties → `RANK` includes ties, `ROW_NUMBER` picks one.
</details>

**I8. OFFSET pagination vs keyset pagination?**
<details><summary>Model answer</summary>
`OFFSET n` scans and discards n rows — cost grows with page depth and results can shift under concurrent writes. Keyset: `WHERE (created_at, id) < (:last_ts, :last_id) ORDER BY created_at DESC, id DESC LIMIT 20` using an index — constant cost and stable. Trade-off: no random page jump.
</details>

**Whiteboard drill:** Write queries 4, 5, 6 from the homework from memory, then explain how you'd verify correctness with edge cases (ties, NULLs, empty groups).
