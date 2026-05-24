# C# 9 (.NET 5, November 2020)

> Records, init-only setters, top-level statements, target-typed `new()`, source generators GA, function pointers, `with` expressions. C# 9 made data types easy.

---

## Records

The headline feature. Reference type with value-equality semantics:

```csharp
public record Person(string Name, int Age);

var a = new Person("Alice", 30);
var b = new Person("Alice", 30);
a == b;                  // true — value equality
a.Equals(b);             // true
a.GetHashCode() == b.GetHashCode();  // true

var older = a with { Age = 31 };   // non-destructive mutation
```

Compiler synthesizes: properties, constructor, Deconstruct, Equals, GetHashCode, ToString, ==/!= operators, copy constructor, `<Clone>$` for `with`.

Records became the default choice for DTOs, events, commands, immutable data containers. See [Chapter 03 §03](../03-TypeSystem/03-Records.md) and [Chapter 10 §03](../10-AdvancedLanguage/03-RecordsDeep.md).

---

## Init-only setters

```csharp
public class Person {
    public string Name { get; init; } = "";   // settable only at construction
    public int Age { get; init; }
}

var p = new Person { Name = "Alice", Age = 30 };   // OK
p.Name = "Bob";   // ❌ — init-only after construction
```

The compromise between `set` (always mutable) and `get-only` (only via constructor). Records use init-only properties by default for their positional parameters.

---

## Top-level statements

```csharp
// Program.cs
using System;
Console.WriteLine("Hello, World!");
```

The whole program. No class, no Main. Compiler synthesizes.

Became the default for `dotnet new console` since .NET 6. See [Chapter 10 §06](../10-AdvancedLanguage/06-TopLevelStatements.md).

---

## Target-typed `new()`

```csharp
List<int> nums = new();
Dictionary<string, User> users = new() { ["Alice"] = alice };

public class C {
    private readonly List<int> _items = new();   // type inferred from field
}
```

Skip the type on the right when the context makes it obvious.

---

## `with` expressions

```csharp
var p = new Person("Alice", 30);
var older = p with { Age = 31 };           // new instance with Age replaced
var renamed = p with { Name = "Alicia" };   // new instance with Name replaced
```

Records primary use. C# 10 extended `with` to anonymous types; C# 12 to structs (when the struct is a record struct or has the right shape).

Internally: compiler calls a synthesized `<Clone>$()` method that uses a copy constructor + applies the property assignments.

---

## Pattern matching enhancements

### Relational patterns

```csharp
return n switch {
    < 0 => "negative",
    0 => "zero",
    > 0 and < 100 => "small positive",
    >= 100 => "big"
};
```

`<`, `>`, `<=`, `>=` against constants.

### Logical patterns

```csharp
if (n is > 0 and < 100) { ... }
if (s is "yes" or "y") { ... }
if (s is not null) { ... }
```

`and`, `or`, `not` combine patterns.

### Type patterns without explicit binding

```csharp
return obj switch {
    Person => "person",          // no binding needed
    int n when n > 0 => "positive int",
    _ => "other"
};
```

Both `Person p` and just `Person` work.

---

## Source generators (GA)

C# 9 made source generators generally available. Roslyn compilers run user-supplied generators that emit additional C# during compilation:

```csharp
[Generator]
public class MyGenerator : IIncrementalGenerator {
    public void Initialize(IncrementalGeneratorInitializationContext context) {
        // Inspect syntax, emit source files.
    }
}
```

Used by:
- `[LoggerMessage]` — generates fast logging methods.
- `[GeneratedRegex]` — compiles regex at build time.
- `System.Text.Json` source generator — generates serialization code.
- ASP.NET Core Minimal APIs route generators.

Eliminates runtime reflection in many scenarios. Faster, AOT-friendly. See [Chapter 12 §05](../12-Reflection/05-SourceGenerators.md).

---

## Function pointers (unsafe)

```csharp
unsafe {
    delegate*<int, int> sq = &Square;
    int result = sq(5);

    int Square(int x) => x * x;
}
```

Function pointers WITHOUT delegate overhead. Used for high-performance interop and library code. Niche; most code uses `Func<>` / `Action<>`.

---

## Improved interpolated string handlers (foundation)

C# 9 set up the infrastructure that became fully exposed in C# 10 (`[InterpolatedStringHandler]`). Started to optimize common cases internally.

---

## Module initializers

```csharp
[System.Runtime.CompilerServices.ModuleInitializer]
internal static void Init() {
    Console.WriteLine("module loaded");
}
```

Runs once when the assembly is first loaded. Used for type registration, telemetry init, custom marshalling setup. Source generators heavily use module initializers for "register at startup" patterns.

---

## Native size integers (`nint` / `nuint`)

```csharp
nint x = 1024;   // pointer-sized integer
nuint y = 4096;  // unsigned
```

Same width as the platform pointer (4 bytes on 32-bit, 8 on 64-bit). Used for interop and `Span<T>` index arithmetic.

---

## Pattern matching: `not`

```csharp
if (obj is not null) { ... }
if (s is not "" and not null) { ... }
```

Negation pattern. Often more readable than `!`.

---

## Static lambdas

```csharp
Func<int, int> sq = static x => x * x;   // can't capture locals
```

Forbids the lambda from capturing — guaranteed no closure allocation. Used in hot paths.

---

## Pattern matching with `var`

```csharp
return obj switch {
    int i and var x when ExpensiveCheck(x) => x,   // bind to var for `when`
    _ => -1
};
```

`var x` always matches; binds the value. Useful in switch arms for letting the `when` clause use a name.

---

## Records: positional vs nominal

```csharp
// Positional — terse
public record Person(string Name, int Age);

// Nominal — more flexible
public record Config {
    public string Host { get; init; } = "localhost";
    public int Port { get; init; } = 80;
}
```

Both compile to similar IL. Positional is shorter; nominal supports defaults and validation more easily.

---

## Modules initializer + generators = compile-time runtime

The combination of source generators + module initializers enabled **compile-time** features that previously required runtime reflection:
- Serializers register types at startup.
- DI containers can pre-wire bindings.
- Validation rules are compiled, not interpreted.

This shift toward "compile-time work" is one of the biggest perf wins in modern .NET.

---

## Performance refinements

C# 9 brought:
- Smaller async state machine boxes (less GC pressure).
- Better inlining decisions for small methods.
- `Span<T>` improvements.

---

## Summary of C# 9

**Big wins**:
- Records — DTOs without boilerplate.
- `init` setters — immutable-after-construction.
- Top-level statements — clean entry points.
- Target-typed `new()` — less type repetition.
- Source generators GA — compile-time work.

**Smaller wins**:
- Relational and logical patterns.
- `not` pattern.
- Static lambdas.
- nint/nuint.
- Function pointers.

C# 9 + .NET 5 was the "modernization" inflection. The language felt much more concise; runtime got significantly faster; the toolchain became more powerful.

→ Next: [03-CSharp10.md](03-CSharp10.md)
