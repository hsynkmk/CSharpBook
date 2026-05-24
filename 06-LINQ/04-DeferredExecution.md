# Deferred Execution

## What it is

Most LINQ operators **don't execute** when written. They build up a query that runs only when **enumerated** — when something calls `MoveNext()` on the resulting sequence (typically through `foreach`, `ToList`, `Count`, etc.).

```csharp
Console.WriteLine("about to define query");
var query = users.Where(u => {
    Console.WriteLine($"  checking {u.Name}");
    return u.IsActive;
});
Console.WriteLine("query defined, but nothing logged yet");

foreach (var u in query) {
    Console.WriteLine($"got {u.Name}");
}
```

Output:
```
about to define query
query defined, but nothing logged yet
  checking Alice
got Alice
  checking Bob
  checking Carol
got Carol
```

The `Where` lambda runs only during the `foreach` — and per item, interleaved with consuming code. This is **deferred** (lazy) execution.

Understanding when execution happens — and when it happens **multiple times** — is the single most important LINQ skill.

---

## Why it exists

Lazy evaluation makes LINQ pipelines composable and efficient:
- **Compose freely without paying cost** — build up a query in 5 stages, no work until you ask for results.
- **Short-circuit** — `query.First()` stops at the first match, not iterating the rest.
- **Stream-friendly** — process huge sequences (millions of items) without materializing them.
- **One pass through the source** — operators chain through a single iteration, not N passes.

The flip side: lazy queries can run **more than once** (each `foreach` re-executes), and they can run **at unexpected times** (like when serializing, or being inspected).

---

## Deferred operators

These build a query without running it. Each returns a new sequence object that, when enumerated, walks the input.

| Operator | Defers? |
|---|---|
| `Where` | ✓ |
| `Select`, `SelectMany` | ✓ |
| `OrderBy`, `ThenBy`, `OrderByDescending`, `ThenByDescending` | ✓ (but eagerly sort when first enumerated) |
| `GroupBy` | ✓ |
| `Join`, `GroupJoin` | ✓ |
| `Distinct`, `DistinctBy` | ✓ |
| `Skip`, `Take`, `SkipWhile`, `TakeWhile` | ✓ |
| `Concat`, `Union`, `Intersect`, `Except` | ✓ |
| `Reverse` | ✓ (must materialize once to flip) |
| `Cast<T>`, `OfType<T>` | ✓ |

The vast majority of LINQ operators defer.

---

## Materializing operators

These **execute the query right now** and return a result:

| Operator | Returns |
|---|---|
| `ToList`, `ToArray` | concrete collection |
| `ToDictionary`, `ToHashSet`, `ToLookup` | concrete collection |
| `Count`, `LongCount` | int / long (iterates all unless source has Count) |
| `Sum`, `Min`, `Max`, `Average`, `Aggregate` | scalar |
| `First`, `FirstOrDefault`, `Single`, `SingleOrDefault`, `Last`, `LastOrDefault` | scalar (short-circuit when possible) |
| `Any`, `All`, `Contains` | bool (short-circuit when possible) |
| `ElementAt`, `ElementAtOrDefault` | scalar |
| `MinBy`, `MaxBy` | scalar |
| `SequenceEqual` | bool |

After these, you have a concrete value or collection — no more deferred computation.

---

## The multi-enumeration trap

Because deferred queries re-execute on each enumeration:

```csharp
IEnumerable<int> source = SlowExpensiveSource();   // some expensive iterator

var query = source.Where(x => x > 0).Select(x => x * 2);

int sum = query.Sum();         // runs the whole chain
int count = query.Count();     // runs the WHOLE chain AGAIN
var list = query.ToList();     // and AGAIN
```

Three full traversals. Each `.Sum()`, `.Count()`, `.ToList()` re-iterates the source and re-runs the predicates.

For an in-memory `List<int>` source, this is wasteful but tolerable. For a database query (IQueryable), it means **three round trips to the database**. For a file reader source, it means **reading the file three times**.

### Fix — materialize once

```csharp
var query = source.Where(x => x > 0).Select(x => x * 2).ToList();  // one pass

int sum = query.Sum();
int count = query.Count;
var list = query;   // already materialized
```

After `ToList()`, the result is a concrete list. Subsequent operations iterate the list (fast, no re-running).

