# JPA/Hibernate — Entity Relationships, Fetching & Cascades

## 1. Relationship Types

### `@ManyToOne`

The most common relationship. Many child rows reference one parent.

```java
@Entity
class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}
```

The foreign key lives on the many side (`orders.customer_id`), so this is normally the owning side.

### `@OneToMany`

The reverse view of a many-to-one relationship:

```java
@Entity
class Customer {
    @OneToMany(mappedBy = "customer")
    private List<Order> orders = new ArrayList<>();
}
```

`mappedBy = "customer"` refers to the Java field in `Order`, not the database column. It means this side is inverse/non-owning and `Order.customer` manages the relationship.

### Bidirectional vs unidirectional

You do not have to map both directions.

- Unidirectional: `Order -> Customer` only.
- Bidirectional: `Order -> Customer` and `Customer -> Orders`.

Add the inverse side only when the application actually needs navigation in that direction. More mappings mean more complexity.

If a bidirectional relationship is present, keep both Java sides synchronized:

```java
public void addOrder(Order order) {
    orders.add(order);
    order.setCustomer(this);
}

public void removeOrder(Order order) {
    orders.remove(order);
    order.setCustomer(null);
}
```

The owning side controls the database FK; both sides should remain consistent in the Java object graph.

### `@OneToOne`

One entity references exactly one other entity. The side containing the FK is normally the owning side.

```java
@OneToOne
@JoinColumn(name = "profile_id")
private Profile profile;
```

### `@ManyToMany`

Many-to-many relationships are represented using a join table.

```java
@ManyToMany
@JoinTable(
    name = "student_course",
    joinColumns = @JoinColumn(name = "student_id"),
    inverseJoinColumns = @JoinColumn(name = "course_id")
)
private Set<Course> courses;
```

If the relationship itself has attributes such as `grade`, `status`, or `enrolledAt`, model the join table as its own entity instead of using `@ManyToMany`.

---

## 2. Owning Side vs Inverse Side

The owning side is normally the side that contains/manages the foreign key relationship.

Example:

```java
class Order {
    @ManyToOne
    @JoinColumn(name = "customer_id")
    Customer customer;
}

class Customer {
    @OneToMany(mappedBy = "customer")
    List<Order> orders;
}
```

`Order.customer` is the owning side.

`Customer.orders` is the inverse side.

`mappedBy` tells Hibernate that both fields represent the same relationship; it does not create another FK.

Adding only to the inverse collection:

```java
customer.getOrders().add(order);
```

does not establish the FK. The owning side should also be updated:

```java
order.setCustomer(customer);
```

---

## 3. FetchType — LAZY vs EAGER

### LAZY

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

The association is not necessarily loaded when the parent is loaded. Hibernate can load it when it is accessed.

Advantages:

- Avoids loading unnecessary object graphs.
- Usually a better default for associations.
- Gives the application control over what data a use case needs.

### EAGER

```java
@ManyToOne(fetch = FetchType.EAGER)
private Customer customer;
```

The association is expected to be loaded along with the entity.

Important: **EAGER does not mean Hibernate will always use one JOIN query.** It is different from explicit `JOIN FETCH`.

Generally avoid making relationships EAGER just to solve query problems. Explicitly fetch the associations needed for a particular use case.

---

## 4. N+1 Query Problem

Example:

```java
List<Order> orders = orderRepository.findAll();

for (Order order : orders) {
    System.out.println(order.getCustomer().getName());
}
```

This can cause:

```text
1 query → fetch Orders
N queries → fetch each Customer
```

For 100 orders, potentially 101 queries.

Common solutions include explicit fetching such as:

```java
@Query("""
    select o
    from Order o
    join fetch o.customer
""")
List<Order> findAllWithCustomer();
```

or `@EntityGraph`:

```java
@EntityGraph(attributePaths = "customer")
List<Order> findAll();
```

Do not solve N+1 by blindly making everything EAGER.

---

## 5. Cascade

Cascade controls whether JPA/Hibernate propagates persistence operations from one entity to its related entities.

It is separate from the relationship itself and separate from FetchType.

```text
Relationship → @OneToMany / @ManyToOne / etc.
Fetching     → LAZY / EAGER
Cascade      → persistence-operation propagation
```

### `CascadeType.PERSIST`

Persisting a new parent also persists the child.

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.PERSIST)
private List<OrderItem> items;
```

Useful when the child is created as part of creating the parent.

### `CascadeType.MERGE`

Merging a parent propagates the merge to the children.

This matters particularly for detached entity graphs. In ordinary transactional code, managed entities are normally updated through dirty checking, so explicit merge is often not needed.

### `CascadeType.REMOVE`

Deleting the parent propagates removal to the children.

```java
@OneToMany(mappedBy = "order", cascade = CascadeType.REMOVE)
private List<OrderItem> items;
```

Use when the child lifecycle is tightly coupled to the parent.

### `CascadeType.REFRESH`

Refreshing the parent also refreshes the children.

```java
entityManager.refresh(order);
```

Less commonly used in typical Spring Boot applications.

### `CascadeType.DETACH`

Detaching the parent also detaches the children.

```java
entityManager.detach(order);
```

Less commonly used in typical Spring Boot applications.

### `CascadeType.ALL`

Equivalent to:

```text
PERSIST
MERGE
REMOVE
REFRESH
DETACH
```

`ALL` does **not** include `orphanRemoval`.

---

## 6. Why `PERSIST` + `MERGE` Is Common

A common configuration is:

```java
cascade = {
    CascadeType.PERSIST,
    CascadeType.MERGE
}
```

This means:

> Propagate creation of a new entity graph and merging of a detached entity graph, but do not automatically cascade removal.

For example:

```text
New Order graph
    ↓
