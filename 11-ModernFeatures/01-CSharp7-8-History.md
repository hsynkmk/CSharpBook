# C# 7 → 8 — The Modernization Era

> C# 7 (2017) and C# 8 (2019) reshaped the language. Tuples, patterns, async streams, nullable references — the foundations of "modern C#" land here.

---

## C# 7.0 (March 2017, .NET Framework 4.7 / .NET Core 1.x)

### `out var` — inline declaration

```csharp
// Pre-C# 7
int n;
if (int.TryParse(input, out n)) { /* use n */ }

// C# 7
if (int.TryParse(input, out int n)) { /* use n */ }
if (int.TryParse(input, out var n2)) { /* use n2 */ }
```

Eliminates the dummy declaration before TryParse-style methods.

### Tuples — ValueTuple

```csharp
public (int Min, int Max) MinMax(int[] arr) {
    int mn = int.MaxValue, mx = int.MinValue;
    foreach (var x in arr) { if (x < mn) mn = x; if (x > mx) mx = x; }
    return (mn, mx);
}

var result = MinMax(new[] { 3, 1, 4 });
Console.WriteLine($"{result.Min}, {result.Max}");
```

Lightweight value-typed tuples with named members. Used everywhere now.

### Deconstruction

```csharp
var (min, max) = MinMax(arr);
foreach (var (k, v) in dict) { /* ... */ }
```

Plus `Deconstruct` method on any type:

```csharp
public class Point {
    public int X, Y;
    public void Deconstruct(out int x, out int y) { x = X; y = Y; }
}

var p = new Point { X = 3, Y = 4 };
var (x, y) = p;
```

### Pattern matching v1

`is` pattern + `switch` with type patterns:

```csharp
if (obj is string s) Console.WriteLine(s.Length);

switch (obj) {
    case int n when n > 0:
        Console.WriteLine($"positive int {n}");
        break;
    case string s:
        Console.WriteLine($"string of length {s.Length}");
        break;
}
```

The seed of what would become full pattern matching in C# 8–11.

### Local functions

```csharp
public int Process(int[] data) {
    return Sum(Filter(data));

    int[] Filter(int[] arr) => arr.Where(x => x > 0).ToArray();
    int Sum(int[] arr) => arr.Sum();
}
```

Methods nested in methods — useful for encapsulation, iterator helpers, async helpers.

### `throw` expressions

```csharp
public string Name {
    get => _name ?? throw new InvalidOperationException();
    set => _name = value ?? throw new ArgumentNullException();
}
```

Throw in any expression context.

### Expression-bodied members (expanded)

```csharp
public Person(string name) => Name = name;   // expression-bodied ctor
~Person() => Console.WriteLine("finalized");   // expression-bodied finalizer
public string Name { get; }
public string Greeting => $"Hi, {Name}";   // already in C# 6
```

### Ref returns and ref locals

```csharp
public ref int Find(int[] arr, int target) {
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] == target) return ref arr[i];
    throw new InvalidOperationException();
}

ref int slot = ref Find(data, 5);
slot = 99;   // modifies data[i] in place
```

Direct memory aliasing without unsafe pointers.

### Binary literals + digit separators

```csharp
int b = 0b_1010_1100;   // binary literal with underscore separators
int big = 1_000_000;     // underscore for readability
```

---

## C# 7.1 (August 2017)

### `async Main`

```csharp
static async Task Main(string[] args) {
    await Task.Delay(100);
}
```

Async entry point. No more `.Result` hacks at startup.

### `default` literal

```csharp
int n = default;          // 0
string? s = default;      // null
List<int>? l = default;   // null
```

Inferred from target. Replaces `default(int)` etc.

### Inferred tuple element names

```csharp
int count = 5;
string label = "items";
var t = (count, label);   // member names inferred: t.count, t.label
```

---

## C# 7.2 (December 2017)

### `Span<T>` becomes usable

C# 7.2 added language features to make `Span<T>` and `ref struct` work properly:

- `ref struct` types (Span<T>, ReadOnlySpan<T>).
- `in` parameters (read-only by-reference).
- `ref readonly` returns.
- `Span<T>` as a return type (with lifetime rules).

```csharp
public void Process(in BigStruct s) { /* read s */ }
public Span<byte> AsSpan(byte[] data) => data;
```

The infrastructure for high-performance APIs.

### Stackalloc in expressions

```csharp
Span<int> buf = stackalloc int[10];   // works in any context, not just unsafe
```

Pre-7.2, stackalloc was unsafe-only. Now it's the safe span form.

### Private protected

```csharp
public class Base {
    private protected int X;   // same assembly AND derived
}
```

New access modifier — stricter than `protected internal`.

---

## C# 7.3 (May 2018)

Minor refinements:
- Tuple equality (`==` between two ValueTuples).
- `where T : unmanaged` constraint.
- `where T : Delegate`, `where T : Enum`.
- `ref` reassignment.
- Improved overload resolution.

---

## C# 8.0 (September 2019, .NET Core 3.x)

