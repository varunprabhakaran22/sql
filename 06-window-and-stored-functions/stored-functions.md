# Stored Functions & Stored Procedures

Theory recap is in [theory.md](./theory.md). This file has a real
function and a real procedure created against `studentsDB`, with
verified output.

Uses the same `orders` table from [05-joins](../05-joins/joins-examples.md).

---

## 1. Stored Function — returns a single value

**Goal:** given a `customer_id`, return their total amount spent.

```sql
DELIMITER //

CREATE FUNCTION get_customer_total(cust_id INT)
RETURNS DECIMAL(10,2)
DETERMINISTIC
READS SQL DATA
BEGIN
    DECLARE total DECIMAL(10,2);
    SELECT SUM(amount) INTO total FROM orders WHERE customer_id = cust_id;
    RETURN IFNULL(total, 0);
END //

DELIMITER ;
```

**What each piece means:**
- `CREATE FUNCTION get_customer_total(cust_id INT)` — defines a function
  named `get_customer_total`, taking one parameter `cust_id`.
- `RETURNS DECIMAL(10,2)` — declares the function always returns a decimal number.
- `DETERMINISTIC` — tells MySQL this function always returns the same
  result for the same input (required for some MySQL configurations).
- `READS SQL DATA` — declares the function reads from tables (but doesn't modify them).
- `DECLARE total DECIMAL(10,2);` — creates a local variable to hold the result.
- `SELECT SUM(amount) INTO total ...` — runs a query and stores its result into that variable.
- `RETURN IFNULL(total, 0);` — returns the total, or `0` instead of `NULL`
  if the customer has no orders at all.

**Calling it** (used inside a `SELECT`, just like a built-in function):
```sql
SELECT get_customer_total(1) AS ravi_total;
```
| ravi_total |
|---|
| 55500.00 |

```sql
SELECT get_customer_total(4) AS priya_total;
```
| priya_total |
|---|
| 50.00 |

---

## 2. Stored Procedure — runs a multi-step operation, returns a result set

**Goal:** given a minimum amount, list all orders above it.

```sql
DELIMITER //

CREATE PROCEDURE get_orders_above(IN min_amount DECIMAL(10,2))
BEGIN
    SELECT customer_id, product, amount
    FROM orders
    WHERE amount > min_amount
    ORDER BY amount DESC;
END //

DELIMITER ;
```

**What each piece means:**
- `CREATE PROCEDURE get_orders_above(IN min_amount DECIMAL(10,2))` —
  defines a procedure with one **input** parameter (`IN` — as opposed to
  `OUT` for returning a value by reference, or `INOUT` for both).
- The body is a plain `SELECT` — procedures can contain any number of
  statements (multiple queries, `INSERT`/`UPDATE`, conditionals, loops),
  unlike a function which is limited mostly to reads and a single `RETURN`.

**Calling it** (with `CALL`, not inside a `SELECT`):
```sql
CALL get_orders_above(5000);
```
| customer_id | product | amount |
|---|---|---|
| 1 | Laptop | 55000.00 |
| 2 | Office Table | 12000.00 |
| 3 | Desk Chair | 7500.00 |

---

## Why use these at all?

- **Reusability** — write the logic once, call it from anywhere (other
  queries, other procedures, application code) instead of retyping the
  same `SELECT`/calculation every time.
- **Consistency** — if the business logic changes (e.g. how "total
  spend" is calculated), update it in one place.
- **Performance** — stored routines are precompiled by MySQL, and can
  reduce back-and-forth between an application and the database for
  multi-step operations.

## Managing stored routines

```sql
SHOW FUNCTION STATUS WHERE Db = 'studentsDB';   -- list functions
SHOW PROCEDURE STATUS WHERE Db = 'studentsDB';  -- list procedures

DROP FUNCTION IF EXISTS get_customer_total;
DROP PROCEDURE IF EXISTS get_orders_above;
```

## Quick summary table

| | Function | Procedure |
|---|---|---|
| Keyword to create | `CREATE FUNCTION` | `CREATE PROCEDURE` |
| Must declare a return type? | Yes (`RETURNS ...`) | No |
| How it's called | `SELECT my_func(x)` | `CALL my_procedure(x)` |
| Can return multiple rows? | No — single value only | Yes — full result sets |
| Parameter modes | Implicit input only | `IN`, `OUT`, `INOUT` |
