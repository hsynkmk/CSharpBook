# Type Conversions

## What it is

A **conversion** changes a value from one type to another. C# supports several flavors:

- **Implicit conversion** — happens automatically when safe (no data loss possible).
- **Explicit conversion** (cast) — you spell it out; signals possible data loss or runtime check.
- **User-defined conversion** — operator overloads you write on your own types.
- **`as` operator** — safe cast that returns null on failure.
- **`is` operator** — type test (often with pattern).
- **`checked` / `unchecked`** — control overflow behavior for arithmetic conversions.
- **Conversion methods** — `Convert.ToInt32`, `int.Parse`, etc.

```csharp
int i = 5;
long l = i;             // implicit — int → long, always fits
double d = i;           // implicit — int → double (may lose precision above 2^53)
int back = (int)d;      // explicit — could truncate
string s = i.ToString();// method call — formatted text

object o = i;           // implicit (with boxing — see §07)
int? n = i;             // implicit T → Nullable<T>
```

---

## Why it exists

C# is strongly typed. Mixing types without explicit conversions would be:
- **Ambiguous** — what should `5 + "hello"` produce? An int? A string?
- **Bug-prone** — silent narrowing (long → int) can lose data.
- **Implementation-hidden** — when does `Person → string` mean "name", and when does it call `ToString()`?

Conversions force you to be explicit about lossy or unsafe operations, while implicit conversions stay invisible when safe.

---

## Implicit conversions

Automatic, no syntax. Happen when:
- The target type can represent **all** values of the source type.
- The conversion is safe (no possible data loss).

Examples for numerics:

| From | To (implicit) |
|---|---|
| `sbyte` | `short`, `int`, `long`, `float`, `double`, `decimal` |
| `byte` | `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`, `decimal` |
| `short` | `int`, `long`, `float`, `double`, `decimal` |
| `int` | `long`, `float`, `double`, `decimal` |
| `long` | `float`, `double`, `decimal` |
| `float` | `double` |
| `char` | `int`, `long`, `float`, `double` (and others — char is a 16-bit unsigned) |

For reference types:
- **Up the inheritance chain**: `Derived → Base → object`.
- **To implemented interfaces**: `MyList → IEnumerable<T>`.

```csharp
List<int> list = new();
IEnumerable<int> seq = list;   // implicit — List implements IEnumerable
object o = list;                // implicit — everything is an object
```

For value types:
- **Boxing**: implicit cast from `T` to `object` / interface (allocates — see §07).
- **Wrap in `Nullable<T>`**: `int → int?` is implicit.

```csharp
int n = 5;
object boxed = n;     // implicit, but boxes!
int? maybe = n;       // implicit
```

### Implicit conversion to `string` does NOT exist

```csharp
int n = 5;
string s = n;     // ❌ compile error — no implicit conversion

string s2 = n.ToString();   // explicit
string s3 = $"{n}";          // interpolation
string s4 = (string)(object)n;  // boxes then... still doesn't work — InvalidCastException
```

C# has no auto-coerce to string. You always opt in.

---

## Explicit conversions (casts)

When implicit conversion isn't safe, you must spell it out:

```csharp
long bigNum = 100_000_000_000L;
int truncated = (int)bigNum;   // explicit — overflow possible

double pi = 3.14;
int rounded = (int)pi;          // truncates to 3 (NOT rounding!)

short s = 256;
byte b = (byte)s;               // 0 — wrapped (256 mod 256)
```

Behavior on overflow:
- **Integer to integer**: wraps silently in `unchecked` (default), throws in `checked`.
- **Float to integer**: truncates toward zero. NaN → 0. Out of range → implementation-defined behavior in older versions; explicit MaxValue/MinValue saturation in modern .NET.
- **Larger float to smaller**: rounds to nearest, ties to even (IEEE 754).

### `checked` and `unchecked`

```csharp
int max = int.MaxValue;
int next1 = max + 1;                  // unchecked (default): wraps to int.MinValue
int next2 = checked(max + 1);         // throws OverflowException

// Block form:
checked {
    int n = max + 1;       // throws
    int m = (int)1e20;     // also throws — out of range from double
}

unchecked {
    int n = max + 1;       // explicitly silent
}
```

The default in C# is **unchecked** for performance. You can flip the project default:

```xml
<CheckForOverflowUnderflow>true</CheckForOverflowUnderflow>
```

In financial / safety-critical code, consider doing this and explicitly marking known-safe expressions `unchecked`.

### Reference type casts

```csharp
object o = "hello";
string s = (string)o;            // explicit, succeeds: o is actually a string
int n = (int)o;                  // throws InvalidCastException

Animal a = new Dog();
Dog d = (Dog)a;                  // OK — a is actually a Dog
```

Casts down the type hierarchy verify at runtime. Failure throws.

### `as` — safe cast

