# Standard LINQ Operators — Complete Catalog

> The full set of LINQ operators in `System.Linq.Enumerable` (and mirrored in `System.Linq.Queryable`), grouped by purpose. Each has a one-line description, an example, and notes on common pitfalls.

This file is a reference. Scan once, return to look things up.

---

## Filtering

### `Where(predicate)`
Keep items where predicate is true.

```csharp
nums.Where(n => n > 0);                          // positive only
users.Where((u, i) => u.IsActive && i < 100);    // first 100 actives (with index)
```

### `OfType<T>()`
Keep items of a specific type. Skips others (no throw).

```csharp
object[] mixed = { 1, "hello", 2, 3.14 };
mixed.OfType<int>();   // 1, 2
```

### `Distinct()` / `DistinctBy(keySelector)` (.NET 6+)
Unique values / unique by key.

```csharp
nums.Distinct();                              // unique ints
users.DistinctBy(u => u.Country);              // one per country (first encountered)
```

For `Distinct`, equality is `EqualityComparer<T>.Default`. Pass a custom `IEqualityComparer<T>` if you need other semantics.

---

## Projection

### `Select(selector)`
Transform each item.

```csharp
users.Select(u => u.Name);
users.Select((u, i) => new { Index = i, User = u });
```

### `SelectMany(selector)`
Flatten nested sequences.

```csharp
orders.SelectMany(o => o.Items);                            // all items, flat
orders.SelectMany(o => o.Items, (o, item) => new { o, item }); // pair with parent
```

### `Cast<T>()`
Cast every element to T. Throws `InvalidCastException` on mismatch.

```csharp
ArrayList list = ...;
list.Cast<int>();
```

For tolerant casting, use `OfType<T>()` instead.

---

## Ordering

### `OrderBy(keySelector)` / `OrderByDescending(keySelector)`
Sort by a key. Stable sort.

```csharp
users.OrderBy(u => u.Name);
users.OrderByDescending(u => u.Age);
```

### `ThenBy(keySelector)` / `ThenByDescending(keySelector)`
Secondary sort. Only after `OrderBy` / `OrderByDescending`.

```csharp
users.OrderBy(u => u.LastName).ThenBy(u => u.FirstName);
```

### `Reverse()`
Reverse the order of elements. Materializes the entire input.

```csharp
nums.Reverse();
```

For sorting in reverse, prefer `OrderByDescending` — same result without materialization for IQueryable.

---

## Grouping

### `GroupBy(keySelector)`
Group items into `IGrouping<TKey, T>` buckets, one per distinct key.

```csharp
users.GroupBy(u => u.Country);
// Each: IGrouping<string, User> — Key + items

foreach (var group in users.GroupBy(u => u.Country)) {
    Console.WriteLine($"{group.Key}: {group.Count()}");
}
```

Variants:
```csharp
GroupBy(keySelector, elementSelector)              // group + project items
GroupBy(keySelector, resultSelector)               // group + final shape
GroupBy(keySelector, elementSelector, resultSelector)
GroupBy(keySelector, IEqualityComparer<TKey>)      // custom equality
```

---

## Joining

### `Join(inner, outerKey, innerKey, resultSelector)`
SQL-style inner join.

```csharp
orders.Join(customers, o => o.CustomerId, c => c.Id, (o, c) => new { o.Id, c.Name });
```

### `GroupJoin(inner, outerKey, innerKey, resultSelector)`
Each outer item paired with the GROUP of matching inner items (left outer join shape).

```csharp
customers.GroupJoin(orders, c => c.Id, o => o.CustomerId, (c, os) => new { Customer = c, Orders = os });
```

For full outer or right outer join, you'd combine GroupJoin + SelectMany + DefaultIfEmpty.

---

## Set operations

### `Concat(other)`
Concatenate two sequences (NOT deduplicated).

```csharp
list1.Concat(list2);
```

### `Union(other)`
Concat + distinct.

