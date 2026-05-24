# Chapter 06 — Questions

> Drilling questions for LINQ. The deferred-execution model is the topic that catches people most often — focus there.

---

## Basics

**Q1.** What's the difference between LINQ to Objects and LINQ to Providers (IQueryable)?
<details><summary>Answer</summary>LINQ to Objects works on `IEnumerable<T>` — operators take compiled `Func<>` delegates and run in C# on in-memory data. LINQ to Providers works on `IQueryable<T>` — operators take `Expression<Func<>>` trees that a provider (EF Core, Cosmos, etc.) translates to SQL or other languages.</details>

**Q2.** Predict the output:
```csharp
var query = Enumerable.Range(1, 5).Select(n => {
    Console.Write("S");
    return n * 2;
});
query.Count();
query.Sum();
```
<details><summary>Answer</summary>`SSSSSSSSSS` — 10 S's. Each `Count()` and `Sum()` re-enumerates, so `Select`'s lambda runs 5 times per terminal call, 10 times total.</details>

**Q3.** Why is `Count() > 0` worse than `Any()`?
<details><summary>Answer</summary>`Count()` iterates the entire sequence; `Any()` short-circuits at the first item. For an IEnumerable of 1M items where you just want to know "is there any," `Any()` is dramatically faster. For databases, `Count()` translates to `SELECT COUNT(*)` (full scan); `Any()` to `SELECT EXISTS(...)` (stops at first match).</details>

---

## Deferred execution

**Q4.** What does this print and why?
```csharp
var nums = new List<int> { 1, 2, 3 };
var query = nums.Where(n => n > 0);
nums.Add(4);
foreach (var n in query) Console.Write(n);
```
<details><summary>Answer</summary>`1234`. The query holds a reference to `nums` (not a snapshot). Iterating later sees the updated list.</details>

**Q5.** Which of these run immediately vs are deferred?
- `Where`, `Select`, `OrderBy`, `ToList`, `Count`, `First`, `Any`, `GroupBy`.

<details><summary>Answer</summary>
**Deferred**: Where, Select, OrderBy (defers, then materializes when iterated), GroupBy.
**Immediate**: ToList, Count, First, Any.

OrderBy is special: deferred to start, but when iterated must consume the whole input to sort.
</details>

**Q6.** Predict the output:
```csharp
var nums = new List<int> { 1, 2, 3, 4, 5 };
var doubled = nums.Select(n => n * 2);
nums.Clear();
Console.WriteLine(doubled.Count());
```
<details><summary>Answer</summary>**0.** `doubled` holds a reference to `nums`. By the time `Count()` enumerates, `nums` is empty.</details>

---

## Standard operators

**Q7.** Difference between `Single()` and `First()`?
<details><summary>Answer</summary>`First` returns the first matching item (throws if none). `Single` returns the one item — throws if 0 or more than 1. Use `Single` when uniqueness is part of the contract (a primary key lookup); use `First` when you just want any matching item.</details>

**Q8.** What does `SelectMany` do vs `Select`?
<details><summary>Answer</summary>`Select(o => o.Items)` returns `IEnumerable<List<Item>>` — a sequence of lists. `SelectMany(o => o.Items)` flattens to `IEnumerable<Item>` — all items across all orders.</details>

**Q9.** Difference between `MaxBy(u => u.Age)` and `Max(u => u.Age)`?
<details><summary>Answer</summary>`Max(u => u.Age)` returns the maximum **age** (an int). `MaxBy(u => u.Age)` returns the **User** with the maximum age. (`MaxBy` / `MinBy` are .NET 6+.)</details>

**Q10.** What does `DefaultIfEmpty()` do?
<details><summary>Answer</summary>If the source is empty, yields one default value; otherwise yields the source unchanged. Used in left outer joins: `from c in customers join o in orders on c.Id equals o.CustomerId into og from o in og.DefaultIfEmpty() select (c, o?.Id)`.</details>

---

## Query syntax

**Q11.** Convert this query syntax to method syntax:
```csharp
var result = from u in users
             where u.IsActive
             let upper = u.Name.ToUpper()
             orderby upper
             select new { u.Id, Name = upper };
```
<details><summary>Answer</summary>
```csharp
var result = users
    .Where(u => u.IsActive)
    .Select(u => new { u, upper = u.Name.ToUpper() })
    .OrderBy(x => x.upper)
    .Select(x => new { x.u.Id, Name = x.upper });
```
Note the intermediate anonymous type for `let` — query syntax is shorter here.
</details>

**Q12.** What's the difference between `join ... on a equals b` and `==`?
<details><summary>Answer</summary>`equals` is the C# keyword in query syntax for join keys. It's NOT the `==` operator. Compile error if you write `==` in a `join` clause.</details>

---

## IQueryable

**Q13.** Predict — how many SQL queries does this run?
```csharp
var users = db.Users.Where(u => u.IsActive);
int count = users.Count();
var first = users.First();
var list = users.ToList();
```
<details><summary>Answer</summary>**3.** Each terminal operator (`Count`, `First`, `ToList`) executes a fresh SQL query against the DB. Multi-enumeration of a deferred IQueryable. Materialize once: `var list = users.ToList(); int count = list.Count;`.</details>

**Q14.** Why does this throw?
```csharp
db.Users.Where(u => MyHelper(u.Name)).ToList();   // EF Core 6+
```
<details><summary>Answer</summary>EF Core can't translate `MyHelper` to SQL — it's an arbitrary C# method. EF 6+ throws `InvalidOperationException`. EF 5 and earlier silently fell back to client evaluation (pulling all rows + filtering in C#) — surprising perf hit. To use C# methods, `.AsEnumerable()` first (but be aware that fetches all rows).</details>

