# Caller Info Attributes

## What it is

C# 5+ introduced **caller info attributes**: when applied to optional method parameters, the compiler fills them in with information about the **call site** at each call. Used for logging, validation, debugging.

```csharp
public static void Log(
    string message,
    [CallerMemberName] string? caller = null,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"[{caller}@{Path.GetFileName(file)}:{line}] {message}");
}

// In another method:
public void DoSomething() {
    Log("hello");
    // Prints: [DoSomething@MyFile.cs:42] hello
}
```

The compiler injects the calling method's name, file, and line. The values are baked in at **compile time** — no runtime reflection.

C# 11 added `[CallerArgumentExpression(...)]` for capturing the actual text of an argument — used heavily by modern guard methods (`ArgumentException.ThrowIfNull`).

---

## The attributes

| Attribute | What it injects |
|---|---|
| `[CallerMemberName]` | Name of the calling method, property, or event |
| `[CallerFilePath]` | Absolute path of the source file |
| `[CallerLineNumber]` | Line number of the call site |
| `[CallerArgumentExpression("paramName")]` | Source text of the named argument |

All apply to **optional parameters** with default values. The compiler replaces the default at each call site.

---

## `[CallerMemberName]`

```csharp
public class Vm : INotifyPropertyChanged {
    public event PropertyChangedEventHandler? PropertyChanged;

    private int _count;
    public int Count {
        get => _count;
        set {
            if (_count != value) {
                _count = value;
                OnPropertyChanged();   // compiler injects "Count"
            }
        }
    }

    private void OnPropertyChanged([CallerMemberName] string? name = null) {
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
    }
}
```

The MVVM `INotifyPropertyChanged` pattern. Pre-CallerMemberName, you'd hardcode property names — and break when you renamed.

```csharp
OnPropertyChanged();   // becomes OnPropertyChanged("Count") at compile time
```

For events / methods: same thing — the enclosing member's name.

---

## `[CallerFilePath]` and `[CallerLineNumber]`

```csharp
public static void Log(
    string msg,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"[{file}:{line}] {msg}");
}

// Caller
Log("oops");   // becomes Log("oops", "C:\\src\\MyApp\\Service.cs", 42)
```

For diagnostics, error reports, debug logs.

**Caveat**: file path is the absolute path on the build machine. For redistributed binaries, this might be a developer's path — confusing or sensitive.

To deterministically map: use `[DeterministicBuild]` or `<PathMap>` MSBuild property to normalize paths in the assembly.

---

## `[CallerArgumentExpression]` (C# 10+)

The killer attribute. Captures the **source text** of another argument:

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression(nameof(argument))] string? paramName = null)
{
    if (argument is null) throw new ArgumentNullException(paramName);
}

// Caller
ThrowIfNull(user);
// Compiles to: ThrowIfNull(user, "user")
// If thrown: ArgumentNullException with paramName = "user"

ThrowIfNull(user.Address.City);
// Compiles to: ThrowIfNull(user.Address.City, "user.Address.City")
```

The compiler injects the full expression as a string. Used heavily by modern BCL guard methods:
- `ArgumentNullException.ThrowIfNull(arg)`.
- `ArgumentException.ThrowIfNullOrEmpty(str)`.
- `ArgumentOutOfRangeException.ThrowIfNegative(n)`.
- `Debug.Assert(cond)`.

No need to manually write `nameof(arg)` everywhere — the compiler captures the expression.

---

## Practical patterns

### Modern guard methods

```csharp
public void Process(string input) {
    ArgumentException.ThrowIfNullOrWhiteSpace(input);
    // If input is null/whitespace: throws with paramName="input"
}
```

The `[CallerArgumentExpression]` mechanism captures "input" automatically. Pre-attribute, you'd write:

```csharp
if (string.IsNullOrWhiteSpace(input)) throw new ArgumentException("...", nameof(input));
```

The attribute form is shorter AND robust to renaming.

### Custom validators

```csharp
public static T NotNull<T>(
    T? arg,
    [CallerArgumentExpression(nameof(arg))] string? expression = null) where T : class
{
    if (arg is null) throw new ArgumentNullException(expression);
    return arg;
}

// Use
var user = NotNull(repo.FindUser(id));   // throws with paramName="repo.FindUser(id)"
```

Inline validation that propagates expressive error info.

### Logging with context

```csharp
public static void LogError(
    Exception ex,
    [CallerMemberName] string? caller = null,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0)
{
    _logger.LogError(ex, "Error in {Caller} at {File}:{Line}", caller, file, line);
}
```

Every error logs its location automatically.

### Test assertion helpers

```csharp
public static void ShouldEqual<T>(
    this T actual,
    T expected,
    [CallerArgumentExpression(nameof(actual))] string? actualExpr = null,
    [CallerArgumentExpression(nameof(expected))] string? expectedExpr = null)
{
    if (!EqualityComparer<T>.Default.Equals(actual, expected)) {
        throw new($"Expected: {expectedExpr} ({expected}), Actual: {actualExpr} ({actual})");
    }
}

user.Name.ShouldEqual("Alice");
// If fails: "Expected: \"Alice\" (Alice), Actual: user.Name (Bob)"
```

Better assertion messages with no extra typing.

---

## Inside an interpolated string

```csharp
public static void Log(string msg, [CallerMemberName] string? caller = null) {
    Console.WriteLine($"[{caller}] {msg}");
}
```

The injected `caller` is just a string — interpolate it however you like.

---

## With `params` and other modifiers

`[CallerMemberName]` and friends only work on **optional parameters** (must have a default value). Not on:
- Required parameters (no default).
- `params` parameters.
- `ref` / `out` parameters.

```csharp
// ✗ — params can't have CallerMemberName
public void M([CallerMemberName] params string[]? caller) { }
```

For most uses, you want exactly one optional `string? param = null`.

---

## Internals — what the compiler emits

```csharp
public void M() {
    Log("hi");
}

