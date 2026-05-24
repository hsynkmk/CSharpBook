# 04 — LINQ — Coding Questions

> Predict the output / find the bug. (Concepts: [04-LINQ.md](04-LINQ.md))

---

### Q1 — Deferred execution
```csharp
var nums = new List<int> { 1, 2, 3 };
var query = nums.Where(n => n > 1);
nums.Add(4);
Console.WriteLine(string.Join(",", query));
```
<details><summary>Answer</summary>

**`2,3,4`**. The query is **deferred** — it runs when enumerated (`string.Join`), *after* `4` was added. It re-reads the live source. (If you'd done `.ToList()` before adding 4, you'd get `2,3`.)
</details>

---

### Q2 — Captured variable in a deferred query
```csharp
int threshold = 2;
var nums = new[] { 1, 2, 3 };
var q = nums.Where(n => n > threshold);
threshold = 0;
Console.WriteLine(string.Join(",", q));
```
<details><summary>Answer</summary>

**`1,2,3`**. The predicate reads `threshold` **when enumerated** (now 0), so all pass. Deferred execution + closure capture combine here.
</details>

---

### Q3 — Multiple enumeration: what's the bug?
```csharp
IEnumerable<int> Numbers() { Console.WriteLine("generating"); return new[] {1,2,3}; }
var q = Numbers().Where(n => n > 1);
var count = q.Count();
var first = q.First();        // ?
```
<details><summary>Answer</summary>

`Numbers()` (and the `Where` pipeline) **executes twice** — once for `Count()`, once for `First()`. With a DB source that's two round-trips. **Fix:** materialize once: `var list = q.ToList();` then use `list.Count` / `list[0]`.
</details>

---

### Q4 — First vs Single
```csharp
var nums = new[] { 1, 2, 2, 3 };
Console.WriteLine(nums.First(n => n == 2));
Console.WriteLine(nums.Single(n => n == 2));   // ?
```
<details><summary>Answer</summary>

`First` prints **`2`**. `Single` **throws** `InvalidOperationException` ("Sequence contains more than one matching element") — there are two 2s. `Single` means *exactly one*.
</details>

---

### Q5 — What runs where? (EF Core)
```csharp
var bad  = db.Orders.AsEnumerable().Where(o => o.Total > 100).ToList();
var good = db.Orders.Where(o => o.Total > 100).ToList();
```
<details><summary>Answer</summary>

`good` filters in **SQL** (`IQueryable` → `WHERE Total > 100`). `bad` calls `AsEnumerable()` first → it pulls the **entire Orders table** into memory, then filters client-side. Huge difference at scale. Keep filters on `IQueryable`.
</details>

---

### Q6 — Any vs Count
```csharp
// huge list / DB table
bool a = items.Count() > 0;
bool b = items.Any();
```
<details><summary>Answer</summary>

Both give the same bool, but `Any()` **short-circuits** at the first element (and emits SQL `EXISTS`), while `Count()` may enumerate/`COUNT(*)` everything. Prefer **`Any()`** for existence checks.
</details>

---

### Q7 — Predict the output
```csharp
var result = new[] { 1, 2, 3, 4 }
    .Where(n => { Console.Write($"W{n} "); return n % 2 == 0; })
    .Select(n => { Console.Write($"S{n} "); return n * 10; })
    .First();
Console.WriteLine($"=> {result}");
```
<details><summary>Answer</summary>

**`W1 W2 S2 => 20`**. Streaming + deferred: `First` pulls elements one at a time. `W1` (1 fails), `W2` (2 passes) → `S2` → `First` has its element and **stops** (never touches 3, 4). LINQ is lazy and pull-based.
</details>

---

### Q8 — GroupBy buffering
```csharp
var q = Enumerable.Range(1, 5).GroupBy(n => n % 2);
// is GroupBy lazy like Where?
```
<details><summary>Answer</summary>

`GroupBy` is deferred to *start*, but it's a **buffering** operator — when enumerated it must read the **entire** source to form groups (unlike `Where`/`Select` which stream). `OrderBy`/`Reverse`/`ToList` also buffer. Relevant for memory on large sequences.
</details>

---

### Q9 — Find the N+1
```csharp
var orders = db.Orders.ToList();
foreach (var o in orders)
    Console.WriteLine(o.Customer.Name);    // ?
```
<details><summary>Answer</summary>

**N+1 queries** — 1 for orders + 1 per order to lazily load `Customer`. Fix: `db.Orders.Include(o => o.Customer)` or project: `db.Orders.Select(o => new { o.Id, o.Customer.Name })`. ([17-EFCore.md](17-EFCore.md))
</details>

---

### Q10 — Deferred + ToList interaction
```csharp
var source = new List<int> { 1, 2, 3 };
var deferred = source.Select(x => x * 2);
var materialized = source.Select(x => x * 2).ToList();
source.Clear();
Console.WriteLine(deferred.Count());
Console.WriteLine(materialized.Count);
```
<details><summary>Answer</summary>

**`0`** then **`3`**. `deferred` re-evaluates against the now-empty `source` → 0. `materialized` was snapshotted by `ToList()` before `Clear()` → 3. Materialize when you need a stable snapshot.
</details>
