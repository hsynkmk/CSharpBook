# Writing Custom LINQ Operators

## What it is

LINQ operators are just **extension methods** on `IEnumerable<T>` (or `IQueryable<T>`). Adding your own is a one-line change. Done well, custom operators read like first-class LINQ.

```csharp
public static class MyExt {
    public static IEnumerable<T> WhereNot<T>(this IEnumerable<T> source, Func<T, bool> predicate) =>
        source.Where(x => !predicate(x));
}

// Use as if it shipped with LINQ:
nums.WhereNot(x => x < 0);
```

If you've ever wanted `Median`, `WeightedAverage`, `Batch`, `RandomSample`, or `Cartesian`, you can write them once and use them everywhere.

---

## Why bother

The BCL has ~50 operators. Real codebases use 15-20 regularly and reach for custom ones to express domain-specific transforms cleanly. Writing your own:

- **Encapsulates** repeated query patterns behind a name.
- **Reads naturally** — `users.ActiveOnly().ByCountry("US")` beats two `Where` calls with magic predicates.
- **Composes** with the rest of LINQ.
- **Tests well** — pure functions over sequences.

The downside is mostly culture — a library of 50 custom operators can confuse newcomers who don't know which are BCL and which are local.

---

## The simplest case — delegating to BCL

When your operator is "BCL but renamed for clarity":

```csharp
public static class MyExt {
    public static IEnumerable<T> NotNull<T>(this IEnumerable<T?> source) where T : class =>
        source.Where(x => x != null)!;

    public static IEnumerable<T> NotNull<T>(this IEnumerable<T?> source) where T : struct =>
        source.Where(x => x.HasValue).Select(x => x!.Value);
}

users.Select(u => u.Manager).NotNull();   // skip null managers
```

Just a wrapper. Same speed as `Where`, but more expressive.

---

## A custom iterator — `Batch`

Process items in groups of N. Useful for batch DB inserts, API calls, file writes.

```csharp
public static IEnumerable<T[]> Batch<T>(this IEnumerable<T> source, int size) {
    if (size <= 0) throw new ArgumentOutOfRangeException(nameof(size));

    var batch = new List<T>(size);
    foreach (var item in source) {
        batch.Add(item);
        if (batch.Count == size) {
            yield return batch.ToArray();
            batch.Clear();
        }
    }
    if (batch.Count > 0) yield return batch.ToArray();
}

// Use
foreach (var chunk in users.Batch(100)) {
    db.SaveAll(chunk);
}
```

Note: .NET 6+ has `Chunk(int size)` in the BCL — use it instead if available. The above is the implementation pattern.

The `yield return` makes this **deferred**: the source is consumed lazily, one batch at a time. No materialization of the whole source.

---

## Aggregating custom operators — `Median`

```csharp
public static double Median(this IEnumerable<int> source) {
    var sorted = source.OrderBy(x => x).ToArray();
    if (sorted.Length == 0) throw new InvalidOperationException();
    int mid = sorted.Length / 2;
    return sorted.Length % 2 == 0
        ? (sorted[mid - 1] + sorted[mid]) / 2.0
        : sorted[mid];
}

new[] { 1, 5, 2, 9, 4 }.Median();   // 4
```

Aggregating operators **materialize** (immediate execution). They return a scalar, not an `IEnumerable<T>`.

For generic numeric support (.NET 7+):

```csharp
using System.Numerics;

public static T Median<T>(this IEnumerable<T> source) where T : INumber<T> {
    var sorted = source.OrderBy(x => x).ToArray();
    if (sorted.Length == 0) throw new InvalidOperationException();
    int mid = sorted.Length / 2;
    T two = T.One + T.One;
    return sorted.Length % 2 == 0
        ? (sorted[mid - 1] + sorted[mid]) / two
        : sorted[mid];
}
```

Now works for int, double, decimal — anything that satisfies `INumber<T>`. See [Chapter 04 §06](../04-Generics/06-GenericMath.md).

---

## Returning an `IEnumerable<T>` — keep it lazy

A deferred operator yields items lazily:

```csharp
public static IEnumerable<T> TakeUntil<T>(this IEnumerable<T> source, Func<T, bool> predicate) {
    foreach (var item in source) {
        yield return item;
        if (predicate(item)) yield break;
    }
}

// "take until I find a multiple of 100"
Enumerable.Range(1, 1_000_000)
    .TakeUntil(n => n % 100 == 0)
    .ToList();   // {1, 2, ..., 100}
```

Like `TakeWhile`, but **inclusive** of the stopping element.

