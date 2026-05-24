# 04 — LINQ

## ⚡ 30-second answer

LINQ is a uniform query syntax over any `IEnumerable<T>` (in-memory, LINQ-to-Objects) or `IQueryable<T>` (translated to another language like SQL, e.g., EF Core). The defining behavior is **deferred execution**: query operators (`Where`, `Select`, `OrderBy`…) build a pipeline that runs **only when enumerated** (`foreach`, `ToList`, `Count`, `First`…). The big traps are **multiple enumeration** (the query re-runs every time you iterate) and **mixing `IQueryable` with client-side operations** (forcing the whole table into memory).

---

## Core mechanics

**Deferred vs immediate**:
```csharp
var q = list.Where(x => x > 2);   // nothing runs yet — just a pipeline
foreach (var x in q) { ... }       // NOW it executes
var arr = q.ToList();              // ToList/ToArray/Count/First/Sum = immediate (materialize)
```

**IEnumerable vs IQueryable**:
- `IEnumerable<T>` → **LINQ-to-Objects**, runs in-memory with delegates (`Func<>`).
- `IQueryable<T>` → operators take **expression trees** (`Expression<Func<>>`), which a provider (EF Core) **translates to SQL**. The query executes in the database.

```csharp
// EF Core: translated to SQL, filtered in the DB:
var pending = db.Orders.Where(o => o.Status == "Pending").ToList();
// Forcing client-side too early — pulls the WHOLE table, then filters in memory:
var bad = db.Orders.AsEnumerable().Where(o => o.Status == "Pending").ToList();
```

**Key operators**: `Where` (filter), `Select`/`SelectMany` (project/flatten), `OrderBy`/`ThenBy`, `GroupBy`, `Join`/`GroupJoin`, `Distinct`, `Take`/`Skip`, `Any`/`All`/`Count`, `First`/`Single`/`FirstOrDefault`, `Aggregate`, `Sum`/`Max`/`Average`, `Zip`, `Chunk`.

**`First` vs `Single` vs `...OrDefault`**:
- `First` — first match (throws if none). `FirstOrDefault` — first or default.
- `Single` — exactly one (throws if zero **or** more than one). `SingleOrDefault` — zero or one.

---

## Comparison tables

| | `IEnumerable<T>` | `IQueryable<T>` |
|---|---|---|
| Runs | in-memory (LINQ-to-Objects) | in the provider (e.g., SQL DB) |
| Lambda is | `Func<>` (delegate) | `Expression<Func<>>` (tree) |
| Filtering happens | client | server (translated) |
| Use for | collections in memory | EF Core / remote sources |

| Method | Deferred? | Notes |
|---|---|---|
| `Where`, `Select`, `OrderBy`, `Take`, `GroupBy` | **deferred** | build the pipeline |
| `ToList`, `ToArray`, `Count`, `Sum`, `First`, `Any` | **immediate** | execute now |

---

## 🪤 Traps & gotchas

- **Multiple enumeration**: iterating a deferred query twice **re-runs it** twice (and re-hits the DB twice). Materialize once with `ToList()` if you'll use it multiple times. (Analyzer: "Possible multiple enumeration".)
- **Captured-variable in deferred query**: the query reads the variable **when enumerated**, not when defined:
  ```csharp
  int threshold = 5;
  var q = nums.Where(n => n > threshold);
  threshold = 100;
  q.ToList();   // uses 100, not 5
  ```
- **N+1 in LINQ-to-Entities**: projecting a navigation lazily in a loop fires one query per row. Use `Include` or project to a DTO ([17-EFCore.md](17-EFCore.md)).
- **Unsupported expression in `IQueryable`**: calling a C# method EF can't translate throws at runtime (or silently evaluates client-side in older versions). Keep server queries translatable; switch to `AsEnumerable()` deliberately for the client-side tail.
- **`Single` vs `First`**: `Single` does an extra check for a second element — don't use it (and its DB cost) unless you truly mean "exactly one".
- **`Count() > 0` vs `Any()`**: `Any()` short-circuits; `Count()` may enumerate everything (or do `COUNT(*)`). Prefer `Any()` for existence.
- **`OrderBy` then `Where`**: order doesn't change results for in-memory, but for SQL the provider optimizes; still, filter before ordering logically.

---

## ❓ Likely questions

**Q: What is deferred execution?**
A: LINQ operators build a query pipeline that doesn't run until you enumerate it (foreach / ToList / aggregates). Lets you compose and the source decide execution.

**Q: IEnumerable vs IQueryable?**
A: IEnumerable runs in memory with delegates; IQueryable uses expression trees a provider translates (e.g., to SQL) and executes remotely. Use IQueryable for EF Core so filtering happens in the DB.

**Q: What's the multiple-enumeration problem?**
A: A deferred query re-executes each time it's iterated. If it hits a DB, that's repeated round-trips. Materialize with ToList/ToArray when reused.

**Q: `Any()` vs `Count() > 0`?**
A: `Any()` short-circuits at the first element (and emits `EXISTS` in SQL); `Count()` may count everything. Use `Any()` for existence checks.

**Q: `First` vs `Single` vs `FirstOrDefault`?**
A: `First` = first (throws if none); `Single` = exactly one (throws if 0 or >1); `OrDefault` variants return default instead of throwing on "none".

**Q: How does EF Core turn LINQ into SQL?**
A: `IQueryable` operators take expression trees; EF's provider walks the tree and generates SQL, executing server-side. C# that can't be translated breaks or runs client-side.

**Q: `Select` vs `SelectMany`?**
A: `Select` projects 1→1 (can yield nested sequences); `SelectMany` flattens 1→many into one sequence.

---

## 🎓 Senior Extra

- **Streaming vs buffering operators**: `Where`/`Select`/`Take` stream (one element at a time, lazy); `OrderBy`/`GroupBy`/`Reverse`/`ToList` **buffer** the whole sequence before yielding — they break laziness and use memory proportional to the input.
- **`yield return`** powers deferred iterators: the compiler builds a state machine `IEnumerator` that resumes where it left off — that's how custom LINQ-style operators stream.
- **`IAsyncEnumerable<T>` + `await foreach`** for async streaming (EF Core `AsAsyncEnumerable`, channels) — backpressure-friendly; flow a `CancellationToken` via `[EnumeratorCancellation]`.
- **Expression trees**: `IQueryable` works because the lambda is *data* the provider inspects. You can build/rewrite expression trees for dynamic queries (`PredicateBuilder`, specification pattern).
- **PLINQ** (`.AsParallel()`) parallelizes in-memory LINQ across cores — only worth it for CPU-bound work over large sequences; it has ordering/overhead costs and isn't for I/O.
- **Performance**: LINQ allocates iterators/closures per operator — fine generally, but in hot inner loops a plain `for`/`foreach` avoids the allocations ([21-Deployment-Perf-Tooling.md](21-Deployment-Perf-Tooling.md)). Measure first.
- **EF translation pitfalls**: client-eval was made to throw by default; `AsSplitQuery` avoids cartesian explosion on multiple `Include`s; projecting to DTOs (`Select`) fetches only needed columns ([17-EFCore.md](17-EFCore.md)).

→ Deeper: [`../CSharpBook/06-LINQ/`](../CSharpBook/06-LINQ/README.md)
