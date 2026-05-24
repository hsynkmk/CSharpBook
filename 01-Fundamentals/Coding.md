# Chapter 01 — Coding Problems

> ~15 problems covering everything from Chapter 01. Try each before opening solutions.

---

## Problem 1 — Predict the output

```csharp
int a = 5, b = 2;
Console.WriteLine(a / b);
Console.WriteLine((double)a / b);
Console.WriteLine(a / (double)b);
Console.WriteLine(a % b);
```

<details><summary>Answer</summary>

```
2
2.5
2.5
1
```
Integer division when both operands are int. `%` is the remainder.

</details>

---

## Problem 2 — Fix the bug

```csharp
string s = null;
if (s.Length == 0) Console.WriteLine("empty");
```

**Q: What's wrong? Fix three different ways.**

<details><summary>Answer</summary>

`NullReferenceException` — can't dereference null.

**Fix 1** — null check:
```csharp
if (s != null && s.Length == 0) Console.WriteLine("empty");
```

**Fix 2** — null-conditional:
```csharp
if (s?.Length == 0) Console.WriteLine("empty");
```

**Fix 3** — `string.IsNullOrEmpty`:
```csharp
if (string.IsNullOrEmpty(s)) Console.WriteLine("null or empty");
// Or IsNullOrWhiteSpace if " " should count
```

</details>

---

## Problem 3 — Build a string efficiently

Given a `List<string>`, build a comma-separated string. Compare the slow way and the right way.

<details><summary>Answer</summary>

**Slow** — O(n²) due to immutability:
```csharp
string result = "";
foreach (var s in items) {
    if (result.Length > 0) result += ", ";
    result += s;
}
```

**Right** — O(n):
```csharp
string result = string.Join(", ", items);
```

Or with StringBuilder:
```csharp
var sb = new StringBuilder();
foreach (var s in items) {
    if (sb.Length > 0) sb.Append(", ");
    sb.Append(s);
}
string result = sb.ToString();
```

`string.Join` is the idiomatic choice for joining with a separator. StringBuilder for more complex building.

</details>

---

## Problem 4 — Parse user input safely

Ask the user for an integer. Loop until they enter a valid one, then print "Got N".

<details><summary>Solution</summary>

```csharp
int n;
while (true) {
    Console.Write("Enter an integer: ");
    if (int.TryParse(Console.ReadLine(), out n)) break;
    Console.WriteLine("Not a valid integer, try again.");
}
Console.WriteLine($"Got {n}");
```

Key points: `TryParse`, not `Parse` (no exceptions); loop until success; `out` variable.

</details>

---

## Problem 5 — switch expression

Convert this if/else chain to a switch expression:

```csharp
string Category(int age) {
    if (age < 0) throw new ArgumentException();
    if (age < 13) return "child";
    if (age < 18) return "teen";
    if (age < 65) return "adult";
    return "senior";
}
```

<details><summary>Solution</summary>

```csharp
string Category(int age) => age switch {
    < 0 => throw new ArgumentException("Age cannot be negative", nameof(age)),
    < 13 => "child",
    < 18 => "teen",
    < 65 => "adult",
    _ => "senior"
};
```

Note `throw` is allowed as an expression in switch arms.

</details>

---

## Problem 6 — `ref`, `out`, `in`

Write three versions of a `Swap(a, b)` method. Which is the right signature for swap?

<details><summary>Solution</summary>

```csharp
// Wrong — modifies copies; caller sees no change
void Swap(int a, int b) {
    (a, b) = (b, a);
}

// Wrong — out requires assignment in method; doesn't read input
void Swap(out int a, out int b) {
    a = 0; b = 0;   // ??
}

// Right
void Swap(ref int a, ref int b) {
    (a, b) = (b, a);   // tuple swap
}

// Usage
int x = 1, y = 2;
Swap(ref x, ref y);
Console.WriteLine($"{x} {y}");   // 2 1
```

Only `ref` makes sense — we read AND write through the alias.

</details>

---

## Problem 7 — `params ReadOnlySpan<T>` (C# 14)

Write a `Sum` function that accepts any number of `int` arguments with zero allocation per call.

<details><summary>Solution</summary>

```csharp
public int Sum(params ReadOnlySpan<int> nums) {
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

// All zero-alloc:
Sum();
Sum(1);
Sum(1, 2, 3, 4, 5);
```

