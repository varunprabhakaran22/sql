# Window Functions — ROW_NUMBER, RANK, PARTITION BY, LAG, LEAD

Theory recap is in [theory.md](./theory.md). This file has real queries
run against the `orders` table from [05-joins](../05-joins/joins-examples.md),
with verified output.

Reference data (excluding the deliberate orphan row, `customer_id = 99`):

| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 500.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |

---

## 1. ROW_NUMBER() — a unique sequential number per row

Assigns 1, 2, 3, ... in the order specified — no ties possible, every
row gets a distinct number even if values are equal.

```sql
SELECT customer_id, product, amount,
    ROW_NUMBER() OVER (ORDER BY amount DESC) AS row_num
FROM orders
WHERE customer_id != 99;
```
| customer_id | product | amount | row_num |
|---|---|---|---|
| 1 | Laptop | 55000.00 | 1 |
| 2 | Office Table | 12000.00 | 2 |
| 3 | Desk Chair | 7500.00 | 3 |
| 2 | Keyboard | 1200.00 | 4 |
| 1 | Mouse | 500.00 | 5 |
| 4 | Notebook | 50.00 | 6 |

---

## 2. RANK() vs DENSE_RANK() — ranking, with tie-handling

Both assign the same rank to tied rows, but differ in what happens
**after** a tie:

- `RANK()` — skips numbers after a tie (e.g. two rows tied at rank 2 →
  next row is rank 4, not 3)
- `DENSE_RANK()` — never skips (two rows tied at rank 2 → next row is rank 3)

```sql
SELECT customer_id, product, amount,
    RANK() OVER (ORDER BY amount DESC) AS rnk,
    DENSE_RANK() OVER (ORDER BY amount DESC) AS dense_rnk
FROM orders
WHERE customer_id != 99;
```
| customer_id | product | amount | rnk | dense_rnk |
|---|---|---|---|---|
| 1 | Laptop | 55000.00 | 1 | 1 |
| 2 | Office Table | 12000.00 | 2 | 2 |
| 3 | Desk Chair | 7500.00 | 3 | 3 |
| 2 | Keyboard | 1200.00 | 4 | 4 |
| 1 | Mouse | 500.00 | 5 | 5 |
| 4 | Notebook | 50.00 | 6 | 6 |

> Note: this data has no tied amounts, so `RANK`, `DENSE_RANK`, and
> `ROW_NUMBER` all happen to look identical here. The real difference
> only shows up with duplicate values — e.g. if two orders both had
> amount `7500.00`, they'd both get rank 3, and `RANK()`'s next row would
> jump to 5 while `DENSE_RANK()`'s next row would be 4.

---

## 3. PARTITION BY — reset the window per group

`PARTITION BY` splits rows into groups (like `GROUP BY`), but — unlike
`GROUP BY` — every individual row still shows up in the output. The
window function's calculation restarts for each partition.

```sql
SELECT customer_id, product, amount,
    ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY amount DESC) AS rank_within_customer
FROM orders
WHERE customer_id != 99;
```
| customer_id | product | amount | rank_within_customer |
|---|---|---|---|
| 1 | Laptop | 55000.00 | 1 |
| 1 | Mouse | 500.00 | 2 |
| 2 | Office Table | 12000.00 | 1 |
| 2 | Keyboard | 1200.00 | 2 |
| 3 | Desk Chair | 7500.00 | 1 |
| 4 | Notebook | 50.00 | 1 |

Notice the numbering **restarts at 1 for each customer_id** — this
answers "what's this customer's biggest order?" (`rank_within_customer = 1`)
without collapsing away their other orders.

---

## 4. LAG() and LEAD() — look at a neighboring row's value

- `LAG(column)` — value from the **previous** row (based on the `ORDER BY`)
- `LEAD(column)` — value from the **next** row

Useful for comparisons like "how much more/less than the last order,"
month-over-month change, etc.

```sql
SELECT customer_id, product, amount, order_date,
    LAG(amount) OVER (ORDER BY order_date) AS prev_order_amount,
    LEAD(amount) OVER (ORDER BY order_date) AS next_order_amount
FROM orders
WHERE customer_id != 99;
```
| customer_id | product | amount | order_date | prev_order_amount | next_order_amount |
|---|---|---|---|---|---|
| 1 | Laptop | 55000.00 | 2024-01-05 | NULL | 500.00 |
| 1 | Mouse | 500.00 | 2024-01-06 | 55000.00 | 1200.00 |
| 2 | Keyboard | 1200.00 | 2024-01-10 | 500.00 | 7500.00 |
| 3 | Desk Chair | 7500.00 | 2024-01-12 | 1200.00 | 12000.00 |
| 2 | Office Table | 12000.00 | 2024-01-15 | 7500.00 | 50.00 |
| 4 | Notebook | 50.00 | 2024-01-18 | 12000.00 | NULL |

- First row's `prev_order_amount` is `NULL` — there's nothing before it.
- Last row's `next_order_amount` is `NULL` — there's nothing after it.
- This ordering is across **all** orders (no `PARTITION BY`) — combine
  with `PARTITION BY customer_id` to compare each customer's orders only
  against their *own* previous/next order.

---

## 5. Running totals — aggregate functions used as window functions

Any aggregate (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`) can be used with
`OVER (...)` instead of `GROUP BY` to produce a running/cumulative value
per row, instead of one final number.

```sql
SELECT customer_id, product, amount, order_date,
    SUM(amount) OVER (ORDER BY order_date) AS running_total
FROM orders
WHERE customer_id != 99;
```
| customer_id | product | amount | order_date | running_total |
|---|---|---|---|---|
| 1 | Laptop | 55000.00 | 2024-01-05 | 55000.00 |
| 1 | Mouse | 500.00 | 2024-01-06 | 55500.00 |
| 2 | Keyboard | 1200.00 | 2024-01-10 | 56700.00 |
| 3 | Desk Chair | 7500.00 | 2024-01-12 | 64200.00 |
| 2 | Office Table | 12000.00 | 2024-01-15 | 76200.00 |
| 4 | Notebook | 50.00 | 2024-01-18 | 76250.00 |

Each row's `running_total` = sum of its own amount + every row before it
(in `order_date` order) — a classic cumulative sum, common in financial
reports and dashboards.

---

## Summary table

| Function | Purpose | Ties allowed / skips numbers? |
|---|---|---|
| `ROW_NUMBER()` | unique sequential number | no ties — always distinct |
| `RANK()` | rank with gaps after ties | ties share rank, next rank skips |
| `DENSE_RANK()` | rank without gaps | ties share rank, next rank doesn't skip |
| `PARTITION BY` | restart window per group | — (modifier, not a function itself) |
| `LAG()` | previous row's value | returns NULL for the first row |
| `LEAD()` | next row's value | returns NULL for the last row |
| `SUM()/AVG()/... OVER()` | running/cumulative aggregate per row | depends on ORDER BY / frame |
