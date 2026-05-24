# Enums

## What it is

An **enum** is a named set of related integer constants — a small, closed list of options with readable names.

```csharp
public enum Severity { Info, Warning, Error, Critical }

Severity s = Severity.Warning;
Console.WriteLine(s);            // "Warning"
Console.WriteLine((int)s);       // 1
```

Enums are value types — they live where they're declared, copy on assignment, can't be null (unless you wrap in `Nullable<T>`).

---

## Why it exists

Without enums you'd be passing magic numbers around:

```csharp
LogMessage(2, "...");   // 2? What's 2?
LogMessage(Severity.Error, "...");  // obviously an Error
```

Enums give you:
- **Readable code** — names instead of numbers.
- **Type safety** — `LogMessage(int)` accepts any int; `LogMessage(Severity)` only accepts Severity values.
- **IntelliSense** — your editor lists the options.
- **Compile-time validation** — typos become errors, not silent bugs.

---

## Declaring

```csharp
public enum Day { Monday, Tuesday, Wednesday, Thursday, Friday, Saturday, Sunday }
```

Members are implicitly numbered starting at 0. So:
- `Monday = 0`
- `Tuesday = 1`
- ...
- `Sunday = 6`

You can override:

```csharp
public enum Status {
    Pending = 1,
    Active = 5,
    Suspended = 10,
    Deleted = 100
}
```

Or partially:
```csharp
public enum Day {
    Monday = 1,
    Tuesday,        // 2
    Wednesday,      // 3
    Thursday,       // 4
    Friday,         // 5
    Saturday,       // 6
    Sunday          // 7
}
```

Subsequent members count up from the last specified value.

---

## Underlying type

By default an enum's underlying type is `int` (32-bit signed). You can pick another:

```csharp
public enum SmallEnum : byte { A, B, C }
public enum BigEnum : long { ... }
```

Allowed underlying types: `sbyte`, `byte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`. Not `float`, `double`, or `char`.

Why bother?
- **Memory**: arrays/structs of enums save space.
- **Interop**: matching a native type from C/C++.
- **Range**: very large value sets need `long`.

```csharp
public enum HttpStatus : ushort {
    OK = 200,
    NotFound = 404,
    InternalServerError = 500
}
```

---

## Casting

Enums freely convert to and from their underlying type — but explicitly:

```csharp
Severity s = Severity.Error;
int n = (int)s;              // explicit cast required
Severity back = (Severity)2; // even if 2 doesn't correspond to a defined value!

Severity weird = (Severity)999;
Console.WriteLine(weird);    // "999" — no error, no Validation
```

C# does NOT validate the cast. `(Severity)999` produces a Severity holding the value 999, even though no such member exists. This is for performance — validation would cost a runtime check.

To check validity:

```csharp
if (Enum.IsDefined(typeof(Severity), value)) { ... }
// or, generic since .NET 5:
if (Enum.IsDefined<Severity>(value)) { ... }
```

`Enum.IsDefined` is fairly slow (it walks the defined values). For hot paths consider explicit bounds checks or pattern matching.

---

## Parsing strings

```csharp
Severity s = Enum.Parse<Severity>("Error");   // throws if invalid

bool ok = Enum.TryParse<Severity>("Warning", out var sev);
bool ok2 = Enum.TryParse<Severity>("warning", ignoreCase: true, out var sev2);
```

Or for very common use, manual:

```csharp
Severity? Parse(string s) => s switch {
    "Info" => Severity.Info,
    "Warning" => Severity.Warning,
    "Error" => Severity.Error,
    _ => null
};
```

Hand-written switch is **much faster** than `Enum.Parse` (no reflection).

---

## `[Flags]` enums — bitwise composition

When an enum represents a **set of options** that combine, mark it with `[Flags]` and give each member a distinct power-of-two value:

```csharp
[Flags]
public enum Permissions {
    None  = 0,
    Read  = 1 << 0,    //  1 = 0b0001
    Write = 1 << 1,    //  2 = 0b0010
    Edit  = 1 << 2,    //  4 = 0b0100
    Delete= 1 << 3,    //  8 = 0b1000

    All = Read | Write | Edit | Delete   // composite
}
```

