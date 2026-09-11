# Indexes & Query Performance

## Level 1 — What an index actually is

Without an index, MySQL finds rows matching a `WHERE` condition by
checking **every single row** in the table, top to bottom (a "full table
scan"). On a table with 10 rows, that's instant. On a table with 10
million rows, that's slow — and this is the #1 cause of "why is my query
suddenly slow in production" once real data volume shows up.

An **index** is a separate, sorted data structure MySQL maintains
alongside a table, letting it jump directly to matching rows instead of
scanning everything — similar to a book's index letting you jump to a
page instead of reading cover to cover.

### Proof — EXPLAIN before and after an index

`EXPLAIN` shows *how* MySQL plans to run a query, without actually
running it. Verified against the `orders` table from
[05-joins](../05-joins/joins-examples.md):

**Before any index on `product`:**
```sql
EXPLAIN SELECT * FROM orders WHERE product = 'Laptop';
```
| type | possible_keys | key | rows | Extra |
|---|---|---|---|---|
| **ALL** | NULL | NULL | **7** | Using where |

`type: ALL` means a full table scan — MySQL checked all 7 rows to find
the match.

**After adding an index:**
```sql
CREATE INDEX idx_orders_product ON orders(product);

EXPLAIN SELECT * FROM orders WHERE product = 'Laptop';
```
| type | possible_keys | key | rows | Extra |
|---|---|---|---|---|
| **ref** | idx_orders_product | idx_orders_product | **1** | NULL |

`type: ref` means MySQL used the index to jump straight to matching rows
— it only had to examine **1** row instead of 7. On a real production
table with millions of rows, this is the difference between a query
taking milliseconds vs seconds (or timing out).

---

## Level 1 — Types of indexes

```sql
-- Regular index — speeds up lookups/filters on this column
CREATE INDEX idx_orders_product ON orders(product);

-- Unique index — like a regular index, but also enforces no duplicate values
CREATE UNIQUE INDEX idx_unique_email ON customers(email);

-- Composite (multi-column) index — speeds up queries filtering on BOTH columns together
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);

-- View all indexes on a table
SHOW INDEX FROM orders;

-- Remove an index
DROP INDEX idx_orders_product ON orders;
```

> A `PRIMARY KEY` is automatically an index too — that's part of why
> primary-key lookups (`WHERE id = 5`) are always fast, even without you
> creating anything extra (see
> [setup-and-basics.md](../01-intro/setup-and-basics.md) for the PK recap).
> A `FOREIGN KEY` column also gets an automatic index in MySQL (visible
> as the `MUL` marker in `DESC` — same thing you saw back in
> [05-joins](../05-joins/joins-examples.md)).

### Composite index — verified example

```sql
CREATE INDEX idx_customer_date ON orders(customer_id, order_date);

EXPLAIN SELECT * FROM orders WHERE customer_id = 1 AND order_date > '2024-01-01';
```
| type | key | rows | Extra |
|---|---|---|---|
| range | idx_customer_date | 2 | Using index condition |

**Important rule:** a composite index on `(customer_id, order_date)`
only helps if your `WHERE`/`ORDER BY` uses `customer_id` first (or
alone) — it can't be used efficiently to filter by `order_date` alone,
since the index is sorted by `customer_id` first, then `order_date`
within each customer. This column-order rule trips up a lot of people.

---

## Level 2 — When indexes hurt (they're not free)

Indexes aren't "always add more, get more speed" — there's a real
tradeoff, which is why production schemas don't index every column:

| Cost | Why |
|---|---|
| **Slower writes** | Every `INSERT`/`UPDATE`/`DELETE` must also update every index on that table — more indexes = more work per write |
| **Extra storage** | Each index is its own separate data structure taking disk space, sometimes comparable to the table itself |
| **Can be ignored anyway** | MySQL's query planner might decide a full scan is actually faster than using an index (e.g. if a `WHERE` matches most of the table — an index doesn't help if you're retrieving 90% of the rows anyway) |
| **Maintenance overhead** | On very large, frequently-updated tables, keeping many indexes in sync becomes a real operational cost DBAs actively manage |

**Practical rule of thumb:** index columns that are frequently used in
`WHERE`, `JOIN ... ON`, and `ORDER BY` — especially on large tables.
Don't index columns you rarely filter/sort by, and don't index
low-cardinality columns (e.g. a `gender` column with only 2-3 possible
values gives the index little to narrow down).

### Reading `EXPLAIN` output — key columns to know

| Column | Meaning |
|---|---|
| `type` | access method: `ALL` (full scan, worst) → `index` → `range` → `ref` → `eq_ref` → `const` (best) |
| `key` | which index MySQL actually chose to use (`NULL` = none used) |
| `rows` | estimated number of rows MySQL expects to examine — lower is better |
| `Extra` | notes like `Using where`, `Using index` (fast — didn't even need the table), `Using filesort` (slow — had to sort manually) |

## Quick summary

| Concept | Command |
|---|---|
| Create index | `CREATE INDEX idx_name ON table(column);` |
| Create unique index | `CREATE UNIQUE INDEX idx_name ON table(column);` |
| Composite index | `CREATE INDEX idx_name ON table(col1, col2);` |
| View indexes | `SHOW INDEX FROM table;` |
| Remove index | `DROP INDEX idx_name ON table;` |
| Check query plan | `EXPLAIN SELECT ...;` |
