# Error-Handling Strategy

## What it covers

Defensive programming ([08-DefensiveProgramming.md](08-DefensiveProgramming.md)) covers *guarding* against bad input. This file covers the broader **strategy**: how to represent, propagate, and respond to failures across an application — exceptions vs result types vs validation, where to catch, what to log, and how to keep error handling coherent rather than ad-hoc.

A consistent error-handling strategy is the difference between a system that fails clearly and one that fails mysteriously.

---

## The core decision: exceptions vs results

The first question for any failure: **is it exceptional, or expected?**

| Use exceptions | Use a result/Try-pattern |
|---|---|
| Caller bug / contract violation | Validation failure (user input) |
| Broken invariant ("impossible" state) | "Not found" lookups |
| Unrecoverable conditions | Parse failures |
| Rare | Frequent / part of normal flow |
| Programming errors | Business-rule rejections |

```csharp
// Exception — the caller violated the contract (a bug)
public void Withdraw(decimal amount) {
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);   // caller passed garbage
    if (amount > _balance) throw new InvalidOperationException("Insufficient funds");
}

// Result — failure is expected and routine
public Result<User> Authenticate(string user, string password) {
    if (!_users.TryGetValue(user, out var u)) return Result<User>.Fail("Unknown user");
    if (!u.VerifyPassword(password))          return Result<User>.Fail("Bad password");
    return Result<User>.Ok(u);
}
```

**Exceptions are for exceptional conditions, not control flow.** They're expensive (stack capture, ~μs) and noisy (try/catch everywhere) when failure is routine. A login endpoint that throws on every wrong password is using exceptions wrong.

---

## Result types

For expected failures, a result type makes the failure part of the signature, forcing callers to handle it:

```csharp
public readonly record struct Result<T>(bool Success, T? Value, string? Error) {
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}

// Caller must handle both paths — no silent failure, no exception for routine cases
var result = service.Authenticate(user, pwd);
return result.Success ? Ok(result.Value) : Unauthorized(result.Error);
```

Richer libraries (`OneOf`, `ErrorOr`, `FluentResults`, `LanguageExt`) provide discriminated-union-style results with multiple error types, mapping, and railway-oriented composition:

```csharp
// ErrorOr-style
ErrorOr<User> result = await Authenticate(user, pwd);
return result.Match(
    user => Ok(user),
    errors => Problem(errors));
```

Result types shine for **domain/business failures** that are part of the expected flow. Don't use them for genuine bugs (a null where one is impossible) — let those throw and surface loudly.

---

## Exception design

When you do throw, throw well (see also [04-ApiDesign.md](04-ApiDesign.md)):

```csharp
// ✓ — specific type, parameter name, message explaining what AND why
throw new ArgumentNullException(nameof(order));
throw new ArgumentOutOfRangeException(nameof(qty), qty, "Quantity must be positive.");
throw new InvalidOperationException("Cannot ship an unpaid order.");

// ✗ — generic, unhelpful
throw new Exception("error");
```

- Throw the **most specific** standard exception.
- Define **custom exceptions** only when callers need to catch them distinctly; derive from `Exception`, add the standard constructors, include relevant data as properties.
- Never throw `Exception`, `SystemException`, `ApplicationException` directly.
- Document thrown exceptions in XML docs.

```csharp
// Custom exception with context callers can act on
public class InsufficientFundsException(decimal requested, decimal available)
    : Exception($"Requested {requested}, only {available} available.") {
    public decimal Requested { get; } = requested;
    public decimal Available { get; } = available;
}
```

---

## Catch specifically; preserve the stack

```csharp
// ✗ — broad catch hides bugs (and can swallow OutOfMemoryException etc.)
try { Process(); } catch (Exception) { }

// ✗ — 'throw ex' resets the stack trace to here
try { Process(); } catch (Exception ex) { Log(ex); throw ex; }

// ✓ — catch what you can handle; rethrow preserving the original stack
try {
    Process();
} catch (IOException ex) {
    _logger.LogError(ex, "Processing {File} failed", file);
    throw;   // preserves origin; or wrap: throw new ProcessingException("...", ex);
}
```

Rules:
- Catch the **most specific** exception you can actually handle.
- `throw;` (not `throw ex;`) preserves the original stack trace.
- To add context, **wrap** with an inner exception: `throw new XException("context", ex)`.
- **Never swallow** silently — at minimum log; usually rethrow or translate.

### Exception filters

```csharp
try { Call(); }
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.TooManyRequests) {
    await BackoffAndRetry();   // only catches the 429 case; other statuses propagate
}
```

`when` filters let you catch conditionally without catching-and-rethrowing (which would disturb the stack). Prefer filters over `catch { if (...) throw; }`.

---

## Where to handle errors — the boundary principle

