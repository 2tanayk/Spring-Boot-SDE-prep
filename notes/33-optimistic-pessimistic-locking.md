# JPA/Hibernate — Optimistic & Pessimistic Locking

## The problem: concurrent updates

Suppose two transactions read the same row:

```text
Account(id=1, balance=1000)

Transaction A → reads 1000
Transaction B → reads 1000
```

If both modify and save independently, one update can overwrite the other. This is the **lost update** problem.

Two common approaches:

- **Optimistic locking** → allow concurrent work, detect a conflict when writing.
- **Pessimistic locking** → lock the data so competing transactions cannot modify it concurrently.

---

## 1. Optimistic locking with `@Version`

```java
@Entity
public class Account {

    @Id
    private Long id;

    private BigDecimal balance;

    @Version
    private Long version;
}
```

Example DB state:

```text
id | balance | version
---+---------+--------
1  | 1000    | 5
```

Both transactions can initially read `version = 5`. **That is expected.** The version is checked when they write, not when they read.

Transaction A updates first. Hibernate effectively does:

```sql
UPDATE account
SET balance = 900,
    version = 6
WHERE id = 1
  AND version = 5;
```

One row is updated, so the operation succeeds and the version becomes 6.

Transaction B still has the old version 5 and tries:

```sql
UPDATE account
SET balance = 800,
    version = 6
WHERE id = 1
  AND version = 5;
```

No row matches because the database now has `version = 6`. Hibernate detects the failed update and raises an optimistic-locking exception (commonly `OptimisticLockException`, with Spring's exception translation depending on the setup).

### Mental model

```text
READ                         WRITE
  ↓                            ↓
version = 5              UPDATE ... WHERE version = 5
                              ↓
                     ┌────────┴────────┐
                     │                 │
                  success           conflict
                     ↓                 ↓
                version++       update doesn't apply
```

> **Optimistic locking:** “I don't care if you read the same version as me. When you save, prove that nobody changed it since I read it.”

### Relation to dirty checking

`@Version` does not replace dirty checking. Normal entity updates still use the persistence context and dirty checking; Hibernate additionally includes the version in the UPDATE condition and increments it after a successful update.

---

## 2. Pessimistic locking

Pessimistic locking acquires a database lock while working with a row.

Spring Data JPA example:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);
```

Conceptually, Hibernate may issue:

```sql
SELECT *
FROM account
WHERE id = ?
FOR UPDATE;
```

The first transaction obtains the row lock. A competing transaction trying to acquire the same lock generally waits until the first transaction commits or rolls back.

> **Pessimistic locking:** “I'm working on this row; you wait.”

---

## Optimistic vs pessimistic

| | Optimistic | Pessimistic |
|---|---|---|
| Basic idea | Detect conflicts | Prevent conflicts |
| Explicit DB lock | Usually no | Yes |
| JPA mechanism | `@Version` | `@Lock(PESSIMISTIC_*)` |
| Typical SQL | `UPDATE ... WHERE version=?` | `SELECT ... FOR UPDATE` |
| Conflict behavior | Detect/fail; application may retry | Competing transaction generally waits |
| Concurrency | Generally higher | Generally lower |
| Good when | Conflicts are rare | Contention is high / operation is sensitive |

### Example choice

**Optimistic:** employee profile edits where simultaneous edits to the same profile are uncommon.

**Pessimistic:** inventory where only one unit remains and concurrent reservations must be serialized.

---

## Isolation vs locking

Do not confuse transaction isolation with explicit locking.

**Isolation** asks:

> “What can this transaction see from other transactions?”

**Locking** asks:

> “Can another transaction concurrently modify/access this resource?”

For example:

```java
@Transactional(isolation = Isolation.REPEATABLE_READ)
```

is different from:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

They can be used together, but they solve different concerns.

---

## Serializable is not the same as pessimistic locking

`SERIALIZABLE` is an **isolation level**. Its guarantee is that concurrent transactions behave as though they were executed serially. The database chooses the mechanisms needed to provide that guarantee; depending on the database this can involve locks, predicate/range locking, MVCC conflict detection, or other techniques.

Pessimistic locking is an **explicit locking strategy** where the application requests a lock on particular data, such as `SELECT ... FOR UPDATE`.

So:

> **Serializable = a transaction-level correctness guarantee.**
>
> **Pessimistic locking = an explicit mechanism for protecting specific data.**

Serializable can cause transactions to block or abort due to serialization conflicts; pessimistic locking commonly causes competing transactions to wait for the lock.

---

## Interview summary

**Optimistic locking:** JPA uses `@Version`. Hibernate includes the version in the UPDATE condition. If another transaction has already changed the row, the version no longer matches, so the update fails instead of silently overwriting the other transaction's changes.

**Pessimistic locking:** acquire a database lock, commonly via `PESSIMISTIC_WRITE` / `SELECT ... FOR UPDATE`, so competing transactions cannot modify the locked row concurrently.

**Key distinction:** isolation controls transaction visibility/semantics; locking controls concurrent access to specific resources.
