# `@Transactional` — Proxy, Propagation, Isolation & Self-Invocation

## 1. What `@Transactional` Does

`@Transactional` tells Spring to execute a method within a database transaction.

Conceptually:

```text
BEGIN
    business logic
COMMIT
```

If the transaction fails and rollback rules apply:

```text
BEGIN
    business logic
    exception
ROLLBACK
```

Spring handles the transaction machinery around the business method rather than requiring transaction code inside the method.

---

## 2. Proxy-Based Implementation

Spring commonly implements `@Transactional` using a proxy around the target bean:

```text
Caller
  ↓
Spring Proxy
  ↓
Actual Service
```

The proxy intercepts the call, starts/joins a transaction, invokes the target method, and then commits or rolls back.

Conceptually:

```java
beginTransaction();
try {
    actualMethod();
    commit();
} catch (Exception e) {
    rollback();
    throw e;
}
```

This is the same proxy-based AOP mechanism used by other Spring features.

---

## 3. Self-Invocation

Because the transaction interceptor lives on the proxy, only calls that cross the proxy boundary are intercepted.

Example:

```java
@Service
class OrderService {

    public void createOrder() {
        saveOrder();
    }

    @Transactional
    public void saveOrder() {
        // database work
    }
}
```

The internal call is effectively:

```java
this.saveOrder();
```

So the flow is:

```text
Caller
  ↓
Proxy
  ↓
createOrder()
  ↓
this.saveOrder()
```

The second call bypasses the proxy, so the `@Transactional` interceptor on `saveOrder()` does not get a chance to run.

### Best practice

Move the transactional operation to another Spring bean when appropriate:

```java
@Service
class OrderService {

    private final OrderPersistenceService persistenceService;

    public void createOrder() {
        persistenceService.saveOrder();
    }
}

@Service
class OrderPersistenceService {

    @Transactional
    public void saveOrder() {
        // database work
    }
}
```

Now the call crosses a Spring proxy.

---

## 4. Transaction Propagation

Propagation answers:

> **If a transactional method calls another transactional method, which transaction should the second method participate in?**

### `REQUIRED` — default

```java
@Transactional(propagation = Propagation.REQUIRED)
```

- If a transaction already exists → join it.
- If none exists → create one.

Example:

```text
placeOrder() → T1
    ↓
charge() → joins T1
    ↓
reserve() → joins T1
```

This is the normal default for business operations.

---

### `REQUIRES_NEW`

```java
@Transactional(propagation = Propagation.REQUIRES_NEW)
```

Always creates a new transaction. If an existing transaction exists, it is suspended.

```text
T1 active
   ↓
suspend T1
   ↓
start T2
   ↓
run method
   ↓
commit/rollback T2
   ↓
resume T1
```

T2 is independent of T1. Therefore:

```text
T2 commits
T1 later rolls back
→ T2's changes remain
```

A common use case is an audit/log record that must survive failure of the main transaction.

Use carefully because it creates an independent transaction and can have concurrency/connection implications.

---

### `MANDATORY`

```java
@Transactional(propagation = Propagation.MANDATORY)
```

A transaction must already exist.

```text
Existing transaction → join it
No transaction       → exception
```

Useful when a method should only be called as part of a larger transaction.

---

### `SUPPORTS`

```java
@Transactional(propagation = Propagation.SUPPORTS)
```

- Existing transaction → join it.
- No transaction → run without one.

---

### `NOT_SUPPORTED`

```java
@Transactional(propagation = Propagation.NOT_SUPPORTED)
```

Runs without a transaction. If one exists, Spring suspends it temporarily.

---

### `NEVER`

```java
@Transactional(propagation = Propagation.NEVER)
```

A transaction must not exist. If one exists, an exception is raised.

---

### `NESTED`

```java
@Transactional(propagation = Propagation.NESTED)
```

Generally uses a database savepoint inside an existing transaction.

Conceptually:

```text
T1
 ├── operation A
 ├── SAVEPOINT
 ├── operation B
 ├── rollback to SAVEPOINT
 └── operation C
COMMIT T1
```

The exact behavior depends on the transaction manager/database. Know the concept, but it is lower priority than `REQUIRED` and `REQUIRES_NEW` for typical SDE-2 interviews.

---

## 5. Propagation Cheat Sheet

| Propagation | Existing transaction | No transaction |
|---|---|---|
| `REQUIRED` ⭐ | Join | Create |
| `REQUIRES_NEW` ⭐ | Suspend + create new | Create |
| `MANDATORY` | Join | Exception |
| `SUPPORTS` | Join | Run without |
| `NOT_SUPPORTED` | Suspend | Run without |
| `NEVER` | Exception | Run without |
| `NESTED` | Savepoint | Typically starts/uses a transaction depending on configuration |

