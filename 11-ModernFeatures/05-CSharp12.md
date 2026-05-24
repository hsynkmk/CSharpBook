# C# 12 (.NET 8 LTS, November 2023)

> Primary constructors for classes/structs, collection expressions, alias any type, ref readonly parameters, default lambda parameters. C# 12 made everyday code shorter.

---

## Primary constructors (for classes and structs)

```csharp
public class Service(IRepository repo, ILogger<Service> log) {
    public async Task DoAsync() {
        log.LogInformation("starting");
        var data = await repo.LoadAsync();
        // ...
    }
}
```

Constructor parameters declared right next to the class name; available throughout the body. Cuts the dependency-injection boilerplate dramatically.

Record had this since C# 9; C# 12 brought it to regular classes and structs. See [Chapter 02 §12](../02-OOP/12-PrimaryConstructors.md).

Key behavior:
- Parameters are visible to all members (instance properties, methods, field initializers).
- Used parameters become private capture fields.
- If you need a public property, declare one: `public string Name { get; } = name;`.
- An additional constructor must call `this(...)` to chain to the primary.

```csharp
public class Person(string name, int age) {
    public string Name { get; } = name;     // expose
    public int Age { get; } = age;
    public string Greeting => $"Hi, {name}";  // use parameter directly
}
```

---

## Collection expressions

```csharp
int[] arr = [1, 2, 3];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];
ImmutableArray<int> imm = [1, 2, 3];

int[] combined = [..head, x, ..tail];   // spread
```

Unified syntax for collection construction. Compiler picks efficient backing (stackalloc for spans, etc.). See [Chapter 10 §08](../10-AdvancedLanguage/08-CollectionExpressions.md).

Custom types opt in via `[CollectionBuilder]`. Modern collections (FrozenSet, ImmutableArray, etc.) all support it.

---

## Alias any type

```csharp
using TimedEvent = (DateTime At, string Description);
using IntList = System.Collections.Generic.List<int>;
using Vector = (double X, double Y, double Z);

TimedEvent e = (DateTime.Now, "started");
IntList list = [1, 2, 3];
```

Pre-C# 12, `using X = Y;` worked only for namespaces and types — not tuples, arrays, pointers, etc. Now it works for any type expression.

Useful for clarifying complex tuple shapes or shortening long generic names.

---

## `ref readonly` parameters (explicit)

```csharp
public void Process(ref readonly BigStruct s) { /* read s, can't modify */ }

BigStruct b = ...;
Process(ref b);   // pass by readonly reference
```

C# 12 added explicit `ref readonly` parameters. Same effect as `in` but the caller's `ref` makes the by-ref intent explicit at the call site.

Use case: APIs where by-ref passing is part of the contract (signal "I'll read this, not copy it").

For most code, `in` is fine (implicit ref readonly).

---

## Default lambda parameters

```csharp
var greet = (string name = "World") => $"Hello, {name}";
Console.WriteLine(greet());        // "Hello, World"
Console.WriteLine(greet("Alice"));  // "Hello, Alice"
```

Lambdas can have default values, just like methods.

Use case: building delegates with optional behavior, factory methods.

---

## Lambda `params`

```csharp
var sum = (params int[] nums) => nums.Sum();
Console.WriteLine(sum(1, 2, 3));   // 6
```

Lambdas with `params`. Same as method params.

---

## Experimental attribute

```csharp
[Experimental("DOTNET100")]
public class NewApi { }

NewApi x = new();   // warns: DOTNET100 — Experimental
```

Library authors mark experimental APIs. Consumers see warnings. Configurable diagnostic ID per attribute.

---

## InlineArray (advanced)

```csharp
[InlineArray(10)]
public struct Buffer10 {
    private int _element0;
}

Buffer10 buf = default;
buf[0] = 42;
buf[5] = 100;
```

Compiler treats `Buffer10` as a 10-element inline array. Used by high-performance code that needs fixed-size inline buffers without `unsafe`.

Niche; library code mainly. Replaces some uses of `fixed` arrays.

---

## Interceptors (experimental)

```csharp
// Generated code can declare an interceptor
[InterceptsLocation("file.cs", line: 42, character: 10)]
public static void MyInterceptor() { /* ... */ }
```

Source generators can replace specific method calls with their own implementations. Used by ASP.NET Core's Request Delegate Generator (RDG) and similar AOT-friendly optimizations.

This is **library-side** machinery; application code doesn't write interceptors directly. Marked experimental in C# 12; may stabilize in later versions.

---

## Improved `nameof` on instance members

```csharp
public class C {
    public string Name { get; } = "";
    [Required(nameof(Name))]   // C# 12: nameof an instance member inside its own type's attribute
    public string ValidatedName { get; set; } = "";
}
```

`nameof` on instance members inside attributes — previously had quirks; C# 12 cleaned them up.

---

## Performance refinements

.NET 8 LTS brought:
- **Dynamic PGO** (Tiered Compilation Quick JIT v3): more aggressive profile-guided JIT.
- **FrozenDictionary / FrozenSet**: super-fast read-only hash collections.
- **Reduced allocations** in BCL hot paths.
- **Native AOT polished**: many real apps now AOT-compatible.

.NET 8 is the second LTS — many production services run on it through 2026. Solid baseline.

---

## Adoption tips for C# 12

1. **Primary constructors for DI**: turn `public Service(IRepository r) { _repo = r; }` into `public Service(IRepository repo) { ... use repo ... }`.
2. **Collection expressions everywhere**: `[1, 2, 3]` instead of `new int[] { 1, 2, 3 }` or `new List<int> { ... }`.
3. **Type aliases for tuples**: `using Point = (int X, int Y);` for repeating tuple shapes.
4. **FrozenDictionary** for read-only config tables.

The savings per file are small but they add up across a codebase.

---

## Adoption caveat: primary constructors

Primary constructor parameters are **mutable** by default (the compiler stores them as fields you can reassign in methods). For immutable-feeling values, declare explicit `get`-only properties:

```csharp
public class Demo(int x) {
    public int X { get; } = x;   // explicit readonly property — safer

    public void Reset() => x = 0;   // ⚠ mutates the synthesized field!
}
```

The synthesized field is technically mutable. For "I want these to be read-only," use explicit properties.

For records, primary constructor params become `init` properties automatically — different semantics.

---

## Summary of C# 12

**Big wins**:
- Primary constructors for classes / structs.
- Collection expressions.
- Alias any type.

**Smaller wins**:
- `ref readonly` parameters (explicit form).
- Default lambda parameters + `params` lambdas.
- Experimental attribute.

C# 12 + .NET 8 LTS is **the** production baseline as of late 2025 for many teams. Stable, fast, full of modern features.

→ Next: [06-CSharp13.md](06-CSharp13.md)
