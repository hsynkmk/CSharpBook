# 15 — DI, Hosting & Config — Coding Questions

> Find the bug / predict. (Concepts: [15-DI-Hosting-Config.md](15-DI-Hosting-Config.md))

---

### Q1 — The captive dependency
```csharp
builder.Services.AddSingleton<CacheService>();
builder.Services.AddScoped<AppDbContext>();
public class CacheService(AppDbContext db) { }   // ?
```
<details><summary>Answer</summary>

**Captive dependency** — a **Singleton** (`CacheService`) captures a **Scoped** `DbContext`, holding one DbContext for the app's lifetime. DbContext isn't thread-safe and accumulates tracked entities → corruption/leaks. (With scope validation on, the container **throws at startup**.) **Fix:** inject `IServiceScopeFactory`, create a scope per use, resolve DbContext inside.
</details>

---

### Q2 — Which instance count?
```csharp
builder.Services.AddTransient<IThing, Thing>();
// In one HTTP request, IThing is injected into 3 different services. How many Thing instances?
```
<details><summary>Answer</summary>

**3** — `Transient` creates a new instance **per resolution/injection point**. (Scoped → 1 per request; Singleton → 1 for the app.)
</details>

---

### Q3 — IOptionsSnapshot in a singleton
```csharp
builder.Services.AddSingleton<BackgroundProcessor>();
public class BackgroundProcessor(IOptionsSnapshot<MyOptions> opt) { }   // ?
```
<details><summary>Answer</summary>

**Captive dependency** — `IOptionsSnapshot<T>` is **scoped**; injecting it into a singleton captures one scope's value forever (and won't reflect reloads as intended). **Fix:** use **`IOptionsMonitor<T>`** in singletons/background services (live `CurrentValue`).
</details>

---

### Q4 — Config precedence
```jsonc
// appsettings.json:            { "Timeout": 30 }
// appsettings.Production.json: { "Timeout": 60 }
// env var:                     Timeout=90
```
```csharp
// In Production with that env var set:
var t = config.GetValue<int>("Timeout");
```
<details><summary>Answer</summary>

**`90`** — later providers win: appsettings.json → appsettings.Production.json → env vars → command line. The env var overrides both files.
</details>

---

### Q5 — Nested config via env var
```csharp
// appsettings: { "Smtp": { "Port": 25 } }
// You want to override Port to 587 via an environment variable. What name?
```
<details><summary>Answer</summary>

**`Smtp__Port`** (double underscore for hierarchy) = `587`. Using `Smtp:Port` works on some shells but `__` is portable across all platforms. The provider maps `__` → `:`.
</details>

---

### Q6 — Service locator smell
```csharp
public class OrderService {
    private readonly IServiceProvider _sp;
    public OrderService(IServiceProvider sp) => _sp = sp;
    public void Run() { var repo = _sp.GetRequiredService<IRepo>(); /* ... */ }
}
```
<details><summary>Answer</summary>

**Service locator anti-pattern** — the real dependency (`IRepo`) is hidden (not in the constructor signature), hurting testability and clarity. **Fix:** `public OrderService(IRepo repo)` — constructor injection makes dependencies explicit.
</details>

---

### Q7 — Validate options at startup
```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations();   // missing something?
```
<details><summary>Answer</summary>

Missing **`.ValidateOnStart()`**. Without it, validation runs **lazily** on first `.Value` access — a bad config surfaces in production at runtime, not at boot. `.ValidateOnStart()` makes misconfiguration a **startup failure** (fail fast).
</details>

---

### Q8 — Resolving scoped at root (senior)
```csharp
var app = builder.Build();
var db = app.Services.GetRequiredService<AppDbContext>();   // ?
```
<details><summary>Answer</summary>

**Throws** — you can't resolve a **scoped** service from the **root** provider (no active scope). **Fix:** `using var scope = app.Services.CreateScope(); var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();`. Background work must create a scope per unit of work.
</details>