Now you can combine and test:

```csharp
Permissions p = Permissions.Read | Permissions.Write;
bool canRead = (p & Permissions.Read) != 0;
bool canRead2 = p.HasFlag(Permissions.Read);     // shorter, but slower (boxing pre-.NET Core)
```

`[Flags]` makes `ToString` print the combination correctly:

```csharp
Console.WriteLine(p);   // "Read, Write" (with [Flags])
                         // "3" (without [Flags])
```

### Conventions

- Always include a `None = 0`. Lets you start from "no permissions."
- Use powers of two for primitive flags.
- Composite values like `All = Read | Write | ...` are fine and useful.
- Use bitwise OR (`|`) to combine, bitwise AND (`&`) to test.

### Common flag operations

```csharp
p |= Permissions.Edit;             // add Edit
p &= ~Permissions.Write;           // remove Write
bool hasAny = (p & desired) != 0;  // any overlap
bool hasAll = (p & desired) == desired;  // all of `desired`
```

### Pitfalls

- Forgetting `[Flags]` → ToString prints the int.
- Using non-powers-of-two for individual flags → bitwise operations yield nonsense.
- Using `==` with a single flag instead of `&`: `if (p == Permissions.Read)` is false when other flags are also set.

---

## Switching on enums

```csharp
switch (severity) {
    case Severity.Info:
        Console.WriteLine("info");
        break;
    case Severity.Warning:
        Console.WriteLine("warning");
        break;
    // ...
}

// Better: switch expression
string Color(Severity s) => s switch {
    Severity.Info     => "blue",
    Severity.Warning  => "yellow",
    Severity.Error    => "orange",
    Severity.Critical => "red",
    _ => throw new ArgumentException()   // fallback for undefined values from cast
};
```

The compiler **doesn't enforce exhaustiveness** even on enums — adding a new member won't break old switch expressions. Best practice: always have a `_ =>` arm that throws (or returns a default), so you know when something fell through.

In .NET 8+ the compiler will warn about a missing default when switching on an enum **without** any default arm, in a switch expression.

---

## Enum methods

Static helpers on `Enum`:

```csharp
// All defined values
Severity[] all = Enum.GetValues<Severity>();

// All defined names
string[] names = Enum.GetNames<Severity>();

// Validate
bool valid = Enum.IsDefined<Severity>(2);

// Parse
Severity s = Enum.Parse<Severity>("Error");
bool ok = Enum.TryParse<Severity>("Error", out var s2);
bool ok2 = Enum.TryParse<Severity>("error", ignoreCase: true, out var s3);

// To string
string name = Severity.Error.ToString();   // "Error"
string name2 = Enum.GetName(Severity.Error);

// Boolean: HasFlag (for [Flags] enums)
bool hasRead = perms.HasFlag(Permissions.Read);
```

`Enum.IsDefined`, `Enum.GetValues`, `Enum.Parse` all involve reflection-style lookups. For hot paths, hand-written switches are much faster.

---

## Enums and JSON

`System.Text.Json` defaults to writing enums as **integers**. For human-readable JSON, use the `JsonStringEnumConverter`:

```csharp
public record Order([property: JsonConverter(typeof(JsonStringEnumConverter))] Status Status);

// Or globally:
services.AddJsonOptions(o => o.SerializerOptions.Converters.Add(new JsonStringEnumConverter()));
```

Then `Status.Pending` serializes as `"Pending"` instead of `1`.

For Newtonsoft.Json: `[JsonConverter(typeof(StringEnumConverter))]`.

---

## Adding behavior to enums

Enums in C# can't have instance methods. To attach behavior, use **extension methods**:

```csharp
public static class SeverityExtensions {
    public static ConsoleColor ToColor(this Severity s) => s switch {
        Severity.Info => ConsoleColor.White,
        Severity.Warning => ConsoleColor.Yellow,
        Severity.Error => ConsoleColor.Red,
        Severity.Critical => ConsoleColor.DarkRed,
        _ => ConsoleColor.Gray
    };
}

Severity.Error.ToColor();   // ConsoleColor.Red
```

Or, if you really need rich behavior per case, consider a **sealed class hierarchy** or a discriminated-union style using records:

