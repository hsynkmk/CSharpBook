# Static Abstract Members in Interfaces (C# 11+)

## What it is

Since C# 11 (2022), interfaces can declare **static abstract members** — static methods, properties, and operators that implementers must provide. Combined with generics, this enables a contract-based form of polymorphism on *static* things, which previously had no abstraction story in C#.

```csharp
public interface IParseable<T> where T : IParseable<T> {
    static abstract T Parse(string s);
}

public class MyNumber : IParseable<MyNumber> {
    public int Value { get; init; }
    public static MyNumber Parse(string s) => new() { Value = int.Parse(s) };
}

// Generic method using static abstract dispatch
public static T ParseAndUse<T>(string s) where T : IParseable<T> =>
    T.Parse(s);

MyNumber n = ParseAndUse<MyNumber>("42");
```

Notice `T.Parse(s)` — calling a *static method* through a generic type parameter. That's the new ability.

---

## Why it exists

Before C# 11, you couldn't write a generic algorithm like "sum all the numbers" that worked across `int`, `double`, `decimal` without:
- Manual dispatch / dynamic.
- A library wrapper interface with instance methods (boxes value types).
- Per-type duplication.

The root problem: operators (`+`, `*`, ...) and `Parse`, `MinValue`, `MaxValue`, etc., were **static** — interfaces couldn't require them.

Static abstract members fix this. They unlock:
- **Generic math** — `INumber<T>`, `IAdditionOperators<...>`, etc. — see [§06](06-GenericMath.md).
- **Generic parsing** — `IParsable<T>`.
- **Factory contracts** — `IDefault<T>` for default-value creation.
- **Custom number-like types** — your own currency, vector, matrix can plug into generic algorithms.

---

## Declaring a static abstract member

Inside an interface:

```csharp
public interface IShape<T> where T : IShape<T> {
    static abstract T Default();
    static abstract string TypeName { get; }
    static abstract T operator +(T a, T b);
}
```

You can declare static abstract:
- **Methods** (most common).
- **Properties** (read-only or read-write).
- **Operators** (including conversion operators).
- **Events** (rare).
- **Indexers**: no — they're instance only.

The `where T : IShape<T>` constraint is the **curiously recurring** pattern — T constrains itself. This lets `T.Default()` and `T operator +(T, T)` use T concretely.

---

## The implementer side

A class or struct implementing the interface provides static implementations:

```csharp
public class Square : IShape<Square> {
    public double Side { get; init; }

    public static Square Default() => new() { Side = 1 };
    public static string TypeName => "Square";
    public static Square operator +(Square a, Square b) => new() { Side = a.Side + b.Side };
}
```

The static members satisfy the interface's static abstract requirements.

---

## How calls dispatch

A static abstract call goes through a **type parameter**:

```csharp
public static T Make<T>() where T : IShape<T> =>
    T.Default();    // Notice: 'T.Default', not 'this' or 'someInstance'

Square s = Make<Square>();   // calls Square.Default()
```

`T.Default()` at the call site resolves to the static method on whatever type T is at runtime. The JIT generates one body per value-type T (specialized) and shares a body across reference-type Ts (with a runtime token).

Compare to instance virtual dispatch — which goes through a vtable on the object — static dispatch goes through the **type token**. Both happen at runtime; both work over generics.

---

## `static virtual` vs `static abstract`

You can also declare `static virtual` members in interfaces — same idea, but with a default implementation:

```csharp
public interface ICountable<T> where T : ICountable<T> {
    static abstract T Empty { get; }
    static virtual int Total => -1;   // default; implementers may override
}
```

`static virtual` is rarely used directly. `static abstract` is the workhorse.

---

## Generic math example

The standout use case: writing math algorithms that work over any numeric type.

```csharp
using System.Numerics;   // for INumber<T>, IAdditionOperators<>, etc.

public static class MathHelpers {
    public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
        T total = T.Zero;
        foreach (var x in source) total += x;
        return total;
    }

    public static T Average<T>(IEnumerable<T> source) where T : INumber<T> {
        T total = T.Zero;
        int count = 0;
        foreach (var x in source) { total += x; count++; }
        if (count == 0) throw new InvalidOperationException();
        return total / T.CreateChecked(count);
    }
}

int i = MathHelpers.Sum(new[] { 1, 2, 3 });            // works
double d = MathHelpers.Sum(new[] { 1.5, 2.5 });        // works
decimal m = MathHelpers.Sum(new[] { 1.5m, 2.5m });     // works
MyMoney mm = MathHelpers.Sum(new[] { /* MyMoney */ }); // works if MyMoney : INumber<MyMoney>
```

`INumber<T>` requires all the static abstract operators (`+`, `-`, `*`, `/`, `<`, etc.) plus `Zero`, `One`, `MinValue`, etc. The BCL implements `INumber<T>` for every built-in numeric type — so generic math is opt-in for users but works out of the box with primitives.

[§06](06-GenericMath.md) covers `INumber<T>` and friends.

---

## Generic factories

```csharp
public interface IDefault<T> {
    static abstract T Default { get; }
}

public class Settings : IDefault<Settings> {
    public int Retries { get; init; }
    public TimeSpan Timeout { get; init; }
    public static Settings Default => new() { Retries = 3, Timeout = TimeSpan.FromSeconds(30) };
}

public static T Build<T>() where T : IDefault<T> => T.Default;

var s = Build<Settings>();   // calls Settings.Default
```

