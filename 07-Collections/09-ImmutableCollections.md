# Immutable Collections

> `System.Collections.Immutable` — `ImmutableList`, `ImmutableArray`, `ImmutableDictionary`, `ImmutableHashSet`, and friends. Mutations return a **new** collection, sharing structure with the old.

```csharp
ImmutableList<int> a = ImmutableList.Create(1, 2, 3);
ImmutableList<int> b = a.Add(4);     // new list { 1, 2, 3, 4 } — a unchanged
ImmutableList<int> c = a.Add(5);     // new list { 1, 2, 3, 5 } — both a and b unchanged

Console.WriteLine(a.Count);   // 3
Console.WriteLine(b.Count);   // 4
Console.WriteLine(c.Count);   // 4
```

`a`, `b`, and `c` are independent. Mutations don't affect each other. Internally, they share most of their structure to avoid full copies.

---

## Why they exist

Mutable collections are convenient but they're **shared state** — passing a `List<T>` to a method means that method can mutate your list. Concurrency adds locking complexity. Returning a "snapshot" requires defensive copying.

Immutable collections solve all of that:
- **Thread-safe by construction** — no locks needed for reads or "writes" (which produce new instances).
- **Snapshot semantics** — passing an immutable collection guarantees the caller can't change it from under you.
- **Time-travel** — keep old versions cheaply via structural sharing.
- **Predictable behavior** — `a == b` after some operation if the operation didn't change anything.

Trade-off: per-operation cost is higher (O(log n) for typical operations vs O(1) for mutable equivalents). For read-mostly or concurrency-heavy code, the trade-off is worth it.

---

## The family

| Immutable type | Mutable counterpart | Backing structure |
|---|---|---|
| `ImmutableArray<T>` | `T[]` | Frozen array — fastest read, expensive mutation |
| `ImmutableList<T>` | `List<T>` | 2-3 tree — O(log n) ops, structural sharing |
| `ImmutableHashSet<T>` | `HashSet<T>` | HAMT (hash-array-mapped trie) — O(log n) |
| `ImmutableSortedSet<T>` | `SortedSet<T>` | Immutable AVL tree |
| `ImmutableDictionary<K, V>` | `Dictionary<K, V>` | HAMT |
| `ImmutableSortedDictionary<K, V>` | `SortedDictionary<K, V>` | Immutable AVL tree |
| `ImmutableQueue<T>` | `Queue<T>` | Linked structure |
| `ImmutableStack<T>` | `Stack<T>` | Linked structure |

Each is in `System.Collections.Immutable`, an in-box assembly since .NET Standard 1.

---

## Creation

```csharp
// Factory methods on static class
ImmutableList<int> a = ImmutableList.Create<int>();
ImmutableList<int> b = ImmutableList.Create(1, 2, 3);
ImmutableList<int> c = ImmutableList.CreateRange(someEnumerable);

// LINQ-style
ImmutableList<int> d = someEnumerable.ToImmutableList();
ImmutableArray<int> e = someEnumerable.ToImmutableArray();
ImmutableDictionary<string, int> f = pairs.ToImmutableDictionary(p => p.Key, p => p.Value);

// Or just an empty one
var empty = ImmutableList<int>.Empty;     // singleton, no allocation
```

---

## Mutation returns new

The fundamental pattern:

```csharp
ImmutableList<int> a = ImmutableList.Create(1, 2, 3);
ImmutableList<int> b = a.Add(4);
ImmutableList<int> c = a.RemoveAt(0);
ImmutableList<int> d = a.SetItem(1, 99);

// a is unchanged in all cases
```

Common methods:
- `Add`, `AddRange`, `Insert`, `InsertRange`
- `Remove`, `RemoveAt`, `RemoveAll`, `RemoveRange`
- `SetItem(index, value)`, `Replace(oldVal, newVal)`
- `Clear`
- All return a new immutable list.

