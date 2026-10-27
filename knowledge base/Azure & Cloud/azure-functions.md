# Azure Functions

## Purpose
Serverless compute — run small pieces of code triggered by events without managing servers. Pay per execution (Consumption plan) or for reserved capacity (Premium / App Service plan).

## Hosting Plans

| Plan | Scaling | Cold start | When |
|------|---------|------------|------|
| Consumption | Auto, scale-to-zero | Yes | Spiky / low traffic |
| Premium | Pre-warmed instances, VNet | Minimal | Predictable + needs VNet |
| App Service | Manual / VM-based | None | Reuse existing App Service |
| Container Apps | KEDA-based | Some | Container workloads |

## Triggers (entry points)
One per function. Common triggers:

- **HTTP** — REST/webhook endpoints
- **Timer** — CRON schedule
- **Queue / Service Bus / Event Hub** — message-driven
- **Blob Storage** — file uploaded
- **Cosmos DB change feed** — react to document changes
- **Event Grid** — Azure resource events

## Bindings (input/output)
Declarative way to read from / write to other services without SDK boilerplate.

```csharp
[Function("ProcessOrder")]
[BlobOutput("processed/{id}.json")]
public string Run(
    [QueueTrigger("orders")] Order order,
    [CosmosDBInput("db", "users", Id = "{userId}")] User user)
{
    return JsonSerializer.Serialize(new { order, user });
}
```

## Use Cases
- Lightweight HTTP APIs / webhooks
- Background processing (queue consumers)
- Scheduled jobs (cleanup, reports)
- ETL / file processing on upload
- Glue between Azure services

## Limitations
- **Execution timeout**: 5 min default (Consumption), up to 60 min (Premium)
- **Cold starts** on Consumption — first request after idle is slow
- **Stateless** — use Durable Functions for stateful orchestrations
- **No long-running connections** (websockets, etc.) on Consumption

## Durable Functions
Extension that lets you write stateful workflows (orchestrations, fan-out/fan-in, human-in-the-loop, eternal orchestrations) using normal C# `async/await`. State persists in Azure Storage automatically.

## When NOT to Use
- Long-running CPU-bound work
- High-throughput steady load (App Service / AKS often cheaper)
- Latency-critical APIs without Premium pre-warm
