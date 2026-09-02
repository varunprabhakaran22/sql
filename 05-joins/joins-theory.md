# JOINs — Theory

A `JOIN` combines rows from two (or more) tables into a single result,
based on a related column between them (usually a foreign key → primary
key relationship, as covered back in
[setup-and-basics.md](../01-intro/setup-and-basics.md)).

This is the piece that was missing back in
[where-groupby-having-subqueries.md](../04-aggregate-functions/where-groupby-having-subqueries.md)
— subqueries can *check* a value against another table, but only a JOIN
actually merges both tables' columns into one row, side by side.

Industry-standard join types (the names used everywhere — books, job
interviews, real SQL engines):

| Join Type | Description |
|---|---|
| `INNER JOIN` | Returns only the matching rows from both tables. |
| `LEFT JOIN` (a.k.a. `LEFT OUTER JOIN`) | Returns all rows from the left table, and matched rows from the right table (unmatched right side = NULL). |
| `RIGHT JOIN` (a.k.a. `RIGHT OUTER JOIN`) | Returns all rows from the right table, and matched rows from the left table (unmatched left side = NULL). |
| `FULL JOIN` (a.k.a. `FULL OUTER JOIN`) | Returns all rows when there's a match in *either* table (unmatched side = NULL on both). |
| `CROSS JOIN` | Returns the Cartesian product of both tables — every row of table A paired with every row of table B. |
| `SELF JOIN` | Joins a table with itself — used for hierarchical/recursive relationships (e.g. employee → manager). |

---

## Important clarification: FULL JOIN is NOT the same as UNION

This is a common mix-up worth calling out directly, since it's easy to
assume "full join = union" — they are related, but not the same thing:

- **`UNION`** is a completely different SQL operation: it stacks the
  results of two *separate, already-run* `SELECT` queries on top of each
  other (combining rows vertically), removing exact duplicate rows.
  It does not need a shared/related column at all — the two queries just
  need the same number of columns with compatible types.

- **`FULL JOIN`** relates two tables *through a shared column*, matching
  rows side-by-side (horizontally) and filling in `NULL` where there's no
  match on either side.

- **The connection between them:** MySQL does **not** natively support
  `FULL JOIN` as a keyword (unlike PostgreSQL or SQL Server). So in
  MySQL, `FULL JOIN` behavior has to be *simulated* by combining a
  `LEFT JOIN` and a `RIGHT JOIN` result using `UNION`:

```sql
SELECT c.customer_name, o.product, o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
UNION
SELECT c.customer_name, o.product, o.amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;
```

This is why the confusion happens — `UNION` is genuinely *used* to build
a `FULL JOIN` in MySQL, but `UNION` itself is a separate, general-purpose
tool (stacking any two same-shaped result sets), not a join at all.

---

## Visual intuition (Venn diagrams)

**INNER JOIN** — only the overlap:

<svg width="260" height="150" viewBox="0 0 260 150" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="75" r="55" fill="#cfe3ff" fill-opacity="0.5" stroke="#4a7fd6" stroke-width="2"/>
  <circle cx="160" cy="75" r="55" fill="#ffd9b3" fill-opacity="0.5" stroke="#d68a4a" stroke-width="2"/>
  <path d="M130,26 A55,55 0 0,1 130,124 A55,55 0 0,1 130,26 Z" fill="#7d5fd6" fill-opacity="0.6"/>
  <text x="70" y="80" font-size="13" fill="#333">Left</text>
  <text x="190" y="80" font-size="13" fill="#333">Right</text>
  <text x="130" y="140" font-size="12" fill="#333" text-anchor="middle">matched rows only</text>
</svg>

**LEFT JOIN** — all of left, matched part of right:

<svg width="260" height="150" viewBox="0 0 260 150" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="75" r="55" fill="#7d5fd6" fill-opacity="0.6" stroke="#4a7fd6" stroke-width="2"/>
  <circle cx="160" cy="75" r="55" fill="none" stroke="#d68a4a" stroke-width="2"/>
  <path d="M130,26 A55,55 0 0,1 130,124 A55,55 0 0,1 130,26 Z" fill="#7d5fd6" fill-opacity="0.6"/>
  <text x="70" y="80" font-size="13" fill="#333">Left</text>
  <text x="190" y="80" font-size="13" fill="#333">Right</text>
  <text x="130" y="140" font-size="12" fill="#333" text-anchor="middle">all of left + overlap</text>
