# Chapter 06 — Coding Problems

> 15 hands-on LINQ problems. Cover both LINQ-to-Objects and the IQueryable / deferred-execution mental model.

---

## Problem 1 — Predict the output

```csharp
int count = 0;
var query = Enumerable.Range(1, 5).Where(n => {
    count++;
    return n > 2;
});

Console.WriteLine(count);
query.ToList();
Console.WriteLine(count);
query.ToList();
Console.WriteLine(count);
```

<details><summary>Answer</summary>
```
0
5
10
```

The `Where` is deferred — at the first print, the lambda hasn't run. Each `ToList()` enumerates the entire source, running the lambda 5 times. Two `ToList()`s → 10 total invocations.

</details>

---

## Problem 2 — Fix the perf bug

```csharp
public List<UserSummary> Build(IEnumerable<User> users) {
    var result = new List<UserSummary>();
    foreach (var u in users.Where(u => u.IsActive)) {
        result.Add(new UserSummary {
            Id = u.Id,
            Name = u.Name,
            OrderCount = users.Where(o => o.Id == u.Id).Count()   // ⚠
        });
    }
    return result;
}
```

<details><summary>Answer</summary>

The inner `users.Where(...).Count()` runs the predicate on the **whole source** for every active user. If users is deferred (e.g., EF Core IQueryable), every iteration also hits the database.

Also: `OrderCount = users.Where(o => o.Id == u.Id).Count()` looks wrong — it's filtering users by their own Id (each user has one) so it's always 0 or 1. Probably intended to count orders on the user, but the data model isn't clear.

Cleaner:
```csharp
public List<UserSummary> Build(IEnumerable<User> users) =>
    users.Where(u => u.IsActive)
         .Select(u => new UserSummary {
             Id = u.Id,
             Name = u.Name,
             OrderCount = u.Orders.Count()
         })
         .ToList();
```

One pass; access `u.Orders` directly.

</details>

---

## Problem 3 — Top 5 customers by total spend

Given `List<Order>` with each order having `CustomerId` and `Total`, write a LINQ query for the top 5 customer IDs by total spend.

<details><summary>Solution</summary>

```csharp
var top5 = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Total = g.Sum(o => o.Total) })
    .OrderByDescending(x => x.Total)
    .Take(5)
    .ToList();
```

Group, project the sum, sort descending, take 5.

In EF Core, this translates to a single SQL with `GROUP BY ... ORDER BY SUM(...) DESC LIMIT 5`.

</details>

---

## Problem 4 — Distinct ignoring case

Given a list of emails (case-varying), return them uniquely ignoring case.

<details><summary>Solution</summary>

```csharp
var unique = emails.Distinct(StringComparer.OrdinalIgnoreCase).ToList();
```

`Distinct` accepts an `IEqualityComparer<T>`. `StringComparer.OrdinalIgnoreCase` is the standard case-insensitive comparer for ASCII-only paths.

For more complex equality (e.g., "email domain only"):

```csharp
var byDomain = emails
    .Select(e => e.Split('@')[1])
    .Distinct(StringComparer.OrdinalIgnoreCase)
    .ToList();
```

Or `DistinctBy(e => e.Split('@')[1])` (.NET 6+).

</details>

---

## Problem 5 — Group orders by month and year

Given a `List<Order>` with `CreatedAt`, group counts by year-month.

<details><summary>Solution</summary>

```csharp
var byMonth = orders
    .GroupBy(o => new { o.CreatedAt.Year, o.CreatedAt.Month })
    .Select(g => new {
        g.Key.Year,
        g.Key.Month,
        Count = g.Count(),
        Total = g.Sum(o => o.Total)
    })
    .OrderBy(x => x.Year).ThenBy(x => x.Month)
    .ToList();
```

Composite key via anonymous type. Sort by year then month for readability.

</details>

---

## Problem 6 — Sliding window

Implement a `SlidingWindow<T>(int size)` extension that yields overlapping subsequences of length `size`.

```csharp
new[] { 1, 2, 3, 4, 5 }.SlidingWindow(3)
// → [1,2,3], [2,3,4], [3,4,5]
```

<details><summary>Solution</summary>

```csharp
public static IEnumerable<T[]> SlidingWindow<T>(this IEnumerable<T> source, int size) {
    if (size <= 0) throw new ArgumentOutOfRangeException(nameof(size));
    var window = new Queue<T>(size);
    foreach (var item in source) {
        window.Enqueue(item);
        if (window.Count == size) {
            yield return window.ToArray();
            window.Dequeue();
        }
    }
}
```

Uses a `Queue<T>` for FIFO behavior. When the queue reaches size, emit a snapshot and drop the oldest.

For an immutable view (no array copy per window), you'd need `Memory<T>` or `Span<T>` — see CSharpBook chapter 09.

</details>

---

## Problem 7 — Detect multiple enumeration

Add diagnostic logging to find unintended re-enumeration:

```csharp
IEnumerable<int> seq = GetSomeSequence();
seq.Count();
seq.ToList();
```

