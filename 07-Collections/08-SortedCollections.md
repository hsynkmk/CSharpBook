# Sorted Collections

> `SortedDictionary<TKey, TValue>`, `SortedSet<T>`, and `SortedList<TKey, TValue>`. Three different ways to keep things in order at all times.

When you need **ordered iteration** (or fast min/max/range queries), these are the tools. Each has different performance trade-offs.

---

## The three sorted collections

| | `SortedDictionary<K, V>` | `SortedSet<T>` | `SortedList<K, V>` |
|---|---|---|---|
| Backing structure | Red-Black tree | Red-Black tree | Sorted `T[]` + `V[]` |
| Add | O(log n) | O(log n) | O(n) (shift) |
| Remove | O(log n) | O(log n) | O(n) (shift) |
| Lookup (ContainsKey / Contains) | O(log n) | O(log n) | O(log n) (binary search) |
| Indexed access | n/a | n/a | O(1) by index |
| Memory per item | high (tree node) | high (tree node) | low (just T+V) |
| Iteration order | sorted by key | sorted | sorted by key |
| Min / Max | O(log n) | O(log n) | O(1) (first/last) |
| GetViewBetween (range) | n/a | O(log n) | n/a |

**Default choice**: `SortedDictionary<K, V>` for general-purpose ordered maps; `SortedSet<T>` for ordered sets with range queries; `SortedList<K, V>` when you build once + query many.

---

## `SortedDictionary<TKey, TValue>`

A balanced BST (red-black tree) keyed by `TKey`. Iteration is sorted by key. All operations O(log n).

```csharp
var sd = new SortedDictionary<string, int> {
    ["banana"] = 2,
    ["apple"] = 1,
    ["cherry"] = 3,
};

foreach (var (k, v) in sd) Console.WriteLine($"{k}: {v}");
// apple: 1
// banana: 2
// cherry: 3
```

Custom comparer:

```csharp
var sd = new SortedDictionary<string, int>(StringComparer.OrdinalIgnoreCase);
```

Reverse order:

```csharp
var sd = new SortedDictionary<int, string>(
    Comparer<int>.Create((a, b) => b.CompareTo(a)));
```

The API mirrors `Dictionary<K, V>`: `Add`, `Remove`, `ContainsKey`, `TryGetValue`, indexer. Difference: every operation is O(log n) instead of O(1) average, but iteration is sorted and Min/Max are cheap.

---

## `SortedSet<T>`

A red-black tree of unique items, no value side. Like `HashSet<T>` but sorted.

```csharp
var ss = new SortedSet<int> { 3, 1, 4, 1, 5, 9, 2, 6 };
// Stores {1, 2, 3, 4, 5, 6, 9} — duplicates dropped, sorted

ss.Min;             // 1 — O(log n)
ss.Max;             // 9 — O(log n)
ss.Contains(5);     // true — O(log n)

foreach (var x in ss) Console.WriteLine(x);   // sorted ascending
```

The standout feature: **range views**.

```csharp
var betweenThreeAndSix = ss.GetViewBetween(3, 6);
foreach (var x in betweenThreeAndSix) Console.WriteLine(x);   // 3, 4, 5, 6
```

`GetViewBetween` returns a sub-view that's **live** — mutating it mutates the underlying set. Useful for range queries that go through both — "all entries with key in [3, 6]."

```csharp
betweenThreeAndSix.Add(5);    // updates underlying set
ss.Contains(5);                // already true
```

### Use case: leaderboard

```csharp
var scores = new SortedSet<(int Score, string Player)>(
    Comparer<(int, string)>.Create((a, b) =>
        a.Item1 != b.Item1 ? b.Item1 - a.Item1 : a.Item2.CompareTo(b.Item2)));

scores.Add((100, "Alice"));
scores.Add((150, "Bob"));
scores.Add((75, "Carol"));

// Top 3
foreach (var entry in scores.Take(3)) Console.WriteLine(entry);
// (150, Bob), (100, Alice), (75, Carol)
```

Tuple comparison gives both ranking and tie-breaking. SortedSet keeps them in order as you add.

---

## `SortedList<TKey, TValue>`

A hybrid: keys and values in **parallel arrays**, kept sorted. Lookups are O(log n) (binary search), but inserts and removes are O(n) (array shift).

```csharp
var sl = new SortedList<string, int>();
sl.Add("banana", 2);
sl.Add("apple", 1);
sl.Add("cherry", 3);

sl["apple"];           // 1
sl.Keys;                // IList<string>: ["apple", "banana", "cherry"]
sl.Values;              // IList<int>:    [1, 2, 3]

sl.Keys[0];             // "apple" — indexed access to keys
sl.Values[0];           // 1 — indexed access to values

sl.IndexOfKey("banana");   // 1
```

