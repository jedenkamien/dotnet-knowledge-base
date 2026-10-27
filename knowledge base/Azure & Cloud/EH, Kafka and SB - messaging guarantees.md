# Kafka, Event Hubs, and Service Bus: Messaging Guarantees

## Core Difference

The primary difference is not reliability but **who tracks successful processing**.

| Technology | Processing State Managed By |
|------------|----------------------------|
| Kafka | Consumer (offsets) |
| Event Hubs | Consumer (checkpoints) |
| Service Bus | Broker (locks and message settlement) |

---

## Kafka and Event Hubs

Kafka and Event Hubs are **durable event streams**.

Characteristics:

- Events are stored durably for a retention period.
- Consumers track their own progress.
- Reading an event does not remove it.
- Multiple consumer groups can read the same event stream.
- Historical replay is a core capability.

### Consumer Failure

If a consumer processes an event but crashes before saving its offset/checkpoint:

- The event will be read again after restart.
- Duplicate processing may occur.
- The event is not lost.

### Important

Incorrect checkpoint/offset management can cause:

- Duplicate processing (commit too late)
- Skipped processing by the application (commit too early)

This is typically an application-level issue, not data loss within the broker.

---

## Service Bus

Service Bus is **work-oriented messaging**.

Typical flow:

1. Receive message
2. Broker locks message
3. Process message
4. Complete message
5. Broker removes message

If processing fails before completion:

- The lock expires.
- The message becomes available again.
- Another consumer can process it.

### Key Property

A message remains in the broker until it is successfully settled.

---

## Delivery Guarantees

### Kafka

Supports:

- At-most-once
- At-least-once
- Exactly-once (via idempotent producers and transactions)

Exactly-once semantics in Kafka do not automatically guarantee exactly-once execution in external systems such as databases or APIs.

### Event Hubs

Provides durable event storage and replay capability.

Reliability depends largely on proper checkpoint management by consumers.

### Service Bus

Provides broker-managed reliability features, including:

- Message locks
- Retries
- Dead-letter queues
- Duplicate detection
- Explicit completion

---

## Retention vs Completion

### Kafka / Event Hubs

Events remain available after consumption.

Data is removed only when retention expires.

> Consumption does not delete data.

### Service Bus

Messages are removed after successful completion.

> Successful processing deletes the message.

---

## When to Use What

### Kafka / Event Hubs

Use for:

- Telemetry
- Application events
- CDC
- Log ingestion
- Event sourcing
- High-throughput streaming

Think:

> Keep and replay event history.

### Service Bus

Use for:

- Business commands
- Order processing
- Workflows
- Enterprise integration
- Task execution

Think:

> Deliver work and verify completion.

---

## Key Takeaway

- Kafka and Event Hubs optimize for scalable event streaming and replay.
- Service Bus optimizes for reliable execution of discrete units of work.
- The biggest architectural distinction is whether processing state is managed by the **consumer** or by the **broker**.