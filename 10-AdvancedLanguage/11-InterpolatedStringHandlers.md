# Interpolated String Handlers (C# 10+)

## What it is

C# 10 (2021) introduced **interpolated string handlers** — a way for libraries to take an `$"..."` expression and process its parts (literals + interpolation values) **without first building the full string**. Used by `Debug.Assert`, `ILogger.LogInformation`, `string.Create`, and many BCL hot paths.

```csharp
// Pre-handler: $"..." always built a string, even if unused
Debug.Assert(condition, $"Failed with {expensiveValue}");
// expensiveValue was computed AND the string was built every call

// Post-handler (.NET 6+): Debug.Assert takes an interpolated handler
// expensiveValue and string are computed only if condition is false
```

This is mostly an **author-side feature** (library design). As a consumer, you write `$"..."` normally and benefit from the optimization automatically.

---

## Why it exists

The classic problem:

```csharp
if (logger.IsEnabled(LogLevel.Debug)) {
    logger.Log($"Slow: {ExpensiveCompute()}");   // only computes if debug enabled
}
```

You'd write the explicit guard to avoid `ExpensiveCompute` when debug logging is off. Annoying.

With interpolated string handlers, ILogger can defer:

```csharp
logger.LogDebug("Slow: {value}", ExpensiveCompute);   // old style; always computes
```

Wait, the older style with named placeholders ALSO always computes. The handler form is:

```csharp
logger.LogDebug($"Slow: {ExpensiveCompute()}");
// With handler, the args (including ExpensiveCompute()) are evaluated lazily
```

The handler's API can decide whether to even materialize the string, AND whether to evaluate the interpolated expressions. Skip the work entirely if the log level is off.

Similarly for `Debug.Assert`, `ArgumentException.ThrowIf*`, etc. — fully bypass message construction when the condition isn't met.

---

## The user-facing benefit

You write:

```csharp
logger.LogDebug($"Loaded {entity.Name} with {result.Count} items");
```

With handler-aware logger:
- If debug isn't enabled, neither `entity.Name` nor `result.Count` is accessed, AND no string is built. **Zero work** for disabled logs.
- If enabled, the handler builds the string efficiently using a span buffer + formatter, often beating `string.Format` in throughput.

Same syntax. Smarter behavior under the hood.

---

## API: defining a handler

Handlers are types annotated with `[InterpolatedStringHandler]`. The compiler recognizes them and rewrites `$"..."` calls accordingly.

```csharp
using System.Runtime.CompilerServices;

[InterpolatedStringHandler]
public ref struct MyHandler {
    private DefaultInterpolatedStringHandler _inner;
    private readonly bool _enabled;

    public MyHandler(int literalLength, int formattedCount, bool enabled, out bool shouldAppend) {
        if (!enabled) {
            _inner = default;
            _enabled = false;
            shouldAppend = false;   // tells the compiler to SKIP appends
            return;
        }
        _inner = new DefaultInterpolatedStringHandler(literalLength, formattedCount);
        _enabled = true;
        shouldAppend = true;
    }

    public void AppendLiteral(string s) {
        if (_enabled) _inner.AppendLiteral(s);
    }

    public void AppendFormatted<T>(T value) {
        if (_enabled) _inner.AppendFormatted(value);
    }

    public string GetFormattedText() => _enabled ? _inner.ToStringAndClear() : "";
}

public static class Logger {
    public static void Log(bool enabled, [InterpolatedStringHandlerArgument("enabled")] ref MyHandler message) {
        if (enabled) {
            Console.WriteLine(message.GetFormattedText());
        }
    }
}

// Use
Logger.Log(false, $"Expensive: {Compute()}");
// Compute() is NOT called! shouldAppend = false short-circuits.
```

The magic: the `out bool shouldAppend` in the constructor tells the compiler "should I bother appending?" If false, the compiler skips evaluating the interpolated expressions entirely.

---

## How the compiler rewrites

```csharp
Logger.Log(enabled, $"Slow: {expensive}");
```

Compiles to roughly:

