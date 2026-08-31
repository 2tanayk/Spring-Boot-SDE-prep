# 18 — Connection Pooling

A **connection pool** keeps a reusable collection of already-established database connections so the application does not have to create and destroy a database connection for every request.

## 1. What is a database connection?

Conceptually:

```text
Spring Boot
    ↓
JDBC driver
    ↓
TCP connection
    ↓
PostgreSQL
```

Creating a connection involves network setup, authentication, and session setup. Repeating this for every request is expensive and unnecessary.

Without pooling:

```text
request → create connection → query → close connection
```

With pooling:

```text
request → borrow connection → query → return connection
                                      ↓
                              connection stays in pool
```

## 2. How a connection pool works

The pool maintains a set of ready-to-use connections:

```text
Connection Pool
┌────────────────────┐
│ Conn 1             │
│ Conn 2             │
│ Conn 3             │
│ Conn 4             │
│ ...                │
└────────────────────┘
        ↑       ↓
     borrow    return
        ↑       ↓
     application
```

A request borrows an available connection, uses it for database work, and returns it to the pool. Returning it does not normally mean physically closing the database connection.

## 3. Spring Boot request flow

Conceptually:

```text
HTTP request
    ↓
Controller
    ↓
Service
    ↓
Repository
    ↓
borrow connection
    ↓
PostgreSQL query
    ↓
return connection to pool
    ↓
HTTP response
```

In ordinary Spring Data JPA/Hibernate code, connection lifecycle is generally managed by the framework rather than manually creating and closing JDBC connections.

## 4. Why not create one connection per request?

A large application might have thousands of concurrent HTTP requests, but the database cannot efficiently support an unlimited number of independent connections.

A pool provides controlled concurrency:

```text
10,000 HTTP requests
        ↓
connection pool
        ↓
20–50 database connections
        ↓
PostgreSQL
```

Requests that need database access compete for the available connections.

## 5. Pool exhaustion

Suppose:

```text
maximum pool size = 10
```

and all 10 connections are currently in use.

A new request attempts to borrow one:

```text
request
  ↓
borrow connection
  ↓
none available
  ↓
wait
```

The request waits according to the pool's configuration. If a connection does not become available within the configured acquisition timeout, the request can fail with a connection-acquisition timeout.

Therefore a pool is also a form of **backpressure**: it prevents unlimited database concurrency.

## 6. Pool size is not request count

Do not set pool size equal to the number of concurrent HTTP requests.

Many requests may be doing non-database work, waiting on other services, or otherwise not using a DB connection at the same time.

Pool size should be based on workload and database capacity, including:

- database CPU/I/O capacity
- query duration
- application workload
- transaction duration
- number of application instances
- database connection limits

## 7. Bigger pool is not always better

Increasing the pool can initially improve throughput if the pool is too small, but an excessively large pool can hurt performance:

```text
more connections
    ↓
more concurrent DB work
    ↓
CPU / I/O / memory contention
    ↓
queries can become slower
```

So pool sizing is a **concurrency and capacity problem**, not simply "make the pool as large as possible."

## 8. HikariCP

In modern Spring Boot applications, **HikariCP** is the common JDBC connection-pool implementation.

Conceptually:

```text
Spring Boot
    ↓
DataSource
    ↓
HikariCP
    ↓
JDBC connections
    ↓
PostgreSQL
```

Example configuration:

```properties
spring.datasource.hikari.maximum-pool-size=20
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.connection-timeout=30000
```

These are examples, not universal recommended values. Production values should be chosen based on the workload and database capacity.

## 9. Connection leaks

A connection leak occurs when an application borrows a connection and fails to return it to the pool.

Conceptually:

```text
borrow connection
    ↓
execute work
    ↓
BUG / incorrect resource handling
    ↓
connection never returned
```

Repeated leaks eventually cause:

```text
available connections ↓
        ↓
requests wait
        ↓
timeouts
        ↓
connection starvation
```

Spring/Hibernate normally manages connection lifecycle for standard JPA operations, but poorly managed resources, long-running transactions, or integration code can still contribute to connection starvation.

## 10. Connection pool vs transaction

These are different concepts.

```text
Connection pool
→ manages reusable database connections

Transaction
→ defines a unit of database work
```

They interact approximately as:

```text
@Transactional
      ↓
transaction begins
      ↓
connection obtained
      ↓
SQL operations
      ↓
commit / rollback
      ↓
connection returned to pool
```

The connection is generally returned to the pool rather than physically closed after each transaction.

## 11. Multiple application instances

Pool sizing must consider horizontal scaling.

For example:

```text
10 application instances
×
20 connections per instance
=
up to ~200 database connections
```

So a pool size that looks reasonable for one instance can become excessive after scaling to many instances.

## Interview Quick Recall

> Connection pooling reuses established database connections instead of creating one for every request.

> Requests borrow a connection, perform DB work, and return it to the pool.

> Pool exhaustion causes requests to wait and potentially time out.

> A pool is also a form of backpressure against unlimited DB concurrency.

> Bigger pool ≠ automatically better performance; excessive connections can cause database contention.

> HikariCP is the common connection-pool implementation used with modern Spring Boot applications.

> Connection leaks/starvation can make an application appear stuck even when application threads themselves are healthy.

> Pool sizing must account for the number of application instances, not just one instance.

> Connection pools manage connections; transactions manage units of database work.
