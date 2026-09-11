# Transactions, COMMIT, ROLLBACK, SAVEPOINT, Isolation Levels

## Level 1 — Why this exists at all

Every statement we've run so far (`INSERT`, `UPDATE`, `DELETE`) has
executed and committed **immediately** — the instant you hit enter, it's
permanent. That's fine for solo learning, but dangerous in production:

- What if step 2 of a 3-step operation fails halfway? (e.g. transferring
  money: deduct from account A succeeds, but crediting account B fails —
  now money has vanished)
- What if two people modify the same row at the same time?

A **transaction** groups multiple statements into one all-or-nothing
unit: either *all* of them succeed and get saved, or *none* of them do.
This is the foundation of reliable, concurrent, production databases.

### ACID — the four guarantees a transaction gives you

| Letter | Guarantee | Plain meaning |
|---|---|---|
| **A**tomicity | All-or-nothing | Either every statement in the transaction applies, or none do |
| **C**onsistency | Valid state → valid state | The database's rules (constraints, FKs) are never violated, even mid-transaction |
| **I**solation | Transactions don't interfere | Two transactions running at once don't see each other's uncommitted changes (tunable — see Level 2) |
| **D**urability | Once committed, it's permanent | A committed transaction survives a crash/power loss |

---

## Level 1 — Basic usage: START TRANSACTION, COMMIT, ROLLBACK

```sql
START TRANSACTION;
UPDATE orders SET amount = amount - 5000 WHERE customer_id = 1 AND product = 'Laptop';
INSERT INTO orders (customer_id, product, amount, order_date) VALUES (2, 'Webcam', 3000.00, '2024-03-01');
-- if something looks wrong here, undo everything above:
ROLLBACK;
-- if everything looks correct instead, make it permanent:
-- COMMIT;
```

**Verified run** against the `orders` table from
[05-joins](../05-joins/joins-examples.md):

Before:
| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 500.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |

Mid-transaction, uncommitted (Laptop reduced by 5000, new Webcam order added):
| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | **50000.00** | 2024-01-05 |
| 2 | 1 | Mouse | 500.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |
| 8 | 2 | **Webcam** | **3000.00** | 2024-03-01 |

After `ROLLBACK` — **exactly back to the original state**, as if nothing happened:
| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 500.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |

> Note: the `order_id` jumps to 8 because `AUTO_INCREMENT` does **not**
> roll back — MySQL never reuses an id once it's been allocated, even if
> the row itself is rolled back. This is expected and harmless.

---

## Level 1 — SAVEPOINT: partial rollback within a transaction

A `SAVEPOINT` marks a checkpoint inside a transaction, so you can undo
*part* of it without throwing away everything.

```sql
START TRANSACTION;
UPDATE orders SET amount = amount + 100 WHERE product = 'Mouse';
SAVEPOINT after_mouse_update;
DELETE FROM orders WHERE product = 'Notebook';
-- realize the DELETE was a mistake, but keep the Mouse price update:
ROLLBACK TO after_mouse_update;
COMMIT;
```

**Verified run:**

After the Mouse update + Notebook delete (both uncommitted):
| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 600.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
*(Notebook is gone here)*

After `ROLLBACK TO after_mouse_update` — **Notebook is back, but Mouse's
price increase is kept**:
| order_id | customer_id | product | amount | order_date |
|---|---|---|---|---|
| 1 | 1 | Laptop | 55000.00 | 2024-01-05 |
| 2 | 1 | Mouse | 600.00 | 2024-01-06 |
| 3 | 2 | Keyboard | 1200.00 | 2024-01-10 |
| 5 | 2 | Office Table | 12000.00 | 2024-01-15 |
| 4 | 3 | Desk Chair | 7500.00 | 2024-01-12 |
| 6 | 4 | Notebook | 50.00 | 2024-01-18 |

This is the key difference from a plain `ROLLBACK`: `ROLLBACK TO
savepoint_name` only undoes statements *after* that savepoint, keeping
everything before it — including work still not yet permanently
committed (that only happens at the final `COMMIT`).

---

## Level 2 — Isolation levels: what one transaction can see of another

When multiple transactions run **at the same time** (the normal case in
production, with many users/services hitting the same database), the
"Isolation" in ACID becomes tunable — how much one transaction is
allowed to see of another's *uncommitted* changes.

| Isolation Level | Allows dirty reads? | Allows non-repeatable reads? | Allows phantom reads? | Notes |
|---|---|---|---|---|
| `READ UNCOMMITTED` | ✅ Yes | ✅ Yes | ✅ Yes | Fastest, least safe — almost never used in production |
| `READ COMMITTED` | ❌ No | ✅ Yes | ✅ Yes | Common default in other databases (e.g. PostgreSQL) |
| `REPEATABLE READ` | ❌ No | ❌ No | ✅ Yes (partially, InnoDB mitigates this) | **MySQL's default** |
| `SERIALIZABLE` | ❌ No | ❌ No | ❌ No | Safest, slowest — transactions effectively run one at a time |

**Key terms:**
- **Dirty read** — reading another transaction's *uncommitted* change,
  which might get rolled back later (you'd have read data that never
  really existed).
- **Non-repeatable read** — reading the same row twice in one
  transaction and getting different values, because another transaction
  committed a change in between.
- **Phantom read** — re-running the same query twice in one transaction
  and getting a *different number of rows*, because another transaction
  inserted/deleted matching rows in between.

```sql
-- check current isolation level
SELECT @@transaction_isolation;

-- set it for just the next transaction
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
START TRANSACTION;
...
COMMIT;
```

**Practical takeaway for you:** you'll rarely need to change this — MySQL's
default (`REPEATABLE READ`) is a safe, sensible choice for almost
everything. Know it exists so that when someone says "we had a race
condition" or "this read stale data," you understand *why* that's a
concept, not a bug in your query itself.

## Quick summary

| Command | What it does |
|---|---|
| `START TRANSACTION` | begins a group of statements to be treated as one unit |
| `COMMIT` | makes all changes in the transaction permanent |
| `ROLLBACK` | undoes all changes in the transaction |
| `SAVEPOINT name` | marks a checkpoint inside a transaction |
| `ROLLBACK TO name` | undoes only statements after that checkpoint |
| `SET TRANSACTION ISOLATION LEVEL ...` | controls what one transaction can see of another's uncommitted work |
