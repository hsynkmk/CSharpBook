# Immutability Patterns

## Why immutability

An immutable object can't change after construction. Benefits:
- **Thread-safe by default** — no locks needed for shared reads; no torn state.
- **Predictable** — once you have a reference, its value never surprises you.
- **Safe to share/cache** — no defensive copies; no "who mutated this?" bugs.
- **Easier to reason about** — no hidden state changes across method calls.

The cost: producing a "changed" version means creating a new object. For most code that's negligible; for hot paths with frequent updates it can matter (mitigated by structural sharing — see below).

---

## The immutability toolbox (C# options)

| Tool | Use for |
|---|---|
| `record` (class) | Immutable reference-type data with value equality |
| `readonly record struct` | Small immutable value-type data |
| `init`-only properties | Immutable after construction, set in initializer |
| `required` members | Force initialization without a constructor |
| `readonly` fields | Field can't change after construction |
| `ImmutableArray<T>` / `ImmutableList<T>` | Immutable collections (structural sharing) |
| `FrozenDictionary`/`FrozenSet` | Immutable read-optimized lookup |

---

## Records — the default for immutable data

```csharp
public record Money(decimal Amount, string Currency);

var price = new Money(9.99m, "USD");
var discounted = price with { Amount = 7.99m };   // non-destructive update; price unchanged
```

`record` synthesizes value equality, `ToString`, deconstruction, and the `with` expression. It's the idiomatic choice for DTOs, value objects, events, and messages. The `with` expression makes "change" ergonomic without mutation. See [Chapter 03 §03](../03-TypeSystem/03-Records.md).

```csharp
// record with validation and computed members
public record Order(int Id, IReadOnlyList<LineItem> Items) {
    public decimal Total => Items.Sum(i => i.Price);   // computed, no stored state
    public Order {
        if (Id <= 0) throw new ArgumentOutOfRangeException(nameof(Id));
    }
}
```

---

## `init`-only and `required` for classes

When a record isn't appropriate (you need reference equality, or class semantics):

```csharp
public class Configuration {
    public required string ConnectionString { get; init; }   // must set, then immutable
    public int Timeout { get; init; } = 30;                    // optional, immutable
    public IReadOnlyList<string> Hosts { get; init; } = [];
}

var config = new Configuration {
    ConnectionString = "...",   // required — compiler enforces
    Timeout = 60,
};
// config.Timeout = 90;   // ✗ — init-only, can't change after construction
```

`init` accessors allow setting only during initialization; `required` forces the caller to set them. Together they give immutable-from-outside types without constructor boilerplate. See [Chapter 10 §04](../10-AdvancedLanguage/04-RequiredMembers.md).

---

## Immutable collections

```csharp
using System.Collections.Immutable;

ImmutableArray<int> a = [1, 2, 3];
ImmutableArray<int> b = a.Add(4);    // a unchanged; b is a new array

ImmutableList<string> list = ImmutableList.Create("x", "y");
var list2 = list.Add("z");           // structural sharing — doesn't copy everything
```

- **`ImmutableArray<T>`** — backed by a plain array; fast reads/iteration, but every mutation copies the whole array. Best for collections that rarely change after creation.
- **`ImmutableList<T>`** — tree-backed; mutations share structure (O(log n) update, no full copy) but reads are slower than an array. Best for collections updated frequently while shared.

This **structural sharing** is what makes immutability practical: `list.Add(z)` reuses most of the existing tree rather than copying.

```csharp
// Builder for efficient bulk construction (avoids per-step copies)
var builder = ImmutableArray.CreateBuilder<int>();
for (int i = 0; i < 1000; i++) builder.Add(i);
ImmutableArray<int> result = builder.ToImmutable();
```

See [Chapter 07 §09](../07-Collections/09-ImmutableCollections.md).

---

## The builder pattern for complex construction

When an object has many optional parts or staged construction, a builder produces an immutable result:

```csharp
public sealed class HttpRequestBuilder {
    private string _url = "";
    private readonly Dictionary<string, string> _headers = new();

    public HttpRequestBuilder Url(string url) { _url = url; return this; }
    public HttpRequestBuilder Header(string k, string v) { _headers[k] = v; return this; }

    public HttpRequest Build() => new(_url, _headers.ToImmutableDictionary());  // immutable result
}

var request = new HttpRequestBuilder()
    .Url("https://api.example.com")
    .Header("Accept", "application/json")
    .Build();   // immutable HttpRequest
```

The mutable builder accumulates state fluently; `Build()` snapshots it into an immutable object. Useful when `with`-expression chains get unwieldy or construction has validation/ordering rules.

---

## When immutability is overkill

Immutability isn't free or always appropriate:

- **Large objects updated frequently** — copying on every change is wasteful. A mutable type (or `ImmutableList`'s structural sharing) may be better. Profile.
- **Entities with identity and lifecycle** (EF Core entities) — ORMs track and mutate them; forcing immutability fights the framework.
- **Local, short-lived, single-threaded mutable state** — a `var list = new List<T>()` you build and discard needs no immutability.
- **Performance-critical inner loops** — mutable buffers (`Span`, pooled arrays) beat allocating new immutable objects.

Use immutability where it pays off: data crossing boundaries, shared/cached state, value objects, configuration, messages. Don't dogmatically make everything immutable.

---

## Defensive copying (when you can't use immutable types)

If you must expose or accept a mutable collection but want to protect state:

```csharp
private readonly List<int> _data;

public MyClass(IEnumerable<int> data) => _data = data.ToList();   // copy in — caller can't mutate ours

public IReadOnlyList<int> Snapshot() => _data.ToArray();           // copy out — caller's mutations don't affect us
```

Copy on the way in (so the caller's later mutations don't affect you) and on the way out (so the caller can't mutate your internals). Immutable types make these copies unnecessary — another reason to prefer them.

---

## Common bugs / gotchas

### Shallow immutability

```csharp
public record Container(List<int> Items);   // ⚠ — the record is immutable, but Items is a mutable List!
var c = new Container([1, 2]);
c.Items.Add(3);   // mutates "immutable" record's contents
```

A `record` with a mutable member (`List<T>`, array) isn't truly immutable — callers mutate the contained collection. Use `IReadOnlyList<T>`/`ImmutableArray<T>` for the member.

### `with` does a shallow copy

`record with { ... }` copies references, not deep object graphs. Nested mutable objects are shared between the original and the copy.

### `ImmutableArray<T>` default is uninitialized

```csharp
ImmutableArray<int> arr = default;   // ⚠ — IsDefault == true; throws on use
arr = [];                            // ✓ — empty, usable
```

A `default` `ImmutableArray<T>` is null-like (not empty). Initialize it (`[]` or `ImmutableArray<T>.Empty`).

### Overusing immutability in hot paths

Allocating a new immutable object per loop iteration creates GC pressure. Use mutable buffers in measured hot paths.

---

## Summary

- Immutability gives thread-safety, predictability, and safe sharing — at the cost of creating new objects on change.
- Default to **records** (value equality + `with`) for immutable data; **`init`/`required`** for immutable classes.
- Use **`ImmutableArray<T>`** (read-fast, copy-on-write) for rarely-changed collections, **`ImmutableList<T>`** (structural sharing) for frequently-updated shared collections; **builders** for complex construction.
- Beware **shallow immutability** — an immutable type with a mutable member isn't immutable; use read-only/immutable members.
- It's **not always right**: ORM entities, large frequently-updated objects, and hot loops may need mutability. Profile and choose deliberately.
- Without immutable types, copy defensively in and out.

→ Next: [08-DefensiveProgramming.md](08-DefensiveProgramming.md)
