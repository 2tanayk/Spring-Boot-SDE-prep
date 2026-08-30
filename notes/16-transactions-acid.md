# 16 — Transactions, ACID & Concurrency

## 1. What is a Transaction?

A **transaction** is a unit of work that the database treats as one logical operation.

Example: transferring ₹1,000 from account A to account B requires two changes:

```sql
UPDATE accounts
SET balance = balance - 1000
WHERE id = 1;

UPDATE accounts
SET balance = balance + 1000
WHERE id = 2;
```

These should behave as one unit:

```text
BEGIN
  ↓
debit A
  ↓
credit B
  ↓
COMMIT
```

If something fails:

```text
ROLLBACK
```

so the partial work is not left committed.

## 2. COMMIT vs ROLLBACK

`COMMIT` makes the transaction's changes committed/persistent.

`ROLLBACK` discards its uncommitted changes.

Conceptually:

```text
BEGIN → operations → COMMIT
                    ↘ failure → ROLLBACK
```

## 3. ACID

```text
A → Atomicity
C → Consistency
I → Isolation
D → Durability
```

### Atomicity

**All or nothing.** A transaction does not leave behind only part of its intended changes after rollback.

For a transfer, debit + credit must both succeed or neither should be committed.

### Consistency

A successfully committed transaction must leave the database satisfying its defined integrity rules and invariants.

Think:

```text
valid state → transaction → valid state
```

Consistency does **not** mean "everyone sees the latest data"; that is primarily an isolation/visibility concern.

### Isolation

Controls how concurrent transactions see and interfere with each other's work.

It answers questions such as:

- Can one transaction see another's uncommitted changes?
- Can a repeated read return a different value?
- Can a repeated query return a different set of rows?

### Durability

Once a transaction commits, its committed changes should survive failures such as a database crash/restart.

Databases use mechanisms such as write-ahead logging (WAL) and persistent storage to provide durability.

## 4. Atomicity vs Isolation

These are different guarantees.

```text
Atomicity
→ If my transaction fails halfway, do all its changes roll back?

Isolation
→ While transactions run concurrently, what can each transaction see/do?
```

## 5. Isolation Levels

The standard levels are:

```text
READ UNCOMMITTED
READ COMMITTED
REPEATABLE READ
SERIALIZABLE
```

Generally, stronger isolation provides stronger concurrency guarantees but can involve more coordination, contention, or transaction retries.

### READ UNCOMMITTED

The weakest conceptual isolation level. The SQL model permits dirty reads.

PostgreSQL does **not** provide dirty reads: requesting `READ UNCOMMITTED` is treated as `READ COMMITTED`.

It therefore isn't a meaningful PostgreSQL choice.

### READ COMMITTED

PostgreSQL's default isolation level.

A statement sees data committed before that statement's snapshot. In PostgreSQL, each statement in a READ COMMITTED transaction gets its own snapshot.

Therefore the same transaction can see different committed values on two separate statements:

```text
BEGIN

SELECT balance → 1000

another transaction changes balance → 500
and COMMITs

SELECT balance → 500

COMMIT
```

This is a **non-repeatable read**.

Typical use: ordinary CRUD, order creation, normal account/profile updates, and most application transactions where a transaction-wide frozen snapshot is not required.

### REPEATABLE READ

Provides a stable transaction-level snapshot in PostgreSQL.

Conceptually:

```text
BEGIN
  ↓
snapshot established
  ↓
SELECT → 1000
  ↓
other transaction commits a change
  ↓
SELECT → 1000
  ↓
COMMIT
```

Useful when multiple reads/calculations within one transaction must see a consistent point-in-time view, such as generating a consistent financial/reporting view.

PostgreSQL's REPEATABLE READ is stronger than the minimum SQL-standard definition and prevents the classic phantom-read phenomenon through its snapshot behavior.

### SERIALIZABLE

The strongest isolation level.

The goal is that the outcome is equivalent to some serial ordering of the concurrent transactions.

If concurrent transactions cannot safely be serialized, PostgreSQL can abort one with a serialization failure. The application must be prepared to retry.

Useful for operations where concurrent business decisions must behave as though transactions ran one at a time, especially complex cross-row business invariants.

Do not choose SERIALIZABLE everywhere simply because it is strongest; stronger isolation can reduce concurrency and introduce retries.

## 6. Concurrency Anomalies

### Dirty read

A transaction reads another transaction's uncommitted change:

```text
A: UPDATE balance = 500
   (not committed)

B: SELECT balance → 500

A: ROLLBACK
```

PostgreSQL does not allow this.

### Non-repeatable read

A transaction reads the same row twice and gets different values because another transaction committed a change between the reads.

```text
A: SELECT → 1000
B: UPDATE → 500; COMMIT
A: SELECT → 500
```

Possible under PostgreSQL READ COMMITTED; prevented by REPEATABLE READ and SERIALIZABLE.