</svg>

**RIGHT JOIN** — all of right, matched part of left:

<svg width="260" height="150" viewBox="0 0 260 150" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="75" r="55" fill="none" stroke="#4a7fd6" stroke-width="2"/>
  <circle cx="160" cy="75" r="55" fill="#d68a4a" fill-opacity="0.6" stroke="#d68a4a" stroke-width="2"/>
  <path d="M130,26 A55,55 0 0,1 130,124 A55,55 0 0,1 130,26 Z" fill="#d68a4a" fill-opacity="0.6"/>
  <text x="70" y="80" font-size="13" fill="#333">Left</text>
  <text x="190" y="80" font-size="13" fill="#333">Right</text>
  <text x="130" y="140" font-size="12" fill="#333" text-anchor="middle">all of right + overlap</text>
</svg>

**FULL JOIN** — everything, matched or not:

<svg width="260" height="150" viewBox="0 0 260 150" xmlns="http://www.w3.org/2000/svg">
  <circle cx="100" cy="75" r="55" fill="#7d5fd6" fill-opacity="0.55" stroke="#4a7fd6" stroke-width="2"/>
  <circle cx="160" cy="75" r="55" fill="#d68a4a" fill-opacity="0.55" stroke="#d68a4a" stroke-width="2"/>
  <text x="70" y="80" font-size="13" fill="#333">Left</text>
  <text x="190" y="80" font-size="13" fill="#333">Right</text>
  <text x="130" y="140" font-size="12" fill="#333" text-anchor="middle">everything, both sides</text>
</svg>

**CROSS JOIN** — no relation at all, every combination (not a Venn shape —
think of it as a full grid instead):

<svg width="260" height="150" viewBox="0 0 260 150" xmlns="http://www.w3.org/2000/svg">
  <g stroke="#4a7fd6" stroke-width="1.5">
    <line x1="20" y1="20" x2="20" y2="130"/><line x1="60" y1="20" x2="60" y2="130"/>
    <line x1="100" y1="20" x2="100" y2="130"/><line x1="140" y1="20" x2="140" y2="130"/>
    <line x1="180" y1="20" x2="180" y2="130"/>
  </g>
  <g stroke="#d68a4a" stroke-width="1.5">
    <line x1="20" y1="20" x2="180" y2="20"/><line x1="20" y1="55" x2="180" y2="55"/>
    <line x1="20" y1="90" x2="180" y2="90"/><line x1="20" y1="130" x2="180" y2="130"/>
  </g>
  <text x="100" y="146" font-size="12" fill="#333" text-anchor="middle">every Left row × every Right row</text>
</svg>

**SELF JOIN** — not a new shape at all, same table playing two roles:

<svg width="260" height="110" viewBox="0 0 260 110" xmlns="http://www.w3.org/2000/svg">
  <circle cx="130" cy="50" r="45" fill="#cfe3ff" fill-opacity="0.5" stroke="#4a7fd6" stroke-width="2"/>
  <text x="130" y="45" font-size="13" fill="#333" text-anchor="middle">employees</text>
  <text x="130" y="62" font-size="11" fill="#555" text-anchor="middle">(as "e" and as "m")</text>
  <text x="130" y="100" font-size="12" fill="#333" text-anchor="middle">joined to itself via manager_id</text>
</svg>

---

## Syntax pattern (all joins except CROSS/SELF share this shape)

```sql
SELECT columns
FROM table_a a
<JOIN_TYPE> table_b b ON a.shared_column = b.shared_column;
```

`CROSS JOIN` has no `ON` condition (no relationship needed — it's every
combination). `SELF JOIN` uses the *same* table twice with two aliases.

See [joins-examples.md](./joins-examples.md) for the actual schema, real
queries, and verified output for every join type above.
