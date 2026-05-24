# LINQ Method Syntax

## What it is

Method syntax (sometimes called "fluent" or "extension method" syntax) writes LINQ queries as **chains of method calls** on `IEnumerable<T>` or `IQueryable<T>`, each taking a lambda for filtering, projecting, or aggregating.

```csharp
var result = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => u.Email)
    .ToList();
```

This is **the dominant form** of LINQ in modern C# code. Most teams use it exclusively; query syntax (the SQL-like form) is reserved for cases it actually wins.

---

## Why it's preferred

Method syntax:
- **Mirrors the API surface** — every operator has a 1:1 method form.
- **Composes naturally** — chain calls without intermediate variables.
- **All operators available** — query syntax only covers a subset.
- **Reads like a pipeline** — filter → transform → aggregate.

Query syntax wins for specific cases (multi-source `join into`, `let` for intermediate values). Otherwise, method syntax is shorter and more general.

---

## The basic operators

### `Where(predicate)` — filter

```csharp
var actives = users.Where(u => u.IsActive);
```

Returns `IEnumerable<T>` with only items matching the predicate. Lazy.

### `Select(selector)` — project (transform)

```csharp
var names = users.Select(u => u.Name);                        // IEnumerable<string>
var pairs = users.Select(u => new { u.Id, u.Name });           // anonymous type
var indexed = users.Select((u, i) => new { Index = i, User = u });  // with index
```

Returns `IEnumerable<TResult>` after applying the projection.

### `OrderBy(keySelector)` / `OrderByDescending(...)` — sort

```csharp
var sorted = users.OrderBy(u => u.Name);
var byAgeDesc = users.OrderByDescending(u => u.Age);
```

### `ThenBy` / `ThenByDescending` — secondary sort

```csharp
var sorted = users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName);
```

Only valid after an `OrderBy` (or `ThenBy`). Stable sort — preserves order of equal-key items.

### `GroupBy(keySelector)` — group

```csharp
var byCountry = users.GroupBy(u => u.Country);
// each item is IGrouping<string, User> — Key + items

foreach (var group in byCountry) {
    Console.WriteLine($"{group.Key}: {group.Count()}");
    foreach (var u in group) Console.WriteLine($"  - {u.Name}");
}
```

### `Take(n)` / `Skip(n)` — partition

```csharp
var firstTen = users.Take(10);
var afterTen = users.Skip(10);
var page = users.Skip(20).Take(10);   // page 3 of 10
```

C# 8 also supports `Take(Range)`:

```csharp
var middle = users.Take(10..20);
```

### `First` / `FirstOrDefault` / `Single` / `SingleOrDefault`

```csharp
var alice = users.First(u => u.Name == "Alice");          // throws if none
var alice2 = users.FirstOrDefault(u => u.Name == "Alice"); // null if none
var only = users.Single(u => u.Id == 5);                   // throws if 0 or >1
var only2 = users.SingleOrDefault(u => u.Id == 5);         // null if 0; throws if >1
```

`First` accepts any matching item. `Single` insists on exactly one.

### `Any` / `All` — quantifier

```csharp
bool hasActive = users.Any(u => u.IsActive);
bool allActive = users.All(u => u.IsActive);
bool any = users.Any();   // any items at all?
```

### `Count` / `LongCount`

```csharp
int total = users.Count();
int activeCount = users.Count(u => u.IsActive);   // matches predicate
```

### `Sum` / `Average` / `Min` / `Max` / `MinBy` / `MaxBy`

```csharp
int totalAge = users.Sum(u => u.Age);
double avgAge = users.Average(u => u.Age);
int oldest = users.Max(u => u.Age);
var oldestUser = users.MaxBy(u => u.Age);   // .NET 6+ — returns the User
var youngestUser = users.MinBy(u => u.Age);
```

`MaxBy` / `MinBy` return the item with the largest/smallest key, NOT the key itself.

### `ToList` / `ToArray` / `ToDictionary` / `ToHashSet`

These **materialize** the query:

```csharp
List<User> list = users.Where(u => u.IsActive).ToList();
User[] array = users.Where(u => u.IsActive).ToArray();
Dictionary<int, string> dict = users.ToDictionary(u => u.Id, u => u.Name);
HashSet<int> ids = users.Select(u => u.Id).ToHashSet();
```

After materialization, the result is in memory and can be enumerated repeatedly without re-running the query.

---

## Chaining

Operators chain naturally because each returns an `IEnumerable<T>` (or `IQueryable<T>`):

