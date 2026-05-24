# C# 10 (.NET 6 LTS, November 2021)

> Global usings, file-scoped namespaces, record struct, parameterless struct constructors, extended property patterns, lambda improvements. C# 10 polished the developer experience.

---

## Global usings

```csharp
// In any .cs file:
global using System;
global using System.Linq;
global using System.Collections.Generic;
```

Or implicit (set in csproj):

```xml
<PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

SDK auto-includes namespace defaults per project type (Console, Web, Worker). See [Chapter 10 §07](../10-AdvancedLanguage/07-GlobalAndImplicitUsings.md).

After C# 10, most files don't need `using` directives — the SDK + globals cover the common namespaces.

---

## File-scoped namespaces

```csharp
// Before
namespace MyApp {
    public class Service { ... }
}

// C# 10
namespace MyApp;

public class Service { ... }
```

`namespace X;` (with semicolon) applies to the whole file. One less indentation level. Combined with implicit usings: minimal-boilerplate file.

The compiler accepts only one such declaration per file.

---

## Record struct

```csharp
public record struct Point(int X, int Y);
public readonly record struct Coord(double Lat, double Lng);
```

Records but with value-type semantics:
- Stack-allocated (or inline in containing types).
- Value equality + ToString + Deconstruct synthesized (same as record class).
- No inheritance (structs are sealed).
- Cheaper allocation for small data.

For small immutable value types (Point, Money, Vector), `readonly record struct` is the modern default. See [Chapter 03 §02](../03-TypeSystem/02-Structs.md).

---

## Parameterless struct constructors

```csharp
public struct Counter {
    public int Value;
    public Counter() {
        Value = 1;   // default constructor (pre-C# 10 was forbidden!)
    }
}
```

Pre-C# 10: structs **couldn't** have parameterless constructors (the compiler insisted on the all-zeros default). C# 10 lifted the restriction.

**Big caveat**: `default(Counter)` still gives the all-zeros instance (Value = 0), NOT the constructor's result. Only `new Counter()` calls the user-defined constructor. Worth understanding:

```csharp
Counter c1 = new();     // calls user ctor — Value = 1
Counter c2 = default;    // all-zeros — Value = 0
```

Same for field initializers:

```csharp
public struct Counter {
    public int Value = 5;   // C# 10+: field initializer in struct
    public Counter() {}      // explicit ctor required if you have field init
}

Counter c1 = new();      // Value = 5
Counter c2 = default;     // Value = 0
```

The `default` path bypasses field initializers and user constructors. Always.

For most code: just use classes if you need guaranteed initialization. Structs are for pure value-data.

---

## Extended property patterns

```csharp
public record Address(string City, string State);
public record Person(string Name, Address Address);

if (p is { Address.City: "Springfield" }) { ... }   // nested via dots
```

Pre-C# 10:
```csharp
if (p is { Address: { City: "Springfield" } }) { ... }   // nested braces
```

Shorter dot syntax for nested property access in patterns. See [Chapter 10 §02](../10-AdvancedLanguage/02-PatternMatchingDeep.md).

---

## Lambda improvements

### Inferred return type

```csharp
var f = (int x) => x * x;   // Func<int, int>
```

Earlier C# inferred input types from context but couldn't combine them with no-context. Now full inference works.

### Attributes on lambdas

```csharp
Func<int, int> sq = [SomeAttribute] (int x) => x * x;
```

Useful when the lambda's metadata matters (e.g., serializers reading attributes).

### Explicit return type

```csharp
var f = int (string s) => int.Parse(s);   // explicit return type
```

For ambiguous cases.

### Natural function type

```csharp
var inc = (int x) => x + 1;
// inc has type System.Func<int, int>
```

Lambdas with full type info now have a "natural type." Used by the compiler for overload resolution and generic inference.

---

## CallerArgumentExpression

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression(nameof(argument))] string? paramName = null)
{
    if (argument is null) throw new ArgumentNullException(paramName);
}

ThrowIfNull(user.Address);
// paramName captures "user.Address" automatically
```

