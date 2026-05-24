# Design Patterns in Modern C#

## What they are

Design patterns are reusable solutions to recurring design problems, popularized by the "Gang of Four" (GoF). They're a shared vocabulary — saying "use a Strategy here" communicates a design instantly. But many classic patterns were workarounds for language limitations that **modern C# has since absorbed into the language or BCL**. Knowing which patterns still pull their weight — and which are now one-liners — is the real skill.

This chapter groups patterns by relevance today: still useful, language-replaced, and the ones worth knowing by name.

---

## Patterns C# made (mostly) obsolete

These classic patterns are now built into the language or BCL — you rarely implement them by hand.

| GoF Pattern | Modern C# replacement |
|---|---|
| **Iterator** | `IEnumerable<T>` + `yield return` — the language *is* the iterator |
| **Singleton** | `Lazy<T>` or a DI singleton registration — don't hand-roll it |
| **Observer** | `event` / `IObservable<T>` / `IAsyncEnumerable<T>` |
| **Command** | Delegates (`Action`/`Func`), lambdas, local functions |
| **Strategy** | Inject a `Func<>` or a small interface |
| **Template Method** | `Func<>`/`Action<>` parameters, or default interface methods |
| **Prototype** | `record` + `with` expression (clone-with-changes) |
| **Builder (for data)** | Object initializers, `required`/`init`, `with` |
| **Adapter (simple)** | Extension methods |

```csharp
// Iterator — no IEnumerator implementation needed
public IEnumerable<int> EvenNumbers(int max) {
    for (int i = 0; i <= max; i += 2) yield return i;   // the language generates the iterator
}

// Singleton — Lazy<T> (thread-safe) or DI, never double-checked-locking by hand
private static readonly Lazy<Config> _config = new(() => Load());
public static Config Instance => _config.Value;

// Strategy — a delegate instead of an interface + classes
void Sort(int[] data, Comparison<int> strategy) => Array.Sort(data, strategy);
Sort(data, (a, b) => b - a);   // descending — no IComparer class needed

// Prototype — records clone with changes for free
var modified = original with { Status = Status.Done };

// Observer — events (or IObservable for reactive streams)
public event EventHandler<PriceChangedEventArgs>? PriceChanged;
```

The lesson: **reach for a language feature before a pattern class.** A `Func<>` parameter beats a Strategy class hierarchy; `record with` beats a Prototype clone method.

---

## Patterns still genuinely useful

### Factory / Factory Method

When object creation is non-trivial (chooses a type at runtime, needs configuration), centralize it.

```csharp
public interface IExporter { void Export(Report r); }

public static class ExporterFactory {
    public static IExporter Create(ExportFormat format) => format switch {
        ExportFormat.Pdf  => new PdfExporter(),
        ExportFormat.Csv  => new CsvExporter(),
        ExportFormat.Json => new JsonExporter(),
        _ => throw new ArgumentOutOfRangeException(nameof(format)),
    };
}
```

In DI-heavy code, factories are often replaced by `IServiceProvider`, keyed services (`[FromKeyedServices]`), or a `Func<T>` injected by the container. Use an explicit factory when creation logic is genuinely complex.

### Decorator

Wrap an object to add behavior without changing it — composing cross-cutting concerns (logging, caching, retry).

```csharp
public interface IRepository { Order Get(int id); }

public class CachingRepository(IRepository inner, IMemoryCache cache) : IRepository {
    public Order Get(int id) =>
        cache.GetOrCreate(id, _ => inner.Get(id))!;   // adds caching around inner
}

public class LoggingRepository(IRepository inner, ILogger log) : IRepository {
    public Order Get(int id) {
        log.LogInformation("Getting {Id}", id);
        return inner.Get(id);   // adds logging around inner
    }
}

// Compose: logging → caching → real repo
IRepository repo = new LoggingRepository(new CachingRepository(new SqlRepository(), cache), log);
```

Decorators stack cleanly and keep each concern isolated (SRP). .NET's `Stream` classes (`GZipStream`, `CryptoStream`) and ASP.NET middleware are decorator chains. DI containers support decorator registration (often via `Scrutor`).

### Strategy (when behavior is complex)

When a "strategy" is more than a one-liner (stateful, multiple methods), a small interface is clearer than a fat delegate:

```csharp
public interface IPricingStrategy { decimal Price(Order o); }
public class StandardPricing : IPricingStrategy { ... }
public class VipPricing : IPricingStrategy { ... }

public class Checkout(IPricingStrategy pricing) {
    public decimal Total(Order o) => pricing.Price(o);
}
```

