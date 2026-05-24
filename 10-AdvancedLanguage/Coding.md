# Chapter 10 — Coding Problems

> 12 hands-on problems on modern C# features.

---

## Problem 1 — Annotate this API with NRT

```csharp
public class UserService {
    public User FindUser(int id) { /* may return null */ }
    public string GetName(int id) { /* may return null */ }
    public void Save(User user) { /* user must be non-null */ }
}
```

<details><summary>Solution</summary>

```csharp
public class UserService {
    public User? FindUser(int id) { /* may return null */ }
    public string? GetName(int id) { /* may return null */ }
    public void Save(User user) {
        ArgumentNullException.ThrowIfNull(user);
        // ...
    }
}
```

Mark optional returns with `?`. For non-null parameters, validate explicitly — the type alone is a hint, not enforcement.

</details>

---

## Problem 2 — Build a state machine with patterns

Model order states + valid transitions. Use sealed records + pattern matching.

<details><summary>Solution</summary>

```csharp
public abstract record OrderState;
public sealed record Pending : OrderState;
public sealed record Paid(string TransactionId) : OrderState;
public sealed record Shipped(string TrackingNumber, DateTime ShippedAt) : OrderState;
public sealed record Delivered(DateTime DeliveredAt) : OrderState;
public sealed record Cancelled(string Reason) : OrderState;

public abstract record OrderEvent;
public sealed record Pay(string TransactionId) : OrderEvent;
public sealed record Ship(string TrackingNumber) : OrderEvent;
public sealed record Deliver : OrderEvent;
public sealed record Cancel(string Reason) : OrderEvent;

public static OrderState Apply(OrderState state, OrderEvent ev) =>
    (state, ev) switch {
        (Pending, Pay(var tx)) => new Paid(tx),
        (Paid, Ship(var t)) => new Shipped(t, DateTime.UtcNow),
        (Shipped, Deliver) => new Delivered(DateTime.UtcNow),
        (Pending or Paid, Cancel(var r)) => new Cancelled(r),
        _ => throw new InvalidOperationException($"Illegal transition: {state.GetType().Name} + {ev.GetType().Name}")
    };

// Use
var s = (OrderState)new Pending();
s = Apply(s, new Pay("tx-123"));
s = Apply(s, new Ship("trk-456"));
s = Apply(s, new Deliver());
Console.WriteLine(s);   // Delivered { DeliveredAt = ... }
```

Tuple pattern of (state, event), destructure to capture payload. Combine type patterns with positional. Default arm throws on invalid transitions.

</details>

---

## Problem 3 — DTO with required + init

Build `CreateUserRequest` with mandatory `Email` and `Password`, optional `DisplayName`.

<details><summary>Solution</summary>

```csharp
public record CreateUserRequest {
    public required string Email { get; init; }
    public required string Password { get; init; }
    public string? DisplayName { get; init; }
}

// Use
var r = new CreateUserRequest {
    Email = "alice@example.com",
    Password = "secret",
    // DisplayName optional
};
```

`required` enforces Email and Password are set. `init` makes them immutable after construction. No constructor needed.

For JSON deserialization (.NET 8+), required is enforced — if JSON omits Email, throw.

</details>

---

## Problem 4 — Implement INotifyPropertyChanged with CallerMemberName

<details><summary>Solution</summary>

```csharp
public abstract class ViewModelBase : INotifyPropertyChanged {
    public event PropertyChangedEventHandler? PropertyChanged;

    protected void OnPropertyChanged([CallerMemberName] string? name = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));

    protected bool SetField<T>(ref T field, T value, [CallerMemberName] string? name = null) {
        if (EqualityComparer<T>.Default.Equals(field, value)) return false;
        field = value;
        OnPropertyChanged(name);
        return true;
    }
}

public class UserVm : ViewModelBase {
    private string _name = "";
    public string Name {
        get => _name;
        set => SetField(ref _name, value);
    }
}
```

No magic strings; refactor-friendly.

</details>

---

## Problem 5 — UTF-8 HTTP response writer

Write a method that writes an HTTP/1.1 200 OK response with a JSON body, **zero allocations beyond the body string**.

<details><summary>Solution</summary>

