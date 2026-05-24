# Chapter 03 — Coding Problems

> 15 hands-on problems. Each tests one or more of: value vs reference, structs, records, enums, tuples, nullable, boxing, conversion, pattern matching. Solutions in `<details>` blocks — try first.

---

## Problem 1 — Predict and explain

```csharp
class Box { public int X; }
struct StructBox { public int X; }

var a = new Box      { X = 5 };
var b = new StructBox{ X = 5 };
var a2 = a;
var b2 = b;
a2.X = 99;
b2.X = 99;
Console.WriteLine($"{a.X} {a2.X} {b.X} {b2.X}");
```

<details><summary>Answer</summary>

`99 99 5 99`

`a2 = a` copies the reference; mutating `a2.X` mutates the same object — both show 99.

`b2 = b` copies the struct's fields; `b2` is independent — original b stays 5, b2 is 99.

</details>

---

## Problem 2 — Implement a value-equality `Money` type three ways

Implement a value-type `Money` (amount + currency) with value equality:
1. As a class with manual `Equals`/`GetHashCode`/operator overloads.
2. As a record class.
3. As a `readonly record struct`.

Which is the right default?

<details><summary>Solution</summary>

```csharp
// (1) Manual class — lots of boilerplate
public sealed class MoneyA : IEquatable<MoneyA> {
    public decimal Amount { get; }
    public string Currency { get; }
    public MoneyA(decimal amount, string currency) {
        Amount = amount;
        Currency = currency ?? throw new ArgumentNullException(nameof(currency));
    }
    public bool Equals(MoneyA? other) => other is not null && Amount == other.Amount && Currency == other.Currency;
    public override bool Equals(object? o) => Equals(o as MoneyA);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
    public static bool operator ==(MoneyA? a, MoneyA? b) => Equals(a, b);
    public static bool operator !=(MoneyA? a, MoneyA? b) => !Equals(a, b);
    public override string ToString() => $"{Amount} {Currency}";
}

// (2) Record class — one line
public sealed record MoneyB(decimal Amount, string Currency);

// (3) readonly record struct — best default for small value-like types
public readonly record struct MoneyC(decimal Amount, string Currency);
```

Right default: **(3) `readonly record struct`** — small (24 bytes), value semantics, no heap allocation, value equality, immutable, `with` expressions for free.

Caveat: `default(MoneyC)` has `Currency = null`. If that's intolerable, switch to a class.

</details>

---

## Problem 3 — Find the boxing

How many heap allocations does this code do?

```csharp
int n = 42;
object o = n;
IComparable<int> c = n;
List<int> list = new();
list.Add(n);
string s = $"n is {n}";
Console.WriteLine("{0}", n);
ArrayList al = new();
al.Add(n);
```

<details><summary>Answer</summary>

- `object o = n;` — **1 box** (int → object).
- `IComparable<int> c = n;` — **1 box** (struct → interface).
- `list.Add(n);` — **0 boxes** (List&lt;int&gt; stores ints inline).
- `$"n is {n}"` — **0 boxes** in C# 10+ (interpolated string handler `AppendFormatted(int)`).
- `Console.WriteLine("{0}", n)` — **1 box** (varargs `object[]`).
- `al.Add(n)` — **1 box** (ArrayList stores objects).
- `new ArrayList()`, `new List<int>()` — 1 heap allocation each, but those are objects, not boxes.

Total boxes: **4** (one each for o, c, the WriteLine format, and the ArrayList.Add).

Plus 2 object allocations (the two collections), one StringBuilder-like buffer for the interpolated string.

</details>

---

## Problem 4 — Define a closed set with records and pattern matching

Model "payment result" with three cases:
- Success (with transaction ID).
- Rejected (with reason).
- Requires further info (with a URL).

Then write a `Display` method that returns a string per case.

<details><summary>Solution</summary>

```csharp
public abstract record PaymentResult;
public sealed record PaymentSuccess(string TransactionId) : PaymentResult;
public sealed record PaymentRejected(string Reason) : PaymentResult;
public sealed record PaymentRequiresInfo(string PromptUrl) : PaymentResult;

string Display(PaymentResult r) => r switch {
    PaymentSuccess(var id) => $"Paid! Ref: {id}",
    PaymentRejected(var why) => $"Rejected: {why}",
    PaymentRequiresInfo(var url) => $"Continue at {url}",
    _ => throw new InvalidOperationException()
};

// Test
foreach (var r in new PaymentResult[] {
    new PaymentSuccess("abc"),
    new PaymentRejected("insufficient funds"),
    new PaymentRequiresInfo("https://bank.com/verify"),
}) Console.WriteLine(Display(r));
```

