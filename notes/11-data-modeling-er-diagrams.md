# 11 — Data Modeling & ER Diagrams

## What is Data Modeling?

Data modeling is the process of translating a real-world business domain into a structured representation of data and its relationships.

The basic flow is:

```text
Business requirements
        ↓
Identify entities
        ↓
Identify attributes
        ↓
Identify relationships
        ↓
Determine cardinality
        ↓
Choose keys
        ↓
Map to relational tables
        ↓
Add constraints
```

The important idea is to understand the domain **before** writing SQL or Java classes.

---

## Entity

An **entity** is something in the business domain that has its own identity and for which the system needs to maintain data.

Banking example:

```text
Customer
Account
Transaction
```

These are potential entities.

Ask:

> Does this concept need its own identity and does the system need to maintain it independently?

Not every noun in a requirements document needs to become an entity.

---

## Entity vs Attribute

An **attribute** describes an entity.

Example:

```text
Customer
---------
customerId
name
email
dateOfBirth
phone
```

Here:

```text
Customer = Entity

customerId
name
email
dateOfBirth
phone
        ↓
     Attributes
```

Another example:

```text
Account
---------
accountId
accountNumber
type
balance
createdAt
```

`Account` is the entity; the remaining fields are attributes.

A property such as `Customer.email` does not automatically deserve its own entity. It should have an independent entity/table only when the domain requires its own identity, lifecycle, relationships, or other meaningful structure.

---

## Relationships

A **relationship** describes how entities are associated with one another.

Example:

> A customer can have multiple accounts.

```text
Customer 1 ─────── N Account
```

And:

```text
Account 1 ─────── N Transaction
```

So the domain can be visualized as:

```text
Customer
    │
    │ 1:N
    ↓
 Account
    │
    │ 1:N
    ↓
Transaction
```

---

## Cardinality

**Cardinality** describes how many instances of one entity can be associated with another.

The important relationship types are:

```text
1 : 1
1 : N
M : N
```

---

## One-to-One (1:1)

One entity corresponds to at most one entity on the other side.

Example:

```text
Customer 1 ───── 1 CustomerProfile
```

In a relational model, one table will typically contain a foreign key to the other. A `UNIQUE` constraint can enforce that the relationship remains one-to-one.

---

## One-to-Many (1:N)

This is one of the most common relationships in backend systems.

Example:

```text
Customer 1 ─────── N Account
```

One customer can have many accounts, while an individual account belongs to one customer.

```text
Customer
   │
   ├──── Account 1
   ├──── Account 2
   └──── Account 3
```

The relational representation normally puts the foreign key on the **many side**:

```text
customers
---------
id PK
name

accounts
--------
id PK
customer_id FK → customers.id
type
balance
```

Key rule:

> **For a 1:N relationship, the foreign key normally lives on the N side.**

---

## Many-to-Many (M:N)

Example:

> A customer can have multiple accounts, and an account can have multiple customers.

```text
Customer M ─────── N Account
```

A single foreign key in either table cannot represent this correctly.

Instead, introduce an associative/join table:

```text
Customer
    │
    ↓
CustomerAccount
    ↑
    │
 Account
```

Tables:

```text
customers
---------
id PK
name

accounts
--------
id PK
account_number

customer_account
----------------
customer_id FK
account_id  FK
```

A common design is:

```text
PRIMARY KEY (customer_id, account_id)
```

The join table converts the M:N relationship into two 1:N relationships:

```text
Customer 1 ──── * CustomerAccount * ──── 1 Account
```

---

## ER Diagram

An **Entity-Relationship Diagram (ERD)** visually represents entities, their attributes, relationships, and cardinality.

Banking example:

```text
┌─────────────────┐
│    CUSTOMER     │
├─────────────────┤
│ PK id           │
│ name            │
│ email           │
└────────┬────────┘
         │
         │ 1:N
         ↓
┌─────────────────┐
│     ACCOUNT     │
├─────────────────┤
│ PK id           │
│ account_number  │
│ customer_id FK  │
│ type            │
│ balance         │
└────────┬────────┘
         │
         │ 1:N
         ↓
┌─────────────────┐
│   TRANSACTION   │
├─────────────────┤
│ PK id           │
│ account_id FK   │
│ amount          │
│ created_at      │
│ type            │
└─────────────────┘
```

