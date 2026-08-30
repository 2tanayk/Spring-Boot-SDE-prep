# 15 — SQL NULLs & PostgreSQL

`NULL` represents a missing, unknown, or not-applicable value. It is not the same as `0`, an empty string, `false`, or the literal string `'NULL'`.

## 1. NULL is not equal to anything

Do not use:

```sql
WHERE email = NULL
```

Use:

```sql
WHERE email IS NULL
```

and:

```sql
WHERE email IS NOT NULL
```

Normal comparisons involving NULL produce `UNKNOWN` rather than TRUE or FALSE:

```text
NULL = 5       → UNKNOWN
NULL = NULL    → UNKNOWN
NULL > 5       → UNKNOWN
NULL <> NULL   → UNKNOWN
```

## 2. SQL uses three-valued logic

SQL conditions can evaluate to:

```text
TRUE
FALSE
UNKNOWN
```

`WHERE` keeps only rows for which the condition is TRUE. Both FALSE and UNKNOWN are filtered out.

Example:

```text
age = 25   → TRUE     → keep
age = 15   → FALSE    → discard
age = NULL → UNKNOWN  → discard
```

## 3. AND / OR with NULL

NULL introduces three-valued logic into boolean expressions.

Useful cases:

```text
TRUE  AND UNKNOWN → UNKNOWN
FALSE AND UNKNOWN → FALSE
TRUE  OR  UNKNOWN → TRUE
FALSE OR  UNKNOWN → UNKNOWN
```

You do not need to memorize the entire truth table. Remember that UNKNOWN propagates unless the other operand already determines the result.

## 4. NOT NULL

Use a `NOT NULL` constraint when a column must contain a value:

```sql
email VARCHAR(255) NOT NULL
```

This is a database-level data-integrity rule.

## 5. COALESCE

`COALESCE(value, fallback)` returns the first non-NULL value.

```sql
SELECT COALESCE(phone, 'No phone')
FROM customers;
```

It can be chained:

```sql
COALESCE(phone, email, 'No contact')
```

Meaning:

```text
phone?
 ↓ NULL
email?
 ↓ NULL
'No contact'
```

## 6. NULL and aggregates

Most aggregate functions ignore NULL values rather than treating NULL as zero.

For:

```text
100
200
NULL
300
```

`AVG(amount)` averages the three non-NULL values.

`COUNT(*)` counts rows, while `COUNT(column)` counts only non-NULL values:

```text
COUNT(*)       → 4
COUNT(amount)  → 3
```

## 7. NULL and UNIQUE in PostgreSQL

In PostgreSQL, an ordinary unique constraint allows multiple NULL values.

For example, this is valid:

```text
id | email
---|------
1  | a@example.com
2  | NULL
3  | NULL
```

If the value must both exist and be unique:

```sql
email VARCHAR(255) NOT NULL UNIQUE
```

## 8. NULL and LEFT JOIN

A `LEFT JOIN` can produce NULLs for right-side columns when there is no matching row:

```text
customer | order
---------|------
Amit     | NULL
```

To find customers with no matching order:

```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o
    ON c.id = o.customer_id
WHERE o.id IS NULL;
```

Do not write `o.id = NULL`; use `IS NULL`.

## 9. NULLIF

`NULLIF(a, b)` returns NULL when `a = b`; otherwise it returns `a`.

Example:

```sql
NULLIF(amount, 0)
```

This can be useful for avoiding division-by-zero:

```sql
SELECT total / NULLIF(count, 0)
FROM ...;
```

If `count` is zero, the denominator becomes NULL instead of zero.

## Interview Quick Recall

> NULL means missing/unknown/not-applicable; it is not zero or an empty string.

> Use `IS NULL` / `IS NOT NULL`, not `= NULL` / `<> NULL`.

> SQL has three-valued logic: TRUE, FALSE, UNKNOWN.

> `WHERE` keeps only TRUE; UNKNOWN is filtered out.

> `COALESCE` returns the first non-NULL value.

> `COUNT(*)` counts rows; `COUNT(column)` ignores NULLs.

> Aggregates such as SUM/AVG generally ignore NULL values.

> PostgreSQL's ordinary UNIQUE constraint normally allows multiple NULLs; combine `NOT NULL` with `UNIQUE` when the value must exist and be unique.
