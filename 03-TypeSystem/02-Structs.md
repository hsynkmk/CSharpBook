# Structs

## What it is

A `struct` defines a **value type**. Instances live where they're declared — on the stack, inside another struct/class, in an array slot. They're copied on assignment. They can't be `null` (unless wrapped in `Nullable<T>`). They don't support inheritance.

```csharp
public struct Point {
    public int X { get; init; }
    public int Y { get; init; }
}

var p = new Point { X = 3, Y = 4 };
```

Structs are how you get C-like value semantics in a managed language. Done right, they're cheap and fast. Done wrong, they're a footgun.

---

## Why it exists

The CLR needs to support tiny scalar-like types (`int`, `double`, etc.) cheaply. Those are built-in structs. The language exposes the same mechanism so you can build your own value types:

- Geometric primitives — `Point`, `Vector`, `Color`.
- Domain values — `Money`, `Duration`, `Temperature`, `Percentage`.
- Performance optimizations — keys in a hot dictionary, elements of a struct-of-arrays buffer.
- Span-like views — `Span<T>`, `ReadOnlySpan<T>`.

The right scenario is **small, immutable, value-semantic**. Anything bigger or mutable, and you should reach for a class.

---

## Declaring a struct

```csharp
public struct Money {
    public decimal Amount;
    public string Currency;
}

public struct Money(decimal amount, string currency) {   // C# 12+ primary ctor
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;
}

public readonly struct Money(decimal amount, string currency) {  // immutable
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;
}
```

Struct rules:
- Cannot inherit from another struct or class. (Implicitly inherits `System.ValueType`.)
- Can implement interfaces.
- Cannot be `abstract` or `sealed` (sealed implicitly; abstract has no meaning).
- Pre-C# 10: no parameterless constructor. C# 10+ allows them, but field initializers run only when a constructor is called.

---

## Copy semantics

```csharp
var p1 = new Point { X = 3, Y = 4 };
var p2 = p1;          // bitwise copy
p2 = p2 with { X = 99 }; // change only p2 (using `with` requires record struct)
Console.WriteLine(p1.X); // 3 — independent
```

Every assignment, parameter pass, return — copies the bytes. A 100-byte struct copies 100 bytes. That's why "keep structs small" matters.

---

## `readonly struct`

A struct marked `readonly` cannot mutate any of its fields after construction. The compiler enforces:
- All instance fields must be `readonly`.
- Auto-property setters must be `init` (not `set`).
- Instance methods receive `this` as a read-only reference.

```csharp
public readonly struct Vector {
    public double X { get; init; }
    public double Y { get; init; }
    public Vector(double x, double y) { X = x; Y = y; }
    public double Magnitude => Math.Sqrt(X * X + Y * Y);
    public Vector Scale(double k) => new(X * k, Y * k);   // returns new instance
}
```

Benefits:
- The compiler can elide defensive copies (see below).
- Static guarantee — no `Scale` method can accidentally mutate `this`.
- Clear intent for callers.

**Make every value-type "data" struct `readonly`** unless you have a strong reason. Modern .NET libraries do this universally.

---

## `readonly` member modifier

If you want **just one method** to promise it doesn't mutate `this`, mark the method `readonly`:

```csharp
public struct Counter {
    public int Value;
    public readonly int Peek() => Value;       // doesn't mutate
    public void Increment() { Value++; }       // mutates
}
```

Useful when you have a mostly-mutable struct with a few read-only operations. But: if you ever find yourself wanting one read-only method on a struct, consider making the **whole struct** readonly.

---

## The defensive copy trap

When you call a non-`readonly` instance method on a `readonly` field/variable, the compiler **copies** the struct first, calls the method on the copy, then discards it. This prevents the method from mutating the readonly storage but creates a hidden allocation and an obscure performance pitfall:

```csharp
public class Container {
    public readonly Counter Counter = new();
    public void Bump() {
        Counter.Increment();   // ⚠ silently copies, then calls Increment on the COPY
        Console.WriteLine(Counter.Value);  // 0 — original unchanged
    }
}
```

