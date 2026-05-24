# Primitive Types

## What it is

Every variable in C# has a **type**. Types are either:
- **Value types** — `int`, `double`, `bool`, `char`, structs. Stored where they're declared.
- **Reference types** — `string`, `object`, classes, arrays, delegates. Stored on the heap; variables hold references.

This file covers the **built-in primitive types** — the ones with C# keywords for them. They map 1-to-1 to types in the `System` namespace.

| C# keyword | .NET type | Size | Range / notes |
|---|---|---|---|
| `bool` | `System.Boolean` | 1 byte | `true` or `false` |
| `byte` | `System.Byte` | 1 byte | 0 to 255 (unsigned) |
| `sbyte` | `System.SByte` | 1 byte | −128 to 127 |
| `char` | `System.Char` | 2 bytes | A single UTF-16 code unit (≈ "character") |
| `short` | `System.Int16` | 2 bytes | −32,768 to 32,767 |
| `ushort` | `System.UInt16` | 2 bytes | 0 to 65,535 |
| `int` | `System.Int32` | 4 bytes | ±~2.1 billion |
| `uint` | `System.UInt32` | 4 bytes | 0 to ~4.3 billion |
| `long` | `System.Int64` | 8 bytes | ±~9.2 quintillion |
| `ulong` | `System.UInt64` | 8 bytes | 0 to ~18.4 quintillion |
| `float` | `System.Single` | 4 bytes | ~7 significant decimal digits |
| `double` | `System.Double` | 8 bytes | ~15-17 significant decimal digits |
| `decimal` | `System.Decimal` | 16 bytes | 28-29 significant decimal digits, exact for base 10 |
| `nint` | `System.IntPtr` | platform | 4 bytes on 32-bit, 8 on 64-bit |
| `nuint` | `System.UIntPtr` | platform | 4 bytes on 32-bit, 8 on 64-bit |

Plus `string` (reference type — covered in [§03](03-StringsAndChars.md)) and `object`.

`int x = 5;` and `Int32 x = 5;` are exactly the same — the C# keyword is just sugar.

---

## Integers

The most common types. Use **`int`** by default; `long` when you need >2 billion; `byte`/`short` only for arrays where size matters (e.g., image data).

```csharp
int    age      = 30;
long   bigNum   = 5_000_000_000L;   // L suffix marks the literal as long
byte   green    = 0;
short  signed16 = -1000;
```

### Numeric literals

- Underscores allowed as separators: `1_000_000` is `1000000`.
- Suffixes: `L` (long), `UL` (ulong), `U` (uint).
- Bases: `0x` for hex, `0b` for binary, decimal is default.

```csharp
int    decimal_  = 255;
int    hex       = 0xFF;       // 255
int    binary    = 0b1111_1111; // 255
long   bigHex    = 0xFFFF_FFFF_FFFF_FFFFL;
ulong  bigger    = 0xFFFF_FFFF_FFFF_FFFFUL;
```

### Overflow

Default integer arithmetic **wraps around silently**:

```csharp
int max = int.MaxValue;          // 2_147_483_647
int next = max + 1;              // -2_147_483_648 — overflow, no error!
Console.WriteLine(next);         // -2147483648
```

To get an exception on overflow, wrap in `checked`:

```csharp
checked {
    int wraps = int.MaxValue + 1;   // throws OverflowException
}
```

There's also `unchecked` (the default; rarely written explicitly), and `checked(...)` / `unchecked(...)` as expressions. You can change the project-wide default with `<CheckForOverflowUnderflow>` in csproj — but the default is `unchecked` for performance.

Casting between integer types uses the same overflow rules:

```csharp
int big = 300;
byte small = (byte)big;     // 44 — wrapped (300 - 256)
byte safer = checked((byte)big);   // throws OverflowException
```

### Min/Max constants

Every numeric type has `MinValue` and `MaxValue`:

```csharp
Console.WriteLine(int.MaxValue);    // 2147483647
Console.WriteLine(long.MinValue);   // -9223372036854775808
Console.WriteLine(byte.MaxValue);   // 255
```

---

## Floating point

`float`, `double`, and `decimal`. They are NOT the same.

### `float` and `double` — binary floating point

