# Checked / Unchecked, User-Defined Checked Operators

## What it is

`checked` and `unchecked` control how **integer arithmetic** behaves on overflow:

- **`unchecked`** (default): silently wraps (`int.MaxValue + 1` becomes `int.MinValue`).
- **`checked`**: throws `OverflowException` on overflow.

```csharp
int n = int.MaxValue;
int wrap = n + 1;                 // unchecked: int.MinValue (silent wrap)
int crash = checked(n + 1);        // checked: throws OverflowException
checked {
    int crash2 = n + 1;             // also throws
}
```

C# 11 added **user-defined checked operators** — types can define their own `checked` arithmetic, used inside `checked` contexts.

This is a niche but important feature for financial / safety-critical code, generic math, and library design.

---

## Why both modes exist

For most code: silent overflow is fast and rarely matters. The CPU has no built-in cost for unchecked arithmetic.

For financial / counting / safety-critical code: silent overflow can corrupt data. `checked` makes it explicit.

C# picked unchecked as the default for performance. You opt into checked where it matters.

---

## The keywords

### Expression form

```csharp
int a = checked(int.MaxValue + 1);   // throws
int b = unchecked(int.MaxValue + 1);  // wraps
```

Apply to a single expression.

### Block form

```csharp
checked {
    int x = a + b;
    int y = c * d;
}
```

Apply to everything in the block.

### Project-wide setting

In csproj:

```xml
<PropertyGroup>
    <CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
</PropertyGroup>
```

Changes the **default** for arithmetic in your code. You'd then use `unchecked` for the rare safe overflow.

Common in financial / safety domains. Defaults to `false` (unchecked) for most projects.

---

## What triggers checked overflow

Integer arithmetic that produces a value outside the type's range:

```csharp
checked {
    int x = int.MaxValue + 1;       // throws — overflow on add
    int y = (int)long.MaxValue;     // throws — overflow on cast
    byte z = (byte)1000;            // throws — overflow on narrowing cast
    int w = 0 - int.MinValue;       // throws — overflow on negation (MinValue can't be positive)
}
```

The throw happens at the operation site. Use a try/catch (or let it propagate).

---

## What's NOT affected

- **Floating point** (float, double, decimal): `checked` doesn't affect them. `decimal` already throws on overflow; float/double produce infinity or NaN.
- **Pointer arithmetic**: not affected.
- **Bitwise operations**: `&`, `|`, `^`, `<<`, `>>` don't overflow check (they're not arithmetic in the same sense).

```csharp
checked {
    double d = double.MaxValue * 2;   // = double.PositiveInfinity (no throw)
    decimal m = decimal.MaxValue + 1; // throws (decimal always checks)
    int s = -1 << 31;                  // works (bitwise; no overflow check)
}
```

---

## Performance impact

`checked` arithmetic compiles to extra CPU instructions:
- On x86/x64: `add` followed by `jno` (jump on no overflow). ~1 extra cycle.
- For tight loops with millions of ops: ~5-15% overhead.

For typical code: invisible.

For numeric kernels (matrix multiply, image processing): noticeable. Profile if you set the project-wide flag.

---

## User-defined checked operators (C# 11+)

A type can define operators that behave differently in `checked` vs `unchecked` contexts:

```csharp
public struct MyMoney {
    public decimal Amount;

    public static MyMoney operator +(MyMoney a, MyMoney b) =>
        new() { Amount = a.Amount + b.Amount };

    public static MyMoney operator checked +(MyMoney a, MyMoney b) {   // C# 11+
        try { return new() { Amount = a.Amount + b.Amount }; }
        catch (OverflowException) { throw new InvalidOperationException("Money overflow"); }
    }
}

checked {
    var x = m1 + m2;   // calls the `checked +` operator
}
unchecked {
    var y = m1 + m2;   // calls the regular `+` operator
}
```

The `checked` keyword AFTER `operator` declares a separate overload. The compiler picks based on the surrounding context.

Without a `checked` overload, the regular operator is used in both contexts.

---

## Why user-defined checked

For domain types that wrap numeric data (Money, Vector, Matrix), you might want:
- Default (unchecked) arithmetic: fast, may overflow silently (acceptable for some domains).
- Checked arithmetic: validates, throws on overflow.

Library authors expose both; consumers pick based on need.

Required by the **generic math** interfaces:

```csharp
public interface ICheckedArithmetic<T> {
    static abstract T operator checked +(T a, T b);
}
```

If your type implements `INumber<T>`, you must provide checked operators for full conformance. Lets generic algorithms opt-in to overflow checking:

```csharp
public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    checked {
        foreach (var x in source) total += x;   // uses checked operator
    }
    return total;
}
```

If T's checked + overflows, exception. The generic code is overflow-safe.

---

## Internals — IL emitted

For `checked(a + b)` where a, b are `int`:

```il
ldarg.0
ldarg.1
add.ovf       // add with overflow check; throws if overflow
```

`add.ovf` is the IL opcode. The JIT emits the platform-specific `add + jno` sequence (or `setno` etc.).