### Options / Builder (for complex construction)

When construction has many optional, ordered, or validated steps, a fluent builder produces an immutable result. See [Chapter 17 §07](07-ImmutabilityPatterns.md). For simple data, prefer object initializers + `required`/`init` instead.

### State machine

When an object's behavior depends on its state and transitions are rules, model states explicitly (often with the State pattern or pattern matching) rather than a tangle of boolean flags.

```csharp
public abstract record OrderState {
    public abstract OrderState Pay();
    public abstract OrderState Ship();
}
public record Pending : OrderState {
    public override OrderState Pay() => new Paid();
    public override OrderState Ship() => throw new InvalidOperationException("Pay first");
}
public record Paid : OrderState {
    public override OrderState Pay() => throw new InvalidOperationException("Already paid");
    public override OrderState Ship() => new Shipped();
}
public record Shipped : OrderState { /* terminal */ }
```

### Mediator

Decouples senders from handlers via a central dispatcher. Popularized by MediatR for CQRS in ASP.NET apps. Useful at application scale; overkill for small apps (don't add a mediator just to call one method). Covered more in DotNetBook's architecture chapter.

---

## Patterns to know by name (so you recognize them)

- **Repository / Unit of Work** — abstracts persistence. Note: EF Core's `DbContext` *is* a Unit of Work + repositories; adding another layer on top is often redundant (see DotNetBook).
- **Dependency Injection** — see [12-DependencyInjection.md](12-DependencyInjection.md).
- **Null Object** — a do-nothing implementation instead of null (`NullLogger`, `Stream.Null`). Reduces null checks.
- **Composite** — tree structures treated uniformly (`Control` hierarchies).
- **Visitor** — operations over an object structure; in C#, often replaced by pattern matching (`switch` on type).
- **Chain of Responsibility** — ASP.NET middleware pipeline is this.
- **Pub/Sub** — events, channels, message buses.

```csharp
// Null Object — no null checks needed at call sites
ILogger logger = configured ? new FileLogger() : NullLogger.Instance;   // always safe to call
logger.Log("hi");

// Visitor replaced by pattern matching
decimal Evaluate(Expr e) => e switch {
    Literal l    => l.Value,
    Add a        => Evaluate(a.Left) + Evaluate(a.Right),
    Multiply m   => Evaluate(m.Left) * Evaluate(m.Right),
    _ => throw new NotSupportedException(),
};
```

---

## When NOT to use a pattern

Patterns are tools, not goals. Anti-uses:
- **Pattern for its own sake** — adding a Factory/Mediator/Repository "because it's good practice" when a direct call is clearer.
- **Speculative flexibility** — building extension points for variation that may never come (YAGNI).
- **Reimplementing the language** — hand-rolling Iterator/Singleton/Observer instead of `yield`/`Lazy`/`event`.
- **Pattern soup** — stacking five patterns where one method would do.

The right question: *does this pattern reduce complexity for a real, present problem?* If not, skip it.

---

## Common bugs / gotchas

### Hand-rolled singleton with broken thread-safety

Double-checked locking done wrong has memory-model bugs. Use `Lazy<T>` or a DI singleton. See [Chapter 08 §09](../08-Concurrency/09-LockingPrimitives.md).

### Repository over EF Core

Wrapping `DbContext` (already a UoW + repository) in another repository layer usually adds indirection without value, and hides EF's querying power. Evaluate whether you need it.

### Mediator/CQRS for a CRUD app

MediatR + CQRS adds ceremony. Use it when command/query separation and cross-cutting pipeline behaviors pay off — not for simple CRUD.

### Decorator order matters

`Logging(Caching(Repo))` logs every call; `Caching(Logging(Repo))` logs only cache misses. Order the chain deliberately.

---

## Summary

- Design patterns are a shared vocabulary, but **modern C# absorbed many** into the language/BCL: Iterator (`yield`), Singleton (`Lazy<T>`/DI), Observer (`event`), Strategy/Command (delegates), Prototype/Builder-for-data (`record`/`with`/`init`).
- Still genuinely useful: **Factory** (complex creation), **Decorator** (cross-cutting concerns), **Strategy** (complex behavior), **State machine**, **Mediator** (at scale).
- Know by name: Repository/UoW (but EF Core already is one), Null Object, Composite, Visitor (→ pattern matching), Chain of Responsibility (→ middleware).
- Reach for a **language feature before a pattern class**; use patterns only when they reduce complexity for a real problem — never speculatively.

→ Next: [12-DependencyInjection.md](12-DependencyInjection.md)