Iterator notes:
- `yield return item` produces one item.
- `yield break` ends the sequence.
- Code between yields runs lazily, on demand.

Compiled by C# into a state-machine class. Same patterns as BCL operators.

---

## A more complex custom — `RandomSample`

Random subsample of N items:

```csharp
public static IEnumerable<T> RandomSample<T>(this IEnumerable<T> source, int sampleSize) {
    var arr = source.ToArray();
    var rng = Random.Shared;
    for (int i = 0; i < Math.Min(sampleSize, arr.Length); i++) {
        int j = rng.Next(i, arr.Length);
        (arr[i], arr[j]) = (arr[j], arr[i]);   // partial Fisher–Yates
        yield return arr[i];
    }
}

users.RandomSample(10);
```

Fisher–Yates shuffle, but only enough to produce the sample. O(n) memory (the array copy), O(sample) time.

For streaming (no full materialization), use **reservoir sampling**:

```csharp
public static IEnumerable<T> ReservoirSample<T>(this IEnumerable<T> source, int sampleSize) {
    var reservoir = new T[sampleSize];
    int i = 0;
    foreach (var item in source) {
        if (i < sampleSize) {
            reservoir[i] = item;
        } else {
            int j = Random.Shared.Next(0, i + 1);
            if (j < sampleSize) reservoir[j] = item;
        }
        i++;
    }
    return reservoir.Take(Math.Min(sampleSize, i));
}
```

Works on infinite sequences without buffering them.

---

## Helper operators for projections

```csharp
public static IEnumerable<(T1, T2)> Zip<T1, T2>(this IEnumerable<T1> a, IEnumerable<T2> b) =>
    a.Zip(b, (x, y) => (x, y));   // tuple form already in .NET 6+ Zip

public static IEnumerable<(T value, int index)> WithIndex<T>(this IEnumerable<T> source) =>
    source.Select((value, index) => (value, index));

foreach (var (user, i) in users.WithIndex()) {
    Console.WriteLine($"{i}: {user.Name}");
}
```

Small helpers that make calling code clearer.

---

## Operators on `IQueryable<T>` — staying provider-friendly

A custom operator on `IEnumerable<T>` works in-memory. On `IQueryable<T>` (EF Core), you can only use what the provider can translate.

**Bad** (cannot translate to SQL):
```csharp
public static IQueryable<T> ActiveOnly<T>(this IQueryable<T> source) where T : IUser {
    return source.AsEnumerable().Where(u => u.IsActive).AsQueryable();
    // ⚠ AsEnumerable pulls all rows into memory!
}
```

**Good** (composes into the expression tree):
```csharp
public static IQueryable<T> ActiveOnly<T>(this IQueryable<T> source) where T : IUser {
    return source.Where(u => u.IsActive);
}

db.Users.ActiveOnly().Where(u => u.Country == "US").ToList();
// → translated to a SQL query with WHERE IsActive AND Country = 'US'
```

The trick: keep your operator as a `Where(...)`-style lambda, and the provider sees through it.

For more advanced cases, you can build the expression tree manually — but it gets hairy. Most custom operators on `IQueryable` are wrappers around `Where`/`Select`/`OrderBy`.

---

## Internals — what's inside an iterator-based operator

The C# compiler turns an iterator method (one with `yield return`) into a state machine. For:

```csharp
public static IEnumerable<T> WhereNot<T>(this IEnumerable<T> source, Func<T, bool> predicate) {
    foreach (var item in source)
        if (!predicate(item)) yield return item;
}
```

The compiler generates a private class implementing `IEnumerable<T>` and `IEnumerator<T>`:

```csharp
private sealed class WhereNotIterator<T> : IEnumerable<T>, IEnumerator<T> {
    private readonly IEnumerable<T> _source;
    private readonly Func<T, bool> _predicate;
    private IEnumerator<T>? _enum;
    private int _state;
    public T Current { get; private set; } = default!;

    public WhereNotIterator(IEnumerable<T> source, Func<T, bool> predicate) {
        _source = source; _predicate = predicate;
    }

    public bool MoveNext() {
        switch (_state) {
            case 0:
                _enum = _source.GetEnumerator();
                _state = 1;
                goto case 1;
            case 1:
                while (_enum!.MoveNext()) {
                    if (!_predicate(_enum.Current)) {
                        Current = _enum.Current;
                        return true;
                    }
                }
                _enum.Dispose();
                _state = -1;
                return false;
        }
        return false;
    }

    public IEnumerator<T> GetEnumerator() => /* return this if first; else new instance */;
    // ... other interface members
}
```

