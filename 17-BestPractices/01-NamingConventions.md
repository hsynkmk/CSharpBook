# Naming Conventions

## Why conventions matter

Consistent naming makes code readable without thinking — you instantly know `_count` is a private field, `Count` is a public member, and `count` is a local. The .NET ecosystem follows a well-established set of conventions; matching them makes your code look like it belongs. These are the conventions enforced by the BCL, the C# style analyzers, and `.editorconfig` (see [Chapter 15 §06](../15-BuildTooling/06-EditorConfig.md)).

---

## The casing rules

| Identifier | Casing | Example |
|---|---|---|
| Class, struct, record, enum | PascalCase | `OrderService`, `HttpClient` |
| Interface | PascalCase, `I` prefix | `IDisposable`, `IOrderRepository` |
| Method | PascalCase | `CalculateTotal`, `SendAsync` |
| Public property/field | PascalCase | `FirstName`, `MaxValue` |
| Public constant | PascalCase | `DefaultTimeout` |
| Local variable | camelCase | `orderCount`, `index` |
| Method parameter | camelCase | `customerId`, `cancellationToken` |
| Private field | `_camelCase` | `_repository`, `_count` |
| Private static field | `_camelCase` (or `s_`) | `_cache` (or `s_cache`) |
| Enum member | PascalCase | `Status.Active` |
| Type parameter | PascalCase, `T` prefix | `T`, `TKey`, `TResult` |
| Namespace | PascalCase, dotted | `MyCompany.Orders.Domain` |
| Async method | PascalCase + `Async` suffix | `LoadAsync`, `SaveChangesAsync` |

Note: C# does **not** use `SCREAMING_SNAKE_CASE` for constants (unlike Java/C). Constants are PascalCase: `public const int MaxRetries = 3;`.

---

## Private field conventions

The two common styles for private fields:

```csharp
public class Service {
    private readonly ILogger _logger;        // _camelCase — most common
    private static readonly object s_lock;   // s_ prefix for static (used in dotnet/runtime)
    private int _counter;
}
```

`_camelCase` (underscore + camelCase) is the dominant convention and what the .NET runtime style guide recommends for instance fields. Some codebases (notably the runtime itself) use `s_` for static and `t_` for thread-static fields. Pick one and apply it consistently via `.editorconfig`.

Avoid `this.field` qualification when the `_` prefix already disambiguates:

```csharp
// ✗ — redundant with _ prefix convention
public Service(ILogger logger) { this._logger = logger; }
// ✓
public Service(ILogger logger) { _logger = logger; }
```

---

## Interfaces and abstract types

```csharp
public interface IRepository { }        // I prefix — always
public abstract class RepositoryBase { }  // "Base" suffix is conventional, not required
```

The `I` prefix for interfaces is one of the few "Hungarian" survivors in .NET — it's universal and expected. Don't prefix classes with `C` or fields with `m_` (legacy Win32/MFC styles).

---

## Async methods — the `Async` suffix

```csharp
public async Task<Order> GetOrderAsync(int id, CancellationToken ct = default);
public async Task SaveAsync();
```

Methods returning `Task`/`Task<T>`/`ValueTask` get an `Async` suffix by convention. It signals "await me" at the call site and disambiguates sync/async overloads. (Exception: ASP.NET Core controller actions and some event handlers, where the framework convention differs.) See [05-AsyncIdioms.md](05-AsyncIdioms.md).

---

## Booleans — read as a question

```csharp
public bool IsActive { get; }
public bool HasItems => _items.Count > 0;
public bool CanExecute(object param);
public bool ShouldRetry(int attempt);
```

Prefix booleans with `Is`, `Has`, `Can`, `Should`, `Allows` so they read as yes/no questions. Avoid negatives (`IsNotReady` → prefer `IsReady` and negate at use).

---

## Generic type parameters

```csharp
public class Cache<TKey, TValue> { }
public T Map<TSource, TResult>(TSource input) { }
```

- Single type parameter → `T`.
- Multiple → descriptive names with `T` prefix: `TKey`, `TValue`, `TResult`, `TInput`.

Don't use bare `K`, `V`, `R` — `TKey`/`TValue`/`TResult` are the conventions.

---

## Meaningful names over abbreviations

```csharp
// ✗ — cryptic
int n; var dt; var lst; CalcAmt();

// ✓ — clear
int count; var startDate; var orders; CalculateAmount();
```

Favor clarity over brevity. Exceptions: well-known short names (`i`, `j` for loop indices; `id`; `ok`; standard math like `x`, `y`). Don't abbreviate domain terms (`amt`, `qty`, `cust`) — spell them out (`amount`, `quantity`, `customer`).

---

## Consistency in collections and pairs

```csharp
public IReadOnlyList<Order> Orders { get; }    // plural for collections
public int OrderCount { get; }                  // singular + "Count"

// Symmetric pairs
Start() / Stop()        Open() / Close()
Add() / Remove()        Begin() / End()
Create() / Delete()     Enable() / Disable()
```

Use plural names for collections. Use established symmetric verb pairs — don't mix `Add`/`Delete` or `Open`/`Stop`.

---

## Avoid abbreviations the framework doesn't use

Match BCL terminology: `Initialize` not `Init` (in public APIs), `Configuration` not `Config` (mostly), `Information` not `Info`. But common accepted ones are fine: `Id`, `Ok`, `Async`, `Db` (debatable), `Json`/`Xml`/`Html` (acronyms ≤2 letters uppercased: `IOStream`, `IPAddress`; ≥3 letters PascalCased: `HtmlParser`, `XmlReader`).

Acronym rule:
- Two-letter acronym → both uppercase: `IOException`, `UIElement`, `IPAddress`.
- Three+ letters → PascalCase: `HtmlDocument`, `XmlReader`, `HttpClient` (note: not `HTTPClient`).

---

## Common bugs / gotchas

### Inconsistent field prefixes

Mixing `_field`, `m_field`, and `field` in one codebase. Pick `_camelCase`, enforce via `.editorconfig` naming rules.

### `SCREAMING_CASE` constants (Java habit)

```csharp
public const int MAX_SIZE = 100;   // ✗ — not C# style
public const int MaxSize = 100;     // ✓
```

### Async method without `Async` suffix

`public Task<int> GetCount()` reads like a sync method. Name it `GetCountAsync`.

### Hungarian notation creep

`strName`, `intCount`, `arrItems` — the type is already in the declaration. Drop the prefixes (the `I` for interfaces is the one survivor).

---

## Summary

- PascalCase for types/members/constants; camelCase for locals/parameters; `_camelCase` for private fields.
- `I` prefix for interfaces; `T`-prefixed PascalCase for generic parameters (`TKey`, `TResult`).
- `Async` suffix for Task-returning methods; boolean members read as questions (`IsActive`, `HasItems`).
- C# constants are PascalCase, **not** `SCREAMING_SNAKE_CASE`.
- Acronyms: 2 letters all-caps (`IOStream`), 3+ PascalCase (`HtmlParser`); `HttpClient` not `HTTPClient`.
- Prefer clear names over abbreviations; enforce consistency with `.editorconfig`.

→ Next: [02-CodingGuidelines.md](02-CodingGuidelines.md)