Use IEEE 754 binary representation. **Fast, but inexact** for many decimal values.

```csharp
double d = 0.1 + 0.2;       // 0.30000000000000004
float  f = 0.1f + 0.2f;     // 0.3 (less precision; f suffix marks literal as float)
```

That weirdness is fundamental — `0.1` in binary is a repeating fraction, like `1/3` in decimal. You can't represent it exactly in finite bits.

When to use:
- **`double`** — default for scientific and statistical work. ~15 digits, fast on modern CPUs.
- **`float`** — half the memory, half the precision. Used for graphics/3D math where the loss is acceptable and you need many of them.

When NOT to use:
- **Money**. Don't store dollars as `double` — the rounding errors accumulate and accountants get angry. Use `decimal`.

### `decimal` — base-10 arithmetic

Stored as `(int sign, ulong mantissa, byte exponent)` — base-10 internally. Slower (~10× slower than `double` on most CPUs) but **exact for decimal fractions**:

```csharp
decimal d = 0.1m + 0.2m;    // exactly 0.3 (m suffix marks literal as decimal)
decimal price = 19.99m;
decimal tax = price * 0.08m;
```

**Always use `decimal` for currency, accounting, or any user-facing exact-decimal value.**

Range is smaller than `double` (±7.9 × 10²⁸) and precision is ~28-29 significant digits.

### Special floating-point values

`double` and `float` can hold:

```csharp
double posInf = double.PositiveInfinity;   //  ∞
double negInf = double.NegativeInfinity;   // -∞
double nan    = double.NaN;                // not a number (e.g., 0.0/0.0)

double.IsNaN(nan);        // true
double.IsInfinity(posInf); // true
double.IsFinite(0.1);     // true

nan == nan;               // false! NaN is never equal to anything, including itself
double.IsNaN(nan);        // use this instead
```

`decimal` has none of these — it throws `DivideByZeroException` or `OverflowException` instead.

### Comparing floats — never with `==`

Because of binary representation, direct equality is rarely correct:

```csharp
double a = 0.1 + 0.2;
double b = 0.3;
Console.WriteLine(a == b);              // false (!)
Console.WriteLine(Math.Abs(a - b) < 1e-10);  // true
```

Define a tolerance and compare differences. Or use `decimal` if exact equality matters.

---

## `bool`

Two values: `true` and `false`. C# has no implicit conversion from int to bool — `if (5)` doesn't compile, unlike in C or JavaScript.

```csharp
bool isReady = true;
bool isEmpty = items.Count == 0;

if (isReady) { ... }          // ✓
if (5) { ... }                // ❌ compile error
```

`bool` takes 1 byte (not 1 bit) — the runtime can't address sub-byte storage easily.

For arrays of many booleans where memory matters, use `System.Collections.BitArray`.

---

## `char`

A single 16-bit UTF-16 code unit. **Not** a "character" in the human sense.

```csharp
char c = 'A';                  // single quotes
char tab = '\t';               // escape sequences
char nl = '\n';
char unicode = 'é';       // é
char emoji = '😀';             // ❌ — most emoji are TWO code units (surrogate pair)
```

For codepoints outside the Basic Multilingual Plane (which includes most emoji), one `char` is half. Use `Rune` for full Unicode codepoint operations:

```csharp
Rune r = new Rune(0x1F600);    // 😀
string s = r.ToString();
```

[§03](03-StringsAndChars.md) goes deep on strings, chars, and Unicode.

### Char helpers

```csharp
char.IsDigit('5')         // true
char.IsLetter('A')        // true
char.IsLetterOrDigit('_') // false
char.IsWhiteSpace(' ')    // true
char.IsUpper('A')         // true
char.ToUpper('a')         // 'A'
char.ToLower('A')         // 'a'
```

---

## `nint` / `nuint` — native-sized integers

Same width as a pointer on the running platform. `nint` is 4 bytes on 32-bit, 8 bytes on 64-bit. Useful for interop and `Span<T>` index arithmetic.

```csharp
nint offset = 1024;     // platform-sized
```

Maps to `System.IntPtr` / `UIntPtr` which now have arithmetic operators (since .NET 5).

---

## Conversions

### Implicit (safe, no data loss)

