# 07 — Messaging

Messaging enables independently running services to communicate asynchronously through a message broker.

```text
Service A → Message Broker → Service B
```

This is primarily useful for communication between independently running processes/services. Two `@Service` classes inside the same Spring Boot JVM are not separate services; they normally communicate through ordinary method calls rather than Kafka/RabbitMQ.

Messaging can also exist inside a monolith through in-process application events, but that is different from distributed messaging because there is no network/broker boundary.

---

## Why Messaging Exists

Suppose a Payment Service needs to trigger fraud analysis, notifications, audit processing, and analytics.

Synchronous approach:

```text
Payment Service
      ↓
 Fraud Service
      ↓
Notification Service
      ↓
 Audit Service
```

The request becomes tightly coupled to downstream services. A slow or unavailable downstream service can delay or fail the request.

With messaging:

```text
                 Message Broker
                      ↓
Payment Service → PaymentCreated
                      ↓
                /      |      \
               ↓       ↓       ↓
            Fraud   Notification Audit
```

The Payment Service can publish an event and downstream services can process it asynchronously.

### Main benefits

- **Decoupling:** producer does not need to synchronously call the consumer.
- **Asynchronous processing:** downstream work can happen after the producer's request completes.
- **Buffering:** a broker can absorb traffic spikes while consumers process messages at their own rate.

---

## Producer and Consumer

### Producer

The component/service that creates and sends a message.

```text
Payment Service
      ↓
   Producer
      ↓
    Broker
```

### Consumer

The component/service that receives and processes a message.

```text
Broker
  ↓
Consumer
  ↓
Fraud Service
```

Mental model:

```text
Producer → Message Broker / Queue → Consumer
```

---

## Queue as a Buffer

A queue stores messages until consumers can process them.

```text
Producer
   ↓
┌─────────────┐
│    Queue    │
│ M1 M2 M3 M4 │
└──────┬──────┘
       ↓
   Consumer
```

If the consumer is temporarily unavailable:

```text
Producer → Queue → [messages waiting]
                         ↓
                  Consumer returns
                         ↓
                    processes backlog
```

The queue therefore decouples the rate at which work is produced from the rate at which it is consumed.

---

## Messaging vs HTTP

### Synchronous HTTP

```text
Service A ───── HTTP ─────→ Service B
```

Service A directly depends on Service B responding.

### Asynchronous messaging

```text
Service A → Broker → Service B
```

Service A publishes work/event data and Service B can process it later.

Messaging is not automatically better. It introduces additional distributed-system complexity such as retries, duplicate delivery, ordering, broker failures, and message retention.

---

## Real-World Banking Example

A transaction completes:

```text
Transaction Service
        ↓
Transaction DB
        ↓
TransactionCompleted event
        ↓
      Broker
      / | \
     ↓  ↓  ↓
  Fraud Audit Notification
```

Conceptual event:

```json
{
  "event": "TransactionCompleted",
  "transactionId": "TX123",
  "accountId": "ACC42",
  "amount": 5000
}
```

Multiple independent systems can react to the same business event without the Transaction Service making synchronous calls to each one.

---

## Kafka vs RabbitMQ

Both can act as the message broker between independently running services, but their core models differ.

### RabbitMQ — message broker / queues

The useful mental model is:

```text
Producer
   ↓
 Queue
   ↓
Consumer
```

Messages are primarily managed around brokered delivery and acknowledgements.

With multiple consumers, work can be distributed among them:

```text
                Queue
             /    |    \
            ↓     ↓     ↓
          C1     C2     C3
```

RabbitMQ is a natural fit for task/work distribution such as background jobs, email processing, and other "some consumer should process this work" use cases.

### Kafka — distributed durable log

Kafka is better understood as an append-only distributed log:

```text
M1 | M2 | M3 | M4 | M5 | M6
```

Consumers track their position using offsets.

```text
M1 | M2 | M3 | M4 | M5 | M6
          ↑
       offset
```

Messages are retained according to configured retention rather than simply disappearing because one consumer processed them.

Different consumer groups can independently consume the same topic:

```text
              PaymentCreated topic
                       │
              ┌────────┼────────┐
              ↓        ↓        ↓
            Fraud   Analytics   Audit
            group     group     group
```

