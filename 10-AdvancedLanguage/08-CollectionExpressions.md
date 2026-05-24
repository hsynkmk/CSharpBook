# Collection Expressions (C# 12+)

## What it is

C# 12 (2023) added **collection expressions** — a unified syntax for creating collections. The same `[a, b, c]` literal works for arrays, lists, spans, and more, with the compiler choosing the most efficient construction.

```csharp
int[] arr = [1, 2, 3, 4, 5];
List<int> list = [1, 2, 3];
Span<int> span = [1, 2, 3];
ImmutableArray<int> immut = [1, 2, 3];
IEnumerable<int> seq = [1, 2, 3];
```

Plus **spread** for unpacking:

```csharp
int[] head = [0];
int[] tail = [5, 6, 7];
int[] all = [..head, 1, 2, 3, ..tail];   // {0, 1, 2, 3, 5, 6, 7}
```

The compiler picks the right backing storage based on the target type — no more remembering `new int[] { ... }` vs `new List<int> { ... }` vs `ImmutableArray.Create(...)`.

---

## Why it exists

Pre-C# 12, collection initialization syntax varied by type:

```csharp
int[] arr = new int[] { 1, 2, 3 };
int[] arr2 = { 1, 2, 3 };               // array initializer (limited contexts)
List<int> list = new List<int> { 1, 2, 3 };
HashSet<int> set = new HashSet<int> { 1, 2, 3 };
ImmutableArray<int> imm = ImmutableArray.Create(1, 2, 3);
Span<int> span = stackalloc int[] { 1, 2, 3 };
```

Each had its own syntax. Adding spread / range operations was awkward.

Collection expressions unify it: `[...]` everywhere. The compiler chooses the construction strategy based on target type.

---

## Basics

```csharp
// Arrays
int[] arr = [1, 2, 3];
double[] doubles = [1.0, 2.5, 3.7];
string[] strings = ["a", "b", "c"];

// List<T>
List<int> list = [1, 2, 3];

// Span<T> / ReadOnlySpan<T>
Span<int> span = [1, 2, 3];
ReadOnlySpan<char> chars = ['a', 'b', 'c'];

// ImmutableArray<T>
ImmutableArray<int> imm = [1, 2, 3];

// HashSet<T>
HashSet<string> set = ["a", "b"];

// Dictionary — NO! (different syntax: { [key] = value })
Dictionary<int, string> dict = ...;   // collection expressions don't (yet) support dictionary
```

Target-typed: the compiler reads the destination type and produces the right collection.

---

## Spread (`..`)

Concatenate or unpack:

```csharp
int[] a = [1, 2, 3];
int[] b = [4, 5];
int[] c = [..a, ..b];          // {1, 2, 3, 4, 5}
int[] d = [0, ..a, 99, ..b, 100];   // {0, 1, 2, 3, 99, 4, 5, 100}
```

Spread accepts anything iterable (`IEnumerable<T>`) plus arrays / spans.

For records / immutable collections, spread is the cleanest way to "add an element":

```csharp
ImmutableArray<int> items = [1, 2, 3];
ImmutableArray<int> withFour = [..items, 4];   // new array {1, 2, 3, 4}
```

---

## Empty collections

```csharp
int[] empty = [];
List<int> empty2 = [];
Span<int> empty3 = [];
```

Empty arrays use `Array.Empty<T>()` under the hood (no allocation). Empty lists allocate a small List.

---

## Conditional addition (NOT supported)

```csharp
int[] arr = [1, 2, condition ? 3 : 0];   // ✓ — each element is an expression
int[] arr2 = [1, 2, ...condition ? [3] : []];   // ⚠ — spread + conditional must be carefully typed
```

For "include if condition" patterns, sometimes you need:

```csharp
int[] arr = condition ? [1, 2, 3] : [1, 2];
```

The collection expression itself isn't a conditional builder.

---

## In method calls

```csharp
public void Process(IEnumerable<int> items) { ... }

Process([1, 2, 3]);   // ✓
Process([..source, 100]);   // ✓
```

Pass a collection expression as a method argument. The compiler targets the parameter type.

Particularly clean with `params`:

```csharp
public static int Sum(params ReadOnlySpan<int> values) {   // C# 14+
    int s = 0;
    foreach (var v in values) s += v;
    return s;
}

Sum([1, 2, 3]);   // works for params ReadOnlySpan<int>
```

---

## In return values

```csharp
public int[] GetData() => [1, 2, 3];
public IReadOnlyList<int> GetItems() => [1, 2, 3];
public ImmutableArray<int> GetImmutable() => [1, 2, 3];
```

Same syntax; compiler picks construction based on return type.

---

## Collection expressions in fields / properties

```csharp
public class C {
    public int[] Data { get; } = [1, 2, 3];
    public ImmutableArray<string> Tags { get; } = ["a", "b"];
}
```

Initializer expressions can use collection literals. Target-typed against the property type.

---

## Internals — what the compiler emits

For `int[] arr = [1, 2, 3];`:

```il
ldc.i4.3
newarr int32
dup
ldc.i4.0
ldc.i4.1
stelem.i4
dup
ldc.i4.1
ldc.i4.2
stelem.i4
dup
ldc.i4.2
ldc.i4.3
stelem.i4
stloc.0
```

Just a regular array creation. The compiler emits direct IL for the target.

For `List<int> list = [1, 2, 3];`:

```csharp
// Roughly:
var list = new List<int>(3);   // capacity 3
list.Add(1);
list.Add(2);
list.Add(3);
```

Or, the compiler may use `CollectionsMarshal.AsSpan` / similar internals to be even more efficient. The exact code is JIT- and compiler-version-dependent.

