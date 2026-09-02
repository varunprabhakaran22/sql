# Aggregate Functions — Top 7

Aggregate functions take *many rows* and collapse them into a **single
summary value** (or one value per group, when used with `GROUP BY`).
They belong to DQL (used inside `SELECT`) — see
[sql-command-categories.md](../03-Sql-commands/sql-command-categories.md).

They always run on the `SELECT` step of the query, so remember the
execution order from [query-execution-order.md](../03-Sql-commands/query-execution-order.md):
`FROM → WHERE → GROUP BY → HAVING → SELECT`. This is why aggregates can be
filtered with `HAVING` but not `WHERE`.

---

## Practice table: `sales`

```sql
CREATE TABLE sales (
    sale_id INT AUTO_INCREMENT PRIMARY KEY,
    product VARCHAR(50),
    category VARCHAR(30),
    quantity INT,
    price DECIMAL(10,2),
    sale_date DATE
);

INSERT INTO sales (product, category, quantity, price, sale_date) VALUES
('Laptop', 'Electronics', 2, 55000.00, '2024-01-05'),
('Mouse', 'Electronics', 5, 500.00, '2024-01-06'),
('Keyboard', 'Electronics', 3, 1200.00, '2024-01-10'),
('Desk Chair', 'Furniture', 1, 7500.00, '2024-01-12'),
('Office Table', 'Furniture', 2, 12000.00, '2024-01-15'),
('Notebook', 'Stationery', 10, 50.00, '2024-01-18'),
('Pen', 'Stationery', 20, 10.00, '2024-01-18'),
('Monitor', 'Electronics', 4, 9000.00, '2024-02-01'),
('Bookshelf', 'Furniture', 1, 4500.00, '2024-02-05'),
('Stapler', 'Stationery', 6, 150.00, '2024-02-10'),
('Headphones', 'Electronics', 3, 2500.00, '2024-02-14'),
('Sofa', 'Furniture', 1, 25000.00, '2024-02-20');
```

Resulting table (`SELECT * FROM sales;`):

| sale_id | product | category | quantity | price | sale_date |
|---|---|---|---|---|---|
| 1 | Laptop | Electronics | 2 | 55000.00 | 2024-01-05 |
| 2 | Mouse | Electronics | 5 | 500.00 | 2024-01-06 |
| 3 | Keyboard | Electronics | 3 | 1200.00 | 2024-01-10 |
| 4 | Desk Chair | Furniture | 1 | 7500.00 | 2024-01-12 |
| 5 | Office Table | Furniture | 2 | 12000.00 | 2024-01-15 |
| 6 | Notebook | Stationery | 10 | 50.00 | 2024-01-18 |
| 7 | Pen | Stationery | 20 | 10.00 | 2024-01-18 |
| 8 | Monitor | Electronics | 4 | 9000.00 | 2024-02-01 |
| 9 | Bookshelf | Furniture | 1 | 4500.00 | 2024-02-05 |
| 10 | Stapler | Stationery | 6 | 150.00 | 2024-02-10 |
| 11 | Headphones | Electronics | 3 | 2500.00 | 2024-02-14 |
| 12 | Sofa | Furniture | 1 | 25000.00 | 2024-02-20 |

---

## 1. COUNT() — how many rows

Counts the number of rows (or non-NULL values in a specific column).

```sql
SELECT COUNT(*) AS total_sales FROM sales;
```
**Result:** `total_sales = 12` (all 12 sale records)

> `COUNT(*)` counts all rows regardless of NULLs. `COUNT(column_name)`
> counts only rows where that column is NOT NULL — useful when a column
> has missing data.

---

## 2. SUM() — total of a numeric column

Adds up all values. Very commonly combined with an expression like
`quantity * price` to get total revenue, not just a raw column sum.

```sql
SELECT SUM(quantity * price) AS total_revenue FROM sales;
```
**Result:** `total_revenue = 222200.00`

---

## 3. AVG() — average (mean)

```sql
SELECT AVG(price) AS avg_price FROM sales;
```
**Result:** `avg_price = 9784.166667` (average unit price across all 12 rows)

---

## 4. MIN() — smallest value

```sql
SELECT MIN(price) AS cheapest_item FROM sales;
```
**Result:** `cheapest_item = 10.00` (the Pen)

---

## 5. MAX() — largest value

```sql
SELECT MAX(price) AS most_expensive_item FROM sales;
```
**Result:** `most_expensive_item = 55000.00` (the Laptop)

---

## 6. GROUP_CONCAT() — merge values from multiple rows into one string

Less universal than the first 5, but extremely useful in practice —
combines values from a group into a single comma-separated (or custom
separator) string, instead of returning multiple rows.

```sql
SELECT GROUP_CONCAT(product SEPARATOR ', ') AS all_products
FROM sales
WHERE category = 'Electronics';
```
**Result:** `all_products = 'Laptop, Mouse, Keyboard, Monitor, Headphones'`

---

## 7. STDDEV() — standard deviation

Measures how spread out the values are from the average. High STDDEV =
values vary a lot; low STDDEV = values are close together. Common in
data analysis to spot pricing inconsistency, grading spread, etc.

```sql
SELECT STDDEV(price) AS price_stddev FROM sales;
```
**Result:** `price_stddev = 15304.4277...` (prices vary widely — from ₹10 pens to ₹55,000 laptops)

---

## Combining aggregates with GROUP BY (the real power move)

All 5 core aggregates become far more useful grouped by a column —
this gives one summary value **per group** instead of one for the whole table.

```sql
SELECT category, SUM(quantity * price) AS category_revenue
FROM sales
GROUP BY category;
```

| category | category_revenue |
|---|---|
| Electronics | 159600.00 |
| Furniture | 61000.00 |
| Stationery | 1600.00 |

This is exactly the DQL execution flow from
[query-execution-order.md](../03-Sql-commands/query-execution-order.md):
`FROM sales → GROUP BY category → SELECT (aggregate per group)`.

To then filter which *groups* show up (e.g. only categories earning over
₹50,000), use `HAVING`, not `WHERE`:

```sql
SELECT category, SUM(quantity * price) AS category_revenue
FROM sales
GROUP BY category
HAVING SUM(quantity * price) > 50000;
```

---

## Quick summary table

| Function | What it returns | Example use case |
|---|---|---|
| `COUNT()` | number of rows | how many sales happened |
| `SUM()`   | total of a column/expression | total revenue |
| `AVG()`   | mean value | average sale price |
| `MIN()`   | smallest value | cheapest product |
| `MAX()`   | largest value | most expensive product |
| `GROUP_CONCAT()` | merges group values into one string | list all products in a category |
| `STDDEV()` | spread/variation of values | how inconsistent prices are |
