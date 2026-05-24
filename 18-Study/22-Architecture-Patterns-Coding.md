# 22 — Architecture & Patterns — Coding Questions

> Find the smell / name the principle. (Concepts: [22-Architecture-Patterns.md](22-Architecture-Patterns.md))

---

### Q1 — Which SOLID principle is violated?
```csharp
class ReportService {
    public void Generate() { /* build report */ }
    public void SaveToDisk() { /* file IO */ }
    public void SendEmail() { /* SMTP */ }
    public void FormatHtml() { /* rendering */ }
}
```
<details><summary>Answer</summary>

**Single Responsibility** — the class has many reasons to change (formatting, IO, email, generation). **Fix:** split into focused collaborators (`IReportFormatter`, `IReportStore`, `IEmailSender`) and compose them.
</details>

---

### Q2 — Which principle?
```csharp
double Area(object shape) {
    if (shape is Circle c) return Math.PI * c.R * c.R;
    if (shape is Square s) return s.Side * s.Side;
    // every new shape => edit this method
    throw new NotSupportedException();
}
```
<details><summary>Answer</summary>

**Open/Closed** violation — adding a shape requires **modifying** this method. **Fix:** make shapes implement `IShape { double Area(); }` (polymorphism) so you **extend** by adding a type, not editing existing code.
</details>

---

### Q3 — Which principle?
```csharp
interface IRepository {
    void Add(); void Remove(); void Update();
    void GenerateReport(); void ExportCsv(); void SendNotification();
}
class ReadOnlyCache : IRepository { /* forced to implement Add/Remove/... */ }
```
<details><summary>Answer</summary>

**Interface Segregation** — a fat interface forces implementers to support methods they don't need (`ReadOnlyCache` shouldn't have `Add`/`Remove`). **Fix:** split into small, focused interfaces (`IReadRepository`, `IWriteRepository`, `IReportable`).
</details>

---

### Q4 — DIP: what's wrong?
```csharp
class OrderService {
    private readonly SqlServerOrderRepository _repo = new();   // concrete dependency
    public OrderService() { }
}
```
<details><summary>Answer</summary>

**Dependency Inversion** violation — depends on a **concrete** class, not an abstraction; hard to test/swap. **Fix:** depend on `IOrderRepository` and **inject** it (`public OrderService(IOrderRepository repo)`). DI realizes DIP.
</details>

---

### Q5 — Identify the anti-pattern
```csharp
public class OrderController {
    public IActionResult Place(OrderDto dto) {
        // 200 lines: validation, mapping, DB access, payment, email, logging, response shaping...
    }
}
```
<details><summary>Answer</summary>

**Fat controller / god method** — business logic crammed into the controller, untestable and a change magnet. **Fix:** thin controller that delegates to an application service / command handler (CQRS); controller only handles HTTP concerns.
</details>

---

### Q6 — Singleton pattern vs DI
```csharp
public sealed class Logger {
    public static readonly Logger Instance = new();
    private Logger() { }
}
// used as Logger.Instance.Log(...) everywhere
```
<details><summary>Answer</summary>

The classic **Singleton pattern** creates hidden global state, hard to mock/test, and couples callers to the concrete type. **Prefer a DI singleton lifetime**: register `ILogger` and inject it — same single-instance benefit, but testable and decoupled.
</details>

---

### Q7 — Anemic vs rich domain (senior)
```csharp
class Order { public List<Item> Items {get;set;} public OrderStatus Status {get;set;} }
// elsewhere: order.Status = OrderStatus.Shipped;  // anyone can set any state
```
<details><summary>Answer</summary>

**Anemic domain model** — a data bag with logic scattered in services; invariants unprotected (anyone can set an invalid `Status`). For complex domains, prefer a **rich model**: private setters + methods that enforce rules (`order.Ship()` checks it's `Paid` first). (Anemic DTOs are fine for simple CRUD.)
</details>

---

### Q8 — When is CQRS overkill? (senior)
```csharp
// A simple admin CRUD screen: list, create, edit, delete users. Add MediatR + CQRS commands/queries/handlers?
```
<details><summary>Answer</summary>

**Overkill** — read and write models don't diverge; CQRS adds command/query/handler ceremony for no benefit. Use a controller + EF Core directly. Apply CQRS when models genuinely differ, you need uniform cross-cutting behaviors (pipeline), or reads need independent scaling. Match the pattern to the problem (YAGNI).
</details>
