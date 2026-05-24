# 20 — Observability, Messaging & Background — Coding Questions

> Find the bug / predict. (Concepts: [20-Observability-Messaging-Background.md](20-Observability-Messaging-Background.md))

---

### Q1 — Unstructured logging
```csharp
logger.LogInformation($"Order {orderId} placed for {amount}");   // ?
```
<details><summary>Answer</summary>

**Loses structure** — string interpolation produces one flat message; you can't query/filter by `orderId`. **Fix:** message template with named placeholders — `logger.LogInformation("Order {OrderId} placed for {Amount}", orderId, amount);` → `OrderId`/`Amount` become queryable fields.
</details>

---

### Q2 — Non-idempotent handler
```csharp
void Handle(PaymentMessage msg) {
    _accounts.Charge(msg.AccountId, msg.Amount);   // ?
}
```
<details><summary>Answer</summary>

**Double-charge risk** — broker delivery is **at-least-once**, so this message may be redelivered (after a transient failure/timeout). **Fix:** make the handler **idempotent** — dedupe by `msg.Id` (record processed ids) so a redelivery is a no-op.
</details>

---

### Q3 — Dual-write problem
```csharp
await db.SaveChangesAsync();          // 1) save order
await bus.PublishAsync(orderPlaced);  // 2) publish event  — what if this fails?
```
<details><summary>Answer</summary>

If the publish (step 2) fails after the DB commit (step 1), the order exists but **no event is sent** → DB and downstream diverge (dual-write problem). **Fix:** **outbox pattern** — write the event to an outbox table in the **same transaction** as the order; a separate process publishes it (publish iff committed).
</details>

---

### Q4 — BackgroundService without a scope
```csharp
public class Worker(AppDbContext db) : BackgroundService {   // ?
    protected override async Task ExecuteAsync(CancellationToken stop) {
        while (!stop.IsCancellationRequested) { await ProcessBatch(db); await Task.Delay(1000, stop); }
    }
}
```
<details><summary>Answer</summary>

`BackgroundService` is a **singleton**, so injecting **scoped** `AppDbContext` in the constructor is a **captive dependency** (one context reused forever → corruption). **Fix:** inject `IServiceScopeFactory`; create a **scope per iteration/item** and resolve `AppDbContext` inside it. ([15](15-DI-Hosting-Config-Coding.md))
</details>

---

### Q5 — Ignoring the stopping token
```csharp
protected override async Task ExecuteAsync(CancellationToken stop) {
    while (true) { await DoWork(); await Task.Delay(5000); }   // ?
}
```
<details><summary>Answer</summary>

Ignores `stop` → on shutdown, work is killed mid-flight and the host waits/forces termination. **Fix:** `while (!stop.IsCancellationRequested)` and pass `stop` to awaited calls (`Task.Delay(5000, stop)`, `DoWork(stop)`) for graceful drain.
</details>

---

### Q6 — Logging in a hot loop
```csharp
foreach (var item in millionItems)
    logger.LogDebug($"Processing {item.ToExpensiveString()}");   // Debug filtered out in prod
```
<details><summary>Answer</summary>

Even when `Debug` is filtered out, `$"...{ToExpensiveString()}"` **still evaluates** the expensive interpolation every iteration. **Fix:** use the template form (`LogDebug("Processing {Item}", item)` — args evaluated only if enabled) or `LoggerMessage` source-gen, or check `IsEnabled(LogLevel.Debug)`.
</details>

---

### Q7 — Three pillars mapping
```csharp
// You need: (a) alert when error rate > 5%, (b) debug one failed request end-to-end, (c) audit what happened.
```
<details><summary>Answer</summary>

(a) **Metrics** (error-rate counter → alerting). (b) **Traces** (the request's span tree across services, correlated by trace id). (c) **Logs** (structured event detail). The three pillars; OpenTelemetry exports all three.
</details>

---

### Q8 — Liveness probe coupling (senior)
```csharp
// Health check used as the K8s LIVENESS probe pings the database.
```
<details><summary>Answer</summary>

**Restart loops** — when the DB blips, liveness fails → K8s restarts pods (which can't fix the DB) → cascading restarts. **Liveness** = cheap "process up" (`/alive`); put dependency checks in **readiness** (`/health`) which stops routing traffic without restarting. ([16](16-AspNetCore-Coding.md), [21](21-Deployment-Perf-Tooling.md))
</details>