For reference types and nullable value types only. Returns null on failure instead of throwing:

```csharp
object o = "hello";
string? s = o as string;        // "hello"
int? n = o as int?;             // null — o isn't an int (or Nullable<int>)

Animal a = new Dog();
Dog? d = a as Dog;              // OK
Cat? c = a as Cat;              // null
```

`as` is faster than a try/catch around a cast, and clearer in intent.

Use:
```csharp
if (o is string s2) { ... }     // modern preferred form
```

vs:
```csharp
string? s3 = o as string;
if (s3 is not null) { ... }
```

Both work; the `is` pattern is usually cleaner.

### `is` — type test

```csharp
if (obj is string s) Console.WriteLine(s.Length);
if (obj is int n && n > 0) { ... }
if (obj is null) { ... }
if (obj is not null) { ... }
if (obj is Point { X: 0, Y: 0 }) { ... }   // property pattern
```

Returns `bool`. The pattern-introduced variable (`s`, `n`) is in scope inside the `if`.

---

## User-defined conversions

You can define **implicit** or **explicit** conversion operators on your own types:

```csharp
public readonly struct Celsius {
    public double Degrees { get; }
    public Celsius(double d) { Degrees = d; }

    // Implicit Celsius → double (you'd only do this if it really feels safe and natural)
    public static implicit operator double(Celsius c) => c.Degrees;

    // Explicit double → Celsius (you've chosen to require a cast)
    public static explicit operator Celsius(double d) => new(d);
}

Celsius c = new(20);
double d = c;                  // implicit — works automatically
Celsius c2 = (Celsius)25.0;    // explicit — cast required
```

When to use which:
- `implicit` — only if the conversion is truly safe and free of surprises.
- `explicit` — for anything that's a "view" rather than "the same thing."