```csharp
list1.Union(list2);   // distinct union
```

### `Intersect(other)`
Distinct items present in both.

```csharp
list1.Intersect(list2);
```

### `Except(other)`
Distinct items in first but not second.

```csharp
list1.Except(list2);
```

All set ops use `EqualityComparer<T>.Default`. Custom equality via the `comparer` overload.

### `Zip(other, resultSelector)` / `Zip(other)` (.NET 6+)
Pair items by position.

```csharp
a.Zip(b, (x, y) => x + y);   // pairwise sum
a.Zip(b);                     // (T1, T2) tuples
a.Zip(b, c);                  // three-way zip (returns 3-tuples)
```

Stops at the shorter sequence.

---

## Partitioning

### `Take(count)` / `Take(Range)`
First N (or a range).

```csharp
nums.Take(5);
nums.Take(2..7);   // C# 8 ranges
nums.Take(..5);    // first 5
nums.Take(5..);    // skip 5, take the rest
```

### `Skip(count)`
Skip first N.

```csharp
nums.Skip(5);
```

### `TakeWhile(predicate)` / `SkipWhile(predicate)`
Take or skip while predicate holds (stops at first false).

```csharp
nums.TakeWhile(n => n < 10);
nums.SkipWhile(n => n < 10);
```

### `TakeLast(count)` / `SkipLast(count)` (.NET Core 2.0+)
From the end.

```csharp
nums.TakeLast(5);
nums.SkipLast(5);
```

### `Chunk(size)` (.NET 6+)
Split into chunks of size N.

```csharp
nums.Chunk(100);   // IEnumerable<int[]> — batches of 100
```

Useful for batching DB inserts, API calls, etc.

---

## Element access

### `First(predicate)` / `FirstOrDefault(predicate)`
First (matching). `First` throws if none; `FirstOrDefault` returns default.

```csharp
users.First();
users.First(u => u.IsActive);
users.FirstOrDefault(u => u.IsAdmin) ?? new User();
```

### `Last(predicate)` / `LastOrDefault(predicate)`
Same, but from the end. Iterates the whole sequence (unless source supports IList).

### `Single(predicate)` / `SingleOrDefault(predicate)`
Expects exactly one (or zero, for `OrDefault`). Throws on multiple matches.

```csharp
users.Single(u => u.Id == 5);          // throws if 0 or >1
users.SingleOrDefault(u => u.Email == email);   // null OK; throws if >1
```

Use `Single` when uniqueness is part of the contract (a primary key). Use `First` when you just want one.

### `ElementAt(index)` / `ElementAtOrDefault(index)`
Indexed access. O(1) for `IList<T>`; O(n) otherwise.

```csharp
nums.ElementAt(5);
```