The abstract base + sealed records pattern is C#'s pragmatic discriminated union.

</details>

---

## Problem 5 — Fix the boxing in this hot path

```csharp
public bool Contains(int[] arr, int needle) {
    foreach (object o in arr) {
        if (o.Equals(needle)) return true;
    }
    return false;
}
```

Where's the boxing? Fix without changing the algorithm.

<details><summary>Solution</summary>

Two boxes per iteration:
1. `foreach (object o in arr)` — boxes each int to object.
2. `o.Equals(needle)` — boxes `needle` to object for the call.

Fix:
```csharp
public bool Contains(int[] arr, int needle) {
    foreach (int i in arr) {
        if (i == needle) return true;
    }
    return false;
}
```

Or even better, use `Array.IndexOf` or `Span<T>.IndexOf`:
```csharp
return arr.AsSpan().IndexOf(needle) >= 0;
```

For 1M elements, the original was 2M boxes (~48MB). The fix: zero.

</details>

---

## Problem 6 — Predict the output (pattern matching)

```csharp
object o = new int[] { 1, 2, 3 };
string r = o switch {
    int[] [] => "empty array",
    int[] [_] => "one element",
    int[] [1, _, _] => "three starting with 1",
    int[] [.., 3] => "ends with 3",
    int[] _ => "some int array",
    _ => "not an int array"
};
Console.WriteLine(r);
```

<details><summary>Answer</summary>

`"three starting with 1"` — third arm matches first.

If the array were `{ 4, 5, 3 }`, the fourth arm `[.., 3]` would match.
If `{ 5 }`, the second arm.
If `{ }`, the first.

The order of arms matters — top-to-bottom, first match wins.

</details>

---

## Problem 7 — Convert classic try-cast to pattern

Convert this to use pattern matching:

```csharp
public string Describe(object obj) {
    if (obj == null) return "null";
    if (obj is int) {
        int n = (int)obj;
        if (n > 0) return $"positive int {n}";
        return $"non-positive int {n}";
    }
    if (obj is string) {
        string s = (string)obj;
        if (string.IsNullOrEmpty(s)) return "empty string";
        return $"string of length {s.Length}";
    }
    return obj.GetType().Name;
}
```

<details><summary>Solution</summary>

```csharp
public string Describe(object? obj) => obj switch {
    null => "null",
    int n when n > 0 => $"positive int {n}",
    int n => $"non-positive int {n}",
    string { Length: 0 } => "empty string",
    string s => $"string of length {s.Length}",
    _ => obj.GetType().Name
};
```

Half the lines, more declarative, exhaustiveness analysis from the compiler.

</details>

---

## Problem 8 — Build a tuple-based dictionary key

You have a list of `(City, Year, Sales)` tuples. Aggregate sales by `(City, Year)` and return the result.

<details><summary>Solution</summary>

```csharp
var data = new List<(string City, int Year, decimal Sales)> {
    ("NYC", 2024, 100m), ("NYC", 2024, 50m),
    ("NYC", 2025, 200m),
    ("LA",  2024, 75m),
};

var grouped = data
    .GroupBy(d => (d.City, d.Year))
    .ToDictionary(g => g.Key, g => g.Sum(d => d.Sales));

foreach (var ((city, year), total) in grouped) {
    Console.WriteLine($"{city} {year}: {total}");
}
```

Tuples make excellent composite dictionary keys — value equality, value hash, no allocation.

</details>

---

## Problem 9 — Implement `IEquatable<T>` correctly

Given:
```csharp
public struct Coord { public int X, Y; }
```

Implement IEquatable, override Equals/GetHashCode, and overload `==` / `!=`. Then measure the difference for a `HashSet<Coord>` with 1M elements vs the default reflection-based equality.

<details><summary>Solution</summary>

```csharp
public struct Coord : IEquatable<Coord> {
    public int X, Y;

    public bool Equals(Coord other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Coord c && Equals(c);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public static bool operator ==(Coord a, Coord b) => a.Equals(b);
    public static bool operator !=(Coord a, Coord b) => !a.Equals(b);
}

// Or simpler:
public readonly record struct Coord(int X, int Y);   // auto-generates all of the above
```

