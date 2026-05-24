# Operators

## What it is

An **operator** is a symbol (like `+`, `==`, `!`) that performs a computation on one or more operands. C# inherits much of its operator set from C/C++/Java, with additions for null safety and pattern matching.

This file is a reference. Skim it once; come back to look up specifics.

---

## Arithmetic operators

| Operator | Meaning | Example | Result |
|---|---|---|---|
| `+` | Addition | `3 + 4` | `7` |
| `-` | Subtraction (or unary negation) | `5 - 2` | `3` |
| `*` | Multiplication | `3 * 4` | `12` |
| `/` | Division | `10 / 3` | `3` (integer!) |
| `%` | Remainder (modulo) | `10 % 3` | `1` |
| `++` | Increment | `i++` | post-increment |
| `--` | Decrement | `i--` | post-decrement |

### Integer division surprise

```csharp
Console.WriteLine(5 / 2);     // 2 — both operands are int, result is int
Console.WriteLine(5.0 / 2);   // 2.5 — at least one is double, result is double
Console.WriteLine(5 / 2.0);   // 2.5
Console.WriteLine((double)5 / 2);  // 2.5

Console.WriteLine(-7 / 2);    // -3 — truncation toward zero
Console.WriteLine(-7 % 2);    // -1 — sign matches dividend
```

For floor division use `Math.Floor`:

```csharp
Math.Floor(-7.0 / 2);   // -4.0
```

### Pre vs post-increment

```csharp
int a = 5;
int b = a++;     // b = 5, then a becomes 6
int c = ++a;     // a becomes 7 first, then c = 7
```

Use in expressions sparingly — readability suffers. Most C# style guides recommend `i++;` as a standalone statement and avoid embedding it in larger expressions.

### Compound assignment

```csharp
int n = 10;
n += 5;   // n = n + 5  → 15
n -= 3;   // n = n - 3  → 12
n *= 2;   // n = n * 2  → 24
n /= 4;   // n = n / 4  → 6
n %= 4;   // n = n % 4  → 2
```

For all binary operators. The expression `a op= b` is generally equivalent to `a = a op b` (with `a` evaluated only once on the LHS).

**C# 14 adds user-defined compound assignment** — you can override `+=` independently of `+`. See [Chapter 11 §07](../11-ModernFeatures/07-CSharp14.md).

---

## Comparison operators

| Operator | Meaning |
|---|---|
| `==` | Equal |
| `!=` | Not equal |
| `<` | Less than |
| `>` | Greater than |
| `<=` | Less than or equal |
| `>=` | Greater than or equal |

Each returns `bool`.

### Equality details

For **value types**: `==` compares the actual values.
For **reference types**: `==` compares references (whether two variables point to the same object) — **with two exceptions**: `string` and `Nullable<T>`, where `==` is overloaded to compare values.

```csharp
int a = 5, b = 5;
a == b;                          // true (value comparison)

string s1 = "hi", s2 = "hi";
s1 == s2;                        // true (string overrides == for value comparison)

object o1 = "hi", o2 = "hi";
o1 == o2;                        // true at compile time! Both literals are interned.

class Box { public int X; }
new Box() == new Box();          // false — different references
```

For custom classes you can overload `==` (and **must** also overload `!=`, `Equals`, and `GetHashCode` together — see [Chapter 07 §11](../07-Collections/11-EqualityContract.md)).

For records, the compiler generates value-based `==` for free.

---

## Logical operators

| Operator | Meaning |
|---|---|
| `&&` | Logical AND (short-circuit) |
| `\|\|` | Logical OR (short-circuit) |
| `!` | Logical NOT |
| `&` | Logical AND (not short-circuit) |
| `\|` | Logical OR (not short-circuit) |
| `^` | Logical XOR |

### Short-circuit

`&&` and `||` evaluate the right side only if necessary:

```csharp
if (s != null && s.Length > 0) { ... }   // safe — s.Length not evaluated if s is null
if (s != null & s.Length > 0)  { ... }   // ⚠ — both sides evaluated, NRE if s is null
```

Always use `&&` and `||` for control flow. The non-short-circuit forms (`&`, `|`) are for bitwise operations on integers.

### `^` (XOR)

```csharp
true ^ false;    // true
true ^ true;     // false
3 ^ 5;           // 6 — bitwise XOR for integers
```

For `bool`, `^` is "exactly one is true." Rare but occasionally useful.

---

## Bitwise operators

For `int`, `long`, `byte`, `ulong`, etc.:

| Operator | Meaning |
|---|---|
| `&` | Bitwise AND |
| `\|` | Bitwise OR |
| `^` | Bitwise XOR |
| `~` | Bitwise complement (flip all bits) |
| `<<` | Left shift |
| `>>` | Right shift (arithmetic — preserves sign) |
| `>>>` | Right shift (logical / unsigned) — C# 11+ |

```csharp
int a = 0b_1010;
int b = 0b_1100;
a & b;        // 0b_1000 = 8
a | b;        // 0b_1110 = 14
a ^ b;        // 0b_0110 = 6
~a;           // 0b...1111_0101 (32-bit complement)
a << 2;       // 0b_101000 = 40
a >> 1;       // 0b_0101 = 5
```

