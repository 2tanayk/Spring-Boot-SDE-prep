# 14 — Database Constraints

A **database constraint** is a rule enforced by the database that determines what data is valid.

Constraints protect data integrity regardless of whether data comes from the Spring Boot application, another service, a batch job, a script, or an admin tool.

## 1. PRIMARY KEY

A primary key uniquely identifies a row.

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY
);
```

A primary key guarantees:

```text
UNIQUE + NOT NULL
```

A table has one primary key constraint, although the key can contain multiple columns.

### Composite primary key

When one column is not sufficient to identify a row:

```sql
CREATE TABLE enrollments (
    student_id BIGINT,
    course_id BIGINT,
    PRIMARY KEY (student_id, course_id)
);
```

The **combination** must be unique.

## 2. UNIQUE

`UNIQUE` prevents duplicate values (or duplicate combinations for a composite unique constraint).

```sql
CREATE TABLE customers (
    id BIGINT PRIMARY KEY,
    email VARCHAR(255) UNIQUE
);
```

Unlike a primary key, a table can have multiple unique constraints.

Important distinction:

```text
PRIMARY KEY → row identity; unique + non-null
UNIQUE      → uniqueness of a business value/combination
```

NULL behavior for unique constraints is database-specific. In PostgreSQL, an ordinary unique constraint allows multiple NULL values. If the value must both exist and be unique:

```sql
email VARCHAR(255) NOT NULL UNIQUE
```

## 3. FOREIGN KEY

A foreign key enforces a valid relationship between tables.

```sql
CREATE TABLE orders (
    id BIGINT PRIMARY KEY,
    customer_id BIGINT,
    FOREIGN KEY (customer_id)
        REFERENCES customers(id)
);
```

The database now prevents an order from referencing a customer that does not exist.

This is **referential integrity**.

Application code may check that a referenced row exists, but that check alone is not a database-level guarantee and can be affected by concurrent changes or other writers. A foreign key enforces the invariant at the persistence boundary.

## 4. Foreign-key delete behavior

When a referenced parent row is deleted, common options include:

### RESTRICT / NO ACTION

Reject the deletion while dependent rows still reference the parent.

### CASCADE

Delete dependent rows automatically:

```sql
FOREIGN KEY (customer_id)
REFERENCES customers(id)
ON DELETE CASCADE
```

Useful when the child has no meaningful independent existence. Be careful with important historical/financial data.

### SET NULL

Set the foreign key to NULL when the parent is deleted:

```sql
ON DELETE SET NULL
```

The foreign-key column must allow NULL.

Choose behavior based on the business relationship rather than using CASCADE automatically.

## 5. NOT NULL

Requires a value to be present:

```sql
email VARCHAR(255) NOT NULL
```

`NULL` is not the same as an empty string, zero, or another sentinel value. It represents a missing/unknown/not-applicable value depending on the model.

## 6. CHECK

A `CHECK` constraint requires a row to satisfy a condition.

```sql
age INT CHECK (age >= 18)
```

Another example:

```sql
status VARCHAR(20)
CHECK (status IN ('ACTIVE', 'BLOCKED', 'CLOSED'))
```

This is useful for enforcing simple data invariants directly in the database.

## 7. DEFAULT

A `DEFAULT` supplies a value when the column is omitted from an INSERT.

```sql
status VARCHAR(20) DEFAULT 'ACTIVE'
```

Important: a default is not generally a replacement for an explicitly supplied NULL.

```text
column omitted → default can be applied
column explicitly set to NULL → NULL is supplied
```

## 8. Constraints vs Application Validation

Application validation and database constraints solve different problems.

### Application validation

Useful for:

- friendly API errors
- request-level validation
- rejecting bad input early
- rules requiring application context

### Database constraints

Useful for:

- protecting persisted data
- guaranteeing integrity regardless of caller
- preventing race-condition-related integrity violations
- enforcing relational integrity

For invariants the database can enforce, do not rely solely on application checks.

### Example: uniqueness race

Two requests can concurrently perform:

```text
Request A                  Request B
    ↓                          ↓
check email                  check email
    ↓                          ↓
not found                    not found
    ↓                          ↓
insert                       insert
```

An application-only existence check does not provide the final uniqueness guarantee. A database `UNIQUE` constraint does: one insert succeeds and the conflicting insert is rejected.

## 9. Constraints vs Indexes

Constraints and indexes are related but serve different purposes:

```text
Constraint → correctness / data integrity
Index      → performance / access path
```

Primary-key and unique constraints are commonly backed by unique indexes or equivalent database structures. A foreign-key constraint is a referential-integrity rule; an index on the foreign-key column is a separate performance consideration.

## Interview Quick Recall

> PRIMARY KEY = row identity; unique and non-null.

> UNIQUE = prevents duplicate values/combinations; NULL behavior is database-specific.

> FOREIGN KEY = enforces referential integrity between tables.

> NOT NULL = value must be present.

> CHECK = row must satisfy a condition.

> DEFAULT = value used when a column is omitted from INSERT.

> Application validation is useful, but database constraints are the final integrity boundary for persisted data.

> Indexes and constraints are not the same thing: indexes primarily optimize access; constraints primarily enforce correctness.