For ImmutableDictionary:
- `SetItem(key, value)` — add or replace.
- `Add(key, value)` — throws if key already present.
- `Remove(key)`.

---

## ImmutableArray vs ImmutableList

Both are immutable. But fundamentally different storage:

### ImmutableArray<T>
- Backed by a regular `T[]`.
- Reads are **as fast as array reads**.
- Mutations **allocate a new array and copy everything** — O(n) per op.

```csharp
ImmutableArray<int> arr = ImmutableArray.Create(1, 2, 3);
var added = arr.Add(4);   // allocates a 4-element array and copies
```

Best for **build once, read many** scenarios. Frozen data.

### ImmutableList<T>
- Backed by a balanced tree (variant of 2-3 tree).
- Reads are **O(log n)** — slower than array.
- Mutations are **O(log n)** with structural sharing — much faster than ImmutableArray.

```csharp
ImmutableList<int> list = ImmutableList.Create(1, 2, 3);
var added = list.Add(4);   // O(log n), shares most of list's structure
```

Best for **mutation-heavy** immutable workflows.

**Rule of thumb**: `ImmutableArray<T>` for "read-only constant table"; `ImmutableList<T>` for "I'll be transforming this a lot."

---

## Builders for batch mutation

Adding 1M items to an `ImmutableList<T>` one at a time:

```csharp
ImmutableList<int> list = ImmutableList<int>.Empty;
for (int i = 0; i < 1_000_000; i++) list = list.Add(i);
// 1M new immutable lists allocated! Slow.
```

For batch builds, use a **builder** — a mutable buffer that produces an immutable result at the end:

```csharp
var builder = ImmutableList.CreateBuilder<int>();
for (int i = 0; i < 1_000_000; i++) builder.Add(i);
ImmutableList<int> list = builder.ToImmutable();
```

The builder is mutable; the final `ToImmutable()` freezes it into an immutable form. Single allocation. Same total cost as building a `List<T>`.

Every immutable collection has a corresponding builder. Use them for batch construction.

---

## Structural sharing

For trees-backed immutables (`ImmutableList`, `ImmutableDictionary`, etc.), most of the tree is **shared** between versions.

Consider:

```
a = [1, 2, 3, 4, 5]   (tree of 5 nodes)
b = a.Add(6)          (tree of 6 nodes, shares 4 nodes with a)
```

Only the path from root to the new leaf changes. The other paths are shared. So `Add` is O(log n) memory **and** time, not O(n).

This is called **persistent data structures**. Each version is independently usable, and total memory across versions is much smaller than copying.

---

## Equality

Immutable collections support **value equality** for comparing contents:

```csharp
ImmutableArray<int> a = ImmutableArray.Create(1, 2, 3);
ImmutableArray<int> b = ImmutableArray.Create(1, 2, 3);
a == b;                            // false — different instances
a.SequenceEqual(b);                 // true — same contents
```

`==` is reference equality (mostly). For value comparison, use `SequenceEqual`. For `ImmutableArray<T>`, two instances **can** be the same wrapped array via factory `ImmutableArray.Create(arr)` — be aware.

For records containing immutable collections, the record's synthesized equality uses `EqualityComparer<T>.Default` on the collection — which is reference equality. To get value semantics in records, override `Equals` or use a wrapper type.

---

## Thread safety

**100% thread-safe** by construction. Multiple threads can read any immutable collection concurrently. Mutations return new instances; the old instance is still valid for other threads.

```csharp
// Thread A
var addedA = original.Add(item1);

// Thread B (simultaneous)
var addedB = original.Add(item2);

// All three (original, addedA, addedB) are valid and independent.
```

For a **shared mutable reference** to an immutable collection (common pattern):

```csharp
private ImmutableList<Item> _items = ImmutableList<Item>.Empty;

public void Add(Item item) {
    ImmutableList<Item> current, updated;
    do {
        current = _items;
        updated = current.Add(item);
    } while (Interlocked.CompareExchange(ref _items, updated, current) != current);
}
```

