# The Collection Interface Hierarchy

> `IEnumerable<T>` → `ICollection<T>` → `IList<T>` and `IReadOnly*` variants. Choosing the right interface for API parameters and return types.

```
IEnumerable<T>                     // Can iterate
    │
    ├── ICollection<T>              // ... and has Count, Add, Remove, Clear, Contains
    │       │
    │       ├── IList<T>            // ... and indexed access
    │       │
    │       └── ISet<T>             // ... and set operations
    │
    ├── IReadOnlyCollection<T>      // Has Count (read-only)
    │       │
    │       └── IReadOnlyList<T>    // ... and indexed access (read-only)
    │
    └── IDictionary<K, V> (separate hierarchy)
            │
            └── IReadOnlyDictionary<K, V>
```

Each interface adds capability. Code accepting a parameter should ask for the **least capable** interface that suffices — maximizing what callers can pass.

---

## `IEnumerable<T>`

```csharp
public interface IEnumerable<out T> : IEnumerable {
    IEnumerator<T> GetEnumerator();
}
```

The most basic — anything you can iterate. Implemented by all collections, LINQ queries, iterator methods (`yield return`), arrays.

```csharp
public int Sum(IEnumerable<int> nums) {
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}
```

`Sum` accepts arrays, lists, sets, LINQ queries, custom iterators — anything iterable.

**Covariant** (`out T`) — `IEnumerable<string>` can be assigned to `IEnumerable<object>`.

**Beware**: `IEnumerable<T>` is one-shot for many sources. Iterating a LINQ query twice runs it twice. For multi-use, materialize first or accept `IReadOnlyCollection<T>`.

---

## `ICollection<T>`

```csharp
public interface ICollection<T> : IEnumerable<T> {
    int Count { get; }
    bool IsReadOnly { get; }
    void Add(T item);
    bool Remove(T item);
    void Clear();
    bool Contains(T item);
    void CopyTo(T[] array, int arrayIndex);
}
```

Mutable, finite-size collection. `List<T>`, `HashSet<T>`, `LinkedList<T>` implement it. `string[]` does NOT (arrays are weird — they expose `ICollection<T>` via System.Array but the implementation throws on mutation).

Less commonly used as a parameter type than `IEnumerable<T>` or `IList<T>`.

---

## `IList<T>`

```csharp
public interface IList<T> : ICollection<T> {
    T this[int index] { get; set; }
    int IndexOf(T item);
    void Insert(int index, T item);
    void RemoveAt(int index);
}
```

Adds indexed access. `List<T>` is the canonical implementer.

Use as a parameter when you specifically need indexed access AND mutation.

**Invariant** — `IList<string>` is NOT `IList<object>`. Unlike `IEnumerable<T>` which is covariant, mutable sequence interfaces must be invariant for type safety.

---

## `ISet<T>`

```csharp
public interface ISet<T> : ICollection<T> {
    new bool Add(T item);
    void UnionWith(IEnumerable<T> other);
    void IntersectWith(IEnumerable<T> other);
    void ExceptWith(IEnumerable<T> other);
    void SymmetricExceptWith(IEnumerable<T> other);
    bool IsSubsetOf(IEnumerable<T> other);
    bool IsSupersetOf(IEnumerable<T> other);
    bool IsProperSubsetOf(IEnumerable<T> other);
    bool IsProperSupersetOf(IEnumerable<T> other);
    bool Overlaps(IEnumerable<T> other);
    bool SetEquals(IEnumerable<T> other);
}
```

Implemented by `HashSet<T>` and `SortedSet<T>`. Less often used as a parameter — most code accepts `IEnumerable<T>` and decides whether to dedupe internally.

`IReadOnlySet<T>` (.NET 5+) is the read-only counterpart.

---

## `IReadOnlyCollection<T>`

```csharp
public interface IReadOnlyCollection<out T> : IEnumerable<T> {
    int Count { get; }
}
```

