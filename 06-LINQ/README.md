# Chapter 06 — LINQ

> Language-Integrated Query. Filter, project, group, join, aggregate — across in-memory collections, SQL databases, JSON documents, XML, or anything else with a provider.

**Prerequisites**: [Chapter 04 (Generics)](../04-Generics/README.md), [Chapter 05 (Delegates)](../05-DelegatesEvents/README.md).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Overview.md](01-Overview.md) | What LINQ is and isn't, the IEnumerable/IQueryable split, why deferred execution exists. |
| [02-MethodSyntax.md](02-MethodSyntax.md) | The fluent API: `Where`, `Select`, `OrderBy`, `GroupBy`, `Join`, `Aggregate`, `Take`, `Skip`, every common operator with an example. |
| [03-QuerySyntax.md](03-QuerySyntax.md) | `from x in xs where ... select ...`. When query syntax beats method syntax (multi-from, let, join with into). |
| [04-DeferredExecution.md](04-DeferredExecution.md) | The single biggest LINQ trap. What's deferred, what's not, multiple enumeration, materialization. |
| [05-StandardOperators.md](05-StandardOperators.md) | The full catalog: filtering, projection, partitioning, ordering, grouping, joining, set, conversion, element, generation, quantifier, aggregation. |
| [06-CustomOperators.md](06-CustomOperators.md) | Writing your own extension method on `IEnumerable<T>`, when to materialize, lazy iterator patterns. |
| [07-AsyncLinq.md](07-AsyncLinq.md) | `IAsyncEnumerable<T>` operators (in `System.Linq.Async`), `ToListAsync`, when to choose async LINQ. |
| [08-IQueryable.md](08-IQueryable.md) | Expression trees as queries, providers (EF Core, Cosmos, OData), what translates and what falls back to in-memory. |
| [09-PerformanceAndPitfalls.md](09-PerformanceAndPitfalls.md) | Multiple enumeration, lazy chains in hot paths, `Count()` vs `Any()`, `OrderBy` in loops, when LINQ is the wrong tool. |
| [Questions.md](Questions.md) | ~25 questions. |
| [Coding.md](Coding.md) | ~15 problems: predict execution count, fix N+1, write custom Median. |

→ Begin: [01-Overview.md](01-Overview.md)
