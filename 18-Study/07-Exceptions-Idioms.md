# 07 — Exceptions & Idioms (error handling, best practices, anti-patterns)

## ⚡ 30-second answer

Exceptions are for **exceptional, unexpected** conditions — not control flow (they're expensive: stack capture + unwinding). Use **`try/catch/finally`**; prefer **`using`** for deterministic cleanup of `IDisposable`. Catch the **most specific** exception you can handle, and **don't swallow** exceptions (empty `catch`). For *expected* failures (parse, lookup, validation) use the **`Try...` pattern** or a result type instead of throwing. Rethrow with `throw;` (preserves the stack); `throw ex;` resets it.

---

## Core mechanics

```csharp
try {
    var data = await LoadAsync(ct);
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) {  // filter
    return null;                       // handle the specific case
}
catch (OperationCanceledException) {
    throw;                             // rethrow — PRESERVES stack trace
}
finally {
    // always runs (cleanup) — even on exception or return
}
```

- **`throw;`** rethrows preserving the original stack trace. **`throw ex;`** *resets* the stack to here (loses the origin) — almost always wrong.
- **Exception filters** (`catch ... when (cond)`) avoid catching/rethrowing and keep the stack; the filter runs *before* the stack unwinds.
- **`finally`** always runs (for cleanup); `using` is sugar for try/finally + `Dispose`.

**The `Try` pattern** — for expected failures, return a bool instead of throwing:
```csharp
if (int.TryParse(s, out var n)) { ... }   // no exception on bad input
if (dict.TryGetValue(key, out var v)) { ... }
```

**Guard clauses (defensive programming)** — fail fast at the boundary:
```csharp
ArgumentNullException.ThrowIfNull(input);
ArgumentException.ThrowIfNullOrEmpty(name);
ObjectDisposedException.ThrowIf(_disposed, this);
```

**Custom exceptions**: derive from `Exception`, add context, keep them meaningful; include an inner exception when wrapping.

---

## Comparison tables

| Situation | Use |
|---|---|
| Unexpected/unrecoverable | throw an exception |
| Expected failure (parse/lookup/validation) | `Try...` pattern / result type |
| Cleanup needed | `using` / `finally` |
| Rethrow after logging | `throw;` (not `throw ex;`) |
| Conditional catch | `catch ... when (...)` filter |

| | `throw;` | `throw ex;` |
|---|---|---|
| Stack trace | **preserved** | **reset to here** |
| Use | rethrow | (avoid) |

---

## 🪤 Traps & gotchas

- **Swallowing exceptions**: empty `catch {}` hides bugs and corrupts state silently. Catch only what you can handle; let the rest propagate.
- **`catch (Exception)` everywhere**: too broad — masks `OutOfMemory`, programming bugs (`NullReferenceException`), etc. Catch specific types.
- **`throw ex;`** resets the stack trace — debugging nightmare. Use `throw;`.
- **Exceptions for control flow**: throwing on every "not found"/"invalid input" is slow and obscures intent. Use `Try`/result types ([21](21-Deployment-Perf-Tooling.md) — high exception rate is a perf smell).
- **`async void` + exceptions**: an exception in `async void` can't be caught by the caller and **crashes the process**. Use `async Task` (except event handlers) ([10](10-AsyncAwait.md)).
- **`finally` can swallow**: a `return`/`throw` inside `finally` overrides the original exception. Don't throw from `finally`.
- **`OperationCanceledException`**: cancellation throws this — don't treat it as an error; let it propagate (it's expected) ([10](10-AsyncAwait.md)).
- **Catching and not preserving**: wrapping without an inner exception loses the root cause. `throw new MyException("...", ex);`

---

## ❓ Likely questions

**Q: `throw;` vs `throw ex;`?**
A: `throw;` rethrows the current exception preserving its original stack trace. `throw ex;` throws it anew, resetting the stack to the rethrow point (loses origin). Use `throw;`.

**Q: When should you NOT use exceptions?**
A: For expected, frequent failures (parsing, lookups, validation) — use the `Try` pattern or a result type. Exceptions are costly and meant for exceptional cases.

**Q: What does `finally` guarantee?**
A: It runs whether the try succeeds, throws, or returns — for cleanup. `using` compiles to try/finally + `Dispose`.

**Q: Why is `async void` dangerous?**
A: Its exceptions escape to the synchronization context and can crash the process; the caller can't await or catch it. Use `async Task`.

**Q: What are exception filters?**
A: `catch (E ex) when (condition)` — catches only if the condition is true, evaluated before unwinding, so you don't catch-and-rethrow (which would disturb the stack).

**Q: How do guard clauses help?**
A: They validate inputs at the top of a method and fail fast with a clear exception (`ArgumentNullException.ThrowIfNull`), preventing corrupted state and giving precise errors.

**Q: How do you preserve the root cause when wrapping?**
A: Pass the original as the `innerException`: `throw new DomainException("...", ex);`.

---

## 🎓 Senior Extra

- **Cost model**: throwing captures a stack trace (walks frames) and unwinds — orders of magnitude slower than a return. Profilers show "Exception Count" ([21](21-Deployment-Perf-Tooling.md)); a high rate signals exceptions-as-control-flow.
- **Result types / discriminated-union style**: for expected error paths, returning `Result<T>`/`OneOf<...>` (or out-params) makes failure explicit in the signature and avoids throw/catch overhead — popular in functional-leaning and high-throughput code; trade-off is ceremony.
- **`AggregateException`**: `Task.WhenAll`/parallel work surfaces multiple failures wrapped in an `AggregateException`; `await` unwraps the *first*, but inspect `.InnerExceptions` for the rest ([09](09-Threading-and-Tasks.md)).
- **Global handling**: ASP.NET Core uses exception-handling middleware → **ProblemDetails** (RFC 7807) for consistent API errors ([16](16-AspNetCore.md)); don't try/catch in every controller.
- **`ExceptionDispatchInfo.Capture(ex).Throw()`** rethrows a stored exception preserving the original stack (used by infrastructure to marshal exceptions across boundaries).
- **First-chance vs handled**: debuggers/`AppDomain.FirstChanceException` see exceptions before any catch — useful for diagnostics.
- **Don't catch what you can't handle**: catching `StackOverflowException`/`OutOfMemoryException`/`AccessViolationException` is impossible or pointless — let the process fail fast. Reserve broad catches for top-level boundaries that log + translate.
- **Idiomatic best practices**: prefer immutability, accept the narrowest type (`IEnumerable<T>`) and return the most useful (`IReadOnlyList<T>`), name async methods `...Async`, put `CancellationToken` last, and avoid the anti-patterns in [22](22-Architecture-Patterns.md) (god classes, primitive obsession, service locator, sync-over-async).

→ Deeper: [`../CSharpBook/17-BestPractices/`](../CSharpBook/17-BestPractices/README.md)
