# 17 — Entity Framework Core

## ⚡ 30-second answer

EF Core is the ORM that maps C# entities to relational tables and translates **LINQ → SQL** (`IQueryable` — [04](04-LINQ.md)). The **`DbContext` is a Unit of Work** + change tracker: you query, modify tracked entities, then `SaveChanges()` commits all changes in one transaction. It's **scoped** (one per request — [15](15-DI-Hosting-Config.md)), not thread-safe. The #1 performance trap is the **N+1 query** (lazy-loading a navigation per row); fix with `Include` (eager) or projection. Use **`AsNoTracking`** for read-only queries, optimistic concurrency for conflicts, and migrations to evolve the schema.

---

## Core mechanics

**Query → modify → save**:
```csharp
var order = await db.Orders.Include(o => o.Lines)         // eager load
                           .FirstAsync(o => o.Id == id, ct);
order.Status = "Shipped";                                  // tracked change
await db.SaveChangesAsync(ct);                             // one transaction: UPDATE …
```

**Change tracking**: the context tracks loaded entities (Added/Modified/Deleted/Unchanged) and generates the right SQL on `SaveChanges`. **`AsNoTracking()`** skips tracking for read-only queries (faster, less memory).

**N+1 problem**:
```csharp
var orders = await db.Orders.ToListAsync();
foreach (var o in orders)
    Console.WriteLine(o.Customer.Name);   // ❌ lazy-loads Customer → 1 + N queries
// ✅ eager:  db.Orders.Include(o => o.Customer)
// ✅ project: db.Orders.Select(o => new { o.Id, o.Customer.Name })  // only needed columns
```

**Migrations** — evolve the schema in code:
```bash
dotnet ef migrations add AddOrders
dotnet ef database update           # or generate idempotent SQL for prod
```

**Optimistic concurrency** — detect conflicting updates with a `[Timestamp]`/`RowVersion`:
```csharp
public byte[] RowVersion { get; set; }   // throws DbUpdateConcurrencyException on conflict
```

**Transactions**: `SaveChanges` is atomic for one call; wrap multiple with `db.Database.BeginTransactionAsync()`.

---

## Comparison tables

| Loading strategy | What | When |
|---|---|---|
| **Eager** (`Include`) | load navigations with the query | you know you need them |
| **Lazy** (proxies) | load on first access | convenient but **N+1 risk** |
| **Explicit** (`Entry().LoadAsync`) | load on demand manually | selective |
| **Projection** (`Select` DTO) | fetch only needed columns | read APIs — best perf |

| Tracking | Use |
|---|---|
| Tracked (default) | you'll modify + `SaveChanges` |
| `AsNoTracking()` | read-only queries (faster, no change tracking) |

---

## 🪤 Traps & gotchas

- **N+1 queries** — the classic. Lazy-loading inside a loop fires a query per row. Use `Include` or project to a DTO; detect with logging / the VS DB profiler ([21](21-Deployment-Perf-Tooling.md)).
- **Tracking read-only queries** wastes memory/CPU — add `AsNoTracking()`.
- **`DbContext` is not thread-safe / not shared**: never use one context from parallel tasks or register it singleton. Scoped per request; `IDbContextFactory` for parallel/background work.
- **Cartesian explosion**: multiple collection `Include`s in one query multiply rows. Use **`AsSplitQuery()`** to split into separate queries.
- **Client-side evaluation**: a LINQ expression EF can't translate throws (or, historically, silently ran in memory pulling the whole table). Keep server queries translatable.
- **`SaveChanges` per item in a loop** → many round-trips. Batch changes, call `SaveChanges` once (EF batches the SQL).
- **Loading whole entities to update one column** — use `ExecuteUpdate`/`ExecuteDelete` (EF7+) for set-based updates without loading.
- **Migrations applied at startup in prod** can race across instances — prefer generating idempotent SQL / a controlled migration step.
- **InMemory provider in tests** isn't relational (no constraints/real SQL) → hides bugs. Use SQLite in-memory or Testcontainers ([22](22-Architecture-Patterns.md)).

---

## ❓ Likely questions

**Q: What design patterns does `DbContext` implement?**
A: Unit of Work (tracks all changes, commits atomically on `SaveChanges`) and its `DbSet<T>`s act as Repositories. So a separate generic repository over EF Core is often redundant.

**Q: What is the N+1 problem and how do you fix it?**
A: One query for the parents + N queries to lazily load each child. Fix with eager loading (`Include`) or projecting to a DTO (`Select`) so it's one query.

**Q: When use `AsNoTracking`?**
A: For read-only queries — it skips change tracking, reducing memory and CPU. Use tracking only when you'll modify and save.

**Q: Is `DbContext` thread-safe? What lifetime?**
A: No. Register it **scoped** (one per request). For parallel/background work, use `IDbContextFactory`.

**Q: How does optimistic concurrency work?**
A: A version column (`RowVersion`/`[Timestamp]`) is checked in the UPDATE's `WHERE`; if another transaction changed the row, zero rows match and EF throws `DbUpdateConcurrencyException`.

**Q: Eager vs lazy vs explicit loading?**
A: Eager (`Include`) loads related data with the query; lazy loads on first access (convenient, N+1 risk); explicit loads on demand. Projection is best for read-only.

**Q: How do migrations work?**
A: `migrations add` snapshots the model diff into C#/SQL; `database update` applies it. For prod, generate idempotent SQL and apply in a controlled step.

**Q: How do you do a bulk update efficiently?**
A: `ExecuteUpdate`/`ExecuteDelete` (EF7+) run set-based SQL without loading entities — far faster than load-modify-save loops.

---

## 🎓 Senior Extra

- **Compiled queries** (`EF.CompileAsyncQuery`) cache the LINQ→SQL translation for hot queries.
- **`AsSplitQuery`** trades one round-trip for avoiding row multiplication on multiple collection includes; weigh round-trips vs payload.
- **Query tags / logging** (`.TagWith`, interceptors) help find slow/N+1 queries; **`ISaveChangesInterceptor`** powers auditing/soft-delete uniformly.
- **Global query filters** (`HasQueryFilter`) implement soft-delete and **multi-tenancy** (auto `WHERE TenantId = …`) so you can't forget the filter — a key isolation/security mechanism ([22](22-Architecture-Patterns.md)).
- **Owned types / value objects** map a value object (Money, Address) into the owner's table; **JSON columns** map complex sub-objects.
- **Connection resiliency** (`EnableRetryOnFailure`) for transient DB faults; combine with the resilience mindset ([18](18-Caching-Resilience-Http.md)).
- **Tracking pitfalls at scale**: large tracked graphs slow `SaveChanges` — use `AsNoTracking` for reads and short-lived contexts.
- **Repository/UoW debate**: since DbContext already is UoW+repo, add a custom repository only for domain-specific aggregate access or to decouple the domain — not a generic `Repository<T>` wrapper ([22](22-Architecture-Patterns.md)).
- **DbContext pooling** (`AddDbContextPool`) reuses context instances to cut allocation under high request rates.

→ Deeper: [`../DotNetBook/05-EFCore/`](../DotNetBook/05-EFCore/README.md)
