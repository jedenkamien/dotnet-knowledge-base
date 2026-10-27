# Database Partitioning Strategies

## Why Partition
- **Scale** beyond what one node can hold or serve
- **Performance** — smaller indexes, parallel queries
- **Availability** — failure of one shard doesn't take down everything
- **Cost** — store hot vs cold data differently

## Vertical vs Horizontal

**Vertical partitioning** (split columns)
- Move rarely-used / large columns to a separate table
- Example: `User` (id, name, email) + `UserProfile` (id, bio, avatar_blob)

**Horizontal partitioning / Sharding** (split rows)
- Same schema, rows distributed across nodes/files
- The default meaning of "partitioning" in distributed systems

## Sharding Strategies

### 1. Range Partitioning
Rows assigned to shards by value range (e.g., `A-M` → shard 1, `N-Z` → shard 2; or by date).

- ✅ Efficient range queries (`WHERE date BETWEEN ...`)
- ❌ Hot spots if data is skewed or monotonic (most recent date = all writes to one shard)

### 2. Hash Partitioning
`shard = hash(key) % N`

- ✅ Even distribution, no hot spots
- ❌ Range queries scan all shards
- ❌ Resharding when N changes is expensive (mitigated by **consistent hashing**)

### 3. Consistent Hashing
Map keys and nodes onto a ring; key goes to next node clockwise. Adding/removing a node only reshuffles ~1/N of keys. Used by Cassandra, DynamoDB, Riak.

### 4. Directory / Lookup-based
A separate service maps key → shard. Maximum flexibility (rebalance freely) at cost of an extra hop and a single point of complexity.

### 5. Geo / Tenant Partitioning
Shard by geography (`region = EU` → EU cluster) or by `tenantId` for multi-tenant SaaS. Keeps related data co-located, simplifies compliance (GDPR data residency).

## Choosing a Partition Key
Good keys are:
- **High cardinality** — many distinct values
- **Evenly accessed** — no single key dominates traffic
- **Aligned with queries** — most queries can be answered from one partition
- **Stable** — value doesn't change after insert

Common picks: `tenantId`, `userId`, `deviceId`, `customerId + month` (composite).

## Hot Partitions — Symptoms & Fixes
- One shard at 100% CPU while others idle
- Throttling errors despite low total throughput

Fixes:
- Add a **suffix / synthetic key** (`userId_0..userId_9`) to spread writes
- Use **composite key** (`userId#date`)
- Cache reads
- Pre-aggregate or denormalize

## Trade-offs to Know
- Cross-shard joins / transactions are expensive or impossible
- Aggregations require fan-out + merge
- Resharding live systems is hard — design partition key carefully *up front*
- Secondary indexes may be **local** (per shard) or **global** (extra coordination)

## In Practice
- **SQL Server**: partitioned tables, partition functions/schemes (single instance)
- **PostgreSQL**: declarative partitioning + Citus for sharding
- **Cosmos DB / DynamoDB**: hash on partition key, automatic
- **Cassandra**: partition key + clustering key, consistent hashing
- **MongoDB**: shard key, range or hashed
- **Kafka / Event Hubs**: partition by key for ordering within a partition
