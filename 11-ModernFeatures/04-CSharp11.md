# C# 11 (.NET 7, November 2022)

> Required members, raw string literals, generic math, list patterns, UTF-8 string literals, file-scoped types, static abstract members. C# 11 was the "power user" release.

---

## Required members

```csharp
public class Person {
    public required string Name { get; init; }
    public required int Age { get; init; }
}

new Person { Name = "Alice", Age = 30 };   // OK
new Person { Name = "Alice" };              // ❌ compile error — Age required
```

Compiler enforces that callers initialize the marked members. See [Chapter 10 §04](../10-AdvancedLanguage/04-RequiredMembers.md).

Combined with `init`, gives "set-once, must-set" properties without constructor boilerplate. Records benefit hugely — clean DTOs.

`[SetsRequiredMembers]` on constructors says "this ctor handles required members" — caller can skip object initializer.

---

## Raw string literals

```csharp
string json = """
    {
        "name": "Alice",
        "age": 30
    }
    """;

string regex = """\d{3}-\d{4}""";   // no \\d needed
```

`"""` delimiters; no escape interpretation. Multi-line with smart indent stripping. Interpolated versions: `$"""..."""`. See [Chapter 10 §09](../10-AdvancedLanguage/09-RawStringLiterals.md).

Eliminates escape-soup for JSON, regex, SQL, XML embedded in C# code.

---

## UTF-8 string literals

```csharp
ReadOnlySpan<byte> http = "HTTP/1.1 200 OK\r\n"u8;
```

`u8` suffix — produces a `ReadOnlySpan<byte>` of UTF-8 bytes. Bytes embedded in assembly; zero allocation. See [Chapter 10 §10](../10-AdvancedLanguage/10-UTF8StringLiterals.md).

Critical for byte-oriented hot paths (HTTP servers, JSON parsers, network protocols).

---

## List patterns

```csharp
int[] arr = { 1, 2, 3 };

if (arr is [1, 2, 3]) { /* exact */ }
if (arr is [1, .., 3]) { /* starts 1, ends 3 */ }
if (arr is [_, _, _]) { /* exactly 3 elements */ }
if (arr is [var first, .. var rest]) { /* split head + tail */ }
```

Pattern-match array-like structures by element. `..` matches zero or more. Works on arrays, lists, spans, anything with `Length`/`Count` + indexer. See [Chapter 10 §02](../10-AdvancedLanguage/02-PatternMatchingDeep.md).

---

## Generic math (static abstract members)

```csharp
using System.Numerics;

public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}

Sum(new[] { 1, 2, 3 });        // works for int
Sum(new[] { 1.5, 2.5 });        // works for double
Sum(new[] { 1m, 2m, 3m });      // works for decimal
```

Interfaces can declare `static abstract` members — methods, properties, operators. Generic algorithms can call them via type parameters.

`INumber<T>` requires `+`, `-`, `*`, `/`, comparison operators, `Zero`, `One`, conversion methods. Built-in types (int, long, double, decimal, BigInteger) all implement it.

See [Chapter 04 §05](../04-Generics/05-StaticAbstractMembers.md) and [Chapter 04 §06](../04-Generics/06-GenericMath.md).

---

## File-scoped types

```csharp
file class InternalHelper {
    public static int Compute(int n) => n * 2;
}
```

Class visible only within its source file. Used heavily by source generators. See [Chapter 10 §05](../10-AdvancedLanguage/05-FileScopedTypes.md).

---

## Auto-default struct

C# 11 ensures struct fields get initialized to default at construction even if you skip them:

```csharp
public struct Point {
    public int X;
    public int Y;
    public Point(int x) { X = x; /* Y not set */ }   // OK — Y defaults to 0
}
```

Pre-C# 11: error CS0171 — "Field must be fully assigned before control leaves the constructor." C# 11 auto-defaults missing fields.

---

## `nameof` for parameters

```csharp
public class C {
    [Foo(nameof(arg))]   // refer to parameter name in attribute
    public void M(int arg) { }
}
```

Pre-C# 11, `nameof(arg)` in an attribute wasn't allowed (arg not in scope). Now it is.

Useful for attributes that reference parameters (`[CallerArgumentExpression(nameof(arg))]`).

---

## `ref` field

```csharp
public ref struct Holder {
    public ref int Value;   // C# 11+ — ref field in ref struct
}
```

Lets ref structs hold ref fields. Used by `Span<T>` internally (in .NET 7+). Lifetime rules ensure safety.

For library code only — application code rarely needs this.

---

## Generic attributes

```csharp
public class TypeAttribute<T> : Attribute { }

[Type<string>]
public class C { }
```

Pre-C# 11, attributes couldn't be generic. Now they can. Used by source generators and frameworks that need type-parameterized metadata.

---

## Pattern matching: `Span<char>` matches `string` constants

```csharp
ReadOnlySpan<char> s = "yes".AsSpan();
if (s is "yes") { ... }   // C# 11+ — Span pattern-matches against constant string
```

Lets you use string patterns on spans without converting back to string. Useful for parsing.

---

## `Range` and `Index` as switch patterns (improved)

Slice patterns work cleanly with the existing Range/Index syntax in switches.

---

## newlines in interpolation

```csharp
var s = $"value: {
    someExpression
}";
```

The interpolated expression can span lines. Useful for complex expressions, especially with raw strings.

---

## Performance improvements (.NET 7)

.NET 7 brought:
- **PGO (Profile-Guided Optimization)** stable: JIT collects runtime profile + optimizes accordingly.
- **Faster JIT**: many micro-optimizations.
- **Smaller async**: state machine boxes shrunk.
- **AOT improvements**: Native AOT became fully usable for many apps.

Combined with C# 11's features, .NET 7 was a noticeable perf jump over .NET 6.

---

## Adoption tips for C# 11

1. Adopt `required` for DTO mandatory fields. Eliminate constructor parameter sprawl.
2. Use raw strings for embedded JSON / SQL / regex.
3. Mark UTF-8 literals (`"x"u8`) on hot byte paths.
4. Try generic math for utility numeric methods.
5. Add `[CallerArgumentExpression]` to your guard methods.

---

## Summary of C# 11

**Big wins**:
- Required members.
- Raw string literals.
- UTF-8 string literals.
- List patterns.
- Generic math (static abstract).
- File-scoped types.

**Smaller wins**:
- Auto-default struct fields.
- `nameof` for parameters.
- Generic attributes.
- ref fields (library-side).

C# 11 was the "performance / power" release. Heavy adoption in libraries; modern application code uses required, raw strings, and list patterns regularly.

→ Next: [05-CSharp12.md](05-CSharp12.md)