```csharp
public abstract record Severity {
    public abstract ConsoleColor Color { get; }
    public sealed record Info() : Severity { public override ConsoleColor Color => ConsoleColor.White; }
    public sealed record Warning() : Severity { public override ConsoleColor Color => ConsoleColor.Yellow; }
    // ...
}
```

Heavier, but each value can have its own logic.

---

## Internals — what an enum is at runtime

An enum is a struct deriving from `System.Enum`, which derives from `System.ValueType`, which derives from `System.Object`. So:
- Enums are **value types**.
- Casting to `Enum` or `object` **boxes** them.
- Enum size equals the underlying type's size (typically 4 bytes for the default `int`).

In IL, `Severity.Error` becomes the integer 2 — type info is metadata only:

```il
ldc.i4.2
stloc.0
```

The enum's name list is held in metadata, used by `ToString`, `Enum.Parse`, `Enum.GetNames`. Hence why these reflection-based operations are slow.

### Boxing surprises

```csharp
Severity s = Severity.Error;
object o = s;                  // box
bool eq = o.Equals(Severity.Error);  // boxes Severity.Error too
```

Each conversion to object allocates. In a hot loop comparing enums via `object` or interfaces, that's a hidden allocation per iteration.

Modern API tips:
- `Enum.HasFlag` boxed pre-.NET Core 2.1; now it's optimized.
- Avoid `Enum.IsDefined(typeof(MyEnum), val)` in hot paths; the generic form `IsDefined<T>(val)` (since .NET 5) is faster but still slow compared to a hand-coded check.

### `Enum` methods are reflection-based

`Enum.GetValues`, `Enum.GetNames`, `Enum.Parse`, `Enum.Format` all walk reflection metadata. For one-off use this is fine. For per-request paths in a web app, cache the result:

```csharp
private static readonly Severity[] AllSeverities = Enum.GetValues<Severity>();
```

### `[Flags]` and `ToString`

For `[Flags]` enums, `ToString` does a clever decomposition:
- Walks through the defined values from largest to smallest.
- Subtracts each one that fits, recording its name.
- Joins with ", ".

If the bits don't match any defined combination, it prints the number.

```csharp
[Flags] enum E { A=1, B=2, C=4, AB=A|B }
Console.WriteLine((E)3);   // "AB" (because AB=3 was defined as a composite)
Console.WriteLine((E)6);   // "B, C" (because 6 = B | C, no composite defined)
Console.WriteLine((E)8);   // "8"   (no match)
```

---

## When to use enums

✓ A small, closed set of named integer-like constants.
✓ Combinations of independent options (`[Flags]`).
✓ Domain states (`OrderStatus`, `Severity`, `Day`).
✓ Compile-time validation that callers pick from a known set.

✗ When the set is open or dynamic (configurable per environment).
✗ When the choices carry significant behavior — model with a class hierarchy or records.
✗ When you need to associate strings or other data with each case (`enum` only stores int values).

---

## Common bugs

- **Missing `[Flags]` on a flags-style enum** — `ToString` shows numbers; combinations behave wrong.
- **Casting an arbitrary int and treating it as valid** — the cast doesn't validate. Check with `Enum.IsDefined` or use the value carefully.
- **`HasFlag` for performance-sensitive code on pre-.NET Core 2.1** — boxes both operands; use `(p & flag) == flag`.
- **Switch missing a case after adding a new enum value** — the compiler doesn't enforce exhaustiveness. Add a `_ => throw new()` arm and rely on tests.
- **Comparing flag with `==` instead of bitwise** — `p == Permissions.Read` is false if other flags are set.
- **Default value is 0** — if you don't have a "0" member, `default(MyEnum)` is meaningless. Always include `None = 0` for flags, or be aware of the default.

---

## Performance summary

- Reading and comparing enums = reading and comparing integers. Free.
- `Enum.Parse`, `Enum.GetValues`, `Enum.IsDefined` = reflection. Slow on hot paths. Cache if needed.
- `Enum.HasFlag` since .NET Core 2.1 = JIT-intrinsic. Fast.
- Boxing an enum = ~24 bytes allocated. Avoid in inner loops.

→ Next: [05-Tuples.md](05-Tuples.md)
