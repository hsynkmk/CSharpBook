# C# 13 (.NET 9, November 2024)

> `params` collections, partial properties/indexers, `lock` object type, escape sequence `\e`, ref struct generic args. C# 13 added small wins across the language.

---

## `params` collections — beyond arrays

```csharp
public int Sum(params IEnumerable<int> nums) {           // any IEnumerable
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

public int Sum(params ReadOnlySpan<int> nums) {           // ReadOnlySpan — zero-alloc params!
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

Sum(1, 2, 3);   // works for any of the above
```

Pre-C# 13, `params` was restricted to arrays (`params T[]`). Each call allocated a new array. C# 13 lets `params` work with:

- `T[]` (existing)
- `IEnumerable<T>`
- `ICollection<T>`, `IList<T>`, `IReadOnlyList<T>`
- `ReadOnlySpan<T>`, `Span<T>`
- Any type with the right collection-expression support

The big win: `params ReadOnlySpan<T>` — the compiler stack-allocates the span. **Zero heap allocation per call.** Hot paths can use `params` without GC pressure.

Used widely in modern BCL (Console.WriteLine, StringBuilder.Append, etc., have ReadOnlySpan overloads).

---

## Partial properties and indexers

```csharp
public partial class Person {
    public partial string FullName { get; }
}

public partial class Person {
    public string FirstName { get; init; } = "";
    public string LastName { get; init; } = "";
    public partial string FullName => $"{FirstName} {LastName}";
}
```

Like partial methods, partial properties / indexers let you split declaration from implementation. Used by source generators that emit property getters/setters.

Useful for:
- `[GeneratedRegex]` — emits regex-backed property.
- Source generators that produce property bodies based on attributes.

For hand-written code, you'll rarely use this.

---

## `System.Threading.Lock` type

```csharp
private readonly Lock _gate = new();   // System.Threading.Lock

public void Increment() {
    lock (_gate) {
        _counter++;
    }
}
```

A dedicated lock type. Replaces locking on `object` for two benefits:
- Type-safe — can't accidentally lock a value type or use a confusing object.
- Slightly faster — no lock-word indirection per call.

The C# compiler recognizes `Lock`-typed expressions and uses the optimized methods (`Enter()`, `Exit()`) instead of the generic `Monitor.Enter/Exit`.

```csharp
// Old style still works
private readonly object _gate = new();
lock (_gate) { ... }

// C# 13+ style — faster and clearer
private readonly Lock _gate = new();
lock (_gate) { ... }
```

For new code targeting .NET 9+, prefer `Lock`. See [Chapter 08 §09](../08-Concurrency/09-LockingPrimitives.md).

---

## Escape sequence `\e`

```csharp
Console.Write("\e[31mRed text\e[0m");   // ANSI red
```

`\e` is the ASCII escape character (0x1B). Useful for ANSI terminal codes.

Pre-C# 13:
```csharp
Console.Write("[31mRed text[0m");   // verbose Unicode escape
```

Saves a few characters. Tiny win, but pleasant.

---

## `^` (index from end) in initializers

```csharp
int[] arr = new int[10];
arr[^1] = 99;       // set last element
arr[^2] = 98;

var dict = new Dictionary<int, string> {
    [^1] = "last",   // ⚠ — only works on indexable types; varies
};
```

C# 13 extended where the `^` index can appear. Mostly subtle — array indexing already supported it.

---

## Allow ref struct as generic argument

```csharp
public T Process<T>(T input) where T : allows ref struct {
    return input;
}

Process(stackalloc byte[100]);   // ✓ — Span<byte> as generic argument
```

Pre-C# 13, `Span<T>` (a `ref struct`) couldn't be passed as a generic type parameter — the framework assumed Ts could escape, which is unsafe for ref structs.

C# 13 added the `allows ref struct` constraint. Code with the constraint can accept ref struct args; the compiler enforces the lifetime rules.

Niche but powerful for generic high-performance code (e.g., generic helpers that work over both `byte[]` and `Span<byte>`).

---

## Method group natural type improvements

```csharp
int Square(int x) => x * x;
var f = Square;   // C# 13: f is Func<int, int> via natural type
```

Pre-C# 13, `var f = Square;` was a compile error (no target type for the method group). C# 13 infers the natural delegate type.

Minor convenience for functional-style code.

---

## Improved overload resolution

Various edge cases improved — method group conversions, generic inference, collection expressions in overloads. Mostly invisible quality-of-life.

---

## Performance improvements (.NET 9)

.NET 9 brought:
- **OSR (On-Stack Replacement) v2**: better Tier 1 promotion mid-execution.
- **`Task.WhenEach`**: stream completing tasks (see [Chapter 08 §15](../08-Concurrency/15-TaskWhenAllWhenAny.md)).
- **HybridCache**: in-process + distributed cache combined.
- **OpenAPI native support** in ASP.NET Core (no Swashbuckle needed for basic schemes).
- **System.Text.Json improvements**: faster source-gen, more edge cases.

.NET 9 is STS (Short-Term Support — ends mid-2026). Many teams skip it; the next LTS is .NET 10.

---

## Adoption tips for C# 13

1. **`Lock` type** for new locking code.
2. **`params ReadOnlySpan<T>`** in hot APIs to eliminate allocations.
3. **Partial properties** if you write source generators.

For most application code, C# 13 is incremental polish over C# 12. The big features come back with C# 14 / .NET 10.

---

## Summary of C# 13

**Big wins**:
- `params` collections (especially `params ReadOnlySpan<T>`).
- `Lock` type — faster locking.

**Smaller wins**:
- Partial properties.
- `\e` escape.
- `allows ref struct` generic constraint.
- Method group natural type.

C# 13 was a polishing release. The next, C# 14, brings more transformative features.

→ Next: [07-CSharp14.md](07-CSharp14.md)
