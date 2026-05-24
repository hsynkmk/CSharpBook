# Tuples

## What it is

A **tuple** is a lightweight, ordered collection of values of (possibly) different types. C# has two flavors:

- **`ValueTuple`** (modern, C# 7+) — value type, named members, deconstructable. The default.
- **`System.Tuple`** (legacy, .NET Framework era) — reference type, members named `Item1`/`Item2`/.... Rarely used in new code.

```csharp
(int Min, int Max) MinMax(int[] arr) {
    int mn = int.MaxValue, mx = int.MinValue;
    foreach (var x in arr) { if (x < mn) mn = x; if (x > mx) mx = x; }
    return (mn, mx);
}

var result = MinMax(new[] { 5, 2, 8, 1 });
Console.WriteLine(result.Min);   // 1
Console.WriteLine(result.Max);   // 8

// Or deconstruct
var (min, max) = MinMax(new[] { 5, 2, 8, 1 });
```

This file is about ValueTuple. Tuples are everywhere in modern C# — multi-return methods, ad-hoc grouping, deconstruction patterns.

---

## Why it exists

Before ValueTuple (C# 7, 2017), returning multiple values from a method meant:

- `out` parameters: clunky call syntax, can't be used in LINQ.
- A custom class/struct: heavy for one-off pairs.
- `System.Tuple`: reference type allocation, ugly `.Item1` naming.

ValueTuple solves all three:
- Value type — no heap allocation.
- Named members — `(int Min, int Max)`.
- Deconstructable — `var (a, b) = ...`.

It became the natural way to return small composites.

---

## Syntax

### Construction

```csharp
(int, string) t = (5, "hi");                       // unnamed
(int Count, string Label) t2 = (5, "items");       // named
var t3 = (Count: 5, Label: "items");                // names inferred via target type or labels

// Inferred names from variable names (C# 7.1+):
int count = 5;
string label = "items";
var t4 = (count, label);     // names inferred as Count, Label
```

### Accessing members

```csharp
var t = (Count: 5, Label: "items");
Console.WriteLine(t.Count);   // 5 — named
Console.WriteLine(t.Item1);   // 5 — Item1 still works
Console.WriteLine(t.Label);   // items
Console.WriteLine(t.Item2);   // items
```

`Item1`, `Item2`, ... are always available; the names are sugar for the same fields.

### Deconstruction

```csharp
var (count, label) = (5, "items");
Console.WriteLine(count);   // 5

// With var:
(var count, var label) = (5, "items");
(int count, string label) = (5, "items");   // explicit types

// Discard with `_`:
var (_, label) = (5, "items");

// In foreach:
var pairs = new Dictionary<string, int>();
foreach (var (k, v) in pairs) { /* ... */ }

// In switch:
return shape switch {
    (0, 0) => "origin",
    (_, 0) => "on x-axis",
    _ => "elsewhere"
};
```

The deconstruction works because `ValueTuple<T1, T2>` has a `Deconstruct` method.

### As method return type

```csharp
public (string Name, int Age) GetUser(int id) {
    // ...
    return (name, age);
}
```

In call sites:
```csharp
var user = GetUser(1);
Console.WriteLine(user.Name);

var (name, age) = GetUser(1);
```

### As a parameter

```csharp
public void Print((string Name, int Age) user) {
    Console.WriteLine($"{user.Name}, {user.Age}");
}

Print(("Alice", 30));
```

Useful but a bit awkward. Often a record or class is cleaner.

---

## Tuple equality

Tuples compare element-by-element:

```csharp
(int, int) a = (1, 2);
(int, int) b = (1, 2);
Console.WriteLine(a == b);      // true
Console.WriteLine(a.Equals(b)); // true

(int x, int y) c = (1, 2);
Console.WriteLine(a == c);      // true — names don't affect equality
```

The names are purely a compile-time convenience; the runtime type is `ValueTuple<int, int>` regardless of what you called the members.

`GetHashCode` also composes from the elements:

```csharp
var dict = new Dictionary<(int, int), string> {
    [(0, 0)] = "origin",
    [(1, 0)] = "unit x"
};
dict[(0, 0)];   // "origin"
```

This makes tuples great composite dictionary keys for ad-hoc joining.

---

## Big tuples

ValueTuple supports up to **7 named elements** directly. Beyond that, the 8th becomes a "rest" tuple recursively:

```csharp
(int A, int B, int C, int D, int E, int F, int G, int H) big = (1, 2, 3, 4, 5, 6, 7, 8);
big.H;   // 8 — works, but internally A-G are direct fields and H lives in TRest
```

Tuples of 9+ elements just work, but at that point you should be reaching for a class or record.

**Rule of thumb**: 2-3 elements → tuple is fine. 4+ → consider a named type for clarity.

---

## Tuples vs records — which?

Both bundle data with synthesized equality. When to use which?

| Tuple | Record |
|---|---|
| Quick ad-hoc grouping | Named domain concept |
| Local return, throwaway value | Long-lived data type |
| No methods or behavior | Optional methods |
| Compiler-known structural type | Distinct type for type system |
| Member names are sugar | Member names are semantic |

```csharp
// Tuple — fine for a private helper return
(int min, int max) FindBounds(int[] arr) { ... }

// Record — better when this is a recurring domain concept
public record struct Bounds(int Min, int Max);
public Bounds FindBounds(int[] arr) { ... }
```

The record gives you a real type with a real name. Tuples are for **one-shot** uses where naming the type would be overkill.

---

## Deconstruction for arbitrary types

Any type with a `Deconstruct` method (instance or extension) can be deconstructed:

```csharp
public class Person {
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public void Deconstruct(out string name, out int age) {
        name = Name;
        age = Age;
    }
}

var p = new Person { Name = "Alice", Age = 30 };
var (n, a) = p;
```

Records auto-generate `Deconstruct` for positional records. Other types you can opt into.

### Multiple `Deconstruct` overloads

You can have several. The compiler picks based on the number of `out` parameters:

```csharp
public class Address {
    public void Deconstruct(out string city, out string country) { ... }
    public void Deconstruct(out string street, out string city, out string country) { ... }
}

var (city, country) = address;
var (street, city2, country2) = address;
```

---

## Pattern matching with tuples

Switch expressions love tuples — they let you switch on multiple values together:

```csharp
string Region(int x, int y) => (x, y) switch {
    (0, 0) => "origin",
    (> 0, > 0) => "Q1",
    (< 0, > 0) => "Q2",
    (< 0, < 0) => "Q3",
    (> 0, < 0) => "Q4",
    (0, _) => "y-axis",
    (_, 0) => "x-axis",
};
```

The `(x, y) switch` syntactically creates a tuple just for pattern matching. The tuple isn't stored — it's purely a compile-time construct.

You can also match positional patterns on records and any type with `Deconstruct`:

```csharp
string Classify(Person p) => p switch {
    (_, < 18) => "minor",
    (_, < 65) => "adult",
    _ => "senior"
};
```

---

## Internals — how tuples are implemented

### `ValueTuple<T1, T2, ...>`

Tuples are instances of `System.ValueTuple<T1, T2, ...>` — value types in the BCL:

```csharp
public struct ValueTuple<T1, T2> {
    public T1 Item1;
    public T2 Item2;
    public ValueTuple(T1 item1, T2 item2) { Item1 = item1; Item2 = item2; }
}
```

There are overloads up to `ValueTuple<T1,...T8>`. The 8th type parameter is itself a `ValueTuple` (the "rest"), allowing tuples of arbitrary arity.

In IL, `(int Count, string Label) t = (5, "items")` becomes:

```il
ldc.i4.5
ldstr "items"
newobj instance void valuetype [System.Runtime]System.ValueTuple`2<int32, string>::.ctor(!0, !1)
stloc.0
```

The names `Count` / `Label` are NOT stored in IL of the local; they live in **metadata** on the parameter / return-type / variable declaration via `[TupleElementNames]` attribute:

```csharp
[return: TupleElementNames(new[] { "Count", "Label" })]
public static ValueTuple<int, string> GetCounts() { ... }
```

Reflection can read these. The IL itself has no notion of tuple member names.

### Naming is metadata

This has a consequence: **tuple names don't affect runtime equality, hashing, or storage**. `(int Count, string Label)` and `(int A, string B)` are the **same runtime type**: `ValueTuple<int, string>`.

```csharp
(int X, int Y) a = (1, 2);
(int Width, int Height) b = (1, 2);
Console.WriteLine(a == b);       // true — same runtime type
```

This is unlike records, where different declarations are different types.

### Stack-friendly

Because tuples are value types, the JIT often keeps them entirely in CPU registers. A `(int, int)` is just two ints; the runtime treats it accordingly. That's why returning a small tuple is essentially free.

For a tuple with 8 elements or contains reference types, the picture is more complex (the struct has heap pointers; the struct itself is still stack-allocated unless boxed).

---

## Common patterns

### Multi-return method

```csharp
public (bool Success, T? Value) TryGet<T>(string key) where T : class {
    if (_cache.TryGetValue(key, out var v) && v is T t) return (true, t);
    return (false, null);
}

if (TryGet<User>("alice") is (true, var user)) {
    // ...
}
```

### Composite dictionary key

```csharp
var grid = new Dictionary<(int X, int Y), Cell>();
grid[(0, 0)] = new Cell();
```

### Tuple-based "swap"

```csharp
(a, b) = (b, a);  // swap without a temp
```

### Group-by composite key

```csharp
var byCountryAndYear = data
    .GroupBy(d => (d.Country, d.Year))
    .Select(g => new { g.Key.Country, g.Key.Year, Total = g.Sum(d => d.Amount) });
```

### Inline composite return

```csharp
var (name, age) = ParseUser(input);
```

---

## Performance notes

- Tuples are value types — no GC pressure for short-lived ones.
- Returning a tuple from a method is generally as fast as multiple `out` parameters, often inlined by the JIT.
- Boxing a tuple (`object o = (1, 2)`) allocates. Same as any value type.
- Tuples of value types stay on stack/registers. Tuples containing reference types still have stack-side overhead, but each reference field points to heap data normally.
- Hash codes for tuples use `HashCode.Combine` internally (well-distributed, no allocations).

---

## Common bugs

- **Confusing element names with element types** — `(int A, string B)` and `(int X, string Y)` are the **same type** at runtime.
- **Wrong order in deconstruction** — `var (age, name) = GetUser();` if GetUser returns `(name, age)` is a subtle bug (compiles, swaps values). Mitigated by named tuples.
- **Too-large tuples** — at 5+ elements, switch to a record.
- **Using `System.Tuple` (the legacy reference-type)** in new code — use ValueTuple unless you have a specific need.
- **Comparing tuples with reference-type members** — `((object)x, (object)y) == ((object)x, (object)y)` uses default `EqualityComparer<object>` which is reference equality. Watch out.

---

## When to use a tuple

✓ Returning 2-3 values from a private/internal method.
✓ Composite dictionary keys.
✓ Local grouping for LINQ projections.
✓ Pattern matching on multiple values.
✓ Deconstructing for clarity.

✗ Public API return types — name the type for clarity (record/class).
✗ More than 3-4 elements.
✗ When the tuple is going to be passed around extensively — give it a name.

→ Next: [06-NullableTypes.md](06-NullableTypes.md)