Each group maintains its own progress.

This makes Kafka particularly useful for high-throughput event streams, multiple independent consumers, durable event retention, and replaying events.

### Important interview nuance

Do not say "RabbitMQ is for queues and Kafka cannot do queues."

Kafka can support queue-like workloads through consumer groups. A stronger comparison is:

> RabbitMQ is primarily centered around brokered message delivery and queues, while Kafka is centered around a durable distributed log and consumer offsets. RabbitMQ is a natural fit for task/work distribution, while Kafka is particularly useful for event streaming, multiple independent consumers, and replay. Both can support asynchronous at-least-once processing.

---

## At-Least-Once Delivery

A consumer can receive the same message more than once.

Example:

```text
Broker
  ↓
Message M1
  ↓
Consumer
  ↓
DB update ✓
  ↓
Consumer crashes before acknowledgement/commit
  ↓
Broker redelivers M1
```

The consumer therefore sees:

```text
M1
M1  ← duplicate
```

**At-least-once delivery** means the system aims to ensure the message is delivered one or more times rather than losing it after an unsuccessful processing attempt.

It does **not** mean exactly once.

### Why duplicates happen

There is an unavoidable failure window between:

```text
message processing succeeds
        ↓
acknowledgement / successful processing state recorded
```

If the consumer dies during that window, the broker may not know that processing succeeded and can redeliver the message.

---

## Idempotent Consumers

At-least-once delivery means consumers must often be designed to tolerate duplicates.

Suppose:

```text
Message: Debit ₹500
```

If processed twice:

```text
₹50,000 → ₹49,500
₹49,500 → ₹49,000   ← incorrect duplicate effect
```

A common approach is to give each message a unique ID:

```text
messageId = MSG123
```

and record processed IDs or otherwise make the business operation idempotent.

```text
Receive MSG123
      ↓
Already processed?
   /          \
 yes           no
  ↓             ↓
ignore      process safely
```

Key connection:

```text
At-least-once delivery
        ↓
Possible duplicate delivery
        ↓
Consumer must handle duplicates
        ↓
Idempotency
```

For banking/payment workflows, this is especially important because duplicate processing can cause duplicate financial effects.

---

## Messaging in a Monolith vs Microservices

### Same Spring Boot process

```text
┌─────────────────────────────┐
│          Monolith           │
│                             │
│ PaymentService              │
│      ↓                      │
│ In-process event            │
│      ↓                      │
│ FraudHandler                │
└─────────────────────────────┘
```

These are objects/components inside one process. Ordinary method calls are usually sufficient. In-process events can be used for loose coupling, but there is no network/broker boundary.

### Independent services

```text
┌─────────────────┐
│ Payment Service │
│    Producer     │
└────────┬────────┘
         │ network
         ↓
┌─────────────────┐
│ Kafka/RabbitMQ  │
│     Broker      │
└────────┬────────┘
         │ network
         ↓
┌─────────────────┐
│ Fraud Service   │
│    Consumer     │
└─────────────────┘
```

This introduces distributed-system concerns such as network failures, retries, duplicate delivery, ordering, broker availability, and delivery guarantees.

### Important distinction

> Messaging does not equal microservices. Messaging is a communication mechanism, but broker-based messaging is especially valuable when components are independently running processes.

---

## Interview Quick Recall

> Messaging = asynchronous communication through a broker between independently running components/services.

> Producer = creates/sends the message.

> Consumer = receives/processes the message.

> Queue = buffers work so production and consumption can happen at different rates.

> Messaging reduces synchronous coupling and can absorb traffic spikes, but introduces distributed-system complexity.

> RabbitMQ = think brokered message delivery and queues; natural fit for task/work distribution.

> Kafka = think durable distributed log, topics, offsets, consumer groups, event streaming and replay.

> Kafka can also provide queue-like work distribution through consumer groups.

> At-least-once = a message can be delivered more than once.

> At-least-once delivery implies consumers often need idempotency.

> Stateless `@Service` classes inside one Spring Boot JVM are not separate services; don't introduce Kafka/RabbitMQ merely to communicate between them.
