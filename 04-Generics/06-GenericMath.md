# Generic Math

## What it is

A set of interfaces in `System.Numerics` (.NET 7+) that let you write **algorithms generic over numeric types**. `int`, `double`, `decimal`, `BigInteger`, even your own types can all plug into one shared codebase.

```csharp
using System.Numerics;

public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}

Sum(new[] { 1, 2, 3 });         // int: 6
Sum(new[] { 1.5, 2.5 });        // double: 4.0
Sum(new[] { 1m, 2m, 3m });      // decimal: 6
Sum(new BigInteger[] { 1, 2 }); // BigInteger: 3
```

Pre-C# 11 this required either `dynamic` (slow), per-type overloads (duplication), or expression-tree wizardry. Generic math, built on **static abstract members in interfaces**, makes it natural.

---

## The interface zoo

`System.Numerics` defines a tower of small interfaces, each requiring specific operators and members:

| Interface | Requires |
|---|---|
| `IAdditionOperators<TSelf, TOther, TResult>` | `operator +` |
| `ISubtractionOperators<...>` | `operator -` |
| `IMultiplyOperators<...>` | `operator *` |
| `IDivisionOperators<...>` | `operator /` |
| `IModulusOperators<...>` | `operator %` |
| `IUnaryNegationOperators<TSelf, TResult>` | `operator -` (unary) |
| `IUnaryPlusOperators<TSelf, TResult>` | `operator +` (unary) |
| `IIncrementOperators<TSelf>` | `operator ++` |
| `IDecrementOperators<TSelf>` | `operator --` |
| `IComparisonOperators<TSelf, TOther, bool>` | `<`, `>`, `<=`, `>=` |
| `IEqualityOperators<TSelf, TOther, bool>` | `==`, `!=` |
| `IBitwiseOperators<TSelf, TOther, TResult>` | `&`, `|`, `^`, `~` |
| `IShiftOperators<TSelf, TOther, TResult>` | `<<`, `>>`, `>>>` |
| `IAdditiveIdentity<TSelf, TResult>` | `Zero` |
| `IMultiplicativeIdentity<TSelf, TResult>` | `One` |
| `IMinMaxValue<TSelf>` | `MinValue`, `MaxValue` |
| `IFloatingPoint<TSelf>` | floor, ceiling, truncate, etc. |
| `IFloatingPointConstants<TSelf>` | `E`, `Pi`, `Tau` |
| `IExponentialFunctions<TSelf>` | `Exp`, `Log` |
| `ITrigonometricFunctions<TSelf>` | `Sin`, `Cos`, `Tan`, ... |
| `INumberBase<TSelf>` | converts, parses, basic number-ness |
| `INumber<TSelf>` | inherits a LOT of the above |

`INumber<T>` is the big "I'm a regular number" combo interface. Built-in types like `int`, `long`, `double`, `decimal`, `BigInteger` all implement it.

---

## Writing algorithms

### Sum

```csharp
public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}
```

`T.Zero` is a static abstract member of `INumberBase<T>`. `+=` resolves through `IAdditionOperators<T, T, T>`.

### Average

```csharp
public static T Average<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    int count = 0;
    foreach (var x in source) { total += x; count++; }
    if (count == 0) throw new InvalidOperationException();
    return total / T.CreateChecked(count);
}

double a = Average(new[] { 1.0, 2.0, 3.0 });        // 2.0
decimal b = Average(new[] { 10m, 20m, 30m });        // 20m
```

`T.CreateChecked` converts an int to T with overflow check.

### Min/Max

```csharp
public static T Min<T>(T a, T b) where T : IComparisonOperators<T, T, bool> =>
    a < b ? a : b;

public static T Max<T>(T a, T b) where T : IComparisonOperators<T, T, bool> =>
    a > b ? a : b;
```

You can use raw operators because the interface requires them.

### Sum of squares

```csharp
public static T SumOfSquares<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x * x;
    return total;
}
```

### Vector dot product

```csharp
public static T Dot<T>(IEnumerable<T> a, IEnumerable<T> b) where T : INumber<T> {
    T sum = T.Zero;
    foreach (var (x, y) in a.Zip(b)) sum += x * y;
    return sum;
}

double d = Dot(new[] { 1.0, 2.0, 3.0 }, new[] { 4.0, 5.0, 6.0 });   // 32.0
```