**Q15.** What's `AsEnumerable()` for?
<details><summary>Answer</summary>Switches the static type from `IQueryable<T>` to `IEnumerable<T>`. Subsequent operators run in C# (LINQ-to-Objects), not as SQL. Doesn't materialize — just changes type. Use when you need a C# operation EF can't translate. Be careful: everything before AsEnumerable runs in SQL; everything after, in memory.</details>

---

## Async LINQ

**Q16.** What's `IAsyncEnumerable<T>`?
<details><summary>Answer</summary>An async pull-based sequence. Each `MoveNextAsync` returns a `ValueTask<bool>`. Used for streaming data (database row streaming, network streams, gRPC server-streaming). Consume with `await foreach`.</details>

**Q17.** What does `[EnumeratorCancellation]` do?
<details><summary>Answer</summary>Marks a CancellationToken parameter on an async iterator. When the consumer calls `.WithCancellation(token)` on the enumerable, the framework wires their token to the marked parameter. Without it, `WithCancellation` is a no-op for the iterator's own awaits.</details>

---

## Custom operators

**Q18.** Sketch a `Batch<T>(int size)` extension that yields lists of `size` items.
<details><summary>Answer</summary>
```csharp
public static IEnumerable<List<T>> Batch<T>(this IEnumerable<T> source, int size) {
    var batch = new List<T>(size);
    foreach (var item in source) {
        batch.Add(item);
        if (batch.Count == size) { yield return batch; batch = new(size); }
    }
    if (batch.Count > 0) yield return batch;
}
```
.NET 6+ has `Chunk(size)` in the BCL — use it instead in new code.
</details>

**Q19.** Why might a custom operator break IQueryable's translation?
<details><summary>Answer</summary>If it calls `.AsEnumerable()` or wraps logic in an iterator method, it shifts execution to C#. The provider (EF Core) can't translate iterator methods. Custom IQueryable operators should compose using existing IQueryable methods (`Where`, `Select`, etc.) that the provider knows.</details>

---

## Performance

**Q20.** What's wrong with this code?
```csharp
public IEnumerable<User> GetActiveUsers() => db.Users.Where(u => u.IsActive);

// Caller:
var users = repository.GetActiveUsers();
foreach (var u in users) Process1(u);
foreach (var u in users) Process2(u);
```
<details><summary>Answer</summary>Two SQL queries — each foreach re-enumerates the deferred IQueryable. Fix: materialize in the repository (`.ToList()`) OR document that the returned IEnumerable defers.</details>

**Q21.** Predict the time complexity:
```csharp
items.OrderBy(x => x.Key).First(x => x.IsActive);
```
<details><summary>Answer</summary>O(n log n) — full sort. If you only need the first active item by Key, do `items.Where(x => x.IsActive).MinBy(x => x.Key)` for O(n).</details>

**Q22.** When does LINQ allocate?
<details><summary>Answer</summary>
- One iterator state-machine object per deferred operator in the chain.
- One enumerator object per `foreach` / `ToList`.
- One closure object per capturing lambda (none for static lambdas).
- One delegate per lambda.
- Plus the materialized collection (`ToList` allocates a List).

For pure hot paths, it adds up. For typical code, negligible.
</details>

---

## Synthesis

**Q23.** Given:
```csharp
var inactive = db.Users.Where(u => !u.IsActive).ToList();
var us = inactive.Where(u => u.Country == "US").ToList();
```
What's wrong, and how would you fix it?
<details><summary>Answer</summary>The first `ToList()` materializes ALL inactive users, then filters in memory. Could be 1M+ rows. Combine the filters into one IQueryable expression:
```csharp
var us = db.Users.Where(u => !u.IsActive && u.Country == "US").ToList();
```
Now a single SQL query with the combined WHERE clause.</details>

**Q24.** What's the cleanest way to "find the customer who spent the most in the last 30 days"?
<details><summary>Answer</summary>
```csharp
var top = await db.Customers
    .Select(c => new {
        Customer = c,
        Total = c.Orders.Where(o => o.CreatedAt >= DateTime.UtcNow.AddDays(-30)).Sum(o => o.Total)
    })
    .OrderByDescending(x => x.Total)
    .FirstAsync();
```
Sums per customer, sorts, takes top one. EF Core translates to a single SQL with subquery sum + ORDER BY + LIMIT.
</details>

**Q25.** A coworker writes:
```csharp
var grouped = users.GroupBy(u => u.Country).ToDictionary(g => g.Key, g => g.ToList());
```
What's worth knowing?
<details><summary>Answer</summary>
- This materializes everything: groups + each group's full list.
- The same effect as `users.ToLookup(u => u.Country)` — but ToLookup returns `ILookup<TKey, T>`, a built-in multi-value dictionary. Slightly less allocation.
- For EF Core: this triggers a full materialization. EF will pull all users first, then group in memory. If you only need counts per country, use a server-side projection (`GroupBy + Count + ToDictionaryAsync`).
</details>

**Q26.** Open-ended — explain why `IEnumerable<T>` is covariant (`out T`) but `IList<T>` is not.
<details><summary>Sample answer</summary>
`IEnumerable<out T>` only produces T (via `GetEnumerator + Current`). So substituting up (e.g., `IEnumerable<string>` → `IEnumerable<object>`) is safe — every produced string IS an object.

`IList<T>` both produces (indexer get) AND consumes T (indexer set, Add). Substituting up would let you `Add(42)` to what's really a `List<string>` — type unsafe. So `IList<T>` must be invariant.

The same logic applies to mutable vs read-only collections in general.
</details>

---

→ [Coding.md](Coding.md)