```csharp
var result = users
    .Where(u => u.IsActive)
    .Where(u => u.Country == "US")
    .OrderBy(u => u.LastName)
    .ThenBy(u => u.FirstName)
    .Skip(10)
    .Take(20)
    .Select(u => new { u.Name, u.Email })
    .ToList();
```

Reads top-to-bottom. Each line refines the previous. Like a Unix pipeline:
```
users | where active | where US | order by lastname | order by firstname | skip 10 | take 20 | select name+email | toList
```

For LINQ to Providers (EF Core), the entire chain compiles to ONE SQL query, not many.

---

## Some less obvious operators

### `SelectMany` — flatten

```csharp
public class Order { public List<Item> Items { get; set; } = new(); }
List<Order> orders = ...;

// Flatten all items from all orders into one sequence
var allItems = orders.SelectMany(o => o.Items);
```

`Select` would give you `IEnumerable<List<Item>>`. `SelectMany` flattens one level, giving `IEnumerable<Item>`.

With index:
```csharp
var allItems = orders.SelectMany((o, i) => o.Items.Select(item => new { OrderIdx = i, Item = item }));
```

### `Join` — SQL inner join

```csharp
var query = orders.Join(
    customers,
    o => o.CustomerId,
    c => c.Id,
    (o, c) => new { o.Id, CustomerName = c.Name }
);
```

Arguments: inner sequence, outer-key selector, inner-key selector, result selector. Returns the cartesian product where keys match.

### `GroupJoin` — left outer join

```csharp
var query = customers.GroupJoin(
    orders,
    c => c.Id,
    o => o.CustomerId,
    (c, os) => new { Customer = c, Orders = os.ToList() }
);
```

Each customer paired with all their orders (possibly an empty list).

### `Aggregate` — reduce

```csharp
int sum = nums.Aggregate(0, (acc, n) => acc + n);
string joined = nums.Aggregate("", (acc, n) => acc + n + ",");
```

General "fold" operation. Takes a seed and a binary function.

Without a seed, uses the first element as the seed:
```csharp
int min = nums.Aggregate((a, b) => a < b ? a : b);   // throws if empty
```

### `Distinct` / `DistinctBy`

```csharp
var uniqueEmails = users.Select(u => u.Email).Distinct();
var oneUserPerCountry = users.DistinctBy(u => u.Country);   // .NET 6+
```

`DistinctBy` returns one item per distinct key (the first one encountered).

### `Concat` / `Union` / `Intersect` / `Except`

```csharp
var combined = list1.Concat(list2);      // simple append
var union = list1.Union(list2);          // distinct union
var common = list1.Intersect(list2);     // distinct intersection
var only1 = list1.Except(list2);         // in list1 but not list2 (distinct)
```

### `Reverse`

```csharp
var reversed = users.OrderBy(u => u.Name).Reverse();
```

Reverses the entire sequence. Use `OrderByDescending` if you just want to sort the other way (less memory, more efficient).

### `Cast<T>` / `OfType<T>`

```csharp
ArrayList list = new() { 1, 2, 3 };
var ints = list.Cast<int>();   // throws on mismatch

object[] mixed = { 1, "hello", 2, 3.14 };
var onlyInts = mixed.OfType<int>();   // returns only ints, no throw
```

---

## Method syntax with index

Some operators have overloads with element index:

```csharp
var indexed = users.Select((u, i) => new { Index = i, User = u });
var firstThree = users.TakeWhile((u, i) => i < 3);
var withIndex = users.Where((u, i) => i % 2 == 0);   // even indices
```

In query syntax this is harder to express. Method syntax wins here.

---

## Operator-to-operator transition (LINQ-to-Objects to LINQ-to-Provider)

When you write LINQ on `IQueryable<T>` instead of `IEnumerable<T>`, the lambdas become **expression trees** instead of compiled delegates. The same method-syntax call site works for both:

```csharp
IEnumerable<User> users = ...;             // local list
var local = users.Where(u => u.IsActive);  // Func<User, bool>

IQueryable<User> dbUsers = db.Users;        // EF Core
var remote = dbUsers.Where(u => u.IsActive); // Expression<Func<User, bool>>
```

Switching between in-memory and database is often just a matter of changing the source. The query syntax adapts.

If you accidentally convert to `IEnumerable<T>` mid-query, the rest runs in-memory:

```csharp
var bad = db.Users
    .AsEnumerable()                          // ⚠ pulls all users into memory!
    .Where(u => u.IsActive)                  // runs in C# now
    .Select(u => new { u.Name });
```

`AsEnumerable()` is the explicit "I want IEnumerable from here on" switch. Useful when you specifically want client-side evaluation, e.g., for methods EF can't translate. But unintentional use is a perf disaster.

