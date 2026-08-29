# 10 — DBMS Fundamentals

## DBMS

A **Database Management System (DBMS)** is software that manages persistent data and provides mechanisms to:

- store data
- retrieve data
- modify data
- enforce rules/constraints
- handle concurrent access
- recover from failures
- manage transactions

For a backend application:

```text
Spring Boot application
        ↓
       JDBC
        ↓
      DBMS
        ↓
   PostgreSQL
        ↓
      Disk
```

PostgreSQL is an **RDBMS** — a relational DBMS.

---

## DBMS vs RDBMS

**DBMS** is the general term for software that manages databases.

An **RDBMS** is a DBMS based on the relational model. Data is represented primarily as relations, which we normally visualize as tables.

Example:

```text
CUSTOMER

id | name  | email
---|-------|----------------
1  | Tanay | t@x.com
2  | Rahul | r@x.com
```

RDBMSs provide concepts such as:

- tables/relations
- primary keys
- foreign keys
- constraints
- SQL
- transactions
- relationships between data

Examples include PostgreSQL, MySQL, Oracle, and SQL Server.

---

## Tables, Rows, and Columns

At the conceptual level:

```text
USER
--------------------------------
id | name | email | created_at
--------------------------------
1  | A    | a@x  | ...
2  | B    | b@x  | ...
```

- **Table / relation** → collection of related records
- **Row / tuple** → one record
- **Column / attribute** → one property of a record

For example:

```text
id = 42
name = "Tanay"
email = "tanay@example.com"
```

is one row.

---

## Primary Key

A **primary key (PK)** uniquely identifies each row in a table.

```text
USER

id (PK) | name
--------|------
101     | Tanay
102     | Rahul
```

A primary key is unique and cannot be `NULL`. A table has one primary-key constraint.

Example:

```sql
CREATE TABLE users (
    id BIGINT PRIMARY KEY,
    name TEXT
);
```

---

## Candidate Key

A **candidate key** is a minimal set of attributes that can uniquely identify a row.

Suppose:

```text
USER

id | email
---|----------------
1  | tanay@example.com
2  | rahul@example.com
```

Both `id` and `email` could uniquely identify a user, so both are candidate keys.

One candidate key is selected as the primary key:

```text
Candidate keys:
    id
    email

Chosen primary key:
    id
```

The other candidate key can be enforced with a `UNIQUE` constraint.

---

## Foreign Key

A **foreign key (FK)** represents a reference to a key in another table.

Example:

```text
USERS

id | name
---|------
1  | Tanay
2  | Rahul
```

```text
ORDERS

id | user_id | amount
---|---------|-------
10 | 1       | 5000
11 | 1       | 2000
12 | 2       | 1000
```

`ORDERS.user_id` references `USERS.id`:

```text
USERS
 id (PK)
   ↑
   │ referenced by
   │
ORDERS
 user_id (FK)
```

The database can therefore enforce that an order's `user_id` refers to an existing user.

An invalid reference such as:

```sql
INSERT INTO orders(user_id, amount)
VALUES (999, 5000);
```

should fail when user `999` does not exist and the foreign-key constraint is enforced.

This is **referential integrity**.

---

## Why Database Constraints Matter

Application code can validate data, but the database should also enforce its own integrity rules.

There may be multiple application instances or other clients accessing the same database:

```text
App Instance 1 ──┐
                 ├──→ PostgreSQL
App Instance 2 ──┘
```

The database is the final authority over the validity of its stored data.

Therefore:

```text
Application validation
        +
Database constraints
        ↓
stronger data integrity
```

Constraints are covered more systematically later in this section.

---

## Relationships

Data entities can have different relationship cardinalities.

### One-to-one

```text
USER 1 ───────── 1 PROFILE
```

### One-to-many

```text
CUSTOMER 1 ──────── * ORDER
```

For example:

```text
Customer 1
   │
   ├── Order 101
   ├── Order 102
   └── Order 103
```

### Many-to-many

```text
STUDENT * ─────── * COURSE
```

A relational database normally represents this using a join table:

```text
STUDENT
   │
   ↓
STUDENT_COURSE
   ↑
   │
COURSE
```

Example:

```text
student_course

student_id | course_id
-----------|----------
1          | 101
1          | 102
2          | 101
```

The detailed modeling of these relationships and ER diagrams is the next topic.

---

## Database vs Schema

For PostgreSQL, a useful hierarchy is:

```text
PostgreSQL server
   │
   ├── Database A
   │      ├── schema A
   │      │     ├── users
   │      │     └── orders
   │      │
   │      └── schema B
   │            └── audit_logs
   │
   └── Database B
          └── ...
```

A **database** is a higher-level data container/boundary.

A **schema** is a logical namespace inside a database used to organize database objects such as tables and views.

For example:

```sql
SELECT * FROM app.users;
```

Here:

```text
app   = schema
users = table
```

### Why multiple schemas?

Schemas can provide:

- logical organization
- namespace separation
- object/permission boundaries
- separation of concerns

A typical application may simply use one database and one application schema with many tables.

### Why multiple databases?

Multiple databases provide a stronger separation boundary and can be useful when applications/workloads need different:

- access controls
- operational policies
- backup/lifecycle management
- workload isolation
- ownership boundaries

Multiple databases may also be appropriate when different workloads use different database technologies.

Do not confuse logical schema separation with completely separate databases.

---

## Data Ownership

In larger architectures, particularly microservices, a useful principle is that a service should own its data rather than allowing other services to directly manipulate its tables.

For example:

```text
Payment Service  → Payment DB
Account Service  → Account DB
Fraud Service    → Fraud DB
```

The important architectural idea is **ownership and access boundaries**, not necessarily that every service must have a physically separate database server.

---

## Mental Model

The basic relational model is:

```text
                 Database
                    │
          ┌─────────┼─────────┐
          ↓         ↓         ↓
        USERS     ORDERS    PAYMENTS
          │         │          │
          │         │          │
         PK        PK         PK
          ↑         │
          │         │
          └───────── FK
```

Tables store data, keys identify records and connect related data, and constraints define which states are valid.

This foundation leads naturally into:

```text
Data model
    ↓
tables + relationships
    ↓
constraints
    ↓
transactions
    ↓
concurrency / locking
    ↓
indexes
    ↓
query performance
```

---

## Interview Quick Recall

> **DBMS:** software that manages persistent data and provides storage, retrieval, modification, integrity, concurrency, recovery, and transaction capabilities.

> **RDBMS:** a DBMS based on the relational model, using relations/tables and relational constraints.

> **Table/relation:** structured collection of related records.

> **Row/tuple:** one record.

> **Column/attribute:** one property of a record.

> **Primary key:** the chosen unique identifier for rows.

> **Candidate key:** a minimal set of attributes capable of uniquely identifying a row.

> **Foreign key:** a reference to a key in another table.

> **Referential integrity:** ensures foreign-key references point to valid rows.

> **Schema:** a logical namespace inside a database.

> **Database:** a higher-level container providing a stronger boundary for data and administration.

> **1:1 / 1:N / M:N:** relationship cardinalities between entities.

> **Data ownership:** in service-oriented architectures, the owning service should control its data rather than other services directly manipulating its tables.
