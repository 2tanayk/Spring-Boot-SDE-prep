# 21 — SQL vs NoSQL

The key question is not "Which is better?" but:

> **What kind of data and workload does the application have, and which database model fits it?**

## 1. SQL databases

SQL databases are relational databases. Examples include PostgreSQL, MySQL, Oracle, and SQL Server.

They organize data into related tables:

```text
customers
----------------
id | name | city

orders
----------------
id | customer_id | total
```

Relationships are explicit and can be queried with joins:

```sql
SELECT c.name, o.total
FROM customers c
JOIN orders o
  ON c.id = o.customer_id;
```

SQL databases are particularly strong when the application needs structured data, relationships, joins, constraints, and multi-step transactions.

## 2. NoSQL

NoSQL is a broad category rather than one database model.

Common types include:

```text
Document
Key-Value
Wide-column
Graph
```

A document database may store data in document-shaped structures:

```json
{
  "id": 101,
  "customer": "Tanay",
  "items": [
    { "product": "Laptop", "quantity": 1 },
    { "product": "Mouse", "quantity": 2 }
  ],
  "total": 85000
}
```

Instead of always splitting related data across multiple relational tables, the document can keep data needed together for a particular access pattern together.

## 3. Useful mental model

A practical simplification is:

```text
SQL
→ model the data and its relationships

NoSQL
→ often model the data around access patterns
```

This is a simplification, not a universal rule.

With SQL, you naturally consider entities, relationships, constraints, and queries. With many NoSQL systems, you often design the stored representation around how the application will read and write it.

## 4. SQL strengths

Relational databases are especially useful when the domain has:

- strong relationships between entities
- complex queries and joins
- strict data integrity requirements
- foreign keys and other constraints
- multi-step transactional operations
- structured data

Example e-commerce model:

```text
customers
    ↓
orders
    ↓
order_items
    ↓
products
```

This is naturally relational because the relationships and transactional integrity matter.

## 5. NoSQL strengths

NoSQL can be attractive when requirements point toward:

- very large-scale workloads
- specific, predictable access patterns
- flexible or heterogeneous data
- high read/write throughput
- distributed architectures
- data naturally represented as documents, key-value records, etc.

For example, an event-ingestion workload may receive large volumes of events with evolving metadata:

```json
{
  "userId": 123,
  "event": "product_viewed",
  "timestamp": "...",
  "device": {},
  "metadata": {}
}
```

A document-oriented or other NoSQL system may be a good fit depending on the exact workload and requirements.

## 6. NoSQL does not mean "no schema"

Do not say:

> "NoSQL databases don't have schemas."

A better statement is:

> **Many NoSQL databases provide more flexible schemas or rely more heavily on application-level schema enforcement rather than requiring every record to follow one rigid relational structure.**

The application can still have strong expectations about fields and data types.

## 7. Transactions and consistency

Do not say:

> "NoSQL doesn't support transactions."

Transactional capabilities vary significantly between NoSQL databases.

A better interview statement is:

> **Relational databases traditionally provide strong transactional and consistency semantics across related data, while NoSQL systems vary in their transaction scope and make different trade-offs around consistency, distribution, and performance.**

Always discuss the specific NoSQL database when making a claim about its guarantees.

## 8. Horizontal/distributed scaling

Many NoSQL databases were designed from the beginning with large distributed workloads in mind. Partitioning/sharding data across multiple nodes is often a fundamental part of their architecture.

Conceptually:

```text
                NoSQL cluster
             ┌───────┼───────┐
             ↓       ↓       ↓
           Node 1  Node 2  Node 3
             │       │       │
             └───────┼───────┘
                     ↓
              distributed data
```

For example, data may be partitioned across nodes based on a partition/shard key.

The historical reason this became important is that many NoSQL systems targeted very large datasets and high request volumes where adding machines horizontally was an important scaling strategy.

### Important correction

Do **not** memorize:

```text
SQL = vertical scaling
NoSQL = horizontal scaling
```

That is false.

Modern relational databases can also scale horizontally using techniques such as replication, partitioning, sharding, and distributed SQL architectures. Some NoSQL databases can also run on a single machine.

The accurate statement is:

> **Many NoSQL databases were designed with distributed, horizontally scalable workloads as a primary consideration, so partitioning and adding nodes are often fundamental parts of their architecture.**

## 9. NoSQL is not automatically faster

Do not claim:

> "NoSQL is faster than SQL."

Performance depends on:

- data model
- query/access patterns
- indexes
- partitioning
- dataset size
- consistency requirements
- hardware
- workload
- database implementation

A well-designed PostgreSQL database can outperform a poorly designed NoSQL system for a particular workload, and vice versa.

## 10. SQL example: e-commerce

An e-commerce system with:

```text
Customer
   ↓
Order
   ↓
OrderItem
   ↓
Product
```

and requirements such as:

- transactional order/payment operations
- foreign-key relationships
- strict integrity
- reporting and complex queries
- joins

is a natural candidate for PostgreSQL or another relational database.

## 11. NoSQL example: event ingestion

Consider an event system receiving millions of events per minute where events can have evolving metadata and the dominant workload is high-volume ingestion followed by processing/querying according to known access patterns.

A NoSQL system may be a good fit if its particular data model and scaling characteristics match those requirements.

The important point is the **workload**, not the label "NoSQL."

## 12. SQL and NoSQL can coexist

A production system does not have to choose one database for everything.

For example:

```text
Spring Boot application
       │
       ├────────→ PostgreSQL
       │             ↓
       │         orders/payments
       │
       ├────────→ Redis
       │             ↓
       │          cache/sessions
       │
       └────────→ Document DB
                     ↓
                   catalog
```

Using different databases for different workloads is commonly called **polyglot persistence**.

## 13. Choosing between SQL and NoSQL

When asked "Why PostgreSQL instead of MongoDB?", don't answer with a blanket claim that one is better.

Evaluate:

```text
1. What does the data look like?
        ↓
2. What relationships exist?
        ↓
3. What queries/access patterns are required?
        ↓
4. How important are transactions?
        ↓
5. What consistency guarantees are needed?
        ↓
6. What is the scale/workload?
        ↓
7. How will the system partition and scale?
        ↓
8. What operational complexity is acceptable?
```

Then choose the database whose trade-offs fit the requirements.

## Interview Quick Recall

> SQL is relational and strong at relationships, joins, constraints, and transactional business data.

> NoSQL is a broad category containing document, key-value, wide-column, graph, and other database models.

> A useful simplification: SQL often models entities and relationships; many NoSQL systems model data around access patterns.

> NoSQL does not mean "no schema."

> NoSQL does not mean "no transactions"; capabilities vary by database.

> Many NoSQL systems were designed heavily around distributed/horizontal scaling, but SQL databases can also scale horizontally.

> NoSQL is not automatically faster than SQL.

> A system can use both SQL and NoSQL databases when different workloads benefit from different models (polyglot persistence).

> Choose based on data model, relationships, access patterns, transactions, consistency, scale, and operational trade-offs — not on which technology is supposedly "better."
