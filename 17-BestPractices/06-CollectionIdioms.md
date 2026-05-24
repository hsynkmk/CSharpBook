# Collection Idioms

## The guiding principle

"Be liberal in what you accept, conservative in what you return." Accept the **broadest** collection type you can work with; return the **most restrictive** type that still gives callers what they need. This decouples your API from internal representations and protects your state from mutation.

The collection internals are in [Chapter 07](../07-Collections/README.md); this is the API-shaping practice.

---

## Accept the least-specific type for parameters

```csharp
// ✓ — works with arrays, lists, LINQ results, anything enumerable
public int Sum(IEnumerable<int> numbers) { ... }

// ✗ — forces callers to materialize a List
public int Sum(List<int> numbers) { ... }
```

If you only **enumerate** the input, accept `IEnumerable<T>`. Callers can pass an array, `List<T>`, `HashSet<T>`, or a lazy LINQ query without converting. Accept the most specific type only when you need its capabilities (e.g., `IReadOnlyList<T>` if you need indexing/count, `ICollection<T>` if you need `Count` cheaply).

### Capability ladder (least → most demanding)

```
IEnumerable<T>        — can iterate once
IReadOnlyCollection<T>— + Count
IReadOnlyList<T>      — + indexing
ICollection<T>        — + Add/Remove/Count (mutable)
IList<T>              — + indexed mutation
List<T> / T[]         — concrete
```

Pick the **lowest rung** that satisfies your method's needs.

---

## Return read-only types — never expose mutable internals

```csharp
public class Order {
    private readonly List<LineItem> _items = new();

    // ✗ — hands out the internal list; callers can Add/Remove/Clear your state!
    public List<LineItem> Items => _items;

    // ✓ — read-only view; mutations go through your methods
    public IReadOnlyList<LineItem> Items => _items;

    public void AddItem(LineItem item) {   // controlled mutation with invariants
        if (_items.Count >= 100) throw new InvalidOperationException("Max 100 items");
        _items.Add(item);
    }
}
```

Returning your internal `List<T>` lets any caller bypass your invariants (`order.Items.Clear()`). Return `IReadOnlyList<T>`/`IReadOnlyCollection<T>` so callers can read but not mutate. `List<T>` implements `IReadOnlyList<T>` directly — no copy needed.

> Caveat: `IReadOnlyList<T>` is a read-only *view*, not an immutable *snapshot*. The caller can't mutate through it, but if you mutate `_items`, they see the change. For a true snapshot, return `.ToArray()`/`.ToList()` or an `ImmutableArray<T>`. See [07-ImmutabilityPatterns.md](07-ImmutabilityPatterns.md).

---

## Materialize defensively when needed

```csharp
// If the caller's IEnumerable might be a lazy query you'll enumerate multiple times,
// or might change underneath you, materialize once.
public Report Build(IEnumerable<Order> orders) {
    var snapshot = orders as IReadOnlyList<Order> ?? orders.ToList();   // enumerate once
    var total = snapshot.Sum(o => o.Total);
    var count = snapshot.Count;       // no re-enumeration
    ...
}
```

`IEnumerable<T>` may be lazy (re-runs each enumeration) or backed by a mutating source. If you enumerate it more than once, materialize it first — otherwise you pay the cost twice and risk inconsistency (deferred execution surprises, see [Chapter 06 §04](../06-LINQ/04-DeferredExecution.md)).

---

## Return empty, never null

```csharp
// ✗ — forces callers to null-check; NullReferenceException waiting to happen
public IReadOnlyList<Order>? GetOrders() => _orders.Count > 0 ? _orders : null;

// ✓ — always return a valid (possibly empty) collection
public IReadOnlyList<Order> GetOrders() => _orders;

// Helpers for empty collections (no allocation)
return Array.Empty<Order>();
return [];                          // collection expression (C# 12+)
return Enumerable.Empty<Order>();
```

A method returning a collection should **never return null** — return an empty collection. Callers can `foreach` without null checks, and `.Count == 0` is the natural "no results" signal. `Array.Empty<T>()` / `[]` are cached/cheap.

---

## Prefer the right collection for the job

```csharp
// Membership testing → HashSet (O(1)), not List.Contains (O(n))
var seen = new HashSet<int>();
if (seen.Add(id)) { /* first time */ }

// Key lookup → Dictionary, not list scan
var byId = orders.ToDictionary(o => o.Id);

// Read-only lookup table built once → FrozenDictionary/FrozenSet
private static readonly FrozenSet<string> Stopwords = LoadStopwords().ToFrozenSet();

// Immutable shared state → ImmutableArray / ImmutableList
public ImmutableArray<string> Tags { get; init; } = [];
```

Choosing the right structure (O(1) vs O(n)) usually dwarfs micro-optimizations. See [Chapter 07](../07-Collections/README.md).

---

## Use `TryGetValue` / modern dictionary APIs

```csharp
// ✗ — double lookup
if (dict.ContainsKey(key)) value = dict[key];

// ✓ — single lookup
if (dict.TryGetValue(key, out var value)) { ... }

// Get-or-add patterns
var list = dict.TryGetValue(key, out var l) ? l : dict[key] = new();
int count = dict.GetValueOrDefault(key);                 // default if absent
ref var slot = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out _);  // advanced
```

`ContainsKey` + indexer is two hashes; `TryGetValue` is one. Analyzer CA1854 flags this. See [Chapter 07 §04](../07-Collections/04-Dictionary.md).

---

## Collection expressions and target typing (C# 12+)

```csharp
int[] a = [1, 2, 3];
List<int> b = [1, 2, 3];
ReadOnlySpan<int> c = [1, 2, 3];
int[] combined = [..a, ..b, 99];          // spread

private readonly List<int> _items = [];    // empty
public IReadOnlyList<string> Tags => [];    // empty read-only
```

Use `[...]` for concise, target-typed collection initialization. See [Chapter 10 §08](../10-AdvancedLanguage/08-CollectionExpressions.md).

---

## Common bugs / gotchas

### Exposing a mutable internal list

`public List<T> Items => _items` lets callers corrupt your state. Return `IReadOnlyList<T>` (view) or a snapshot.

### Returning null instead of empty

Forces null checks everywhere; one missed check = NRE. Return `[]`/`Array.Empty<T>()`.

### Multiple enumeration of a lazy IEnumerable

```csharp
if (query.Any()) total = query.Sum();   // ⚠ — enumerates the query twice
```

Materialize once (`.ToList()`) if you'll touch it more than once. Analyzer warns (CA1851-ish). See [Chapter 06 §04](../06-LINQ/04-DeferredExecution.md).

### `List.Contains` for membership in a loop

O(n) per lookup → O(n·m) overall. Use a `HashSet<T>`.

### Accepting `List<T>` when `IEnumerable<T>` would do

Couples callers to a concrete type and forces conversions. Accept the broadest type you need.

---

## Summary

- **Accept** the least-specific type (`IEnumerable<T>`); **return** the most restrictive useful type (`IReadOnlyList<T>`).
- Never expose mutable internal collections — return read-only views (or true snapshots/`ImmutableArray<T>` when callers must not see later mutations).
- Return **empty, never null** (`[]`, `Array.Empty<T>()`).
- Materialize lazy `IEnumerable<T>` once if you enumerate it multiple times.
- Pick the right structure (HashSet/Dictionary/Frozen) — algorithmic choice beats micro-opts.
- Use `TryGetValue`/`GetValueOrDefault` over `ContainsKey`+indexer; use collection expressions for concise init.

→ Next: [07-ImmutabilityPatterns.md](07-ImmutabilityPatterns.md)
