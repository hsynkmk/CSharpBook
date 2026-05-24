# LINQ — Overview

## What it is

**LINQ** — Language Integrated Query — is C#'s unified syntax for querying collections, databases, XML, JSON, and any other data source that implements the right contract. Introduced in C# 3 (2007), LINQ collapsed dozens of disparate query APIs into one consistent vocabulary.

```csharp
// Old way (pre-LINQ)
var actives = new List<User>();
foreach (var u in users) {
    if (u.IsActive) actives.Add(u);
}
actives.Sort((a, b) => a.Name.CompareTo(b.Name));

// LINQ
var actives = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .ToList();
```

Same result, declarative, composable, idiomatic. Once you internalize LINQ, hand-rolled loops feel verbose.

---

## Why it exists

Before LINQ, every data source had its own API:

- Lists → `foreach` + manual filter/transform.
- SQL → ADO.NET, string concatenation, stored procedures.
- XML → `XPathNavigator`, `XmlReader`, `XmlDocument`.
- Web services → custom serializers.

LINQ unified them under one set of operators (`Where`, `Select`, `OrderBy`, `GroupBy`, etc.) that:
- Are **type-safe** — caught at compile time.
- Are **composable** — chain them freely.
- **Defer execution** — built up cheaply, evaluated lazily.
- Adapt to the source — same operators talk to lists, databases, XML, or custom providers.

The result: one query language, many backends.

---

## Two flavors: LINQ to Objects vs LINQ to Providers

LINQ comes in **two fundamental forms**, and the distinction is crucial.

### LINQ to Objects — `IEnumerable<T>`

Operates on **in-memory** collections. Each operator takes a `Func<>` (a compiled delegate) and uses C# code to filter, transform, etc.

```csharp
List<int> nums = new() { 1, 2, 3, 4, 5 };
var evens = nums.Where(n => n % 2 == 0);   // Func<int, bool>
```

Runs as plain compiled C# code. The lambda's body is executed once per element.

### LINQ to Providers — `IQueryable<T>`

Operates on **expression trees**. Each operator takes an `Expression<Func<>>` instead of a `Func<>`. The provider (EF Core, Cosmos, OData, ...) walks the expression tree at execution time and translates it to its native query language (SQL, KQL, OData, etc.).

```csharp
IQueryable<User> users = db.Users;   // IQueryable from EF Core
var actives = users.Where(u => u.IsActive);   // Expression<Func<User, bool>>
// → translated to "WHERE IsActive = 1" SQL when materialized
```

Same syntax. Different mechanics. The lambda is **never compiled to a delegate**; it stays as a tree and gets translated.

This is the deep insight of LINQ: the surface looks identical, but the underlying machinery differs by source. Chapter 06 §08 covers IQueryable in detail.

---

## The vocabulary

LINQ defines about **50 standard query operators**. The common ones, by category:

### Filtering
- `Where(predicate)` — keep items matching a condition.
- `OfType<T>()` — keep items of a specific type.
- `Distinct()` — remove duplicates.

### Projection
- `Select(selector)` — transform each item.
- `SelectMany(selector)` — flatten nested sequences.

### Ordering
- `OrderBy(keySelector)`, `OrderByDescending(...)`
- `ThenBy(...)`, `ThenByDescending(...)` — secondary sort.
- `Reverse()` — flip order.

### Grouping
- `GroupBy(keySelector)` — group items into `IGrouping<TKey, T>` buckets.

### Joining
- `Join(inner, outerKey, innerKey, result)` — SQL-style inner join.
- `GroupJoin(...)` — left outer join (groups inner per outer).

### Set operations
- `Union(other)`, `Intersect(other)`, `Except(other)`, `Concat(other)`.