Supports `Index` (C# 8+):

```csharp
nums.ElementAt(^1);   // last
```

### `MinBy(keySelector)` / `MaxBy(keySelector)` (.NET 6+)
Returns the item with min/max key.

```csharp
var oldest = users.MaxBy(u => u.Age);   // returns the User, not the age
```

vs `Max(u => u.Age)` which returns the age.

### `DefaultIfEmpty()`
If empty, yields one default value; otherwise, the original sequence.

```csharp
list.DefaultIfEmpty();
list.DefaultIfEmpty(fallbackItem);
```

Used for left outer joins.

---

## Quantifier

### `Any()` / `Any(predicate)`
Returns true if any (matching) item exists. Short-circuits.

```csharp
users.Any();
users.Any(u => u.IsActive);
```

### `All(predicate)`
True if every item matches.

```csharp
users.All(u => u.IsActive);
```

### `Contains(value)` / `Contains(value, comparer)`
Value present? Short-circuits.

```csharp
ids.Contains(5);
emails.Contains("x@y.com", StringComparer.OrdinalIgnoreCase);
```

---

## Aggregation

### `Count()` / `Count(predicate)` / `LongCount()`
Number of items / matching items.

```csharp
users.Count();
users.Count(u => u.IsActive);
```

For very large counts: `LongCount()` returns `long`.

### `Sum()` / `Sum(selector)`
Sum of items (or a projection). For numeric types only.

```csharp
nums.Sum();
orders.Sum(o => o.Total);
```

### `Average()` / `Average(selector)`
Mean. Returns `double` (for `int` source) or `decimal` (for `decimal` source).

```csharp
nums.Average();
users.Average(u => u.Age);
```

### `Min()` / `Min(selector)`, `Max()` / `Max(selector)`
Smallest / largest. For source items: `Min()`. For projected keys: `Min(u => u.Age)`.

For the source item with min/max key: use `MinBy` / `MaxBy` instead.

### `Aggregate(seed, func, resultSelector)`
General-purpose fold. Three overloads:

```csharp
// (1) Aggregate with no seed — uses first item
int product = nums.Aggregate((acc, n) => acc * n);   // throws if empty

// (2) Aggregate with seed
int sum = nums.Aggregate(0, (acc, n) => acc + n);
string csv = words.Aggregate("", (acc, s) => acc + s + ",");

// (3) Aggregate with seed + result selector
string upper = words.Aggregate(
    new StringBuilder(),
    (sb, w) => sb.Append(w).Append(","),
    sb => sb.ToString().ToUpper()
);
```

Most aggregations are clearer with `Sum`, `Max`, `Count`, etc. Use `Aggregate` for fold patterns that don't fit those.

---

## Conversion

### `ToList()` / `ToArray()`
Materialize to concrete collection.

```csharp
var list = query.ToList();
var array = query.ToArray();
```

### `ToDictionary(keySelector, valueSelector?, comparer?)`
Materialize to a dictionary.

```csharp
var byId = users.ToDictionary(u => u.Id);             // keyed by Id, value = User
var byIdName = users.ToDictionary(u => u.Id, u => u.Name);   // value = name
```

Throws if duplicate keys. For relaxed handling, use `GroupBy` + `ToDictionary`.

### `ToHashSet()`
Materialize to `HashSet<T>`.

```csharp
var ids = users.Select(u => u.Id).ToHashSet();
```

### `ToLookup(keySelector, valueSelector?)`
Materialize to `ILookup<TKey, TValue>` — like a dictionary but multiple values per key.

```csharp
var byCountry = users.ToLookup(u => u.Country);
foreach (var u in byCountry["US"]) { ... }
```

### `AsEnumerable()`
Force the static type to `IEnumerable<T>`. Useful for hopping out of `IQueryable` semantics.

```csharp
db.Users.Where(serverSide).AsEnumerable().Where(clientSide);   // first runs SQL, second runs in C#
```

### `AsQueryable()`
Force the static type to `IQueryable<T>`. Lets you use IQueryable-only operators.

---

## Generation (static)

### `Enumerable.Range(start, count)`
Integers `start, start+1, ..., start+count-1`.

```csharp
Enumerable.Range(0, 10);   // 0..9
Enumerable.Range(1, 5);    // 1..5
```

### `Enumerable.Repeat(value, count)`
Repeat the same value.

```csharp
Enumerable.Repeat("x", 5);   // x, x, x, x, x
```

### `Enumerable.Empty<T>()`
An empty sequence (singleton — no allocation per call).

```csharp
IEnumerable<int> empty = Enumerable.Empty<int>();
```

---

## Sequence comparison

### `SequenceEqual(other)` / `SequenceEqual(other, comparer)`
Elementwise equality.

```csharp
list1.SequenceEqual(list2);   // same length + each element equal
```

Stricter than set equality — order matters.

---

## Less common operators worth knowing

### `Append(item)` / `Prepend(item)` (.NET Core 1.0+)
Add a single item to one end.

```csharp
list.Append(42);
list.Prepend(0);
```

### `Reverse()`
(see above)

### `OrderBy().ThenBy()` chain — typed pipeline
Note: after `OrderBy`, the type is `IOrderedEnumerable<T>` — only `ThenBy`/`ThenByDescending` continue the sort. To do `Where` after sorting, the type "downgrades" back to `IEnumerable<T>`:

```csharp
list.OrderBy(x => x).Where(x => x > 0);   // OK — Where returns IEnumerable
list.OrderBy(x => x).ThenBy(y => y.Other);  // OK — same orderbygroup
```

---

## Equality customization

Many operators accept a custom `IEqualityComparer<T>`:

```csharp
nums.Distinct(EqualityComparer<int>.Default);
strings.Contains("hi", StringComparer.OrdinalIgnoreCase);
users.GroupBy(u => u.Email, StringComparer.OrdinalIgnoreCase);
```

For LINQ on case-insensitive strings, `StringComparer` is your friend.

---

## A practical reference cheat sheet

```csharp
// Filter
.Where(pred)
.OfType<T>()
.Distinct() / .DistinctBy(key)

// Project
.Select(sel)
.SelectMany(sel)
.Cast<T>()

// Order
.OrderBy(key) / .OrderByDescending(key)
.ThenBy / .ThenByDescending
.Reverse()

// Group
.GroupBy(key)

// Join
.Join(inner, outerKey, innerKey, result)
.GroupJoin(inner, outerKey, innerKey, result)

// Set
.Concat / .Union / .Intersect / .Except
.Zip

// Partition
.Take(n) / .TakeLast(n) / .TakeWhile(pred)
.Skip(n) / .SkipLast(n) / .SkipWhile(pred)
.Chunk(size)

// Element
.First / .FirstOrDefault
.Last / .LastOrDefault
.Single / .SingleOrDefault
.ElementAt(i) / .ElementAtOrDefault(i)
.MinBy(key) / .MaxBy(key)
.DefaultIfEmpty(val?)

// Test
.Any / .Any(pred) / .All(pred) / .Contains(val)

// Aggregate
.Count / .LongCount / .Sum / .Average / .Min / .Max / .Aggregate

// Materialize
.ToList / .ToArray / .ToDictionary / .ToHashSet / .ToLookup

// Convert
.AsEnumerable / .AsQueryable

// Generate
Enumerable.Range / .Repeat / .Empty

// Misc
.Append / .Prepend / .SequenceEqual
```

---

## Performance hints per operator

- `Count()` on `ICollection<T>` (List, Array, etc.) is O(1) — uses the underlying `.Count`. On a deferred query, it's O(n).
- `Any()` is O(1) for the first match. O(n) worst case.
- `Contains(value)` on `HashSet<T>` is O(1); on `List<T>` it's O(n).
- `First()`, `FirstOrDefault()`, `Take(n)` short-circuit — they don't iterate beyond what's needed.
- `Last()`, `LastOrDefault()` iterate the whole sequence (unless source is IList).
- `Reverse()` materializes — O(n) memory.
- `OrderBy()` materializes to sort — O(n) memory.
- `GroupBy()` materializes — O(n) memory.
- `Distinct()` materializes a HashSet under the hood — O(n) memory.

---

## When to use what — selected guidance

| Goal | Operator |
|---|---|
| Existence test | `Any()` (not `Count() > 0`) |
| Find one matching | `FirstOrDefault(pred)` |
| Find unique matching | `SingleOrDefault(pred)` if uniqueness is guaranteed |
| Top N by criterion | `OrderByDescending(...).Take(N)` |
| Largest item | `MaxBy(key)` |
| Largest key | `Max(selector)` |
| Group + count | `GroupBy(...).Select(g => new { g.Key, Count = g.Count() })` |
| Dictionary lookup | `ToDictionary(keySel)` (or `ToLookup` for multiple per key) |
| Batch processing | `Chunk(size)` |
| Avoid multi-enumeration | Materialize with `ToList()` first |

→ Next: [06-CustomOperators.md](06-CustomOperators.md)
