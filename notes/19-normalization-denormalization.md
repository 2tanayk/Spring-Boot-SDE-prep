# 19 — Normalization & Denormalization

Normalization is about organizing relational data so that information lives in the appropriate place, unnecessary duplication is reduced, and common data anomalies are avoided.

Denormalization is the deliberate introduction of some redundancy when a known read/access pattern justifies the trade-off.

## 1. Why normalization exists

Consider an orders table that repeats customer information:

```text
order_id | customer_id | customer | customer_city
101      | 1           | Tanay    | Mumbai
102      | 1           | Tanay    | Mumbai
103      | 2           | Rahul    | Delhi
```

The same customer information appears in multiple rows.

If customer 1 moves from Mumbai to Pune, every duplicated row must be updated. Missing one creates contradictory data.

A normalized model separates the entities:

```text
customers
----------------
id | name  | city
1  | Tanay | Mumbai
2  | Rahul | Delhi
```

```text
orders
-----------------
order_id | customer_id
101      | 1
102      | 1
103      | 2
```

Now the customer's city is stored in the customer record rather than copied into every order.

## 2. Data anomalies

Normalization helps avoid three classic anomalies.

### Update anomaly

The same fact exists in many rows. Updating only some copies creates inconsistent data.

```text
Tanay → Mumbai
```

appearing in 50 rows means a city change may require 50 updates.

### Insert anomaly

A fact cannot be inserted without some unrelated fact.

For example, if courses exist only inside a `student_courses` table, adding a new course before any student enrolls may be awkward or impossible under the schema.

Separating:

```text
students
courses
enrollments
```

allows each entity to exist independently.

### Delete anomaly

Deleting one relationship accidentally removes the only stored information about another entity.

For example, if the only record that the Java course exists is:

```text
Tanay | Java
```

then deleting Tanay's enrollment could accidentally remove the only representation of the course.

Separate `courses` and `enrollments` tables avoid this.

## 3. First Normal Form — 1NF

The practical idea is that a column should contain an atomic value rather than a list/repeating group.

Bad:

```text
customer_id | phone_numbers
1           | 9876, 1234, 5555
```

Better represented as separate rows/table structure:

```text
customer_id | phone
1           | 9876
1           | 1234
1           | 5555
```

Mental model:

```text
1NF → do not pack a collection/repeating group into one relational cell
```

## 4. Second Normal Form — 2NF

2NF matters particularly with **composite keys**.

A non-key attribute should depend on the **whole composite key**, not only part of it.

Example:

```text
order_id | product_id | product_name | quantity
```

with:

```text
PRIMARY KEY (order_id, product_id)
```

`quantity` depends on the combination `(order_id, product_id)` because it describes that product within that particular order.

But `product_name` depends only on `product_id`.

That is a partial dependency.

Normalize into:

```text
products
-----------------
product_id | name
```

and:

```text
order_items
-------------------------------
order_id | product_id | quantity
```

Mental model:

```text
2NF → with a composite key, avoid attributes that depend on only part of the key
```

## 5. Third Normal Form — 3NF

3NF removes transitive dependencies where a non-key attribute really belongs to another entity.

Example:

```text
employees
------------------------------------------------
employee_id | employee_name | dept_id | dept_name
```

The dependencies are:

```text
employee_id → dept_id
dept_id     → dept_name
```

So `dept_name` is indirectly dependent on `employee_id` through `dept_id` and really belongs to the department entity.

Normalize into:

```text
departments
-------------------
dept_id | dept_name
```

and:

```text
employees
--------------------------------
employee_id | employee_name | dept_id
```

Mental model:

```text
3NF → avoid storing attributes that really belong to another entity through an indirect dependency
```

## 6. Practical normal-form recall

For interview purposes:

```text
1NF
→ don't store lists/repeating groups in one column

2NF
→ with composite keys, don't store attributes dependent on only part of the key

3NF
→ don't store attributes that really belong to another entity through a transitive dependency
```

The overarching goal is to reduce unnecessary duplication and avoid update, insert, and delete anomalies.

## 7. Denormalization

Denormalization means **intentionally introducing some redundancy** to optimize a known access/read pattern or preserve useful historical/read-model data.

A normalized model might require joins across:

```text
orders
  JOIN customers
  JOIN order_items
  JOIN products
```

For a hot read path, a system may deliberately store some frequently needed information closer to the read model.

Example:

```text
orders
------------------------------------------------
order_id | customer_id | customer_name | total
```

`customer_name` may also exist in the customer table, so this introduces duplication intentionally.

## 8. Normalization vs Denormalization trade-off

```text
Normalization
→ less duplication
→ easier consistency
→ often more joins

Denormalization
→ intentional duplication
→ simpler/faster reads for some access patterns
→ more consistency/update complexity
```

| | Normalization | Denormalization |
|---|---|---|
| Duplication | Minimized | Intentionally introduced |
| Consistency | Easier | More complex |
| Reads | May require more joins | Can simplify hot read paths |
| Writes | Usually one source of truth | May require maintaining duplicated data |
| Storage | Generally lower | Generally higher |
| Main goal | Integrity/maintainability | Read/access-pattern optimization |

## 9. Historical data can justify duplication

Not all duplicated data is bad.

Suppose a product currently costs ₹1,200 but an old order was purchased when the price was ₹1,000.

The product table may contain:

```text
products.price = 1200
```

while the order item stores:

```text
order_items.price = 1000
```

The order should normally preserve the price actually paid.

This duplication represents **historical state**, not merely accidental redundancy.

## 10. Practical design approach

Do not denormalize automatically because "joins are slow." Relational databases can execute joins efficiently when queries and indexes are designed appropriately.

A useful approach is:

```text
start reasonably normalized
        ↓
measure real workload/bottlenecks
        ↓
identify expensive/hot read paths
        ↓
denormalize selectively when justified
        ↓
have a strategy for consistency of duplicated data
```

## Interview Quick Recall

> Normalization organizes data to reduce unnecessary duplication and prevent update/insert/delete anomalies.

> 1NF: avoid lists/repeating groups in a single column.

> 2NF: with composite keys, non-key attributes should depend on the whole key.

> 3NF: avoid transitive dependencies where an attribute really belongs to another entity.

> Denormalization intentionally introduces redundancy for a known access/read-performance reason or to preserve meaningful historical/read-model state.

> Start reasonably normalized and denormalize selectively when there is a demonstrated reason, with a clear consistency strategy.
