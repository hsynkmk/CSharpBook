# Chapter 07 — Coding Problems

> 15 hands-on collections problems. Each tests a real choice you'd make in production code.

---

## Problem 1 — Implement an LRU cache

Build `LruCache<TKey, TValue>` with capacity, O(1) Get + Set, automatic eviction of least-recently-used items.

<details><summary>Solution</summary>

```csharp
public class LruCache<TKey, TValue> where TKey : notnull {
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey K, TValue V)>> _map;
    private readonly LinkedList<(TKey K, TValue V)> _order;

    public LruCache(int capacity) {
        _capacity = capacity;
        _map = new(capacity);
        _order = new();
    }

    public bool TryGet(TKey key, out TValue value) {
        if (_map.TryGetValue(key, out var node)) {
            _order.Remove(node);
            _order.AddFirst(node);
            value = node.Value.V;
            return true;
        }
        value = default!;
        return false;
    }

    public void Set(TKey key, TValue value) {
        if (_map.TryGetValue(key, out var node)) {
            _order.Remove(node);
            node.Value = (key, value);
            _order.AddFirst(node);
            return;
        }

        if (_map.Count == _capacity) {
            var oldest = _order.Last!;
            _map.Remove(oldest.Value.K);
            _order.RemoveLast();
        }

        var newNode = _order.AddFirst((key, value));
        _map[key] = newNode;
    }

    public int Count => _map.Count;
}

var cache = new LruCache<string, int>(3);
cache.Set("a", 1); cache.Set("b", 2); cache.Set("c", 3);
cache.Set("d", 4);   // evicts "a"
cache.TryGet("a", out _);   // false
cache.TryGet("b", out _);   // true — also bumps b to most recent
```

Dictionary for O(1) lookup; LinkedList for O(1) move-to-front and evict-from-back.

</details>

---

## Problem 2 — Count word frequencies efficiently

Given an `IEnumerable<string>`, return a `Dictionary<string, int>` of word frequencies. Use the most efficient pattern.

<details><summary>Solution</summary>

```csharp
public static Dictionary<string, int> Count(IEnumerable<string> words) {
    var counts = new Dictionary<string, int>();
    foreach (var w in words) {
        ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(counts, w, out _);
        count++;
    }
    return counts;
}
```

Single hash lookup per word — gets a ref to the slot, increments in place.

Without `CollectionsMarshal`, the natural approach is:
```csharp
counts[w] = counts.TryGetValue(w, out var c) ? c + 1 : 1;
```
Two hash lookups (TryGetValue + indexer). Slower in tight loops but more readable.

</details>

---

## Problem 3 — Find duplicates in a sequence

Given an `IEnumerable<T>`, return the items that appear more than once (each duplicate listed once).

<details><summary>Solution</summary>

```csharp
public static List<T> FindDuplicates<T>(IEnumerable<T> source) where T : notnull {
    var seen = new HashSet<T>();
    var duplicates = new HashSet<T>();
    foreach (var item in source) {
        if (!seen.Add(item)) duplicates.Add(item);
    }
    return duplicates.ToList();
}
```

Two passes-worth of hashing, single iteration. `seen.Add` returns false if already there → duplicate found.

LINQ equivalent (less efficient — multiple passes):
```csharp
source.GroupBy(x => x).Where(g => g.Count() > 1).Select(g => g.Key).ToList();
```

</details>

---

## Problem 4 — Top-K largest

Implement `TopK<T>` that returns the K largest items from an `IEnumerable<T>`. O(n log K) time, O(K) memory.

<details><summary>Solution</summary>

```csharp
public static List<T> TopK<T>(IEnumerable<T> source, int k, IComparer<T>? comparer = null) {
    comparer ??= Comparer<T>.Default;
    var minHeap = new PriorityQueue<T, T>(comparer);
    foreach (var item in source) {
        if (minHeap.Count < k) minHeap.Enqueue(item, item);
        else if (comparer.Compare(item, minHeap.Peek()) > 0) {
            minHeap.DequeueEnqueue(item, item);
        }
    }
    var result = new List<T>(k);
    while (minHeap.Count > 0) result.Add(minHeap.Dequeue());
    result.Reverse();   // descending
    return result;
}

TopK(new[] { 3, 1, 4, 1, 5, 9, 2, 6 }, 3);   // {9, 6, 5}
```

Min-heap of size K. Push if not full; otherwise dequeue smallest and push new if it's bigger. At the end, drain — that gives ascending; reverse for descending.

</details>

---

## Problem 5 — Why does this leak entries?

```csharp
public class Coord {
    public int X, Y;
    public override bool Equals(object? o) => o is Coord c && c.X == X && c.Y == Y;
    public override int GetHashCode() => HashCode.Combine(X, Y);
}

var set = new HashSet<Coord>();
var p = new Coord { X = 1, Y = 2 };
set.Add(p);
p.X = 5;
set.Contains(p);   // ?
```

What does Contains return, and why?

<details><summary>Answer</summary>

