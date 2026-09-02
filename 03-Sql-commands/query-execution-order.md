# SQL Query Execution Order

This is one of the most important — and most confusing — things in SQL:
**the order you *write* a query is not the order MySQL actually *runs* it.**

This explains a lot of "why doesn't this work?" moments later (e.g. why a
column alias works in `ORDER BY` but not in `WHERE`).

---

## The order you WRITE a SELECT query

```sql
SELECT column_list        -- 1 (but runs almost last!)
FROM table_name            -- 2
JOIN other_table ON ...    -- 3
WHERE condition             -- 4
GROUP BY column              -- 5
HAVING condition               -- 6
ORDER BY column                  -- 7
LIMIT n;                          -- 8
```

## The order MySQL actually EXECUTES it

| Step | Clause | What happens |
|---|---|---|
| 1 | `FROM` | Identifies which table(s) to pull data from |
| 2 | `JOIN` | Combines rows from multiple tables based on the `ON` condition |
| 3 | `WHERE` | Filters individual rows *before* any grouping — row-by-row condition |
| 4 | `GROUP BY` | Groups the filtered rows into buckets (e.g. by student, by course) |
| 5 | `HAVING` | Filters the *groups* created by GROUP BY (not individual rows) |
| 6 | `SELECT` | Picks/computes the actual columns or expressions to return |
| 7 | `ORDER BY` | Sorts the final result set |
| 8 | `LIMIT` | Cuts the result down to N rows |

So the real execution sequence is:
```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```
...even though you *type* `SELECT` first and `FROM` second.

---

## Why this matters (practical consequences)

**1. You can't filter on a column alias in `WHERE`, but you can in `ORDER BY`**

```sql
-- ❌ ERROR — WHERE runs before SELECT, so 'total_credits' doesn't exist yet
SELECT Credits * 2 AS total_credits FROM courses WHERE total_credits > 5;

-- ✅ WORKS — ORDER BY runs after SELECT, so the alias already exists
SELECT Credits * 2 AS total_credits FROM courses ORDER BY total_credits;
```

**2. `WHERE` vs `HAVING` — this is the big one**

- `WHERE` filters rows **before** grouping (can't use aggregate functions like `COUNT()`, `SUM()` here)
- `HAVING` filters groups **after** grouping (this is where aggregate functions belong)

```sql
-- Count enrollments per course, but only show courses with more than 1 enrollment
SELECT CourseID, COUNT(*) AS enrollment_count
FROM enrollment
GROUP BY CourseID
HAVING COUNT(*) > 1;
```
```sql
-- ❌ ERROR — can't use COUNT() in WHERE, because WHERE runs before grouping exists
SELECT CourseID, COUNT(*) AS enrollment_count
FROM enrollment
WHERE COUNT(*) > 1
GROUP BY CourseID;
```

**3. `LIMIT` always runs last** — so `ORDER BY ... LIMIT n` correctly gives
you "top N" results, because sorting happens before the cut-off.

---

## Quick reference: full command categories vs execution order

These are two different concepts — don't mix them up:

- [sql-command-categories.md](./sql-command-categories.md) → **what kind of job** a
  command does (DDL/DML/DCL/DQL) — a classification.
- This file → **what order clauses run in**, specifically within a single
  `SELECT` (DQL) statement — a sequence.

## Summary table

| Write order | Execution order | Clause | Purpose |
|---|---|---|---|
| 2 | 1 | `FROM` | choose source table(s) |
| 3 | 2 | `JOIN` | combine tables |
| 4 | 3 | `WHERE` | filter rows |
| 5 | 4 | `GROUP BY` | group rows |
| 6 | 5 | `HAVING` | filter groups |
| 1 | 6 | `SELECT` | pick/compute columns |
| 7 | 7 | `ORDER BY` | sort results |
| 8 | 8 | `LIMIT` | cap row count |
