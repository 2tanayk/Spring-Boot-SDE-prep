# 13 — Indexes

Indexes are additional data structures maintained alongside a table to make common lookups faster without scanning the entire table.

## 1. Why indexes exist

Without a suitable index, a query such as:

```sql
SELECT *
FROM transactions
WHERE customer_id = 123;
```

may require a full/table scan:

```text
row 1 → check
row 2 → check
...
row N → check
```

An index provides another lookup path into the table so the database can potentially locate matching rows much more efficiently.

Think of indexes as different access paths to the same underlying table.

```text
PK index
id → row

customer_id index
customer_id → matching rows

account_id index
account_id → matching rows
```

## 2. Primary key vs other indexes

A primary key is normally backed by a unique index, which makes lookups such as:

```sql
SELECT *
FROM transactions
WHERE id = 8472;
```

efficient.

But the primary-key index is ordered around the primary-key column. It does not efficiently answer a query such as:

```sql
SELECT *
FROM transactions
WHERE customer_id = 123;
```

The database does not know the transaction IDs in advance, so it needs an access path for `customer_id`.

A useful mental model:

> Ask what information the query has that identifies or narrows the desired rows, then design indexes around those access patterns.

## 3. B-tree basics

B-tree-style indexes keep indexed values ordered and allow the database to navigate to relevant values rather than examining every table row.

This makes them useful for:

- Equality lookups
- Range predicates
- Ordering when the index order is suitable

Examples:

```sql
WHERE customer_id = 101
```

```sql
WHERE amount > 5000
```

```sql
WHERE amount BETWEEN 1000 AND 5000
```

```sql
ORDER BY amount
```

You do not need to memorize the internal node layout for an SDE-2 interview. The key point is that the index is ordered, allowing efficient navigation and range access.

## 4. Indexes are separate maintained structures

The index is additional to the table data:

```text
Table
─────
actual rows

Index
─────
ordered lookup structure
        ↓
references corresponding rows
```

Therefore indexes consume additional storage and must be maintained when indexed data changes.

## 5. When indexes help

Indexes are especially useful when a query is selective — the predicate eliminates a large portion of the table.

For example, with 10 million transactions and only 500 belonging to a particular customer:

```text
10,000,000 rows
       ↓
customer_id index
       ↓
~500 relevant rows
```

This can be substantially cheaper than scanning all 10 million rows.

## 6. When indexes can hurt

### Write overhead

`INSERT`, `UPDATE`, and `DELETE` operations may need to maintain relevant indexes.

```text
More indexes
    ↓
more index maintenance
    ↓
potentially more expensive writes
```

### Storage

Indexes consume disk space and may also consume memory/cache resources.

### Low selectivity

An index is not automatically useful just because it exists.

For a column such as `status` where 99% of rows are `SUCCESS`, an index used for:

```sql
WHERE status = 'SUCCESS'
```

may not be worthwhile because almost the whole table qualifies. The optimizer may prefer a table scan.

The exact decision depends on the database, query, statistics, indexes, data distribution, and execution plan.

## 7. Composite indexes

A composite index covers multiple columns:

```sql
CREATE INDEX idx_customer_status
ON transactions(customer_id, status);
```

This is useful for query patterns such as:

```sql
WHERE customer_id = 123
```

and especially:

```sql
WHERE customer_id = 123
  AND status = 'SUCCESS'
```

## 8. Column order matters

For:

```text
(customer_id, status, created_at)
```

the useful prefixes are approximately:

```text
(customer_id)
(customer_id, status)
(customer_id, status, created_at)
```

This is the common **leftmost-prefix** mental model.

The same composite index should not be assumed to be equally useful for predicates that start with only `status` or `created_at`.

Exact optimizer behavior varies by database, but the ordering of columns in a composite index is a fundamental design consideration.

## 9. Indexes should follow query patterns

Do not ask only:

> Which columns should I index?

Ask:

> What queries does the application actually run, and what access patterns do those queries require?

For example, suppose the API frequently executes:

```sql
SELECT *
FROM transactions
WHERE customer_id = ?
ORDER BY created_at DESC
LIMIT 20;
```

A potentially useful index is:

```sql
CREATE INDEX idx_customer_created
ON transactions(customer_id, created_at DESC);
```

This aligns the index with both the customer lookup and the desired ordering.

## 10. The optimizer chooses whether to use an index

Having an index does not guarantee that the database will use it.

The optimizer estimates the cost of available plans and may choose a table scan when that is cheaper.

For example, an index on a very low-selectivity `status` column may be ignored when most rows match.

Use the database's query-plan tooling to investigate actual behavior. In PostgreSQL:

```sql
EXPLAIN ANALYZE
SELECT *
FROM transactions
WHERE customer_id = 123;
```

Look for operations such as an index scan versus a sequential/table scan, along with the estimated and actual costs/rows.

## Interview Quick Recall

> An index is an additional data structure that provides an alternative access path to table rows.

> A primary key is normally indexed, but that index is useful primarily for access through the primary-key value. It does not replace indexes for other query patterns.

> B-tree indexes are ordered and are useful for equality, range, and suitable ordering operations.

> Composite-index column order matters; use the leftmost-prefix mental model.

> Indexes can improve reads but add storage and write/maintenance overhead.

> Low-selectivity predicates may not benefit from an index.

> The existence of an index does not guarantee that the optimizer will use it.

> Design indexes around real application query patterns and verify performance with the query plan rather than assuming an index is beneficial.