Useful when you need "create a fresh T" but `new()` isn't enough (T's constructor needs values).

---

## Conversion operators

You can require static conversion operators:

```csharp
public interface IParsable<T> where T : IParsable<T> {
    static abstract T Parse(string s, IFormatProvider? provider);
    static abstract bool TryParse(string? s, IFormatProvider? provider, out T result);
}
```

This is the actual BCL interface (modulo `out` and method signature details). Built-in types implement it; you can too.

```csharp
public static T ReadConfig<T>(string s) where T : IParsable<T> =>
    T.Parse(s, CultureInfo.InvariantCulture);

int n = ReadConfig<int>("42");
double d = ReadConfig<double>("3.14");
DateTime dt = ReadConfig<DateTime>("2026-05-19");
```

---

## Internals — how static abstract dispatch works

This is the deepest of the static abstract corners; you don't usually need to know it. But for the curious:

At the IL level, calling `T.Method()` (where T is constrained to `IFoo<T>` with `static abstract Method`) compiles to:

```il
call !!T class IFoo`1<!!T>::Method()
```

The `call` (not `callvirt`) means **direct static dispatch**. But the target is a type parameter (`!!T`), not a known type. The JIT resolves this:

- For value-type T: the JIT specializes the code per T. The target becomes a known static method address. Direct call, often inlined.
- For reference-type T: the JIT uses a "method dispatch token" tied to T's identity. One body of code is shared across all reference Ts; the method lookup happens once per T per JIT specialization.

The runtime introduces an intermediate **frozen MT** for static abstract methods — essentially a per-type lookup table mapping interface static methods to their implementations.

The performance: roughly **as fast as a direct call** for value-type T. For reference-type T, slightly slower due to the dispatch token, but still much faster than reflection-based alternatives.

### IL flag

A static abstract method in an interface has metadata flags:
```il
.method public hidebysig static abstract virtual newslot
    !!T Default() cil managed
```

The `virtual` keyword on a static method is the new bit — usually static methods can't be virtual. Static-abstract-virtual is the runtime model for this dispatch.

---

## When static abstract becomes overengineering

Static abstract isn't always the right tool. Alternatives:
- A `Func<>` or delegate — simpler, no constraint plumbing. Works when you just need "create a T given some inputs."
- A factory class — `IRepositoryFactory<TEntity>` — explicit and easy to mock.
- Just having the consumer pass in the value: `Sum(source, T.Zero)` — sometimes the most readable.

Use static abstract when:
- You need to call **operators** (where delegate alternatives don't work).
- You're building a math-like or numeric-like API.
- The type itself is "the thing" being parameterized over, not just a state machine.

---

## Common patterns

### Generic conversion

```csharp
public static T Convert<T, U>(U value)
    where T : INumberBase<T>
    where U : INumberBase<U>
{
    return T.CreateChecked(value);
}

double d = Convert<double, int>(42);
```

### Generic equality

```csharp
public interface IComparable2<T> where T : IComparable2<T> {
    static abstract int Compare(T a, T b);
}

public static T Max<T>(T a, T b) where T : IComparable2<T> =>
    T.Compare(a, b) >= 0 ? a : b;
```

(In practice, just use `IComparable<T>` for ordering — the BCL already has it. This is for educational purposes.)

### Generic units

```csharp
public interface IUnitOf<T, TUnit> where T : IUnitOf<T, TUnit> {
    static abstract T FromUnit(double value);
    double Value { get; }
}

public readonly record struct Meters(double Value) : IUnitOf<Meters, Meters> {
    public static Meters FromUnit(double v) => new(v);
}

public readonly record struct Feet(double Value) : IUnitOf<Feet, Feet> {
    public static Feet FromUnit(double v) => new(v);
}
```

A type-safe units-of-measure system, leveraging static abstract for unit construction.

---

## Common bugs

- **Forgetting the curiously recurring constraint** — `where T : IFoo<T>` ensures T can be the target type of static members like `T.Default()`. Without it, you can't refer to T concretely.
- **`T.MemberName` when T isn't constrained appropriately** — compile error. Add the constraint.
- **Mixing static abstract with regular interface methods** — fine, but think about what's instance vs class-level.
- **Static abstract on reference types vs value types** — works for both, but the dispatch costs differ (slightly).

---

## Performance

- Value type T → effectively direct static call. As fast as hand-written.
- Reference type T → slight dispatch overhead, much faster than reflection.
- Generic math algorithms compile to per-T specializations. For hot numeric code, this is competitive with hand-tuned versions.

---

## When to use

✓ Building a generic numeric or math API.
✓ Generic parsing (`IParsable<T>`) — works for any type with `Parse`.
✓ Generic factories where `new()` isn't enough.
✓ Self-describing types — `T.TypeName`, `T.Empty`, etc.
✓ When you find yourself reaching for `Activator.CreateInstance` and want a typed alternative.

✗ For one-off "create a T" — a `Func<T>` delegate is simpler.
✗ Mocking — static abstract members can't be easily mocked (the implementing type is concrete).
✗ When the abstraction is over instances, not types — use a regular interface.

---

## Quick reference

| Feature | Available |
|---|---|
| Static abstract method | ✓ |
| Static abstract property | ✓ |
| Static abstract operator | ✓ |
| Static abstract event | ✓ |
| Static virtual (default) | ✓ |
| Static abstract indexer | ✗ |
| Default interface methods + static abstract | ✓ |

→ Next: [06-GenericMath.md](06-GenericMath.md)
