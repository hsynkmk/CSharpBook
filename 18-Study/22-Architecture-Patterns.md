# 22 — Architecture & Patterns (SOLID, GoF, CQRS, anti-patterns)

## ⚡ 30-second answer

**SOLID** is the five OO design principles that keep code maintainable and testable: **S**ingle responsibility, **O**pen/closed, **L**iskov substitution, **I**nterface segregation, **D**ependency inversion. **Design patterns** (GoF) are reusable solutions to common problems — know Strategy, Factory, Singleton, Decorator, Observer, Repository. For app architecture: layer dependencies **inward** (domain has no dependencies), consider **CQRS** (separate read/write) when models diverge, and avoid the classic **anti-patterns** (god class, anemic domain, service locator, sync-over-async, magic strings). The meta-skill: match the pattern to the problem — don't over-engineer.

---

## Core mechanics — SOLID

| Letter | Principle | One-liner |
|---|---|---|
| **S** | Single Responsibility | a class should have one reason to change |
| **O** | Open/Closed | open for extension, closed for modification (add via new types, not editing) |
| **L** | Liskov Substitution | a subtype must be usable wherever its base is (don't break the contract) |
| **I** | Interface Segregation | many small focused interfaces > one fat one |
| **D** | Dependency Inversion | depend on **abstractions**, not concretions (the basis of DI — [15](15-DI-Hosting-Config.md)) |

```csharp
// D + O: depend on an interface; add behavior by adding implementations
public class OrderProcessor(IPaymentGateway gateway) { … }   // not a concrete Stripe class
```

## Core mechanics — key patterns

| Pattern | Solves | Example in .NET |
|---|---|---|
| **Strategy** | swap an algorithm at runtime | inject `IComparer`/`IPolicy` |
| **Factory** | encapsulate creation | `IHttpClientFactory`, `ILoggerFactory` |
| **Singleton** | one shared instance | DI singleton lifetime (prefer over static) |
| **Decorator** | add behavior without changing the type | logging/caching wrapper around a service |
| **Observer** | notify subscribers | events / `IObservable` |
| **Repository / Unit of Work** | abstract persistence | `DbContext` *is* UoW ([17](17-EFCore.md)) |
| **Adapter** | bridge incompatible interfaces | wrap a third-party API |
| **Mediator** | decouple request → handler | MediatR-style (CQRS) |

## App architecture

- **Layered/Clean**: dependencies point **inward** — Domain (no deps) ← Application ← Infrastructure; Infrastructure implements the interfaces the domain declares (Dependency Inversion). Domain is testable without a DB/web.
- **CQRS**: separate **commands** (writes, go through the rich domain) from **queries** (reads, project to DTOs, bypass the domain). Lightweight (handlers, one DB) is the common form; full CQRS+event sourcing is rarely needed.
- **Domain events / outbox**: an aggregate records "what happened"; handlers react; the **outbox** publishes integration events atomically ([20](20-Observability-Messaging-Background.md)).
- **Monolith → modular monolith → microservices**: start simple; extract services only when independent scaling/team autonomy justifies the distributed cost.

---

## 🪤 Anti-patterns to retire

| Anti-pattern | Why bad | Instead |
|---|---|---|
| **God class / fat controller** | does everything, untestable | single responsibility; thin controllers + handlers ([16](16-AspNetCore.md)) |
| **Anemic domain model** | data bags + logic scattered in services; invariants unprotected | rich models that own behavior (for complex domains) |
| **Service locator** | hidden dependencies, hard to test | constructor injection ([15](15-DI-Hosting-Config.md)) |
| **Sync-over-async** (`.Result`) | deadlock / thread-pool starvation | async all the way ([10](10-AsyncAwait.md), [12](12-Concurrent-Parallel-AsyncBugs.md)) |
| **Magic strings / primitive obsession** | typos compile, no refactor safety | constants, `nameof`, value objects, enums |
| **Generic `Repository<T>` over EF Core** | duplicates/hides EF Core | use DbContext directly or a domain-specific repo ([17](17-EFCore.md)) |
| **Swallowed exceptions** (`catch {}`) | hides failures | catch specific, handle or propagate ([07](07-Exceptions-Idioms.md)) |
| **Premature microservices/optimization** | unearned complexity | start simple, measure first ([21](21-Deployment-Perf-Tooling.md)) |

---

## ❓ Likely questions

**Q: Explain SOLID.**
A: Single responsibility (one reason to change), Open/closed (extend without modifying), Liskov (subtypes honor base contracts), Interface segregation (small focused interfaces), Dependency inversion (depend on abstractions). Together they reduce coupling and improve testability.

**Q: Dependency Inversion vs Dependency Injection?**
A: DI principle (the "D") says depend on abstractions. Dependency *injection* is a technique that realizes it — supplying those abstractions from outside (a container). DI is how you achieve DIP.

**Q: Give a real use of Strategy / Factory / Decorator.**
A: Strategy: inject an `IPricingPolicy` to swap pricing logic. Factory: `IHttpClientFactory` creates configured clients. Decorator: wrap a repository with a caching/logging decorator without changing it.

**Q: What is CQRS and when is it overkill?**
A: Separating read and write models/handlers. Use it when reads and writes genuinely diverge or need independent optimization. Overkill for simple CRUD — just use controllers + EF Core.

**Q: Singleton pattern vs DI singleton?**
A: The classic Singleton (static instance) is hard to test and hides global state. Prefer a DI **singleton lifetime** — same single-instance benefit but injectable/mockable.

**Q: Repository pattern with EF Core — needed?**
A: Often not — `DbContext` is already Unit of Work and `DbSet` is a repository. Add a domain-specific repository only to decouple the domain or encapsulate complex queries; avoid a generic wrapper.

**Q: Name anti-patterns you avoid.**
A: God classes/fat controllers, anemic domain models, service locator, sync-over-async, magic strings/primitive obsession, swallowed exceptions, premature microservices. Each has a cleaner alternative.

**Q: How do you decide monolith vs microservices?**
A: Start with a (modular) monolith — simplest to build/deploy. Split into services only when you need independent scaling, separate release cadence, or team autonomy enough to justify distributed-systems complexity.

---

## 🎓 Senior Extra

- **DDD building blocks**: **aggregates** (consistency boundary with a root that enforces invariants), **value objects** (identity-less, immutable — [01](01-TypeSystem.md)), **domain events**, **bounded contexts**. Reference other aggregates **by id**; save one aggregate atomically.
- **Vertical slice architecture**: organize by *feature* (endpoint+handler+validation together) instead of by technical layer — high cohesion, low coupling; pairs with CQRS/Minimal APIs. Accepts some duplication over a premature shared abstraction.
- **Outbox + idempotency + sagas** give reliable, eventually-consistent cross-service workflows over at-least-once messaging ([20](20-Observability-Messaging-Background.md)).
- **Multi-tenancy** isolation: tenant-per-row (`TenantId` + EF **global query filter** so you can't forget it), per-schema, or per-database — trade isolation vs cost; treat as a **security boundary** ([17](17-EFCore.md), [19](19-Security-Auth.md)).
- **Feature flags** decouple deploy from release (ship dark, enable centrally, roll back instantly) — `Microsoft.FeatureManagement` + App Configuration.
- **Versioning**: SemVer for libraries (MAJOR = breaking); API versioning (`Asp.Versioning`) serves v1/v2 simultaneously so clients migrate on their own schedule ([16](16-AspNetCore.md)).
- **Composition over inheritance**: deep hierarchies are fragile; compose behavior via injected collaborators (Strategy/DI) — the bias behind most modern patterns ([02](02-OOP.md)).
- **YAGNI / match the tool**: patterns are solutions to *problems you have* — applying CQRS/microservices/repository everywhere is itself an anti-pattern. Justify complexity.

→ Deeper: [`../CSharpBook/17-BestPractices/`](../CSharpBook/17-BestPractices/README.md), [`../DotNetBook/22-BestPractices/`](../DotNetBook/22-BestPractices/README.md)
