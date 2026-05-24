# Attributes

## What they are

Attributes are **declarative metadata** attached to code elements (types, members, parameters, assemblies). At runtime, you query them via reflection. At compile time, some attributes alter behavior (e.g., `Obsolete`, `Conditional`, `CallerMemberName`).

```csharp
[Serializable]
public class MyData { ... }

[Obsolete("Use NewMethod instead", error: false)]
public void OldMethod() { ... }

[HttpGet("api/users/{id}")]
public IActionResult GetUser(int id) { ... }
```

They're how:
- ASP.NET maps routes (`[HttpGet]`, `[Route]`).
- xUnit discovers tests (`[Fact]`, `[Theory]`).
- Serializers know property names (`[JsonPropertyName]`).
- The compiler emits warnings (`[Obsolete]`).

---

## Defining a custom attribute

```csharp
[AttributeUsage(AttributeTargets.Class | AttributeTargets.Method, AllowMultiple = false, Inherited = true)]
public class AuthorAttribute : Attribute {
    public string Name { get; }
    public string? Email { get; init; }   // named optional

    public AuthorAttribute(string name) {
        Name = name;
    }
}
```

Rules:
- Class name ends with `Attribute` (convention; usage drops the suffix).
- Inherits from `System.Attribute`.
- Marked with `[AttributeUsage]` to declare where it applies.
- Constructor params → positional args; settable props → named args.

Apply:
```csharp
[Author("Alice")]
[Author("Bob", Email = "bob@example.com")]   // requires AllowMultiple = true
public class MyClass { ... }
```

---

## `AttributeUsage`

Controls how your attribute can be applied:

```csharp
[AttributeUsage(
    AttributeTargets.Class | AttributeTargets.Struct,
    AllowMultiple = false,    // default false
    Inherited = true          // default true
)]
```

- `AttributeTargets` — flags enum: `Class`, `Method`, `Field`, `Property`, `Parameter`, `ReturnValue`, `Assembly`, `Module`, `All`, etc.
- `AllowMultiple` — whether the same attribute can appear more than once on the target.
- `Inherited` — whether derived classes inherit the attribute (only for class-targets).

Without `[AttributeUsage]`, defaults are: any target, single, inherited.

---

## Reading attributes via reflection

```csharp
var attr = typeof(MyClass).GetCustomAttribute<AuthorAttribute>();
if (attr is not null) Console.WriteLine(attr.Name);

// Multiple
var all = typeof(MyClass).GetCustomAttributes<AuthorAttribute>();
foreach (var a in all) Console.WriteLine(a.Name);

// On a method
var mAttr = typeof(MyClass).GetMethod("DoWork")!.GetCustomAttribute<ObsoleteAttribute>();
```

`GetCustomAttribute<T>()` returns null if absent. `GetCustomAttributes<T>()` returns an `IEnumerable<T>`.

---

## Common BCL attributes

