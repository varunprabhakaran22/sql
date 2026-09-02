# 02 - Inserting Values (INSERT)

## Basic syntax

```sql
INSERT INTO table_name (col1, col2, col3) VALUES
(val1, val2, val3),
(val1, val2, val3);
```
- Column list is optional but recommended (makes intent explicit, order-safe).
- Multiple rows can be inserted in one statement, comma-separated.
- String/date values go in quotes `'...'`; numbers don't.

## Sample data loaded into studentsDB

```sql
INSERT INTO students (studentid, firstname, lastname, BirthDate, gender) VALUES
(1, 'John', 'Doe', '2000-05-15', 'Male'),
(2, 'Jane', 'Smith', '1999-04-12', 'Female'),
(3, 'Emily', 'Johnson', '2001-07-22', 'Female'),
(4, 'Michael', 'Williams', '2000-12-30', 'Male'),
(5, 'Sarah', 'Brown', '1998-10-10', 'Female'),
(6, 'David', 'Jones', '2002-03-25', 'Male'),
(7, 'Emma', 'Garcia', '2000-11-08', 'Female'),
(8, 'James', 'Martinez', '1999-01-01', 'Male'),
(9, 'Olivia', 'Hernandez', '2001-08-30', 'Female'),
(10, 'William', 'Lopez', '2000-02-14', 'Male');

INSERT INTO courses (CourseID, CourseName, Credits) VALUES
(1, 'Mathematics', 3),
(2, 'Computer Science', 4),
(3, 'Biology', 3),
(4, 'Chemistry', 4),
(5, 'Physics', 3),
(6, 'Literature', 2),
(7, 'History', 3),
(8, 'Economics', 3),
(9, 'Engineering', 4),
(10, 'Data Science', 4);

INSERT INTO enrollment (EnrollmentID, StudentID, CourseID, EnrollmentDate) VALUES
(1, 1, 1, '2021-08-20'),
(2, 1, 2, '2021-08-20'),
(3, 2, 1, '2021-08-20'),
(4, 2, 3, '2021-08-20'),
(5, 3, 4, '2021-08-20'),
(6, 4, 2, '2022-01-15'),
(7, 5, 3, '2021-08-20'),
(8, 6, 5, '2022-01-15'),
(9, 7, 6, '2021-08-20'),
(10, 8, 7, '2022-01-15'),
(11, 9, 8, '2021-08-20');
```

> Note: enrollment only goes up to EnrollmentID 11 here — the course
> screenshot's table was scrolled/cut off after that row. Add any
> remaining rows here if more were shown later in the video.

## Verifying inserted data

```sql
SELECT * FROM students;
SELECT * FROM courses;
SELECT * FROM enrollment;
```