```csharp
int   i  = 5;
long  l  = i;     // int → long: always fits
double d = i;     // int → double: but watch for precision loss above ~2^53
```

### Explicit (cast — possible data loss)

```csharp
double d = 3.7;
int i = (int)d;    // 3 — truncates, doesn't round
```

### Parsing from strings

```csharp
int n = int.Parse("42");                     // throws if invalid
bool ok = int.TryParse("42", out int parsed); // doesn't throw — returns success
bool ok2 = int.TryParse("abc", out int n2);  // ok2 = false, n2 = 0

double d = double.Parse("3.14", CultureInfo.InvariantCulture); // explicit culture for portability
decimal m = decimal.Parse("19.99", CultureInfo.InvariantCulture);
```

**Always prefer `TryParse`** unless you genuinely want an exception. And **always pass `CultureInfo.InvariantCulture`** for machine-readable data (config files, network protocols) — otherwise European users with comma decimal separators will break your code.

### Formatting to strings

```csharp
int n = 42;
string s = n.ToString();             // "42"
string s2 = n.ToString("D5");        // "00042"
string s3 = $"{n:N0}";               // "42" (interpolated; N0 = thousands separator)
string s4 = $"{1234.5:N2}";          // "1,234.50"
string s5 = $"{0.05:P}";             // "5.00 %"
string s6 = $"{0xFF:X}";             // "FF"
string s7 = $"{DateTime.Now:yyyy-MM-dd HH:mm:ss}"; // "2026-05-19 14:30:00"
```

[Chapter 13 §08](../13-IO/08-Globalization.md) covers culture and formatting in detail.

---

## `default` and uninitialized fields

```csharp
int x = default;       // 0
bool b = default;      // false
double d = default;    // 0.0
char c = default;      // '\0'
```

For fields (members of a class/struct), default is automatic. For local variables, you must initialize explicitly.

---

## Boxing — when value types pretend to be objects

```csharp
int x = 42;
object o = x;       // boxing: x is wrapped in a heap-allocated object
int y = (int)o;     // unboxing
```

Boxing allocates memory on the heap and is **expensive on hot paths**. Generic constraints (`where T : ...`) usually let you avoid it. Full coverage in [Chapter 03 §07](../03-TypeSystem/07-BoxingUnboxing.md).

---

## When to use which numeric type

| Need | Use |
|---|---|
| General-purpose integer counting | `int` |
| Count that might exceed 2 billion | `long` |
| Memory-tight arrays (bitmap pixels, file bytes) | `byte` |
| Indices for very small buffers | `int` (don't bother saving with `short`) |
| Statistics, sensor data, scientific | `double` |
| 3D graphics, machine learning weights | `float` (size matters) |
| Money, exact decimals | `decimal` |
| Pointer-like arithmetic (interop) | `nint` |
| Boolean flag | `bool` |
| Single Unicode codepoint | `Rune` (not `char`) |

---

## Performance notes

- `int` is the same speed as `long` on 64-bit platforms; `byte`/`short` aren't faster (they're widened to `int` for arithmetic).
- `decimal` is ~10× slower than `double` for math.
- `float` vs `double`: `float` saves memory, but the FPU on modern x64 does double-precision natively. Use `float` for storage; `double` for compute.
- Integer division by a constant is fast — the JIT replaces it with multiply-by-reciprocal.

---

## Common bugs

- **`int.Parse(userInput)` crashes on bad input**. Use `TryParse`.
- **`0.1 + 0.2 != 0.3`** with `double` or `float`. Use `decimal` for exact decimal, or compare with tolerance.
- **`int.MaxValue + 1` silently wraps**. Use `checked` when you can't tolerate it.
- **`Console.WriteLine(0.1)` prints `0.1` but `0.1 + 0.2` prints `0.30000000000000004`**. The literal `0.1` happens to format as `0.1`; the sum of two binary approximations doesn't.
- **`int.Parse("3.14")` throws**. `int.Parse` doesn't read decimals. Use `int.Parse((int)double.Parse("3.14"))` or just `double.Parse`.
- **Mixed types**: `int / int` is integer division. `5 / 2 == 2`, not `2.5`. Force one operand to double: `5.0 / 2`.

→ Next: [03-StringsAndChars.md](03-StringsAndChars.md)
