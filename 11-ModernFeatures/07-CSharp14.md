# C# 14 (.NET 10, November 2025)

> The current LTS. `field` keyword, extension members, null-conditional assignment `?.=`, first-class span conversions, partial constructors/events, unbound generics in `nameof`, file-based apps. The biggest C# release since C# 9.

---

## The `field` keyword — backing field without ceremony

```csharp
public class Person {
    public string Name {
        get;
        set => field = value?.Trim() ?? "";   // C# 14 — refer to compiler-generated backing field
    }

    public int Age {
        get => field;
        set {
            if (value < 0) throw new ArgumentOutOfRangeException(nameof(value));
            field = value;
        }
    }
}
```

Before C# 14, if you wanted custom getter/setter logic, you had to declare an explicit private field:

```csharp
// Pre-C# 14
private string _name = "";
public string Name {
    get => _name;
    set => _name = value?.Trim() ?? "";
}
```

C# 14's `field` keyword is a **contextual keyword** that refers to the compiler-generated backing field inside an accessor. The compiler:
- Generates the backing field automatically (private, mangled name).
- Lets you read/write it via `field`.
- Falls back to auto-implemented behavior in any accessor that doesn't reference `field`.

**Hybrid auto-properties** — you can mix custom and auto:

```csharp
public string Name {
    get;                                       // auto getter
    set => field = value?.Trim() ?? "";        // custom setter writes to backing field
}
```

### Breaking change risk

`field` was a common variable name. In C# 14, **inside a property accessor**, `field` resolves to the backing field. If you had a local variable or parameter named `field`, you must rename or use `@field`.

The compiler emits a warning for ambiguous use.

### Internals

The compiler emits:
```csharp
private string <Name>k__BackingField;
public string Name {
    get => <Name>k__BackingField;
    set => <Name>k__BackingField = value?.Trim() ?? "";
}
```

Same as before — just hidden naming.

See [Chapter 02 §03](../02-OOP/03-Properties.md).

---

## Extension members — properties, operators, statics

C# 3 introduced extension methods. C# 14 extends "extensions" to **properties**, **indexers**, **operators**, and **static members**.

```csharp
public static class StringExtensions {
    extension(string s) {                                  // new syntax
        public bool IsEmpty => s.Length == 0;              // extension property
        public char this[Index i] => s[i.GetOffset(s.Length)];  // extension indexer
        public static string Empty => string.Empty;        // extension static
        public static string operator +(string a, int n) => a + n.ToString();  // extension operator
    }
}

// Usage
string s = "hello";
bool empty = s.IsEmpty;       // extension property
char last = s[^1];             // extension indexer (overload)
string e = string.Empty;       // resolves to standard Empty (or extension's, if disambiguated)
```

The `extension(type)` block scopes all members as extensions to that type.

### Why this matters

Before C# 14, you could only add **methods**:
```csharp
public static bool IsEmpty(this string s) => s.Length == 0;
// Usage: s.IsEmpty()   // ⚠ method call, not property
```

Now `s.IsEmpty` reads like a real property. This makes LINQ-style fluent code more natural.

### Operators

```csharp
public static class VectorExtensions {
    extension(Vector v) {
        public static Vector operator +(Vector a, Vector b) => new(a.X + b.X, a.Y + b.Y);
    }
}
```