PERSIST
    ↓
OrderItems persisted

Detached Order graph
    ↓
MERGE
    ↓
OrderItems merged
```

This can be preferable to `CascadeType.ALL` when child deletion should not be propagated.

---

## 7. Cascade vs Database `ON DELETE`

JPA cascade and database FK cascading are different mechanisms.

### JPA cascade

```java
cascade = CascadeType.REMOVE
```

Hibernate propagates the remove operation to child entities.

### Database cascade

```sql
FOREIGN KEY (customer_id)
REFERENCES customers(id)
ON DELETE CASCADE
```

The database handles dependent rows when the parent row is deleted.

If no JPA remove cascade is defined, Hibernate does not automatically propagate the delete to children. The database FK rules still apply when the DELETE reaches the database.

Typical DB behavior:

```text
NO ACTION / RESTRICT → parent deletion fails if referenced
ON DELETE CASCADE     → child rows are deleted
ON DELETE SET NULL    → FK is set to NULL (if allowed)
```

Think about **DB FK behavior first** when asked what happens if a referenced parent row is deleted, and then separately consider JPA cascade behavior when deletion is performed through Hibernate.

---

## 8. `orphanRemoval`

`orphanRemoval = true` means a child removed from the parent's relationship is treated as an orphan and deleted from the database.

```java
@OneToMany(
    mappedBy = "order",
    cascade = CascadeType.ALL,
    orphanRemoval = true
)
private List<OrderItem> items;
```

Then:

```java
order.removeItem(item);
```

can result in the child being deleted from the database.

### `CascadeType.REMOVE` vs `orphanRemoval`

```text
CascadeType.REMOVE
    → parent is deleted
    → child removal is cascaded

orphanRemoval
    → child is removed from parent's relationship
    → child is deleted
```

They are independent settings.

---

## 9. Best Practices

### 1. Cascade based on lifecycle ownership

Use cascading when the child's lifecycle is tightly coupled to the parent.

Good candidate:

```text
Order
 └── OrderItem
```

Potentially reasonable:

```java
cascade = CascadeType.ALL,
orphanRemoval = true
```

Avoid cascading blindly for independent business entities such as:

```text
Customer
 └── Orders
```

Deleting a Customer should not automatically mean deleting historical Orders unless that is explicitly the business rule.

### 2. Don't blindly use `CascadeType.ALL`

Use only the operations that make domain sense. `PERSIST + MERGE` is a useful combination when creation and merge should propagate but delete should not.

### 3. Be especially careful with cascade on `@ManyToOne`

For example, avoid casually doing:

```java
@ManyToOne(cascade = CascadeType.ALL)
private Customer customer;
```

An Order generally should not own the lifecycle of its Customer. Cascading REMOVE in this direction could cause dangerous deletes.

### 4. Prefer LAZY associations and explicit fetching

Don't use EAGER as a blanket solution. Use `JOIN FETCH`, `@EntityGraph`, or an appropriate query when a use case needs related data.

### 5. Keep bidirectional relationships synchronized

Use helper methods so both Java-side references remain consistent:

```java
public void addItem(OrderItem item) {
    items.add(item);
    item.setOrder(this);
}
```

### 6. Treat `orphanRemoval` as a data-deletion feature

It can cause a `DELETE` simply because a child was removed from a collection, so only use it when that is the intended lifecycle behavior.

---

## 10. Complete Practical Mapping

```java
@Entity
class Order {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "customer_id")
    private Customer customer;

    @OneToMany(
        mappedBy = "order",
        cascade = CascadeType.ALL,
        orphanRemoval = true
    )
    private List<OrderItem> items = new ArrayList<>();

    public void addItem(OrderItem item) {
        items.add(item);
        item.setOrder(this);
    }

    public void removeItem(OrderItem item) {
        items.remove(item);
        item.setOrder(null);
    }
}
```

```java
@Entity
class OrderItem {

    @Id
    @GeneratedValue
    private Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private Order order;
}
```

This mapping means:

```text
Order 1 ───── * OrderItem

OrderItem.order
    → owning side
    → controls orders.order_id

Order.items
    → inverse side
    → mappedBy = "order"

LAZY
    → associations aren't unnecessarily loaded

Cascade ALL
    → persist / merge / remove / refresh / detach propagate

orphanRemoval
    → removing an item from Order.items can delete that item
```

---

## Interview Mental Model

When designing or debugging a JPA relationship, ask in this order:

```text
1. What is the database relationship?
2. Where is the FK?
3. Which side owns the relationship?
4. Do I actually need navigation in both directions?
5. Should the association be LAZY or EAGER?
6. Could this create N+1 queries?
7. Which persistence operations should cascade?
8. Should removing a child from the relationship delete it?
9. What does the database FK do if the parent is deleted?
```

### Core distinctions to remember

```text
Relationship
    → @OneToMany / @ManyToOne / @OneToOne / @ManyToMany

Ownership
    → @JoinColumn / mappedBy

Fetching
    → LAZY / EAGER / explicit fetching

Lifecycle propagation
    → Cascade

Orphan lifecycle
    → orphanRemoval

Database parent deletion
    → FK ON DELETE behavior
```
