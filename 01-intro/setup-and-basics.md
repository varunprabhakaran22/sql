# 01 - Intro: MySQL Setup, DB & Table Basics

## MySQL server control (macOS, installed via Homebrew)

Check if the server is running:
```bash
ps aux | grep mysqld | grep -v grep
```
(prints a line if running, nothing if stopped — most reliable check)

Start manually (only when needed, no auto-start):
```bash
mysql.server start
```

Stop when done (frees memory):
```bash
mysql.server stop
```

Connect as root:
```bash
mysql -u root -p
```
(enter password when prompted — nothing shows on screen while typing, that's normal)

> Note: `brew services start/stop mysql@8.0` registers MySQL as an
> auto-start-on-login background service — we deliberately avoided this
> and use `mysql.server start/stop` instead for manual, on-demand control.

## Key concept: statements, not lines

MySQL doesn't execute line-by-line. It waits until it sees a semicolon `;`
before running anything. Until then it shows a continuation prompt `->`
and keeps appending whatever you type next to the same statement.

- ✅ Success → prints `Query OK, ... (0.0X sec)`
- ❌ Error → prints `ERROR <code> (<sqlstate>): <message>` explaining what's wrong, and nothing is applied

If stuck at a `->` prompt, type `;` and enter to force it to try running
(and likely error) so you can start clean again.

## Database vs Table commands

```sql
SHOW DATABASES;        -- lists every database on the server (not context-specific)
USE studentsDB;         -- switches your session INTO a specific database
SELECT DATABASE();      -- shows which database your session is currently "inside"
SHOW TABLES;             -- lists tables inside the currently USE'd database
DESC table_name;         -- shows column structure of a table (or DESCRIBE table_name)
SHOW CREATE TABLE table_name;  -- shows full CREATE statement incl. constraints/foreign keys
                                -- (DESC does NOT show foreign key relationships, only SHOW CREATE TABLE does)
```

## Primary Key

- A column (or set of columns) that uniquely identifies each row in a table.
- Cannot be NULL, cannot repeat.
- MySQL does not force you to have one, but every table should — without it,
  updates/deletes/joins on a specific row become unreliable.
- Declared with `PRIMARY KEY` (two words, no underscore).

Example:
```sql
CREATE TABLE student(
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(50),
    dob DATE
);
```

## Foreign Key

- A column whose values must match an existing value in another table's
  (usually primary key) column. Enforces relationships between tables and
  prevents orphaned/invalid references.
- In `DESC`, a column that's part of a foreign key shows `MUL` in the Key column
  (short for a non-unique index MySQL auto-creates to support the FK) —
  the relationship itself is NOT visible in `DESC`, only via `SHOW CREATE TABLE`.

Example (added after table creation, via ALTER):
```sql
ALTER TABLE marksheet ADD COLUMN student_id INT;
ALTER TABLE marksheet ADD FOREIGN KEY (student_id) REFERENCES student(id);
```

To remove a foreign key (must drop the constraint before dropping its column):
```sql
ALTER TABLE subject DROP FOREIGN KEY subject_ibfk_1;  -- get exact name from SHOW CREATE TABLE
ALTER TABLE subject DROP COLUMN student_id;
```

## Today's final schema — studentsDB

Reset and rebuilt to match course material (students / courses / enrollment):

```sql
DROP DATABASE studentsDB;
CREATE DATABASE studentsDB;
USE studentsDB;

CREATE TABLE students (
    studentid INT PRIMARY KEY,
    firstname VARCHAR(50),
    lastname VARCHAR(50),
    BirthDate DATE,
    gender VARCHAR(10)
);

CREATE TABLE courses (
    CourseID INT PRIMARY KEY,
    CourseName VARCHAR(100),
    Credits INT
);

CREATE TABLE enrollment (
    EnrollmentID INT AUTO_INCREMENT PRIMARY KEY,
    StudentID INT,
    CourseID INT,
    EnrollmentDate DATE,
    FOREIGN KEY (StudentID) REFERENCES students(studentid),
    FOREIGN KEY (CourseID) REFERENCES courses(CourseID)
);
```

`enrollment` is the classic "junction table" pattern — it links `students`
and `courses` via two foreign keys, representing a many-to-many relationship
(one student can enroll in many courses, one course can have many students).

Sample data for these tables (students, courses, enrollment rows) is
inserted in [02-inserting-values](../02-inserting-values/) using `INSERT`.
