# 06 — Distributed Systems Basics

## Why Distributed Systems?

A distributed system uses multiple independent machines/processes that cooperate to provide a service.

A simple backend can start as:

```text
Request → Application → Database
```

At larger scale, we may have:

```text
                 Load Balancer
                /      |      \
               ↓       ↓       ↓
            App 1    App 2    App 3
               \       |       /
                \      |      /
                 └── Database
```

This enables scaling and resilience, but introduces problems around shared state, communication failures, and consistency.

---

## Stateless Services

A **stateless service** does not depend on request/session state stored exclusively inside a particular application instance.

For example, with JWT authentication:

```text
Request 1 → App 1
Request 2 → App 3
```

If the JWT contains the information needed for authentication and can be independently validated, App 3 does not need App 1's local memory.

### Important clarification

Stateless does **not** mean the system has no state.

Shared state can live in external systems:

```text
App 1 ─┐
App 2 ─┼──→ PostgreSQL
App 3 ─┘

App 1 ─┐
App 2 ─┼──→ Redis
App 3 ─┘
```

The important point is that shared application state is not trapped inside one instance.

### Why statelessness helps

Any instance can handle any request:

```text
Request 1 → App 1
Request 2 → App 3
Request 3 → App 2
```

This makes horizontal scaling and load balancing much easier.

---

## Horizontal vs Vertical Scaling

### Vertical scaling

Increase the capacity of one machine:

```text
4 CPU / 8 GB RAM
        ↓
32 CPU / 128 GB RAM
```

### Horizontal scaling

Add more machines/instances:

```text
Before:
┌─────────┐
│ Server  │
└─────────┘

After:
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Server 1│  │ Server 2│  │ Server 3│
└─────────┘  └─────────┘  └─────────┘
```

For backend services, horizontal scaling allows capacity to be increased by adding application instances behind a load balancer.

### Real-world example

If one instance handles roughly 500 requests/sec and traffic reaches 5,000 requests/sec, the service may scale to roughly 10 instances, subject to the actual workload and headroom required.

---

## Consistency

Distributed data can temporarily exist in different states across replicas.

Example:

```text
Primary   = ₹1000
Replica 1 = ₹1000
Replica 2 = ₹1000
```

After a ₹500 withdrawal, replication may temporarily lag:

```text
Primary   = ₹500
Replica 1 = ₹1000  ← stale
Replica 2 = ₹1000  ← stale
```

A read from a stale replica can therefore return an older value.

---

## Strong Consistency

With **strong consistency**, after a successful write, subsequent reads observe the latest committed state according to the system's consistency guarantee.

Conceptually:

```text
WRITE ₹500
    ↓
write succeeds
    ↓
READ
    ↓
₹500
```

Strong consistency is important when stale data can cause serious business consequences, such as critical financial or authorization decisions.

---

## Eventual Consistency

With **eventual consistency**, replicas may temporarily disagree, but assuming updates stop and the system continues operating, they eventually converge.

```text
Before:
Primary   = ₹1000
Replica   = ₹1000

Withdrawal:
Primary   = ₹500
Replica   = ₹1000  ← temporarily stale

Later:
Primary   = ₹500
Replica   = ₹500
```

Eventual consistency can be appropriate when brief staleness is acceptable, such as some analytics, counters, search indexes, or recommendation data.

### Trade-off

```text
Strong consistency
→ more coordination / potentially higher latency
→ fresher reads

Eventual consistency
→ less coordination / potentially better availability and performance
→ temporary stale reads are acceptable
```

The correct choice depends on the business requirement.

---

## CAP Theorem

CAP states that **when a network partition occurs**, a distributed data system cannot simultaneously guarantee both:

- **Consistency (C)**
- **Availability (A)**

while also tolerating the partition (**P**).

### Consistency (C)

Clients see a consistent/latest value according to the system's consistency guarantee.

### Availability (A)

Every request to a non-failing node receives a response. Availability does **not** necessarily mean the response contains the latest data.

### Partition Tolerance (P)

The system continues operating despite communication failure between distributed nodes.

