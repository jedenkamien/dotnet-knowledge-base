# Domain-Driven Design (DDD)

## Definition
An approach to software design that puts the **business domain** at the center. Code structure, language, and boundaries mirror how domain experts think and talk about the problem.

Coined by Eric Evans (2003, "the Blue Book"). Refined by Vaughn Vernon ("Red Book", "IDDD").

## Why
- Complex business rules get lost in CRUD/anemic models
- Misalignment between dev language and business language causes bugs
- Large systems need **explicit boundaries** to stay maintainable
- Enables teams to scale (each owns a context)

## Two Halves of DDD

### Strategic DDD — the "where" and "why"
How you carve the system into pieces and align teams.

- **Ubiquitous Language** — one shared vocabulary per context, used in code, tests, conversations, and docs. No translation layer between business and devs.
- **Bounded Context** — an explicit boundary inside which a model and language are consistent. The same word ("Customer", "Order") can mean different things in different contexts — and that's fine.
- **Context Map** — diagram of how bounded contexts relate. Patterns:
  - **Partnership** — two teams succeed or fail together
  - **Shared Kernel** — small shared model (use sparingly)
  - **Customer/Supplier** — upstream serves downstream's needs; downstream has negotiating power
  - **Conformist** — downstream has no power; just accepts upstream's model as-is
  - **Anti-Corruption Layer (ACL)** — translation layer to protect your model from a messy/legacy upstream
  - **Open Host Service / Published Language** — well-documented integration API
  - **Separate Ways** — no integration; cheaper to duplicate
- **Core / Supporting / Generic Subdomains**
  - **Core** — your competitive advantage; invest your best people here
  - **Supporting** — needed but not differentiating; build simply
  - **Generic** — buy or use off-the-shelf (auth, billing)

### Tactical DDD — the "how" inside one context
Building blocks for modelling a single bounded context.

- **Entity** — has identity that persists over time. Equality by ID, not by attributes. e.g. `Match` (identified by `MatchId`), `GroupStanding` (one per team within a group — not the whole table; a `Group` holds a collection of them).
- **Value Object** — defined by its attributes, immutable, no identity. Free of side effects, easy to test. e.g. `Score (homeGoals, awayGoals)`, `MatchStatus`, `Money`, `Address`.
  - **"Free of side effects"** means: (1) immutable — state can't change after creation; (2) no external dependencies — `Score(3,2).GoalDifference()` always returns `1` regardless of when/where called; (3) pure methods — they compute and return, they never mutate the object or anything outside it.
- **Aggregate** — cluster of entities + value objects treated as one consistency boundary, accessed only through its **Aggregate Root**. Invariants are enforced inside; one transaction = one aggregate. e.g. `Match` (root) + `Score` + `MatchStatus`; `Group` (root) + collection of `GroupStanding` entities.
- **Domain Event** — something meaningful that happened. Past tense. Drives integration with other aggregates/contexts. e.g. `MatchResultRecorded` — raised by `Match`, consumed by `Group` to update standings.
- **Repository** — collection-like abstraction for loading/saving aggregates. One repo per aggregate root. e.g. `IMatchRepository`, `IGroupRepository`.
- **Domain Service** — operation that doesn't naturally belong to an entity/VO. Lives in the domain layer as a separate class — **not inside an aggregate**. Rule of thumb: if logic naturally belongs to one aggregate's root → make it a method on that root; if it spans multiple entities or needs data not naturally owned by one aggregate → extract to a domain service. e.g. `RankingTieBreaker` — resolves equal-points standings using head-to-head rules that span multiple `GroupStanding` entities and may require querying match history between tied teams.
- **Application Service** — thin orchestration layer: load aggregate → call domain method → save → publish events. No business rules here. e.g. loads `Match`, calls `RecordResult()`, saves, dispatches `MatchResultRecorded`.
- **Factory** — encapsulates complex aggregate creation. e.g. `GroupFactory.CreateWithTeams(groupId, teamIds)`.

