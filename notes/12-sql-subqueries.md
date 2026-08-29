# 12 — SQL Subqueries

## What is a Subquery?

A **subquery** is a query nested inside another query.

```text
Outer query
    ↓
uses result of
    ↓
Inner query
```

---

## 1. Subquery Returning One Value

Use a scalar subquery when the inner query produces one value.

Example: find orders whose amount is greater than the average order amount.

```sql
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount)
    FROM orders
);
```

The inner query produces one value, such as `3200`, which the outer query compares against each order.

---

## 2. Subquery Returning Multiple Values

Use `IN` when the inner query produces a set of values.

Example: find customers who have placed an order.

```sql
SELECT *
FROM customers
WHERE id IN (
    SELECT customer_id
    FROM orders
);
```

The inner query might produce:

```text
1
1
2
3
```

The outer query keeps customers whose ID appears in that result set.

---

## 3. EXISTS

`EXISTS` checks whether the subquery produces at least one matching row.

Example:

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

This means:

> Return customers for whom at least one order exists.

The inner query is **correlated** because it refers to `c.id` from the outer query.

`EXISTS` cares about whether a matching row exists, not about retrieving the actual values from those rows.

---

## 4. Correlated vs Non-Correlated Subqueries

### Non-correlated

The inner query is independent of the outer query:

```sql
SELECT *
FROM orders
WHERE amount > (
    SELECT AVG(amount)
    FROM orders
);
```

The inner query can be evaluated independently.

### Correlated

The inner query references a value from the outer query:

```sql
SELECT *
FROM customers c
WHERE EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.customer_id = c.id
);
```

Conceptually:

```text
Customer 1
    ↓
Does an order exist for customer 1?
    ↓
YES → keep customer
```

The same logic is applied for each outer row.

---

## 5. Subquery in FROM

A subquery can also produce an intermediate result that the outer query treats like a table.

Example:

```sql
SELECT *
FROM (
    SELECT customer_id, SUM(amount) AS total
    FROM orders
    GROUP BY customer_id
) customer_totals
WHERE total > 10000;
```

Conceptually:

```text
orders
  ↓
GROUP BY customer
  ↓
customer_totals
  ↓
WHERE total > 10000
```

The subquery in `FROM` is commonly called a **derived table**.

---

## Subquery vs JOIN

There is no universal rule that one is always better.

For example, these can express the same business requirement:

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

The `JOIN` can produce multiple rows for a customer with multiple orders, which is why `DISTINCT` may be needed in this example. `EXISTS` directly expresses the requirement that a related row exists.

Do not assume that subqueries are inherently slower than joins. Modern optimizers can transform many logically equivalent queries. Actual performance depends on the database engine, query, indexes, data distribution, and execution plan.

---

## Core Patterns

### Single value

```sql
WHERE x > (
    SELECT ...
)
```

The inner query returns one value.

### Multiple values

```sql
WHERE x IN (
    SELECT ...
)
```

The inner query returns a set of values.

### Existence

```sql
WHERE EXISTS (
    SELECT 1
    FROM ...
    WHERE ...
)
```

Use when you care whether at least one related row exists.

### Derived result

```sql
FROM (
    SELECT ...
) alias
```

Use when an intermediate query result needs to be treated like a table.

---

## Interview Quick Recall

> **Subquery:** a query nested inside another query.

> **Scalar subquery:** inner query returns one value.

> **`IN`:** tests membership in the set returned by a subquery.

> **`EXISTS`:** checks whether at least one matching row exists.

> **Correlated subquery:** inner query references a value from the outer query.

> **Non-correlated subquery:** inner query is independent of the outer query.

> A subquery in `FROM` can act as a derived table.

> Don't claim that joins are always faster than subqueries; use the actual query plan when performance matters.