In a benchmark: a `HashSet<Coord>` Add+Contains loop with the default (reflection-based) `ValueType.Equals` is typically **20-100× slower** than the one above. For dictionary keys, ALWAYS implement IEquatable.

</details>

---

## Problem 10 — Spot the default-value bug

```csharp
public readonly record struct Email(string Value) {
    public string Domain => Value.Split('@')[1];
}

Email e = default;
Console.WriteLine(e.Domain);
```

What happens? How would you fix it?

<details><summary>Answer</summary>

`NullReferenceException`. `default(Email)` has `Value = null`. `null.Split('@')` throws.

**You cannot prevent default**. Options:

1. Make it tolerate default:
```csharp
public string Domain => Value?.Split('@')[1] ?? "";
```

2. Throw a clearer error:
```csharp
public string Domain => Value is null
    ? throw new InvalidOperationException("Email not initialized")
    : Value.Split('@')[1];
```

3. Use a class instead — you can validate in the ctor and the object can never exist in a bad state:
```csharp
public sealed class Email {
    public Email(string value) {
        ArgumentException.ThrowIfNullOrWhiteSpace(value);
        if (!value.Contains('@')) throw new ArgumentException();
        Value = value;
    }
    public string Value { get; }
    public string Domain => Value.Split('@')[1];
}
```

The fundamental trade-off: struct → fast, no allocation, but `default` is always reachable. Class → can enforce invariants.

</details>

---

## Problem 11 — Exhaustive switch over enum

You have:
```csharp
enum Severity { Info, Warning, Error, Critical }
```

Write a `Color` function that returns `ConsoleColor` for each — and verify the compiler warns if you forget one.

<details><summary>Solution</summary>

```csharp
ConsoleColor Color(Severity s) => s switch {
    Severity.Info => ConsoleColor.White,
    Severity.Warning => ConsoleColor.Yellow,
    Severity.Error => ConsoleColor.Red,
    Severity.Critical => ConsoleColor.DarkRed,
    // _ => throw new()  // would suppress the warning
};
```

If you remove one arm (say `Critical`), the compiler emits warning CS8509: *"The switch expression does not handle all possible values of its input type."*

**Best practice**: include a `_ => throw new()` arm to handle (a) future enum members and (b) cast-injected invalid values. The warning then disappears AND your code crashes loudly if it encounters something unexpected.

</details>

---

## Problem 12 — Implement Optional&lt;T&gt; from scratch

Implement a `readonly record struct Optional<T>` with `HasValue` and `Value`, plus a static factory and `TryGet`.

<details><summary>Solution</summary>

```csharp
public readonly record struct Optional<T> {
    public bool HasValue { get; }
    private readonly T _value;
    public T Value => HasValue ? _value : throw new InvalidOperationException("No value");

    private Optional(T value, bool hasValue) {
        _value = value;
        HasValue = hasValue;
    }

    public static Optional<T> Some(T value) => new(value, true);
    public static Optional<T> None => default;

    public bool TryGet(out T value) {
        value = _value;
        return HasValue;
    }

    public T Or(T fallback) => HasValue ? _value : fallback;

    public Optional<TResult> Map<TResult>(Func<T, TResult> f) =>
        HasValue ? Optional<TResult>.Some(f(_value)) : Optional<TResult>.None;
}

// Use:
var o = Optional<int>.Some(42);
if (o.TryGet(out var v)) Console.WriteLine(v);

var o2 = o.Map(n => n * 2);          // Optional<int>(84)
var o3 = Optional<int>.None.Map(n => n * 2);  // Optional<int>.None
```

Modern alternative: this is basically what `T?` (Nullable for structs) and pattern matching give you for free in most cases. But sometimes a real `Optional<T>` is clearer for domain code.

</details>

---

## Problem 13 — `with` on records vs structs

Predict each line:

```csharp
public record class CP(int X, int Y);
public readonly record struct CS(int X, int Y);

var cp1 = new CP(1, 2);
var cp2 = cp1;
var cp3 = cp1 with { Y = 99 };

var cs1 = new CS(1, 2);
var cs2 = cs1;
var cs3 = cs1 with { Y = 99 };

Console.WriteLine($"{cp1.Y} {cp2.Y} {cp3.Y}");
Console.WriteLine($"{cs1.Y} {cs2.Y} {cs3.Y}");
Console.WriteLine(object.ReferenceEquals(cp1, cp2));
```

<details><summary>Answer</summary>

```
2 2 99
2 2 99
True
```

