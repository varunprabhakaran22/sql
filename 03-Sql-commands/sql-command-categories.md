# SQL Command Categories: DDL, DML, DCL, DQL

SQL commands are grouped into four categories based on *what kind of job*
they do. This matters because it explains why some commands (like `CREATE`)
behave very differently from others (like `SELECT`) — they're not just
"different commands," they belong to entirely different purposes.

---

## 1. DDL — Data Definition Language

**Purpose:** defines/changes the *structure* (schema) of the database —
databases, tables, columns — not the data inside them.

**Commands:** `CREATE`, `ALTER`, `DROP`, `TRUNCATE`

| Command | What it does |
|---|---|
| `CREATE` | makes a new database, table, etc. |
| `ALTER`  | modifies an existing table's structure (add/drop/change columns) |
| `DROP`   | permanently deletes a database or table (structure + all its data) |
| `TRUNCATE` | empties all rows from a table, but keeps the table structure |

### TRUNCATE vs DELETE

Both remove rows, but they work very differently under the hood —
`TRUNCATE` is DDL, `DELETE` is DML (see [section 2](#2-dml--data-manipulation-language) below):

| | `DELETE` | `TRUNCATE` |
|---|---|---|
| Category | DML | DDL |
| What it removes | Specific rows (or all, if no `WHERE`) | All rows, always — no partial delete possible |
| Can filter with `WHERE`? | ✅ Yes | ❌ No — always wipes the entire table |
| Speed | Slower — deletes row by row, logs each one | Much faster — deallocates the whole data at once |
| Auto-increment counter | Stays where it was (next insert continues the sequence) | Resets back to 1 |
| Can be rolled back? | Yes, if inside a transaction, before commit | No — auto-commits immediately, permanent |
| Fires triggers? | Yes, per-row triggers fire | No |

```sql
DELETE FROM students WHERE studentid = 1;   -- removes just student 1
TRUNCATE TABLE students;                     -- wipes ALL rows, resets id counter, no WHERE possible
```

**Rule of thumb:** use `DELETE` for specific rows or when you need transaction
safety. Use `TRUNCATE` to fully wipe a table fast when you don't care about
preserving the auto-increment history (e.g. clearing test data before reload).

Examples we've already used:
```sql
CREATE DATABASE studentsDB;
CREATE TABLE students (studentid INT PRIMARY KEY, firstname VARCHAR(50));
ALTER TABLE subject ADD COLUMN subject_name VARCHAR(50);
DROP TABLE subject;
```

> Note: DDL statements in MySQL auto-commit immediately — there's no
> "undo" once run (unlike DML, which can be rolled back before commit).

---

## 2. DML — Data Manipulation Language

**Purpose:** manages the *data* (rows) inside tables — adding, changing,
removing — without touching the table's structure.

**Commands:** `INSERT`, `UPDATE`, `DELETE`

| Command | What it does |
|---|---|
| `INSERT` | adds new row(s) to a table |
| `UPDATE` | modifies existing row(s) |
| `DELETE` | removes row(s) (structure stays intact, unlike DROP/TRUNCATE) |

Example (from [insert-statements.md](./insert-statements.md)):
```sql
INSERT INTO students (studentid, firstname, lastname, BirthDate, gender)
VALUES (1, 'John', 'Doe', '2000-05-15', 'Male');

UPDATE students SET gender = 'Male' WHERE studentid = 1;

DELETE FROM students WHERE studentid = 1;
```

---

## 3. DCL — Data Control Language

**Purpose:** controls *access* — who is allowed to do what in the database.
Permissions/security, not structure or data.

**Commands:** `GRANT`, `REVOKE`

| Command | What it does |
|---|---|
| `GRANT`  | gives a user permission to do something (e.g. SELECT, INSERT on a table) |
| `REVOKE` | takes a previously granted permission away |

Example:
```sql
GRANT SELECT, INSERT ON studentsDB.* TO 'someuser'@'localhost';
REVOKE INSERT ON studentsDB.* FROM 'someuser'@'localhost';
```

> We're using the `root` user for everything so far, which already has
> full permissions — DCL becomes relevant once multiple users/roles are
> involved (e.g. a real application with limited-permission accounts).

### Untangling "is GRANT/REVOKE a command, or is SELECT/INSERT?" — both, at different layers

`GRANT` and `REVOKE` are the actual DCL command keywords — same way
`SELECT` is a DQL command and `INSERT` is a DML command. What's easy to
misread in an example like this:

```sql
GRANT SELECT, INSERT ON studentsDB.* TO 'someuser'@'localhost';
REVOKE INSERT ON studentsDB.* FROM 'someuser'@'localhost';
```

is that `SELECT` and `INSERT` are **not being executed** here — they're
being named as **arguments**: a list of *which privileges* to hand out or
take away. `GRANT` is the command actually doing the action (granting);
`SELECT, INSERT` describe *what* is being granted.

- Line 1 reads: *"Give `someuser` permission to run `SELECT` and `INSERT`
  on every table (`.*`) in `studentsDB`."*
- Line 2 reads: *"Take away `someuser`'s `INSERT` permission — they keep
  `SELECT`, since only `INSERT` was revoked."*

So the layering is:
- `GRANT` / `REVOKE` → the DCL commands that perform granting/revoking
- `SELECT`, `INSERT`, `UPDATE`, `DELETE`, etc. → the privileges being
  granted/revoked, which *also* happen to be real commands in their own
  categories (DQL/DML) — the same keyword plays two different roles
  depending on context (executing the action, vs. naming a permission).

Analogy: `GRANT`/`REVOKE` is like giving or taking away someone's
*permission to drive*; `SELECT`/`INSERT` in that context is naming
*driving* itself as the permission — not actually driving.

---

## 4. DQL — Data Query Language

**Purpose:** *reads/retrieves* data — asking questions of the data without
changing anything.

**Command:** `SELECT` (technically the only DQL command)

```sql
SELECT * FROM students;
SELECT firstname, lastname FROM students WHERE gender = 'Female';
```

This is the category we'll spend the most time on next — filtering
(`WHERE`), sorting (`ORDER BY`), joining tables (`JOIN`), grouping
(`GROUP BY`), etc. all build on top of basic `SELECT`.

---

## Quick summary table

| Category | Full name | Affects | Key commands |
|---|---|---|---|
| DDL | Data Definition Language | Structure (schema) | CREATE, ALTER, DROP, TRUNCATE |
| DML | Data Manipulation Language | Data (rows) | INSERT, UPDATE, DELETE |
| DCL | Data Control Language | Access/permissions | GRANT, REVOKE |
| DQL | Data Query Language | Reading data | SELECT |