<details><summary>Solution</summary>

Wrap the source in a counting iterator:

```csharp
public static IEnumerable<T> CountIterations<T>(this IEnumerable<T> source, string name) {
    int iteration = 0;
    return Wrap();

    IEnumerable<T> Wrap() {
        iteration++;
        Console.WriteLine($"{name}: enumeration #{iteration} started");
        foreach (var item in source) yield return item;
        Console.WriteLine($"{name}: enumeration #{iteration} ended");
    }
}

var seq = GetSomeSequence().CountIterations("mySeq");
seq.Count();        // "mySeq: enumeration #1 started ... ended"
seq.ToList();       // "mySeq: enumeration #2 started ... ended"
```

In a real codebase, an analyzer rule (Roslyn) can warn about multiple enumeration. JetBrains Rider has this warning built in.

</details>

---

## Problem 8 — Dynamic predicate composition

Build a predicate function dynamically from a list of filters:

```csharp
var filters = new List<Expression<Func<User, bool>>> {
    u => u.IsActive,
    u => u.Country == "US",
    u => u.Age >= 18
};
```

Combine into one `Expression<Func<User, bool>>` and use it with `db.Users.Where(...)`.

<details><summary>Solution</summary>

```csharp
public static Expression<Func<T, bool>> And<T>(
    Expression<Func<T, bool>> a, Expression<Func<T, bool>> b)
{
    var parameter = Expression.Parameter(typeof(T), "x");
    var leftBody = new ParameterReplacer(a.Parameters[0], parameter).Visit(a.Body)!;
    var rightBody = new ParameterReplacer(b.Parameters[0], parameter).Visit(b.Body)!;
    return Expression.Lambda<Func<T, bool>>(Expression.AndAlso(leftBody, rightBody), parameter);
}

private class ParameterReplacer : ExpressionVisitor {
    private readonly ParameterExpression _from, _to;
    public ParameterReplacer(ParameterExpression from, ParameterExpression to) { _from = from; _to = to; }
    protected override Expression VisitParameter(ParameterExpression node) =>
        node == _from ? _to : base.VisitParameter(node);
}

// Use
Expression<Func<User, bool>> combined = filters.Aggregate((a, b) => And(a, b));
var users = db.Users.Where(combined).ToList();
```

The tricky part is **parameter unification** — each filter has its own `x` parameter; we rewrite them to share one. This is essentially what libraries like LinqKit do.

For EF Core, the combined tree gets translated to a single SQL with `WHERE A AND B AND C`.

</details>

---

## Problem 9 — Async LINQ — read lines from URL, count keywords

Implement an async method that streams lines from a URL and counts how many contain "error".

<details><summary>Solution</summary>

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string url,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var client = new HttpClient();
    using var resp = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    using var stream = await resp.Content.ReadAsStreamAsync(ct);
    using var reader = new StreamReader(stream);
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null) {
        yield return line;
    }
}

public async Task<int> CountErrorsAsync(string url) {
    int count = 0;
    await foreach (var line in ReadLinesAsync(url)) {
        if (line.Contains("error", StringComparison.OrdinalIgnoreCase)) count++;
    }
    return count;
}
```

Streams the response line-by-line — never holds the whole body in memory. Useful for huge log files.

</details>

---

## Problem 10 — Build a dynamic IQueryable filter

A controller receives optional filter parameters. Apply each to an IQueryable conditionally:

```csharp
public List<User> Search(string? name, int? minAge, string? country) {
    IQueryable<User> q = db.Users;
    // Apply filters only if provided
}
```

<details><summary>Solution</summary>

```csharp
public List<User> Search(string? name, int? minAge, string? country) {
    IQueryable<User> q = db.Users;
    if (!string.IsNullOrEmpty(name)) q = q.Where(u => u.Name.Contains(name));
    if (minAge.HasValue) q = q.Where(u => u.Age >= minAge.Value);
    if (!string.IsNullOrEmpty(country)) q = q.Where(u => u.Country == country);
    return q.ToList();
}
```

Each `Where` extends the expression tree. EF Core emits a single SQL with only the conditions that were specified.

Note: EF Core's query cache may treat each parameter combination as a different query. For a heavily-conditional API, consider raw SQL or a query builder.

</details>

---

## Problem 11 — Implement Median via INumber

Write a generic `Median<T>` that works for any `INumber<T>`:

<details><summary>Solution</summary>

```csharp
using System.Numerics;

public static T Median<T>(this IEnumerable<T> source) where T : INumber<T> {
    var arr = source.OrderBy(x => x).ToArray();
    if (arr.Length == 0) throw new InvalidOperationException();
    int mid = arr.Length / 2;
    T two = T.One + T.One;
    return arr.Length % 2 == 0
        ? (arr[mid - 1] + arr[mid]) / two
        : arr[mid];
}

