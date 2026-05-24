# Dictionary&lt;TKey, TValue&gt;

## What it is

A **hash table** mapping keys to values. O(1) average for Add, Lookup, Remove. The most-used non-trivial collection in .NET — every cache, lookup table, and "I need to find X by name" call probably uses one.

```csharp
var ages = new Dictionary<string, int> {
    ["Alice"] = 30,
    ["Bob"] = 25,
};
ages["Carol"] = 40;

Console.WriteLine(ages["Alice"]);   // 30
ages.Remove("Bob");
bool has = ages.ContainsKey("Alice");

foreach (var (name, age) in ages) Console.WriteLine($"{name}: {age}");
```

Internally: an array of buckets, with collision resolution via chaining (linked list per bucket). `Dictionary<TKey, TValue> where TKey : notnull` — keys can't be null.

---

## Why it exists

Without a hash table, "find item by key" is O(n) (scan a list). Hash tables collapse this to O(1) average — pay a small per-key cost to compute a hash, jump straight to the right bucket.

Every modern language has one (Python `dict`, Java `HashMap`, JavaScript `Map`/`Object`, Rust `HashMap`). C#'s `Dictionary<TKey, TValue>` is the canonical implementation.

---

## API surface

```csharp
var d = new Dictionary<string, int>();
var sized = new Dictionary<string, int>(capacity: 1000);
var custom = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
var fromPairs = new Dictionary<string, int> { ["a"] = 1, ["b"] = 2 };

// Set / get
d["key"] = 42;                  // add or overwrite
int v = d["key"];                // throws if missing
d.Add("new", 99);                // throws if key exists
d.Remove("key");                 // returns bool
bool removed = d.Remove("key", out int oldValue);   // .NET 6+

// Test
bool has = d.ContainsKey("key");
bool hasValue = d.ContainsValue(42);     // O(n)!

// Try
if (d.TryGetValue("key", out var x)) { ... }

// .NET 6+
int v2 = d.GetValueOrDefault("key");        // default if missing
int v3 = d.GetValueOrDefault("key", 99);    // custom default

// Enumerate
foreach (var (k, v) in d) { ... }
foreach (var key in d.Keys) { ... }
foreach (var val in d.Values) { ... }

d.Clear();
int count = d.Count;

// Bulk
d.EnsureCapacity(10_000);    // .NET 6+
d.TrimExcess();
```

---

## Equality and hashing

Dictionary uses `IEqualityComparer<TKey>` to compare keys. By default, `EqualityComparer<TKey>.Default` — calls `key.GetHashCode()` and `key.Equals(otherKey)`.

For strings, the default is **case-sensitive ordinal**. To make case-insensitive:

```csharp
var d = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
d["Hello"] = 1;
Console.WriteLine(d["HELLO"]);    // 1
```

Custom keys must implement `Equals` and `GetHashCode` consistently. The **equality contract**:

1. If `a.Equals(b)`, then `a.GetHashCode() == b.GetHashCode()`.
2. Equality is reflexive, symmetric, transitive.
3. Hash codes should be **stable for the duration of the key's life as a dictionary key**.

Violating these leads to "items mysteriously disappear" bugs. See [§11 — EqualityContract](11-EqualityContract.md).

For records, equality and hashing are auto-synthesized — they're great Dictionary keys.

For mutable classes used as keys: don't. If a key's hash changes after insertion, the dictionary can't find it again.

---

## Common patterns

### `TryGetValue` for safe access

```csharp
if (cache.TryGetValue(key, out var value)) {
    return value;
}
value = LoadFromDb(key);
cache[key] = value;
return value;
```

`TryGetValue` is faster than `if (ContainsKey(key)) return d[key];` — one lookup instead of two.

### `GetValueOrDefault` (.NET 6+)

```csharp
int retries = config.GetValueOrDefault("retries", 3);
```

Cleaner for "use the value if set, fallback if not."

### Counting

```csharp
var counts = new Dictionary<string, int>();
foreach (var word in words) {
    counts.TryGetValue(word, out var count);
    counts[word] = count + 1;
}
```