A partition can happen because of network link failures, router/switch failures, availability-zone connectivity problems, firewall/configuration errors, severe network congestion, or other communication failures.

Example:

```text
Node A  ←──── network ────→  Node B

             ❌

Node A  ←──── X ──────────→  Node B
```

Both nodes can still be running; they simply cannot reliably communicate.

---

## Why C and A Conflict During a Partition

Suppose two replicas initially contain:

```text
Node A = ₹1000
Node B = ₹1000
```

A partition occurs:

```text
Node A  XXXXXXXXX  Node B
```

A write reaches Node A:

```text
Node A = ₹500
Node B = ₹1000
```

Node B cannot learn about the write because communication is unavailable.

### Preserve consistency

Node B can refuse/delay a read because it cannot guarantee that its value is current.

```text
Consistency = preserved
Availability = sacrificed
```

### Preserve availability

Node B can return its local value:

```text
₹1000
```

The request receives a response, but the value is stale.

```text
Availability = preserved
Consistency = sacrificed
```

Therefore, during a partition:

```text
              PARTITION
                  ↓
          ┌───────┴───────┐
          ↓               ↓
     preserve C       preserve A
          ↓               ↓
       reject         return data
```

### Interview trap

Avoid saying simply:

> CAP means choose any two of C, A and P.

A better statement is:

> When a network partition occurs, a distributed system that must tolerate that partition has to trade off consistency and availability.

Partition tolerance is effectively required if the system is expected to continue operating despite network failures.

---

## Statelessness Does Not Eliminate CAP

A common misconception is:

> "If my application servers are stateless and all state is in Redis/PostgreSQL, CAP doesn't apply."

Statelessness solves a different problem: application instances do not depend on state trapped in one instance.

Consider:

```text
             Stateless Apps
          /       |       \
         ↓        ↓        ↓
       App 1    App 2    App 3
          \       |       /
           \      |      /
              Redis
```

If App 2 cannot communicate with Redis during a network partition, it must choose what to do:

```text
Preserve consistency
→ refuse/delay the request
→ availability sacrificed

Preserve availability
→ answer using stale/local information if possible
→ consistency sacrificed
```

The CAP problem has simply moved to the **shared state layer**.

If Redis/PostgreSQL itself is replicated, the same issue can occur between its nodes.

### Key distinction

```text
Statelessness
    ↓
Any app instance can handle a request
    ↓
Easy horizontal scaling

CAP
    ↓
Distributed state/data nodes
    ↓
What happens when they cannot communicate?
    ↓
Consistency ↔ Availability
```

If the only partition is between stateless application servers, CAP is not the interesting issue because they do not need to share local state. The distributed state system underneath them is where CAP becomes relevant.

---

## Real-World Backend Example

A production backend might look like:

```text
                     Load Balancer
                  /       |       \
                 ↓        ↓        ↓
               App 1    App 2    App 3
                 \        |        /
                  \       |       /
                   ┌────────────┐
                   │ PostgreSQL │
                   └────────────┘
                        │
                      Redis
```

- **Horizontal scaling:** multiple application instances handle traffic.
- **Stateless services:** any instance can handle a request.
- **Shared state:** PostgreSQL and Redis hold state accessible by multiple instances.
- **Consistency choice:** decide whether a particular use case requires strong or eventual consistency.
- **CAP:** if distributed state nodes experience a network partition, consistency and availability cannot both be guaranteed simultaneously while tolerating that partition.

---

## Interview Quick Recall

> Horizontal scaling = add more machines/instances.

> Vertical scaling = make an existing machine bigger.

> Stateless service = an instance does not depend on request/session state stored exclusively in its own local memory.

> Stateless does not mean there is no state; shared state can live in systems such as PostgreSQL or Redis.

> Strong consistency = reads observe the latest committed state according to the system's consistency guarantee.

> Eventual consistency = replicas may temporarily disagree but converge if updates stop and the system continues operating.

> CAP applies to distributed data systems when a network partition occurs.

> During a partition, preserving consistency can require sacrificing availability, while preserving availability can require serving stale/inconsistent data.

> Stateless application servers do not eliminate CAP; distributed shared state can still experience partitions.
