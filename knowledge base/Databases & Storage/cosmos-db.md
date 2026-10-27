# Azure Cosmos DB

## What It Is
Microsoft's **globally distributed, multi-model** NoSQL database. Single-digit ms latency, 99.999% SLA, automatic horizontal scaling, turnkey multi-region replication.

## APIs (multi-model)
Same engine, different "personality":

- **NoSQL (Core SQL)** — native, JSON documents, SQL-like queries (recommended)
- **MongoDB** — wire-protocol compatible
- **Cassandra** — wire-protocol compatible
- **Gremlin** — graph
- **Table** — Azure Table Storage compatible

## Key Concepts

| Term | Meaning |
|------|---------|
| Account | Top-level resource, region & API choice |
| Database | Logical container of containers |
| Container | Collection of items (≈ table); the unit of scale & throughput |
| Item | A JSON document |
| Partition Key | Property used to shard data across physical partitions |
| Logical partition | All items with the same partition key value |
| Physical partition | Underlying storage shard (managed automatically) |

## Partition Key — The Most Important Choice
- Determines scalability and cost.
- **Good key**: high cardinality, evenly distributed access (e.g., `tenantId`, `userId`, `deviceId`).
- **Bad key**: low cardinality (`country`), monotonic (`timestamp`) → hot partitions.
- Logical partition cap: **20 GB** of data per partition key value.
- Cannot be changed after container creation (migrate via copy).

## Throughput (RU/s)
Everything is measured in **Request Units** (RU). 1 KB point read ≈ 1 RU; query/write costs more.

- **Provisioned**: fixed RU/s on container or database (shared)
- **Autoscale**: scales between 10% and 100% of max
- **Serverless**: pay-per-request, lower limits

`RU/s` is **distributed evenly across physical partitions** — one hot partition can throttle even if total RU is high.

## Consistency Levels (5)
Strongest → weakest:
1. **Strong** — linearizable, single-region or higher latency
2. **Bounded staleness** — lag bounded by K versions or T time
3. **Session** — read-your-own-writes per session (default, sweet spot)
4. **Consistent prefix** — never see out-of-order writes
5. **Eventual** — cheapest, no ordering guarantees

Choose per request if needed; weaker = lower RU + lower latency.

## Change Feed
Persistent, ordered log of changes per partition. Drives event-driven patterns (Functions trigger, materialized views, cache invalidation, ETL). Replayable.

## Indexing
Automatic indexing of all properties by default (great for dev, costly at scale). Tune via indexing policy: include/exclude paths, composite indexes for multi-property ORDER BY.

## When to Use
- Global apps needing multi-region writes
- Predictable low-latency at any scale
- JSON / flexible schema
- Event-driven via change feed

## When NOT to Use
- Heavy relational queries with joins
- Cost-sensitive low-volume apps (RU model can surprise — serverless or SQL DB may be cheaper)
- Need full-text search (pair with Azure AI Search)