Capture the source text of an argument as a string. Massive win for modern guard methods. See [Chapter 10 §12](../10-AdvancedLanguage/12-CallerInfoAttributes.md).

---

## Interpolated string handler hooks

```csharp
[InterpolatedStringHandler]
public ref struct MyHandler { ... }

public void Log(bool enabled, [InterpolatedStringHandlerArgument("enabled")] ref MyHandler msg) { ... }

Log(false, $"expensive: {ComputeBig()}");   // ComputeBig NOT called when enabled=false
```

Library-side feature for high-throughput logging / asserting. See [Chapter 10 §11](../10-AdvancedLanguage/11-InterpolatedStringHandlers.md).

The BCL adopted this for `Debug.Assert`, `ArgumentException.ThrowIf*`, `ILogger` extension methods.

---

## `record` and `class` keywords are now interchangeable in context

```csharp
public record class Person(string Name);   // explicit
public record Person(string Name);          // implicit class
public record struct Coord(int X, int Y);   // explicit struct
```

`record` = `record class`. The `class` keyword can be added for clarity.

---

## Sealed `ToString` on records

```csharp
public sealed record Person(string Name) {
    public sealed override string ToString() => $"Person:{Name}";
}
```

Pre-C# 10, record's auto-synthesized `ToString` was virtual; overriding in a sealed record didn't matter much. C# 10 made the synthesized `ToString` overridable as `sealed` to lock down the output format.

---

## Mix declarations and `var` in deconstruction

```csharp
public record Point(int X, int Y);
var p = new Point(3, 4);
(int x, var y) = p;   // mix: x is explicit int, y is inferred
```

Pre-C# 10, deconstruction required all-`var` or all-types. Now mix.

---

## Better definite assignment

```csharp
bool succeeded = false;
string? result;
if (TryGet(out result)) succeeded = true;
if (succeeded) Console.WriteLine(result.Length);   // C# 10: knows result is set
```

The compiler tracks definite assignment across more code paths. Previously you'd need a `!` or default value. Now the analyzer recognizes the pattern.

---

## Performance refinements

C# 10 + .NET 6 brought:
- Faster string formatting via DefaultInterpolatedStringHandler.
- Improved Span<T> intrinsics.
- Smaller async state machine allocations.
- More aggressive devirtualization.

For most apps, just upgrading to .NET 6 LTS yielded measurable perf gains with no code changes.

---

## .NET 6 LTS context

.NET 6 was the first LTS release of "modern .NET." Long-term support meant 3 years of patches. Many production apps still run on .NET 6 (support ends late 2024).

Combined with C# 10's polish, .NET 6 was the production-ready baseline for new apps from 2022 onward.

---

## Adoption tips for C# 10

If you're modernizing a C# 9 (or earlier) codebase:

1. Enable `<ImplicitUsings>enable</ImplicitUsings>` + `<Nullable>enable</Nullable>`.
2. Convert `namespace X { ... }` blocks to `namespace X;`.
3. Adopt `record struct` for small value-equal types you currently write as classes or structs.
4. Modernize guard methods with `CallerArgumentExpression`.
5. Use nested property patterns with `.` for cleaner switch expressions.

Each change is small; the cumulative effect is significantly less boilerplate.

---

## Summary of C# 10

**Big wins**:
- Global / implicit usings — most files become using-free.
- File-scoped namespace — one less indent.
- Record struct — value-type records.
- CallerArgumentExpression — modern guard methods.
- Extended property patterns — nested via dots.

**Smaller wins**:
- Parameterless struct constructors (with caveats).
- Lambda inference improvements.
- Interpolated string handlers (foundation).

C# 10 polished the language. Combined with .NET 6 LTS, this was the era of "less ceremony" — fewer lines, more meaning per line.

→ Next: [04-CSharp11.md](04-CSharp11.md)
