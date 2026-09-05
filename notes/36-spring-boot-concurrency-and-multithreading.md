# Concurrency & Multithreading in Production Spring Boot

## 1. Concurrent Requests

Spring Boot commonly runs on Tomcat/Jetty, whose worker-thread pool processes HTTP requests concurrently.

```text
HTTP requests
      ↓
HTTP server thread pool
      ↓
Controller → Service → Repository
      ↓
Database
```

You normally do **not** create a thread per request yourself.

A singleton Spring bean can be executed concurrently by many threads:

```text
              OrderService instance
                 /       |       \
              T1        T2        T3
           request A  request B  request C
```

Therefore singleton beans should generally be **thread-safe**.

### Safe vs unsafe state

Local variables are thread-confined:

```java
public void process() {
    int total = calculateTotal();
}
```

Mutable instance fields in singleton beans are shared:

```java
private int counter;
```

`counter++` can race because read/add/write are separate operations.

**Production rule:** prefer stateless singleton services and keep request-specific state in local variables.

---

## 2. Database Concurrency

Not every concurrency problem is a Java-thread problem. Shared business state is often protected by database transactions and concurrency control.

Example: two requests withdraw ₹700 from an account containing ₹1000. Both could read ₹1000 unless the operation is protected.

Pessimistic locking can serialize access:

```java
@Transactional
public void withdraw(Long accountId, BigDecimal amount) {
    Account account = accountRepository.findByIdForUpdate(accountId);

    if (account.getBalance().compareTo(amount) < 0) {
        throw new InsufficientFundsException();
    }

    account.setBalance(account.getBalance().subtract(amount));
}
```

```java
@Lock(LockModeType.PESSIMISTIC_WRITE)
Optional<Account> findById(Long id);
```

Conceptually this uses `SELECT ... FOR UPDATE`.

For lower-contention cases, optimistic locking with `@Version` detects conflicting updates rather than blocking them.

**Java-level synchronization protects one JVM; database concurrency control protects shared persistent state across application instances.**

---

## 3. Asynchronous Work

Some work does not need to block the HTTP request:

```text
Place order
   ├── save order
   ├── send email
   └── update analytics
```

Spring can run asynchronous methods using an executor:

```java
@Async
public void sendOrderEmail(...) {
    ...
}
```

with async support enabled.

`@Async` is not free parallelism: it consumes another finite thread-pool resource.

Also remember the Spring proxy model: self-invocation can bypass `@Async`, just like `@Transactional`.

```text
process()
   ↓
this.sendOrderEmail()
   X  proxy bypassed
```

Prefer calling the async method through another Spring bean.

---

## 4. Thread Pools & Executors

Production applications use bounded thread pools rather than creating unlimited threads.

```java
@Bean
public Executor orderExecutor() {
    ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
    executor.setCorePoolSize(10);
    executor.setMaxPoolSize(20);
    executor.setQueueCapacity(100);
    return executor;
}
```

Conceptually:

```text
Tasks → queue → worker threads
```

If 100 tasks arrive and only 10 workers exist, the remaining work waits in the queue rather than creating 100 threads.

Too many threads can cause context switching, memory overhead and contention.

---

## 5. CPU-Bound vs I/O-Bound

### CPU-bound

Examples:

- complex calculations
- image processing
- heavy hashing

Threads spend their time consuming CPU. Creating far more threads than available CPU capacity can hurt performance.

### I/O-bound

Examples:

- database calls
- HTTP calls
- file operations

Threads often spend time waiting, so concurrency can improve throughput by allowing other work to execute during waits.

---

## 6. `CompletableFuture`

Suppose an API needs three independent downstream calls:

```text
             ┌── Customer Service
Request ─────┼── Fraud Service
             └── Rewards Service
```

Sequential execution takes approximately the sum of their latencies.

Independent calls can be started concurrently:

```java
CompletableFuture<Customer> customer =
    CompletableFuture.supplyAsync(
        () -> customerClient.getCustomer(id), executor);

CompletableFuture<FraudResult> fraud =
    CompletableFuture.supplyAsync(
        () -> fraudClient.check(id), executor);

CompletableFuture<Rewards> rewards =
    CompletableFuture.supplyAsync(
        () -> rewardsClient.getRewards(id), executor);

CompletableFuture.allOf(customer, fraud, rewards).join();
```

Latency can approach the slowest call rather than the sum, **provided the executor and downstream systems can handle the extra concurrency**.

Avoid blocking tasks waiting on other tasks from the same small executor; this can cause thread-pool starvation.

---

## 7. Thread-Safe Collections & Atomics

`ArrayList` and `HashMap` are not designed for concurrent modification. `ConcurrentHashMap` supports concurrent access.

But a thread-safe collection does not automatically make a multi-step business operation atomic.

Instead of:

```java
if (!map.containsKey(id)) {
    map.put(id, createOrder());
}
```

use an atomic operation where appropriate:

```java
map.computeIfAbsent(id, key -> createOrder());
```

`AtomicInteger` can safely perform atomic in-memory updates:

```java
AtomicInteger counter = new AtomicInteger();
counter.incrementAndGet();
```

But an atomic variable only protects state inside **one JVM**. It is not a replacement for database locking/transactions for shared business state such as account balances or inventory.

---

## 8. Multiple Application Instances

Production often looks like:

```text
             Load Balancer
              /    |    \
             /     |     \
         App #1  App #2  App #3
```

Each JVM has its own heap, threads, locks and atomic variables.

Therefore:

> `synchronized`, `AtomicInteger`, and Java locks coordinate threads within one JVM only.

Cross-instance coordination needs shared infrastructure such as:

- database transactions/locks
- Redis or another shared store
- message brokers
- distributed locking where genuinely required

---

## 9. Production Resource Pipeline

Concurrency is not just about Java threads. A typical service has several bounded resources:

```text
HTTP requests
      ↓
Tomcat worker threads
      ↓
Spring service
      ↓
Async executor (if used)
      ↓
Hikari connection pool
      ↓
Database
```

For example:

```text
1000 incoming requests
        ↓
200 HTTP workers
        ↓
30 DB connections
        ↓
Database
```

Increasing application threads from 20 to 500 does not create more database capacity if the DB pool still has only 30 connections. It may simply increase waiting and contention.

Thread pools, queues and connection pools therefore need to be considered together.

---

## Key Mental Model

```text
                 CONCURRENCY
                      │
        ┌─────────────┴─────────────┐
        │                           │
    JVM-level                  Distributed
    concurrency                concurrency
        │                           │
   Threads                     Multiple JVMs
   Executors                       │
   Locks                           ├── DB transactions
   Atomics                         ├── Optimistic locking
   Concurrent collections          ├── Pessimistic locking
                                   ├── Redis/shared state
                                   └── Messaging
```

### SDE-2 Interview Answer

**Spring Boot applications handle concurrent requests using a server-managed worker-thread pool. Spring beans are usually singletons, so services should generally be stateless and thread-safe rather than storing mutable request state in fields. Asynchronous work can use dedicated executors, `@Async` or `CompletableFuture`, but thread pools are finite resources and must be sized with downstream capacity such as the database connection pool. For shared business state, Java synchronization only protects one JVM; in a multi-instance deployment, transactions and database concurrency controls such as optimistic or pessimistic locking are usually required.**