## Aggregate Design Rules of Thumb (Vernon)
1. Model true invariants in consistency boundaries — e.g. "a match can't have a result unless it's InProgress" lives in `Match`, not in a service
2. Design **small** aggregates — `Match` and `Group` are separate, not one giant tournament object
3. Reference other aggregates by **ID**, not by object reference — `Match` holds `TeamId`, not a `Team` object
4. Update other aggregates using **eventual consistency** (domain events) — `Group` reacts to `MatchResultRecorded`, it is not updated in the same transaction as `Match`

## Layered Architecture (typical with DDD)

```
┌─────────────────────────────────────┐
│  Presentation / API                 │  controllers, DTOs
├─────────────────────────────────────┤
│  Application                        │  use cases, orchestration
├─────────────────────────────────────┤
│  Domain                             │  entities, VOs, domain services  ← the heart
├─────────────────────────────────────┤
│  Infrastructure                     │  EF Core, messaging, HTTP, files
└─────────────────────────────────────┘
```

Often combined with **Hexagonal / Ports & Adapters** or **Clean Architecture** so the domain depends on nothing external.

## DDD ↔ Microservices
Microservices boundaries should follow **bounded contexts**, not technical layers. A bounded context can map to one service or to a small cluster of services owned by one team. Without DDD, microservices easily devolve into a distributed monolith.

## Event Storming (discovery technique)
Workshop with domain experts: stick orange notes for **domain events** on a wall in time order, then add commands, actors, policies, aggregates. Cheap way to discover bounded contexts and the ubiquitous language before writing code.

## When DDD Is Worth It
- Genuinely complex domain with non-trivial rules (insurance, logistics, finance, healthcare)
- Long-lived system that must evolve for years
- Multiple teams need to work in parallel

## When It's Overkill
- CRUD admin tools
- Short-lived prototypes
- Domain is already well-understood and trivial

## Common Anti-Patterns
- **Anemic Domain Model** — entities are just bags of getters/setters; logic leaks into services. Defeats the point of DDD.
- **God Aggregate** — one huge aggregate that locks everything
- **Leaky Bounded Contexts** — sharing entities/DB tables across contexts
- **DDD-by-folder** — naming folders `Domain`, `Application` without changing how you actually model
- **Ubiquitous Language only in code** — if business people don't use the same words, it's not ubiquitous

---

## Context Map Patterns — Deep Dive

### Upstream / Downstream
Like a river: data and influence **flow downstream**. The upstream produces/owns the model; the downstream consumes it. A breaking change upstream flows down and affects everything below — the downstream has no automatic protection.

### Customer/Supplier vs. Conformist vs. ACL — the power spectrum
The key question: **can the downstream influence the upstream?**

| Pattern | Downstream power | What downstream does |
|---|---|---|
| **Customer/Supplier** | Yes — negotiates API, requests features | Relies on upstream committing to its needs |
| **Conformist** | No | Accepts upstream model as-is, even if inconvenient |
| **ACL** | No | Translates upstream model to protect its own clean model |

### Separate Ways
Two bounded contexts with **no integration at all** — each team solves the problem independently, even duplicating logic. Used when the cost of integration exceeds the cost of duplication.

### Open Host Service / Published Language
- **Open Host Service** — upstream exposes a well-defined, stable API (not its internal model) that any downstream can consume.
- **Published Language** — the data format/schema used in that API is explicitly documented and versioned (OpenAPI spec, Protobuf, event schema).
- They typically go together: the upstream publishes a clean intentional contract, so downstreams don't need ACLs to cope with a messy internal model.

### Multi-context example (e-commerce)
- **Catalog** (upstream) — owns product data, exposes OHS with Published Language
- **Orders** (downstream of Catalog) — uses ACL to translate `CatalogProduct` into its own `OrderItem`
- **Shipping** (downstream of Orders) — listens for `OrderPlaced` domain event and runs its own fulfillment workflow

Each context has its own model, language, and team. The arrows describe who depends on whom.

---

## Aggregates — Deep Dive

### Example: `Order` aggregate (e-commerce)

**Aggregate Root:** `Order` (entity)

