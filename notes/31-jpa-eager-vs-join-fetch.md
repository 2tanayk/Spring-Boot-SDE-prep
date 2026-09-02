# JPA/Hibernate — EAGER vs JOIN FETCH

## Core distinction

`FetchType.EAGER` does **not** mean Hibernate must use a SQL `JOIN`.

- **EAGER** = the association must be loaded when the entity is loaded.
- **JOIN FETCH** = explicitly fetch the association using a SQL join in the query.

```java
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

EAGER may result in a join, but Hibernate can also load the association using separate SQL queries depending on the situation.

## Example

EAGER:

```sql
SELECT o.*, c.*
FROM orders o
LEFT JOIN customer c ON o.customer_id = c.id;
```

A join is possible, but **not guaranteed just because the mapping is EAGER**.

Explicit fetch join:

```java
@Query("""
    SELECT o
    FROM Order o
    JOIN FETCH o.customer
""")
List<Order> findAllWithCustomer();
```

Here the query explicitly asks Hibernate to fetch the customer through a join.

## Interview distinction

| Approach | Guarantees JOIN? | Association loaded? |
|---|---:|---:|
| `EAGER` | ❌ No | ✅ Yes |
| `JOIN FETCH` | ✅ Yes | ✅ Yes |
| `@EntityGraph` | Controls the fetch plan | ✅ Yes |
| `LAZY` | ❌ No | ❌ Not initially |

## Why this matters for N+1

Do **not** make an association EAGER simply to fix N+1.

EAGER is a mapping-level loading requirement, not a guarantee about the SQL strategy. For a specific use case, prefer an explicit fetch plan such as:

- `JOIN FETCH`
- `@EntityGraph`
- batch fetching
- DTO projection

### Mental model

> **EAGER answers: “When should it be loaded?”**
>
> **JOIN FETCH answers: “Fetch it as part of this query using a join.”**
