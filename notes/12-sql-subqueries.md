# 12 — SQL

Interview-focused SQL revision covering the practical concepts most relevant to backend development.

## 1. SELECT

Basic form:

```sql
SELECT column1, column2
FROM table
WHERE condition;
```

SQL is declarative: we describe the result we want rather than giving the database a procedural sequence of operations.

## 2. WHERE

`WHERE` filters individual rows.

Common operators:

```text
=  <>  >  <  >=  <=
AND  OR  NOT
BETWEEN
IN
LIKE
```

Examples:

```sql
SELECT * FROM orders WHERE amount > 3000;
SELECT * FROM customers WHERE city IN ('Mumbai', 'Delhi');
SELECT * FROM customers WHERE name LIKE 'Ta%';
```

## 3. ORDER BY and LIMIT

```sql
SELECT *
FROM orders
ORDER BY amount DESC
LIMIT 10;
```

`ASC` = ascending; `DESC` = descending.

If the order matters, always specify `ORDER BY`. `LIMIT` without an explicit ordering does not define a meaningful deterministic "first N" set.

## 4. Logical SQL Execution Order

Although SQL is written approximately as:

```text
SELECT
FROM
WHERE
GROUP BY
HAVING
ORDER BY
LIMIT
```

the conceptual processing order is roughly:

```text
FROM / JOIN
    ↓
WHERE
    ↓
GROUP BY
    ↓
HAVING
    ↓
SELECT
    ↓
ORDER BY
    ↓
LIMIT
```

This explains why a `SELECT` alias generally cannot be referenced in `WHERE`: `WHERE` is logically evaluated before `SELECT`.

## 5. GROUP BY and Aggregation

`GROUP BY` divides rows into groups based on one or more columns.

Common aggregate functions:

```text
COUNT()
SUM()
AVG()
MIN()
MAX()
```

Example:

```sql
SELECT customer_id, SUM(amount) AS total
FROM orders
GROUP BY customer_id;
```

Without `GROUP BY`, an aggregate normally operates over the whole input. With `GROUP BY`, it is calculated once per group.

Multiple grouping columns:

```sql
SELECT customer_id, status, SUM(amount) AS total
FROM orders
GROUP BY customer_id, status;
```

A selected column should either participate in the grouping or be reduced by an aggregate. Selecting an ungrouped, non-aggregated column is ambiguous because a group can contain multiple values for it.

### COUNT(*) vs COUNT(column)

```text
COUNT(*)       → counts rows
COUNT(column)  → counts non-NULL values
```

## 6. WHERE vs HAVING

```text
WHERE  → filters individual rows before grouping
HAVING → filters groups after grouping
```

Example:

```sql
SELECT customer_id, SUM(amount) AS total
FROM orders
WHERE status = 'SUCCESS'
GROUP BY customer_id
HAVING SUM(amount) > 5000;
```

Do not put aggregate conditions such as `SUM(amount) > 5000` in `WHERE`; use `HAVING`.

## 7. JOINs

Joins combine data from multiple tables using a matching condition.

Example:

```sql
SELECT o.id, o.amount, c.name
FROM orders o
JOIN customers c
    ON o.customer_id = c.id;
```

### INNER JOIN

`JOIN` normally means `INNER JOIN`.

It returns only rows where the condition matches on both sides.

```sql
SELECT c.name, o.amount
FROM customers c
INNER JOIN orders o
    ON c.id = o.customer_id;
```

Customers without orders are excluded.

### LEFT JOIN

Keeps every row from the left table. If no matching right-side row exists, right-side columns are `NULL`.

```sql
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o
    ON c.id = o.customer_id;
```

Use it when records with no match must still appear.

### SELF JOIN

Joins a table to itself using aliases.

```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
JOIN employees m
    ON e.manager_id = m.id;
```

## 8. ON vs WHERE with LEFT JOIN

Important trap:

```sql
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o
    ON c.id = o.customer_id
WHERE o.amount > 3000;
```

Unmatched customers have `o.amount = NULL`, so the `WHERE` condition removes them. This can effectively defeat the preservation intended by the `LEFT JOIN`.

If the requirement is "keep all customers, but only treat orders above 3000 as matches":

```sql
SELECT c.name, o.amount
FROM customers c
LEFT JOIN orders o
    ON c.id = o.customer_id
   AND o.amount > 3000;
```

Mental model:

```text
ON
→ determines what counts as a match

WHERE
→ filters the resulting rows
```

## 9. Joining Multiple Tables

Real backend queries often traverse several relationships:

```text
Customer → Account → Transaction
```

```sql
SELECT c.name, a.account_number, t.amount
FROM customers c
JOIN accounts a
    ON c.id = a.customer_id
JOIN transactions t
    ON a.id = t.account_id;
```

## 10. JOINs Can Multiply Rows

A JOIN does not necessarily preserve the row count of either input table.

If one customer has three orders, a customer-to-orders join naturally produces three result rows for that customer.

With multiple 1:N relationships, row multiplication can become significant. This is not automatically a bug; it depends on what the query should return.

If you need one row per customer, aggregation or sometimes `DISTINCT` may be appropriate. Do not blindly add `DISTINCT` without understanding the duplication.