### Phantom read

A transaction repeats a predicate query and sees a different set of matching rows because another transaction inserted/deleted matching rows and committed.

```text
A: SELECT WHERE amount > 1000 → 5 rows
B: INSERT matching row; COMMIT
A: SELECT WHERE amount > 1000 → 6 rows
```

PostgreSQL READ COMMITTED can show this behavior. PostgreSQL REPEATABLE READ provides a stable snapshot that prevents the classic phantom phenomenon; SERIALIZABLE provides serializability guarantees.

## 7. MVCC

PostgreSQL uses **Multi-Version Concurrency Control (MVCC)**.

Core idea:

> The database keeps track of row versions so transactions can determine which version is visible to their snapshot.

Conceptually:

```text
old version → balance = 1000
new version → balance = 500
```

A transaction's snapshot determines which version is visible to it.

MVCC allows readers to continue seeing an appropriate committed version while another transaction is working on a newer version, rather than making every reader wait for every writer.

### MVCC does not mean no locks

PostgreSQL uses MVCC **and** locks and other concurrency mechanisms.

Useful simplified mental model:

```text
MVCC
→ which row version can I see?

Locks
→ how do conflicting operations coordinate?
```

For example:

```sql
SELECT *
FROM accounts
WHERE id = 1
FOR UPDATE;
```

requests a row lock so conflicting operations can be coordinated.

## 8. Lost Updates

A lost update can occur when two transactions read the same value, independently calculate new values, and then overwrite one another.

```text
Initial balance = 1000

A reads 1000
B reads 1000

A calculates 900
B calculates 800

A writes 900
B writes 800
```

One logical update has been lost.

The exact behavior depends on the SQL statements, locking, isolation level, and database implementation.

Common solutions include:

- atomic SQL updates
- row locking
- optimistic locking/version checks
- appropriate transaction/isolation design

## 9. Atomic SQL Operations

An atomic SQL operation puts the **condition and state change into the same database statement**, allowing the database to perform the state transition safely as one operation.

### Unsafe read-then-write pattern

```text
SELECT quantity
    ↓
application checks quantity > 0
    ↓
application calculates quantity - 1
    ↓
UPDATE
```

Concurrent requests can both read the same quantity before either update occurs.

### Atomic conditional update

```sql
UPDATE inventory
SET quantity = quantity - 1
WHERE product_id = 10
  AND quantity > 0;
```

Then inspect the affected-row count:

```text
1 row affected → operation succeeded
0 rows affected → condition was not satisfied
```

The database evaluates the condition and performs the update as one statement.

### Banking example

Instead of:

```text
SELECT balance
→ application checks balance >= 1000
→ UPDATE balance
```

use an atomic conditional update where appropriate:

```sql
UPDATE accounts
SET balance = balance - 1000
WHERE id = 123
  AND balance >= 1000;
```

Then:

```text
1 row affected → withdrawal succeeded
0 rows affected → insufficient balance / condition not satisfied
```

## 10. Atomic SQL does not replace transactions

An atomic statement is useful for one state transition, but multi-step business operations still need a transaction.

For a transfer:

```text
BEGIN
  ↓
atomic debit
  ↓
atomic credit
  ↓
COMMIT
```

If the credit fails after the debit, the transaction can roll the whole operation back.

Useful combined mental model:

```text
Transaction
    +
Atomic SQL statements
    ↓
all-or-nothing business operation
with safe individual state transitions
```

## 11. Practical Isolation-Level Examples

```text
READ COMMITTED
→ normal CRUD/order/account operations
→ each statement sees a current committed snapshot

REPEATABLE READ
→ multi-read processing/reporting that needs a stable snapshot

SERIALIZABLE
→ complex concurrent business decisions where the result must be
  equivalent to serial execution
```

The appropriate choice depends on the business invariant and workload. Often an atomic SQL statement or targeted row lock is preferable to increasing the isolation level for an entire transaction.

## Interview Quick Recall

> Transaction = a logical unit of database work.

> Atomicity = all or nothing.

> Consistency = committed state satisfies defined constraints/invariants.

> Isolation = controls visibility/interference between concurrent transactions.

> Durability = committed changes survive failures.

> PostgreSQL default isolation = READ COMMITTED.

> READ COMMITTED uses statement-level snapshots in PostgreSQL.

> REPEATABLE READ gives a stable transaction-level snapshot in PostgreSQL.

> SERIALIZABLE aims for an outcome equivalent to serial execution and can require retries after serialization failures.

> PostgreSQL does not allow dirty reads and maps READ UNCOMMITTED to READ COMMITTED.

> MVCC manages visibility of row versions; it does not eliminate locks.

> Atomic SQL means combining the condition and state transition into one database statement where appropriate.

> Atomic SQL does not replace a transaction when multiple statements form one logical business operation.
