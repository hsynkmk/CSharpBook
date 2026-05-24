# Frozen Collections

## What it is

`FrozenDictionary<TKey, TValue>` and `FrozenSet<T>` — added in **.NET 8 (2023)** — are **read-only, built-once** hash collections optimized for **fastest possible lookups**. They sacrifice mutation entirely in exchange for ~20-40% faster reads than `Dictionary<TKey, TValue>`.

```csharp
using System.Collections.Frozen;

var frozen = new Dictionary<string, int> {
    ["alice"] = 1,
    ["bob"] = 2,
}.ToFrozenDictionary();

Console.WriteLine(frozen["alice"]);     // 1
frozen["carol"] = 3;                     // ❌ compile error — no setter
```

Use when you build a lookup table once at startup and read it millions of times — config tables, lookup maps for parsers, enum-to-string mappings.

---

## Why they exist

A regular `Dictionary<TKey, TValue>` is general-purpose: it must handle Add, Remove, growth, mid-life mutation. That generality costs a few cycles per lookup.

If you **never** mutate, the dictionary could be specialized:
- Bucket layout chosen for the specific keys present.
- Hash function tuned to minimize collisions for those keys.
- No version-tracking overhead.
- Smaller memory footprint.

`FrozenDictionary` does exactly this. Construction is slow — it analyzes the keys and picks the best layout. Then every lookup is faster than `Dictionary`.

The trade-off: **construction time matters**. If you build a FrozenDictionary inside a request handler, you've lost more than you saved. Build at startup, hold for the app lifetime.

---

## Creation

```csharp
using System.Collections.Frozen;

// From a Dictionary
var dict = new Dictionary<string, int> { ["a"] = 1, ["b"] = 2 };
FrozenDictionary<string, int> frozen = dict.ToFrozenDictionary();

// From IEnumerable<KeyValuePair<>>
var pairs = new[] { KeyValuePair.Create("a", 1), KeyValuePair.Create("b", 2) };
FrozenDictionary<string, int> frozen2 = pairs.ToFrozenDictionary();

// From IEnumerable<T> with key selector
FrozenDictionary<int, User> byId = users.ToFrozenDictionary(u => u.Id);

// With custom comparer
FrozenDictionary<string, int> caseInsensitive =
    dict.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);

// FrozenSet
FrozenSet<int> ids = users.Select(u => u.Id).ToFrozenSet();
FrozenSet<string> roles = new HashSet<string> { "admin", "viewer" }.ToFrozenSet();
```

`ToFrozenDictionary` / `ToFrozenSet` are LINQ-style extension methods. They do the analysis and return the optimized form.

---

## Operations

```csharp
var fd = users.ToFrozenDictionary(u => u.Id);

// Lookup — same API as Dictionary
fd[42];                          // User
fd.TryGetValue(42, out var u);    // bool + out
fd.ContainsKey(42);
fd.Count;
foreach (var (k, v) in fd) { ... }
foreach (var key in fd.Keys) { ... }
foreach (var val in fd.Values) { ... }

// No mutation methods — by design
// fd[99] = newUser;       // ❌ no setter
// fd.Add(...);            // ❌ no Add
```

Same for `FrozenSet<T>` — Contains, Count, iteration, set operations (Union, Intersect, etc.) returning new collections.

---

## Performance characteristics

For lookups (random access by key):

| Collection | Approx ns per lookup |
|---|---|
| `Dictionary<string, int>` | 30-40 ns |
| `FrozenDictionary<string, int>` | 15-25 ns |
| `ImmutableDictionary<string, int>` | 60-100 ns |

(Numbers from BenchmarkDotNet, vary by hardware and key type. Use your own benchmarks.)

`Frozen` is meaningfully faster — sometimes 2× — for the hot path. The win comes from:
- Custom hash analysis: minimal collisions for the specific keys.
- No version checks (no mutation possible).
- Specialized inner loops per key type.

For construction:

| Collection | Build cost |
|---|---|
| `Dictionary<TKey, TValue>` | O(n) — fast |
| `FrozenDictionary<TKey, TValue>` | O(n × analysis) — slower, can be 5-10× slower to build |
| `ImmutableDictionary<TKey, TValue>` | O(n log n) — slow |

So Frozen has the slowest build but fastest reads. If you build once and read often, the net is a win.

---

## When to use

Frozen collections are right when:

1. **The collection is built once and rarely (never) changed.**
2. **Lookups are very hot** — millions per second.
3. **You can pay the build cost up-front** (startup, configuration load).

Examples:
- ASP.NET Core's route table.
- Localization tables.
- Enum-to-display-string mappings.
- Cached lookup tables built from a JSON/config file at startup.

Not the right tool when:
- You mutate the collection.
- The collection is small (< 100 items) — overhead dominates the win.
- The collection is built fresh per request.

---

## Common patterns

### Static config table

