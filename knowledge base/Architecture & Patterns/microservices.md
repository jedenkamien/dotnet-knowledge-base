# Microservices

## Definition
Architectural style where an application is composed of **small, independently deployable services**, each owning a specific business capability and its own data, communicating over the network.

## Why (Goals)
- **Independent deployability** — ship one service without redeploying the world
- **Independent scaling** — scale only the hot services
- **Team autonomy** — small teams own services end-to-end
- **Technology diversity** — pick the right stack per service
- **Fault isolation** — one crash doesn't take down the system

## Why NOT (Costs)
- **Distributed system complexity** — network, partial failures, latency
- **Operational overhead** — many services to deploy, monitor, secure
- **Data consistency** — no cross-service transactions; eventual consistency
- **Versioning** of contracts between services
- **Local dev** is harder

> Rule of thumb: don't start with microservices. Start with a well-modularized monolith and split when bottlenecks become real.

## Sizing — How "Micro"?
Not lines of code. A service should:
- Own a single **bounded context** (DDD)
- Be replaceable in a few weeks
- Be owned by **one team**
- Have **its own database** (no shared schemas)

## Communication

| Style | When |
|-------|------|
| Sync REST/gRPC | Query, request/response, low latency |
| Async messaging (queues, events) | Workflows, decoupling, resilience |
| Event streaming (Kafka/Event Hubs) | Many consumers, replay, analytics |

Prefer **async** for write paths between services to avoid tight coupling and cascading failures.

## Data Ownership
- **Database per service** — no foreign keys across services
- Cross-service queries via **API composition** or **CQRS read models**
- Cross-service consistency via **Saga pattern** (choreography or orchestration)
- Outbox pattern to atomically persist state + emit event

## Scaling
- Stateless services + load balancer → horizontal scale
- Scale per-service based on its own load profile
- Auto-scale on CPU/RPS/queue depth
- **Cache** at edges to reduce downstream load
- **Bulkheads + circuit breakers** to contain failures (Polly, Resilience4j)

## Operating Microservices
You need a real platform, not just code:

- **Containers + orchestration** (Kubernetes, Azure Container Apps)
- **Service discovery** & load balancing
- **API gateway** (auth, routing, rate limiting)
- **Centralized logging** (ELK, Seq)
- **Distributed tracing** (OpenTelemetry, Application Insights)
- **Metrics & alerting** (Prometheus, Grafana)
- **CI/CD per service**
- **Secrets management** (Key Vault)
- **Schema/contract testing** (Pact)

## Common Patterns
- API Gateway / BFF
- Circuit Breaker, Retry with backoff, Bulkhead
- Saga (orchestration / choreography)
- CQRS, Event Sourcing
- Strangler Fig (incremental migration from monolith)
- Sidecar (cross-cutting concerns: Dapr, service mesh)

## Anti-Patterns
- **Distributed monolith** — services that must be deployed together
- **Shared database** across services
- **Chatty sync calls** (A→B→C→D for one request)
- **Microservices because Netflix does it**
