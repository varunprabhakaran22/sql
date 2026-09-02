# JOINs — Tables & Worked Examples

Theory/definitions are in [joins-theory.md](./joins-theory.md). This file
has the actual schema and every join type run against real data, with
verified output.

---

## Schema

```sql
CREATE TABLE customers (
    customer_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_name VARCHAR(50),
    city VARCHAR(50)
);

CREATE TABLE orders (
    order_id INT AUTO_INCREMENT PRIMARY KEY,
    customer_id INT,
    product VARCHAR(50),
    amount DECIMAL(10,2),
    order_date DATE,
    FOREIGN KEY (customer_id) REFERENCES customers(customer_id)
);

CREATE TABLE employees (
    employee_id INT AUTO_INCREMENT PRIMARY KEY,
    employee_name VARCHAR(50),
    manager_id INT
);

INSERT INTO customers (customer_name, city) VALUES
('Ravi Kumar', 'Bangalore'),
('Anita Sharma', 'Mumbai'),
('Vikram Singh', 'Bangalore'),
('Priya Nair', 'Chennai'),
('Karan Mehta', 'Delhi');
-- Karan Mehta (id 5) deliberately has NO orders — used to demo LEFT/RIGHT/FULL JOIN gaps

INSERT INTO orders (customer_id, product, amount, order_date) VALUES
(1, 'Laptop', 55000.00, '2024-01-05'),
(1, 'Mouse', 500.00, '2024-01-06'),
(2, 'Keyboard', 1200.00, '2024-01-10'),
(3, 'Desk Chair', 7500.00, '2024-01-12'),
(2, 'Office Table', 12000.00, '2024-01-15'),
(4, 'Notebook', 50.00, '2024-01-18');

-- One deliberate "orphan" order for a customer_id that doesn't exist (99),
-- inserted with FK checks briefly disabled — used to demo RIGHT/FULL JOIN gaps
SET FOREIGN_KEY_CHECKS=0;
INSERT INTO orders (customer_id, product, amount, order_date) VALUES
(99, 'Mystery Item', 999.00, '2024-01-20');
SET FOREIGN_KEY_CHECKS=1;

INSERT INTO employees (employee_name, manager_id) VALUES
('Suresh (CEO)', NULL),
('Anjali (Manager)', 1),
('Rahul (Manager)', 1),
('Deepa (Dev)', 2),
('Kiran (Dev)', 2),
('Manoj (Dev)', 3);
```

### customers

| customer_id | customer_name | city |
|---|---|---|
| 1 | Ravi Kumar | Bangalore |
| 2 | Anita Sharma | Mumbai |
| 3 | Vikram Singh | Bangalore |
| 4 | Priya Nair | Chennai |
| 5 | Karan Mehta | Delhi |

### orders

| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 500.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |
| 7 | 99 | Mystery Item | 999.00 | 2024-01-20 |

### employees (self-referencing manager_id)

| employee_id | employee_name | manager_id |
|---|---|---|
| 1 | Suresh (CEO) | NULL |
| 2 | Anjali (Manager) | 1 |
| 3 | Rahul (Manager) | 1 |
| 4 | Deepa (Dev) | 2 |
| 5 | Kiran (Dev) | 2 |
| 6 | Manoj (Dev) | 3 |

---

## 1. INNER JOIN — only matching rows

```sql
SELECT c.customer_name, o.product, o.amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;
```
| customer_name | product | amount |
|---|---|---|
| Ravi Kumar | Laptop | 55000.00 |
| Ravi Kumar | Mouse | 500.00 |
| Anita Sharma | Keyboard | 1200.00 |
| Anita Sharma | Office Table | 12000.00 |
| Vikram Singh | Desk Chair | 7500.00 |
| Priya Nair | Notebook | 50.00 |

Notice: **Karan Mehta is missing** (no orders) and **Mystery Item is
missing** (no matching customer) — INNER JOIN only keeps rows that exist
on both sides.

---

## 2. LEFT JOIN — all of the left table, matched or not

```sql
SELECT c.customer_name, o.product, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;
```
| customer_name | product | amount |
|---|---|---|
| Ravi Kumar | Laptop | 55000.00 |
| Ravi Kumar | Mouse | 500.00 |
| Anita Sharma | Keyboard | 1200.00 |
| Anita Sharma | Office Table | 12000.00 |
| Vikram Singh | Desk Chair | 7500.00 |
| Priya Nair | Notebook | 50.00 |
| Karan Mehta | NULL | NULL |

