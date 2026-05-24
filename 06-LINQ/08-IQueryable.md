# IQueryable — LINQ to Providers

## What it is

`IQueryable<T>` is the LINQ interface for **remote data**. Operators don't execute against in-memory delegates — they build an **expression tree** that a **provider** translates to its native query language (SQL for EF Core, OData URLs for OData clients, Cosmos SQL for Cosmos DB, etc.).

```csharp
IQueryable<User> users = db.Users;           // EF Core's IQueryable<User>
var actives = users.Where(u => u.IsActive);   // builds expression tree, NOT a Func
var list = actives.ToList();                   // NOW runs SQL: SELECT * FROM Users WHERE IsActive = 1
```

`IQueryable<T>` looks identical to `IEnumerable<T>` at the call site, but the underlying machinery is fundamentally different. Understanding that difference is the difference between writing fast queries and accidentally pulling a whole table into memory.

---

## The interface

```csharp
public interface IQueryable<T> : IEnumerable<T> {
    Type ElementType { get; }
    Expression Expression { get; }
    IQueryProvider Provider { get; }
}
```

Every `IQueryable<T>` carries:
- An **expression tree** (`Expression`) describing the query built so far.
- A **provider** (`IQueryProvider`) that knows how to execute the tree.

When you call `.Where(...)` on an `IQueryable<T>`, it doesn't filter anything yet — it builds a new `IQueryable<T>` whose `Expression` is "the previous tree, wrapped in a Where node with this lambda."

Execution happens only when:
- You enumerate (`foreach`, `ToList`, etc.).
- You call a terminal operator (`Count`, `Sum`, `First`, etc.).

At that moment, the provider walks the tree and produces a SQL query (or Cosmos query, or HTTP request, etc.), executes it, and returns results.

---

## Why expression trees, not Funcs?

`IEnumerable<T>.Where` takes a `Func<T, bool>` — a compiled delegate. The provider can't inspect it; it can only call it.

`IQueryable<T>.Where` takes an `Expression<Func<T, bool>>` — a data structure representing the lambda. The provider can:
- Walk the tree.
- See property accesses, method calls, comparisons.
- Translate them to SQL (or other backends).

This is why this works:

```csharp
db.Users.Where(u => u.IsActive && u.Country == "US").ToList();
```

EF Core sees the lambda tree `u => u.IsActive && u.Country == "US"`, translates it to `WHERE IsActive = 1 AND Country = 'US'`, and the SQL runs.

Without expression trees, EF would have a black-box `Func<User, bool>` and couldn't possibly know to write SQL — it would have to load every user and filter in memory.

---

## Anatomy of an IQueryable query

```csharp
var query = db.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new { u.Id, u.Name });
```

At this point, `query` is an `IQueryable` whose Expression looks like:

```
Select(
    OrderBy(
        Where(
            Constant(db.Users),
            u => u.IsActive
        ),
        u => u.Name
    ),
    u => new { u.Id, u.Name }
)
```

It's a tree of method calls. No SQL yet, no work done.

Then:

```csharp
var list = query.ToList();
```

`ToList` is materialization — calls `GetEnumerator()` on the IQueryable, which triggers `Provider.Execute(Expression)`. EF Core walks the tree, emits SQL:

```sql
SELECT Id, Name FROM Users
WHERE IsActive = 1
ORDER BY Name
```

Executes, fetches rows, materializes `User` (or anonymous) objects.

**One SQL query for the whole chain.** That's the win.

---

## What a provider can translate — and what it can't

Each provider has its own translation rules. For EF Core (the most common):

**Translates fine:**
- Basic LINQ operators: Where, Select, OrderBy, Skip, Take, Join, GroupBy.
- Properties of entities.
- Comparison operators (`==`, `<`, etc.).
- Common methods: `string.Contains`, `string.StartsWith`, `Math.Abs`.
- `DateTime` arithmetic and component access.
- `Any`, `All`, `Count`, `Sum`, `Min`, `Max`.

**Sometimes doesn't translate (or falls back to client eval):**
- Custom methods (`MyHelper(u)`).
- Complex C# language features (some patterns).
- Specific BCL methods the provider doesn't recognize.

**Old EF Core versions** (5 and earlier) silently fell back to client evaluation — pulling rows and running the predicate in C#. Massive perf surprise.

**EF Core 6+** throws an `InvalidOperationException` at runtime when a query can't translate. You see the failure immediately.