For `Span<int> span = [1, 2, 3];`:

```csharp
Span<int> span = stackalloc int[3] { 1, 2, 3 };
```

The compiler often uses `stackalloc` for span targets — saves heap allocation. Brilliant for short-lived buffers.

For `ImmutableArray<int>`:

```csharp
ImmutableArray<int> imm = ImmutableArray.Create(1, 2, 3);
```

Calls the appropriate factory.

### Custom collections: `[CollectionBuilder]`

Library authors can opt their type into collection expressions:

```csharp
[CollectionBuilder(typeof(MyListBuilder), nameof(MyListBuilder.Create))]
public class MyList<T> { ... }

public static class MyListBuilder {
    public static MyList<T> Create<T>(ReadOnlySpan<T> elements) => /* ... */;
}
```

Now `MyList<int> list = [1, 2, 3];` works. The compiler calls the builder method.

---

## Spread internals

`[..source, x]` is compiled as:
1. Compute source's element count (if available via `Count`/`Length`).
2. Allocate target with the right size.
3. Copy source's elements; then append x.

For arrays / spans / collections with known size, this is a single allocation + memcpy. For `IEnumerable<T>` (unknown size), the compiler may fall back to growing.

---

## Common patterns

### Initialize from constants

```csharp
public static readonly int[] PrimeStartList = [2, 3, 5, 7, 11, 13];
```

Cleaner than `new int[] { ... }`.

### Append / prepend

```csharp
int[] withZero = [0, ..arr];          // prepend 0
int[] withLast = [..arr, 99];          // append 99
int[] doubled = [..arr, ..arr];        // concatenate with itself
```

### Method args

```csharp
Console.WriteLine(string.Join(",", [1, 2, 3]));   // doesn't work — string.Join expects IEnumerable<int>
                                                     // but maybe ambiguous; use explicit IEnumerable
IEnumerable<int> e = [1, 2, 3];
Console.WriteLine(string.Join(",", e));
```

Some APIs need explicit typing.

### Empty literal

```csharp
public int[] Get() => [];
public List<string> Defaults() => [];
```

Cleaner than `Array.Empty<int>()` or `new List<string>()`.

### Spread for variadic-like calls

```csharp
public void Log(params string[] messages) { ... }

string[] common = ["a", "b", "c"];
Log(..common, "extra");      // ✓ — spread the array, plus extra
```

Spread expands into the params array.

---

## Comparison with previous syntax

| Pre-C# 12 | C# 12+ |
|---|---|
| `new int[] { 1, 2, 3 }` | `[1, 2, 3]` (when typed) |
| `new[] { 1, 2, 3 }` | `[1, 2, 3]` |
| `new List<int> { 1, 2, 3 }` | `[1, 2, 3]` (when typed List<int>) |
| `ImmutableArray.Create(1, 2, 3)` | `[1, 2, 3]` (when typed ImmutableArray) |
| `Array.Empty<int>()` | `[]` |

Cleaner across the board. The new syntax is target-typed; the old syntax is still valid (and sometimes clearer for readers unfamiliar with C# 12+).

---

## When NOT to use

- For very explicit type situations, the old syntax may be clearer.
- Mixed-type collections aren't supported (no `[1, "hi"]` — what type?).
- Dictionaries: no syntax yet for `[key: value]`.

For most array / list / span construction in new code: collection expressions are cleaner.

---

## Common bugs

### Ambiguous target type

```csharp
var arr = [1, 2, 3];   // ⚠ — no target type, compiler can't infer
```

`var` can't infer from a collection expression alone. Specify:

```csharp
int[] arr = [1, 2, 3];                 // ✓
var arr2 = (int[])[1, 2, 3];           // cast
List<int> arr3 = [1, 2, 3];
```

### Method overload resolution

```csharp
public void Process(int[] arr) { }
public void Process(List<int> list) { }

Process([1, 2, 3]);   // ⚠ — ambiguous: could be either target type
```

Compiler error if both overloads can accept a collection expression. Add explicit cast:

```csharp
Process((int[])[1, 2, 3]);
```

### Spread of `null`

```csharp
int[]? maybe = null;
int[] all = [..maybe];   // ⚠ — null source, NRE at runtime
```

The compiler may warn about null. Check first:

```csharp
int[] all = maybe is not null ? [..maybe] : [];
```

---

## Performance

For arrays: identical IL to `new T[] { ... }`. Same speed.
For lists: identical to `new List<T>(capacity) + Add()`. Same.
For spans: often uses `stackalloc` — faster than going through an array.
For immutable collections: same as the factory method.

The win is **clarity**, not raw speed. Sometimes the compiler can be smarter than hand-written init (e.g., choosing stackalloc), but usually neutral.

---

## When to use

✓ All new code creating arrays / lists / spans / immutable collections.
✓ Anywhere `new T[]`, `new List<T>`, or `ImmutableArray.Create` is currently used.
✓ When you want to compose collections with spread.

✗ Mixed-type collections (no support).
✗ Dictionary initialization (use `{ [key] = value }` initializer for now).
✗ Old codebases where mixing styles confuses readers — pick one style.

---

## Summary

- `[a, b, c]` — collection expression. Target-typed.
- Works for arrays, lists, spans, immutable collections, custom collections via `[CollectionBuilder]`.
- Spread `..` concatenates or unpacks.
- `[]` is the empty collection.
- Same syntax across collection types — fewer things to remember.
- Compiler picks efficient construction (often `stackalloc` for spans, `Array.Empty<T>` for empty).
- C# 12+; replaces `new T[] { ... }`, `new List<T> { ... }`, `ImmutableArray.Create(...)`.

→ Next: [09-RawStringLiterals.md](09-RawStringLiterals.md)
