# LINQ Query Syntax

## What it is

Query syntax is C#'s SQL-like keyword form for LINQ. The compiler rewrites it into method calls — same machinery, different surface.

```csharp
// Query syntax
var result = from u in users
             where u.IsActive
             orderby u.Name
             select new { u.Id, u.Name };

// Equivalent method syntax
var result = users
    .Where(u => u.IsActive)
    .OrderBy(u => u.Name)
    .Select(u => new { u.Id, u.Name });
```

Both produce identical IL. Query syntax was added in C# 3 alongside LINQ; some teams use it heavily, others avoid it. The mainstream choice today is **method syntax for everything**, with query syntax sometimes used for multi-source queries.

---

## Why it exists

Query syntax was meant to make LINQ feel like SQL to developers comfortable with database query languages. Where method syntax pipes through `.Where(...).Select(...)`, query syntax reads:

```csharp
from u in users
where u.IsActive
select u
```

For people thinking in "give me items from this source where condition select shape", the SQL-like form is intuitive.

In practice, method syntax has won. Most modern C# uses methods almost exclusively. Query syntax shines for specific cases (multi-source joins, `let`-introduced intermediate values) but is a niche tool.

---

## Anatomy

A query expression has parts:

```csharp
from x in source         // mandatory — at least one
where predicate          // optional, any number
let intermediate = expr  // optional, any number — introduce a named variable
orderby keys             // optional — sorting
join b in source2 on...  // optional — joining
group x by key           // optional — grouping (vs select)
select projection        // mandatory (or group ... by)
```

Every query starts with `from` and ends with `select` (or `group ... by`). Everything between is optional.

---

## `from` — source

```csharp
from u in users
```

Introduces a **range variable** (`u` here) that ranges over each item in the source. The source must be `IEnumerable<T>` or `IQueryable<T>`.

Multiple `from` clauses produce a Cartesian product (compiler emits `SelectMany`):

```csharp
from x in xs
from y in ys
select new { x, y }

// Compiles to:
xs.SelectMany(_ => ys, (x, y) => new { x, y });
```

For an N×M cross-product. Useful for "for each x, find each y" patterns.

---

## `where`

Filter. Compiles to `Where`.

```csharp
from u in users
where u.IsActive
select u
```

Multiple `where` clauses are combined (like multiple `&&`):

```csharp
from u in users
where u.IsActive
where u.Country == "US"
select u
```

Or you can use `&&`:

```csharp
from u in users
where u.IsActive && u.Country == "US"
select u
```

Same result, same SQL when translated.

---

## `orderby`

Sort. Multiple keys separated by commas:

```csharp
from u in users
orderby u.LastName, u.FirstName
select u

// = users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName)
```

Descending:

```csharp
from u in users
orderby u.Age descending, u.Name
select u

// = users.OrderByDescending(u => u.Age).ThenBy(u => u.Name)
```

---

## `select`

Project. The final expression.

```csharp
from u in users
select u.Name              // single property

from u in users
select new { u.Id, u.Name } // anonymous shape

from u in users
select new User { Id = u.Id, Name = u.Name.ToUpper() }   // explicit type
```

---

## `let` — introduce intermediate

`let` introduces a **named** intermediate value that's available in subsequent clauses. The compiler turns it into an anonymous-type tuple.

```csharp
from u in users
let fullName = $"{u.FirstName} {u.LastName}"
let isAdmin = u.Roles.Contains("admin")
where isAdmin
orderby fullName
select new { Name = fullName, u.Email }
```

This is the **single biggest advantage** of query syntax. Method syntax has no direct equivalent — you'd have to compute the expression twice or use complex projections.

```csharp
// Method syntax equivalent — more verbose
users
    .Select(u => new { User = u, FullName = $"{u.FirstName} {u.LastName}", IsAdmin = u.Roles.Contains("admin") })
    .Where(x => x.IsAdmin)
    .OrderBy(x => x.FullName)
    .Select(x => new { Name = x.FullName, x.User.Email });
```

If you use `let` heavily, query syntax stays compact. Otherwise, method syntax tends to win.

---

## `group ... by`

Group items by a key. Returns `IEnumerable<IGrouping<TKey, T>>`:

```csharp
from u in users
group u by u.Country
```

Each iteration gives an `IGrouping<string, User>` — a Key (the country) and the matching items.

Often you `group into` then continue the query:

```csharp
from u in users
group u by u.Country into countryGroup
where countryGroup.Count() > 10
orderby countryGroup.Key
select new { Country = countryGroup.Key, Count = countryGroup.Count() };
```

`into` introduces a new range variable for the rest of the query.

---

## `join` — inner join

```csharp
from o in orders
join c in customers on o.CustomerId equals c.Id
select new { OrderId = o.Id, CustomerName = c.Name }

// = orders.Join(customers, o => o.CustomerId, c => c.Id, (o, c) => new {...})
```

The `equals` keyword is significant — it must be exactly `equals` (not `==`). The left side is the outer key, right is the inner key.

### `join ... into` (group join / left outer join)

```csharp
from c in customers
join o in orders on c.Id equals o.CustomerId into customerOrders
select new { Customer = c, Orders = customerOrders.ToList() }
```

Each customer with a list of their orders (possibly empty). Compiles to `GroupJoin`.

To flatten to a true left outer join (include customers with NO orders):

```csharp
from c in customers
join o in orders on c.Id equals o.CustomerId into customerOrders
from o in customerOrders.DefaultIfEmpty()
select new { Customer = c.Name, OrderId = o?.Id }
```