To force client evaluation when you really mean it: call `.AsEnumerable()` to switch from `IQueryable<T>` to `IEnumerable<T>` mid-pipeline.

```csharp
db.Users
    .Where(u => u.IsActive)                          // SQL
    .AsEnumerable()                                   // ← switch to in-memory
    .Where(u => MyHelper(u))                          // C#
    .ToList();
```

`AsEnumerable()` doesn't materialize; it just changes the static type. After that point, operators take `Func`s and run in memory.

**Caveat**: everything **before** AsEnumerable runs as SQL. Everything **after** runs in C#. If the SQL produces 1M rows and you filter to 10 in C#, you've pulled 1M rows for no reason. Filter as much as you can in the IQueryable part.

---

## Comparing IEnumerable vs IQueryable

```csharp
// IEnumerable — runs in memory
List<User> localUsers = new() { /* ... */ };
var active = localUsers.Where(u => u.IsActive);   // Func, runs in C#

// IQueryable — runs via provider
IQueryable<User> dbUsers = db.Users;
var active = dbUsers.Where(u => u.IsActive);       // Expression, runs as SQL
```

Same source code. Vastly different mechanics.

Methods chained off an `IQueryable<T>` stay queryable — they continue building the tree. Methods chained off `IEnumerable<T>` (or after `AsEnumerable()`) run in memory.

---

## Common bugs and confusions

### Accidentally materializing too early

```csharp
db.Users.ToList().Where(u => u.IsActive).ToList();
// ⚠ ToList() pulls ALL users from DB, then filters in C#.
// What you wanted:
db.Users.Where(u => u.IsActive).ToList();
```

The first `ToList()` defeats EF's filtering. Common in confused refactorings.

### Calling C# methods inside expressions

```csharp
db.Users.Where(u => MyHelper(u.Name)).ToList();
// EF Core 6+ throws: cannot translate.
```

EF can only translate what it understands. To use C# methods, materialize first:

```csharp
db.Users.AsEnumerable().Where(u => MyHelper(u.Name)).ToList();
// But this pulls ALL users!
```

Better: write the equivalent logic as a Where clause EF can translate.

### `IEnumerable<T>` return from a repository

```csharp
public IEnumerable<User> GetUsers() => db.Users.Where(u => u.IsActive);   // ⚠
```

This compiles because `IQueryable<T> : IEnumerable<T>`. But the caller sees `IEnumerable<T>` and might use LINQ operators that work in memory, defeating EF's translation.

For repositories: either return `IQueryable<T>` (let caller compose more) or materialize (`.ToList()`) before returning.

### Multiple enumeration

```csharp
var query = db.Users.Where(u => u.IsActive);
int count = query.Count();         // SQL: SELECT COUNT(*) ...
var first = query.First();          // SQL: SELECT TOP 1 ...
var list = query.ToList();          // SQL: SELECT ... again
// THREE round-trips!
```

Each terminal operator runs a fresh SQL query. Materialize once:

```csharp
var users = db.Users.Where(u => u.IsActive).ToList();
int count = users.Count;
var first = users.First();
```

### Lambdas inside LINQ that don't translate

```csharp
db.Users.Where(u => u.Name.Length > 5).ToList();   // EF translates to SQL: LEN(Name) > 5
db.Users.Where(u => u.Name.Reverse().Take(3).Count() == 3).ToList();   // probably can't translate
```

Stick to simple comparisons and string methods you know EF handles.

---

## Internals — what an IQueryable provider does

When you write:

```csharp
db.Users.Where(u => u.IsActive)
```

The compiler emits a call to `Queryable.Where`:

```csharp
public static IQueryable<TSource> Where<TSource>(
    this IQueryable<TSource> source,
    Expression<Func<TSource, bool>> predicate)
{
    return source.Provider.CreateQuery<TSource>(
        Expression.Call(
            ((MethodInfo)MethodBase.GetCurrentMethod()!).MakeGenericMethod(typeof(TSource)),
            source.Expression,
            Expression.Quote(predicate)
        ));
}
```

