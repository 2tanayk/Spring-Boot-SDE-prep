# 08 — Reliability & Observability

## Idempotency

An operation is **idempotent** if performing it multiple times has the same intended effect as performing it once.

This matters because clients, networks, and message brokers can cause the same logical operation to be retried or delivered more than once.

### Payment example

```text
POST /payments
₹5,000 from A → B
        ↓
Payment succeeds
        ↓
Response is lost
        ↓
Client retries
```

Without protection, the payment could be executed twice.

A common solution is an **idempotency key**:

```http
Idempotency-Key: abc123
```

The server records the result for the logical operation:

```text
abc123 → Payment P1001 → SUCCESS
```

If the same request is retried with `abc123`, the server can return the existing result instead of executing the payment again.

### Important distinction

HTTP method semantics and application-level idempotency are different concepts. A `POST` can be made idempotent at the application level using an idempotency mechanism.

---

## Rate Limiting

**Rate limiting** controls how many requests a client or identity can make over a period.

Example:

```text
100 requests / minute / user
```

Request 101 can receive:

```text
HTTP 429 Too Many Requests
```

### Why it exists

Rate limiting protects:

- application capacity
- downstream services
- databases
- APIs from abuse
- expensive endpoints from runaway clients

Conceptually:

```text
Incoming requests
        ↓
   Rate Limiter
      /     \
     ↓       ↓
 Allowed   Limit exceeded
     ↓       ↓
 Service    429
```

Rate limiting can be enforced at different layers, such as an API gateway or inside an application, depending on the architecture.

---

## Circuit Breaker

A **circuit breaker** prevents a failing dependency from repeatedly consuming resources and potentially causing a cascading failure.

Consider:

```text
Payment Service → Fraud Service
```

Normally:

```text
Payment → Fraud → response
```

If Fraud starts timing out:

```text
Payment → Fraud → waits → timeout
Payment → Fraud → waits → timeout
Payment → Fraud → waits → timeout
...
```

Many requests waiting on an unhealthy dependency can consume threads, connections, and other resources until the calling service is also unhealthy.

The circuit breaker stops this.

### Three states

```text
             normal
               │
               ▼
            CLOSED
               │
        too many failures
               │
               ▼
             OPEN
               │
           wait period
               │
               ▼
          HALF-OPEN
           /       \
       success    failure
         │           │
         ▼           ▼
      CLOSED       OPEN
```

### CLOSED

Normal operation. Requests are allowed through while the breaker observes failures.

If failures exceed a configured threshold, the breaker opens.

### OPEN

The dependency is considered unhealthy. Calls are **short-circuited** rather than sent to the dependency.

```text
Payment
   ↓
[OPEN]
   ✕
   ↓
Fraud
```

Requests fail fast or follow an application-defined fallback instead of waiting for another timeout.

The key purpose is:

> Once a dependency is known to be unhealthy, stop wasting resources calling it.

### HALF-OPEN

After a configured waiting period, the breaker allows a limited number of test requests through.

```text
OPEN
 ↓
wait
 ↓
HALF-OPEN
 ↓
test request
```

If the test succeeds, the dependency is considered recovered and the circuit returns to `CLOSED`.

If it fails, the circuit returns to `OPEN`.

### Real-world example

Suppose the configuration is roughly:

```text
Failure threshold = 5
Open duration     = 30 seconds
```

After repeated failures:

```text
CLOSED
   ↓
5+ failures
   ↓
OPEN
```

For the next 30 seconds, requests fail fast rather than calling Fraud.

Then:

```text
OPEN
 ↓
HALF-OPEN
 ↓
small test request
```

Success → `CLOSED`.
Failure → `OPEN`.

### Circuit breaker vs retry

**Retry** says:

> Try the operation again because the failure may be transient.

```text
Call → failure → retry → failure → retry
```

**Circuit breaker** says:

> The dependency has failed repeatedly; stop calling it temporarily.

```text
Call → failure
Call → failure
Call → failure
      ↓
Circuit OPEN
      ↓
STOP CALLING
```