For unchecked:
```il
ldarg.0
ldarg.1
add           // plain add; no overflow check
```

Simple `add` instruction.

For casts:
```csharp
checked((byte)x)
// IL: conv.ovf.u1   // convert with overflow check to byte
```

`conv.*` opcodes have `conv.ovf.*` variants for checked.

For user-defined `operator checked +`:

The compiler emits TWO operators in IL:
```il
.method public static MyMoney op_Addition(MyMoney, MyMoney) { /* unchecked impl */ }
.method public static MyMoney op_CheckedAddition(MyMoney, MyMoney) { /* checked impl */ }
```

The C# compiler at call site picks which to invoke based on `checked` / `unchecked` context.

---

## Common patterns

### Financial code

```csharp
public class Money {
    public decimal Amount { get; }

    public Money(decimal amount) {
        Amount = amount;
    }

    public static Money operator +(Money a, Money b) =>
        new(checked(a.Amount + b.Amount));   // always checked
}
```

For money, always check. The decimal type already throws on overflow; the cast does too.

### Per-operation override

```csharp
int total = 0;
foreach (var x in nums) {
    total = checked(total + x);   // only this op is checked; rest of method is unchecked
}
```

Narrow scope of checked to where it matters.

### Cast checks

```csharp
public byte ToByteOrThrow(int n) => checked((byte)n);
```

Throws on out-of-range; safer than silent truncation.

---

## When to use `checked`

- Financial calculations.
- Counters that must never wrap (transaction IDs, quotas).
- Type conversions where overflow indicates a bug.
- Generic math (when implementing INumber<T>'s checked operators).

When **NOT**:
- Hot CPU loops where overflow is impossible by design.
- Hash functions (overflow is part of the algorithm).
- Bit-packing (you want wraparound).

---

## When to use `unchecked`

- Hashing algorithms that intentionally overflow.
- CPU-intensive math where overflow can't happen given input ranges.
- Encryption / cryptographic primitives that rely on modular arithmetic.

```csharp
public override int GetHashCode() {
    unchecked {
        int hash = 17;
        hash = hash * 23 + X;
        hash = hash * 23 + Y;
        return hash;
    }
}
```

Hash code combination wraps by design. Explicit `unchecked` makes intent clear (and matters if the project has CheckForOverflowUnderflow enabled).

---

## Project-wide flag

```xml
<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
```

With this set:
- Default arithmetic in your code is `checked`.
- You must explicitly use `unchecked { ... }` for intentional wraparound.
- Slower but safer.

For financial / accounting projects: usually worth enabling.

For game engines / numerical code: usually keep default (unchecked).

The .NET BCL is built with `unchecked` default. Doesn't affect your code's behavior, just BCL internal arithmetic.

---

## Cast vs arithmetic

Casts narrowing to smaller types are checked separately:

```csharp
int big = 1000;
byte b1 = (byte)big;             // unchecked: 232 (1000 mod 256)
byte b2 = checked((byte)big);     // throws

// Even in a checked block:
checked {
    byte b3 = (byte)big;          // throws
}
```

The cast itself checks. Same for `long` → `int`, etc.

---

## Common bugs

### Forgetting that `unchecked` is default

```csharp
public int Count() {
    int sum = 0;
    for (int i = 0; i < items.Length; i++) sum += items[i];   // wraps silently if it does
    return sum;
}
```

If overflow matters, wrap in `checked`. Defaults are silent.

### Assuming overflow doesn't happen

```csharp
public int Subtract(int a, int b) => a - b;
// If a = int.MinValue and b = 1, result wraps to int.MaxValue. Silent.
```

For domain operations where the result might exceed int range, use long or checked.

### Floating point doesn't honor checked

```csharp
checked {
    double d = double.MaxValue * 2;   // infinity, no throw
}
```

`checked` only affects integer arithmetic. Float/double overflow → infinity / NaN.

For checked floating-point behavior, use `decimal` (throws on overflow) or check manually with `double.IsInfinity` / `double.IsNaN`.

### User-defined operator only for unchecked

```csharp
public static MyMoney operator +(MyMoney a, MyMoney b) {
    return new(a.Amount + b.Amount);
}
// No 'checked +' overload
```

In a `checked` block, the regular operator is used. If your domain needs different behavior in checked context, define both.

---

## Performance

| Operation | Time |
|---|---|
| Unchecked int add | ~1 cycle |
| Checked int add | ~1-2 cycles (extra branch) |
| Checked cast | ~1-2 cycles |
| User-defined checked operator | depends on impl |

For most code, the difference is invisible. For tight numerical loops, ~5-15% slower in checked mode.

---

## Summary

- `checked` / `unchecked` control integer overflow behavior (throw vs wrap).
- Default is `unchecked` (silent wrap) for performance.
- Per-expression or per-block, or project-wide via `CheckForOverflowUnderflow`.
- Only affects integers — float/double infinities and decimal already throws.
- C# 11+ allows user-defined `operator checked +` for custom types.
- Use checked for financial / safety-critical math; unchecked for hashing / intentional wrap.

→ Continue to: [Questions.md](Questions.md)