The big one. C# 8 introduced features that defined modern C#.

### Nullable Reference Types

```csharp
#nullable enable

string s = "hello";        // non-null
string? maybe = null;       // nullable
maybe.Length;               // ⚠ warning
if (maybe != null) maybe.Length;   // OK
```

Compile-time tracking of null state for reference types. Massive win — catches NREs before runtime. See [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md).

### Default interface methods

```csharp
public interface ILogger {
    void Log(string m);
    void LogError(string m) => Log($"ERROR: {m}");   // default implementation
}
```

Lets you add methods to interfaces without breaking implementers. Used for evolving APIs. See [Chapter 02 §08](../02-OOP/08-Interfaces.md).

### Switch expressions

```csharp
string Category(int age) => age switch {
    < 0 => "invalid",
    < 13 => "child",
    < 18 => "teen",
    < 65 => "adult",
    _ => "senior"
};
```

Expression-based switch with patterns. Cleaner than statement switches; supports exhaustiveness checking. See [Chapter 03 §09](../03-TypeSystem/09-PatternMatching.md).

### Async streams — `IAsyncEnumerable<T>`

```csharp
public async IAsyncEnumerable<int> CountAsync() {
    for (int i = 0; i < 10; i++) {
        await Task.Delay(100);
        yield return i;
    }
}

await foreach (var n in CountAsync()) Console.WriteLine(n);
```

Streaming async data. Foundation for modern data pipelines. See [Chapter 08 §07](../08-Concurrency/07-AsyncStreams.md).

### Ranges and indices

```csharp
int[] arr = { 1, 2, 3, 4, 5 };
arr[^1];     // 5 — last element
arr[..3];    // { 1, 2, 3 }
arr[2..];    // { 3, 4, 5 }
arr[1..^1];  // { 2, 3, 4 }

Index lastIdx = ^1;
Range range = 1..4;
```

Concise slice / index syntax. Works on arrays, strings, `Span<T>`, `List<T>`.

### `using` declarations

```csharp
public void Read(string path) {
    using var stream = File.OpenRead(path);   // disposed at end of method
    // ... use stream ...
}
```

No more `using (...)` block indentation. Scope is the enclosing block.

### `await using`

```csharp
public async Task ProcessAsync() {
    await using var ctx = new DbContext();
    await ctx.SaveChangesAsync();
}
```

Async disposal. Pairs with `IAsyncDisposable`. See [Chapter 08 §08](../08-Concurrency/08-AsyncDisposable.md).

### Recursive patterns

```csharp
if (obj is Person { Address: { City: "Springfield" }, Age: > 18 } adult) { /* ... */ }
```

Property + type + binding combined into one pattern. The seed of full pattern matching power.

### Null-coalescing assignment (`??=`)

```csharp
string? cached = null;
cached ??= LoadFromDb();   // assign only if null
```

### `static` local functions

```csharp
void M() {
    static int Square(int x) => x * x;   // can't capture locals
    Square(5);
}
```

Forbid local function from capturing locals → no allocation when called by name.

### `readonly` struct members

```csharp
public struct Counter {
    public int Value;
    public readonly int Peek() => Value;   // promises not to mutate
}
```

Per-method "doesn't mutate" annotation for struct members.

### Stackalloc in any expression (now allowed everywhere)

C# 8 expanded stackalloc to nested expressions:

```csharp
Span<int> buf = stackalloc int[256];
SomeMethod(stackalloc[] { 1, 2, 3 });
```

---

## What C# 7 and 8 changed culturally

Pre-C# 7, the language felt "complete but verbose." Tuples and patterns brought functional-style brevity.

C# 8's NRT was the biggest leap — eliminating an entire class of runtime exceptions at compile time. Combined with `using` declarations, switch expressions, and async streams, C# became significantly more expressive.

These versions also introduced the **annual release** cadence — November releases each year, paired with a new .NET runtime (Core 2.1, 3.0, 3.1, then .NET 5+).

---

## Adoption tips for legacy codebases

If you're moving an older project to C# 7-8 era:

1. Enable C# 8 in csproj: `<LangVersion>latest</LangVersion>`.
2. Adopt `using` declarations everywhere — quick win, cleaner code.
3. Replace TryParse patterns with `out var`.
4. Use switch expressions for value-returning dispatch.
5. Enable NRT (`<Nullable>annotations</Nullable>` first, then `enable`).
6. Annotate APIs with `?` where they return null.

Big payoff for small effort.

---

## Summary

C# 7 added: tuples, deconstruction, pattern matching v1, local functions, ref features, `out var`, throw expressions.

C# 7.1-7.3 polished: async Main, default literal, Span foundations, in/ref readonly parameters, unmanaged constraint.

C# 8 transformed: NRT, switch expressions, async streams, ranges/indices, using declarations, default interface methods, recursive patterns.

This was the era that made "modern C#" feel modern. Most code you write today builds on these.

→ Next: [02-CSharp9.md](02-CSharp9.md)
