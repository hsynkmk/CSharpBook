# LINQ Performance & Pitfalls

## Overview

LINQ is one of the most loved features in C# — and one of the easiest places to write code that's quietly slow. This file is a survey of the common traps, why they happen, how to fix them, and when LINQ is the wrong tool.

---

## The big traps

### 1. Multiple enumeration of a deferred query

```csharp
var query = users.Where(u => Expensive(u));   // deferred
int count = query.Count();                     // runs Expensive for every user
var list = query.ToList();                     // runs Expensive AGAIN for every user
```

**Fix**: materialize once.

```csharp
var matches = users.Where(u => Expensive(u)).ToList();
int count = matches.Count;
var list = matches;
```

For databases (IQueryable), multiple enumeration means multiple round trips. Same fix.

---

### 2. `Count() > 0` instead of `Any()`

```csharp
if (users.Count() > 0) { ... }    // iterates the whole sequence
if (users.Any()) { ... }           // short-circuits at the first item
```

For `List<T>` and arrays, `Count()` is O(1) (uses the underlying `.Count`). For a deferred query or `IEnumerable<T>` without a known size, `Count()` walks everything. Use `Any()` for existence tests.

For EF Core: `Count() > 0` translates to `SELECT COUNT(*) ...`; `Any()` translates to `SELECT EXISTS(...)` — much faster on the DB side.

---

### 3. Side effects in `Select` / `Where`

```csharp
var result = users.Select(u => { Log(u); return u.Name; });
foreach (var n in result) { /* ... */ }   // Log fires per item
foreach (var n in result) { /* ... */ }   // Log fires AGAIN per item
```

LINQ's contract assumes lambdas are pure. Don't log, count, mutate state, or write to disk inside them. Iterate explicitly for side effects:

```csharp
foreach (var u in users) {
    Log(u);
    Process(u.Name);
}
```

---

### 4. Allocating in tight loops

```csharp
foreach (var u in users) {
    var allOrders = u.Orders.Where(o => o.Active).ToList();   // allocates a new list per user
    Console.WriteLine(allOrders.Count);
}
```

`ToList()` allocates. Inside a hot loop, this adds up. If you just need a count or the first match, skip materialization:

```csharp
foreach (var u in users) {
    int activeCount = u.Orders.Count(o => o.Active);   // no allocation
}
```

---

### 5. LINQ-to-Objects vs LINQ-to-Entities confusion

```csharp
var users = db.Users
    .Where(u => u.IsActive)
    .ToList()                    // ← pulled all active users to memory
    .Where(u => u.Country == "US")   // now LINQ-to-Objects — but already paid SQL cost
    .ToList();
```

Once you call `ToList()`, you're in C# memory — but `.Where(u => u.Country == "US")` could have been a SQL filter, reducing the rows fetched. Push filters down.

```csharp
var users = db.Users
    .Where(u => u.IsActive && u.Country == "US")
    .ToList();
```

---

### 6. Forgetting that `OrderBy` materializes

```csharp
var seq = source.OrderBy(x => x.Key);
seq.First();    // sorts the WHOLE source to find the first item
```

For "first by key," use `MinBy` (.NET 6+):

```csharp
var first = source.MinBy(x => x.Key);   // O(n), single pass
```

For "top N," use `OrderByDescending().Take(N)` (the BCL detects this and partially sorts) OR a priority queue.

---

### 7. `Where`-after-`OrderBy` flips the type

```csharp
var sorted = source.OrderBy(x => x.Key);                 // IOrderedEnumerable<T>
var filtered = sorted.Where(x => x.Active);              // back to IEnumerable<T>
var thenBy = filtered.ThenBy(x => x.Name);               // ❌ — ThenBy needs IOrderedEnumerable
```

After `Where`, you've lost the "I'm already sorted" type. Do all sorts together:

```csharp
var sorted = source
    .Where(x => x.Active)
    .OrderBy(x => x.Key)
    .ThenBy(x => x.Name);
```

---

### 8. `First()` throws on empty

```csharp
var first = list.First(u => u.IsAdmin);   // throws InvalidOperationException if none
```

Use `FirstOrDefault` when no match is possible:

