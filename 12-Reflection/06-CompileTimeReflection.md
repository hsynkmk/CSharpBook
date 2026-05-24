# Compile-Time Reflection — `nameof`, `typeof`, caller info

## What it is

Some "reflection-like" needs are satisfied **entirely at compile time** with zero runtime cost. The compiler bakes the answer into the assembly as a constant or a direct reference. These are the cheapest forms of metaprogramming.

```csharp
nameof(Console.WriteLine)   // "WriteLine" — a compile-time string constant
typeof(int)                 // System.Type token, resolved cheaply
```

Prefer these over runtime reflection wherever the information is known at compile time.

---

## `nameof`

```csharp
string n = nameof(Customer.Email);   // "Email"
```

`nameof(x)` evaluates to the **simple name** of `x` as a compile-time string constant. No runtime work — it's literally a string literal in the IL.

### Why use it instead of a string literal

Refactoring safety. If you rename `Email`, `nameof(Customer.Email)` updates automatically (or fails to compile). A magic `"Email"` string silently breaks.

```csharp
// Magic string — breaks silently on rename
throw new ArgumentNullException("value");

// nameof — refactor-safe, compile-checked
throw new ArgumentNullException(nameof(value));
```

### Common uses

```csharp
// Argument validation
void Set(string name) {
    ArgumentException.ThrowIfNullOrEmpty(name, nameof(name));
}

// INotifyPropertyChanged
OnPropertyChanged(nameof(FullName));

// Logging member names
logger.LogError("Failed in {Method}", nameof(ProcessOrder));

// EF Core / config
builder.Property(nameof(User.Email));
```

### What `nameof` returns

Only the **last segment**:
```csharp
nameof(System.Console)            // "Console" — not "System.Console"
nameof(person.Address.City)       // "City"
nameof(List<int>.Count)           // "Count"
```

For the full name, you need `typeof(...).FullName` (runtime) or a literal.

### Unbound generics (C# 14)

```csharp
nameof(List<>)          // "List"
nameof(Dictionary<,>)   // "Dictionary"
```

Pre-C# 14 you had to write `nameof(List<int>)`. C# 14 allows the unbound form. See [Chapter 11 §07](../11-ModernFeatures/07-CSharp14.md).

---

## `typeof`

```csharp
Type t = typeof(string);
```

`typeof(T)` produces a `Type` object via the IL `ldtoken` + `GetTypeFromHandle` — very cheap (resolved from a metadata token, no name lookup).

### `typeof` vs `GetType()`

```csharp
object o = "hello";
Type a = typeof(string);   // compile-time — the declared/literal type
Type b = o.GetType();       // runtime — the actual instance type
```

- `typeof(T)` is resolved from a compile-time type. No instance needed.
- `obj.GetType()` reads the runtime type from the object header — requires an instance, returns the most-derived type.

```csharp
Animal a = new Dog();
typeof(Animal)    // Animal
a.GetType()       // Dog — runtime type
```

### Open generics

```csharp
typeof(List<>)             // open generic type definition
typeof(List<int>)          // closed
typeof(Dictionary<,>)      // two type params unspecified
```

Use `typeof(List<>)` with `MakeGenericType` for runtime construction.

### Performance

`typeof` is essentially free (token resolution). `GetType()` is a single field read (~1 ns). Both are far cheaper than name-based reflection (`Type.GetType("...")`).

---

## Caller info attributes

The compiler fills these optional parameters at the **call site** — zero runtime reflection.

```csharp
using System.Runtime.CompilerServices;

public void Log(
    string message,
    [CallerMemberName] string member = "",
    [CallerFilePath] string file = "",
    [CallerLineNumber] int line = 0)
{
    Console.WriteLine($"[{Path.GetFileName(file)}:{line}] {member}: {message}");
}

void DoWork() {
    Log("starting");   // compiler injects member="DoWork", file=..., line=...
}
```