CAS-loop pattern — try to publish a new version; if another thread beat you, retry with the new state. Lock-free.

`ImmutableInterlocked.Update` does this for you:

```csharp
ImmutableInterlocked.Update(ref _items, items => items.Add(item));
```

---

## Common patterns

### Snapshot for reading

```csharp
public class Cache {
    private ImmutableDictionary<string, User> _cache = ImmutableDictionary<string, User>.Empty;

    public User? Get(string key) => _cache.GetValueOrDefault(key);
    public void Set(string key, User user) =>
        ImmutableInterlocked.Update(ref _cache, d => d.SetItem(key, user));
}
```

Readers don't lock. Writers atomically swap to a new version.

### Persistent data history

```csharp
public class History<T> {
    private readonly Stack<ImmutableList<T>> _versions = new();

    public void Snapshot(ImmutableList<T> list) => _versions.Push(list);
    public ImmutableList<T> Restore() => _versions.Pop();
}
```

Each version is a complete state — keep as many as you want without exploding memory (thanks to sharing).

### Functional updates

```csharp
var updated = users
    .Where(u => u.IsActive)
    .Aggregate(ImmutableList<UserSummary>.Empty,
        (acc, u) => acc.Add(new UserSummary(u.Id, u.Name)));
```

Pure functional style. Useful in event-sourcing, redux-pattern UIs, etc.

---

## Internals — under the hood

### ImmutableList<T>
Backed by a height-balanced AVL-ish tree:

```
         (3)
        /   \
      (1)   (5)
            / \
           (4) (6)
```

Each node has two children + the value. To add, walk to the right place, allocate new nodes along the path, leave other nodes shared. O(log n) time, O(log n) memory per op.

### ImmutableDictionary<K, V>
HAMT — hash-array-mapped trie. The hash is split into 5-bit chunks; each chunk indexes a node's children. Effective branching factor 32, so trees are very shallow (~log₃₂(n)).

### ImmutableArray<T>
Just a wrapper struct around a `T[]`. Every mutation allocates a new array.

### Persistent stacks/queues
Linked structures — Push prepends a new node, sharing the rest. O(1) Push and Pop.

---

## Performance vs mutable counterparts

| Operation | Mutable | Immutable (tree-based) | ImmutableArray |
|---|---|---|---|
| Read | O(1) | O(log n) | O(1) |
| Add | O(1) amortized | O(log n) | O(n) |
| Remove | O(1) ~ O(n) | O(log n) | O(n) |
| Snapshot | O(n) copy | O(1) — reference | O(1) — reference |
| Thread safety | manual | built-in | built-in |

For workloads with frequent reads + occasional swaps, immutables are extremely competitive. For tight inner loops with millions of mutations, mutables win.

---

## Common bugs

- **Forgetting to capture the result** — `list.Add(x)` doesn't modify `list`; the return value has the change.

```csharp
var list = ImmutableList.Create(1);
list.Add(2);             // result discarded! list is still [1]
list = list.Add(2);      // ✓
```

- **Using `Add` in a hot loop** — creates one new list per iteration. Use a Builder.
- **Treating `ImmutableArray<T> == ImmutableArray<T>`** as value equality — it's reference.
- **Modifying through `CollectionsMarshal.AsSpan`** — Some immutables expose underlying data; modifying through low-level APIs violates immutability invariants. Don't do it.

---

## When to use immutable collections

✓ Shared state read by many threads, occasionally updated.
✓ Functional / event-sourcing style applications.
✓ APIs that should signal "this is a snapshot, don't mutate."
✓ Configuration tables built at startup, read forever.
✓ Time-travel debugging / state history.

✗ Performance-critical inner loops with many mutations.
✗ When `IReadOnlyList<T>` over a mutable list would suffice (don't enforce immutability if you don't need it).
✗ Very small collections — overhead is more than the benefit.

→ Next: [10-FrozenCollections.md](10-FrozenCollections.md)
