# REST Fundamentals

## What REST Is
**Representational State Transfer** — an architectural style (Roy Fielding, 2000), not a protocol. Most "REST APIs" today are HTTP+JSON APIs that follow some, but rarely all, REST constraints.

## Six Constraints (Fielding)
1. **Client–Server** — separation of concerns
2. **Stateless** — each request contains all info needed; no server-side session
3. **Cacheable** — responses indicate cacheability
4. **Uniform Interface** — resources, representations, self-descriptive messages, HATEOAS
5. **Layered System** — proxies, gateways, CDNs are transparent
6. **Code on Demand** (optional)

The **uniform interface** (especially HATEOAS) is what most "REST" APIs skip.

## Resources & URIs
- A resource is a noun: `/users/42`, `/orders/2024/pending`
- Use plural nouns; avoid verbs in paths (`/getUser` ❌ → `GET /users/42` ✅)
- Hierarchy reflects relationships: `/users/42/orders`
- Use query params for filtering/sorting/pagination: `/orders?status=pending&page=2`

## HTTP Methods (Verbs)

| Method | Purpose | Idempotent | Safe |
|--------|---------|-----------|------|
| GET | Read | ✅ | ✅ |
| POST | Create / non-idempotent action | ❌ | ❌ |
| PUT | Replace whole resource | ✅ | ❌ |
| PATCH | Partial update | ❌ (typically) | ❌ |
| DELETE | Remove | ✅ | ❌ |
| HEAD | Headers only | ✅ | ✅ |
| OPTIONS | Capabilities / CORS | ✅ | ✅ |

**Idempotent**: same request multiple times → same effect.
**Safe**: no server state change.

## Status Codes (essentials)

| Range | Meaning |
|-------|---------|
| 2xx | Success — `200 OK`, `201 Created`, `204 No Content` |
| 3xx | Redirection — `301`, `304 Not Modified` |
| 4xx | Client error — `400 Bad Request`, `401 Unauthorized`, `403 Forbidden`, `404 Not Found`, `409 Conflict`, `422 Unprocessable`, `429 Too Many Requests` |
| 5xx | Server error — `500 Internal`, `502 Bad Gateway`, `503 Unavailable` |

Pick the *most specific* code; don't return `200 { error: ... }`.

## Statelessness
Each request must be self-contained: auth tokens, context, etc. Enables horizontal scaling — any server can handle any request. Sessions live client-side (cookies, JWTs) or in shared store (Redis), not in server memory.

## Caching
- `Cache-Control`, `ETag`, `Last-Modified`
- Conditional requests: `If-None-Match` → `304 Not Modified`
- Public CDN-cacheable for `GET` of public data

## Versioning
- URI: `/v1/users` (most common, simplest)
- Header: `Accept: application/vnd.myapi.v2+json`
- Query: `?version=2`
Pick one and be consistent.

## Common Anti-Patterns
- Verbs in URLs (`/createUser`)
- Tunnelling everything through `POST`
- Returning `200` with error payload
- Ignoring status codes
- Massive nested resources (`/a/b/c/d/e/f`)
- Breaking changes without versioning

## REST vs Alternatives
- **GraphQL** — single endpoint, client picks fields, good for varied UI needs
- **gRPC** — binary, contract-first (protobuf), high performance, service-to-service
- **SOAP** — XML, contract-heavy, legacy enterprise
REST stays popular because of HTTP ubiquity, tooling, and cacheability.