**Rule of thumb**: if you'll consume the query more than once, materialize it.

---

## Re-execution and side effects

Deferred operators evaluate their lambdas **every time** the query is enumerated.

```csharp
int counter = 0;
var query = Enumerable.Range(1, 5).Select(x => {
    counter++;
    return x * 2;
});

query.Count();    // counter → 5
query.Sum();      // counter → 10
query.ToList();   // counter → 15
```

The lambda fires per item per enumeration. Total: 15 invocations.

Avoid putting side effects inside `Select` / `Where`. They're meant to be **pure** (input → output, no other behavior). If you need a side effect, materialize and iterate:

```csharp
var items = query.ToList();
foreach (var item in items) {
    Log(item);
    Process(item);
}
```

---

## OrderBy is a hybrid

`OrderBy` defers — but when it's first enumerated, it must materialize **all** input to sort, then yield sorted items:

```csharp
var sorted = source.OrderBy(x => x.Key);
foreach (var x in sorted) { ... }  // entire source iterated, sorted, then yielded
```

Memory cost: O(N) for the sort buffer. Subsequent enumerations re-sort (unless materialized).

If you OrderBy a database query (IQueryable), the sort happens in SQL — no memory cost on the client.

---

## Short-circuit operators

Some materializing operators **don't enumerate everything**:

- `First()` — stops at the first item.
- `FirstOrDefault()` — same.
- `Any()` — stops at the first match.
- `Take(n)` — yields first n, stops.
- `Contains(x)` — stops at first match.

```csharp
bool hasAny = Enumerable.Range(1, 1_000_000)
    .Where(x => { Console.Write("."); return x > 0; })
    .Any();
// Prints only ONE dot — Any stops at the first match
```

For finding "is there one?" or "what's the first?", these are much cheaper than `Count() > 0`.

`Count() > 0` is a classic anti-pattern: it iterates the entire source. `Any()` is one comparison.

---

## Iterator detail — when execution happens

The fundamental contract: `IEnumerator<T>.MoveNext()` advances the iteration, and `Current` exposes the current item.

A deferred query exposes an `IEnumerator<T>` that, on each `MoveNext`, pulls one item through the chain:

```
Consumer.MoveNext()
  → Select.MoveNext()
    → Where.MoveNext()
      → Source.MoveNext()        // get next from source
      ← yields item to Where
    ← Where filters (calls predicate); if accept, yield up
  ← Select projects; yield up
← Consumer receives item
```

One item flows through the pipeline per `MoveNext`. No batch processing, no intermediate lists.

This is why LINQ to Objects is efficient: pipelined iteration, not multi-pass batch.

---

## Internals — the iterator state machine

Each deferred operator is implemented as an iterator method (using `yield return`). The compiler generates a state machine class:

```csharp
public static IEnumerable<T> Where<T>(IEnumerable<T> source, Func<T, bool> predicate) {
    foreach (var item in source)
        if (predicate(item))
            yield return item;
}
```

The compiler turns this into something like:

```csharp
private sealed class WhereIterator<T> : IEnumerator<T>, IEnumerable<T> {
    private readonly IEnumerable<T> _source;
    private readonly Func<T, bool> _predicate;
    private IEnumerator<T>? _sourceEnumerator;
    private int _state;

    public bool MoveNext() {
        switch (_state) {
            case 0:
                _sourceEnumerator = _source.GetEnumerator();
                _state = 1;
                goto case 1;
            case 1:
                while (_sourceEnumerator!.MoveNext()) {
                    if (_predicate(_sourceEnumerator.Current)) {
                        Current = _sourceEnumerator.Current;
                        return true;
                    }
                }
                _sourceEnumerator.Dispose();
                _state = -1;
                return false;
        }
        return false;
    }

    public T Current { get; private set; } = default!;
    // ... GetEnumerator, Dispose, etc.
}
```

Each call to `MoveNext` advances the state machine. Operators chain by composing these iterators.

The BCL's actual implementations are more sophisticated — they often have optimized fast paths for `List<T>`, arrays, and other specific source types (e.g., `Where(List<int>, predicate)` can avoid the enumerator allocation entirely).

### Allocation cost