Iterable + has a Count. **Covariant**. Implemented by `List<T>` (yes, the mutable list also implements the read-only variant), `IReadOnlyList<T>` derivatives, etc.

Useful as a return type or parameter when you need to know the size up front but won't mutate.

---

## `IReadOnlyList<T>`

```csharp
public interface IReadOnlyList<out T> : IReadOnlyCollection<T> {
    T this[int index] { get; }
}
```

Indexed access without mutation. **Covariant**. `List<T>` implements it.

**This is the right return type for "give the caller a snapshot they can iterate and index."** Doesn't reveal whether you used a list, an array, or some other indexable collection. Doesn't let them Add/Remove.

```csharp
public IReadOnlyList<User> ActiveUsers() => _users.FindAll(u => u.IsActive);
// Caller can iterate and index, but not mutate
```

---

## `IReadOnlyDictionary<K, V>` / `IDictionary<K, V>`

Separate hierarchy for key-value pairs. `Dictionary<K, V>` implements both.

```csharp
public interface IReadOnlyDictionary<TKey, TValue> : IReadOnlyCollection<KeyValuePair<TKey, TValue>> {
    TValue this[TKey key] { get; }
    IEnumerable<TKey> Keys { get; }
    IEnumerable<TValue> Values { get; }
    bool ContainsKey(TKey key);
    bool TryGetValue(TKey key, out TValue value);
}
```

Use `IReadOnlyDictionary<K, V>` for read-only lookups. `IDictionary<K, V>` when you need to mutate.

---

## Choosing parameter types

The general rule: **accept the least-capable interface that works**.

| Need | Parameter type |
|---|---|
| Just iterate (no count, no index) | `IEnumerable<T>` |
| Iterate + know count, no mutation | `IReadOnlyCollection<T>` |
| Iterate + count + indexed read | `IReadOnlyList<T>` |
| Iterate + mutate | `ICollection<T>` |
| Iterate + mutate + indexed | `IList<T>` |
| Sets specifically | `ISet<T>` / `IReadOnlySet<T>` |
| Key-value lookup | `IReadOnlyDictionary<K, V>` / `IDictionary<K, V>` |

The looser the type, the more callers can pass. Don't ask for `List<T>` if `IEnumerable<T>` suffices.

---

## Choosing return types

The general rule: **return a type that signals what the caller can do**.

| Want callers to... | Return type |
|---|---|
| Iterate, possibly lazy | `IEnumerable<T>` (document if deferred) |
| Iterate, know it's materialized | `IReadOnlyList<T>` |
| Mutate (rare) | `List<T>` or `IList<T>` |
| Snapshot of a key-value lookup | `IReadOnlyDictionary<K, V>` |

For repositories or services: **`IReadOnlyList<T>` is usually the right return**. Callers can iterate, count, and index — but can't accidentally mutate the underlying storage.

Avoid returning `IEnumerable<T>` from public methods unless you specifically mean "this is a deferred query, materialize it yourself when ready." Otherwise callers might enumerate it many times without realizing.

---

## Common patterns

### Accept the least, return the most useful

```csharp
public IReadOnlyList<User> FilterActive(IEnumerable<User> users) =>
    users.Where(u => u.IsActive).ToList();
```

Accept anything iterable. Return a snapshot they can iterate and index.

### Defensive return — don't leak internal mutation

```csharp
public class Service {
    private readonly List<Item> _items = new();

    // BAD — caller can mutate internal state
    public List<Item> Items => _items;

    // BETTER — read-only view
    public IReadOnlyList<Item> Items => _items;

    // STRICTEST — snapshot, caller can't see future mutations
    public IReadOnlyList<Item> ItemsSnapshot => _items.ToList();
}
```

The "read-only view" is usually the right balance. The list reference is the same; the interface restricts what callers can do.

### Defensive accept — copy if you'll keep a reference