This is a famous, hard-to-spot bug. **The fix**: mark `Counter` itself `readonly struct`, OR mark `Increment` `readonly` (if it doesn't actually mutate), OR don't use `readonly` on the field. Compiler warnings (CS8656) help, and modern analyzers flag the pattern.

---

## `ref struct`

A struct marked `ref struct` is **stack-only**. The compiler prohibits any operation that could put it on the heap:

- Cannot be a field of a class or non-`ref` struct.
- Cannot be an array element.
- Cannot be boxed.
- Cannot be a captured local of a lambda or local function.
- Cannot cross an `await` or `yield return`.
- Cannot be used as a generic type argument (until C# 13 with `allows ref struct`).

```csharp
public ref struct StackBuffer {
    public Span<byte> Data;
}
```

Examples in the BCL:
- `Span<T>` and `ReadOnlySpan<T>`.
- `Utf8JsonReader`.
- `SpanWriter<T>`.

Why all the restrictions? `ref struct` exists to hold **pointers into the stack or pinned memory**. Letting one escape to the heap could leave a dangling pointer after the stack frame returns. The compiler enforces the safe usage.

Full coverage in [Chapter 09 §09](../09-MemoryPerformance/09-RefStructsRefLocals.md).

---

## `record struct` (C# 10+)

A struct with synthesized value equality, `ToString`, deconstruction, and (optionally) `with` expressions:

```csharp
public readonly record struct Point(int X, int Y);

var p = new Point(1, 2);
var q = p with { Y = 99 };       // new struct with Y replaced
Console.WriteLine(p == q);       // false — value equality
Console.WriteLine(p);            // Point { X = 1, Y = 2 }
```

The `record struct` is, in 90% of cases, **the right way to declare a small value type today**. You get all the benefits of a struct plus the conveniences of a record.

```csharp
// Recommended
public readonly record struct Money(decimal Amount, string Currency);

// vs the verbose version:
public struct Money : IEquatable<Money> {
    public decimal Amount { get; init; }
    public string Currency { get; init; }
    public Money(decimal amount, string currency) { ... }
    public bool Equals(Money other) => Amount == other.Amount && Currency == other.Currency;
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public override string ToString() => $"Money {{ Amount = {Amount}, Currency = {Currency} }}";
    public static bool operator ==(Money a, Money b) => a.Equals(b);
    public static bool operator !=(Money a, Money b) => !a.Equals(b);
}
```

Same behavior, vastly less code.

---

## Boxing

Storing a value type in a variable of type `object` (or an interface) **boxes** it — wraps the value in a heap-allocated object. Boxing costs an allocation; unboxing requires the runtime type to match exactly.

```csharp
int n = 42;
object o = n;      // boxing: heap allocation
int back = (int)o; // unboxing: must match type exactly
```

Common (silent) box sources:
- Assigning a struct to `object` / `dynamic`.
- Passing a struct to a method expecting `object`.
- Calling a non-generic interface method on a struct (`((IComparable)myStruct).CompareTo(...)`).
- `string.Format("{0}", myStruct)` — boxes for each `{n}`.

Boxing is the #1 performance trap with structs. [Chapter 03 §07](07-BoxingUnboxing.md) goes deep.

---

## Equality

By default, struct equality is **field-by-field via reflection** — and SLOW. The runtime walks the fields and compares them.

```csharp
struct Coord { public int X, Y; }
var a = new Coord { X = 1, Y = 2 };
var b = new Coord { X = 1, Y = 2 };
a.Equals(b);   // true, but via reflection — slow on hot paths
```

For any struct used as a dictionary key, hash set element, or compared in tight loops, **implement `IEquatable<T>`** (and `GetHashCode`):

```csharp
struct Coord : IEquatable<Coord> {
    public int X, Y;
    public bool Equals(Coord other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Coord c && Equals(c);
    public override int GetHashCode() => HashCode.Combine(X, Y);
}
```

Or — much shorter — use `record struct`:

```csharp
public readonly record struct Coord(int X, int Y);
```

Records implement `IEquatable<T>` for you.

---

## Default value

Every struct has a default value: all fields zeroed.

```csharp
Point p = default;
Console.WriteLine(p);   // (0, 0) or whatever your ToString prints
```

You **cannot prevent** the default. Even if you have only a parameterized constructor, callers can always do `default(MyStruct)` and get one without going through your code.

This is a famous gotcha. If your struct has an invariant that can't hold for all-zeros (e.g., "currency must be non-null"), you have to design around it — invariants that can't be enforced at construction time mean you can't enforce them at all.

```csharp
public readonly record struct Money(decimal Amount, string Currency);
Money m = default;   // Amount=0, Currency=null!
m.Currency.ToUpper();   // 💥 NullReferenceException
```

A class makes this much harder to hit by accident (null check or NRT warns you).

---

## Internals — how structs are stored

### Layout

A struct's fields are laid out sequentially, with **alignment padding** between them on most platforms:

```csharp
struct Mixed {
    public byte A;
    public int B;
    public byte C;
}
```

On 64-bit .NET:
```
offset 0: A (1 byte)
offset 1: 3 bytes padding
offset 4: B (4 bytes)
offset 8: C (1 byte)
offset 9: 3 bytes padding
total:    12 bytes (rounded to multiple of 4)
```

The CLR can **reorder fields** to pack better (or skip reordering if you specify `[StructLayout(LayoutKind.Sequential)]`, common for interop).

```csharp
[StructLayout(LayoutKind.Explicit)]
struct U {
    [FieldOffset(0)] public int Asint;
    [FieldOffset(0)] public float AsFloat;   // shares the same 4 bytes
}
```

Used for low-level reinterpret-cast tricks.

### On the stack vs in registers

For very small structs (often ≤16 bytes), the JIT may keep them entirely **in CPU registers** — no memory location at all. This is one reason `Span<T>` and `ReadOnlySpan<T>` are so fast: a single span (pointer + length) easily fits in two registers.

### Inside a class

A struct field of a class lives **inside the class's heap allocation**, not separately:

```csharp
class Container { public Point P; }
```

Memory of `new Container()`:
```
[ sync block | MT ptr | P.X | P.Y ]
```

No separate allocation for `P`. That's the per-instance overhead win.

### Boxing in IL

```csharp
int n = 5;
object o = n;
```

IL:
```il
ldc.i4.5
box [System.Runtime]System.Int32
stloc.0
```

The `box` instruction allocates a small heap object holding the int and a method-table pointer. Now `o` references that object.

Unboxing:
```csharp
int back = (int)o;
```

IL:
```il
ldloc.0
unbox.any [System.Runtime]System.Int32
stloc.1
```

`unbox.any` verifies the heap object's MT matches the target type, then copies the value back out. Mismatch → `InvalidCastException`.

### `record struct` synthesized members

For `public readonly record struct Point(int X, int Y);` the compiler generates:

- `public int X { get; init; }`
- `public int Y { get; init; }`
- A constructor taking X and Y.
- A `Deconstruct(out int X, out int Y)` method.
- `bool Equals(Point other)`, override of `Equals(object?)`, `GetHashCode`, `ToString`.
- `==` / `!=` operators.
- `IEquatable<Point>` implementation.

All inline, no boxing, no virtual dispatch. The JIT inlines aggressively.

---

## When to use a struct

✓ Logically a single value (a point, a date, a duration).
✓ Small (~16 bytes or less).
✓ Immutable (or carefully controlled mutation).
✓ Doesn't get boxed in hot paths.
✓ Equality is value-based.

✗ Has identity that should persist across copies.
✗ Has lots of fields (you'd copy 100+ bytes constantly).
✗ Implements interfaces and gets passed around as the interface (boxes).
✗ Captured by lambdas (hoists to heap-allocated closure class).
✗ Needs inheritance.

---

## Common bugs

- **Mutable struct in a collection** — `list[0].X = 5` doesn't compile; `dict[key].X = 5` doesn't compile. Indexers return copies.
- **Default value violates invariants** — `default(Money).Currency` is null.
- **Boxing in hot paths** — assigning a struct to `object`, calling non-generic interface methods.
- **Defensive copies from `readonly` field + non-readonly method** — silent perf loss.
- **Implementing `IEquatable<T>` partially** — implement `IEquatable<T>`, override `Equals(object?)`, override `GetHashCode`, and (ideally) overload `==`/`!=`. All four together.

---

## Performance notes

- Stack allocation is free. Heap allocation is cheap but pressures the GC.
- Field-by-field reflection-based equality is ~10–100× slower than a hand-written `IEquatable<T>`.
- Boxing allocates ~24 bytes (16 header + value) on 64-bit. In a tight loop this adds up.
- For pass-by-value of a 64+ byte struct, use `in` to pass by read-only reference: `void Process(in BigStruct s)`.
- Use `record struct` over hand-written struct to get optimized equality for free.

→ Next: [03-Records.md](03-Records.md)
