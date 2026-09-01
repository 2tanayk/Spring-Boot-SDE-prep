# 27 — JPA/Hibernate — Persistence Context, First-Level Cache, Entity Lifecycle, Dirty Checking, Flush & Lazy Loading

## 1. Persistence Context

A persistence context is Hibernate/JPA's workspace of **managed entity objects** for a unit of work.

```text
Persistence Context
┌───────────────────────────────┐
│ Account #42 → Java object     │
│ Account #51 → Java object     │
└───────────────────────────────┘
```

When an entity is managed, Hibernate tracks it and can detect changes to it.

## 2. First-level cache

The persistence context also provides Hibernate's **first-level cache**.

```java
Account a1 = accountRepository.findById(42L).orElseThrow();
Account a2 = accountRepository.findById(42L).orElseThrow();
```

Within the same persistence context, Hibernate can reuse the already-managed entity instead of loading the same database row again.

Conceptually:

```text
find(Account, 42)
       ↓
Already managed in persistence context?
       ↓
      YES
       ↓
return existing entity instance
```

Therefore, within the same persistence context, the same entity identity is represented by the same managed Java object.

Important:

- automatic
- scoped to the persistence context
- not a distributed cache
- not shared across application instances
- different from Hibernate's optional second-level cache

> **Persistence Context is the broader concept; first-level cache is one important behavior of it.**

## 3. Entity lifecycle — using Spring Data JPA code

In day-to-day Spring Boot code, you usually interact through `JpaRepository`, not `EntityManager` directly. The lifecycle can be understood through operations such as `new`, `save`, `findById`, modifying fields, and `delete`.

### Transient

Creating an entity normally:

```java
Account account = new Account();
account.setName("Tanay");
account.setBalance(BigDecimal.valueOf(1000));
```

Before it is saved, Hibernate does not manage it.

```text
new Account()
     ↓
  TRANSIENT
```

### Managed

For a new entity:

```java
accountRepository.save(account);
```

Spring Data JPA ultimately uses JPA persistence operations to make the new entity persistent/managed.

```text
TRANSIENT
    ↓
repository.save()
    ↓
MANAGED
```

The important point is that `save()` does not necessarily mean the `INSERT` SQL executes immediately. SQL execution is tied to flushing.

### Existing entity becomes managed

A much more common operation is:

```java
@Transactional
public void renameAccount(Long id) {

    Account account =
            accountRepository.findById(id)
                    .orElseThrow();

    account.setName("New Name");
}
```

`findById()` loads the entity and the resulting entity is managed by the persistence context associated with the transaction.

```text
findById()
    ↓
SELECT
    ↓
Account object
    ↓
Persistence Context
    ↓
MANAGED
```

### Dirty checking

Because the entity is managed, Hibernate tracks its state.

```java
account.setName("New Name");
```

No explicit `update()` is required.

Conceptually:

```text
Original state:
name = "Old Name"
       ↓
account.setName("New Name")
       ↓
Hibernate compares current state with tracked/original state
       ↓
Changed → entity is dirty
```

At flush, Hibernate generates the required SQL:

```sql
UPDATE account
SET name = 'New Name'
WHERE id = 42;
```

> **Dirty checking = Hibernate detects changes made to managed entities and synchronizes those changes to the database during flush.**

### No `save()` required for a managed update

This is important in day-to-day Spring Data JPA code:

```java
@Transactional
public void renameAccount(Long id) {
    Account account = accountRepository.findById(id).orElseThrow();
    account.setName("New Name");
}
```

You generally do **not** need:

```java
accountRepository.save(account);
```

The entity is already managed, so dirty checking + flush can produce the update.

### Detached

After the persistence context ends, the Java entity object can still exist in memory, but Hibernate is no longer tracking it through that persistence context.

```text
Managed entity
      ↓
Persistence Context ends
      ↓
Detached entity
```

Changing a detached entity does not automatically trigger an update because Hibernate is no longer tracking that object.

If an existing detached entity is passed to:

```java
repository.save(detachedEntity);
```

Spring Data JPA may use JPA `merge()` underneath. Conceptually, the detached state is copied onto a managed representation, which can then participate in dirty checking and flushing.

Important nuance:

> `merge()` should not be thought of as simply reattaching the exact same Java object. It returns/works with a managed representation whose state incorporates the detached entity's state.

### Removed

For example:

```java
@Transactional
public void deleteAccount(Long id) {
    Account account = repository.findById(id).orElseThrow();
    repository.delete(account);
}
```

Conceptually:

