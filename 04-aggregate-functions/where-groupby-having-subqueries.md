# WHERE, GROUP BY, HAVING, WHERE IN, Subqueries (with aggregates)

This builds on [aggregate-functions.md](./aggregate-functions.md) and
[query-execution-order.md](../03-Sql-commands/query-execution-order.md).
Same execution order applies here: `FROM → WHERE → GROUP BY → HAVING → SELECT`.

> Note: everything here uses a **single table** (`customer_name`/`city` as
> plain columns, not a separate linked table). Actually joining two
> related tables together (e.g. a real `customers` table linked to
> `sales`) is deliberately saved for **Chapter 05 — JOINs**.

---

## Schema used in this file — one rich `sales` table

```sql
CREATE TABLE sales (
    sale_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(50),
    city VARCHAR(50),
    product VARCHAR(50),
    category VARCHAR(30),
    quantity INT,
    price DECIMAL(10,2),
    sale_date DATE
);

INSERT INTO sales (customer_name, city, product, category, quantity, price, sale_date) VALUES
('Ravi Kumar', 'Bangalore', 'Laptop', 'Electronics', 2, 55000.00, '2024-01-05'),
('Ravi Kumar', 'Bangalore', 'Mouse', 'Electronics', 5, 500.00, '2024-01-06'),
('Anita Sharma', 'Mumbai', 'Keyboard', 'Electronics', 3, 1200.00, '2024-01-10'),
('Vikram Singh', 'Bangalore', 'Desk Chair', 'Furniture', 1, 7500.00, '2024-01-12'),
('Anita Sharma', 'Mumbai', 'Office Table', 'Furniture', 2, 12000.00, '2024-01-15'),
('Priya Nair', 'Chennai', 'Notebook', 'Stationery', 10, 50.00, '2024-01-18'),
('Priya Nair', 'Chennai', 'Pen', 'Stationery', 20, 10.00, '2024-01-18'),
('Ravi Kumar', 'Bangalore', 'Monitor', 'Electronics', 4, 9000.00, '2024-02-01'),
('Vikram Singh', 'Bangalore', 'Bookshelf', 'Furniture', 1, 4500.00, '2024-02-05'),
('Arjun Rao', 'Mumbai', 'Stapler', 'Stationery', 6, 150.00, '2024-02-10'),
('Anita Sharma', 'Mumbai', 'Headphones', 'Electronics', 3, 2500.00, '2024-02-14'),
('Arjun Rao', 'Mumbai', 'Sofa', 'Furniture', 1, 25000.00, '2024-02-20');
```

### Full table — `SELECT * FROM sales;`

| sale_id | customer_name | city | product | category | quantity | price | sale_date |
|---|---|---|---|---|---|---|---|
| 1 | Ravi Kumar | Bangalore | Laptop | Electronics | 2 | 55000.00 | 2024-01-05 |
| 2 | Ravi Kumar | Bangalore | Mouse | Electronics | 5 | 500.00 | 2024-01-06 |
| 3 | Anita Sharma | Mumbai | Keyboard | Electronics | 3 | 1200.00 | 2024-01-10 |
| 4 | Vikram Singh | Bangalore | Desk Chair | Furniture | 1 | 7500.00 | 2024-01-12 |
| 5 | Anita Sharma | Mumbai | Office Table | Furniture | 2 | 12000.00 | 2024-01-15 |
| 6 | Priya Nair | Chennai | Notebook | Stationery | 10 | 50.00 | 2024-01-18 |
| 7 | Priya Nair | Chennai | Pen | Stationery | 20 | 10.00 | 2024-01-18 |
| 8 | Ravi Kumar | Bangalore | Monitor | Electronics | 4 | 9000.00 | 2024-02-01 |
| 9 | Vikram Singh | Bangalore | Bookshelf | Furniture | 1 | 4500.00 | 2024-02-05 |
| 10 | Arjun Rao | Mumbai | Stapler | Stationery | 6 | 150.00 | 2024-02-10 |
| 11 | Anita Sharma | Mumbai | Headphones | Electronics | 3 | 2500.00 | 2024-02-14 |
| 12 | Arjun Rao | Mumbai | Sofa | Furniture | 1 | 25000.00 | 2024-02-20 |

This one table is rich enough (name, city, product, category, quantity,
price, date) to demonstrate every topic below without needing a second
linked table yet.

---

## 1. WHERE — filter individual rows