```csharp
var handler = new MyHandler(literalLength: 6, formattedCount: 1, enabled, out var shouldAppend);
if (shouldAppend) {
    handler.AppendLiteral("Slow: ");
    handler.AppendFormatted(expensive);   // only evaluated here
}
Logger.Log(enabled, ref handler);
```

The handler is constructed first. It decides whether the rest should run. If not, the expensive expression isn't evaluated.

---

## Built-in handlers

The BCL ships with `DefaultInterpolatedStringHandler` — used when you write `$"..."` in a `string` context:

```csharp
string s = $"Hello {name}!";
```

Becomes:

```csharp
var handler = new DefaultInterpolatedStringHandler(7, 1);
handler.AppendLiteral("Hello ");
handler.AppendFormatted(name);
handler.AppendLiteral("!");
string s = handler.ToStringAndClear();
```

`DefaultInterpolatedStringHandler` uses a stack-allocated buffer (Span<char>) for the result, expanding to ArrayPool if needed. Often beats `string.Format` and StringBuilder for typical sizes.

For most code, you don't think about handlers — `$"..."` just works fast.

---

## When library authors care

Built-in handler users in the BCL:
- `Debug.Assert(cond, $"...")` — message lazy.
- `ArgumentNullException.ThrowIfNull(arg, $"...")` — message lazy (only built on throw).
- `ILogger.LogInformation($"...")` — lazy / level-aware.
- `StringBuilder.Append($"...")` — appends directly without intermediate string.
- `string.Create(...)` — for advanced custom formatting.

For library authors writing high-throughput APIs, custom handlers let you:
- Skip work conditionally.
- Use a pooled buffer.
- Stream to a Stream / TextWriter directly.

For most application code, you benefit automatically without writing handlers.

---

## ILogger handler

ASP.NET Core's `ILogger` extension methods accept interpolated handlers:

```csharp
_logger.LogInformation($"User {user.Name} logged in at {DateTime.UtcNow:O}");
```

Under the hood:
- Handler checks `IsEnabled(LogLevel.Information)`.
- If not enabled, doesn't evaluate `user.Name`, `DateTime.UtcNow`, doesn't build the string.
- If enabled, formats efficiently.

**However**: this hides the **structured logging** that ILogger supports:

```csharp
_logger.LogInformation("User {Name} logged in at {Time}", user.Name, DateTime.UtcNow);
```

The named-placeholder form lets log providers (Serilog, Application Insights) capture `Name` and `Time` as **structured properties**. The interpolated form produces a single rendered string — loses the structure.

For production logging with structured providers: use the named-placeholder form. The interpolated form is fine for simple cases or where you don't structure-log.

---

## DefaultInterpolatedStringHandler internals

```csharp
public ref struct DefaultInterpolatedStringHandler {
    private char[]? _arrayToReturnToPool;
    private Span<char> _chars;
    private int _pos;

    public DefaultInterpolatedStringHandler(int literalLength, int formattedCount) {
        _chars = (literalLength + formattedCount * 11 < 256)
            ? stackalloc char[literalLength + formattedCount * 11]   // estimate
            : (_arrayToReturnToPool = ArrayPool<char>.Shared.Rent(...));
        _pos = 0;
    }

    public void AppendLiteral(string s) {
        s.AsSpan().CopyTo(_chars[_pos..]);
        _pos += s.Length;
    }

    public void AppendFormatted<T>(T value) {
        if (value is ISpanFormattable sf) {
            sf.TryFormat(_chars[_pos..], out int w, default, null);
            _pos += w;
        } else {
            string s = value?.ToString() ?? "";
            s.AsSpan().CopyTo(_chars[_pos..]);
            _pos += s.Length;
        }
    }

    public string ToStringAndClear() {
        var s = new string(_chars[.._pos]);
        if (_arrayToReturnToPool is not null) {
            ArrayPool<char>.Shared.Return(_arrayToReturnToPool);
        }
        return s;
    }
}
```

(Simplified; real code is more complex.)