### Partitioning
- `Take(n)`, `Skip(n)`.
- `TakeWhile(predicate)`, `SkipWhile(predicate)`.
- `Take(Range)` (C# 8+ ranges).

### Element access
- `First()`, `FirstOrDefault()`.
- `Last()`, `LastOrDefault()`.
- `Single()`, `SingleOrDefault()` — expect exactly one.
- `ElementAt(n)`, `ElementAtOrDefault(n)`.
- `MinBy(keySelector)`, `MaxBy(keySelector)` (.NET 6+).

### Quantifier
- `Any()` / `Any(predicate)` — any matching?
- `All(predicate)` — all match?
- `Contains(value)` — value present?

### Aggregation
- `Count()`, `LongCount()`.
- `Sum()`, `Min()`, `Max()`, `Average()`.
- `Aggregate(seed, func)` — general fold.

### Conversion
- `ToList()`, `ToArray()`, `ToDictionary()`, `ToHashSet()`, `ToLookup()`.
- `AsEnumerable()`, `AsQueryable()` — switch the static type.
- `Cast<T>()` — force a type via cast (throws on mismatch).

### Generation (static methods, not on a sequence)
- `Enumerable.Range(start, count)` — 0, 1, 2, ...
- `Enumerable.Repeat(value, count)`.
- `Enumerable.Empty<T>()`.

You'll use 15-20 of these constantly; the rest are tools for specific jobs.

[Chapter 06 §05 Standard Operators](05-StandardOperators.md) walks through every one with examples.

---

## Two syntaxes: method vs query

Every LINQ query can be written two ways.

### Method syntax

```csharp
var result = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new { u.Id, u.Name });
```

Chain of method calls with lambdas. Closer to the underlying API. Most C# code uses this exclusively.

### Query syntax

```csharp
var result = from u in users
             where u.IsActive
             orderby u.Name
             select new { u.Id, u.Name };
```

SQL-like keywords (`from`, `where`, `orderby`, `select`, `group`, `join`). Compiler rewrites this to method calls. Some operations (multi-source `join` with `into`, `let`) are more readable in query syntax; others are clearer in method syntax.

You can mix:

```csharp
var groups = (from u in users
              where u.IsActive
              group u by u.Country into g
              select new { Country = g.Key, Count = g.Count() })
            .OrderByDescending(g => g.Count)
            .Take(5);
```

[§02](02-MethodSyntax.md) and [§03](03-QuerySyntax.md) cover both.

---

## Deferred execution

**The most important LINQ concept**: most operators **don't execute** when written. They build up a query that runs only when **enumerated**.

```csharp
var query = users.Where(u => u.IsActive);   // nothing happens yet
foreach (var u in query) { /* THIS triggers the Where logic */ }
```

Operators like `Where`, `Select`, `OrderBy`, `GroupBy`, etc., **defer**. Operators like `ToList`, `ToArray`, `Count`, `Sum`, `First`, etc., **materialize** — they consume the sequence right then and return a result.

Why this matters:
- Building queries is cheap (no work, no allocation).
- Re-enumerating runs everything again.
- Side effects in `Select`/`Where` fire each time you iterate.

[§04 Deferred Execution](04-DeferredExecution.md) is dedicated to this.

---

## Type inference

LINQ leans heavily on `var` and type inference. Most queries produce intermediate types you don't name:

```csharp
var grouped = users
    .GroupBy(u => u.Country)
    .Select(g => new { Country = g.Key, Total = g.Count() });
```

The intermediate type is `IEnumerable<IGrouping<string, User>>`; the final is `IEnumerable<{ Country: string, Total: int }>` (anonymous type). You almost never write these explicitly.

When you need to **return** a query from a method, you typically materialize first:

```csharp
public List<UserSummary> GetActiveSummary() =>
    users.Where(u => u.IsActive)
         .Select(u => new UserSummary(u.Id, u.Name))
         .ToList();
```

[§04](04-DeferredExecution.md) explains why materializing matters when crossing API boundaries.

---

## Anonymous types in LINQ

LINQ projections often create one-off shapes:

```csharp
var summary = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new {
        CustomerId = g.Key,
        Count = g.Count(),
        Total = g.Sum(o => o.Total)
    });
```

The `new { ... }` syntax creates an **anonymous type** — a compiler-generated class. See [Chapter 03 §10](../03-TypeSystem/10-AnonymousTypes.md). Don't try to return them from public methods; use a record instead.

---

## A typical mid-size example

A report query: top-spending customers in each country, with their order count and total.

```csharp
var report = orders
    .Where(o => o.CreatedAt >= DateTime.UtcNow.AddDays(-30))
    .Join(customers,
          o => o.CustomerId,
          c => c.Id,
          (o, c) => new { Order = o, Customer = c })
    .GroupBy(x => x.Customer.Country)
    .Select(g => new {
        Country = g.Key,
        Top5 = g.GroupBy(x => x.Customer)
                .Select(cg => new {
                    Customer = cg.Key,
                    OrderCount = cg.Count(),
                    Total = cg.Sum(x => x.Order.Total)
                })
                .OrderByDescending(c => c.Total)
                .Take(5)
                .ToList()
    });
```

This produces a fully-typed report from raw data. Replace `orders` and `customers` with `db.Orders` and `db.Customers` (EF Core IQueryable), and the same code translates to one SQL query.

---

## Internals — what LINQ actually generates

LINQ to Objects operators are **just extension methods** on `IEnumerable<T>`:

```csharp
public static class Enumerable {
    public static IEnumerable<T> Where<T>(this IEnumerable<T> source, Func<T, bool> predicate) {
        foreach (var item in source)
            if (predicate(item)) yield return item;
    }
}
```

That's the actual implementation of `Where` (simplified). It's an iterator method — defers execution until `MoveNext` is called.

`Select`, `OrderBy`, `Where`, `Skip`, `Take`, etc., are all iterator-based extension methods. They return new `IEnumerable<T>` objects without enumerating their input until someone iterates the result.

The runtime form: a small state-machine class generated by the compiler, holding the source enumerator and the predicate. Walks the source one item at a time.

### For LINQ to Providers (IQueryable)

The same extension method names exist on `IQueryable<T>`, but with `Expression<Func<>>` parameters:

```csharp
public static IQueryable<T> Where<T>(
    this IQueryable<T> source, Expression<Func<T, bool>> predicate)
{
    return source.Provider.CreateQuery<T>(
        Expression.Call(
            typeof(Queryable).GetMethod("Where")!.MakeGenericMethod(typeof(T)),
            source.Expression, Expression.Quote(predicate)));
}
```

The operator doesn't execute the predicate. It wraps the existing query's expression tree in a new tree node representing the `Where` + the lambda body. When you finally materialize (ToList, etc.), the provider walks the accumulated tree and translates it.

Two completely different mechanisms, one identical syntax. This is the LINQ magic.

---

## When LINQ is the right tool

✓ **Read-mostly transformations** — filtering, projecting, grouping data.
✓ **Database queries** via EF Core (IQueryable).
✓ **Pipeline-style processing** of in-memory sequences.
✓ **Declarative one-pass aggregations**.

✗ **Performance-critical inner loops** — manual `for` loops can be faster (no allocator pressure from iterators / delegates).
✗ **Side-effect-heavy operations** — LINQ assumes pure projections / filters. `ForEach`-style side effects belong in a regular `foreach`.
✗ **Complex stateful pipelines** — sometimes a regular method is clearer than a chain of operators.

---

## Reading order for this chapter

1. **§01 (this file)** — what LINQ is, why it exists.
2. **§02 Method Syntax** — the most common form, fluent operator chains.
3. **§03 Query Syntax** — SQL-like keywords, when they win.
4. **§04 Deferred Execution** — the #1 gotcha and how to think about it.
5. **§05 Standard Operators** — reference catalog.
6. **§06 Custom Operators** — writing your own.
7. **§07 Async LINQ** — `IAsyncEnumerable<T>` for streaming async data.
8. **§08 IQueryable** — how EF Core translates LINQ to SQL.
9. **§09 Performance & Pitfalls** — common traps, when LINQ is the wrong tool.

By the end you should be fluent in LINQ — able to read, write, and reason about query performance across in-memory and database scenarios.

→ Next: [02-MethodSyntax.md](02-MethodSyntax.md)
