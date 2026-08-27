# 05 — Caching

Caching stores a temporary copy of data so frequently requested data can be served faster and with less load on the primary data store.

## Why Caching Exists

Without a cache:

```text
Request → API → Database → Response
```

If the same data is requested repeatedly, the database may perform the same expensive read many times.

With a cache:

```text
Request → API → Cache
                  │
              ┌───┴───┐
             HIT     MISS
              │        │
           Return   Database
                       │
                     Cache
                       │
                     Return
```

The database remains the **source of truth** when the cache is being used purely as a cache.

---

## Cache-Aside

**Cache-aside** means the application explicitly manages the cache.

### Read

1. Application checks the cache.
2. If the entry exists (**cache hit**), return it.
3. If it does not exist (**cache miss**), query the database.
4. Put the result into the cache.
5. Return the result.

Example:

```text
GET /users/42
      ↓
Cache["user:42"]
      ↓
    MISS
      ↓
Database
      ↓
User 42
      ↓
Cache["user:42"] = User 42
      ↓
Response
```

On subsequent reads, the cache can satisfy the request without querying the database.

### Why the name?

The cache sits **aside from the database** and the application decides when to read from or write to it. The database does not need to know that the application is caching its data.

---

## Cache Invalidation

A cached value can become stale when the underlying database value changes.

Example:

```text
Database: role = USER
Cache:    role = USER

Database updated:
role = ADMIN

Cache still:
role = USER
```

A common cache-aside write strategy is:

```text
Update Database
      ↓
Invalidate/Delete Cache Entry
```

Then the next read is a cache miss and repopulates the cache with the fresh database value.

```text
UPDATE DB
   ↓
DELETE cache["user:42"]
   ↓
Next GET
   ↓
Cache MISS
   ↓
DB → fresh value
   ↓
Cache → fresh value
```

Cache invalidation is difficult because the cache is a second copy of the data whose consistency must be managed.

---

## TTL — Time To Live

A cache entry can have a **TTL**, which defines how long it is allowed to remain cached.

Example:

```text
"user:42" → User 42
TTL = 5 minutes
```

After the TTL expires, the entry is no longer usable and a later request can fetch fresh data from the database.

TTL is useful because it puts a bound on how long a stale entry can remain in the cache.

### TTL vs Invalidation

They solve different problems:

```text
Explicit invalidation:
Data changes → delete/update cache immediately

TTL:
Cache entry exists → expires automatically after a period
```

They can be used together. Explicit invalidation provides freshness when writes are known, while TTL provides a fallback if an invalidation path fails.

---

## Redis

**Redis** is an in-memory data store commonly used as a distributed cache.

A simple mental model is:

```text
key → value

"user:42" → serialized user data
"product:1001" → serialized product data
```

Redis supports multiple data structures, but for basic caching the key-value model is sufficient.

### Why Redis?

Redis is useful when applications repeatedly need the same data and database reads are becoming a bottleneck.

Example:

```text
1000 requests for product:1001

Without cache:
1000 database reads

With Redis:
Redis handles most reads
Database handles cache misses
```

---

## Distributed Cache vs Local Cache

Suppose an application is horizontally scaled:

```text
             Load Balancer
            /      |      \
           ↓       ↓       ↓
        App 1   App 2   App 3
```

With separate in-process caches:

```text
App 1 → Cache A
App 2 → Cache B
App 3 → Cache C
```

Different instances can hold different values.

A shared Redis cache gives:

```text
             Redis
            /  |  \
           ↓   ↓   ↓
        App 1 App 2 App 3
```

All application instances can use the same cached data.

This is one reason Redis is useful in horizontally scaled services.

---

## Cache Failure

If Redis is being used purely as a cache, its failure should ideally affect **performance rather than correctness**.

For example:

```text
Redis unavailable
      ↓
Fall back to database
      ↓
Serve request, possibly with higher latency
```

The key principle is:

> A cache should normally improve performance, not become the only source of truth.

If Redis is being used for something other than caching, such as distributed locks or application state, its failure characteristics are different.

---

## Real-World Examples

### Banking — account metadata

Frequently requested, relatively stable account metadata can potentially be cached:

```text
GET /accounts/12345
        ↓
Redis → HIT → return
        ↓
      MISS
        ↓
PostgreSQL → Redis → return
```

However, authoritative transactional values such as an account balance require much stronger freshness/consistency considerations. Whether to cache them depends on how the value is used and the business requirements.

### E-commerce — product catalogue

Product information may be requested thousands of times while changing relatively infrequently:

```text
product:1001 → { name, price, category, ... }
```

A cache can significantly reduce repeated database reads. When a product changes, the application can invalidate the corresponding cache entry.

### Reference data

Data such as country lists, currencies, branches, or transaction types may change rarely and can be cached with an appropriate TTL.

The acceptable TTL depends on how much staleness the business can tolerate.

---

## Trade-Off: Performance vs Freshness

Caching introduces a fundamental trade-off:

```text
More caching
→ lower latency / lower DB load
→ potentially more stale data

Less caching
→ fresher data
→ more database load / potentially higher latency
```

There is no universally correct TTL. The correct strategy depends on the consistency requirements of the data.

---

## Interview Quick Recall

> Cache = temporary copy of data used to improve access speed and reduce load on the primary data store.

> Cache-aside = application checks cache → database on miss → populate cache.

> Database = source of truth when Redis is used purely as a cache.

> Cache invalidation = removing/updating cached data when the underlying data changes.

> TTL = maximum lifetime configured for a cache entry.

> Redis = in-memory data store commonly used as a shared/distributed cache.

> Shared Redis avoids each horizontally scaled application instance maintaining an independent cache.

> A cache failure should ideally degrade performance rather than compromise correctness when the cache is not the source of truth.

> Caching trades freshness/consistency for performance and lower database load.