```csharp
public static async Task WriteJsonResponseAsync(Stream stream, string body) {
    // Write static parts as UTF-8 spans
    await stream.WriteAsync("HTTP/1.1 200 OK\r\n"u8);
    await stream.WriteAsync("Content-Type: application/json\r\n"u8);
    await stream.WriteAsync("Content-Length: "u8);

    Span<byte> lenBuf = stackalloc byte[16];
    if (Encoding.UTF8.TryGetBytes(body.Length.ToString(), lenBuf, out int written)) {
        await stream.WriteAsync(lenBuf[..written]);
    }

    await stream.WriteAsync("\r\n\r\n"u8);

    // Body
    byte[] bodyBytes = Encoding.UTF8.GetBytes(body);   // one allocation
    await stream.WriteAsync(bodyBytes);
}
```

Most parts are `u8` literals — embedded bytes, zero allocation. The body string is the only allocation (and unavoidable since the body's content varies). For Kestrel-class performance, even more aggressive: keep the body as `ReadOnlySpan<byte>` from the source.

</details>

---

## Problem 6 — Convert to collection expressions

Modernize:

```csharp
var arr = new int[] { 1, 2, 3 };
var list = new List<string> { "a", "b" };
var imm = ImmutableArray.Create(1, 2, 3);
var combined = arr.Concat(new[] { 4, 5 }).ToArray();
```

<details><summary>Solution</summary>

```csharp
int[] arr = [1, 2, 3];
List<string> list = ["a", "b"];
ImmutableArray<int> imm = [1, 2, 3];
int[] combined = [..arr, 4, 5];
```

Cleaner, often more efficient (compiler picks the best construction).

</details>

---

## Problem 7 — Raw string for JSON test fixture

A test verifies parsing of nested JSON. Write the input cleanly with raw strings + interpolation.

<details><summary>Solution</summary>

```csharp
[Fact]
public void Parse_Nested_Works() {
    int userId = 42;
    string name = "Alice";
    string json = $$"""
        {
            "user": {
                "id": {{userId}},
                "name": "{{name}}",
                "tags": ["admin", "early-adopter"]
            },
            "metadata": {
                "version": 1
            }
        }
        """;

    var doc = JsonDocument.Parse(json);
    doc.RootElement.GetProperty("user").GetProperty("id").GetInt32().Should().Be(userId);
}
```

Multi-line raw string with `$$` for placeholder safety (literal `{` `}` for JSON object syntax; `{{userId}}` for interpolation).

</details>

---

## Problem 8 — Implement a generic guard

`Ensure<T>` that throws if the argument is null, with rich context.

<details><summary>Solution</summary>

```csharp
public static T Ensure<T>(
    [NotNull] T? value,
    [CallerArgumentExpression(nameof(value))] string? expression = null,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0) where T : class
{
    if (value is null) {
        throw new InvalidOperationException(
            $"Expected non-null '{expression}' at {Path.GetFileName(file)}:{line}");
    }
    return value;
}

// Use
var user = Ensure(repo.FindById(42));
// On null: "Expected non-null 'repo.FindById(42)' at MyService.cs:23"
```

`[NotNull]` makes the compiler treat the returned `value` as non-null after the call (for NRT). `[CallerArgumentExpression]` captures the call's text. CallerFilePath / CallerLineNumber give location.

</details>

---

## Problem 9 — Pattern matching for input parsing

Parse a simple "name=value" format. Handle multiple cases: empty, only key, key=value, multiple pairs.

<details><summary>Solution</summary>

```csharp
public static IEnumerable<(string Key, string? Value)> Parse(string input) {
    if (string.IsNullOrWhiteSpace(input)) yield break;

    foreach (var part in input.Split(';', StringSplitOptions.RemoveEmptyEntries)) {
        var trimmed = part.Trim();
        var result = trimmed switch {
            "" => default,
            var p when p.IndexOf('=') is int idx && idx >= 0 =>
                (p[..idx].Trim(), p[(idx + 1)..].Trim() as string?),
            var p => (p, null as string?)
        };
        if (result != default) yield return result;
    }
}

foreach (var (k, v) in Parse("name=Alice; admin; age=30")) {
    Console.WriteLine($"{k} = {v ?? "(no value)"}");
}
// name = Alice
// admin = (no value)
// age = 30
```

Switch expression with `when` for the parseable cases. Tuple return.

</details>

---

## Problem 10 — Optimize a logging hot path

This logs in a tight loop. Make it efficient:

```csharp
for (int i = 0; i < 1_000_000; i++) {
    _logger.LogDebug($"Processing item {i}: {ComputeData(i)}");
}
```

<details><summary>Solution</summary>

The interpolated handler approach IS already optimized — `ComputeData(i)` is called only when debug is enabled. But for max efficiency:

```csharp
// Best: use structured logging
for (int i = 0; i < 1_000_000; i++) {
    if (_logger.IsEnabled(LogLevel.Debug)) {
        _logger.LogDebug("Processing item {Index}: {Data}", i, ComputeData(i));
    }
}
```

The explicit `IsEnabled` check is theoretically redundant (the interpolated handler does it), but for structured loggers (Serilog, App Insights) the named placeholder version preserves structure.

The `if (IsEnabled)` guard ensures ComputeData isn't called even if the named-placeholder version of LogDebug always evaluates its args (older logging APIs). Modern frameworks defer; the guard is belt-and-suspenders.

</details>

---

## Problem 11 — Custom interpolated string handler

Implement a handler that ONLY writes to the output if a condition is true. Skip both string-building and argument-evaluation if false.

<details><summary>Solution</summary>

```csharp
using System.Runtime.CompilerServices;

[InterpolatedStringHandler]
public ref struct ConditionalHandler {
    private DefaultInterpolatedStringHandler _inner;
    private readonly bool _enabled;

    public ConditionalHandler(
        int literalLength, int formattedCount, bool enabled,
        out bool shouldAppend)
    {
        _enabled = enabled;
        if (!enabled) {
            _inner = default;
            shouldAppend = false;
            return;
        }
        _inner = new DefaultInterpolatedStringHandler(literalLength, formattedCount);
        shouldAppend = true;
    }

    public bool AppendLiteral(string s) {
        if (_enabled) _inner.AppendLiteral(s);
        return _enabled;
    }

    public bool AppendFormatted<T>(T value) {
        if (_enabled) _inner.AppendFormatted(value);
        return _enabled;
    }

    public string GetText() => _enabled ? _inner.ToStringAndClear() : "";
}

public static class ConditionalLogger {
    public static void Log(
        bool enabled,
        [InterpolatedStringHandlerArgument(nameof(enabled))]
        ref ConditionalHandler message)
    {
        if (enabled) Console.WriteLine(message.GetText());
    }
}

// Use
ConditionalLogger.Log(false, $"Expensive: {ComputeBigData()}");
// ComputeBigData NOT called!
```

`shouldAppend` is the key. When false, the compiler skips evaluating each interpolated expression entirely.

</details>

---

## Problem 12 — File-scoped helper for source generator

Write source-generated code that adds a `ToJson` method to a `[Serialize]`-attributed class. Use a `file` class for the helper.

<details><summary>Solution sketch (generated code)</summary>

```csharp
// Generated file: PersonSerializer.g.cs
file static class PersonSerializer {
    public static string Serialize(Person p) {
        return $$"""
            {
                "name": "{{p.Name}}",
                "age": {{p.Age}}
            }
            """;
    }
}

partial class Person {
    public string ToJson() => PersonSerializer.Serialize(this);
}
```

`file static class PersonSerializer` — only visible in this generated file. If a similar generator runs for another `[Serialize]` class (e.g., Order), its `OrderSerializer` doesn't collide because each is `file`-scoped.

Without `file`, you'd need mangled names or namespace tricks.

</details>

---

## Summary

You've now drilled:
- NRT annotations and flow analysis.
- Pattern matching at depth (positional, property, list, slice, logical, when).
- Records (synthesized members, EqualityContract, with expressions).
- Required members + [SetsRequiredMembers].
- File-scoped types for source generators.
- Top-level statements (modern entry point).
- Global / implicit usings.
- Collection expressions ([1, 2, 3], spread ..).
- Raw strings ("""..."""), interpolation, u8 literals.
- Interpolated string handlers (DefaultInterpolatedStringHandler).
- Caller info attributes (especially CallerArgumentExpression).
- Checked / unchecked + user-defined operators.

This is the modern C# vocabulary. Master it.

→ [Chapter 11 — Modern Features (Version by Version)](../11-ModernFeatures/README.md)
