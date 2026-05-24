# Dependency Injection as a Design Principle

## What it is

Dependency Injection (DI) is the practice of **giving an object its dependencies from outside** instead of having it create or locate them. It's the concrete application of the Dependency Inversion Principle ([10-SOLID.md](10-SOLID.md)): depend on abstractions, supply implementations from the outside.

```csharp
// ✗ — the class creates its own dependencies (tight coupling, untestable)
public class OrderService {
    private readonly SqlRepository _repo = new();
    private readonly SmtpEmail _email = new();
}

// ✓ — dependencies injected via constructor (loose coupling, testable)
public class OrderService(IOrderRepository repo, IEmailSender email) {
    public void Place(Order o) { repo.Save(o); email.Send(o.Email, "Confirmed"); }
}
```

This file covers DI as a **language-level design principle and pattern** — the *why* and the *how* of structuring code around injected dependencies. The framework mechanics of the `Microsoft.Extensions.DependencyInjection` container (lifetimes, registration, scopes) are covered in depth in **DotNetBook's Hosting & DI chapter**; here we focus on the design discipline that applies regardless of container.

---

## Why it matters

DI delivers the benefits SOLID promises:

- **Testability** — inject a fake/mock in tests instead of hitting a real database/network (see [Chapter 16 §03](../16-Testing/03-Mocking.md)).
- **Loose coupling** — `OrderService` doesn't know or care whether the repository is SQL, in-memory, or a stub.
- **Flexibility** — swap implementations (SQL → Postgres, SMTP → SendGrid) without touching the consumer.
- **Explicit dependencies** — a class's constructor *documents* exactly what it needs. No hidden surprises.
- **Single composition point** — wiring happens in one place (the composition root), not scattered through `new` calls.

---

## The three forms of injection

```csharp
public class ReportService {
    // 1. Constructor injection — the default, strongly preferred
    public ReportService(IDataSource source, ILogger logger) { ... }

    // 2. Property injection — for genuinely optional dependencies (rare)
    public IClock? Clock { get; set; }

    // 3. Method injection — pass a dependency to a specific method
    public Report Build(IFormatter formatter) { ... }
}
```

**Constructor injection** is the default and almost always the right choice:
- The object is **fully initialized** after construction — no half-built state.
- Dependencies are **required and explicit** — the type can't exist without them.
- It's **immutable** — store in `readonly` fields (or use a primary constructor).

Use property/method injection only for truly optional or per-call dependencies. If a class has *many* optional injected properties, that's a smell (probably doing too much — SRP).

---

## The composition root

All wiring should happen in **one place** at application startup — the *composition root*. Everywhere else, classes just declare their dependencies and let them be supplied.

```csharp
// Composition root (Program.cs) — the ONE place that knows concrete types
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddScoped<IOrderRepository, SqlOrderRepository>();
builder.Services.AddScoped<IEmailSender, SendGridEmailSender>();
builder.Services.AddScoped<OrderService>();

var app = builder.Build();
// From here on, the container constructs OrderService with its dependencies.
```

Outside the composition root, **avoid `new`-ing dependencies and avoid the service locator** (pulling from the container manually). The container resolves the whole graph from the entry point.

---

## DI is a principle, not (just) a container

You can do DI **without any container** — "pure DI" / "poor man's DI" is just constructing the graph by hand:

```csharp
// Pure DI — no container, fully explicit, perfectly valid for small apps
var repo = new SqlOrderRepository(connectionString);
var email = new SendGridEmailSender(apiKey);
var service = new OrderService(repo, email);
```

The container automates this for large graphs (and manages lifetimes/disposal), but the *principle* — inject dependencies, depend on abstractions — stands on its own. Don't conflate "DI" with "a DI container." Small apps and libraries often need no container at all.

---

## Service locator is NOT dependency injection

```csharp
// ✗ — service locator: dependencies are HIDDEN, pulled from a global
public class OrderService {
    public void Place(Order o) {
        var repo = ServiceLocator.Get<IOrderRepository>();   // hidden dependency!
        repo.Save(o);
    }
}
```

