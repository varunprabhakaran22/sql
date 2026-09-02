# Window Functions & Stored Functions/Procedures — Theory

Two different topics grouped in this chapter because they're both
"advanced SQL features" beyond plain SELECT/JOIN, but they solve very
different problems:

| | Window Functions | Stored Functions / Procedures |
|---|---|---|
| What it is | A calculation across a set of rows, *without* collapsing them into one row (unlike `GROUP BY`) | Reusable, saved SQL logic stored inside the database itself |
| Runs on | Every row of a query result | Called explicitly, like a function/subroutine |
| Use case | Rankings, running totals, comparing to previous/next row | Encapsulating repeated business logic (e.g. "get a customer's total spend") |
| Where explained | [window-functions.md](./window-functions.md) | [stored-functions.md](./stored-functions.md) |

---

## Part 1: Window Functions

### The core idea

A normal aggregate (`SUM`, `AVG`, `COUNT` — see
[aggregate-functions.md](../04-aggregate-functions/aggregate-functions.md))
with `GROUP BY` **collapses** many rows into one summary row per group.

A window function computes something *across* related rows too, but
**keeps every row visible** — it adds an extra calculated column instead
of collapsing anything. This "window" is the group of rows the
calculation looks at for each row, defined by `OVER (...)`.

```sql
<function>() OVER (
    PARTITION BY column   -- optional: split into groups (like GROUP BY, but doesn't collapse rows)
    ORDER BY column        -- optional: defines row order for ranking/LAG/LEAD/running totals
)
```

### Why this matters vs GROUP BY

```sql
-- GROUP BY: one row per customer, individual orders are gone
SELECT customer_id, SUM(amount) FROM orders GROUP BY customer_id;

-- Window function: every order row still visible, PLUS a running total column
SELECT customer_id, product, amount,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders;
```

### Categories of window functions covered here

1. **Ranking functions** — `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`
2. **Partitioning** — `PARTITION BY`, splits the window into sub-groups
3. **Offset functions** — `LAG()`, `LEAD()` — look at a previous/next row's value
4. **Running aggregates** — `SUM()`, `AVG()`, etc. used with `OVER` instead of `GROUP BY`

Full explanation with real data and verified output:
[window-functions.md](./window-functions.md)

---

## Part 2: Stored Functions & Procedures

### The core idea

Both let you save a block of SQL logic *inside the database itself*, so
it can be reused instead of retyping the same query pattern everywhere.

| | Stored Function | Stored Procedure |
|---|---|---|
| Returns | Exactly one value (via `RETURN`) | Zero or more result sets (via `SELECT` inside it), or nothing |
| Called from | Inside a query — `SELECT my_func(x)` | Standalone — `CALL my_procedure(x)` |
| Can run SELECT/INSERT/UPDATE freely? | Restricted (mainly reads) | Yes — full flexibility |
| Typical use | Compute and return a single derived value | Run a multi-step operation or return a filtered report |

### Why `DELIMITER` shows up

Normally `;` ends a statement immediately. But a function/procedure body
itself contains multiple `;`-ended statements inside `BEGIN...END`. If we
kept `;` as the delimiter, MySQL would stop at the *first* semicolon
inside the body, thinking the whole thing ended there.

The fix: temporarily change the statement-ending character to something
else (commonly `//`) while defining the function, then switch back:

```sql
DELIMITER //

CREATE FUNCTION ... 
BEGIN
    ... -- statements ending in ; are now safe, they don't end the CREATE FUNCTION
END //

DELIMITER ;   -- switch back to normal
```

Full explanation with real working function + procedure, verified output:
[stored-functions.md](./stored-functions.md)