Examples in the BCL:
- `int → long`: implicit (always safe).
- `long → int`: explicit (may overflow).
- `string → int`: not an operator — use `int.Parse` (failure isn't fine).
- `DateTime → DateTimeOffset`: implicit.
- `int → BigInteger`: implicit.

### Operator overloading example — `Money` with conversion

```csharp
public readonly record struct Money(decimal Amount, string Currency) {
    public static Money operator +(Money a, Money b) {
        if (a.Currency != b.Currency) throw new InvalidOperationException();
        return new(a.Amount + b.Amount, a.Currency);
    }

    public static implicit operator decimal(Money m) => m.Amount;
    public static explicit operator Money(decimal d) => new(d, "USD");
}

Money m = new(100, "USD");
decimal raw = m;                 // implicit — common projection
Money m2 = (Money)25.0m;          // explicit — losing currency info, requires cast
```

### When to overload `==`

If you write an operator that changes equality semantics (like a record), you should also override `Equals(object?)`, `GetHashCode`, and ideally implement `IEquatable<T>`. Records do all of this for you.

[Chapter 07 §11](../07-Collections/11-EqualityContract.md) covers the equality contract.

---

## Conversion methods (`Convert`, `Parse`, `TryParse`)

For converting strings or arbitrary objects to a specific type:

```csharp
int n = int.Parse("42");                        // throws on failure
bool ok = int.TryParse("42", out int parsed);   // never throws
int safe = Convert.ToInt32("42");               // works for many sources

DateTime d = DateTime.Parse("2026-05-19", CultureInfo.InvariantCulture);
decimal m = decimal.Parse("19.99", CultureInfo.InvariantCulture);
```

`Convert.ToX` methods are flexible — they accept many types (`null` returns 0/false, `string` parses, `bool` converts, etc.). Useful when you have an `object` of unknown type.

Always prefer `TryParse` over `Parse` for user input — exceptions are slow and ungraceful.

[Chapter 13 §08](../13-IO/08-Globalization.md) covers culture and parsing.

---

## Reference vs value conversions side by side

```csharp
// Reference type conversions
Animal a = new Dog();          // implicit upcast
Dog d = (Dog)a;                 // explicit downcast (throws on failure)
Dog? d2 = a as Dog;             // safe downcast
bool isDog = a is Dog;          // type test
if (a is Dog dog) { /* dog scope */ }

// Value type conversions
int i = 5;
long l = i;                     // implicit widening
int back = (int)l;              // explicit narrowing
double f = i;                   // implicit
int t = (int)f;                 // explicit (truncates)
int? n = i;                     // implicit T → Nullable<T>
int u = (int)n!;                // explicit Nullable<T> → T

// Boxing / unboxing
object o = i;                   // boxing (implicit)
int v = (int)o;                 // unboxing (explicit)

// User-defined
Celsius c = new(20);
double dg = c;                  // user-defined implicit
Celsius c2 = (Celsius)25.0;     // user-defined explicit
```

---

## Internals — IL for conversions

### Numeric conversions

```csharp
long l = (long)i;     // conv.i8
int  k = (int)l;      // conv.i4 (unchecked) or conv.ovf.i4 (checked)
```

The CLR has dedicated `conv` opcodes for each conversion: `conv.i4`, `conv.i8`, `conv.r4`, `conv.r8`, `conv.u1`, etc. These are typically a single CPU instruction.

### Reference casts

```csharp
Dog d = (Dog)a;
```

```il
ldloc.0
castclass [System.Runtime]Dog
stloc.1
```

`castclass` walks the runtime type's parent chain. If a match isn't found, throws `InvalidCastException`. The JIT often optimizes this to a single MT comparison when the source type is known.

For `as`:

```csharp
Dog? d = a as Dog;
```

```il
ldloc.0
isinst [System.Runtime]Dog
stloc.1
```

`isinst` (is-instance) is like `castclass` but returns null on mismatch instead of throwing.

### Boxing

```csharp
object o = i;
```

```il
ldloc.0
box [System.Runtime]System.Int32
stloc.1
```

Heap allocation, copy, push reference. See [§07](07-BoxingUnboxing.md) for the full picture.

### User-defined operators

Your `static implicit operator T(...)` compiles to a regular static method (call site is just `call`). The compiler inserts the call when it sees a conversion. So user-defined conversions cost roughly one function call (often inlined).

---

## Common conversion patterns

### Safe downcast with pattern

```csharp
public void Handle(Animal a) {
    if (a is Dog d) d.Bark();
    else if (a is Cat c) c.Meow();
    else throw new NotSupportedException();
}
```

Or a switch:

```csharp
return a switch {
    Dog d => d.Bark(),
    Cat c => c.Meow(),
    _ => throw new()
};
```

### Convert with fallback

```csharp
int n = int.TryParse(input, out var parsed) ? parsed : -1;
```

### Defensive cast from `object`

```csharp
public void Process(object value) {
    if (value is not int n) throw new ArgumentException("must be int");
    // use n
}
```

Or via pattern:
```csharp
return value switch {
    int n => ProcessInt(n),
    string s => ProcessString(s),
    _ => throw new ArgumentException()
};
```

### Conversion via interface

```csharp
public interface IConverter<T> { T Convert(string s); }

class IntConverter : IConverter<int> {
    public int Convert(string s) => int.Parse(s);
}
```

Generic conversion contracts. The BCL has `IParsable<T>` (since .NET 7) for this exact purpose.

### Round, floor, ceiling vs cast

```csharp
double d = 3.7;
int truncated = (int)d;           // 3 — truncates
int rounded = (int)Math.Round(d); // 4
int floored = (int)Math.Floor(d); // 3
int ceiled = (int)Math.Ceiling(d);// 4
```

`(int)d` truncates toward zero, NOT toward negative infinity. `(int)-3.7 == -3`, not -4.

---

## Common bugs

- **Truncation vs rounding** — `(int)3.7` is 3, not 4. Use `Math.Round` if you want rounding.
- **Silent overflow** — `(byte)300` is 44, not an exception. Use `checked` or validate.
- **String to number** with default culture — fails in locales with comma decimal separator. Always pass `CultureInfo.InvariantCulture` for machine-readable data.
- **`(int)null` from `Nullable<int>`** throws. Use `?? defaultValue` or `GetValueOrDefault`.
- **Custom `implicit` operators that look innocent** — `var x = obj` might silently invoke a conversion. Use explicit when the conversion can surprise.
- **Boxing-then-unboxing chain** — `(int)(object)5L` is `(int)5L` with intermediate boxing of long; the unbox-as-int throws.

---

## Performance

- Numeric conversions = 1-2 CPU cycles (single `conv.*` instruction).
- Cast (`castclass`) = one type-pointer check, then a direct reference assignment. Cheap.
- `as` = same as `castclass` minus the throw, slightly cheaper if you'd otherwise null-check.
- User-defined conversions = inlined static method call.
- `Convert.ToX` = a bit slower due to type-switching internally.
- `int.Parse` vs `int.TryParse` — same speed on success; `Parse` is dramatically slower on failure due to exception cost.

---

## When to use which conversion

| Goal | Use |
|---|---|
| Widen a numeric type safely | Implicit (`long l = i;`) |
| Narrow a numeric (lossy) | Cast (`(int)l`) or `checked`/`unchecked` |
| Cast reference down | `(Derived)base` or `base as Derived` for safe form |
| Test type without casting | `is` pattern |
| Convert string to number | `TryParse` (or `Parse` if input is trusted) |
| Convert arbitrary `object` | `Convert.ToX` or pattern matching |
| Domain "view" of a value | User-defined explicit operator |
| Define your own type's natural number form | User-defined implicit operator (be careful) |

→ Next: [09-PatternMatching.md](09-PatternMatching.md)