| Attribute | Filled with |
|---|---|
| `[CallerMemberName]` | Name of the calling member |
| `[CallerFilePath]` | Full path of the source file |
| `[CallerLineNumber]` | Line number of the call |
| `[CallerArgumentExpression("p")]` | Source text of argument `p` (C# 10+) |

### `[CallerMemberName]` for INotifyPropertyChanged

```csharp
public class ViewModel : INotifyPropertyChanged {
    public event PropertyChangedEventHandler? PropertyChanged;

    private string _name = "";
    public string Name {
        get => _name;
        set { _name = value; OnChanged(); }   // no string needed
    }

    void OnChanged([CallerMemberName] string prop = "") =>
        PropertyChanged?.Invoke(this, new(prop));
}
```

`OnChanged()` from the `Name` setter automatically passes `"Name"`. Refactor-safe, no magic string, no reflection.

### `[CallerArgumentExpression]`

```csharp
public static void Require(bool condition,
    [CallerArgumentExpression(nameof(condition))] string? expr = null) {
    if (!condition) throw new InvalidOperationException($"Failed: {expr}");
}

Require(user.Age >= 18);
// throws: "Failed: user.Age >= 18"   ← the literal source text
```

The compiler captures the **textual expression** of the argument. Powers modern guard helpers:

```csharp
ArgumentNullException.ThrowIfNull(order);
// internally uses CallerArgumentExpression to name "order" in the exception
```

See [Chapter 10 §12](../10-AdvancedLanguage/12-CallerInfoAttributes.md).

---

## `sizeof`

```csharp
int s = sizeof(int);        // 4 — compile-time constant for primitives
int p = sizeof(Point);      // size of a struct (requires unsafe for custom structs)
```

For built-in types, `sizeof` is a compile-time constant. For custom structs, it requires `unsafe` context (the size is layout-dependent). `Unsafe.SizeOf<T>()` gives a managed alternative.

---

## `default` and `default(T)`

```csharp
int x = default;        // 0
string? s = default;    // null
T value = default!;     // type-appropriate default in generics
```

`default` is resolved at compile time — emits the appropriate zero/null. No runtime computation. See [Chapter 04 §07](../04-Generics/07-DefaultLiteralAndT.md).

---

## Comparison: compile-time vs runtime metaprogramming

| Need | Compile-time tool | Runtime equivalent | Speedup |
|---|---|---|---|
| Member name as string | `nameof(X)` | `MethodInfo.Name` | ∞ (free) |
| Type object | `typeof(T)` | `Type.GetType("...")` | ~1000× |
| Runtime type of instance | `obj.GetType()` | (same) | — |
| Caller's name | `[CallerMemberName]` | stack walking | ~1000× |
| Argument source text | `[CallerArgumentExpression]` | impossible at runtime | — |
| Size of type | `sizeof` | `Marshal.SizeOf` | much faster |
| Default value | `default` | `Activator.CreateInstance` | much faster |

**Rule of thumb**: if the information is known at compile time, use the compile-time tool. Save runtime reflection for genuinely dynamic scenarios (plugins, unknown types).

---

## Common bugs

### Using `nameof` expecting the full path

```csharp
logger.LogInfo(nameof(MyApp.Services.OrderService));  // "OrderService" only
```

For the namespace-qualified name, use `typeof(OrderService).FullName`.

### `GetType()` on a value type boxes

```csharp
int x = 5;
Type t = x.GetType();   // ⚠ — boxes x to call GetType
```

`GetType` is defined on `object`; calling it on a value type boxes. Use `typeof(int)` if you know the type statically.

### Overriding caller info defaults

```csharp
void Log(string msg, [CallerMemberName] string m = "") { ... }

Log("x", "explicit");   // ⚠ — passing the arg overrides the compiler-injected value
```

If you supply the optional argument explicitly, the compiler doesn't inject. Usually you want to omit it.

### `nameof` of a local before declaration

```csharp
Console.WriteLine(nameof(y));   // ✗ — y not in scope yet
int y = 5;
```

`nameof` still requires the symbol to be in scope.

---

## When to use

Always prefer compile-time reflection when the data is statically known:
- Validation messages → `nameof`.
- Property change notification → `[CallerMemberName]`.
- Guards → `[CallerArgumentExpression]`.
- Type tokens → `typeof`.

These cost nothing at runtime and survive refactoring. Reach for runtime reflection only when types/members are genuinely unknown until execution.

---

## Summary

- `nameof` → compile-time string of a symbol's simple name; refactor-safe.
- `typeof(T)` → cheap `Type` from a compile-time type; `obj.GetType()` → runtime type.
- Caller info attributes (`CallerMemberName`, `CallerFilePath`, `CallerLineNumber`, `CallerArgumentExpression`) → compiler-injected at call site, zero runtime cost.
- `sizeof`, `default` → compile-time constants.
- Always prefer these over runtime reflection when the answer is known at compile time.

→ Next: [07-PerformanceConcerns.md](07-PerformanceConcerns.md)
