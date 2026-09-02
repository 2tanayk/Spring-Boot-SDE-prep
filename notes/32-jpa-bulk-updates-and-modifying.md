# JPA/Spring Data — Bulk Updates & `@Modifying`

## Bulk update

For large updates, don't load every entity and modify them one by one:

```java
List<User> users = userRepository.findInactiveUsers();

for (User user : users) {
    user.setActive(false);
}
```

Instead, use a bulk JPQL update:

```java
@Transactional
@Modifying
@Query("""
    UPDATE User u
    SET u.active = false
    WHERE u.lastLogin < :cutoff
""")
int deactivateInactiveUsers(LocalDateTime cutoff);
```

Hibernate sends a bulk SQL `UPDATE` directly to the database. The returned `int` is the number of affected rows.

## Why `@Modifying`?

Spring Data normally treats `@Query` as a SELECT. `@Modifying` tells Spring Data that the query changes data.

Typical combination:

```java
@Transactional
@Modifying
@Query("UPDATE User u SET u.active = false WHERE ...")
int deactivateInactiveUsers(...);
```

## Persistence-context trap

Bulk JPQL `UPDATE`/`DELETE` bypasses normal entity-level persistence-context synchronization.

If a `User` is already managed:

```java
User user = userRepository.findById(10L).get(); // active = true

userRepository.deactivateInactiveUsers(cutoff);
```

The database can now contain `active = false` while the already-managed Java object still contains `active = true`.

```text
Persistence Context       Database
------------------        --------
active = true        !=   active = false
```

This makes the persistence context potentially stale.

### `clearAutomatically`

```java
@Modifying(clearAutomatically = true)
```

Clears the persistence context after the modifying query, preventing stale managed entities from remaining around.

### `flushAutomatically`

```java
@Modifying(
    flushAutomatically = true,
    clearAutomatically = true
)
```

Useful when there are pending entity changes: flush them before the bulk query, execute the bulk operation, then clear the persistence context.

## Bulk DELETE

The same applies to bulk deletes:

```java
@Modifying
@Query("DELETE FROM User u WHERE u.active = false")
int deleteInactiveUsers();
```

## Normal JPQL SELECT vs bulk modification

A normal JPQL SELECT behaves normally with the persistence context:

```java
@Query("SELECT u FROM User u WHERE u.active = true")
List<User> findActiveUsers();
```

The returned entities become managed in the persistence context.

Bulk `UPDATE`/`DELETE` is different because it directly modifies database rows rather than loading and individually updating/removing managed entities.

## When to use bulk operations

Good for large operations such as:

- Updating millions of rows
- Archiving records
- Changing status for a large set of rows
- Deleting old records

Avoid loading huge numbers of entities just to change one simple field.

## Interview mental model

**Normal entity update:**

```text
Entity → Persistence Context → Dirty Checking → SQL
```

**Bulk update:**

```text
Bulk JPQL → Database rows directly
```

Key points:

- `@Modifying` = tells Spring Data the query modifies data.
- Bulk `UPDATE`/`DELETE` = bypasses normal entity-level persistence-context synchronization.
- `clearAutomatically` = clears potentially stale managed entities afterward.
- `flushAutomatically` = flushes pending changes before the bulk query.