```csharp
public class Team {
    private readonly List<Player> _players;
    public Team(IEnumerable<Player> players) {
        _players = new List<Player>(players);   // copy on input
    }
}
```

If you'll store a reference, copy on the way in. Otherwise, a caller mutating their list affects your state.

---

## Performance notes

- `foreach (var x in (IEnumerable<int>)list)` boxes the enumerator (interface-typed iteration). Direct `foreach (var x in list)` uses the concrete struct enumerator — zero boxes.
- Indexed access via `IList<T>` indexer goes through an interface call — slower than the concrete `List<T>` indexer (which the JIT inlines).
- For hot loops where you control both sides, use the concrete type for speed; use interfaces for flexibility.

---

## Variance

Type | Variance
---|---
`IEnumerable<out T>` | covariant
`IReadOnlyCollection<out T>` | covariant
`IReadOnlyList<out T>` | covariant
`IReadOnlyDictionary<TKey, out TValue>` | TValue covariant; TKey invariant
`ICollection<T>` | invariant (mutable)
`IList<T>` | invariant (mutable)
`ISet<T>` | invariant
`IDictionary<TKey, TValue>` | invariant

Read-only interfaces are covariant — `IReadOnlyList<string>` flows as `IReadOnlyList<object>`. Mutable interfaces are invariant — Add operations couldn't be safe.

---

## Common bugs

- **Modifying a `List<T>` returned as `IReadOnlyList<T>`** — caller can cast back and mutate. The interface restricts API but not raw access. If absolute safety needed, use `ImmutableList<T>` or copy on return.
- **`((ICollection<T>)readOnlyArr).Add(x)`** — arrays throw `NotSupportedException`. The interface exists but is non-functional for arrays.
- **Returning `IEnumerable<T>` from a deferred LINQ chain** — caller enumerates multiple times, work repeats. Materialize and return `IReadOnlyList<T>` instead.
- **Using `IList<T>` when `IReadOnlyList<T>` would do** — implicitly invites callers to mutate.

---

## A practical mental model

For your code, default to:
- **Parameters**: `IEnumerable<T>` (most flexible) unless you need indexing or count.
- **Returns**: `IReadOnlyList<T>` (materialized, indexable, can't be mutated by caller).
- **Internal storage**: concrete types (`List<T>`, `Dictionary<K, V>`) — maximum API surface for your own code.

For dictionaries:
- **Parameters**: `IReadOnlyDictionary<K, V>` or `IDictionary<K, V>`.
- **Returns**: `IReadOnlyDictionary<K, V>` for snapshots; concrete `Dictionary<K, V>` for "you can have this and modify it."

Be consistent within your codebase. If everyone uses different conventions, the interface choice becomes noise.

---

## Quick reference

```csharp
// Most flexible parameter (almost anything iterable)
public int Sum(IEnumerable<int> nums) { ... }

// Read-only collection with count
public string Describe(IReadOnlyCollection<int> nums) =>
    $"{nums.Count} items";

// Read-only indexed
public T Median<T>(IReadOnlyList<T> sorted) =>
    sorted[sorted.Count / 2];

// Mutable
public void Append(ICollection<int> nums, int x) =>
    nums.Add(x);

// Mutable indexed
public void Sort(IList<int> nums) { ... }

// Read-only return — caller iterates, indexes, doesn't mutate
public IReadOnlyList<User> GetActiveUsers() => /* ... */.ToList();

// Lookup snapshot
public IReadOnlyDictionary<int, User> ByIdSnapshot() => /* ... */.ToDictionary(u => u.Id);
```

---

That's Chapter 07. You should now know:
- Every BCL collection and when to use which.
- The performance characteristics: O(1) vs O(log n) vs O(n).
- How `Dictionary` and `HashSet` actually work (hashing, buckets, equality).
- When to choose immutable, frozen, or sorted variants.
- The equality contract — the rules that keep hash collections working.
- The interface hierarchy and how to pick parameter / return types.

→ [Questions.md](Questions.md)