The compiler packs the args into a stack-allocated span — no array allocation. Compare to `params int[]` which allocates an array per call.

</details>

---

## Problem 8 — Multi-dimensional or jagged?

You need a 3×4 matrix of doubles. Which is better?

```csharp
double[,] rect = new double[3, 4];
double[][] jag = new double[3][];
```

<details><summary>Answer</summary>

For most code: **jagged** (`double[][]`). Each inner array is a plain `double[]` — the JIT optimizes those extremely well (bounds-check elision, SIMD). Rectangular `[,]` is bounds-checked on every access and the JIT optimizes it less aggressively.

For tight numeric work (matrix multiplication, etc.) consider:
- A flat `double[]` of length 12 with manual `[i*4 + j]` indexing — fastest.
- `Memory<double>` and slicing.
- A library type (`MathNet.Numerics.LinearAlgebra.Matrix`).

</details>

---

## Problem 9 — Predict output (closures in `for`)

```csharp
var fns = new List<Func<int>>();
for (int i = 0; i < 3; i++) fns.Add(() => i);
foreach (var f in fns) Console.Write(f());
```

<details><summary>Answer</summary>

`333`.

All three lambdas closed over the **same** `i`. By the time they run, the loop is done and `i == 3`.

**Fix** — introduce a fresh local:
```csharp
for (int i = 0; i < 3; i++) {
    int copy = i;
    fns.Add(() => copy);
}
// → 012
```

Or use foreach — each iteration gets a new variable:
```csharp
foreach (var i in Enumerable.Range(0, 3))
    fns.Add(() => i);
// → 012
```

</details>

---

## Problem 10 — Custom exception type

Design a custom exception for "user not found by id" with a UserId property.

<details><summary>Solution</summary>

```csharp
public class UserNotFoundException : Exception {
    public int UserId { get; }

    public UserNotFoundException(int userId)
        : base($"User {userId} not found") {
        UserId = userId;
    }

    public UserNotFoundException(int userId, Exception inner)
        : base($"User {userId} not found", inner) {
        UserId = userId;
    }
}

// Usage
public User GetUser(int id) =>
    _repo.Find(id) ?? throw new UserNotFoundException(id);
```

Key conventions: ends with `Exception`, inherits `Exception` (not `ApplicationException`), provides constructors taking message + inner, exposes domain context.

</details>

---

## Problem 11 — Replace boilerplate with ThrowIf helpers

Modernize:

```csharp
public void Configure(string host, int port) {
    if (host == null) throw new ArgumentNullException(nameof(host));
    if (string.IsNullOrEmpty(host)) throw new ArgumentException("host cannot be empty", nameof(host));
    if (port < 1) throw new ArgumentOutOfRangeException(nameof(port), "must be positive");
    if (port > 65535) throw new ArgumentOutOfRangeException(nameof(port), "must be <= 65535");
    // ...
}
```

<details><summary>Solution</summary>

```csharp
public void Configure(string host, int port) {
    ArgumentException.ThrowIfNullOrEmpty(host);
    ArgumentOutOfRangeException.ThrowIfLessThan(port, 1);
    ArgumentOutOfRangeException.ThrowIfGreaterThan(port, 65535);
    // ...
}
```

Cleaner, less duplication, parameter names captured automatically via CallerArgumentExpression.

</details>

---

## Problem 12 — Use raw string literals

You need a JSON template embedded in code with values substituted. Use raw string interpolation.

<details><summary>Solution</summary>

```csharp
string name = "Alice";
int age = 30;

string json = $$"""
    {
        "name":    "{{name}}",
        "age":     {{age}},
        "version": "{{2025}}"
    }
    """;
Console.WriteLine(json);
```

Outputs:
```
{
    "name":    "Alice",
    "age":     30,
    "version": "2025"
}
```

Notes:
- `$$"""` (double `$$`) means placeholders use `{{ }}` instead of `{ }`. Useful so the literal `{` in JSON doesn't confuse the parser.
- The closing `"""` determines the indentation — leading spaces matching its column are stripped from each line.

</details>

---

## Problem 13 — Slice efficiently

You have a string `"name=Alice;age=30"`. Parse out the value of `name` without allocating substrings.

<details><summary>Solution</summary>