**Theory:** `WHERE` filters rows *before* any grouping happens, one row at
a time. Cannot use aggregate functions here (see
[query-execution-order.md](../03-Sql-commands/query-execution-order.md) —
`WHERE` runs before `GROUP BY`/`SELECT`, so aggregates don't exist yet).

**Example 1 — filter by exact match:**
```sql
SELECT product, price FROM sales WHERE category = 'Electronics';
```
| product | price |
|---|---|
| Laptop | 55000.00 |
| Mouse | 500.00 |
| Keyboard | 1200.00 |
| Monitor | 9000.00 |
| Headphones | 2500.00 |

**Example 2 — filter by comparison:**
```sql
SELECT product, price FROM sales WHERE price > 5000;
```
| product | price |
|---|---|
| Laptop | 55000.00 |
| Desk Chair | 7500.00 |
| Office Table | 12000.00 |
| Monitor | 9000.00 |
| Sofa | 25000.00 |

---

## 2. GROUP BY — bucket rows together

**Theory:** groups rows that share the same value in a column, so
aggregate functions can compute one result **per group** instead of one
result for the whole table.

**Example 1 — count sales per category:**
```sql
SELECT category, COUNT(*) AS num_sales FROM sales GROUP BY category;
```
| category | num_sales |
|---|---|
| Electronics | 5 |
| Furniture | 4 |
| Stationery | 3 |

**Example 2 — total spend per customer:**
```sql
SELECT customer_name, SUM(quantity * price) AS total_spent
FROM sales
GROUP BY customer_name;
```
| customer_name | total_spent |
|---|---|
| Ravi Kumar | 148500.00 |
| Anita Sharma | 35100.00 |
| Vikram Singh | 12000.00 |
| Priya Nair | 700.00 |
| Arjun Rao | 25900.00 |

> Rule: every column in `SELECT` that is NOT inside an aggregate function
> must also appear in `GROUP BY` (e.g. `customer_name` here).

---

## 3. HAVING — filter groups (not individual rows)

**Theory:** `HAVING` runs *after* `GROUP BY`, so — unlike `WHERE` — it
CAN use aggregate functions. Use `WHERE` to filter rows before grouping;
use `HAVING` to filter the grouped results themselves.

**Example 1 — only categories earning over ₹10,000:**
```sql
SELECT category, SUM(quantity * price) AS revenue
FROM sales
GROUP BY category
HAVING revenue > 10000;
```
| category | revenue |
|---|---|
| Electronics | 159600.00 |
| Furniture | 61000.00 |

(Stationery earned only 1600, so it's filtered out — correctly excluded
only after the grouping/summing already happened.)

**Example 2 — only customers with more than 2 orders:**
```sql
SELECT customer_name, COUNT(*) AS num_orders
FROM sales
GROUP BY customer_name
HAVING COUNT(*) > 2;
```
| customer_name | num_orders |
|---|---|
| Ravi Kumar | 3 |
| Anita Sharma | 3 |

---

## 4. WHERE IN — match against a list of values

**Theory:** shorthand for multiple `OR` conditions. `WHERE category = 'A' OR category = 'B'`
becomes `WHERE category IN ('A', 'B')` — cleaner and easier to read/extend.

**Example 1 — sales from two categories:**
```sql
SELECT product, price FROM sales WHERE category IN ('Electronics', 'Furniture');
```
Returns all 9 rows from those two categories (Laptop, Mouse, Keyboard,
Desk Chair, Office Table, Monitor, Bookshelf, Headphones, Sofa) —
i.e. everything except the 3 Stationery rows.

**Example 2 — customers from specific cities:**
```sql
SELECT DISTINCT customer_name, city FROM sales WHERE city IN ('Bangalore', 'Chennai');
```
| customer_name | city |
|---|---|
| Ravi Kumar | Bangalore |
| Vikram Singh | Bangalore |
| Priya Nair | Chennai |

> `DISTINCT` is used here because without it, a customer with multiple
> sales rows in that city would show up once per row instead of once total.

---

## 5. Subqueries (with aggregates) — a query inside a query

**Theory:** a subquery is a `SELECT` nested inside another SQL statement.
It runs first, and its result is used by the outer query — commonly
paired with `WHERE IN` (matching against a list) or a comparison operator
(matching against a single aggregate value).

**Example 1 — WHERE IN + subquery:** customers who bought something over ₹20,000
```sql
SELECT DISTINCT customer_name
FROM sales
WHERE customer_name IN (
    SELECT customer_name FROM sales WHERE price > 20000
);
```
| customer_name |
|---|
| Ravi Kumar |
| Arjun Rao |

*How it runs:* inner query first finds `customer_name`s with a sale over
₹20,000 (Ravi bought the ₹55,000 Laptop, Arjun bought the ₹25,000 Sofa),
then the outer query returns those matching names (with `DISTINCT` since
a name could otherwise repeat across matching rows).

**Example 2 — subquery returning a single aggregate value:** products priced above the overall average
```sql
SELECT product, price
FROM sales
WHERE price > (SELECT AVG(price) FROM sales);
```
| product | price |
|---|---|
| Laptop | 55000.00 |
| Office Table | 12000.00 |
| Sofa | 25000.00 |

*How it runs:* inner query computes one number (`AVG(price)` ≈ 9784.17),
then the outer query filters rows against that single value.

**Example 3 — subquery inside HAVING, using a nested subquery on grouped data:**
the category with the single highest revenue
```sql
SELECT category, SUM(quantity * price) AS revenue
FROM sales
GROUP BY category
HAVING revenue = (
    SELECT MAX(rev) FROM (
        SELECT SUM(quantity * price) AS rev FROM sales GROUP BY category
    ) AS sub
);
```
| category | revenue |
|---|---|
| Electronics | 159600.00 |

*How it runs:* the innermost query computes revenue per category, wraps
it as a temporary result named `sub`, then `MAX(rev)` finds the highest
revenue among those groups — and the outer query returns only the
category matching that max value.

---

## Summary table

| Clause/concept | Runs on | Can use aggregates? | Typical use |
|---|---|---|---|
| `WHERE` | individual rows, before grouping | ❌ No | filter raw data |
| `GROUP BY` | buckets rows by column value | — | prep for aggregates per group |
| `HAVING` | groups, after grouping | ✅ Yes | filter aggregated results |
| `WHERE IN` | individual rows | ❌ No (list can come from a subquery) | match against multiple values |
| Subquery | runs first, feeds outer query | ✅ Yes (often is one) | filter using a computed value or another result set |
