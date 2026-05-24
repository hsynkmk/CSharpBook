# 16 — ASP.NET Core — Coding Questions

> Find the bug / predict. (Concepts: [16-AspNetCore.md](16-AspNetCore.md))

---

### Q1 — Middleware ordering bug
```csharp
app.UseAuthorization();
app.UseAuthentication();
app.MapControllers();
```
<details><summary>Answer</summary>

**Wrong order** — `UseAuthorization` runs before `UseAuthentication`, so there's **no identity established** when authorization checks it → 401/403 even for valid users. Correct: **`UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints**, with exception handling first.
</details>

---

### Q2 — Custom middleware forgets next()
```csharp
app.Use(async (ctx, next) => {
    Log(ctx.Request.Path);
    // forgot to call next
});
app.MapGet("/", () => "hello");
```
<details><summary>Answer</summary>

The request **short-circuits** — `next()` is never called, so the endpoint never runs (response is empty/200 with no body). Always `await next(ctx);` unless you intentionally short-circuit.
</details>

---

### Q3 — Blocking in an action
```csharp
[HttpGet]
public IActionResult Get() {
    var data = _service.GetDataAsync().Result;   // ?
    return Ok(data);
}
```
<details><summary>Answer</summary>

`.Result` blocks a thread-pool thread → **thread-pool starvation** under load (and historically deadlocks). **Fix:** `public async Task<IActionResult> Get() => Ok(await _service.GetDataAsync());`. Keep the whole request pipeline async.
</details>

---

### Q4 — Scoped service in middleware constructor
```csharp
public class MyMiddleware {
    public MyMiddleware(RequestDelegate next, AppDbContext db) { }   // ?
    public Task InvokeAsync(HttpContext ctx) => ...;
}
```
<details><summary>Answer</summary>

**Captive dependency** — middleware is constructed **once** (singleton-like), so injecting **scoped** `AppDbContext` in the constructor captures it forever. **Fix:** inject scoped services as **`InvokeAsync` parameters**: `public Task InvokeAsync(HttpContext ctx, AppDbContext db)` — resolved per request.
</details>

---

### Q5 — Model validation auto-400
```csharp
[ApiController]
public class UsersController : ControllerBase {
    [HttpPost] public IActionResult Create(CreateUser dto) {
        // no manual ModelState check
        return Ok();
    }
}
public record CreateUser([property: Required] string Name);
```
<details><summary>Answer</summary>

With **`[ApiController]`**, an invalid model (missing `Name`) **auto-returns 400 ProblemDetails** before the action runs — no manual `if (!ModelState.IsValid)` needed. Without `[ApiController]`, you'd have to check `ModelState` yourself.
</details>

---

### Q6 — Minimal API parameter binding
```csharp
app.MapGet("/search", (string q, int page, IService svc) => svc.Search(q, page));
// GET /search?q=hello&page=2
```
<details><summary>Answer</summary>

`q` ← query string, `page` ← query string (parsed to int), `svc` ← **DI** (services are inferred). Returns the result serialized as JSON. Minimal APIs infer binding sources: route/query for simple types, body for complex, DI for registered services.
</details>

---

### Q7 — Returning raw exception
```csharp
app.MapGet("/x", () => { throw new InvalidOperationException("secret internal detail"); });
// no exception handler middleware
```
<details><summary>Answer</summary>

The exception detail may **leak to the client** (and in Production you get an opaque 500). **Fix:** add `app.UseExceptionHandler()` **first** in the pipeline → consistent **ProblemDetails** responses, internal details logged not exposed.
</details>

---

### Q8 — Liveness vs readiness (senior)
```csharp
app.MapHealthChecks("/health");   // includes a DB check
// configured as the Kubernetes LIVENESS probe
```
<details><summary>Answer</summary>

**Wrong probe mapping** — a liveness probe that checks the DB causes **restart loops** when the DB blips (restarting the pod won't fix the DB). Liveness (`/alive`) = cheap "process up"; **readiness** (`/health`) = dependency checks (stop routing traffic, don't restart). ([21-Deployment-Perf-Tooling.md](21-Deployment-Perf-Tooling.md))
</details>