```csharp
public static class Config {
    public static readonly FrozenDictionary<string, RuleSet> Rules =
        LoadRules().ToFrozenDictionary(r => r.Name);
}

// Used millions of times:
RuleSet rule = Config.Rules["pricing"];
```

Built once at class init. Hot lookups are fast.

### Enum → metadata

```csharp
public enum Status { Pending, Shipped, Delivered, Cancelled }

private static readonly FrozenDictionary<Status, string> DisplayName =
    new Dictionary<Status, string> {
        [Status.Pending] = "Awaiting Shipment",
        [Status.Shipped] = "In Transit",
        [Status.Delivered] = "Delivered",
        [Status.Cancelled] = "Cancelled by Customer",
    }.ToFrozenDictionary();

string Display(Status s) => DisplayName[s];
```

Pattern matching also works, but FrozenDictionary is more efficient for large maps and cleaner for runtime-loaded data.

### Whitelist / allowlist

```csharp
private static readonly FrozenSet<string> AllowedRoles =
    new[] { "admin", "editor", "viewer" }.ToFrozenSet(StringComparer.OrdinalIgnoreCase);

bool HasAccess(string role) => AllowedRoles.Contains(role);
```

Tiny set, but `Contains` is the hottest call. Frozen wins by a few nanoseconds; might not matter, might matter on a high-QPS endpoint.

### Reading from JSON/database at startup

```csharp
var raw = await db.Settings.ToListAsync();
var settings = raw.ToFrozenDictionary(s => s.Key, s => s.Value);
// ... store somewhere accessible, then never modify
```

Loaded once. Read forever.

---

## Internals — what makes it faster

When you call `ToFrozenDictionary`, the framework analyzes the keys:

1. **Type-specific tuning**: there are specialized implementations for `int`, `string` (small), `string` (large), and a generic fallback.
2. **Hash function**: chooses a custom hash that distributes the given keys well — sometimes with fewer collisions than the default.
3. **Bucket layout**: bucket count and arrangement tuned for the actual key set.
4. **Inline operations**: lookup path is shorter — no version check, no resize check, no fallback.
5. **For small dictionaries** (< ~10 items), uses a linear scan over an array (faster than hashing for tiny sets).

The runtime ships at least a dozen specialized FrozenDictionary types under the hood. The `ToFrozenDictionary` method picks the right one based on key type and collection size.

You can inspect the actual runtime type:
```csharp
Console.WriteLine(frozen.GetType().Name);
// Might be: SmallValueTypeComparableFrozenDictionary, OrdinalStringFrozenDictionary, etc.
```

The result is read-only no matter the underlying implementation. The API surface is the same.

---

## FrozenSet specifics

Same idea as FrozenDictionary but for sets:

```csharp
FrozenSet<int> set = new[] { 1, 2, 3, 4, 5 }.ToFrozenSet();
set.Contains(3);           // O(1), fastest possible
set.Count;
set.Min();                 // LINQ — O(n)
set.Max();
```

For numeric sets, specialized layouts can be even faster than `HashSet<int>` — sometimes O(1) without any hashing (a bitset).

---

## Constraints

- **Read-only after construction** — no Add, Remove, Set, Clear.
- **Built from an existing collection or sequence** — there's no "empty FrozenDictionary you grow into" pattern.
- **Custom comparers must be set at build time** — you can't change it later.
- **Available in .NET 8+** — no .NET Standard support; use Immutable if you need it.

---

## Common bugs

- **Building a FrozenDictionary per request** — pays the analysis cost on every call. Build at startup.
- **Replacing a FrozenDictionary atomically when you do need to update** — you can swap a `FrozenDictionary<,>` reference (volatile or via `Interlocked.Exchange`) to publish a new version. Readers see one snapshot at a time.
- **Expecting `==` to be value equality** — it's reference equality.

---

## Performance summary

| | Dictionary | ImmutableDict | FrozenDict |
|---|---|---|---|
| Build n items | O(n) — fast | O(n log n) — slow | O(n × analysis) — slow |
| Lookup | O(1) avg | O(log n) | O(1) avg, fastest |
| Update | O(1) avg | O(log n), returns new | not supported |
| Thread-safe reads | (multiple readers OK) | yes | yes |
| Memory | medium | high (tree) | small (specialized) |

---

## When to choose what

| Need | Use |
|---|---|
| Build once, read many | **FrozenDictionary** |
| Frequent immutable updates | **ImmutableDictionary** |
| Frequent mutable updates, single-thread | `Dictionary` |
| Concurrent updates | `ConcurrentDictionary` |
| Sorted iteration | `SortedDictionary` |

For the "config table at startup" pattern, **FrozenDictionary is the modern best practice**. Replace `Dictionary<TKey, TValue>` declarations that never mutate with FrozenDictionary — small but real perf wins.

→ Next: [11-EqualityContract.md](11-EqualityContract.md)
