# Constraints & Data Integrity

## Level 1 — Column-level constraints

Constraints stop bad data from ever entering the table — enforced by
MySQL itself, not left to application code to remember (which is
unreliable — some app, script, or manual query will eventually skip
that check).

```sql
CREATE TABLE products_demo (
    product_id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50) NOT NULL,
    price DECIMAL(10,2) CHECK (price > 0),
    stock INT DEFAULT 0,
    sku VARCHAR(20) UNIQUE
);
```

| Constraint | What it enforces |
|---|---|
| `NOT NULL` | column can never be left empty |
| `UNIQUE` | no two rows can have the same value in this column |
| `CHECK (condition)` | value must satisfy a condition |
| `DEFAULT value` | if not provided, use this value automatically |
| `PRIMARY KEY` | unique identifier — combines `NOT NULL` + `UNIQUE` (recap: [setup-and-basics.md](../01-intro/setup-and-basics.md)) |

### Verified: CHECK constraint actually blocking bad data

```sql
INSERT INTO products_demo (name, price) VALUES ('Valid Item', 100.00);
-- Query OK, 1 row affected

INSERT INTO products_demo (name, price) VALUES ('Invalid Item', -50.00);
-- ERROR 3819 (HY000): Check constraint 'products_demo_chk_1' is violated.
```

This is the point: the negative price was **rejected by the database
itself**, before it could ever be stored — not caught later by some
report noticing bad data.

---

## Level 2 — Foreign key actions: what happens on DELETE/UPDATE

We've used plain `FOREIGN KEY (col) REFERENCES table(col)` since
[01-intro](../01-intro/setup-and-basics.md), which by default **blocks**
deleting a referenced row (you saw this exact error back when reducing
`subject`'s foreign key — `Cannot add or update a child row`). In
production, you often want to explicitly *choose* what should happen
instead of just blocking it:

```sql
FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE CASCADE
FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE SET NULL
FOREIGN KEY (order_id) REFERENCES orders(order_id) ON DELETE RESTRICT  -- (default behavior)
```

| Option | Behavior when the parent row is deleted |
|---|---|
| `RESTRICT` (default) | Blocks the delete entirely if child rows reference it |
| `CASCADE` | Automatically deletes the matching child rows too |
| `SET NULL` | Keeps the child rows, but sets their FK column to `NULL` (column must allow NULL) |
| `NO ACTION` | Same as RESTRICT in MySQL |

### Verified: ON DELETE CASCADE in action

```sql
CREATE TABLE orders_demo (
    order_id INT PRIMARY KEY,
    customer_id INT
);

CREATE TABLE order_items_demo (
    item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT,
    product VARCHAR(50),
    FOREIGN KEY (order_id) REFERENCES orders_demo(order_id) ON DELETE CASCADE
);

INSERT INTO orders_demo VALUES (1, 100), (2, 101);
INSERT INTO order_items_demo (order_id, product) VALUES (1, 'Laptop'), (1, 'Mouse'), (2, 'Keyboard');
```

Before deleting order 1:
| item_id | order_id | product |
|---|---|---|
| 1 | 1 | Laptop |
| 2 | 1 | Mouse |
| 3 | 2 | Keyboard |

```sql
DELETE FROM orders_demo WHERE order_id = 1;
```

After — **Laptop and Mouse (order 1's items) were automatically deleted
too**, with no separate `DELETE` needed for `order_items_demo`:
| item_id | order_id | product |
|---|---|---|
| 3 | 2 | Keyboard |

**Why this matters in production:** without `CASCADE` (or a deliberate
choice of `SET NULL`), deleting a parent row either fails loudly
(`RESTRICT`) or — worse, if enforced only in application code and
someone forgets — silently leaves orphaned child rows pointing at
nothing, corrupting your data over time.

**Practical rule of thumb:**
- Use `CASCADE` when the child truly has no meaning without the parent
  (e.g. order items without an order).
- Use `SET NULL` when the child can meaningfully exist standalone (e.g.
  keep a log entry but null out a deleted user's reference).
- Use the default `RESTRICT` when you want deletes to force a conscious
  decision (e.g. don't let someone delete a customer with existing orders
  without handling those orders first).

## Quick summary

| Constraint | Level | Purpose |
|---|---|---|
| `NOT NULL` | 1 | column can't be empty |
| `UNIQUE` | 1 | no duplicate values |
| `CHECK (...)` | 1 | value must pass a condition |
| `DEFAULT` | 1 | auto-fill if not provided |
| `ON DELETE CASCADE` | 2 | auto-delete dependent rows |
| `ON DELETE SET NULL` | 2 | null out the reference instead of blocking/deleting |
| `ON DELETE RESTRICT` (default) | 2 | block the delete if children exist |