Don't try/catch everywhere. Handle errors at **strategic boundaries**:

- **Application entry points / global handlers** — catch-all that logs and returns a clean error (ASP.NET Core exception-handling middleware → `ProblemDetails`; a top-level handler in a console/worker).
- **Integration boundaries** — wrap calls to external systems (DB, HTTP, files) where failure is expected, translate to domain errors or apply resilience (retry/circuit breaker — DotNetBook's Resilience chapter).
- **Domain logic** — mostly *don't* catch; let exceptions/results propagate to a boundary that can decide.

```csharp
// ASP.NET Core — one global boundary instead of try/catch in every handler
app.UseExceptionHandler();   // converts unhandled exceptions to ProblemDetails (RFC 7807)
app.MapGet("/orders/{id}", (int id, IOrderService svc) => svc.Get(id));   // throws freely; middleware handles it
```

Catching the same exception at every layer (and re-logging it) produces noise and duplicate logs. Let it bubble to the boundary that owns the response.

---

## Validation strategy

Validation is *expected* failure — prefer results/validation objects over exceptions:

```csharp
// Collect all errors, don't throw on the first (better UX)
public ValidationResult Validate(OrderRequest req) {
    var errors = new List<string>();
    if (req.Items.Count == 0) errors.Add("At least one item required.");
    if (req.Total < 0) errors.Add("Total cannot be negative.");
    return errors.Count == 0 ? ValidationResult.Valid : ValidationResult.Invalid(errors);
}
```

Validate untrusted input at the **boundary** (API edge, deserialization). Once validated, trust it internally (guard clauses there are for *programmer* errors, asserted not user-facing). DataAnnotations / FluentValidation handle this in ASP.NET (DotNetBook).

---

## Logging and observability

When you handle an error, log it with **context** and the right severity:

```csharp
// ✓ — structured logging with context; right level
_logger.LogError(ex, "Failed to charge order {OrderId} for {Amount}", order.Id, order.Total);

// ✗ — no exception object, no context, wrong level
_logger.LogInformation("error");
```

- Log the **exception object** (not just `.Message`) — preserves the stack.
- Use **structured** logging (named placeholders) for queryability.
- Right severity: `LogError`/`LogCritical` for failures, `LogWarning` for recoverable/expected-but-notable, not `LogInformation`.
- **Log once** — at the boundary where you handle it. Logging-and-rethrowing at every layer floods logs with duplicates.
- Never log secrets/PII.

See [Chapter 12 §03](../12-Reflection/03-Attributes.md) for `CallerArgumentExpression`-powered context, and DotNetBook's Observability chapter for the full picture.

---

## Don't use exceptions for control flow

```csharp
// ✗ — exception as control flow (slow, noisy)
try { value = dict[key]; } catch (KeyNotFoundException) { value = fallback; }

// ✓ — Try-pattern
value = dict.TryGetValue(key, out var v) ? v : fallback;

// ✗ — parse-by-exception
try { n = int.Parse(input); } catch (FormatException) { n = 0; }

// ✓
n = int.TryParse(input, out var parsed) ? parsed : 0;
```

The BCL provides Try-patterns precisely so you don't pay exception cost for expected failures. Use them.

---

## Common bugs / gotchas

### Swallowing exceptions

Empty `catch { }` hides failures until they corrupt something far away. Always at least log; usually rethrow.

### `throw ex;` losing the stack

Resets the trace to the rethrow site. Use `throw;` or wrap with an inner exception.

### Catching `Exception` broadly

Hides bugs and can catch things you must not (`OutOfMemoryException`, `StackOverflowException` — the latter isn't even catchable). Catch specific types.

### Exceptions for routine failures

Login failures, validation, not-found — these are expected. Using exceptions makes them slow and noisy. Use results/Try-patterns.

### Logging the same error at every layer

Duplicate log entries, hard to trace. Log once, at the handling boundary.

### Translating without preserving the inner exception

`throw new MyException("failed")` loses the cause. Always pass the original as `innerException`.

---

## Summary

- First decide: **exceptional** (throw) vs **expected** (result/Try-pattern). Don't use exceptions for control flow.
- **Result types** (`Result<T>`, ErrorOr) for routine/business failures; **exceptions** for bugs and broken invariants.
- Throw **specific** exceptions with `nameof` + clear messages; custom exceptions only when callers catch them distinctly.
- Catch **specifically**, use `throw;` (preserve stack) or wrap with inner exception; use `when` filters; **never swallow**.
- Handle errors at **boundaries** (global handler, integration edges), not in every layer; let domain code propagate.
- Validate untrusted input at the edge (collect errors, don't throw); **log once**, with context, the exception object, and the right severity.

→ Next: [14-Equality.md](14-Equality.md)