## 11. DISTINCT

`DISTINCT` removes duplicate result rows.

```sql
SELECT DISTINCT customer_id
FROM orders;
```

If the source contains `1, 1, 2, 1`, the result is `1, 2`.

`DISTINCT` applies to the complete selected row:

```sql
SELECT DISTINCT customer_id, status
FROM orders;
```

Uniqueness is based on the combination `(customer_id, status)`, not only `customer_id`.

`DISTINCT` and `GROUP BY` can sometimes produce the same output:

```sql
SELECT DISTINCT customer_id FROM orders;
```

vs.

```sql
SELECT customer_id FROM orders GROUP BY customer_id;
```

But the intent differs:

```text
DISTINCT → unique combinations of selected values
GROUP BY → create groups, usually for aggregation
```

## 12. Pagination

### Offset pagination

```sql
SELECT *
FROM transactions
ORDER BY id
LIMIT 20 OFFSET 40;
```

Means roughly: skip 40 rows and return the next 20.

Useful when simple page-number navigation or arbitrary page jumps matter.

Problems:

- Deep offsets can become increasingly expensive because many preceding rows may need to be processed/skipped. Exact behavior depends on the database, query, indexes, and execution plan.
- Because it is position-based, inserts/deletes between page requests can cause duplicates or skipped records.

Example:

```text
Page 1:
1 2 3 4 5

new row inserted before next request

Page 2 with OFFSET 5:
5 6 7 8 9
```

`5` is duplicated.

Pagination should use a deterministic `ORDER BY`.

### Cursor / keyset pagination

Instead of "skip N rows", ask for rows after a known position in the ordering:

```sql
SELECT *
FROM transactions
WHERE id > :lastSeenId
ORDER BY id
LIMIT 20;
```

If page 1 ends at ID 500, the next request uses `lastSeenId = 500`.

This is generally better for large, changing datasets and sequential traversal such as feeds and transaction histories.

Cursor pagination requires deterministic ordering. If ordering by a non-unique value such as `created_at`, use a unique tie-breaker:

```sql
ORDER BY created_at DESC, id DESC
```

The cursor must contain enough information to identify a unique position in that ordering.

### Offset vs cursor

```text
OFFSET
→ simple
→ supports arbitrary page navigation
→ can suffer from large-offset cost
→ position can shift when data changes

CURSOR / KEYSET
→ better for large/changing datasets
→ efficient sequential traversal
→ harder to jump to arbitrary page numbers
→ requires deterministic ordering
```

Spring Data can make offset pagination convenient through `Pageable`, but it does not eliminate these database-level trade-offs.

## 13. Subqueries

A subquery is a query nested inside another query.

### Scalar / single-value subquery

```sql
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount)
    FROM orders
);
```

The inner query returns one value.

### Multiple-value subquery with IN

```sql
SELECT *
FROM customers
WHERE id IN (
    SELECT customer_id
    FROM orders
);
```

The inner query returns a set of values.

### EXISTS

Use `EXISTS` when the requirement is whether at least one related row exists:

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

The inner query is correlated because it references `c.id` from the outer query.

### Correlated vs non-correlated

Non-correlated:

```sql
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount)
    FROM orders
);
```

The inner query is independent.

Correlated:

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

The inner query depends on the current outer row.

### Subquery in FROM

A subquery can produce an intermediate result that the outer query treats like a table:

```sql
SELECT *
FROM (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
) customer_totals
WHERE total > 10000;
```

This is commonly called a **derived table**.

## Subquery vs JOIN

Do not memorize a blanket rule such as "JOINs are always faster".

For example, both can express "customers who have at least one order":

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

and:

```sql
SELECT DISTINCT c.*
FROM customers c
JOIN orders o
    ON c.id = o.customer_id;
```

`EXISTS` directly expresses the existence requirement and avoids the duplicate-row issue from the JOIN result.

Actual performance depends on the database engine, query, indexes, data distribution, and execution plan.

## Core SQL Mental Model

```text
FROM / JOIN
    ↓
WHERE              ← filter rows
    ↓
GROUP BY           ← create groups
    ↓
HAVING             ← filter groups
    ↓
SELECT             ← produce selected expressions
    ↓
ORDER BY           ← sort
    ↓
LIMIT              ← restrict result size
```

## Interview Quick Recall

> `WHERE` filters rows; `HAVING` filters groups.

> `GROUP BY` changes the unit of the result from individual rows to groups.

> `COUNT(*)` counts rows; `COUNT(column)` ignores NULL values.

> `INNER JOIN` keeps matches; `LEFT JOIN` preserves the left side.

> A 1:N JOIN can legitimately multiply rows.

> `DISTINCT` removes duplicate result rows based on the complete selected row.

> Pagination needs deterministic ordering. Offset pagination is simple but has large-offset and changing-data problems; cursor/keyset pagination is often better for large, changing datasets.

> A cursor represents a position in an ordering, not simply a page number.

> A subquery is a query nested inside another query; common patterns are scalar subqueries, `IN`, `EXISTS`, and derived tables.

> Do not make blanket performance claims about JOINs vs subqueries; inspect the actual query plan when performance matters.