```csharp
var first = list.FirstOrDefault(u => u.IsAdmin);
if (first is null) { /* handle */ }
```

Same pattern: `Last` vs `LastOrDefault`, `Single` vs `SingleOrDefault`, `ElementAt` vs `ElementAtOrDefault`.

---

### 9. `Single` when you meant `First`

```csharp
var u = users.Single(u => u.Country == "US");   // throws if 0 OR more than 1
```

`Single` is for "exactly one is expected — if not, it's a bug." It's slower than `First` (must scan past the match to verify uniqueness). Use only when you really mean it.

---

### 10. Multiple LINQ-to-Entities `.Where()` calls

EF Core combines them, but if some are on `IEnumerable<T>` (after `.AsEnumerable()`) and some on `IQueryable<T>`, the order matters:

```csharp
db.Users.AsEnumerable().Where(u => u.IsActive).Where(u => u.Country == "US").ToList();
// Pulls ALL users (no SQL filtering), filters in memory.

db.Users.Where(u => u.IsActive).Where(u => u.Country == "US").AsEnumerable().ToList();
// Both filters in SQL, then materialize. Much better.
```

---

## Memory / allocation pitfalls

### Iterator allocations

Each LINQ operator typically allocates one **state-machine object** per query. A chain of `.Where().Select().OrderBy()` allocates 3 — small but real.

For tight inner loops where this matters, hand-write the iteration.

### Delegate allocations

Each lambda is a delegate. If captured, a closure too.

```csharp
public void Process(int min) {
    items.Where(x => x > min).ToList();   // closure capturing min
}
```

For very hot paths:
- Use a method instead of a lambda where possible.
- Mark lambdas `static` when they don't capture.
- Pass state through method parameters explicitly.

### Materialization allocations

`ToList`, `ToArray`, `ToDictionary` all allocate. For multi-megabyte data, consider streaming with `IEnumerable<T>` instead of materializing.

---

## When LINQ is the wrong tool

### Hot inner loops

For tight numeric code over arrays:

```csharp
// LINQ: ~5x slower than hand-coded for-loop
int sum = arr.Sum();

// Hand-coded
int sum = 0;
for (int i = 0; i < arr.Length; i++) sum += arr[i];
```

LINQ's per-element overhead (delegate, iterator) adds up at scale. For algorithm hot paths, hand-write the loop.

### Side-effect-heavy operations

LINQ is for transformations. For "do X for each item," `foreach` is clearer:

```csharp
foreach (var x in items) {
    Log(x);
    Process(x);
}
```

vs:

```csharp
items.Select(x => { Log(x); return x; }).ToList();   // wrong tool
```

### Massively parallel operations

PLINQ exists (`.AsParallel().Where(...)`) but the threshold for it to pay off is high. For pure CPU work over millions of items, `Parallel.ForEach` or hand-rolled parallelism is often better.

---

## Operator-specific performance notes

### `Where` after `OrderBy`

EF Core can often push `Where` into the SQL even after OrderBy. LINQ-to-Objects can't — the sort has happened.

### `Take(n)` after `OrderBy`

The BCL detects this and uses a partial sort (top-N) for in-memory. EF Core translates to `LIMIT n`/`TOP n` — much faster than full sort + take.

### `GroupBy` materializes

`GroupBy` must see all items to form groups. It's O(n) memory and time. For aggregation-only patterns, use `ToLookup` or projecting `Select` instead.

### `Distinct` materializes

Internally builds a `HashSet<T>`. Memory cost = number of distinct items.

### `Reverse` materializes

Has to see the entire input to flip it. Use `OrderByDescending` to sort in reverse without materialization (for IQueryable, translates to `ORDER BY ... DESC`).

### `Last` / `LastOrDefault`

For arbitrary `IEnumerable<T>`: walks the whole sequence.
For `IList<T>` (List, array): O(1) — uses the indexer.

### `ElementAt`

For `IList<T>`: O(1).
For others: O(n).

### `Contains`

For `HashSet<T>`: O(1).
For `List<T>`: O(n).
For `IQueryable<T>`: translates to SQL `IN (...)` if the comparison list is small.

---

## Benchmark methodology

If you suspect LINQ is slow:

1. **Measure** with BenchmarkDotNet, not by intuition.
2. **Compare** LINQ vs hand-rolled. If LINQ is within 10%, leave it (readability wins).
3. **Profile** memory with `[MemoryDiagnoser]`.
4. **Look at the IL** (`SharpLab.io`) — sometimes you'll spot the iterator chain.

Example:

```csharp
[MemoryDiagnoser]
public class SumBenchmark {
    private readonly int[] _data = Enumerable.Range(0, 10_000).ToArray();

    [Benchmark]
    public int LinqSum() => _data.Sum();

    [Benchmark]
    public int ManualSum() {
        int sum = 0;
        for (int i = 0; i < _data.Length; i++) sum += _data[i];
        return sum;
    }
}
```

Typically:
- LinqSum: ~10–50 µs, some allocations.
- ManualSum: ~5–10 µs, zero allocations.

Verdict: if you're calling this once per request, **doesn't matter** — write `Sum()`. If you're calling it 10M times per second, hand-roll.

---

## The OrderBy-then-First trap (continued)

```csharp
var firstActiveByDate = users
    .OrderBy(u => u.CreatedAt)
    .First(u => u.IsActive);   // ← sorts ALL users, then linearly scans for IsActive
```

The OrderBy doesn't know that you only want the first IsActive. It sorts everything.

**Fix 1** — filter first:

```csharp
users.Where(u => u.IsActive).OrderBy(u => u.CreatedAt).First();
```

Sorts only the active users — usually much fewer.

**Fix 2** — use `MinBy`:

```csharp
users.Where(u => u.IsActive).MinBy(u => u.CreatedAt);
```

O(n), no sort.

For EF Core: it's smart enough to push the filter and use `TOP 1 ... ORDER BY` regardless of order. For LINQ-to-Objects, filter first.

---

## Returning IEnumerable from public APIs — be careful

```csharp
public IEnumerable<User> GetActives() => db.Users.Where(u => u.IsActive);
```

The caller gets a deferred query. Multiple enumeration → multiple SQL queries. **Document** or materialize:

```csharp
public List<User> GetActives() => db.Users.Where(u => u.IsActive).ToList();
```

Or expose the IQueryable so callers can compose:

```csharp
public IQueryable<User> ActiveUsers() => db.Users.Where(u => u.IsActive);
// Caller: ActiveUsers().Where(u => u.Country == "US").ToList();
```

Either is fine — just be consistent and clear.

---

## Common-sense rules

1. **Don't optimize prematurely.** Write the clearest LINQ first.
2. **Profile before refactoring.** Most LINQ code is fast enough.
3. **Materialize once per logical unit.** Don't enumerate a deferred query repeatedly.
4. **Push filters as early as possible.** Especially for IQueryable.
5. **For tight inner loops, fall back to `for`/`foreach`.** Readability vs hot-path perf is a real trade-off.

---

## A perf checklist for query-heavy code

- [ ] Materialize multi-use queries (`ToList()` once).
- [ ] Use `Any()` not `Count() > 0`.
- [ ] Filter before sort.
- [ ] For "top N," `OrderByDescending().Take(N)` or a priority queue.
- [ ] For "single item by key," `MinBy`/`MaxBy`, not OrderBy + First.
- [ ] For databases, log generated SQL. Look for unexpected fan-out.
- [ ] Avoid `AsEnumerable` unless intentional.
- [ ] Cache invariant lookups (`ToDictionary` once, query many times).
- [ ] For hot paths, benchmark; if 5% of total runtime, leave it; if 50%, hand-roll.

---

## Performance summary table

| Issue | Cost | Fix |
|---|---|---|
| Multiple enumeration | N times the work | Materialize once |
| `Count() > 0` | O(n) walk | `Any()` (O(1) short-circuit) |
| Side effects in `Select` | Reruns each enum | Use `foreach` |
| Allocation in hot loop | GC pressure | Avoid `ToList`, use counts |
| `AsEnumerable` mid-query | Pulls all rows | Keep IQueryable until you must |
| Sort + First | O(n log n) | `MinBy` or filter-then-sort |
| Multiple SQL round trips | Latency | Combine queries, project sparingly |

→ Continue to: [Questions.md](Questions.md)