For `CP` (record class): `cp1` and `cp2` share the same heap object. `cp1 with { Y = 99 }` creates a new heap object. `ReferenceEquals(cp1, cp2)` is true (same object).

For `CS` (record struct): `cs2 = cs1` copies the struct's bytes. They're independent. `cs1 with { ... }` returns a new struct. All three have independent storage. `ReferenceEquals` on a struct would box twice; we didn't compute that.

Key takeaway: `with` always returns a **new** value, but for record class that "new" is a new heap object; for record struct it's a copied stack value.

</details>

---

## Problem 14 — User-defined implicit operator gone wrong

What goes wrong here?

```csharp
public readonly struct Percentage(double value) {
    public double Value { get; } = value;
    public static implicit operator double(Percentage p) => p.Value;
    public static implicit operator Percentage(double d) => new(d);
}

double tax = 0.0825;
var pct = (Percentage)tax;
Console.WriteLine(pct);

double doubled = pct * 2;
Percentage half = pct / 2;
```

<details><summary>Answer</summary>

Code compiles and produces reasonable output. But the implicit conversions are **dangerous**:

```csharp
Percentage rate = 50;       // Looks like 50% but it's 5000% (50.0, not 0.5)
double pct = rate;           // 50.0
double offsetTax = rate + 0.1;  // pct + 0.1 → just adds — but is 0.1 a percentage or a fraction?
```

The numeric system has no idea what "Percentage" means. Implicit conversions throw away that semantic.

**Better design**:
- Make conversion explicit (`explicit operator`).
- Or provide named conversion methods: `FromFraction(0.0825)`, `FromPercent(50)`.
- Or hide the underlying type and provide operations that make sense.

```csharp
public readonly struct Percentage {
    private readonly double _fraction;  // always 0..1
    private Percentage(double fraction) { _fraction = fraction; }

    public static Percentage FromPercent(double percent) => new(percent / 100.0);
    public static Percentage FromFraction(double frac) => new(frac);

    public double AsFraction() => _fraction;
    public double AsPercent() => _fraction * 100;

    public static decimal operator *(decimal value, Percentage p) =>
        value * (decimal)p._fraction;
}

decimal subtotal = 100m;
decimal tax = subtotal * Percentage.FromPercent(8.25);
```

The named conversions remove ambiguity at the call site. The underlying double is internal.

</details>

---

## Problem 15 — Real-world: pattern matching for command processing

Given a discriminated union of commands:

```csharp
public abstract record Command;
public sealed record CreateUser(string Email, string Password) : Command;
public sealed record DeleteUser(int Id) : Command;
public sealed record ChangeEmail(int Id, string NewEmail) : Command;
public sealed record QueryUser(int Id) : Command;
```

Write a single `Handle` method that dispatches each command to the appropriate code path. Return a `string` describing what happened.

<details><summary>Solution</summary>

```csharp
public string Handle(Command cmd) => cmd switch {
    CreateUser(var email, _) when string.IsNullOrEmpty(email) =>
        throw new ArgumentException("email required"),
    CreateUser(var email, var pwd) =>
        $"Creating {email} with hashed password (length {pwd.Length})",

    DeleteUser(< 0) =>
        throw new ArgumentOutOfRangeException("id"),
    DeleteUser(var id) =>
        $"Deleting user {id}",

    ChangeEmail(_, { Length: 0 }) =>
        throw new ArgumentException("new email cannot be empty"),
    ChangeEmail(var id, var email) =>
        $"User {id} changed email to {email}",

    QueryUser(var id) =>
        $"Querying user {id}",

    _ => throw new NotSupportedException($"Unknown command: {cmd.GetType().Name}")
};
```

This combines:
- Type pattern (CreateUser, DeleteUser, ...).
- Positional pattern (decomposing the record's constructor params).
- Property pattern with relational (`{ Length: 0 }`).
- `when` guards.
- Catch-all for safety.

In MediatR-style applications, this is the dispatch pattern. The compiler tells you when you add a new command but forget to handle it (warning) — unless you have `_ => throw`, in which case you find out at runtime. Pick your trade-off based on team preference.

</details>

---

That's Chapter 03. You should now know:
- The value/reference split and what it implies for memory, copying, equality.
- Modern `record` and `record struct` as the default for data containers.
- Boxing — where it happens, what it costs, how to avoid.
- Pattern matching from constant to list to recursive.
- The C# 14 features (`field`, `?.=`, `params ReadOnlySpan<T>`) in their natural homes.

→ [Chapter 04 — Generics](../04-Generics/README.md)
