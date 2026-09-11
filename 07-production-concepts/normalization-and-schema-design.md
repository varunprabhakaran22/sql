# Normalization & Schema Design

## Level 1 — Why we split customers and orders into two tables at all

Back in [05-joins](../05-joins/joins-examples.md), `customer_name` and
`city` live in `customers`, while `product`/`amount` live in `orders`,
linked by `customer_id`. This wasn't arbitrary — it follows
**normalization**: organizing a schema to avoid storing the same
information redundantly.

**What it would look like *without* normalization** — one big flat table:

| order_id | customer_name | city | product | amount |
|---|---|---|---|---|
| 1 | Ravi Kumar | Bangalore | Laptop | 55000 |
| 2 | Ravi Kumar | Bangalore | Mouse | 500 |
| 3 | Anita Sharma | Mumbai | Keyboard | 1200 |

Notice `Ravi Kumar` and `Bangalore` are repeated on every one of his
orders. This causes real problems:

- **Update anomaly** — if Ravi moves to Chennai, you must update *every
  row* he appears in. Miss one, and now his data is inconsistent (some
  rows say Bangalore, some say Chennai — which is correct?).
- **Insert anomaly** — you can't add a new customer until they place an
  order (there's nowhere else to put their name/city).
- **Delete anomaly** — if you delete Ravi's only order, his customer
  info (name, city) disappears too, since it was never stored
  separately.

Splitting into `customers` + `orders` (like we did) fixes all three: a
customer's city lives in exactly **one place**, updated once.

---

## Level 1 — The Normal Forms (1NF, 2NF, 3NF)

Normalization is done in progressive stages, each fixing a specific kind
of redundancy:

| Form | Rule | Fixes |
|---|---|---|
| **1NF** (First Normal Form) | Every column holds a single, atomic value — no comma-separated lists or repeating groups in one cell | e.g. a `products` column holding `"Laptop, Mouse, Keyboard"` in one cell — split into separate rows instead |
| **2NF** | Must be in 1NF, AND every non-key column depends on the **whole** primary key (only matters for composite/multi-column primary keys) | e.g. in a table keyed by `(order_id, product_id)`, a `customer_city` column would only depend on `order_id`, not the full key — it doesn't belong here |
| **3NF** | Must be in 2NF, AND no non-key column depends on another **non-key** column (no "transitive" dependency) | e.g. storing both `customer_id` and `customer_city` in `orders` — city depends on customer, not on the order itself, so it belongs in `customers` instead |

**Our `customers`/`orders` split already satisfies 3NF**: `orders` only
stores things that truly depend on the order itself (product, amount,
date) plus a reference (`customer_id`) — never city, which depends on
the *customer*, not the order.

### A 1NF violation example, made concrete

```sql
-- ❌ Violates 1NF — multiple values crammed into one column
CREATE TABLE bad_orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(50),
    products VARCHAR(200)  -- e.g. 'Laptop, Mouse, Keyboard'
);
```
```sql
-- ✅ 1NF-compliant — one product per row (this is exactly our order_items pattern)
CREATE TABLE good_orders (
    order_id INT PRIMARY KEY,
    customer_name VARCHAR(50)
);
CREATE TABLE order_items (
    item_id INT AUTO_INCREMENT PRIMARY KEY,
    order_id INT,
    product VARCHAR(50),
    FOREIGN KEY (order_id) REFERENCES good_orders(order_id)
);
```

---

## Level 2 — Denormalization: when breaking these rules is the right call

Normalization optimizes for **data integrity and avoiding redundancy** —
but it costs you **read performance**, because getting a full picture
(e.g. "customer name + their order") now requires a `JOIN` instead of
reading one table.

**Denormalization** is the deliberate choice to duplicate some data back
in, trading storage/integrity risk for faster reads — common in:

- **Reporting/analytics tables** — a `daily_sales_summary` table might
  store `customer_name` directly alongside `total_spent`, purely to
  avoid a `JOIN` every time a dashboard loads.
- **High-read, low-write systems** — if a table is read 10,000x more
  than it's written, the update-anomaly risk from normalization matters
  far less than raw read speed.
- **Caching computed values** — e.g. storing a `total_orders` count
  directly on the `customers` table (updated via a trigger or
  application logic) instead of running `COUNT()` with a `JOIN` every
  single time it's needed.

**Practical rule of thumb, especially relevant to AI/GenAI work:**
transactional/source-of-truth tables (user accounts, orders, billing)
should stay normalized. Tables built specifically for fast reads —
analytics dashboards, feature stores, RAG metadata lookup tables,
pre-computed embeddings-adjacent metadata — often intentionally
denormalize, because they're read constantly and rebuilt/refreshed from
the normalized source of truth rather than edited directly.

## Quick summary

| Concept | One-line takeaway |
|---|---|
| 1NF | one value per cell, no lists crammed into a column |
| 2NF | non-key columns depend on the *whole* primary key |
| 3NF | non-key columns depend *only* on the key, not on each other |
| Normalization tradeoff | less redundancy, safer updates — but more JOINs to read |
| Denormalization tradeoff | faster reads — but risk of inconsistent duplicated data |