| Attribute | Purpose |
|---|---|
| `[Obsolete]` | Warn/error on use. Compiler-aware. |
| `[Serializable]` | Legacy binary serialization (avoid). |
| `[Conditional("DEBUG")]` | Method call elided if symbol not defined. |
| `[CallerMemberName]` | Compiler fills parameter with caller's name. |
| `[CallerFilePath]`, `[CallerLineNumber]` | Same idea for file/line. |
| `[CallerArgumentExpression("paramName")]` | Compiler fills with literal expression text (C# 10+). |
| `[Flags]` | Enums combined with bitwise ops. |
| `[DebuggerDisplay("...")]` | Debugger display format. |
| `[DebuggerStepThrough]` | Skip during F11. |
| `[DllImport("lib")]` | P/Invoke binding (legacy; prefer `LibraryImport` C# 11). |
| `[StructLayout]` | Control struct memory layout. |
| `[MethodImpl]` | JIT hints (`AggressiveInlining`, `NoInlining`). |
| `[Pure]` | Hint: method has no side effects. |
| `[ThreadStatic]` | Each thread has its own copy of the field. |

---

## `[Conditional]` — compile-time eliding

```csharp
[Conditional("DEBUG")]
public static void Log(string msg) {
    Console.WriteLine(msg);
}

// In release builds (DEBUG not defined), all calls to Log() are removed by the compiler.
Log("hi");
```

The **call sites** are removed, not the method body. Useful for logging that should disappear in release builds.

Args are evaluated only if the method survives. Side-effecting args may silently disappear:
```csharp
Log(Compute());   // ⚠ — Compute() not called in Release
```

---

## `[CallerMemberName]` and friends

```csharp
public void LogCall([CallerMemberName] string member = "", [CallerLineNumber] int line = 0) {
    Console.WriteLine($"{member} at line {line}");
}

void DoStuff() {
    LogCall();   // logs: DoStuff at line N
}
```

The compiler fills the optional parameter with caller-context info. Used heavily by `INotifyPropertyChanged` to avoid magic strings.

`[CallerArgumentExpression]` (C# 10+):
```csharp
public static void Verify<T>(T value, [CallerArgumentExpression(nameof(value))] string? expr = null) {
    if (value is null) throw new ArgumentNullException(expr);
}

Verify(user.Profile);   // expr = "user.Profile"
```

Powering `ArgumentNullException.ThrowIfNull` and modern guards.

See [Chapter 10 §12](../10-AdvancedLanguage/12-CallerInfoAttributes.md).

---

## `[MethodImpl(MethodImplOptions.AggressiveInlining)]`

Suggests the JIT inline this method. The JIT decides; not a guarantee.

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static int Square(int x) => x * x;
```

Useful for tiny hot methods. For ordinary code, trust the JIT — it inlines by default.

Other options:
- `NoInlining` — prevent inlining (debugging, ensuring stack traces).
- `AggressiveOptimization` — opt into Tier 1 immediately.

---

## Attribute as marker

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class TestableAttribute : Attribute {}
```

Just a marker — no data. Use to discover types:
```csharp
var testable = typeof(MyClass).GetCustomAttribute<TestableAttribute>() is not null;
```

Or:
```csharp
var allTestable = Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => t.IsDefined(typeof(TestableAttribute), false));
```

`IsDefined` is faster than `GetCustomAttribute` (doesn't instantiate the attribute).

---

## Internals — how attributes work

In the assembly metadata, attributes are stored as `CustomAttribute` entries on type/member rows. Each entry references:
- The attribute class (a `TypeRef`).
- The constructor used.
- The constructor args (encoded blob).
- Named parameters (property/field assignments).

At runtime, `GetCustomAttribute(s)`:
1. Reads the metadata.
2. Resolves the attribute class.
3. Calls its constructor with the encoded args.
4. Sets the named properties.
5. Returns the instance.

So each `GetCustomAttribute` call **instantiates a new attribute** (unless cached). Cache for hot paths.

---

## Common patterns

### Validation via attributes

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class RangeAttribute : Attribute {
    public int Min { get; }
    public int Max { get; }
    public RangeAttribute(int min, int max) { Min = min; Max = max; }
}

public class Item {
    [Range(0, 100)] public int Percent { get; set; }
}

public void Validate(object obj) {
    foreach (var pi in obj.GetType().GetProperties()) {
        var attr = pi.GetCustomAttribute<RangeAttribute>();
        if (attr is null) continue;
        var val = (int)pi.GetValue(obj)!;
        if (val < attr.Min || val > attr.Max)
            throw new ValidationException($"{pi.Name} out of range");
    }
}
```

The pattern behind `System.ComponentModel.DataAnnotations`.

### Auto-registration

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class HandlerAttribute : Attribute {
    public string Command { get; }
    public HandlerAttribute(string command) { Command = command; }
}

public void RegisterHandlers(IServiceCollection services) {
    var handlers = Assembly.GetExecutingAssembly()
        .GetTypes()
        .Where(t => t.IsDefined(typeof(HandlerAttribute), false));

    foreach (var t in handlers) {
        var attr = t.GetCustomAttribute<HandlerAttribute>()!;
        services.AddTransient(t);   // register by attribute metadata
        _commandMap[attr.Command] = t;
    }
}
```

---

## Common bugs

### Attribute parameter must be compile-time constant

```csharp
[Author(GetDefaultName())]   // ✗ — must be a constant
```

Attribute args are encoded into metadata at compile time. They must be `const`-able: primitives, strings, `typeof`, enums, or single-dimensional arrays of these.

### Forgetting `AllowMultiple` for repeating attributes

```csharp
[Author("Alice")]
[Author("Bob")]   // ✗ — only one allowed without AllowMultiple = true
public class X {}
```

### Querying via wrong target type

```csharp
var attr = typeof(MyClass).GetCustomAttribute<HttpGetAttribute>();
// ⚠ — HttpGet is on a method, not the class. attr is null.
```

Query the right `MemberInfo`.

### Per-call allocation

```csharp
foreach (var item in items) {
    var attr = item.GetType().GetCustomAttribute<MyAttr>();   // ⚠ — allocates per call
}
```

Cache the attribute lookup:

```csharp
private static readonly ConcurrentDictionary<Type, MyAttr?> _cache = new();
var attr = _cache.GetOrAdd(item.GetType(), t => t.GetCustomAttribute<MyAttr>());
```

---

## Performance

- `IsDefined(type, inherit)` — fastest. ~50 ns.
- `GetCustomAttribute<T>()` — instantiates the attribute. ~500 ns.
- `GetCustomAttributes(true)` — recursive. ~1-2 μs depending on inheritance depth.

For hot paths, cache results per `Type`/`MemberInfo`. Attribute lookups are otherwise repeated unnecessarily.

---

## AOT and trimming

Custom attributes are preserved by default (the trimmer keeps them for reflection scenarios). For NativeAOT, attribute constructors must not call AOT-incompatible code.

If your attribute is only used at compile time (e.g., by a source generator), mark it with `[Conditional]` so the runtime cost is zero:

```csharp
[Conditional("COMPILE_TIME_ONLY")]   // never defined
public class MyMarkerAttribute : Attribute {}
```

Or use `internal` and never query at runtime — the trimmer can remove it.

---

## When attributes are the right tool

- Declarative configuration that maps to code (routes, validation, serialization).
- One-time discovery at startup (DI registration, test discovery).
- Compiler-aware behaviors (`Obsolete`, `Conditional`, caller info).

When attributes are **wrong**:
- Logic that varies per call (use parameters).
- Performance-critical lookups (use code generation).
- State (attributes are constructed each time; use static fields if you need shared state).

---

## Summary

- Attributes attach declarative metadata to code elements.
- Defined as classes inheriting `Attribute`, marked with `[AttributeUsage]`.
- Args must be compile-time constants.
- Queried at runtime via reflection (`GetCustomAttribute`, `IsDefined`).
- Heavily used by compilers (Obsolete, Conditional, CallerXxx) and frameworks (ASP.NET, EF, xUnit).
- Cache lookups for hot paths.

→ Next: [04-DynamicAndExpando.md](04-DynamicAndExpando.md)
