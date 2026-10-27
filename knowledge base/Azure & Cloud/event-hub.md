# Azure Event Hubs

## What It Is
High-throughput **event streaming** platform (millions of events/sec). Think "Kafka as a service." Events are retained for a window (1–7 days, longer with capture) and consumed by multiple readers independently.

## Protocol
- **AMQP 1.0** (primary)
- **HTTPS** (send-only fallback)
- **Apache Kafka protocol** (Event Hubs is Kafka-compatible — clients use Kafka libs unchanged)

## Service Bus vs Event Hubs

| Feature | Service Bus | Event Hubs |
|---------|-------------|------------|
| Model | Message broker (queue/topic) | Event stream / log |
| Throughput | Thousands msg/s | Millions events/s |
| Retention | Until consumed | Time-based (replayable) |
| Consumer state | Broker tracks | Consumer tracks offset/checkpoint |
| Ordering | Per session | Per partition |
| Use case | Commands, workflows | Telemetry, logs, IoT, analytics |

## Partitions
- Event Hub is split into **partitions** (1–32+ depending on tier).
- Each partition is an ordered, append-only log.
- Throughput scales by partition count (set at creation, not changeable easily).
- Order is guaranteed **within a partition only**.

## Partition Key
- Producer can supply a `PartitionKey` (e.g., DeviceId).
- Hashed → consistent partition assignment → ordered events per device.
- Without a key, events are round-robined across partitions.

## Consumer Groups
- A "view" of the stream. Each consumer group has its own offset/cursor.
- Multiple apps consume the same stream independently (e.g., real-time dashboard + archival).
- Within a group, partitions are leased to consumers (one consumer per partition recommended).

## Checkpointing
Consumer stores the last successfully processed offset (typically in Azure Blob Storage via `EventProcessorClient`). On restart/rebalance, consumption resumes from there.

```csharp
var processor = new EventProcessorClient(blobContainer, consumerGroup, connStr, hubName);
processor.ProcessEventAsync += async args => {
    await Handle(args.Data);
    await args.UpdateCheckpointAsync();
};
await processor.StartProcessingAsync();
```

## Capture
Optional auto-archival of events to Blob Storage / Data Lake (Avro/Parquet) for batch analytics.

## When to Use
- Telemetry, metrics, IoT, click streams, application logs
- Need replay / multiple independent consumers
- High-volume ingestion feeding analytics, Stream Analytics, Spark, Databricks