Or with `CollectionsMarshal.GetValueRefOrAddDefault` (.NET 6+) for the absolute fastest:

```csharp
foreach (var word in words) {
    ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(counts, word, out _);
    count++;
}
```

Single lookup, ref to the slot, increment in place. Used by performance-sensitive code (parsers, tokenizers).

### Lookup-then-update

```csharp
counts[key] = (counts.TryGetValue(key, out var c) ? c : 0) + 1;
```

Cleaner one-liner but does two hash lookups. Use ref-based form for hot paths.

### Initialization from key-selector

```csharp
var byId = users.ToDictionary(u => u.Id);              // value = User itself
var byIdName = users.ToDictionary(u => u.Id, u => u.Name);
```

LINQ's `ToDictionary` is the idiomatic way to build one. Throws on duplicate keys; for "first wins" use `GroupBy + ToDictionary` with first/single per group.

### Multi-value via List

```csharp
var byCountry = new Dictionary<string, List<User>>();
foreach (var u in users) {
    if (!byCountry.TryGetValue(u.Country, out var list)) {
        list = new();
        byCountry[u.Country] = list;
    }
    list.Add(u);
}
```

Or use `ToLookup` for read-only multi-value:

```csharp
ILookup<string, User> byCountry = users.ToLookup(u => u.Country);
foreach (var u in byCountry["US"]) { ... }
```

`ILookup<TKey, T>` is read-only after creation — like a Dictionary of Lists, but built-in.

---

## Internals — how Dictionary works

A `Dictionary<TKey, TValue>` has three private fields (simplified):

```csharp
private int[] _buckets;        // for each bucket, index into _entries (or -1 if empty)
private Entry[] _entries;       // each entry: hashCode, next index, key, value
private int _count;             // number of items

private struct Entry {
    public int hashCode;        // negative if free
    public int next;            // next entry in chain, or -1
    public TKey key;
    public TValue value;
}
```

When you call `d.Add(key, value)`:

1. Compute `key.GetHashCode()`.
2. Index into `_buckets` via `hash % _buckets.Length`.
3. Walk the chain (linked via `Entry.next`):
   - If a matching `key.Equals` is found → key already exists (throw, or overwrite).
   - Else, allocate the next entry in `_entries`, set its fields, link it into the chain.
4. If the load factor would exceed ~1, resize.

When you call `d[key]`:
1. Hash + index.
2. Walk the chain, comparing keys.
3. Return matching value, or throw if not found.

Average O(1). Worst case O(n) — if all keys collide to one bucket. Mitigated by **prime-numbered bucket counts** and a high-quality hash function.

### Resizing

When `Count` reaches `_buckets.Length`, the dictionary resizes to the next prime > `_buckets.Length * 2`. All entries are re-hashed into new buckets. O(n) cost, but happens log₂(n) times → amortized O(1) for Add.

### Collision attacks

If an adversary can choose keys, they can deliberately make many keys collide → O(n) per operation. .NET mitigates this for `string` keys via **randomized hashing** — the `string.GetHashCode` includes a per-process random seed (when `<DefaultStringComparisonComparer>` is the SharedRandomComparer).

For custom types: implement `GetHashCode` using `HashCode.Combine` (which uses xxHash with the per-process seed) — avoids predictable collisions.

```csharp
public override int GetHashCode() => HashCode.Combine(X, Y, Z);
```

Don't roll your own with `X ^ Y ^ Z` — those are predictable and prone to collision attacks.

### Memory cost per item

Per Entry: 4 (hash) + 4 (next) + sizeof(TKey) + sizeof(TValue) + padding.

For `Dictionary<string, int>`:
- Each entry: 4 + 4 + 8 (string ref) + 4 (int) + 4 (padding) = 24 bytes
- Plus the bucket array: ~one int per ~1.5 entries.
- Total overhead: ~30 bytes per entry, on top of the string and int values.

For 1M entries: ~30 MB of bookkeeping, vs ~4 MB for a `List<int>`. Trade-off for O(1) lookup.

### `CollectionsMarshal.GetValueRefOrAddDefault` (.NET 6+)

The performance-tuning escape hatch:

```csharp
ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out bool exists);
if (!exists) count = 1; else count++;
```

Returns a **ref to the slot**, doing one hash lookup. You can read/write directly. Avoids the double-lookup of `d.TryGetValue + d[key] = ...`.

Caveat: if you re-hash the dictionary (Add, Remove that triggers resize) while holding the ref, it becomes invalid. Use ref operations atomically — get ref, mutate, done.

---

## Thread safety

`Dictionary<TKey, TValue>` is **not thread-safe**. Multiple readers are safe **only if no one writes**. Any concurrent write is undefined behavior (and can corrupt the data structure).

For concurrent use:
- `ConcurrentDictionary<TKey, TValue>` — thread-safe, lock-free reads.
- `FrozenDictionary<TKey, TValue>` (.NET 8+) — read-only, fastest possible reads.
- `ImmutableDictionary<TKey, TValue>` — persistent (each mutation returns a new instance).

See [§09](09-ImmutableCollections.md), [§10](10-FrozenCollections.md), and [Chapter 08 §12](../08-Concurrency/12-ConcurrentCollections.md).

---

## Common bugs

### Mutating a key after insertion

```csharp
public class MutableKey {
    public int X;
    public override int GetHashCode() => X.GetHashCode();
    public override bool Equals(object? o) => o is MutableKey m && m.X == X;
}

var d = new Dictionary<MutableKey, string>();
var k = new MutableKey { X = 5 };
d[k] = "five";
k.X = 99;                              // ⚠ — hash changed!
d.ContainsKey(k);                       // false! Can't find it anymore.
```

The entry is still in the dictionary, in the bucket determined by the **original** hash. Looking it up now hashes to a different bucket. The entry is unreachable except by enumeration.

**Rule**: dictionary keys must be **effectively immutable** — at least the parts used in equality/hashing must not change while the key is in the dictionary.

### Default `EqualityComparer<T>.Default` for case-insensitive strings

```csharp
var d = new Dictionary<string, int>();
d["Hello"] = 1;
d["HELLO"];     // throws KeyNotFoundException — case-sensitive
```

Use `StringComparer.OrdinalIgnoreCase` for case-insensitive lookups.

### Adding to Keys / Values collections

```csharp
d.Keys.Add("x");    // ❌ KeyCollection is read-only
```

Modify the dictionary, not the views.

### Treating Values as a fast lookup

```csharp
d.ContainsValue(42);   // O(n) — scans every entry
```

`ContainsKey` is O(1); `ContainsValue` is O(n). For value-based lookup, build the inverse dictionary.

### Missing key

```csharp
int v = d["missing"];   // throws KeyNotFoundException
```

Use `TryGetValue` or `GetValueOrDefault`.

---

## Performance summary

| Operation | Time |
|---|---|
| `d[key]` / `Add` / `TryGetValue` | O(1) average, O(n) worst |
| `Remove` | O(1) average |
| `ContainsKey` | O(1) average |
| `ContainsValue` | O(n) |
| Iteration | O(n) |
| Resize on growth | O(n), happens log times |

For ultra-hot lookups:
- Use a comparer optimized for your key type (e.g., `StringComparer.Ordinal` for case-sensitive strings — fastest).
- Pre-size with `EnsureCapacity` or constructor capacity.
- Use `CollectionsMarshal.GetValueRefOrAddDefault` to avoid double-lookups.
- Consider `FrozenDictionary<TKey, TValue>` for read-only data.

---

## When to use Dictionary

✓ Key-value lookup, O(1) needed.
✓ Cache.
✓ Deduplication by key (`ToDictionary(keySelector)`).
✓ Counting (key → count).

✗ Ordered traversal — use `SortedDictionary<TKey, TValue>`.
✗ Multi-value per key — use `Dictionary<TKey, List<TValue>>` or `ILookup<TKey, TValue>`.
✗ Concurrent access — use `ConcurrentDictionary<TKey, TValue>`.
✗ Read-only data, very hot reads — use `FrozenDictionary<TKey, TValue>`.

→ Next: [05-HashSet.md](05-HashSet.md)