### When to prefer SortedList over SortedDictionary

- **Read-mostly** after build — same lookup speed, much lower memory.
- **You need indexed access to keys or values** (`sl.Keys[i]`, `sl.Values[i]`).
- **Small N** — arrays are cache-friendly; for N < ~50, SortedList outperforms SortedDictionary even on Add.

### When SortedDictionary wins

- **Add-heavy** workloads — O(log n) vs O(n).
- **Large N with mixed reads/writes**.
- **You don't need indexed keys/values**.

---

## Common patterns

### Range lookup

```csharp
var ss = new SortedSet<int> { 1, 5, 10, 15, 20, 25 };
foreach (var x in ss.GetViewBetween(10, 20))
    Console.WriteLine(x);   // 10, 15, 20
```

For SortedDictionary, no built-in `GetViewBetween`. Either iterate and break, or use LINQ:

```csharp
foreach (var (k, v) in sd.SkipWhile(p => p.Key.CompareTo(low) < 0)
                          .TakeWhile(p => p.Key.CompareTo(high) <= 0))
    Console.WriteLine($"{k}: {v}");
```

LINQ is less efficient — full enumeration with predicate per item. For frequent range queries, SortedSet is the better tool.

### Min / Max

```csharp
sortedSet.Min;     // O(log n) — follows left spine
sortedSet.Max;     // O(log n) — follows right spine
sortedDict.First();   // O(log n) for the first KVP
sortedList.Keys[0];   // O(1) — first index
sortedList.Keys[^1];  // O(1) — last index
```

### Ordered iteration during traversal

```csharp
foreach (var (k, v) in sd) {
    if (k > "m") break;   // stop once past target range
    Process(k, v);
}
```

SortedDictionary's iteration is in-order. Use it to scan a range with early termination.

---

## Internals — Red-Black tree

`SortedDictionary` and `SortedSet` use a **red-black tree**:
- Self-balancing binary search tree.
- Rotations on insert/delete maintain the balance invariant.
- Height ≤ 2 log₂(n + 1) — operations stay O(log n).

Each tree node holds:
- Color (red/black, 1 bit).
- Key.
- Value (for Dictionary; not for Set).
- Left, Right, Parent pointers.

For 64-bit, that's ~50+ bytes per node. For a SortedSet<int> with 1M items, ~50 MB of bookkeeping vs ~30 MB for HashSet<int> vs ~4 MB for List<int>.

That memory overhead is the cost of sorted access + O(log n) guarantees.

### IL / runtime behavior

The internal tree class is `SortedSet<T>.Node` / similar. Comparisons go through `IComparer<TKey>.Compare`, which for `Comparer<T>.Default` resolves to:
- For `IComparable<T>` types: direct call.
- For other types: `Comparer<T>.Default` is a singleton that uses the type's IComparable implementation.

Cost per comparison varies by T. Strings are slow (locale-aware unless using Ordinal); ints/longs are fast.

For performance-critical sorted collections of complex types, supply a hand-tuned `IComparer<T>`.

---

## Common bugs

- **Mutating a key after insertion** — like Dictionary, if the comparison-relevant fields change, the tree's invariants break.
- **Using SortedDictionary when SortedList would suffice** — pay the memory + tree overhead unnecessarily.
- **Treating `Keys` / `Values` as snapshots** — they're **live views**. Modifying the sorted collection reflects in the view.
- **Comparing across cultures** — string sort order varies by locale. Use `StringComparer.Ordinal` for stable sorting.

---

## When to use which

| Need | Use |
|---|---|
| Ordered map, frequent mutations | `SortedDictionary<K, V>` |
| Ordered set with range queries | `SortedSet<T>` |
| Read-mostly ordered map, low memory | `SortedList<K, V>` |
| Top-K / priority-based | `PriorityQueue<T, P>` |
| Hash-based map (unordered, fastest avg) | `Dictionary<K, V>` |
| Hash-based set | `HashSet<T>` |
| Read-only, ordered | `ImmutableSortedDictionary<K, V>` / `ImmutableSortedSet<T>` |

---

## Performance summary

For a workload of **N inserts + M lookups**:

| Collection | Insert cost | Lookup cost | Total |
|---|---|---|---|
| `SortedDictionary` | N log N | M log N | (N+M) log N |
| `SortedList` | N² | M log N | N² + M log N |
| `Dictionary` | N | M | N + M |

If M >> N, SortedList becomes competitive. If N ≈ M, SortedDictionary dominates SortedList for large N. For pure speed without sort, Dictionary wins by a factor of log N.

→ Next: [09-ImmutableCollections.md](09-ImmutableCollections.md)