### Generic conversion

```csharp
public static TOut Convert<TIn, TOut>(TIn value)
    where TIn : INumberBase<TIn>
    where TOut : INumberBase<TOut>
{
    return TOut.CreateChecked(value);
}

double d = Convert<int, double>(42);        // 42.0
int i = Convert<double, int>(3.7);          // throws — checked conversion
int i2 = TOut.CreateTruncating(3.7);        // 3 — truncating conversion
int i3 = TOut.CreateSaturating(int.MaxValue + 1.0);   // int.MaxValue
```

The three conversion methods:
- `CreateChecked` — throws on overflow.
- `CreateSaturating` — clamps to MinValue/MaxValue.
- `CreateTruncating` — wraps / truncates.

---

## Implementing INumber on your own type

To plug into all the math algorithms, your type implements `INumber<T>` (or whichever subset suits):

```csharp
public readonly record struct Fixed4 : INumber<Fixed4> {
    public int Raw { get; init; }   // value scaled by 10000

    public Fixed4(int raw) { Raw = raw; }
    public static Fixed4 FromDouble(double v) => new((int)Math.Round(v * 10000));
    public double ToDouble() => Raw / 10000.0;

    // INumber<T> requires a lot. Sample (incomplete):
    public static Fixed4 Zero => new(0);
    public static Fixed4 One => new(10000);
    public static Fixed4 AdditiveIdentity => Zero;
    public static Fixed4 MultiplicativeIdentity => One;

    public static Fixed4 operator +(Fixed4 a, Fixed4 b) => new(a.Raw + b.Raw);
    public static Fixed4 operator -(Fixed4 a, Fixed4 b) => new(a.Raw - b.Raw);
    public static Fixed4 operator *(Fixed4 a, Fixed4 b) => new((int)((long)a.Raw * b.Raw / 10000));
    public static Fixed4 operator /(Fixed4 a, Fixed4 b) => new((int)((long)a.Raw * 10000 / b.Raw));

    public static bool operator <(Fixed4 a, Fixed4 b) => a.Raw < b.Raw;
    public static bool operator >(Fixed4 a, Fixed4 b) => a.Raw > b.Raw;
    public static bool operator <=(Fixed4 a, Fixed4 b) => a.Raw <= b.Raw;
    public static bool operator >=(Fixed4 a, Fixed4 b) => a.Raw >= b.Raw;

    // ... plus CreateChecked, CreateSaturating, CreateTruncating, TryConvert*, etc.
}
```

The full surface of `INumber<T>` is dozens of members. The BCL has many helpers to delegate to underlying types. For most users you'd implement enough to satisfy the algorithms you actually call.

For most code: **don't implement INumber yourself unless you're writing a number library**. The built-in types cover 99% of use.

---

## SIMD interop

Generic math composes with `System.Numerics.Vector<T>`:

```csharp
public static T SimdSum<T>(ReadOnlySpan<T> source) where T : INumber<T> {
    Vector<T> vSum = Vector<T>.Zero;
    int i = 0;
    for (; i <= source.Length - Vector<T>.Count; i += Vector<T>.Count) {
        var v = new Vector<T>(source.Slice(i));
        vSum += v;
    }
    T scalar = T.Zero;
    for (int j = 0; j < Vector<T>.Count; j++) scalar += vSum[j];
    for (; i < source.Length; i++) scalar += source[i];
    return scalar;
}
```

Generic math + SIMD = fastest portable numeric loops.

---

## Where this matters

### Library authors

You're writing a stats library. Pre-generic-math, you had to:
- Hand-roll `MeanInt`, `MeanDouble`, `MeanDecimal`, etc.
- Or, hand-craft a `Numeric<T>` runtime adapter that boxes/unboxes (slow).
- Or, force consumers to convert to `double` (lossy for decimal).

With generic math, one `Mean<T>` function works for everything. Type-safe, no boxing, JIT-specialized.

### Game/graphics code

Vectors, matrices, quaternions — generic over `float`, `double`, half-precision. One implementation works everywhere.

### Financial / measurement code

`decimal`, custom fixed-point, custom currency types — all share the same statistical functions.

---

## Internals — performance