`>>>` shifts in zeros from the left, regardless of sign:

```csharp
int n = -8;            // 0xFFFFFFF8
n >> 1;                // -4 (arithmetic: signed)
n >>> 1;               // 2147483644 (logical: unsigned, fills with 0)
```

Useful for flags (`[Flags]` enums use bitwise operators heavily — see [Chapter 03 §04](../03-TypeSystem/04-Enums.md)).

---

## Null operators

Modern C# has rich null-handling.

### `??` — null-coalescing

```csharp
string? name = null;
string display = name ?? "anonymous";   // "anonymous"
```

Returns the left operand if non-null; otherwise the right.

### `??=` — null-coalescing assignment

```csharp
string? cached = null;
cached ??= LoadFromDb();   // if cached is null, assign LoadFromDb() to it
```

Equivalent to: `cached = cached ?? LoadFromDb();` — but the right side is evaluated only if needed.

### `?.` — null-conditional member access

```csharp
string? s = GetMaybe();
int? len = s?.Length;        // null if s is null, else s.Length

User? user = GetUser();
string? email = user?.Profile?.Email;   // chain of null checks
```

If any link in the chain is null, the whole expression evaluates to null. No more "did I null-check that?" boilerplate.

### `?[]` — null-conditional indexing

```csharp
int[]? arr = GetMaybe();
int? first = arr?[0];        // null if arr is null
```

### `?.=` — null-conditional assignment (C# 14+)

```csharp
User? user = GetUser();
user?.Name = "Alice";    // assigns only if user is non-null
```

Pre-C# 14 you had to write:
```csharp
if (user != null) user.Name = "Alice";
```

`?.=` is more concise but more importantly, the right side is **not evaluated** when the target is null — same short-circuit behavior as `?.`.

### `!` — null-forgiving (the bang operator)

```csharp
string? maybeName = GetMaybe();
int length = maybeName!.Length;   // "trust me, it's not null"
```

Tells the compiler "I know better than you — suppress the nullability warning." Used in code that the compiler can't analyze. Use sparingly — every `!` is a place a future bug could hide.

---

## Conditional / ternary operator

`condition ? whenTrue : whenFalse`:

```csharp
int n = 5;
string msg = n > 0 ? "positive" : "non-positive";
int abs = n >= 0 ? n : -n;
```

Equivalent to a tiny if/else. Chains are legal but quickly become unreadable:

```csharp
// 🥲
string category = age < 13 ? "child" : age < 18 ? "teen" : age < 65 ? "adult" : "senior";

// Better: a switch expression
string category = age switch {
    < 13 => "child",
    < 18 => "teen",
    < 65 => "adult",
    _    => "senior"
};
```

---

## `typeof` and `is` and `as`

### `typeof`

Returns a `System.Type` at compile time:

```csharp
Type t = typeof(int);
Console.WriteLine(t.Name);    // "Int32"
Console.WriteLine(t.FullName); // "System.Int32"

Type listType = typeof(List<>);   // open generic
Type closedList = typeof(List<int>);
```

### `is`

Type test. Returns `bool`. Modern form pulls out a typed variable:

```csharp
object o = "hello";

if (o is string)              // true
    Console.WriteLine("yes");

if (o is string s)            // type pattern — s is a string in scope
    Console.WriteLine(s.Length);

if (o is string { Length: > 0 } nonEmpty)   // property pattern
    Console.WriteLine(nonEmpty);

if (o is null) { ... }        // also valid — null pattern
if (o is not null) { ... }    // C# 9+ — negated
```

### `as`

Type cast that returns null on failure (instead of throwing):

```csharp
object o = "hello";
string? s = o as string;      // "hello"
int? n = o as int?;           // null — o is not an int

// Compare to direct cast (throws InvalidCastException on failure):
string s2 = (string)o;        // works
int n2 = (int)o;              // throws
```

`as` only works for **reference types and nullable value types** — it can't return a non-nullable value type, because there's no "null" to return on failure.

---

## Range and index operators (C# 8+)

### `^` index — from end

```csharp
int[] arr = { 10, 20, 30, 40, 50 };
arr[^1];        // 50 — last element
arr[^2];        // 40 — second-to-last
```

`^n` is shorthand for `Length - n`.

### `..` range

```csharp
arr[0..3];      // { 10, 20, 30 } — start inclusive, end exclusive
arr[..3];       // { 10, 20, 30 } — start defaults to 0
arr[2..];       // { 30, 40, 50 } — end defaults to Length
arr[..];        // full copy

arr[^3..^1];    // { 30, 40 } — 3rd from end, up to (excl.) last
```

Works on `string`, arrays, `Span<T>`, `List<T>` (sort of — via extension methods).

The `..` operator returns a `System.Range`; the `^` returns a `System.Index`. You can store them:

```csharp
Range r = 1..^1;
arr[r];      // { 20, 30, 40 }
```

---

## Operator precedence

The roughly memorized order (highest precedence first):