(The actual generated code is messier — handles thread-id checks, lazy enumerator init, dispose pattern.)

Performance:
- One iterator allocation per query.
- One enumerator allocation per iteration.
- Per-item: one delegate call + one virtual call.

For most queries, fine. For tight inner loops, write a regular method that takes a `List<T>` or `Span<T>` and loops manually.

---

## Patterns

### Fluent helper

```csharp
public static IEnumerable<T> Tap<T>(this IEnumerable<T> source, Action<T> sideEffect) {
    foreach (var x in source) {
        sideEffect(x);
        yield return x;
    }
}

users.Tap(u => Log(u)).Where(u => u.IsActive).ToList();
```

Useful for logging mid-pipeline. Use sparingly — side effects in LINQ chains are usually a smell.

### Pagination

```csharp
public static IEnumerable<T> Page<T>(this IEnumerable<T> source, int pageNumber, int pageSize) =>
    source.Skip((pageNumber - 1) * pageSize).Take(pageSize);

users.Page(3, 20);   // page 3 of size 20
```

### Cartesian product

```csharp
public static IEnumerable<(T1, T2)> Cartesian<T1, T2>(this IEnumerable<T1> a, IEnumerable<T2> b) {
    foreach (var x in a)
        foreach (var y in b)
            yield return (x, y);
}

new[] { 1, 2 }.Cartesian(new[] { "a", "b" });   // (1,"a"), (1,"b"), (2,"a"), (2,"b")
```

### Group every N

```csharp
public static IEnumerable<List<T>> GroupRunsOf<T>(this IEnumerable<T> source, int n) {
    var run = new List<T>(n);
    foreach (var x in source) {
        run.Add(x);
        if (run.Count == n) { yield return run; run = new List<T>(n); }
    }
    if (run.Count > 0) yield return run;
}
```

Similar to `Chunk` but returns lists you can mutate. Useful for one-off processing.

---

## Common bugs

- **Forgetting `yield break`** — pre-`return` from an iterator method is illegal (it's not a method, it's a state machine).
- **Missing null/empty source check** — typical operators throw on null source; replicate that.
- **Materializing eagerly when laziness is expected** — beware `ToList()`/`ToArray()` inside an iterator's body.
- **Operators that don't compose with IQueryable** — anything that materializes early breaks DB query translation.
- **Side effects in a "pure" operator** — same caveats as any LINQ.

---

## A library of common custom operators

These pop up frequently enough that you might want them ready:

```csharp
public static class MyLinq {
    // Skip null reference items
    public static IEnumerable<T> NotNull<T>(this IEnumerable<T?> source) where T : class =>
        source.Where(x => x != null)!;

    // Skip null nullable-value-type items
    public static IEnumerable<T> NotNull<T>(this IEnumerable<T?> source) where T : struct =>
        source.Where(x => x.HasValue).Select(x => x!.Value);

    // Inverse of Where
    public static IEnumerable<T> WhereNot<T>(this IEnumerable<T> source, Func<T, bool> predicate) =>
        source.Where(x => !predicate(x));

    // Index pairs
    public static IEnumerable<(int Index, T Value)> WithIndex<T>(this IEnumerable<T> source) =>
        source.Select((value, index) => (index, value));

    // Side-effect on each item (use carefully)
    public static IEnumerable<T> Tap<T>(this IEnumerable<T> source, Action<T> action) {
        foreach (var x in source) { action(x); yield return x; }
    }

    // Foreach as a terminal action
    public static void ForEach<T>(this IEnumerable<T> source, Action<T> action) {
        foreach (var x in source) action(x);
    }

    // Distinct by selector with case-insensitive comparison for strings
    // (DistinctBy is built-in in .NET 6+, just a reminder)
}
```

---

## When to write a custom operator

✓ The pattern appears 3+ times in your codebase.
✓ It has a name that says more than the inline form.
✓ It encapsulates real logic, not just a rename.
✓ It composes with the rest of LINQ.

✗ It's used once, in one place — keep it inline.
✗ It's just a wrapper with no added clarity.
✗ It breaks IQueryable translation.

---

## Performance summary

- Iterator-based operators: one allocation per query, ~negligible overhead per item.
- Materializing operators: cost proportional to source size, plus allocation for the result.
- Cascading custom operators add per-operator overhead. For mega-hot loops, fuse them or drop to manual `for`.

→ Next: [07-AsyncLinq.md](07-AsyncLinq.md)
