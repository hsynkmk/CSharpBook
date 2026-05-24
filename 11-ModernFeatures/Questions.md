# Chapter 11 — Modern Features — Q & A

---

### Q1. What is the difference between `record` and `record struct`?

`record` (since C# 9) is a reference type with value-based `Equals`/`GetHashCode` synthesized from all primary-constructor and instance fields. `record struct` (C# 10) is a value type with the same synthesized members. Both support `with` expressions and deconstruction.

Use `record` for DTOs, immutable models, message-passing types. Use `record struct` when you want value semantics + low allocation overhead for small data (≤16 bytes).

---

### Q2. What does `init` mean?

`init` (C# 9) is an accessor like `set` that can only be called during object initialization (constructor, object initializer, `with` expression). After construction, the property is effectively read-only.

```csharp
public class Person {
    public string Name { get; init; }
}
var p = new Person { Name = "Alice" };  // OK
p.Name = "Bob";                          // ✗ — Name is init-only
```

Combined with `required` (C# 11), gives you immutable-from-the-outside types without ctor boilerplate.

---

### Q3. Why does C# 9 introduce `record` instead of just letting `class` synthesize Equals?

The compiler generates dozens of members for records: `Equals`, `GetHashCode`, `ToString`, `Deconstruct`, `<Clone>$`, `==`, `!=`, copy constructor, etc. These behaviors aren't appropriate for arbitrary classes — they're a contract that says "this is value-equal data."

A keyword (`record`) is clearer than tagging classes with an attribute. It also signals intent to readers: "this is data, not behavior."

---

### Q4. What's the difference between target-typed `new()` and `var`?

- `var x = new List<int>();` infers `x` as `List<int>` from the RHS.
- `List<int> x = new();` infers RHS from declared type.

Use `new()` when the type is already in the declaration (field, return type, parameter default). Use `var` when the type is verbose and only matters on the RHS.

```csharp
private Dictionary<string, List<int>> _data = new();  // clear — type on LHS
var users = service.GetUsers();                       // var — type clear from method name
```

---

### Q5. What does `required` do?

`required` (C# 11) means callers MUST initialize the property/field in an object initializer or constructor (with appropriate attributes). The compiler emits errors for missing initializations:

```csharp
public class Customer {
    public required string Email { get; init; }
}
var c = new Customer();             // ✗ — Email not initialized
var c2 = new Customer { Email = "x@y" }; // ✓
```

Replaces nullable workaround tricks (`= null!`).

---

### Q6. What is a raw string literal?

A multi-line string with no escaping for quotes, syntactically delimited by 3+ double quotes:

```csharp
string json = """
    { "name": "Alice", "age": 30 }
    """;
```

Indentation is stripped based on the closing `"""` line. Use 4+ quotes if your content needs 3 quotes.

Interpolated form: `$"""..."""` (interpolation holes still use `{}`).

---

### Q7. What does the `u8` suffix do?

```csharp
ReadOnlySpan<byte> json = "{\"name\":\"Alice\"}"u8;
```

`u8` (C# 11) tells the compiler to emit the string as UTF-8 bytes in the metadata, returning a `ReadOnlySpan<byte>`. Skips the runtime `Encoding.UTF8.GetBytes()` call.

Useful for JSON/HTTP header constants. Zero allocation per use.

---

### Q8. Primary constructors — what changed in C# 12?

C# 12 lets non-record classes and structs declare parameters in the type header:

```csharp
public class Service(IRepository repo, ILogger log) {
    public void Run() => log.LogInfo($"using {repo}");
}
```

Parameters are in scope for the entire class body (methods, properties, field initializers). They are **not** automatically exposed as properties (unlike records).

Behind the scenes: the compiler generates private capture fields for any parameter actually used in the body.

---

### Q9. What is a collection expression?

```csharp
int[] a = [1, 2, 3];
List<int> b = [1, 2, 3];
Span<int> c = [1, 2, 3];
ReadOnlySpan<int> d = [..a, 4, 5, ..b];   // spread
```

C# 12 added `[...]` syntax that compiles to the optimal initialization for the target type. Replaces `new[] { ... }`, `new List<int> { ... }`, `stackalloc int[] { ... }`, etc., with one uniform form.

Supports spread operator (`..`) to concatenate.

---

### Q10. `params ReadOnlySpan<T>` vs `params T[]`?

Both let callers pass variable args. `params T[]` allocates an array per call. `params ReadOnlySpan<T>` (C# 13) stack-allocates the span — zero heap allocation.

```csharp
void Log(params ReadOnlySpan<object> args) { ... }
Log("a", 1, true);   // no array allocation
```

Used heavily in modern BCL (StringBuilder, Console, etc.). Caveat: `ReadOnlySpan` can't be captured by lambdas or stored in fields.

---

### Q11. What is the `Lock` type?

C# 13 added `System.Threading.Lock` — a dedicated locking type:

```csharp
private readonly Lock _gate = new();
lock (_gate) { ... }
```

The compiler recognizes `Lock` and emits calls to `Enter()`/`Exit()` (faster than `Monitor.Enter` on object). Avoids the foot-gun of locking on a value type or arbitrary object.

---

### Q12. What does the `field` keyword in C# 14 do?

Inside a property accessor, `field` refers to the compiler-generated backing field:

```csharp
public string Name {
    get;
    set => field = value?.Trim() ?? "";
}
```

Replaces the manual `private string _name;` + accessor pattern. Lets you mix custom logic with auto-generated backing storage.

---

### Q13. Extension members in C# 14 — what's new beyond extension methods?

You can now write extension **properties**, **indexers**, **static members**, and **operators**:

```csharp
public static class StringExt {
    extension(string s) {
        public bool IsEmpty => s.Length == 0;
        public static string Empty => string.Empty;
    }
}

s.IsEmpty   // extension property, not method
```

Significant ergonomics win for fluent APIs.

---

### Q14. What is `?.=` ?

C# 14's null-conditional assignment:

```csharp
person?.Name = "Alice";   // assigns only if person is non-null
```

Equivalent to `if (person is not null) person.Name = "Alice";`. Also supports compound forms (`?.+=`, `?[..] =`).

---

### Q15. What is a file-based app?

C# 14 + .NET 10 lets you run a single `.cs` file via `dotnet run app.cs` — no `.csproj` needed. Directives `#:package`, `#:sdk`, `#:property` configure the implicit project.

```csharp
// hello.cs
#:package Newtonsoft.Json
Console.WriteLine("hi");
```

Great for scripts, demos, learning.

---

### Q16. What does DATAS GC do?

DATAS (Dynamically Adapting To Application Sizes) sizes the heap based on the app's actual working set rather than fixed machine-based defaults. Default in .NET 9+. Saves 20-40% memory for small/medium apps.

Disable with `<GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>` if you have throughput-critical large allocators.

---

### Q17. What is escape analysis in .NET 10?

The JIT proves that a reference-type allocation doesn't outlive the method, then stack-allocates instead of heap-allocates. Many small `new T[]` and `new SomeClass()` patterns are now zero-GC automatically.

Doesn't replace `Span<T>` / `ArrayPool` for predictable hot paths — but covers a lot of incidental allocations.

---

### Q18. Async box elision — what's elided?

When an async method completes synchronously (all awaits resolve immediately), .NET 10 doesn't box the state machine struct to the heap. Common with cached lookups returning `Task<T>`. Combined with `ValueTask<T>`, fast paths become alloc-free.

---

### Q19. What is dynamic PGO?

Profile-Guided Optimization driven by runtime profile data. The JIT instruments tier-0 code, observes hot paths, then recompiles them with optimizations (devirtualization, hot block reordering, etc.). Default in .NET 10 for all release configs.

Result: tighter code on hot paths than static optimizations could produce.

---

### Q20. C# 11 added list patterns — what's a use case?

Pattern matching on the structure of a list:

```csharp
return arr switch {
    [] => "empty",
    [var single] => $"one: {single}",
    [_, _, .. var rest] => $"more: {rest.Length}",
};
```

Useful for parsers, AST visitors, command dispatch. Reads cleaner than nested `if`/`switch` on `arr.Length`.

---

### Q21. What's the difference between C# version and .NET version?

- **C# version** (e.g., C# 14): language features the compiler understands.
- **.NET version** (e.g., .NET 10): runtime + BCL.

A given .NET version implies a default C# version (e.g., .NET 10 → C# 14). You can override with `<LangVersion>` in `.csproj`, but library features (like `Lock`) require the matching runtime.

---

### Q22. When should I adopt the latest C#/.NET version in a production project?

- **.NET 10 is LTS** (Nov 2028) — safe to adopt now.
- For libraries: target broadly (e.g., `netstandard2.0` for max compat, multi-target for new features).
- For apps: target latest LTS for security patches and new features.
- Skip STS (`.NET 9`, `.NET 11`) for production unless you need a specific feature.

---

### Q23. What's `nameof(List<>)` ?

C# 14 allows unbound generic type arguments in `nameof`:

```csharp
nameof(List<>)         // "List"
nameof(Dictionary<,>)  // "Dictionary"
```

Pre-C# 14, you had to pick concrete type args even when you didn't care. Useful in source generators.

---

### Q24. Partial constructors and events — when are they used?

Almost exclusively by source generators:

```csharp
public partial class Person {
    public partial Person(string name);
}
```

A regex source generator might emit the constructor body initializing regex state. Lets the user-facing class stay clean.

---

### Q25. User-defined compound assignment — why?

```csharp
public static Counter operator +=(Counter c, int n) { c.Value += n; return c; }
```

For mutable reference types where `+=` should modify in-place instead of allocating a new instance. Useful for matrix/big-number types where allocation is expensive.

For value types and immutable types, stick with `+` (the compiler synthesizes `+=`).

---

### Q26. What changed about `Span<T>` in C# 14?

First-class conversions: arrays, strings, and collection expressions implicitly convert to `Span<T>` / `ReadOnlySpan<T>`. Generic methods can accept spans as easily as arrays.

```csharp
void Process(ReadOnlySpan<int> data) { }
Process([1, 2, 3]);
Process(new[] { 1, 2, 3 });
Process(arr);
```

A huge ergonomics improvement for span-based APIs.

---

→ Next: [Coding.md](Coding.md)