The service locator anti-pattern (also `IServiceProvider.GetService` scattered through business logic) **hides** dependencies — you can't tell what a class needs from its signature, and tests must configure a global. It's the inverse of DI's explicitness. Inject dependencies through the constructor instead. See [09-CommonAntiPatterns.md](09-CommonAntiPatterns.md).

> Exception: framework integration points and factories sometimes legitimately use `IServiceProvider` (e.g., resolving a scoped service inside a singleton via a scope factory). Keep such usage at the edges, not in domain logic.

---

## Lifetimes (the design implications)

The container offers three lifetimes; the *design* concern is matching lifetime to state and thread-safety (mechanics in DotNetBook):

| Lifetime | One instance per | Use for |
|---|---|---|
| **Transient** | Each resolution | Lightweight, stateless services |
| **Scoped** | Each scope (e.g., web request) | Per-request state (`DbContext`) |
| **Singleton** | Application lifetime | Shared, thread-safe state (caches, config) |

The classic bug — the **captive dependency**: injecting a shorter-lived service (Scoped `DbContext`) into a longer-lived one (Singleton) captures it for the app's lifetime, breaking per-request semantics and causing concurrency bugs. Design rule: a service may only depend on services of **equal or longer** lifetime.

```csharp
// ✗ — Singleton capturing a Scoped DbContext (captive dependency)
services.AddSingleton<CacheService>();   // holds a DbContext forever → bug
// ✓ — resolve the scoped dependency per use via a scope factory, or make CacheService scoped
```

---

## Designing for DI

To make your code DI-friendly:
- **Depend on interfaces** for things with real variation or test seams (not everything — see SOLID's "don't over-abstract").
- **Constructor-inject** required collaborators; keep constructors free of work (no I/O, no heavy logic — just assign fields).
- **Avoid `new` for collaborators** that should be injected; `new` is fine for value objects, DTOs, and pure data.
- **Keep the graph shallow** — deep dependency chains signal over-decomposition.
- **One responsibility per service** — a constructor with 10 dependencies is a SRP red flag.

```csharp
// ✗ — too many dependencies → this class does too much (SRP violation)
public OrderService(IRepo r, IEmail e, ISms s, IPdf p, IAudit a, ICache c, IClock k, IMap m) { }

// ✓ — focused; if you need many collaborators, decompose or introduce a facade
public OrderService(IOrderRepository repo, IOrderNotifier notifier) { }
```

---

## Common bugs / gotchas

### Service locator masquerading as DI

Injecting `IServiceProvider` and resolving from it inside methods is service location, not DI. Inject the specific abstractions you need.

### Captive dependency (lifetime mismatch)

A singleton holding a scoped/transient dependency. Depend only on equal-or-longer lifetimes; use a scope factory for the exception.

### Constructor doing work

Heavy logic or I/O in a constructor makes objects expensive to create and hard to test. Constructors should only capture dependencies.

### Over-injection

A constructor with many dependencies indicates the class violates SRP. Decompose, or aggregate related dependencies behind a facade.

### Interface-for-everything

Wrapping every class in a single-implementation interface "for DI" adds indirection with no benefit. Introduce abstractions where there's variation or a test seam.

---

## Summary

- DI = supply dependencies from outside (the concrete form of the Dependency Inversion Principle).
- Benefits: testability, loose coupling, explicit dependencies, single wiring point.
- Prefer **constructor injection** (required, immutable, fully-initialized); property/method injection only for optional/per-call deps.
- Wire everything at the **composition root**; avoid `new`-ing collaborators and avoid the **service locator** (it hides dependencies).
- DI is a **principle**, not a container — "pure DI" by hand is valid; the container just automates the graph and lifetimes.
- Match lifetimes (avoid the **captive dependency**); keep constructors work-free and dependency counts low (SRP).

→ Next: [13-ErrorHandling.md](13-ErrorHandling.md)
