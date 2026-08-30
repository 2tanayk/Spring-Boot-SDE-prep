# 17 — Locking: Optimistic & Pessimistic

Locking is a concurrency-control technique used when multiple transactions may work with the same data concurrently.

The key distinction from isolation levels:

```text
Isolation level
→ controls transaction visibility/concurrency semantics broadly

Locking strategy
→ controls how specific conflicting operations are coordinated
```

## 1. The problem: lost updates

Suppose an account has:

```text
balance = 1000
```

Two requests can both read the same value:

```text
Request A                 Request B

read 1000                 read 1000
calculate 900             calculate 800
write 900                 write 800
```

One update can overwrite the other. This is a **lost update** scenario.

Optimistic and pessimistic locking are two ways to deal with such concurrency problems.

## 2. Pessimistic Locking

Pessimistic locking assumes a conflict may occur and acquires a lock before doing work on the resource.

In PostgreSQL, a common mechanism is:

```sql
BEGIN;

SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

`FOR UPDATE` requests a row-level lock appropriate for protecting the row from conflicting concurrent operations.

The typical flow is:

```text
BEGIN
  ↓
SELECT ... FOR UPDATE
  ↓
row locked
  ↓
read/check/modify
  ↓
COMMIT
  ↓
lock released
```

Another transaction attempting a conflicting operation may have to wait until the lock is released.

### Example: scarce inventory

Suppose only one item remains in stock.

```text
Transaction A
    ↓
lock product row
    ↓
check stock
    ↓
decrement
    ↓
commit

Transaction B
    ↓
tries to lock same row
    ↓
waits
    ↓
gets lock after A commits
    ↓
sees stock is now 0
    ↓
purchase fails
```

Pessimistic locking can be appropriate when contention is high and waiting is preferable to repeatedly detecting/retrying conflicts.

### Important

The lock is normally held within the transaction. `SELECT ... FOR UPDATE` should be understood as part of a transaction that performs the protected work, not as a standalone permanent lock.

## 3. Optimistic Locking

Optimistic locking assumes conflicts are relatively uncommon. Multiple transactions can read/work without acquiring a lock up front, and the application detects a conflict when saving.

A common mechanism is a version column.

```sql
CREATE TABLE accounts (
    id BIGINT PRIMARY KEY,
    balance NUMERIC(15, 2),
    version BIGINT NOT NULL DEFAULT 0
);
```

Suppose a row contains:

```text
id = 1
balance = 1000
version = 7
```

Two transactions both read version 7.

Transaction A updates using the version it originally read:

```sql
UPDATE accounts
SET balance = 900,
    version = version + 1
WHERE id = 1
  AND version = 7;
```

Result:

```text
1 row affected → success
```

The row is now version 8.

Transaction B still has version 7 and attempts:

```sql
UPDATE accounts
SET balance = 800,
    version = version + 1
WHERE id = 1
  AND version = 7;
```

No row matches:

```text
0 rows affected → conflict
```

The application interprets the zero-row update as an optimistic-locking conflict.

### Important: PostgreSQL does not know the column is an optimistic-locking column

`version` is just a normal column to PostgreSQL.

Optimistic locking is an **application-level pattern** built using ordinary database operations. Hibernate can implement this pattern automatically when a field is annotated with:

```java
@Version
private Long version;
```

Hibernate then generates a version-aware update and detects the failed update as an optimistic-locking conflict.

## 4. What to do after an optimistic-locking conflict

A conflict means another transaction changed the entity after it was read.

The application should not blindly overwrite the newer data.

Possible responses depend on the operation:

### Reject the request

For user-facing edits, return a conflict such as HTTP `409 Conflict` and ask the client to refresh/reapply its changes.

Typical flow:

```text
save
 ↓
version conflict
 ↓
409 Conflict
 ↓
client reloads latest state
```

### Reload and retry

For operations where retrying is safe, the application can reload the latest state, reapply the operation, and retry.

Be careful with non-idempotent operations or business logic whose decision depends on the old state.

### Merge

If two users changed different fields, the application may reload the latest entity, merge compatible changes, and save a new version. This is application/business logic; Hibernate cannot automatically understand arbitrary business-level conflicts.

## 5. Optimistic vs Pessimistic

| | Optimistic | Pessimistic |
|---|---|---|
| Assumption | Conflicts are uncommon | Conflicts may be common |
| Mechanism | Detect conflict at update time | Acquire lock before work |
| Blocking | Usually avoids upfront blocking | Can block competing transactions |
| Conflict handling | Fail/retry/merge | Wait for lock, then proceed |
| Typical mechanism | Version column / `@Version` | `SELECT ... FOR UPDATE` |
| Good fit | Low-contention edits | Highly contested resources |

### Mental model

```text
Optimistic:
read → work → check version → save
                       ↓
                  conflict?

Pessimistic:
lock → read/work → save → unlock
```

## 6. Locking is not an isolation level

They solve related concurrency problems but are different concepts.

```text
Isolation level
→ overall transaction visibility/concurrency semantics

Pessimistic lock
→ explicitly lock a particular resource/row

Optimistic lock
→ detect that a particular resource changed before saving
```

They can be combined. For example, an application can use PostgreSQL's `READ COMMITTED` isolation level while using either optimistic version checks or explicit row locks for particular operations.

## 7. Atomic SQL can sometimes be simpler

Do not automatically reach for locking.

If the business invariant can be expressed in one atomic SQL statement, that may be the simplest solution.

For inventory:

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 10
  AND quantity > 0;
```

Then:

```text
1 row affected → stock reserved/decremented
0 rows affected → condition not satisfied
```

This avoids a separate application-level `SELECT → decide → UPDATE` race for this particular state transition.

For multi-step business operations, transactions may still be required.

## Interview Quick Recall

> Pessimistic locking = acquire a lock first; conflicting operations may wait.

> PostgreSQL supports row-locking mechanisms such as `SELECT ... FOR UPDATE`.

> Optimistic locking = don't lock up front; detect a conflicting modification using a version check.

> PostgreSQL does not inherently know that `version` means optimistic locking; it is an application pattern.

> Hibernate's `@Version` automates the version-checking pattern.

> An optimistic conflict commonly results in a failed update/optimistic-lock exception; the application can reject, retry, or merge depending on the operation.

> Locking strategies are not isolation levels.

> If a business invariant fits cleanly into one atomic SQL statement, that can be preferable to adding a broader locking strategy.