The JIT specializes generic math algorithms per value-type T:

```csharp
public T Sum<T>(IEnumerable<T> source) where T : INumber<T> { ... }
```

`Sum<int>` and `Sum<double>` each get their own machine code. Inside each:
- `T.Zero` → a constant load (`ldc.i4.0` for int, `ldc.r8 0.0` for double).
- `total += x` → a direct `add` IL instruction (no virtual dispatch).
- The loop is identical to what you'd hand-write.

Performance: **as fast as hand-written for-loops** for primitive types. Sometimes the JIT inlines more aggressively than a hand-rolled version (because of cleaner generic specialization).

For reference-type Ts (like `BigInteger`), the algorithm shares code; calls go through dispatch but are typically inlined. Still close to optimal.

---

## Common patterns

### Reusable algorithms in a library

```csharp
public static class Numerics {
    public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> { ... }
    public static T Average<T>(IEnumerable<T> source) where T : INumber<T> { ... }
    public static T StdDev<T>(IEnumerable<T> source) where T : INumber<T>, IRootFunctions<T> { ... }
    public static T Median<T>(IList<T> sorted) where T : INumber<T> { ... }
}
```

### Constraints à la carte

If your algorithm only needs addition + zero:

```csharp
public static T Sum<T>(IEnumerable<T> source)
    where T : IAdditionOperators<T, T, T>, IAdditiveIdentity<T, T>
{
    T total = T.AdditiveIdentity;
    foreach (var x in source) total += x;
    return total;
}
```

Now even types that aren't full `INumber<T>` can use it (e.g., a `TimeSpan`-like type that has addition but not multiplication).

### Generic min/max with comparer

```csharp
public static T Min<T>(IEnumerable<T> source, IComparer<T> comparer) {
    using var e = source.GetEnumerator();
    if (!e.MoveNext()) throw new();
    T best = e.Current;
    while (e.MoveNext()) if (comparer.Compare(e.Current, best) < 0) best = e.Current;
    return best;
}
```

Pre-generic-math, this was the standard approach. Now you can use `IComparisonOperators<T, T, bool>` for direct operator dispatch.

---

## Common bugs

- **Forgetting `T.Zero` exists** — many people manually `default(T)` for "zero." It works for int (default is 0) but not all numeric types (decimal default is 0 too, but other custom types' defaults might not be additive identity).
- **Using `Convert.ChangeType` instead of `T.CreateChecked`** — the BCL way is faster and clearer.
- **Conversion choice mismatch** — `CreateChecked` throws on overflow; `CreateSaturating` clamps; `CreateTruncating` truncates. Pick consciously.
- **Mixing types** — `T + int` doesn't work in generic context without proper constraints. Use `T.CreateChecked(intValue)` to convert.

---

## When generic math is too much

For one-off arithmetic on one type, just use the type. Generic math is for **library and framework** code that benefits from reuse across types.

For two or three numeric types, manual overloading is often clearer than constraints. Generic math wins when you have many implementers or you're building a generic library.

---

## Performance summary

- Generic math is **as fast** as hand-written code for primitive Ts (JIT specialization).
- For reference Ts, slightly higher dispatch cost; still fast.
- Conversion via `CreateChecked` is essentially the same speed as raw cast for primitives.
- Combine with `Span<T>` and SIMD for maximum throughput.

---

## Quick reference

| If you need | Use interface | Get |
|---|---|---|
| Addition | `IAdditionOperators<T, T, T>` | `+` |
| Multiplication | `IMultiplyOperators<T, T, T>` | `*` |
| Comparison | `IComparisonOperators<T, T, bool>` | `<`, `>`, etc. |
| Both equality + comparison | `INumber<T>` | the works |
| Convert from int / double | `INumberBase<T>` | `CreateChecked`, `CreateSaturating`, `CreateTruncating` |
| Identity values | `IAdditiveIdentity<T, T>` / `IMultiplicativeIdentity<T, T>` | `Zero`, `One` |
| Min/Max values | `IMinMaxValue<T>` | `MinValue`, `MaxValue` |
| Generic parse | `IParsable<T>` / `ISpanParsable<T>` | `Parse`, `TryParse` |

→ Next: [07-DefaultLiteralAndT.md](07-DefaultLiteralAndT.md)