new[] { 1, 5, 2, 9, 4 }.Median();      // 4 (int)
new[] { 1.5, 2.5, 3.5 }.Median();      // 2.5 (double)
new[] { 1m, 2m, 3m, 4m }.Median();      // 2.5 (decimal)
```

Works for any numeric type with the right operators.

</details>

---

## Problem 12 — Aggregate with state

Compute moving averages using `Aggregate`:

<details><summary>Solution</summary>

```csharp
public static IEnumerable<double> MovingAverage(this IEnumerable<int> source, int window) {
    if (window <= 0) throw new ArgumentOutOfRangeException(nameof(window));

    var queue = new Queue<int>(window);
    long sum = 0;
    foreach (var x in source) {
        queue.Enqueue(x);
        sum += x;
        if (queue.Count > window) sum -= queue.Dequeue();
        if (queue.Count == window) yield return (double)sum / window;
    }
}

new[] { 1, 2, 3, 4, 5, 6 }.MovingAverage(3).ToList();
// [(1+2+3)/3=2.0, (2+3+4)/3=3.0, (3+4+5)/3=4.0, (4+5+6)/3=5.0]
```

`Aggregate` could express this but a sliding-window with mutable state is clearer as an iterator.

</details>

---

## Problem 13 — Spot the N+1 in EF Core

```csharp
var users = db.Users.Where(u => u.IsActive).ToList();
foreach (var u in users) {
    Console.WriteLine($"{u.Name}: {u.Orders.Count} orders");
}
```

What's wrong, and how to fix?

<details><summary>Answer</summary>

`u.Orders.Count` is **lazy-loaded** per user — one SQL query per user (the "1 + N" problem: 1 for the main query, N for the per-user navigation property access). For 1000 users → 1001 queries.

**Fix 1** — eager load with `Include`:

```csharp
var users = db.Users.Where(u => u.IsActive).Include(u => u.Orders).ToList();
```

Now one SQL query with a JOIN.

**Fix 2** — project to a DTO with the count baked in:

```csharp
var users = db.Users
    .Where(u => u.IsActive)
    .Select(u => new { u.Name, OrderCount = u.Orders.Count() })
    .ToList();
```

EF Core translates `u.Orders.Count()` to a SQL subquery. Most efficient when you don't need full Order entities.

</details>

---

## Problem 14 — Build a query with optional includes

A method accepts a flag `includeOrders` — only load orders if true.

```csharp
public List<User> GetUsers(bool includeOrders) {
    IQueryable<User> q = db.Users.Where(u => u.IsActive);
    // ...
}
```

<details><summary>Solution</summary>

```csharp
public List<User> GetUsers(bool includeOrders) {
    IQueryable<User> q = db.Users.Where(u => u.IsActive);
    if (includeOrders) q = q.Include(u => u.Orders);
    return q.ToList();
}
```

`Include` extends the IQueryable, so we can conditionally add it. EF translates `Include` to either a JOIN (default) or a separate query (with `AsSplitQuery`).

</details>

---

## Problem 15 — Performance face-off

Benchmark these three implementations of "sum even numbers":

```csharp
// 1. LINQ
int sum1 = arr.Where(n => n % 2 == 0).Sum();

// 2. LINQ with Aggregate
int sum2 = arr.Aggregate(0, (s, n) => n % 2 == 0 ? s + n : s);

// 3. Manual for-loop
int sum3 = 0;
for (int i = 0; i < arr.Length; i++) if (arr[i] % 2 == 0) sum3 += arr[i];
```

Which is fastest? By how much (rough estimate)?

<details><summary>Answer</summary>

For a million-element int array on typical x64 hardware:

- **Manual for-loop**: ~500 µs, zero allocations.
- **LINQ Aggregate**: ~3 ms, a few iterator allocations.
- **LINQ Where + Sum**: ~5 ms, more iterators.

Roughly 5-10× difference. For ultra-hot paths, hand-roll. For everyday code, LINQ is fine.

Use BenchmarkDotNet to get real numbers:

```csharp
[MemoryDiagnoser]
public class SumBenchmark {
    private readonly int[] _arr = Enumerable.Range(0, 1_000_000).ToArray();

    [Benchmark] public int Linq() => _arr.Where(n => n % 2 == 0).Sum();
    [Benchmark] public int Aggregate() => _arr.Aggregate(0, (s, n) => n % 2 == 0 ? s + n : s);
    [Benchmark] public int Manual() {
        int sum = 0;
        for (int i = 0; i < _arr.Length; i++) if (_arr[i] % 2 == 0) sum += _arr[i];
        return sum;
    }
}
```

In .NET 10, the gap is narrowing — improved JIT and array specialization. Run your own benchmarks; don't trust assumptions across runtime versions.

</details>

---

That's Chapter 06. You should now have a strong intuition for:
- LINQ as a unified query language across IEnumerable and IQueryable.
- Method vs query syntax — and when each wins.
- Deferred vs immediate execution.
- The vocabulary of standard operators.
- Writing your own operators when needed.
- Async LINQ via `IAsyncEnumerable<T>`.
- IQueryable's expression-tree machinery and how providers (EF Core) translate.
- Common performance traps and how to avoid them.

→ [Chapter 07 — Collections](../07-Collections/README.md)