**False.** When `set.Add(p)` was called, the hash was `HashCode.Combine(1, 2)` — say 5000. The entry went into bucket 5000 (mod bucket count).

When `set.Contains(p)` is called now, the hash is `HashCode.Combine(5, 2)` — say 8123. The set looks in bucket 8123. The entry is in bucket 5000 — invisible.

The entry **still exists** (it shows up in `foreach`) but `Contains` and `Remove` can't find it.

**Fix**: make Coord immutable (`readonly`, `init`-only properties). Or, even simpler, use a `readonly record struct`:
```csharp
public readonly record struct Coord(int X, int Y);
```

</details>

---

## Problem 6 — Group transactions by date and customer

Given `List<Transaction>` with `CustomerId`, `Date`, `Amount`, group by `(CustomerId, Date.Date)` and sum amounts.

<details><summary>Solution</summary>

```csharp
var summary = transactions
    .GroupBy(t => (t.CustomerId, t.Date.Date))
    .ToDictionary(g => g.Key, g => g.Sum(t => t.Amount));

// summary is Dictionary<(int CustomerId, DateTime Date), decimal>
foreach (var ((cid, date), total) in summary) {
    Console.WriteLine($"{cid} on {date:yyyy-MM-dd}: {total:C}");
}
```

Tuple as composite key. Works because tuples have value-based equality and hashing built in.

For EF Core: this entire query translates to `GROUP BY CustomerId, CAST(Date AS DATE)` SQL.

</details>

---

## Problem 7 — Choose the collection

For each scenario, pick the right collection:

a. A web app's permission set, loaded at startup, checked on every request.
b. The list of recent commands for a CLI's undo history (max 50).
c. A scheduler holding "next-to-run" tasks based on scheduled time.
d. A multiplayer game's leaderboard, sorted by score, updated frequently.
e. Storing config keys read from JSON.

<details><summary>Answer</summary>

a. **`FrozenSet<string>`** — built once, read on hot path.
b. **`Stack<T>`** (with manual capacity check) OR `LinkedList<T>` (allows trimming from the back).
c. **`PriorityQueue<Task, DateTime>`** — min-heap on scheduled time.
d. **`SortedSet<(int Score, string Player)>`** with a comparer — keeps items in order across updates.
e. **`FrozenDictionary<string, string>`** if loaded once; `Dictionary<string, string>` if it might be reloaded.

</details>

---

## Problem 8 — Implement a "frequency limiter"

Build a class `FrequencyLimiter` that tracks how many times each key has been seen in the last N seconds and answers "is this key under the limit?". Use the right data structures.

<details><summary>Solution</summary>

```csharp
public class FrequencyLimiter {
    private readonly Dictionary<string, Queue<DateTime>> _hits = new();
    private readonly TimeSpan _window;
    private readonly int _limit;

    public FrequencyLimiter(TimeSpan window, int limit) {
        _window = window; _limit = limit;
    }

    public bool TryAccept(string key) {
        var now = DateTime.UtcNow;
        if (!_hits.TryGetValue(key, out var q)) {
            q = new Queue<DateTime>();
            _hits[key] = q;
        }
        while (q.Count > 0 && now - q.Peek() > _window) q.Dequeue();
        if (q.Count >= _limit) return false;
        q.Enqueue(now);
        return true;
    }
}

var rl = new FrequencyLimiter(TimeSpan.FromSeconds(1), limit: 5);
for (int i = 0; i < 10; i++) Console.WriteLine(rl.TryAccept("user1"));
// First 5 → true; next 5 → false (until time passes)
```

Per-key queue of timestamps. Trim expired entries on each check. Total memory bounded by `limit × keys`.

For production: use `Microsoft.AspNetCore.RateLimiting` instead — pre-built, more sophisticated algorithms.

</details>

---

## Problem 9 — Implement Dijkstra's shortest path

Given a graph (nodes + weighted edges), find the shortest distance from a start node to all others.

<details><summary>Solution</summary>

```csharp
public Dictionary<TNode, int> Dijkstra<TNode>(
    TNode start,
    Func<TNode, IEnumerable<(TNode To, int Weight)>> neighbors
) where TNode : notnull {
    var dist = new Dictionary<TNode, int> { [start] = 0 };
    var pq = new PriorityQueue<TNode, int>();
    pq.Enqueue(start, 0);

    while (pq.TryDequeue(out var u, out var distU)) {
        if (distU > dist[u]) continue;   // skip stale
        foreach (var (v, w) in neighbors(u)) {
            int alt = distU + w;
            if (!dist.TryGetValue(v, out var current) || alt < current) {
                dist[v] = alt;
                pq.Enqueue(v, alt);
            }
        }
    }
    return dist;
}
```

`PriorityQueue` for next-to-process. `Dictionary` for distances. The "skip stale" check handles the lack of decrease-key in PriorityQueue.

</details>

---

## Problem 10 — Validate a configuration table at startup

You're loading 10,000 settings from JSON. After parsing into a `Dictionary<string, string>`, you'll read it millions of times. Optimize.

