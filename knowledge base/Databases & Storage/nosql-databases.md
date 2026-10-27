# NoSQL Databases

## Definition
"Not Only SQL" — data stores that diverge from the relational model. Typically trade strict consistency / joins / schemas for **scalability, flexibility, and performance**.

## Categories

| Type | Example (AWS / Azure) | Data Shape | Best For |
|------|----------------------|------------|----------|
| Document | MongoDB, Cosmos DB, Couchbase | JSON-like documents | Flexible schemas, content, catalogs |
| Key-Value | DynamoDB, Cosmos Table | Key → opaque value | Primary data stores, sessions, counters |
| In-memory cache | ElastiCache (Redis) / Azure Cache for Redis | Key → value in RAM | Caching, sub-ms lookups |
| Column-family | Cassandra, HBase, Bigtable | Sparse rows, wide columns | Time-series, write-heavy |
| Graph | Neo4j, Cosmos Gremlin | Nodes + edges | Relationships, social, fraud |
| Search | Elasticsearch, OpenSearch | Inverted indexes | Full-text search, analytics |

> **DynamoDB ≠ Redis.**  
> DynamoDB (≈ Cosmos DB) is a **persistent NoSQL database** — the primary data store.  
> Redis (ElastiCache / Azure Cache for Redis) is an **in-memory cache** — a speed layer, not a main DB.  
> Rule: DynamoDB = *where you store data*; Redis = *where you cache data for speed*.

## Why NoSQL = High Scale

"High scale" means handling massive data + traffic (millions of rq/s) by adding machines, not upgrading one big server.

**Why it scales better than SQL:**
- **Horizontal scaling built-in** — add nodes; data is partitioned/sharded by key (e.g., `userId`)
- **No complex joins** → easier to distribute — each node can answer a query independently
- SQL struggles here because joins across distributed nodes are expensive, and strong consistency + ACID transactions limit how far you can spread data

## Why NoSQL = Low Latency

"Low latency" means faster responses — typically single-digit ms or less.

**Why it's faster:**
- **Simple access patterns** — key lookup is O(1); no joins, no heavy query planning
- **Data is pre-shaped** — stored exactly as it will be read (denormalized); avoids runtime computation
- **Distributed + local reads** — request hits the node that owns the data
- **Weaker constraints (trade-off)** — often relaxes strict ACID → less overhead per write

| | SQL | NoSQL |
|---|---|---|
| Query flexibility | High (ad-hoc joins) | Low (fixed access patterns) |
| Consistency | Strong (ACID) | Often eventual |
| Scaling | Vertical (harder) | Horizontal (easy) |
| Latency (simple reads) | Higher | Lower |

**One-liner:** NoSQL is fast and scalable because it simplifies the problem — no joins + predictable access + distributed by design. It trades query flexibility for speed and scale.

## Trade-offs (CAP Theorem)

In any **distributed** data system, a network partition (nodes losing contact) will eventually happen. When it does, you must choose:

| Letter | Property | Meaning |
|--------|----------|---------|
| **C** | Consistency | Every read gets the most recent write (no stale data) |
| **A** | Availability | Every request gets a response (no timeouts/errors) |
| **P** | Partition tolerance | System keeps working even when nodes can't communicate |

**The theorem:** You can guarantee at most **two** of the three simultaneously. But since network partitions are unavoidable in distributed systems, **P is always required** in practice — meaning the real choice is between **C** and **A** when a partition occurs.

```
          C
         / \
        /   \
       CP   CA
      /       \
     AP ----  (impossible to have all 3 during a partition)
```

| Choice | Who picks it | Behaviour during partition |
|--------|-------------|---------------------------|
| **CP** (Consistency + Partition) | HBase, MongoDB (strong mode), Zookeeper | Returns an error rather than stale data |
| **AP** (Availability + Partition) | Cassandra, DynamoDB, CouchDB | Returns possibly stale data rather than an error |
| **CA** (Consistency + Availability) | Traditional single-node RDBMS (PostgreSQL, SQL Server) | Only possible without distribution — no real partition tolerance |

**NoSQL typically picks AP** — eventual consistency — for maximum scale and availability.  
**SQL picks CA** on a single node, or CP in distributed form.

Most modern NoSQL (Cosmos DB, DynamoDB) exposes **tunable consistency** — you choose per-request: strong, bounded staleness, session, or eventual.

## When NOT to Use NoSQL (choose SQL instead)
- **Complex joins** across many entities — SQL's killer feature
- **Strong ACID transactions** across aggregates (e.g., bank transfers across tables)
- **Strict schema + referential integrity**
- **Ad-hoc reporting / BI** tools that assume SQL
- When data relationships are core, not access patterns

### Example: when a complex join makes SQL the right choice

E-commerce order history: show user + orders + products + categories + payment status.

```sql
SELECT u.name, o.order_id, p.product_name, c.category_name, pay.status
FROM Users u
JOIN Orders o        ON u.id = o.user_id
JOIN OrderItems oi   ON o.id = oi.order_id
JOIN Products p      ON oi.product_id = p.id
JOIN Categories c    ON p.category_id = c.id
JOIN Payments pay    ON o.id = pay.order_id
WHERE u.id = '123';
```

In NoSQL you'd need heavy duplication, and consistency breaks when a product name changes.  
**Rule:** if your query naturally says "join A with B with C", you almost certainly want SQL.

## Real-Life Use Cases for NoSQL

| Scenario | Why NoSQL |
|----------|-----------|
| High-traffic apps (social, gaming, IoT) | Massive scale, simple access by userId/deviceId |
| Flexible / evolving schemas (profiles, catalogs) | Add fields without migrations |
| Low-latency requirements (carts, sessions, recommendations) | Key-based lookups, no joins |
| Event-driven / microservices | Each service stores data shaped for its needs |
| Big data / logs / telemetry | Huge write volume, append-heavy |

## Modeling Difference
Relational: **normalize, then join**.  
Document/NoSQL: **model around access patterns** — embed what you read together, denormalize when needed, accept duplication.

```json
// Relational: User + Orders + Items in 3 tables
// Document: one User document with embedded recent orders
{
  "userId": "u1",
  "name": "Alice",
  "recentOrders": [
    { "orderId": "o1", "total": 99.0, "items": [...] }
  ]
}
```

SQL: "Combine many related pieces dynamically"  
NoSQL: "Fetch pre-assembled data quickly"

## Quick Picker
- Need flexible JSON, queryable → **Document** (MongoDB, Cosmos DB)
- Need persistent key-based store → **Key-Value** (DynamoDB, Cosmos Table)
- Need sub-ms cache layer → **In-memory** (Redis)
- Massive writes, time-series → **Column-family** (Cassandra)
- "Friends of friends" type queries → **Graph** (Neo4j)
- Free-text search → **Search index** (Elasticsearch)