**Child entities:**
- `OrderLine` — has identity within the order (line #1, #2…)

**Value objects:**
- `Money` — amount + currency, immutable
- `Address` — shipping destination
- `OrderStatus` — `Pending / Confirmed / Shipped`

**Domain methods:**
- `AddItem(productId, quantity, price)` — enforces: can't add to a shipped order
- `Confirm()` — transitions status, raises `OrderConfirmed` event
- `Cancel()` — enforces: can't cancel if already shipped

`OrderLine` is never manipulated directly from outside — always through `Order`. The root guards all invariants.

### Example: Football group ranking (two aggregates)

**`Match` aggregate:**
- Root: `Match` (entity identified by `MatchId`)
- Value objects: `Score (homeGoals, awayGoals)`, `MatchStatus`, `TeamId` (reference by ID, not object)
- Domain methods:
  - `Start()` — guard: must be Scheduled
  - `RecordResult(homeGoals, awayGoals)` — sets Score, transitions to Finished, raises `MatchResultRecorded`

**`Group` aggregate:**
- Root: `Group`
- Child entities: `GroupStanding` per team — one entity per team, not the whole table. If a group has 4 teams, there are 4 `GroupStanding` entities. Sorting them by `Points` desc, then `GoalDifference`, gives the standings table.
- Domain method: `ApplyResult(matchResult)` — updates standings
- Connects to `Match` via `MatchResultRecorded` domain event (eventual consistency — never holds a `Match` object)

**`GroupStanding` example properties:**
```csharp
public class GroupStanding  // entity — has identity (TeamId) within the Group
{
    public TeamId TeamId { get; private set; }
    public int Wins { get; private set; }
    public int Draws { get; private set; }
    public int Losses { get; private set; }
    public int GoalsFor { get; private set; }
    public int GoalsAgainst { get; private set; }

    public int Points => Wins * 3 + Draws * 1;           // computed
    public int GoalDifference => GoalsFor - GoalsAgainst; // computed
}
```

### Invariants
Rules that must **always be true**, enforced inside the aggregate root's methods — never in controllers or external validators.

**`Match` invariants:**
- Can't record a result unless status is `InProgress`
- Goals can't be negative
- A match must have exactly two different teams
- Can't change result once `Finished`

**`Group` invariants:**
- A team can only appear once in the standings
- Points must be consistent with wins/draws/losses (3/1/0)
- Can't apply the same match result twice (idempotency guard)
- Only teams belonging to this group can have standings in it

### Where to implement invariants
In the **domain layer**, directly inside the aggregate root's methods:

```csharp
public class Match
{
    public void RecordResult(int homeGoals, int awayGoals)
    {
        if (_status != MatchStatus.InProgress)
            throw new DomainException("Can only record result for a match in progress.");
        if (homeGoals < 0 || awayGoals < 0)
            throw new DomainException("Goals cannot be negative.");

        _score = new Score(homeGoals, awayGoals);
        _status = MatchStatus.Finished;
        AddDomainEvent(new MatchResultRecorded(Id, _score));
    }
}
```

The application service just calls the method and handles the exception — it contains no business logic.

---

## Anemic vs. Rich Domain Model

### Anemic (anti-pattern)

```csharp
// Match is just a data bag
public class Match
{
    public int HomeGoals { get; set; }
    public int AwayGoals { get; set; }
    public MatchStatus Status { get; set; }
}

// Business logic leaks into a service
public class MatchService
{
    public void RecordResult(Match match, int homeGoals, int awayGoals)
    {
        if (match.Status != MatchStatus.InProgress) throw ...;
        match.HomeGoals = homeGoals;  // directly mutating from outside
        match.AwayGoals = awayGoals;
        match.Status = MatchStatus.Finished;
    }
}
```

Any code anywhere can put `Match` into an invalid state.

### Rich domain model (correct)

```csharp
public class Match
{
    private MatchStatus _status;
    private Score _score;

    public void RecordResult(int homeGoals, int awayGoals)
    {
        if (_status != MatchStatus.InProgress) throw new DomainException(...);
        _score = new Score(homeGoals, awayGoals);  // Score is a Value Object
        _status = MatchStatus.Finished;
        AddDomainEvent(new MatchResultRecorded(Id, _score));
    }
}
```

`Match` owns and protects its own state. No external code can bypass the rules.

---

## In Short
> Talk to domain experts, find the bounded contexts, model each one with a rich language and small aggregates, integrate via events, and protect the core from the rest with anti-corruption layers.