Each deferred operator typically allocates one iterator object per query, plus enumerator state per enumeration. For a 5-operator chain:
- 5 iterator objects (once per query construction).
- 5 enumerators per `foreach` / `ToList`.
- 1 delegate per lambda.

For tight loops, this adds up. For typical code, it's negligible compared to the work done.

For ultra-hot paths, hand-written `for` loops avoid all of this. But you lose readability — measure before optimizing.

---

## Streams and infinite sequences

Deferred execution makes infinite sequences usable:

```csharp
public static IEnumerable<int> Naturals() {
    int n = 0;
    while (true) yield return n++;
}

var firstTen = Naturals().Take(10);
foreach (var n in firstTen) Console.WriteLine(n);
// 0 1 2 3 4 5 6 7 8 9
```

`Naturals()` runs forever in principle, but `Take(10)` short-circuits — only 10 items flow through. No infinite loop.

`Enumerable.Range` is similar in spirit:
```csharp
Enumerable.Range(0, int.MaxValue).Where(...).Take(5);
```

Without deferred execution, this would try to materialize 2 billion ints before applying `Take`.

---

## Common patterns

### Snapshot vs reference

```csharp
var list = new List<int> { 1, 2, 3 };

IEnumerable<int> query = list.Where(x => x > 0);   // not materialized

list.Add(4);

query.Count();   // 4 — query sees the modified list
```

The query holds a **reference** to `list`. Modifications to `list` are visible to the query.

To take a snapshot at query-creation time:

```csharp
var snapshot = list.Where(x => x > 0).ToList();
list.Add(4);
snapshot.Count();   // 3 — list change doesn't affect snapshot
```

### "Lazy parse" — defer expensive computation

```csharp
public IEnumerable<Row> ParseLines(IEnumerable<string> lines) {
    foreach (var line in lines) {
        yield return ParseRow(line);   // only runs when consumed
    }
}

var lazy = ParseLines(File.ReadLines("huge.csv"));
// File not read yet. Each iteration reads one line and parses one row.
foreach (var row in lazy.Take(100)) {
    // 100 lines read + parsed; rest unread
}
```

Streaming a huge file without loading it all. Composable with other LINQ operators.

### Materialize before crossing API boundaries

```csharp
public List<UserSummary> GetActiveSummary() {
    return users
        .Where(u => u.IsActive)
        .Select(u => new UserSummary(u.Id, u.Name))
        .ToList();   // materialize — don't return IEnumerable<T> from a public method
}
```

Returning `IEnumerable<T>` from a public method is dangerous — callers might enumerate it multiple times without realizing the cost.

---

## Common bugs

- **Multiple enumeration** — calling `.Count()` then `.ToList()` runs the chain twice.
- **`Count() > 0` instead of `Any()`** — iterates the whole sequence.
- **Side effects in `Select`** — fire per enumeration, not per query.
- **Returning a deferred query from a public method** — callers may not realize they're re-executing.
- **EF Core query with re-enumeration** — multiple SQL round trips.
- **Foreach over `IQueryable<T>` without materializing** — runs the SQL each time.
- **`.OrderBy().Take(5)` looks like O(N log N + 5) — it actually IS for in-memory** (BCL detects the Take after OrderBy and does a partial sort), **but for EF Core it produces `ORDER BY ... LIMIT 5`** which is much faster.

---

## Performance summary

- Building a deferred query: cheap (a few iterator objects + delegates).
- Running it once: cost proportional to source size + operator complexity.
- Running it twice: double the cost. Materialize if you'll need it.
- Short-circuit operators (Any, First, Take) save work proportionally.

---

## When deferred bites — and what to do

**Symptom**: "the query is slow" or "the database is being hit too much" or "my counter logged way more than expected."

Diagnose:
1. Find where the query is **enumerated** — that's where the work happens.
2. Count how many times it's enumerated.
3. Materialize if multi-enum is needed; restructure if one pass is enough.

**Cheap test**: log inside `Select` or `Where`. Count the prints.

---

## When deferred is fantastic

- Stream processing huge data sources.
- Composing complex queries without intermediate materialization.
- Avoiding work when consumers don't need all results.
- Database queries that translate to one SQL statement, even when built from many small operators.

LINQ is one of the most elegant abstractions in C#. Once you internalize deferred execution, you write better code with less effort.

→ Next: [05-StandardOperators.md](05-StandardOperators.md)