```text
MANAGED
   ↓
repository.delete()
   ↓
REMOVED
   ↓
flush
   ↓
DELETE SQL
```

Again, `delete()` does not necessarily mean the SQL executes at that exact line; synchronization happens during flush.

## 4. Lifecycle mental model

Forget the `EntityManager` API and remember the operations you actually write:

```text
new Account()
     ↓
 TRANSIENT
     │
     │ repository.save()
     ▼
 MANAGED
     │
     │ modify fields
     ▼
 dirty / changed state
     │
     │ flush
     ▼
 SQL
     │
     │ persistence context ends
     ▼
 DETACHED
```

Deletion:

```text
MANAGED
   │
   │ repository.delete()
   ▼
REMOVED
   │
   │ flush
   ▼
DELETE SQL
```

## 5. Flush vs commit

**Flush** means:

> Synchronize the persistence context's changes with the database.

**Commit** means:

> Commit the database transaction.

Conceptually:

```text
Java changes
    ↓
Persistence Context
    ↓
flush
    ↓
SQL sent to DB
    ↓
transaction commit
```

With normal transactional behavior, Hibernate commonly flushes before transaction commit. Flush can also happen earlier or be explicitly requested.

Therefore, do not say:

> "Hibernate only sends SQL at commit."

Prefer:

> **Hibernate normally flushes before transaction commit, but flushing can happen earlier or be triggered explicitly.**

## 6. Lazy loading

Consider:

```java
@ManyToOne(fetch = FetchType.LAZY)
private Customer customer;
```

When an `Order` is loaded, Hibernate may initially load the order without loading the associated `Customer` immediately.

Conceptually:

```text
Order
 ├── id
 ├── amount
 └── customer → not loaded yet
```

When code accesses:

```java
order.getCustomer().getName();
```

Hibernate can initialize the lazy association by fetching the required data:

```text
order.getCustomer()
       ↓
Hibernate lazy-loading mechanism
       ↓
SELECT customer ...
       ↓
Customer loaded
```

> **Lazy loading postpones loading an association until it is actually needed.**

## 7. `LazyInitializationException`

A common problem is accessing an unloaded lazy association after the persistence context is no longer available.

For example, conceptually:

```text
transaction / persistence context
        ↓
find Order
        ↓
Order returned with Customer still lazy
        ↓
persistence context ends
        ↓
later: order.getCustomer()
        ↓
Hibernate needs to load Customer
        ↓
no active persistence context
        ↓
LazyInitializationException
```

The core problem is:

> **Hibernate needs an active persistence context to initialize an unloaded lazy association.**

This is one reason explicit fetching strategies such as `@EntityGraph` are useful when a use case needs the association as part of the query.

## 8. Complete real-world example

```java
@Transactional
public void updateAccount(Long id, String newName) {

    Account account =
            accountRepository.findById(id)
                    .orElseThrow();

    account.setName(newName);
}
```

Mechanically:

```text
@Transactional starts
        ↓
Persistence Context associated
        ↓
findById()
        ↓
SELECT
        ↓
Account becomes MANAGED
        ↓
account.setName(...)
        ↓
Hibernate tracks changed state
        ↓
method finishes
        ↓
flush
        ↓
dirty checking
        ↓
UPDATE account ...
        ↓
transaction commit
        ↓
persistence context ends
```

## Interview Quick Recall

> **Persistence Context** = workspace of managed entities associated with a unit of work.

> **First-level cache** = persistence-context-level caching/reuse of managed entity instances.

> **Transient** = newly created entity not managed by Hibernate.

> **Managed** = entity tracked by the persistence context.

> **Detached** = entity still exists in memory but is no longer tracked by that persistence context.

> **Removed** = managed entity marked for deletion.

> **Dirty checking** = Hibernate detects changes to managed entities and generates the required SQL during flush.

> **Flush ≠ commit.** Flush synchronizes changes to the DB; commit completes the transaction.

> A managed entity loaded with `findById()` can generally be updated by simply changing its fields; an explicit `save()` is not required for dirty checking to work.

> `repository.save()` does not necessarily execute SQL immediately.

> **Lazy loading** postpones loading associations until accessed.

> An unloaded lazy association generally requires an active persistence context when it is initialized; otherwise `LazyInitializationException` can occur.

### The core flow

```text
DB
 ↓
Entity
 ↓
Persistence Context
 ↓
Managed entity
 ↓
Modify Java object
 ↓
Dirty checking
 ↓
Flush
 ↓
SQL
 ↓
Commit
```