The two to know deeply are `REQUIRED` and `REQUIRES_NEW`.

---

## 6. Isolation

Isolation answers:

> **What can concurrent transactions see and how are their operations isolated from each other?**

Example:

```java
@Transactional(isolation = Isolation.READ_COMMITTED)
```

The standard levels are:

```text
READ_UNCOMMITTED
READ_COMMITTED
REPEATABLE_READ
SERIALIZABLE
```

Higher isolation generally provides stronger concurrency guarantees at the cost of concurrency/performance.

---

### `READ_UNCOMMITTED`

A transaction can potentially see uncommitted changes from another transaction.

Example:

```text
T1: balance = 500 (not committed)
T2: reads 500
T1: ROLLBACK
```

T2 saw data that never committed.

This is a **dirty read**.

Rarely appropriate for serious transactional business logic.

---

### `READ_COMMITTED`

A transaction only sees committed changes.

It prevents dirty reads, but repeated reads can see different committed values:

```text
T2: SELECT → 100
T1: UPDATE → 200 + COMMIT
T2: SELECT → 200
```

This is a **non-repeatable read**.

PostgreSQL's default isolation level is `READ COMMITTED`.

---

### `REPEATABLE_READ`

Provides a stronger, consistent view for repeated reads according to the database's isolation semantics.

Conceptually:

```text
T2: SELECT → 100
T1: UPDATE → 200 + COMMIT
T2: SELECT → still sees its consistent version
```

Exact concurrency behavior varies by database implementation.

---

### `SERIALIZABLE`

Strongest standard isolation level.

The database attempts to make concurrent transactions behave as if they executed serially rather than with problematic interleavings.

Trade-off:

```text
More isolation
    ↓
Less concurrency
    ↓
More contention / possible serialization failures
```

Do not use it everywhere by default.

---

## 7. Isolation vs Locking

These are related but different concepts.

**Isolation** defines the rules for visibility and concurrency between transactions.

**Locking** is one mechanism databases can use to enforce concurrency guarantees.

For example:

```java
@Transactional(isolation = Isolation.SERIALIZABLE)
```

sets transaction isolation.

Whereas:

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
```

requests an explicit pessimistic lock.

---

## 8. Rollback Rules

By default, Spring rolls back transactions for unchecked exceptions (`RuntimeException`) and `Error`.

Example:

```java
@Transactional
public void transfer() {
    debit();
    throw new RuntimeException();
}
```

→ rollback.

Checked exceptions do not automatically trigger rollback by default.

You can configure this explicitly:

```java
@Transactional(rollbackFor = IOException.class)
public void process() throws IOException {
    // ...
}
```

---

## 9. Complete Example

```java
@Service
class OrderService {

    @Transactional(
        propagation = Propagation.REQUIRED,
        isolation = Isolation.READ_COMMITTED
    )
    public void placeOrder(Long orderId) {

        orderRepository.updateStatus(orderId, "PLACED");
        paymentService.charge(orderId);
        inventoryService.reserve(orderId);
    }
}
```

If `charge()` is also `REQUIRED`:

```text
placeOrder()
     ↓
    T1
     ├── update order
     ├── charge() → joins T1
     └── reserve() → joins T1
```

If `charge()` fails with a rollback-triggering exception:

```text
T1
 ├── update order
 ├── charge() ❌
 └── ROLLBACK
```

If `charge()` instead uses `REQUIRES_NEW`:

```text
T1: placeOrder
 │
 ├── update order
 │
 ├── suspend T1
 │
 │   T2: charge
 │       ↓
 │      COMMIT
 │
 ├── resume T1
 └── continue
```

The payment transaction can commit independently of the outer transaction.

---

## 10. Interview Mental Model

Think of `@Transactional` as several independent concerns:

```text
@Transactional
      │
      ├── Proxy
      │     → How does Spring intercept the method?
      │
      ├── Propagation
      │     → Which transaction do I participate in?
      │
      ├── Isolation
      │     → What can concurrent transactions see?
      │
      └── Rollback rules
            → What causes this transaction to roll back?
```

The key proxy rule:

> **Spring proxy-based transaction interception only happens when the call crosses the proxy boundary. Self-invocation bypasses the proxy.**

### SDE-2 priority

🟢 **Must know deeply**
- Proxy-based implementation
- Self-invocation
- `REQUIRED`
- `REQUIRES_NEW`
- `READ_COMMITTED`
- `REPEATABLE_READ`
- `SERIALIZABLE`
- Rollback rules
- Propagation vs isolation

🟡 **Know conceptually**
- `MANDATORY`
- `SUPPORTS`
- `NOT_SUPPORTED`
- `NEVER`
- `NESTED`