[§08 IQueryable](08-IQueryable.md) covers this in detail.

---

## Internals — what the chain compiles to

A simple chain:

```csharp
var result = users.Where(u => u.IsActive).Select(u => u.Name).ToList();
```

This is just nested static method calls:

```csharp
var result = Enumerable.ToList(
    Enumerable.Select(
        Enumerable.Where(users, u => u.IsActive),
        u => u.Name
    )
);
```

Each operator is an **extension method** on `IEnumerable<T>`. The compiler resolves `users.Where(...)` to `Enumerable.Where(users, ...)`.

`Where` returns an iterator (a state-machine class) that wraps `users` and the predicate. `Select` wraps that. Each operator is a thin layer.

When `ToList()` finally enumerates:
1. ToList calls `GetEnumerator()` on the Select wrapper.
2. Select's enumerator calls `MoveNext` on Where's enumerator.
3. Where's enumerator calls `MoveNext` on users's enumerator.
4. As items flow up: Where filters; Select projects; ToList accumulates.

Each item flows through the chain **once**. No intermediate lists are materialized. This is why LINQ to Objects is efficient — pipelined iteration, not multi-pass batch processing.

---

## Common patterns

### Pagination

```csharp
int pageSize = 20, pageNumber = 3;
var page = users
    .OrderBy(u => u.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToList();
```

### Group-and-aggregate

```csharp
var report = orders
    .GroupBy(o => o.Status)
    .Select(g => new { Status = g.Key, Count = g.Count(), Total = g.Sum(o => o.Total) })
    .ToList();
```

### Filter + project + sort

```csharp
var topActive = users
    .Where(u => u.IsActive)
    .OrderByDescending(u => u.LastLogin)
    .Select(u => new { u.Name, u.LastLogin })
    .Take(10);
```

### Find with default

```csharp
var alice = users.FirstOrDefault(u => u.Name == "Alice") ?? new User { Name = "Default" };
```

### Distinct values

```csharp
var countries = users.Select(u => u.Country).Distinct().OrderBy(c => c);
```

### Index lookup

```csharp
var byId = users.ToDictionary(u => u.Id);
var alice = byId[5];
```

---

## Common bugs

- **Forgetting `.ToList()` / `.ToArray()`** when you need to enumerate multiple times — each enumeration re-runs the chain.
- **`Count() > 0`** instead of `Any()` — `Count()` iterates the whole sequence; `Any()` stops at the first item.
- **Side effects in `Select`** — `Select(u => { Log(u); return u; })` runs the log every enumeration.
- **`First()` instead of `FirstOrDefault()`** — throws if no match. Catches you when the data unexpectedly changes.
- **`Single()` when you meant `First()`** — throws on multiple matches. Use only when you genuinely require exactly one.
- **Multiple enumeration of `IEnumerable<T>`** — see [§04 Deferred Execution](04-DeferredExecution.md).

---

## Performance notes

- Each operator adds a small overhead — a state-machine allocation, delegate calls per item.
- LINQ over arrays/`List<T>` is fast but not free; for very hot loops, hand-written `for` can be 2-3× faster.
- For large data, prefer pre-materialization (`ToList`) if you'll iterate multiple times.
- For databases (IQueryable), avoid `AsEnumerable()` unless intentional — it pulls everything client-side.

---

## When method syntax wins vs query syntax

Method syntax wins when:
- You use operators query syntax doesn't support (`Take`, `Skip`, `Distinct`, `Any`, etc.).
- The chain is short and reads naturally.
- You want to mix in custom extension methods.
- You're using `Index`-based overloads.

Query syntax wins when:
- You have multiple `from` clauses (cross joins or `SelectMany`).
- You use `let` for intermediate values.
- You use `join ... into` patterns (group joins).
- You find it more readable for complex joins.

You can mix freely — wrap a query expression in parentheses and continue with methods.

---

## Method-syntax-only operators

These have no query-syntax equivalent:
`Take`, `Skip`, `Distinct`, `DistinctBy`, `Any`, `All`, `Count`, `Sum`, `Min`, `Max`, `MinBy`, `MaxBy`, `Average`, `Aggregate`, `First`, `Last`, `Single`, `ElementAt`, `Concat`, `Union`, `Intersect`, `Except`, `SequenceEqual`, `Reverse`, `OfType`, `Cast`, `ToList`, `ToArray`, `ToDictionary`, `ToHashSet`, `ToLookup`, `AsEnumerable`, `AsQueryable`, etc.

If your query uses any of these (most do), at some point you'll write method syntax. The Q→M tools in IDEs convert between forms when both work.

→ Next: [03-QuerySyntax.md](03-QuerySyntax.md)
