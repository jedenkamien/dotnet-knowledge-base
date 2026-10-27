# Azure Service Bus

## What It Is
Enterprise message broker — **queues** (point-to-point) and **topics/subscriptions** (publish/subscribe). Designed for reliable, transactional, ordered messaging between decoupled services.

## Protocol
- **AMQP 1.0** (primary, recommended) — binary, efficient, full feature set.
- **HTTPS / REST** — fallback when AMQP ports (5671) blocked.
- **SBMP** (legacy, .NET only) — deprecated.

## Core Concepts

| Concept | Description |
|---------|-------------|
| Queue | FIFO, single consumer per message |
| Topic | Publishes to multiple subscriptions |
| Subscription | Filtered virtual queue under a topic |
| Dead-letter queue (DLQ) | Holds messages that fail processing |
| Peek-Lock | Receiver locks message; explicit Complete/Abandon |
| Receive-and-Delete | Faster, but message lost on crash |

## Maintaining Ordering

Service Bus guarantees order **only within a session**. To preserve order:

1. Enable **Sessions** on the queue/subscription (`RequiresSession = true`).
2. Set the same `SessionId` on all related messages (e.g., `OrderId`).
3. A single consumer receives all messages for a session, in order.

```csharp
await sender.SendMessageAsync(new ServiceBusMessage(body) {
    SessionId = orderId.ToString()
});

// Consumer
var receiver = await client.AcceptNextSessionAsync(queueName);
ServiceBusReceivedMessage msg = await receiver.ReceiveMessageAsync();
```

Without sessions, parallel consumers and retries can interleave.

## Delivery Guarantees
- **At-least-once** by default (peek-lock + Complete).
- Combine with **idempotent handlers** to tolerate duplicates.
- **Duplicate detection** window dedups by `MessageId`.

## Other Features
- Scheduled messages
- Auto-forward between entities
- Transactions across multiple sends/receives
- Message TTL, DLQ on expiry
- Up to 100 MB messages on Premium tier

## When to Use
- Workflow / business process integration
- Order-sensitive flows (finance, billing)
- Need transactions, sessions, DLQ
- Lower throughput than Event Hubs but richer semantics