An ERD is a modeling tool. The final relational schema represents the same domain using tables, keys, and constraints.

---

## ER Model → Relational Tables

The important mappings are:

### 1:N

ER model:

```text
Customer 1 ─────── N Account
```

Relational model:

```text
CUSTOMER
---------
id PK
name
email

ACCOUNT
-------
id PK
customer_id FK → CUSTOMER.id
type
balance
```

The relationship becomes a foreign key on the N side.

### M:N

ER model:

```text
Customer M ───── N Account
```

Relational model:

```text
CUSTOMER
   │
   ↓
CUSTOMER_ACCOUNT
   ↑
   │
ACCOUNT
```

with:

```text
CUSTOMER_ACCOUNT
----------------
customer_id FK
account_id FK
```

The M:N relationship becomes an associative/join table.

---

## More Realistic Banking Example

Suppose the requirements say:

- A customer can have multiple accounts.
- An account can perform many transactions.
- Each transaction has an amount, timestamp, and type.
- A transaction may involve another account as the recipient.

Potential entities:

```text
Customer
Account
Transaction
```

Potential attributes:

```text
Customer
---------
id
name
email

Account
-------
id
accountNumber
balance

Transaction
-----------
id
amount
timestamp
type
```

Now a modeling question appears:

> A transfer moves money from one account to another. Should Transaction have one account ID or two?

A reasonable transfer model can use:

```text
Transaction
-----------
id
source_account_id
destination_account_id
amount
timestamp
type
```

The same `Account` entity participates in two relationships, but with different roles:

```text
Account 1 ───── N Transaction
       source

Account 1 ───── N Transaction
       destination
```

This illustrates why data modeling should happen before implementation: the business relationship determines the relational structure.

---

## Database Model vs Java Object Model

The database model and Java object model are related but are not identical.

For example, a Java domain model might contain:

```java
class Customer {
    List<Account> accounts;
}
```

while the database represents the relationship as:

```text
CUSTOMER 1 ───── N ACCOUNT
```

JPA/Hibernate mappings such as `@OneToMany` and `@ManyToOne` map object relationships onto the underlying relational model. The database design should be understood independently of the framework mapping.

---

## Practical Modeling Process

When given a data-modeling interview problem:

```text
1. Understand business requirements
        ↓
2. Identify entities
        ↓
3. Identify attributes
        ↓
4. Identify relationships
        ↓
5. Determine cardinality
        ↓
6. Choose primary keys
        ↓
7. Map relationships to FKs / join tables
        ↓
8. Add constraints
        ↓
9. Consider normalization / denormalization
```

Do not immediately start writing `CREATE TABLE`. First understand the domain.

---

## Common Interview Traps

### Trap 1 — FK on the wrong side

For:

```text
Customer 1:N Account
```

the FK normally belongs on `Account`:

```text
Account.customer_id
```

because each account needs to identify its customer.

### Trap 2 — Single FK for M:N

For:

```text
Student M:N Course
```

don't store a single `course_id` in `Student`.

Use:

```text
student_course
--------------
student_id
course_id
```

### Trap 3 — Making everything an entity

Not every business noun deserves a table. Determine whether it needs independent identity, lifecycle, relationships, or other meaningful structure.

### Trap 4 — Designing tables before understanding the domain

Start with:

```text
What are the entities?
How are they related?
What is the cardinality?
Who owns what?
```

Then map the model to relational tables.

---

## Core Relationship Mappings

```text
1:1
   → FK + usually UNIQUE

1:N
   → FK on the N side

M:N
   → join / associative table
```

---

## Interview Quick Recall

> **Entity:** a business-domain concept with its own identity that the system needs to maintain.

> **Attribute:** a property describing an entity.

> **Relationship:** an association between entities.

> **Cardinality:** describes how many instances can participate in a relationship.

> **ERD:** visual representation of entities, attributes, relationships, and cardinality.

> **1:1:** typically represented with an FK plus a `UNIQUE` constraint.

> **1:N:** FK normally lives on the N side.

> **M:N:** represented using an associative/join table.

> Data modeling should start from business requirements and domain relationships, not from prematurely writing SQL or Java classes.

> Java object relationships and database relationships are related but are not the same model.
