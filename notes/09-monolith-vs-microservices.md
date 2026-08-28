# 09 — Monolith vs Microservices

## Monolith

A **monolith** is an application where major business capabilities are deployed as one application unit.

Example:

```text
                 Banking Application
┌─────────────────────────────────────────────┐
│                                             │
│  User Management                            │
│  Account Management                         │
│  Payments                                   │
│  Fraud                                      │
│  Notifications                              │
│                                             │
└──────────────────────┬──────────────────────┘
                       │
                  PostgreSQL
```

The code can still be well structured into modules/packages:

```text
com.bank
 ├── users
 ├── accounts
 ├── payments
 ├── fraud
 └── notifications
```

They are nevertheless part of one deployable application.

### Important

**Monolith does not mean badly designed or one giant class.** A monolith can be well-structured and modular.

---

## Microservices

With microservices, the system is split into **independently deployable services**, generally around business capabilities.

```text
Payment Service       Fraud Service
      │                    │
      └────── network ─────┘

Account Service       Notification Service
```

Each service is its own running application and can be deployed independently.

The important distinction is not simply "many small services":

> A microservice is an independently deployable service with ownership of a particular business capability.

---

## Database Ownership

A monolith commonly has multiple business modules using a shared database:

```text
              Application
             /     |     \
            ↓      ↓      ↓
        Payments Accounts Fraud
             \      |      /
              PostgreSQL
```

With microservices, a strong architectural principle is **service-owned data**:

```text
Payment Service  → Payment DB
Account Service  → Account DB
Fraud Service    → Fraud DB
```

A Payment Service should not directly query the Account Service's database. It should interact through an API or an event/message-based mechanism.

The important boundary is **data ownership**, not necessarily that every service must have a physically separate database server. Multiple logical databases/schemas can live on the same database infrastructure while maintaining ownership boundaries.

---

## Why Microservices?

### Independent deployment

A change to Fraud can potentially be deployed without redeploying Payments or Accounts.

```text
Fraud changed
    ↓
Deploy Fraud Service
```

### Independent scaling

If Payments has much higher traffic than Notifications:

```text
Payment Service       × 20
Account Service       × 5
Notification Service  × 2
```

You scale each service according to its workload.

A monolith can still be horizontally scaled; microservices provide **independent** scaling rather than magically enabling scalability.

### Fault isolation

A well-designed microservice architecture can limit the blast radius of a failure:

```text
Notification Service 💀

Payment Service ✓
Account Service  ✓
Fraud Service    ✓
```

This is not automatic. Poor synchronous dependencies can still cause cascading failures, which is why timeouts, retries, circuit breakers, messaging, and observability matter.

### Organizational ownership

Services can align with business and team boundaries, allowing different teams to own and release different capabilities independently.

---

## The Cost of Microservices

Microservices turn many simple in-process interactions into distributed-system interactions.

Monolith:

```text
PaymentService → FraudService
```

may simply be a method call.

Microservices:

```text
Payment Service
      ↓ network
Fraud Service
```

now introduces concerns such as:

- network latency
- network failures
- timeouts
- retries
- duplicate requests
- service discovery/load balancing
- distributed tracing
- version compatibility
- partial failures

Therefore microservices are not automatically superior. They trade local simplicity for independent deployment/scaling/ownership and distributed-system complexity.

---

## Transactions Become Harder

In a monolith, a transfer might be handled by one local database transaction:

```text
BEGIN

debit A
credit B
record transaction

COMMIT
```

Everything can succeed or roll back together.

With microservices:

```text
Account Service → Account DB
Transaction Service → Transaction DB
```

A single simple local transaction no longer naturally spans both services.

Cross-service consistency and transactions therefore become significantly harder and may require distributed-systems patterns such as events or sagas.

The key SDE-2 takeaway is:

> Microservice boundaries make distributed transactions and strong cross-service consistency harder.

---

## Modular Monolith

The choice is not simply:

```text
Monolith OR dozens of microservices
```

A **modular monolith** keeps one deployable application while maintaining strong internal boundaries:

```text
              ONE APPLICATION
┌─────────────────────────────────────┐
│ Payments │ Accounts │ Fraud │ Users │
└─────────────────────────────────────┘
```

Modules should interact through clear interfaces rather than reaching into each other's internals.

You still have:

```text
ONE PROCESS
ONE DEPLOYMENT
```

but gain much of the organizational/domain structure without immediately paying the distributed-system cost.

A system can start as a modular monolith and later extract a module into a service when there is a real reason to do so.

---

## When Microservices Make Sense

Good reasons include:

- different scaling requirements
- independent deployment requirements
- strong business/domain boundaries
- independent team ownership
- capabilities with substantially different availability or operational requirements

Architecture should support **business and organizational boundaries**, not just technical decomposition.

---

## When Microservices Don't Make Sense

For a small team, simple product, low traffic, or simple domain, splitting the application into many services may add significant complexity without meaningful benefit.

You may end up managing:

```text
many deployments
network calls
service discovery
monitoring
distributed debugging
data consistency problems
```

A modular monolith can be the simpler and more appropriate architecture.

---

## Common Interview Traps

### "Microservices are always more scalable"

False. A monolith can be horizontally scaled. Microservices provide **independent scaling**.

### "Microservices means separate physical databases"

Too simplistic. The important principle is service-owned data and controlled access boundaries.

### "Microservices are just small services"

Incomplete. Independent deployment and business-capability ownership are more important than arbitrary service size.

### "Microservices automatically isolate failures"

False. Poor synchronous dependencies can still create cascading failures.

### "Monolith means bad architecture"

False. A well-designed modular monolith can be a strong architecture.

---

## Interview Quick Recall

> **Monolith:** major business capabilities are deployed as one application unit.

> **Microservices:** independently deployable services generally organized around business capabilities.

> **Service-owned data:** a service should own its data rather than allowing other services to directly access its database.

> Microservices enable independent deployment, scaling, and team ownership, but introduce distributed-system complexity.

> A monolith can still scale horizontally; microservices mainly provide independent scaling.

> Microservices make cross-service transactions and strong consistency harder.

> A modular monolith provides strong internal boundaries while retaining one process/deployment.

> Don't choose microservices merely because they sound more scalable; choose them when their independence provides a real business, organizational, or operational benefit.

---

## Architecture Connection

The preceding topics now fit together:

```text
Monolith
   ↓
in-process calls
   ↓
Microservices
   ↓
network calls
   ↓
distributed failures
   ↓
┌─────────────┬─────────────┬──────────────┐
│  Timeouts   │   Retries   │Circuit Breaker│
└─────────────┴─────────────┴──────────────┘
   ↓
Messaging / Events
   ↓
Distributed consistency concerns
   ↓
Observability
```

The move from a monolith to microservices is therefore not merely a code-organization decision; it changes the system's failure modes, communication model, deployment model, data ownership, and consistency trade-offs.