`DefaultIfEmpty` yields `default(T)` (null for reference types) if the group is empty. Combined with another `from`, you get one row per customer, including those without orders.

---

## A multi-source query — when query syntax really wins

```csharp
var result =
    from o in orders
    join c in customers on o.CustomerId equals c.Id
    join p in products on o.ProductId equals p.Id
    where o.CreatedAt >= DateTime.UtcNow.AddDays(-7)
    let lineTotal = o.Quantity * p.Price
    where lineTotal >= 100m
    group new { o, c, p, lineTotal } by c into customerGroup
    orderby customerGroup.Sum(x => x.lineTotal) descending
    select new {
        Customer = customerGroup.Key.Name,
        Total = customerGroup.Sum(x => x.lineTotal),
        Items = customerGroup.Select(x => new { x.p.Name, x.lineTotal })
    };
```

This has two joins, two wheres, a let, a group, and a sort. The method-syntax equivalent is genuinely uglier because of the let and the multi-source group.

For pure `Where → Select → Order`, method syntax wins. For multi-source + let + group, query syntax often wins.

---

## What the compiler does

Query syntax is **purely syntactic sugar**. The compiler rewrites it to method calls:

```csharp
from x in xs where x > 0 select x * 2
```

becomes

```csharp
xs.Where(x => x > 0).Select(x => x * 2)
```

The compiler does this **structurally** — it doesn't understand "filter" or "project"; it just rewrites by pattern. This is why custom types implementing the right method names (`Where`, `Select`, `SelectMany`, `Join`, `GroupBy`, `OrderBy`, `ThenBy`, `GroupJoin`) can participate in query syntax — no need to implement an interface.

This pattern is sometimes called **duck-typed LINQ**. Most code uses the BCL's built-in extension methods, but in principle you can define `Where`/`Select`/etc. on any type.

---

## Internals — how query keywords map

| Query syntax | Method call |
|---|---|
| `from x in xs` | (starts the chain) |
| `from y in ys` (subsequent) | `xs.SelectMany(_ => ys, (x, y) => ...)` |
| `where p` | `.Where(x => p)` |
| `select x` | `.Select(x => x)` (often elided if trivial) |
| `orderby k` | `.OrderBy(x => k)` |
| `orderby k, k2` | `.OrderBy(...).ThenBy(...)` |
| `orderby k descending` | `.OrderByDescending(...)` |
| `let name = expr` | `.Select(x => new { x, name = expr })` + continuation |
| `group x by k` | `.GroupBy(x => k)` |
| `group x by k into g ... ` | `.GroupBy(...)` then continue with g |
| `join b in ys on lk equals rk` | `.Join(ys, x => lk, b => rk, (x, b) => ...)` |
| `join b in ys on lk equals rk into bs` | `.GroupJoin(...)` |

The compiler runs the rewrite at compile time. The IL is exactly what you'd get from writing the method syntax directly.

---

## Common patterns

### "WHERE clauses with multiple conditions"

```csharp
from u in users
where u.IsActive
where u.Email != null
where u.Roles.Any()
select u

// Same as:
users.Where(u => u.IsActive)
     .Where(u => u.Email != null)
     .Where(u => u.Roles.Any())
```

Pure style preference. EF Core combines them in SQL.

### Multi-from cross product

```csharp
from a in setA
from b in setB
where a.Key == b.Key
select new { a, b }

// Same as SelectMany — sometimes more readable than the equivalent Join
```

### Group join → flatten

```csharp
from c in customers
join o in orders on c.Id equals o.CustomerId into orderGroup
from o in orderGroup.DefaultIfEmpty()
select new { c.Name, OrderId = o?.Id }
```

Left outer join idiom. EF Core handles this well.

---

## When to use which syntax

Use **method syntax** when:
- Your query is mostly Where/Select/OrderBy.
- You need operators not in query syntax (`Take`, `Skip`, `Any`, `Count`, `Aggregate`, etc.).
- You're mixing in custom extension methods.

Use **query syntax** when:
- You have multiple `from` clauses or complex joins.
- You use `let` to name intermediate values.
- The team convention favors it for SQL-like clarity.

You can mix:

```csharp
var top10 = (from u in users
             where u.IsActive
             group u by u.Country into g
             select new { Country = g.Key, Count = g.Count() })
            .OrderByDescending(g => g.Count)
            .Take(10);
```

Parenthesize the query expression, then continue with methods.

---

## Common bugs and confusions

- **`equals` vs `==`** in join — must be `equals`. Compile error if you write `==`.
- **Reordering clauses** — `select` must come last (or `group`).
- **Outer variable order in join** — `on outerKey equals innerKey`. Get them swapped and the join silently produces wrong results.
- **`let` with side effects** — `let` is evaluated lazily, like the rest of LINQ. Don't put side effects in it.
- **Anonymous types in `group by`** — `group new { x, y } by x.Key` — the group's items are the anonymous type, not just x.

---

## Performance

Identical to method syntax — the compiler rewrites to the same calls. No overhead from choosing one over the other.

The decision is purely about readability for your team.

---

## A quick reference card

```csharp
from <var> in <source>           // mandatory
[where <predicate>]              // 0..n
[let <name> = <expr>]            // 0..n
[orderby <keys>]                 // 0..1
[join <var> in <source>          // 0..n
   on <outer> equals <inner>
   [into <group>]]
[group <expr> by <key>]          // 0..1 — alternative to select
   [into <newVar> + continuation]
[select <projection>]            // mandatory (or group by)
```

→ Next: [04-DeferredExecution.md](04-DeferredExecution.md)