`Expression.Quote(predicate)` wraps the predicate tree (so it doesn't get evaluated). `Expression.Call(...)` creates a new tree node: "call Where(source.Expression, predicate)." `source.Provider.CreateQuery(...)` returns a new `IQueryable<T>` with that tree.

The provider holds onto the tree. When materialization happens (`ToList`, `Count`, etc.), the provider:

1. Walks the entire accumulated tree (visitor pattern).
2. Translates each node into its target language (SQL clauses for EF Core).
3. Emits a query.
4. Executes against the database.
5. Reads rows.
6. Materializes them into C# objects (entity types, projections, anonymous types).

EF Core's query translator is thousands of lines of code, handling every translatable expression type. The architecture is shared across providers — the abstract pipeline (Expression → SQL → results) is in EF Core's core, and database-specific code (SQL Server vs SQLite vs PostgreSQL) plugs in.

---

## Common patterns

### Build a query conditionally

```csharp
IQueryable<User> query = db.Users;
if (filter.Active.HasValue) query = query.Where(u => u.IsActive == filter.Active);
if (!string.IsNullOrEmpty(filter.NameContains))
    query = query.Where(u => u.Name.Contains(filter.NameContains));
if (filter.SkipDeleted) query = query.Where(u => !u.IsDeleted);

var results = await query.ToListAsync();
```

Each `Where` appends to the expression tree. The provider sees the combined tree and emits a single SQL query.

### Paged query

```csharp
var page = await db.Users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Id)
    .Skip((pageNumber - 1) * pageSize)
    .Take(pageSize)
    .ToListAsync();

// Plus a separate query for total count:
var total = await db.Users.Where(u => u.IsActive).CountAsync();
```

EF translates Skip/Take to `OFFSET ... FETCH NEXT ...` (or LIMIT/OFFSET depending on dialect).

### Projection to DTO

```csharp
var dtos = await db.Users
    .Where(u => u.IsActive)
    .Select(u => new UserDto {
        Id = u.Id,
        Name = u.Name,
        OrderCount = u.Orders.Count()
    })
    .ToListAsync();
```

EF translates this to a SELECT with just the projected columns + an aggregated subquery for OrderCount. Much faster than loading the full User + Orders.

### Eager loading with Include

```csharp
var users = await db.Users
    .Where(u => u.IsActive)
    .Include(u => u.Orders)
    .ThenInclude(o => o.Items)
    .ToListAsync();
```

`Include` (EF Core specific) tells the provider to also load related data. Translates to a JOIN (or to a separate query for `.AsSplitQuery`).

---

## When to use IQueryable vs materialize

| Stage | Use IQueryable | Materialize (ToList/Async) |
|---|---|---|
| Filter / project / sort to reduce data | ✓ | ✗ |
| Apply filters that depend on database state | ✓ | ✗ |
| Apply filters that depend on C# logic that can't be translated | ✗ (materialize first) | ✓ |
| Multiple uses of the same query result | ✗ | ✓ |
| Combine with non-DB data | ✗ | ✓ |
| Return from a method | Depends — `IQueryable` for composability, `List<T>` for materialized | |

---

## A quick decision tree

```
You have IQueryable<T> and want to apply LINQ:
├── Is your LINQ translatable to the provider's language?
│   ├── Yes → keep using IQueryable. Stays as one DB query.
│   └── No → call .AsEnumerable() first, OR rewrite to be translatable.
└── Are you done filtering/projecting?
    └── Yes → call .ToList() / .ToListAsync() to materialize.
```

---

## Performance

- **IQueryable wins** when you can push work to the DB — filtering 1M rows in SQL is dramatically faster than C#.
- **In-memory LINQ wins** when data is already loaded — no SQL overhead.
- **IQueryable's tree-building costs** are negligible compared to the SQL execution.
- **Watch out for accidental in-memory eval** (AsEnumerable + filtering) — pulls more rows than needed.

EF Core 6+ logs every SQL query at Information level — turn on EF logging in dev and watch what runs. It's often eye-opening.

---

## When IQueryable is the wrong tool

✗ Data is already in memory (just use IEnumerable / LINQ).
✗ Operations are inherently CPU-bound (LINQ to Objects is faster than a round trip).
✗ The query is so complex it's better to write SQL directly.
✗ You're caching the result anyway — materialize once.

---

## A note on Dapper and others

Dapper isn't LINQ — it takes raw SQL. For complex queries beyond what EF translates well, Dapper is a popular companion. EF Core has `FromSql` and `ExecuteSql` to bridge.

For most CRUD: EF Core via LINQ.
For complex reporting: SQL via Dapper or `FromSql`.
For high throughput: bench both, pick what wins.

→ Next: [09-PerformanceAndPitfalls.md](09-PerformanceAndPitfalls.md)