Notable:
- Uses stackalloc for small estimated sizes.
- Falls back to ArrayPool for large.
- Uses `ISpanFormattable` for primitives (int, DateTime, etc.) — formats DIRECTLY to the span, no intermediate string.

For 99% of interpolated strings, this is more efficient than `string.Format`. Same syntax, better performance.

---

## When user code might write a custom handler

Rare. Most app code consumes handlers, doesn't define them.

When you might:
- High-throughput logging library.
- Custom serializer that wants to skip work when output target is closed/null.
- Specialized formatting (e.g., escaped HTML output that's expensive when not actually emitted).

For everyday code: enjoy the compiler optimization, don't write handlers.

---

## Common patterns (consumer side)

### Just write `$"..."`

```csharp
logger.LogDebug($"Processed {result.Count} items in {sw.ElapsedMilliseconds}ms");
```

For Debug level, if disabled, no work. For Information+ (typically enabled), work happens efficiently.

### `ArgumentException.ThrowIf*` with interpolated message

```csharp
ArgumentException.ThrowIfNullOrWhiteSpace(input, $"Invalid input '{input}'");
```

The message is built only if the throw triggers. Lazy by design.

### `Debug.Assert` for expensive checks

```csharp
Debug.Assert(IsValidUnchecked(value), $"Value {value} failed: {ExpensiveValidation(value)}");
```

Expensive validation runs only when assert fires (i.e., in Debug builds AND when condition is false).

---

## Common bugs

### Side effects in interpolated expressions

```csharp
Debug.Assert(cond, $"State: {ResetCounter()}");
// ResetCounter is only called IF cond is false
```

If you depend on `ResetCounter` running, the interpolated form is wrong — it's conditional. Use a regular method call:

```csharp
int c = ResetCounter();
Debug.Assert(cond, $"State: {c}");
```

### Missing structured logging in ILogger

```csharp
_logger.LogInformation($"User {userId} action");   // string only — loses structure
_logger.LogInformation("User {UserId} action", userId);   // structured — Serilog can index userId
```

For structured logging providers, prefer named placeholders. The interpolated form loses information.

(Some loggers support both; check your provider.)

### Long interpolated strings might allocate

```csharp
string s = $"...{very}...{long}...{many}...{interpolations}...";
```

For very large interpolated strings, the handler falls back to ArrayPool. Still beats StringBuilder for moderate sizes.

For mega-strings, consider building incrementally yourself.

---

## Performance vs alternatives

For a typical `$"prefix {value} suffix"`:

| Method | Time | Allocations |
|---|---|---|
| `$"..."` (handler) | ~50 ns | 1 string |
| `string.Format("...{0}...", value)` | ~80 ns | 1 string + boxed args |
| `string.Concat("prefix ", value.ToString(), " suffix")` | ~70 ns | 1 string + intermediate |
| StringBuilder + Append × 3 + ToString | ~150 ns | 1 StringBuilder + 1 string |

Interpolated handler wins. Same syntax as old interpolation; faster underneath.

---

## When to think about handlers

For library authors writing hot-path APIs:
- Consider exposing interpolated-handler overloads.
- The `[InterpolatedStringHandler]` ref struct + the consumer API with `[InterpolatedStringHandlerArgument(...)]` parameter.

For consumers:
- Just write `$"..."`. Library APIs decide if they consume handlers.

For perf-critical formatting:
- Use `string.Create` with a delegate for max control.
- Or `IUtf8SpanFormattable` for UTF-8 byte output.

---

## Summary

- Interpolated string handlers let libraries lazily process `$"..."` arguments.
- Mostly a library-author feature; consumers just write `$"..."`.
- `DefaultInterpolatedStringHandler` (BCL) makes regular interpolation faster than `string.Format`.
- `Debug.Assert`, `ArgumentException.ThrowIf*`, `ILogger.Log*` use handlers — args lazy.
- For structured logging providers, prefer named placeholders over interpolation.
- Compile-time rewrite; no runtime overhead beyond the handler itself.

→ Next: [12-CallerInfoAttributes.md](12-CallerInfoAttributes.md)