1. **Primary**: `x.y`, `f(x)`, `a[i]`, `x?.y`, `x++`, `x--`, `new`, `typeof`, `default`, `nameof`
2. **Unary**: `+x`, `-x`, `!x`, `~x`, `++x`, `--x`, `(T)x`, `*x`, `&x`
3. **Range**: `x..y`
4. **Switch / with**: `switch`, `with`
5. **Multiplicative**: `*`, `/`, `%`
6. **Additive**: `+`, `-`
7. **Shift**: `<<`, `>>`, `>>>`
8. **Relational and type test**: `<`, `>`, `<=`, `>=`, `is`, `as`
9. **Equality**: `==`, `!=`
10. **Logical AND**: `&`
11. **Logical XOR**: `^`
12. **Logical OR**: `|`
13. **Conditional AND**: `&&`
14. **Conditional OR**: `||`
15. **Null-coalescing**: `??`
16. **Conditional (ternary)**: `?:`
17. **Assignment and lambda**: `=`, `+=`, `=>`, etc.

**You don't need to memorize this**. Use parentheses liberally. The compiler doesn't pay you per character.

```csharp
// 🥲 unclear
int x = a + b * c == d || e && f;

// ✓ clear
int x = ((a + (b * c)) == d) || (e && f);
```

A few precedence gotchas:

- **Bitwise `&` `|` are lower than comparison**: `(x & 1 == 0)` is `x & (1 == 0)` = `x & 0` = `0`. Use parens.
- **`?:` is right-associative**: `a ? b : c ? d : e` is `a ? b : (c ? d : e)`.
- **Assignment is right-associative**: `a = b = 5` is `a = (b = 5)` — assigns 5 to b, then b to a.

---

## Lambda operator `=>`

Not an operator in the traditional sense; it's the lambda arrow:

```csharp
Func<int, int> square = x => x * x;
Action<string> log = msg => Console.WriteLine(msg);
Func<int, int, int> add = (a, b) => a + b;
Func<int, int, int> addExplicit = (int a, int b) => a + b;

// Statement body
Func<int, int> doubled = x => {
    int result = x * 2;
    return result;
};
```

Also used in **expression-bodied members** for properties/methods:

```csharp
public int Square(int x) => x * x;
public string Greeting => $"Hello, {Name}";
```

[Chapter 05 §02](../05-DelegatesEvents/02-Lambdas.md) covers lambdas in depth.

---

## Misc operators

### `nameof`

Returns the name of a variable, type, or member at compile time:

```csharp
public void Set(int value) {
    if (value < 0) throw new ArgumentException("must be non-negative", nameof(value));
}
```

Better than the string `"value"` because refactoring tools update it when you rename. C# 14 adds `nameof(List<>)` (unbound generic).

### `default`

```csharp
int n = default;              // 0
string? s = default;          // null
List<int>? l = default;       // null
TimeSpan t = default;         // 00:00:00
```

### `sizeof`

```csharp
sizeof(int);      // 4
sizeof(double);   // 8
sizeof(bool);     // 1
```

Only works for unmanaged types (primitives, structs without reference fields). For arbitrary types use `Marshal.SizeOf` (interop only) or `Unsafe.SizeOf<T>()` (faster).

### `new`

Creates a new instance:

```csharp
var list = new List<int>();
var arr = new int[5];
var sb = new StringBuilder("hi");

// Target-typed new (C# 9+)
List<int> list2 = new();      // type inferred
Dictionary<string, int> dict = new() { ["a"] = 1 };
```

### `is not`, `and`, `or` in patterns

Logical pattern operators (C# 9+):

```csharp
if (n is > 0 and < 100) { ... }
if (n is 0 or 1) { ... }
if (s is not null) { ... }
```

---

## Common bugs

- **`int x = 5 / 2;`** — `x == 2`, not `2.5`. Force one operand to double: `5.0 / 2`.
- **`if (s.Length > 0 && s != null)`** — short-circuit goes left-to-right; `s.Length` runs first and throws if `s` is null. Reorder.
- **`if (x & 1 == 0)`** — `==` has higher precedence than `&`. Use `(x & 1) == 0`.
- **`if (a == b == c)`** — compiles, but means `(a == b) == c`, comparing a bool to c. Almost never what you want.
- **`int a = b = c = 0;`** — chained assignment is legal but unusual. `b = c = 0;` makes both, then `a = b;` separately is clearer.
- **`obj as string + 1`** — `as` has lower precedence than `+`, so this is `obj as (string + 1)` — wait, that doesn't even compile. Hold on — `as` is lower than `+`? Yes — but the compiler resolves this to `(obj as string) + 1` because `string + 1` is invalid. Still: parens are clearer.

---

## Performance

- Integer arithmetic is essentially free on modern CPUs.
- Floating-point: `double` is typically as fast as `int` on x64; `decimal` is much slower (~10×).
- Division is slower than multiplication. The JIT replaces `x / 100` (constant divisor) with a multiply-by-reciprocal trick.
- `&&` and `||` save work via short-circuit; `&` and `|` always evaluate both sides — only matters if the right side has side effects or is expensive.
- `++` / `--` on locals: same speed as `+= 1`. The compiler treats them identically.

→ Next: [05-ControlFlow.md](05-ControlFlow.md)