```csharp
ReadOnlySpan<char> input = "name=Alice;age=30".AsSpan();

int eq = input.IndexOf('=');
int sc = input.IndexOf(';');

ReadOnlySpan<char> key = input[..eq];           // "name"
ReadOnlySpan<char> val = input[(eq + 1)..sc];   // "Alice"

Console.WriteLine($"key={key.ToString()} val={val.ToString()}");
// .ToString() at the end is when you finally allocate
```

For very high throughput, use existing tooling like `MemoryExtensions.IndexOf`, `Split`, or `Tokenizer` patterns.

</details>

---

## Problem 14 — Build a calculator

Write a console program that reads two numbers and an operator (`+`, `-`, `*`, `/`) and prints the result. Use a switch expression. Handle division by zero gracefully.

<details><summary>Solution</summary>

```csharp
Console.Write("Number 1: ");
if (!double.TryParse(Console.ReadLine(), out double a)) {
    Console.WriteLine("Not a number"); return 1;
}

Console.Write("Operator (+ - * /): ");
string op = Console.ReadLine() ?? "";

Console.Write("Number 2: ");
if (!double.TryParse(Console.ReadLine(), out double b)) {
    Console.WriteLine("Not a number"); return 1;
}

double result;
try {
    result = op switch {
        "+" => a + b,
        "-" => a - b,
        "*" => a * b,
        "/" when b == 0 => throw new DivideByZeroException(),
        "/" => a / b,
        _   => throw new ArgumentException($"Unknown operator '{op}'")
    };
} catch (Exception ex) {
    Console.WriteLine($"Error: {ex.Message}");
    return 1;
}

Console.WriteLine($"{a} {op} {b} = {result}");
return 0;
```

Notes:
- `TryParse` for input validation.
- Switch expression + `when` guard for the division-by-zero case.
- `throw` as an expression inside switch arms.
- Returning an exit code.

</details>

---

## Problem 15 — XML doc a method

Add complete XML documentation to:

```csharp
public bool TryConvertToPositive(string input, out int result) {
    result = 0;
    if (!int.TryParse(input, out var parsed)) return false;
    if (parsed <= 0) return false;
    result = parsed;
    return true;
}
```

<details><summary>Solution</summary>

```csharp
/// <summary>
/// Attempts to convert <paramref name="input"/> to a positive integer.
/// </summary>
/// <param name="input">The string to parse. Allowed: any integer representation.</param>
/// <param name="result">
/// When this method returns <see langword="true"/>, the parsed value;
/// otherwise zero.
/// </param>
/// <returns>
/// <see langword="true"/> if the input represents a positive integer (greater than zero);
/// otherwise <see langword="false"/>.
/// </returns>
/// <remarks>
/// Uses <see cref="int.TryParse(string?, out int)"/> internally. Invariant culture
/// is implied by the underlying parser.
/// </remarks>
/// <example>
/// <code>
/// TryConvertToPositive("42", out var n);   // n = 42, returns true
/// TryConvertToPositive("-1", out var n);   // n = 0, returns false
/// TryConvertToPositive("abc", out var n);  // n = 0, returns false
/// </code>
/// </example>
public bool TryConvertToPositive(string input, out int result) { ... }
```

Comprehensive but not bloated. The summary tells you what; remarks adds detail; example shows usage.

</details>

---

## Bonus — write your own `string.Join`

Implement `MyJoin(string sep, IEnumerable<string> items)` using a StringBuilder, then compare its output to `string.Join`.

<details><summary>Solution</summary>

```csharp
public static string MyJoin(string sep, IEnumerable<string> items) {
    ArgumentNullException.ThrowIfNull(items);
    var sb = new StringBuilder();
    bool first = true;
    foreach (var s in items) {
        if (!first) sb.Append(sep);
        sb.Append(s);
        first = false;
    }
    return sb.ToString();
}

// Test
var items = new[] { "a", "b", "c" };
string a = MyJoin(", ", items);
string b = string.Join(", ", items);
Debug.Assert(a == b);
```

Built-in `string.Join` is faster (it pre-computes the total length, allocates exactly once, copies in unmanaged-memcpy style). Yours is correct but not optimal. For learning, this is plenty.

</details>

---

→ Continue to: [Chapter 02 — OOP](../02-OOP/README.md)
