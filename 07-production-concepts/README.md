# 07 — Production-Level SQL Concepts

Everything in chapters 01–06 teaches you to *write correct queries*.
This chapter covers what's missing to *not break things* once that SQL
runs against a real, shared, always-on database — the stuff most
tutorials skip.

## Why this matters for you specifically (FE → AI/GenAI engineer path)

You won't be a full-time DBA, but as an AI/GenAI engineer you will
constantly touch SQL for: querying data to build datasets/evals, logging
LLM calls to a database, reading feature stores, managing metadata next
to a vector DB, and debugging "why is this query slow / why did this
write corrupt something" in a shared production database. The topics
below are picked for that reality — not deep DBA topics like replication
or sharding, which you're unlikely to own directly.

## Files in this chapter

| File | Covers |
|---|---|
| [transactions-and-rollback.md](./transactions-and-rollback.md) | ACID, `COMMIT`/`ROLLBACK`, `SAVEPOINT`, isolation levels, dirty reads |
| [indexes-and-performance.md](./indexes-and-performance.md) | Index types, `EXPLAIN`, when indexes help vs hurt |
| [constraints-and-integrity.md](./constraints-and-integrity.md) | `NOT NULL`/`UNIQUE`/`CHECK`/`DEFAULT`, `ON DELETE CASCADE`/`SET NULL` |
| [normalization-and-schema-design.md](./normalization-and-schema-design.md) | 1NF/2NF/3NF, why we split `customers`/`orders`, denormalization tradeoffs |

All examples reuse the `orders`/`customers` tables from
[05-joins](../05-joins/joins-examples.md) wherever possible, so you're
seeing production concepts applied to data you already know instead of
a brand new unfamiliar schema.
