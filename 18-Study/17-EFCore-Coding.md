# 17 — EF Core — Coding Questions

> Find the bug / predict the SQL behavior. (Concepts: [17-EFCore.md](17-EFCore.md))

---

### Q1 — Spot the N+1
```csharp
var orders = await db.Orders.ToListAsync();
foreach (var o in orders)
    total += o.Customer.Orders.Count;   // lazy navigation per order
```
<details><summary>Answer</summary>

**N+1 queries** — 1 for orders + 1 per order to load `Customer` (and more for `Customer.Orders`). **Fix:** eager-load (`Include(o => o.Customer)`) or project to a DTO with `Select`. Detect via SQL logging / VS DB profiler.
</details>

---

### Q2 — Tracking a read-only query
```csharp
var products = await db.Products.Where(p => p.Active).ToListAsync();
// products are only displayed, never modified
return View(products);
```
<details><summary>Answer</summary>

Wasteful — EF **tracks** all returned entities (memory + change-detection cost) even though you won't save. **Fix:** add **`.AsNoTracking()`** for read-only queries.
</details>

---

### Q3 — DbContext shared across threads
```csharp
await Task.WhenAll(
    Task.Run(() => db.Orders.CountAsync()),
    Task.Run(() => db.Products.CountAsync())
);
```
<details><summary>Answer</summary>

**Bug** — `DbContext` is **not thread-safe**; using one context from concurrent tasks throws ("A second operation was started on this context...") or corrupts state. **Fix:** use `IDbContextFactory<T>` to create a separate context per parallel operation.
</details>

---

### Q4 — SaveChanges in a loop
```csharp
foreach (var item in items) {
    db.Items.Add(item);
    await db.SaveChangesAsync();   // ?
}
```
<details><summary>Answer</summary>

**N round-trips** — one `SaveChanges` per item. **Fix:** add all, then call `SaveChanges` once — EF **batches** the INSERTs into far fewer round-trips. (For huge volumes, consider bulk-insert libraries.)
</details>

---

### Q5 — Cartesian explosion
```csharp
var blogs = await db.Blogs
    .Include(b => b.Posts)
    .Include(b => b.Comments)
    .ToListAsync();
```
<details><summary>Answer</summary>

Two collection `Include`s in one query → a **cartesian product** (Posts × Comments rows) → bloated result set. **Fix:** **`.AsSplitQuery()`** to run separate queries per collection, trading round-trips for avoiding row multiplication.
</details>

---

### Q6 — Update without loading
```csharp
// Mark all pending orders as cancelled:
var pending = await db.Orders.Where(o => o.Status == "Pending").ToListAsync();
foreach (var o in pending) o.Status = "Cancelled";
await db.SaveChangesAsync();
```
<details><summary>Answer</summary>

Loads every entity into memory just to update one column → slow for large sets. **Fix (EF7+):** set-based update without loading — `await db.Orders.Where(o => o.Status=="Pending").ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, "Cancelled"));`.
</details>

---

### Q7 — Optimistic concurrency
```csharp
public class Product { public int Id; public int Stock; [Timestamp] public byte[] RowVersion; }
// Two users load the same product, both decrement Stock, both SaveChanges.
```
<details><summary>Answer</summary>

The second `SaveChanges` throws **`DbUpdateConcurrencyException`** — the `RowVersion` in its UPDATE's `WHERE` no longer matches (the first save changed it), so 0 rows update. You catch it and resolve (reload + retry / merge). Without `[Timestamp]`, the second write would silently overwrite (lost update).
</details>

---

### Q8 — Client evaluation
```csharp
var users = await db.Users
    .Where(u => MyCustomMethod(u.Name))   // C# method EF can't translate
    .ToListAsync();
```
<details><summary>Answer</summary>

**Throws** at runtime — EF can't translate `MyCustomMethod` to SQL (client evaluation of `Where` is disabled by default to prevent accidental full-table pulls). **Fix:** express the filter in translatable SQL terms, or `.AsEnumerable()` *after* a server-side filter to evaluate the rest client-side deliberately.
</details>

---

### Q9 — InMemory provider hides a bug (senior)
```csharp
// Test uses UseInMemoryDatabase. Production uses SQL Server with a unique constraint.
db.Users.Add(new User { Email = "a@b.com" });
db.Users.Add(new User { Email = "a@b.com" });
await db.SaveChangesAsync();   // passes in tests, fails in prod?
```
<details><summary>Answer</summary>

The **InMemory provider doesn't enforce relational constraints** (no unique index), so the test **passes** but production **throws** on the duplicate. InMemory isn't a real database. **Fix:** test with SQLite in-memory or **Testcontainers** (real DB) for constraint/SQL fidelity.
</details>

---

### Q10 — Repository over EF Core: needed? (senior)
```csharp
public class GenericRepository<T> {
    public IQueryable<T> Query() => _db.Set<T>();
    public async Task<T?> GetById(int id) => await _db.Set<T>().FindAsync(id);
}
```
<details><summary>Answer</summary>

Usually **redundant** — `DbContext` is already a Unit of Work and `DbSet<T>` a repository. This wrapper either **leaks `IQueryable`** (no real abstraction) or **hides EF features** (Include/projection/AsNoTracking → N+1/over-fetch). Use DbContext directly, or a **domain-specific** repository (`IOrderRepository` with intention-revealing methods) only to decouple the domain — not a generic wrapper.
</details>