They can be combined, but retries should be bounded and controlled. Unlimited retries can amplify an existing failure.

### Fallback is a separate decision

The circuit breaker does not decide what the business should do when the dependency is unavailable.

Possible application-level responses include:

```text
reject request
return a pending state
queue work for later
use a safe fallback
```

For financial operations, blindly returning success is generally unsafe; the fallback must respect the business consistency requirements.

### Core insight

> A circuit breaker is primarily a **failure-containment mechanism**. It prevents an unhealthy dependency from dragging healthy parts of the system down with it.

---

## Observability

Observability helps answer what is happening inside a production system and why.

The three core signals are:

```text
Logs + Metrics + Traces
```

## Logs

Logs record individual events and details about what happened.

Example:

```text
2026-08-28 16:10:03
paymentId=P123
Calling Fraud Service
```

Then:

```text
2026-08-28 16:10:11
paymentId=P123
Fraud Service timeout
```

Logs are useful for reconstructing the details of a specific event/request.

Production logs should include useful context such as request/correlation identifiers and business identifiers where appropriate.

In a banking environment, do not casually log sensitive data such as credentials, tokens, or protected financial information.

## Metrics

Metrics are aggregated numerical measurements of system behavior.

Examples:

```text
requests/sec = 2,000
error rate   = 1.2%
p95 latency  = 450ms
p99 latency  = 1.2s
queue depth  = 10,000
```

Metrics answer:

> What is the system doing overall?

Useful metrics include request rate, error rate, latency, CPU/memory usage, queue depth, and database connection-pool usage.

## Traces

A distributed trace follows an individual request across services and shows where time was spent.

Example:

```text
Client
  ↓
API
  ↓
Payment Service
  ↓
PostgreSQL
  ↓
Fraud Service
  ↓
Response
```

A trace might reveal:

```text
API              0ms ───────────── 3200ms
Payment Service  20ms ──────────── 3150ms
PostgreSQL       70ms ─ 120ms
Fraud Service   150ms ─────────── 3100ms
```

This quickly identifies Fraud Service as the bottleneck.

## Logs vs Metrics vs Traces

| Signal | Main question |
|---|---|
| **Logs** | What happened? |
| **Metrics** | How is the system behaving overall? |
| **Traces** | Where did this request go and spend time? |

They complement each other.

For example:

```text
Metrics → p99 latency increased
Traces  → Fraud Service calls take 5 seconds
Logs    → Fraud requests are timing out
```

Together they provide detection, localization, and detailed diagnosis.

---

## Putting Reliability Together

A production payment flow may use several mechanisms together:

```text
Client
  ↓
Rate Limiter
  ↓
Payment Service
  │
  ├── Idempotency protection
  │
  └── Circuit Breaker
            ↓
       Fraud Service
```

And observability surrounds the system:

```text
Metrics → detect abnormal behavior
Traces  → locate the slow/failing dependency
Logs   → understand the detailed failure
```

The mechanisms solve different problems:

```text
Idempotency     → correctness under retries/duplicates
Rate limiting   → capacity protection
Circuit breaker → failure containment
Observability   → detection + diagnosis
```

---

## Interview Quick Recall

> Idempotency = repeated execution of the same logical operation has the same intended effect as executing it once.

> Idempotency keys allow a server to recognize retries of the same logical operation.

> Rate limiting protects service and downstream capacity by restricting request volume.

> Circuit breaker states: `CLOSED → OPEN → HALF-OPEN → CLOSED` (or back to `OPEN` on failed recovery test).

> `OPEN` means calls to the unhealthy dependency are short-circuited and fail fast or use an application-defined fallback.

> Retry and circuit breaking are different: retry makes another attempt; circuit breaking stops calls after repeated failures.

> Circuit breakers primarily provide **failure containment** and help prevent cascading failures.

> Logs = detailed events.

> Metrics = aggregated numerical behavior.

> Traces = end-to-end path/timing of an individual request.
