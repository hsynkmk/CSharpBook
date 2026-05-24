# 16 — ASP.NET Core

## ⚡ 30-second answer

ASP.NET Core processes each request through a **middleware pipeline** — an ordered chain where each component can act before and after the next (`Use(next => ...)`). **Order matters**: exception handling first, then routing → auth → endpoints. You expose endpoints via **Minimal APIs** (lightweight, function-style) or **MVC controllers** (attribute-routed, filters, model binding). It's built on the Generic Host + DI ([15](15-DI-Hosting-Config.md)). Cross-cutting concerns (auth, logging, errors, rate limiting) live in middleware/filters; errors return **ProblemDetails** (RFC 7807).

---

## Core mechanics

**The pipeline** (order is the most-asked thing):
```csharp
var app = builder.Build();
app.UseExceptionHandler();      // 1. catch downstream exceptions → ProblemDetails
app.UseHttpsRedirection();
app.UseStaticFiles();
app.UseRouting();               // 2. match the endpoint
app.UseRateLimiter();
app.UseAuthentication();        // 3. who are you?  (after routing, before authorization)
app.UseAuthorization();         // 4. are you allowed?
app.MapControllers();           // 5. execute the endpoint
app.Run();
```
Each middleware: do work → `await next()` → do work on the way back. Skipping `next()` short-circuits.

**Minimal API vs MVC**:
```csharp
// Minimal API
app.MapGet("/orders/{id:int}", async (int id, IOrderService svc) => {
    var o = await svc.GetAsync(id);
    return o is null ? Results.NotFound() : Results.Ok(o);
});

// MVC controller
[ApiController, Route("orders")]
public class OrdersController(IOrderService svc) : ControllerBase {
    [HttpGet("{id:int}")] public async Task<IActionResult> Get(int id) => ...;
}
```

**Model binding** maps route/query/header/body/form → parameters; **validation** via DataAnnotations/FluentValidation; invalid models auto-return 400 ProblemDetails (`[ApiController]`).

**Filters** (MVC) run around actions: authorization → resource → action → exception → result filters. Minimal APIs use **endpoint filters**.

---

## Comparison tables

| | Minimal APIs | MVC Controllers |
|---|---|---|
| Style | function endpoints | class + attribute routing |
| Overhead | lower | more features/ceremony |
| Filters | endpoint filters | full filter pipeline |
| Best for | microservices, simple APIs | large apps, conventions, views |

| Middleware | Filters |
|---|---|
| Runs for **every** request | Runs only for MVC/endpoints |
| No model binding context | Has action/model context |
| Use for cross-cutting (auth, logging, errors) | Use for action-specific concerns (validation, authz policy) |

---

## 🪤 Traps & gotchas

- **Middleware ordering bugs**: `UseAuthorization` before `UseAuthentication` (no identity yet) or before `UseRouting` (no endpoint) → broken auth. Exception handler must be **first** to catch downstream throws. The classic ASP.NET Core question.
- **Forgetting `await next()`** in custom middleware short-circuits the pipeline silently.
- **Blocking in a handler** (`.Result`/sync I/O) → thread-pool starvation ([12](12-Concurrent-Parallel-AsyncBugs.md)). Keep handlers async.
- **Capturing scoped services in singleton middleware** (middleware is effectively singleton): inject scoped deps via the `InvokeAsync` parameter, not the constructor — else captive dependency ([15](15-DI-Hosting-Config.md)).
- **Returning raw exceptions** to clients leaks internals. Use exception-handling middleware → **ProblemDetails**.
- **No `[ApiController]`** → you lose automatic 400 on invalid models and binding-source inference.
- **CORS misconfiguration** (placement/policy) blocks browser calls — `UseCors` before endpoints, after routing.

---

## ❓ Likely questions

**Q: How does the request pipeline work?**
A: A chain of middleware; each runs code, calls `await next()` to pass control downstream, then runs code on the way back. Like nested wrappers. Order determines behavior.

**Q: Correct order of auth middleware?**
A: `UseRouting` → `UseAuthentication` → `UseAuthorization` → endpoints. Authentication establishes identity; authorization checks it; both need routing to know the endpoint. Exception handling goes first.

**Q: Minimal API vs MVC?**
A: Minimal APIs are lightweight function endpoints (microservices/simple APIs); MVC gives controllers, full filter pipeline, conventions, and views (large apps). Both share routing, DI, model binding.

**Q: Middleware vs filters?**
A: Middleware runs for every request at the pipeline level (cross-cutting: auth, logging, errors). Filters run only around MVC actions with model/action context (validation, action-specific policy).

**Q: What is ProblemDetails?**
A: RFC 7807 standardized error response (type, title, status, detail). ASP.NET Core returns it for errors/validation, giving clients consistent, machine-readable errors.

**Q: How does model binding/validation work?**
A: Binding maps request data (route/query/header/body/form) to action parameters; validation runs after; with `[ApiController]`, invalid models auto-return 400 ProblemDetails.

**Q: How do you inject a scoped service into middleware?**
A: Inject scoped services as parameters of `InvokeAsync(HttpContext ctx, IScopedDep dep)` — not via the constructor (which would be a captive dependency since middleware is constructed once).

---

## 🎓 Senior Extra

- **Endpoint routing** resolves the endpoint *before* most middleware runs (`UseRouting`), so later middleware (auth) can inspect endpoint metadata (e.g., `[Authorize]`) — why ordering is routing → auth → endpoints.
- **Filter pipeline order**: Authorization → Resource → Model binding → Action → Exception → Result filters; short-circuit early to skip work. Endpoint filters are the Minimal API equivalent.
- **Output caching** (.NET 7+) vs **response caching**: output caching is server-side, more controllable; combine with cache keys/vary ([18](18-Caching-Resilience-Http.md)).
- **Rate limiting** (built-in .NET 7+): fixed/sliding window, token bucket, concurrency limiters — place `UseRateLimiter` after routing.
- **Health checks** for K8s: `/health` (readiness — includes dependencies; controls traffic) vs `/alive` (liveness — process up; controls restart) ([21](21-Deployment-Perf-Tooling.md)).
- **Content negotiation** & formatters (JSON default via System.Text.Json; add XML if needed).
- **Kestrel** is the cross-platform web server; often behind a reverse proxy — configure forwarded headers for correct client IP/scheme.
- **OpenAPI**: `Microsoft.AspNetCore.OpenApi` (.NET 9+) generates the spec; pairs with versioning (`Asp.Versioning`) ([22](22-Architecture-Patterns.md)).
- **Minimal API perf**: lower per-request overhead and AOT-friendly (Request Delegate Generator) — relevant for high-throughput microservices ([14](14-Runtime-CLR-JIT.md)).

→ Deeper: [`../DotNetBook/04-AspNetCore/`](../DotNetBook/04-AspNetCore/README.md)