<details><summary>Solution</summary>

```csharp
// Load
Dictionary<string, string> raw = JsonSerializer.Deserialize<Dictionary<string, string>>(json)!;

// Validate
foreach (var (k, v) in raw) {
    if (string.IsNullOrWhiteSpace(k)) throw new InvalidOperationException();
    // ... other checks
}

// Freeze
public static readonly FrozenDictionary<string, string> Settings =
    raw.ToFrozenDictionary(StringComparer.OrdinalIgnoreCase);
```

Steps:
- Load to a regular Dictionary for build speed.
- Validate, with mutability still available.
- Freeze for fast reads on the hot path.
- Case-insensitive comparer for config-key flexibility.

</details>

---

## Problem 11 — Implement set difference, two ways

Given two large `HashSet<int>` instances, compute the items in `a` but not in `b`, two ways:
1. Using `set.ExceptWith` (mutating).
2. Using LINQ `Except` (non-mutating).

<details><summary>Solution</summary>

```csharp
var a = new HashSet<int> { 1, 2, 3, 4, 5 };
var b = new HashSet<int> { 3, 4, 5, 6, 7 };

// Mutating — a becomes the difference
var aCopy = new HashSet<int>(a);   // preserve original
aCopy.ExceptWith(b);
// aCopy: {1, 2}

// Non-mutating — returns a new HashSet
HashSet<int> diff = a.Except(b).ToHashSet();
// diff: {1, 2}
```

`ExceptWith` is faster for in-place — direct manipulation. `Except` is more functional — returns new sequence.

</details>

---

## Problem 12 — Spot the bug

```csharp
public IReadOnlyList<User> GetActiveUsers() {
    return _users.Where(u => u.IsActive);   // ⚠
}
```

<details><summary>Answer</summary>

`Where` returns `IEnumerable<User>`, not `IReadOnlyList<User>`. Compile error.

Fix:
```csharp
public IReadOnlyList<User> GetActiveUsers() {
    return _users.Where(u => u.IsActive).ToList();
}
```

Or, if you specifically want a snapshot the caller can iterate but not enumerate twice without seeing different results:

```csharp
public IReadOnlyList<User> GetActiveUsers() =>
    _users.Where(u => u.IsActive).ToList().AsReadOnly();
// AsReadOnly returns a ReadOnlyCollection wrapping the list — explicit immutability
```

</details>

---

## Problem 13 — Pick collection for thread safety

You're building a request counter — "how many times has key X been accessed?" Many threads will call `Increment(key)`. What's the right collection?

<details><summary>Solution</summary>

```csharp
private readonly ConcurrentDictionary<string, int> _counts = new();

public void Increment(string key) =>
    _counts.AddOrUpdate(key, 1, (_, old) => old + 1);

public int Get(string key) =>
    _counts.TryGetValue(key, out var c) ? c : 0;
```

`ConcurrentDictionary.AddOrUpdate` is atomic. For pure counter increments, this is enough.

For extreme performance with hot keys, consider per-thread counters that aggregate periodically (avoids contention on the lock-free CAS loop).

</details>

---

## Problem 14 — Compare two collection types' build performance

Build a `List<int>` of 1M items vs `ImmutableList<int>.Empty.Add` in a loop vs `ImmutableList.CreateBuilder + ToImmutable`. Predict times.

<details><summary>Answer</summary>

Approximate (.NET 10, decent hardware):

- `List<int>` Add in loop with pre-allocated capacity: ~5-10 ms.
- `List<int>` Add without capacity (resizes): ~10-15 ms.
- `ImmutableList<int>.Empty.Add` in loop: ~3-5 **seconds**! (Each Add allocates a new tree path.)
- `ImmutableList.CreateBuilder<int>` + Add + ToImmutable: ~10-20 ms (similar to List + final O(n) freeze).

**Lesson**: never `Add` to an `ImmutableList<T>` in a loop. Use a Builder.

</details>

---

## Problem 15 — Implement Distinct that handles a custom comparer

LINQ's `Distinct` uses `EqualityComparer<T>.Default`. Write your own `DistinctBy` (don't use the .NET 6+ built-in) that takes a key selector:

<details><summary>Solution</summary>

```csharp
public static IEnumerable<T> DistinctBy<T, TKey>(this IEnumerable<T> source, Func<T, TKey> keySelector) where TKey : notnull {
    var seen = new HashSet<TKey>();
    foreach (var item in source) {
        if (seen.Add(keySelector(item))) yield return item;
    }
}

var users = GetUsers();
var oneUserPerCountry = users.DistinctBy(u => u.Country).ToList();
```

`HashSet.Add` returns false if already present — we yield only on true. Single-pass, O(n) time, O(distinct keys) memory.

.NET 6+ has this built in (and uses the comparer pattern internally).

</details>

---

That's Chapter 07. You should now have strong intuition for picking collections based on access patterns and understanding why each performs the way it does.

→ [Chapter 08 — Concurrency](../08-Concurrency/README.md)
