# 15 — Dependency Injection, Hosting & Configuration

## ⚡ 30-second answer

**DI** inverts control: instead of a class `new`-ing its dependencies, they're **injected** (usually via constructor) from a **container** configured at startup. .NET has a **built-in container** with three **lifetimes** — **Singleton** (one for the app), **Scoped** (one per request/scope), **Transient** (new every time). The #1 trap is the **captive dependency**: injecting a Scoped (or Transient) service into a Singleton captures it for the app's lifetime (e.g., a Singleton holding a Scoped `DbContext` → corruption). The **Generic Host** wires DI + configuration + logging + lifetime; **configuration** layers providers (appsettings → env vars → etc.), consumed type-safely via **`IOptions<T>`**.

---

## Core mechanics

**Registration & injection**:
```csharp
builder.Services.AddSingleton<IClock, SystemClock>();
builder.Services.AddScoped<IOrderService, OrderService>();
builder.Services.AddTransient<IEmailSender, EmailSender>();

public class OrderService(IRepo repo, IClock clock) { … }   // constructor injection
```

**Lifetimes**:
- **Singleton** — one instance, shared for the app lifetime. Must be **thread-safe**.
- **Scoped** — one per **scope** = per HTTP request in ASP.NET Core. `DbContext` is scoped.
- **Transient** — a fresh instance each resolution.

**Generic Host** — composition root + lifecycle:
```csharp
var builder = Host.CreateApplicationBuilder(args);   // or WebApplication.CreateBuilder
builder.Services.AddHostedService<Worker>();         // background service
var app = builder.Build();
await app.RunAsync();                                 // starts hosted services, handles SIGTERM
```

**Configuration** — layered providers, **later wins**:
```
appsettings.json → appsettings.{Env}.json → user secrets (dev) → env vars → command line
```
```csharp
builder.Services.AddOptions<SmtpOptions>()
    .Bind(builder.Configuration.GetSection("Smtp"))
    .ValidateDataAnnotations().ValidateOnStart();     // fail fast at startup
```

**IOptions family**:
- `IOptions<T>` — singleton, read once (fixed config).
- `IOptionsSnapshot<T>` — scoped, re-read per request (reflects reload).
- `IOptionsMonitor<T>` — singleton, live `CurrentValue` + change callback (for singletons/background services).

---

## Comparison tables

| Lifetime | Instances | Thread-safe needed? | Example |
|---|---|---|---|
| Singleton | one (app) | **yes** | caches, `HttpClient` factory, config |
| Scoped | one per request | no (one request) | `DbContext`, per-request services |
| Transient | one per resolution | depends | lightweight stateless helpers |

| Accessor | Lifetime | Reloads? | Inject into |
|---|---|---|---|
| `IOptions<T>` | singleton | no | fixed config |
| `IOptionsSnapshot<T>` | scoped | per request | request-scoped services |
| `IOptionsMonitor<T>` | singleton | live | **singletons / background services** |

---

## 🪤 Traps & gotchas

- **Captive dependency**: a **Singleton** that depends on a **Scoped**/Transient captures it permanently — e.g., a singleton holding a scoped `DbContext` (not thread-safe, accumulates state) → corruption/leaks. Fix: inject `IServiceScopeFactory` and create a scope per use, or match lifetimes.
- **Injecting `IOptionsSnapshot<T>` into a singleton** — it's scoped → same captive-dependency bug. Use `IOptionsMonitor<T>` in singletons.
- **Service locator anti-pattern**: pulling from `IServiceProvider` inside a class instead of constructor injection — hides dependencies, hurts testability ([22](22-Architecture-Patterns.md)).
- **`DbContext` as singleton** — not thread-safe and tracks entities forever. Register **scoped** ([17](17-EFCore.md)); for singletons/parallel work use `IDbContextFactory`.
- **Resolving scoped services at app root** (outside a request) throws — create a scope explicitly.
- **Secrets in `appsettings.json`** — leak via source control. Use **user secrets** (dev) / **Key Vault** (prod) ([19](19-Security-Auth.md)).
- **Config key/property mismatch** binds silently to the default — validate at startup (`ValidateOnStart`).
- **Env var nesting**: use `__` (double underscore) for hierarchy (`Smtp__Port`), not `:`, on all platforms.

---

## ❓ Likely questions

**Q: What are the three DI lifetimes?**
A: Singleton (one per app), Scoped (one per request/scope), Transient (new each resolution). Singletons must be thread-safe.

**Q: What's a captive dependency?**
A: When a longer-lived service captures a shorter-lived one — e.g., a Singleton injecting a Scoped service holds it for the whole app lifetime, breaking per-request semantics (and corrupting non-thread-safe services like DbContext). Fix with `IServiceScopeFactory` or matching lifetimes.

**Q: Why is DI useful?**
A: Decouples classes from concrete dependencies, enabling testability (inject fakes), flexibility (swap implementations), and a single composition root. Constructor injection makes dependencies explicit.

**Q: Why is `DbContext` scoped, not singleton?**
A: It's not thread-safe and accumulates tracked entities. Scoped gives one per request. For parallel/background work, use `IDbContextFactory`.

**Q: `IOptions` vs `IOptionsSnapshot` vs `IOptionsMonitor`?**
A: `IOptions` = singleton, read once. `IOptionsSnapshot` = scoped, re-read per request (reflects reload). `IOptionsMonitor` = singleton with live `CurrentValue`/change notifications — use it in singletons/background services.

**Q: How does configuration precedence work?**
A: Providers layer in order; later providers override earlier for the same key. Typical: appsettings.json → appsettings.{Env}.json → user secrets → env vars → command line (command line wins).

**Q: What does the Generic Host do?**
A: It's the composition root — sets up DI, configuration, logging, and the app lifecycle, runs `IHostedService`s, and handles graceful shutdown (SIGTERM).

**Q: Service locator — why avoid it?**
A: Resolving services from the container inside a class hides true dependencies (not in the constructor), making code harder to test and reason about. Prefer constructor injection.

---

## 🎓 Senior Extra

- **Validation of lifetimes**: enable `ValidateScopes`/`ValidateOnBuild` (on by default in Development) to catch captive dependencies at startup.
- **Keyed services** (.NET 8): register multiple implementations under keys and resolve with `[FromKeyedServices("key")]` — avoids hand-rolled factories.
- **Open generics**: `services.AddScoped(typeof(IRepository<>), typeof(EfRepository<>))` registers a generic implementation for all `T`.
- **Decorator pattern**: wrap a registered service (caching/logging decorator) — via Scrutor (`Decorate<T>`) or a manual factory.
- **`IHostedService` vs `BackgroundService`**: `BackgroundService` is the base class with `ExecuteAsync`; both run in the host lifecycle and receive the **stopping token** for graceful shutdown ([20](20-Observability-Messaging-Background.md)).
- **Reload tokens / `IChangeToken`**: file providers watch for changes; `IOptionsMonitor` rebuilds on reload — but `IOptions<T>` won't see changes (read once).
- **Scope per unit of work**: in background processing, create a DI **scope per item/message** so scoped services (DbContext) are fresh and disposed per item ([20](20-Observability-Messaging-Background.md)).

→ Deeper: [`../DotNetBook/03-HostingAndDI/`](../DotNetBook/03-HostingAndDI/README.md), [`../DotNetBook/13-Configuration/`](../DotNetBook/13-Configuration/README.md)