`customers` is the "left" table here (listed first, right after `FROM`).
**Karan Mehta now appears** with `NULL` product/amount, since he has no
orders but LEFT JOIN keeps every left-side row regardless.

---

## 3. RIGHT JOIN — all of the right table, matched or not

```sql
SELECT c.customer_name, o.product, o.amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;
```
| customer_name | product | amount |
|---|---|---|
| Ravi Kumar | Laptop | 55000.00 |
| Ravi Kumar | Mouse | 500.00 |
| Anita Sharma | Keyboard | 1200.00 |
| Vikram Singh | Desk Chair | 7500.00 |
| Anita Sharma | Office Table | 12000.00 |
| Priya Nair | Notebook | 50.00 |
| NULL | Mystery Item | 999.00 |

`orders` is the "right" table here. **Mystery Item now appears** with
`NULL` customer_name, since it points to a non-existent customer (id 99)
but RIGHT JOIN keeps every right-side row regardless. Karan Mehta is
gone now, since he has no matching row on the right side.

---

## 4. FULL JOIN (simulated via UNION, since MySQL has no FULL JOIN keyword)

```sql
SELECT c.customer_name, o.product, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
UNION
SELECT c.customer_name, o.product, o.amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;
```
| customer_name | product | amount |
|---|---|---|
| Ravi Kumar | Laptop | 55000.00 |
| Ravi Kumar | Mouse | 500.00 |
| Anita Sharma | Keyboard | 1200.00 |
| Anita Sharma | Office Table | 12000.00 |
| Vikram Singh | Desk Chair | 7500.00 |
| Priya Nair | Notebook | 50.00 |
| Karan Mehta | NULL | NULL |
| NULL | Mystery Item | 999.00 |

Now **both gaps appear together** — Karan Mehta (no orders) AND Mystery
Item (no customer) — this is the full picture from both directions.
`UNION` here removes the duplicate matched rows that both the LEFT and
RIGHT queries would otherwise return twice.

---

## 5. CROSS JOIN — every combination (Cartesian product)

```sql
SELECT c.customer_name, o.product
FROM customers c
CROSS JOIN orders o
LIMIT 6;
```
| customer_name | product |
|---|---|
| Karan Mehta | Laptop |
| Priya Nair | Laptop |
| Vikram Singh | Laptop |
| Anita Sharma | Laptop |
| Ravi Kumar | Laptop |
| Karan Mehta | Mouse |

No `ON` condition at all — every customer is paired with every order.
Full result would be 5 customers × 7 orders = **35 rows** (limited to 6
here just to keep the output short). Rarely used directly in real
applications, but foundational to understand what a JOIN is doing
underneath (a JOIN is essentially a CROSS JOIN, filtered by the `ON`
condition).

---

## 6. SELF JOIN — a table joined with itself

```sql
SELECT e.employee_name AS employee, m.employee_name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```
| employee | manager |
|---|---|
| Suresh (CEO) | NULL |
| Anjali (Manager) | Suresh (CEO) |
| Rahul (Manager) | Suresh (CEO) |
| Deepa (Dev) | Anjali (Manager) |
| Kiran (Dev) | Anjali (Manager) |
| Manoj (Dev) | Rahul (Manager) |

The same `employees` table is used twice — once as `e` (the employee),
once as `m` (their manager) — joined on `e.manager_id = m.employee_id`.
Suresh (the CEO) has `manager_id = NULL`, so his manager column is NULL
too — a `LEFT JOIN` is used here (not `INNER JOIN`) specifically so the
CEO still shows up despite having no manager to match.

---

## Summary — row counts from this exact data

| Join type | Rows returned | Why |
|---|---|---|
| INNER JOIN | 6 | only matched customer↔order pairs |
| LEFT JOIN | 7 | 6 matched + 1 unmatched customer (Karan Mehta) |
| RIGHT JOIN | 7 | 6 matched + 1 unmatched order (Mystery Item) |
| FULL JOIN | 8 | 6 matched + both unmatched sides |
| CROSS JOIN | 35 (5 × 7) | every customer × every order, no condition |
| SELF JOIN | 6 | one row per employee, paired with their manager |