Useful for adding operator support to existing types (e.g., adding `+` to library types that don't define it).

### Limitations

- Extension members participate in normal overload resolution.
- Cannot access private state of the extended type.
- Cannot override existing members (real members win).

See [Chapter 02 §08](../02-OOP/08-Interfaces.md) for context.

---

## Null-conditional assignment `?.=` and `?[..] =`

```csharp
person?.Name = "Alice";        // C# 14 — assign only if person is non-null
list?[0] = 42;                 // C# 14 — assign only if list is non-null
dict?["key"] = value;          // C# 14
```

Pre-C# 14:
```csharp
if (person is not null) person.Name = "Alice";   // 3 keywords
```

C# 14 makes this a single expression. Equivalent to:
```csharp
_ = person is not null ? person.Name = "Alice" : default;
```

The RHS is evaluated only if the LHS target is non-null. Useful for optional updates.

### Compound forms

```csharp
counter?.Count += 1;           // increment only if counter is non-null
flags?.Mask |= 0x10;           // bitwise update on possibly-null target
```

Each of `+=`, `-=`, `*=`, etc. work with `?.`.

### Pitfall

```csharp
list?[GetIndex()] = value;
```

If `list` is null, `GetIndex()` is **not** called. The whole right-hand expression is skipped. Avoid side effects in the index expression if you care about ordering.

---

## `params ReadOnlySpan<T>` — finally zero-alloc params

Already discussed in [§06 (C# 13)](06-CSharp13.md). C# 14 polished this further:
- Implicit conversions from arrays / collection expressions to `params ReadOnlySpan<T>` are smarter.
- Overload resolution prefers `ReadOnlySpan` over array when both exist (avoiding allocation).

```csharp
void Log(params ReadOnlySpan<object> args) { ... }

Log("a", 1, true);   // stack-allocated span — zero heap alloc
```

The BCL added many overloads. `string.Concat`, `StringBuilder.Append`, etc., now have `ReadOnlySpan` overloads.

---

## First-class `Span<T>` conversions

```csharp
int[] arr = [1, 2, 3];
ReadOnlySpan<int> span = arr;            // implicit
Span<int> writeSpan = arr;               // implicit

void Process(ReadOnlySpan<int> data) { }
Process([1, 2, 3]);                       // collection expression → ReadOnlySpan
Process(arr);                             // array → ReadOnlySpan
Process(new[] { 1, 2, 3 });               // array → ReadOnlySpan
```

C# 14 made `Span<T>` / `ReadOnlySpan<T>` first-class in the type system:
- Implicit conversions from arrays.
- Implicit conversions from `string` to `ReadOnlySpan<char>`.
- Collection expressions target spans.

Generic methods can take spans as easily as arrays:

```csharp
T First<T>(ReadOnlySpan<T> data) => data[0];

First([1, 2, 3]);                  // works
First(new[] { 1, 2, 3 });          // works
First("hello");                    // works — string → ReadOnlySpan<char>
```

A huge ergonomics win for span-based APIs.

---

## Unbound generics in `nameof`

```csharp
string name = nameof(List<>);              // C# 14: "List"
string n2 = nameof(Dictionary<,>);         // "Dictionary"
```

Pre-C# 14, you had to write `nameof(List<int>)` even when you didn't care about the type argument.

Helpful in source generators and reflection code that produces type names.

---

## Partial constructors and events

Already had partial methods, classes, properties. C# 14 adds:

```csharp
public partial class Person {
    public partial Person(string name);            // partial declaration

    public partial event EventHandler<string>? NameChanged;
}

public partial class Person {
    public partial Person(string name) {           // partial implementation
        Name = name;
    }

    public partial event EventHandler<string>? NameChanged;  // generated by SG
}
```

Used by source generators to emit constructors / events while keeping the user-facing class declaration clean.

E.g., the regex source generator can emit a partial constructor that initializes regex state.

See [Chapter 02 §11](../02-OOP/11-NestedAndPartial.md).

---

## User-defined compound assignment operators

C# 14 lets types define `+=`, `-=`, etc., directly, instead of relying on the compiler to expand them via `+`.

```csharp
public class Counter {
    public int Value { get; private set; }
    public static Counter operator +=(Counter c, int n) {
        c.Value += n;
        return c;
    }
}

Counter c = new();
c += 5;          // direct call to operator+=
```

For mutable reference types, this avoids the "create new instance" pattern when you just want to modify in place. Especially useful for big-number / matrix types where allocation is expensive.

For value types and immutable types, stick with `+` (compiler synthesizes `+=`).

See [Chapter 10 §13](../10-AdvancedLanguage/13-CheckedOperators.md).

---

## File-based apps — `dotnet run app.cs`

```bash
dotnet run app.cs
```

No `.csproj`. No `.sln`. Just a `.cs` file. The dotnet CLI:
1. Synthesizes an implicit project.
2. Restores any `#:package` references.
3. Compiles and runs.

```csharp
// hello.cs
#:package Newtonsoft.Json
#:sdk Microsoft.NET.Sdk
#:property TargetFramework=net10.0

using Newtonsoft.Json;

var obj = new { Name = "Alice" };
Console.WriteLine(JsonConvert.SerializeObject(obj));
```

Pre-C# 14: you needed `csi` (C# interactive) or a script project.

For scripts, demos, and quick experiments, file-based apps replace the verbose project setup. Great for learning, teaching, and one-off automation.

### How it works

The SDK creates a transient project in temp space, compiles via Roslyn, runs the assembly. The `#:` directives are parsed before compilation and influence the synthetic project.

Cannot mix multiple `.cs` files yet — single-file only (as of .NET 10 GA). Multi-file file-based apps tracked for future versions.

See [Chapter 00 §04](../00-Introduction/04-FirstProgram.md).

---

## Improved overload resolution and generic inference

Quality-of-life:
- Better inference for collection expressions in generic contexts.
- Smarter narrowing for `Span<T>` / `T[]` / `IEnumerable<T>` overloads.
- Constructor inference for primary constructors.

Mostly invisible — your code just works more often without manual type arguments.

---

## Other small wins

### `ref readonly` parameters everywhere

`ref readonly` parameters were added in C# 12. C# 14 polished interop with `in`:

```csharp
void Process(ref readonly LargeStruct s) { ... }   // explicit pass-by-readonly-ref
```

Use when you want the API contract to say "I won't mutate this, but I want efficient pass-by-reference."

### Lambda parameter modifiers

```csharp
var f = (ref int x) => x++;   // C# 14: ref/in/out/scoped in lambdas without explicit delegate type
```

Pre-C# 14, you had to declare a delegate type to use ref-style lambdas. C# 14 infers.

### Implicit `params` in delegates

```csharp
Action<int[]> log = a => Console.WriteLine(string.Join(",", a));
log(1, 2, 3);    // ⚠ — pre-C# 14: error. C# 14: implicit array creation if delegate type supports it.
```

---

## Performance and runtime (.NET 10)

See [§08](08-DotNet10Runtime.md) for runtime details. Highlights:
- **DATAS GC**: dynamically adapts heap sizes per region.
- **Escape analysis & stack promotion**: many short-lived objects allocated on the stack.
- **Async box elision**: simple async methods don't allocate state machine boxes.
- **Inlining and devirtualization**: smarter JIT.
- **PGO improvements**: dynamic profile-guided optimization mature.

Application code typically sees 5-15% perf gains over .NET 9 with no source changes.

---

## Migration from C# 13 / .NET 9

Update `<TargetFramework>net10.0</TargetFramework>` and `<LangVersion>14</LangVersion>` (often `LangVersion` is implicit).

Most code compiles unchanged. Watch:
- Variables named `field` in properties — rename or use `@field`.
- Code that relied on overload resolution edge cases that C# 14 tightened.

---

## Adoption tips for C# 14

1. **`field` keyword** — modernize properties with custom logic.
2. **Extension properties** — clean up extension method APIs.
3. **`?.= ` assignment** — replace `if (x is not null) x.Y = ...` patterns.
4. **File-based apps** — replace script projects.
5. **`params ReadOnlySpan<T>`** — zero-alloc hot paths.
6. **Trust .NET 10 perf** — DATAS + escape analysis remove many manual optimizations.

---

## Summary of C# 14

**Big wins**:
- `field` keyword.
- Extension members (properties, operators, statics, indexers).
- `?.=` null-conditional assignment.
- Span-first generics.
- File-based apps.
- Partial constructors / events.

**Quality of life**:
- Unbound `nameof`.
- Lambda parameter modifiers.
- Improved overload resolution.
- User-defined compound operators.

**Runtime (.NET 10)**:
- DATAS GC.
- Escape analysis + stack promotion.
- Async box elision.
- Improved PGO.

C# 14 is the most polished language release since C# 9. Combined with .NET 10's runtime gains and LTS status, it's the recommended baseline for new projects.

→ Next: [08-DotNet10Runtime.md](08-DotNet10Runtime.md)