public static void Log(string msg, [CallerMemberName] string? caller = null) { }
```

Compiles to:

```csharp
public void M() {
    Log("hi", "M");   // compiler hardcodes "M"
}
```

The injection happens at compile time. The default value `null` is replaced with the appropriate literal. No runtime cost; no reflection.

For `[CallerArgumentExpression]`:

```csharp
ThrowIfNull(user.Address);
// becomes:
ThrowIfNull(user.Address, "user.Address");
```

The argument's source text is captured (whitespace-normalized).

---

## Mid-string captures

```csharp
ThrowIfNull(user?.Address?.Street);
// captures: "user?.Address?.Street"
```

The exact expression text, including nullable operators, member chains, method calls.

For longer expressions, the captured string can get unwieldy. Still helpful for debugging.

---

## Caveats

### `[CallerMemberName]` in property setters

```csharp
public int Count {
    get => _count;
    set {
        _count = value;
        OnPropertyChanged();   // caller = "Count"
    }
}

private void OnPropertyChanged([CallerMemberName] string? name = null) { ... }
```

Inside a property setter, the "caller" is the property name. Used heavily in MVVM.

### `[CallerMemberName]` in constructors / events

In a constructor: caller is `.ctor` (or `.cctor` for static ctor) — usually not useful.
In an event accessor: caller is the event name.
In an indexer setter: caller is `this[]` (technically `Item`).

Usually you want it from regular methods or property setters.

### `[CallerFilePath]` leaks paths

In published binaries, the absolute path of the developer's machine is embedded. To clean up:

```xml
<PathMap>C:\src\MyApp=/MyApp</PathMap>
```

Normalizes paths in metadata. The injected values use the mapped form.

For privacy / security: be aware that file paths can be visible in stack traces.

### `[CallerArgumentExpression]` and complex expressions

```csharp
ThrowIfNull(GetUser(123).Account.Balance);
// captures: "GetUser(123).Account.Balance"
```

Long expressions are preserved literally. For very complex calls, the captured string is what fits — line breaks within an argument might be condensed.

---

## Common patterns

### Property change notification (MVVM)

The canonical use:

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
    public string Name { get => _name; set => SetField(ref _name, value); }
}
```

`SetField` captures "Name" automatically. Clean.

### Diagnostic helper

```csharp
[Conditional("DEBUG")]
public static void Trace(
    string msg,
    [CallerMemberName] string? caller = null,
    [CallerLineNumber] int line = 0)
{
    Debug.WriteLine($"[{caller}:{line}] {msg}");
}

// In a method
Trace("entered");   // prints in DEBUG only: [MyMethod:42] entered
```

`[Conditional("DEBUG")]` makes the call vanish in Release. The injected names happen at compile time, so even the compile-time work is gone.

### Validation framework

```csharp
public static void Validate(
    bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expr = null,
    [CallerFilePath] string? file = null,
    [CallerLineNumber] int line = 0)
{
    if (!condition) throw new InvalidOperationException(
        $"Validation '{expr}' failed at {Path.GetFileName(file)}:{line}");
}

Validate(user.Age > 0);
// On failure: "Validation 'user.Age > 0' failed at MyService.cs:42"
```

Self-documenting validation.

---

## Common bugs

### Forgetting to make the parameter optional

```csharp
public static void Log(string msg, [CallerMemberName] string caller) {   // ⚠ — not optional
    // ...
}
```

CallerMemberName only injects when the parameter has a default value (so the user can omit it). Without `= null`, callers must pass it explicitly — defeating the purpose.

### Using on properties

```csharp
public string CallerName { [CallerMemberName] get; } = "";   // ⚠ — doesn't work
```

Caller info attributes work on method parameters, not on auto-property accessors. Use a method.

### Path leakage in error messages

```csharp
catch (Exception ex) {
    return $"Error in {ex.StackTrace}";   // may include absolute paths
}
```

Stack traces include `[CallerFilePath]`-style info from the build. For user-facing errors, sanitize.

---

## Performance

Zero runtime cost. The attribute values are injected at compile time. No reflection, no string lookups.

The captured strings (member names, file paths, expressions) are baked into the IL as literals. Same as if you'd hand-coded the values.

---

## When to use

✓ Logging helpers — track caller automatically.
✓ MVVM property notification — eliminate magic strings.
✓ Guard / validation methods — `[CallerArgumentExpression]` for free expression text.
✓ Test framework helpers — better assertion messages.

✗ Anywhere you want runtime control (the values are compile-time only).
✗ For very-high-volume logging (the embedded paths add small string-table overhead).

---

## Summary

- Caller info attributes inject call-site info at compile time into optional parameters.
- `[CallerMemberName]` — calling method/property name.
- `[CallerFilePath]` — source file path.
- `[CallerLineNumber]` — line number.
- `[CallerArgumentExpression("name")]` — source text of the named argument (C# 10+).
- Zero runtime cost.
- Heavy use in MVVM, modern guard methods, diagnostic helpers.

→ Next: [13-CheckedOperators.md](13-CheckedOperators.md)
