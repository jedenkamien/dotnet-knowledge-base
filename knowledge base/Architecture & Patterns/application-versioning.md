# Application Versioning Strategies

## Why It Matters
Once code is shipped, **someone depends on its current behavior**. Versioning lets you evolve software without breaking those consumers.

## Semantic Versioning (SemVer)
`MAJOR.MINOR.PATCH` (e.g., `2.4.1`)

- **MAJOR** — breaking changes
- **MINOR** — new functionality, backward compatible
- **PATCH** — backward-compatible bug fixes

Pre-release: `2.0.0-beta.1`. Build metadata: `+build.123`.

Used by NuGet, npm, etc. Drives dependency resolution rules (`^2.4.1` allows minor/patch).

## Calendar Versioning (CalVer)
`YYYY.MM` or `YY.MM.PATCH` (e.g., `2026.04`). Used by Ubuntu, JetBrains, pip. Good for time-based releases where "breaking" isn't meaningful.

## API Versioning

| Approach | Example | Pros | Cons |
|----------|---------|------|------|
| URI | `/v1/users` | Simple, visible, cache-friendly | Not "RESTful" purist view |
| Header | `Accept: application/vnd.api.v2+json` | Clean URIs | Harder to test/cache |
| Query param | `/users?api-version=2` | Flexible | Easy to forget |
| Content negotiation | media type version | Pure REST | Complex tooling |

Pick **one**, be consistent, document deprecation policy.

## Database Schema Versioning
- **Migration tools**: EF Core Migrations, Flyway, Liquibase
- Migrations are **forward-only**, checked into source control, applied in order
- Support **expand/contract** for zero-downtime:
  1. Add new column (nullable)
  2. Deploy code that writes to both old and new
  3. Backfill
  4. Switch reads to new
  5. Remove old column

## Backward & Forward Compatibility
- **Backward compatible**: new code reads old data / handles old requests
- **Forward compatible**: old code tolerates new fields it doesn't know about (ignore unknown JSON props, optional protobuf fields)

Both are necessary during rolling deployments where multiple versions run simultaneously.

## Breaking vs Non-Breaking Changes

**Non-breaking** (safe in MINOR):
- Adding optional fields / endpoints
- Adding new enum values consumers won't act on
- Loosening validation

**Breaking** (require MAJOR or v2):
- Removing or renaming fields/endpoints
- Changing types or required-ness
- Changing semantics of existing fields
- Tightening validation

## Deprecation Strategy
1. Announce deprecation, set a sunset date
2. Add `Deprecation` / `Sunset` HTTP headers
3. Log usage to identify remaining clients
4. Run old + new in parallel
5. Remove after sunset

## Library / Package Versioning
- Follow SemVer strictly so consumers' lockfiles work
- Provide a CHANGELOG (Keep a Changelog format)
- Use feature flags inside one version instead of frequent majors
- For .NET: `<Version>` / `<AssemblyVersion>` / `<FileVersion>` — `AssemblyVersion` changes break binding redirects, so increase carefully

## Deployment Versioning Tactics
- **Blue/green** — two environments, switch traffic
- **Canary** — release to small % of users first
- **Feature flags** — decouple deploy from release; toggle without redeploy
- **Rolling** — replace instances gradually (default in K8s)

## In Short
> SemVer for libraries, URI versioning for public APIs, expand/contract for databases, feature flags for risky changes, and a written deprecation policy so consumers can plan.
