# HashSet&lt;T&gt;

## What it is

A **hash-based set** — a collection of unique items with O(1) Add, Remove, and Contains. Implements math-set operations (union, intersection, difference). Internally similar to `Dictionary<T, _>` but with no value side.

```csharp
var set = new HashSet<int> { 1, 2, 3 };
bool added = set.Add(2);            // false — already present
set.Contains(2);                     // true
set.Remove(2);
set.UnionWith(new[] { 4, 5 });       // { 1, 3, 4, 5 }
```

Use when you need "is X in this set?" or "set operations" — fast and natural.

---

## Why it exists

For "uniqueness" use cases, a `List<T>` requires O(n) `Contains` check before every Add. A `Dictionary<T, _>` works but the value side is wasted. `HashSet<T>` is the right shape: just keys, no values.

It also provides set operations (union, intersect, etc.) with optimized implementations — much cleaner than rolling your own.

---

## API surface

```csharp
var s = new HashSet<int>();
var initial = new HashSet<int> { 1, 2, 3 };
var fromSeq = new HashSet<int>(someEnumerable);
var custom = new HashSet<string>(StringComparer.OrdinalIgnoreCase);

// Mutation
bool added = s.Add(42);             // returns false if already present
s.Remove(42);                        // returns false if not present
s.Clear();

// Test
bool has = s.Contains(42);
int count = s.Count;

// Set operations (modifying)
s.UnionWith(other);                   // s ∪ other
s.IntersectWith(other);               // s ∩ other
s.ExceptWith(other);                  // s − other
s.SymmetricExceptWith(other);         // (s ∪ other) − (s ∩ other)

// Set predicates (non-modifying)
s.IsSubsetOf(other);                  // s ⊆ other
s.IsSupersetOf(other);                // s ⊇ other
s.IsProperSubsetOf(other);            // strict
s.IsProperSupersetOf(other);
s.Overlaps(other);                    // s ∩ other ≠ ∅
s.SetEquals(other);                   // same elements

// Bulk
s.EnsureCapacity(10_000);             // .NET 6+
s.TrimExcess();
```

---

## Common patterns

### Deduplication

```csharp
var unique = new HashSet<string>(items, StringComparer.OrdinalIgnoreCase);
```

LINQ alternative: `items.Distinct(StringComparer.OrdinalIgnoreCase).ToList();` — also uses HashSet under the hood.

### Membership test

```csharp
var allowedRoles = new HashSet<string> { "admin", "editor", "viewer" };
bool ok = allowedRoles.Contains(user.Role);
```

O(1) per check. For 4 items it doesn't matter; for 100K items it dominates over List.Contains.

### Set arithmetic

```csharp
var groupA = new HashSet<int> { 1, 2, 3 };
var groupB = new HashSet<int> { 3, 4, 5 };

groupA.UnionWith(groupB);       // groupA becomes { 1, 2, 3, 4, 5 }
groupA.IntersectWith(groupB);   // becomes { 3, 4, 5 } if started with all
```

`UnionWith`/`IntersectWith`/`ExceptWith` modify in place. For non-destructive versions, use LINQ:

```csharp
var union = groupA.Union(groupB).ToHashSet();
var intersection = groupA.Intersect(groupB).ToHashSet();
```

### Finding duplicates

```csharp
var seen = new HashSet<int>();
var duplicates = new List<int>();
foreach (var x in items) {
    if (!seen.Add(x)) duplicates.Add(x);
}
```

`Add` returns false if already in the set — neat way to detect duplicates in one pass.

### Working with custom types

```csharp
public record Coord(int X, int Y);

var visited = new HashSet<Coord>();
visited.Add(new Coord(3, 4));
visited.Contains(new Coord(3, 4));   // true — record equality
```

For records, value equality + hash are auto-generated. For classes, you must override `Equals` and `GetHashCode` (or pass an `IEqualityComparer<T>`).

### Custom comparer

```csharp
public class CaseInsensitiveSet : IEqualityComparer<string> {
    public bool Equals(string? x, string? y) =>
        string.Equals(x, y, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(string s) =>
        StringComparer.OrdinalIgnoreCase.GetHashCode(s);
}

var emails = new HashSet<string>(new CaseInsensitiveSet());
emails.Add("alice@example.com");
emails.Contains("ALICE@EXAMPLE.COM");   // true
```

The simpler form: `new HashSet<string>(StringComparer.OrdinalIgnoreCase)`.

---

## Internals — how HashSet works

Like `Dictionary<T, _>` but without the value field. The same chaining-with-buckets design:

```csharp
private int[] _buckets;
private Slot[] _slots;

private struct Slot {
    public int hashCode;
    public int next;
    public T value;
}
```

Add:
1. Hash the item.
2. Look at the appropriate bucket.
3. Walk the chain — if found, return false (no-op).
4. Else, allocate a new slot, link it.

Remove:
1. Hash, find, unlink.

Contains:
1. Hash, walk chain.

All O(1) average. Same caveats as Dictionary — mutation of keys after insertion breaks them.

### Memory cost per item

About the same as Dictionary per-entry overhead (without the value side): ~16-20 bytes per item plus the item's own size.

---

## Thread safety

Not thread-safe. For concurrent set semantics:
- `ConcurrentBag<T>` (unordered, but allows duplicates — not exactly a set).
- `ConcurrentDictionary<T, byte>` (use the key side, value unused — common pattern).
- `ImmutableHashSet<T>` (persistent).
- `FrozenSet<T>` (.NET 8+, read-only).

```csharp
var concurrentSet = new ConcurrentDictionary<T, byte>();
concurrentSet.TryAdd(item, 0);
concurrentSet.ContainsKey(item);
```

A bit clunky but works.

---

## Common bugs

- **Mutating a class after Add** — hash changes, set can't find it.
- **Adding null** — `HashSet<T>` allows null for reference types (unlike Dictionary keys). Be aware: `set.Contains(null)` works.
- **Set equality with default comparer for floats** — `0.0 == -0.0` true, `NaN == NaN` false. Edge cases for IEEE 754.
- **Comparing two HashSets with `==`** — reference equality. Use `setA.SetEquals(setB)`.

---

## Performance vs alternatives

| Operation | HashSet<T> | SortedSet<T> | List<T> (manual) |
|---|---|---|---|
| Add | O(1) | O(log n) | O(n) to dedupe |
| Contains | O(1) | O(log n) | O(n) |
| Remove | O(1) | O(log n) | O(n) |
| Iteration | O(n) unordered | O(n) sorted | O(n) |
| Min/Max | O(n) | O(log n) | O(n) |
| Range query | n/a | O(log n + k) | n/a |

HashSet wins on Add/Contains/Remove. `SortedSet<T>` wins on ordered iteration, min/max, range. Pick by access pattern.

---

## When to use HashSet

✓ "Is X in this set?" lookups.
✓ Deduplication.
✓ Set arithmetic (union, intersection).
✓ Detecting duplicates in a sequence.

✗ Ordered iteration — use `SortedSet<T>`.
✗ Need a per-item value too — use `Dictionary<TKey, TValue>`.
✗ Multi-threaded — use `ConcurrentDictionary<T, byte>` as a workaround.
✗ Read-only after build — use `FrozenSet<T>` for fastest lookups.

→ Next: [06-StackQueue.md](06-StackQueue.md)
