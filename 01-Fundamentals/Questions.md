# Chapter 01 — Questions

> Drilling for everything in Chapter 01. Try to answer before opening the spoiler.

---

## Variables

**Q1.** What's the difference between `var x = 5;` and `dynamic x = 5;`?
<details><summary>Answer</summary>`var` is **static**, compile-time type inference — after compilation `x` is exactly `int`. `dynamic` defers binding to runtime via the DLR — every operation goes through dynamic dispatch. Don't conflate them.</details>

**Q2.** Why does this not compile?
```csharp
int x;
Console.WriteLine(x);
```
<details><summary>Answer</summary>**Definite assignment**. C# requires locals to be assigned before being read. Fields default automatically; locals don't.</details>

**Q3.** What's the difference between `const` and `static readonly`?
<details><summary>Answer</summary>`const` is compile-time — its value is baked into every assembly that references it. Changing a public `const` requires rebuilding all consumers. `static readonly` is set once at runtime (field init or static ctor) and lives in the assembly's data — consumers see new values without recompiling. `const` is also restricted to types that have literal forms (primitives, string, enum).</details>

**Q4.** Which can be declared with `var`?
- a method parameter
- a field of a class
- a local in a method
- a foreach loop variable

<details><summary>Answer</summary>Only local variables (in methods) and foreach loop variables. Method parameters and fields require explicit types.</details>

---

## Primitive types

**Q5.** What does this print?
```csharp
Console.WriteLine(5 / 2);
Console.WriteLine(5.0 / 2);
Console.WriteLine(0.1 + 0.2);
```
<details><summary>Answer</summary>
```
2
2.5
0.30000000000000004
```
Integer division truncates. Mixed int/double promotes to double. 0.1 + 0.2 in binary doesn't exactly equal 0.3.
</details>

**Q6.** When would you use `decimal` instead of `double`?
<details><summary>Answer</summary>Whenever you need exact decimal arithmetic — money, percentages, accounting. `double` is binary floating-point and can't represent 0.1 exactly; `decimal` is base-10 and can.</details>

**Q7.** What does `checked { int n = int.MaxValue + 1; }` do?
<details><summary>Answer</summary>Throws `OverflowException`. Without `checked`, the addition silently wraps to `int.MinValue`.</details>

**Q8.** Cast `300` to `byte` (`byte b = (byte)300;`). What does `b` equal?
<details><summary>Answer</summary>`44`. Default integer casts wrap (300 mod 256 = 44). To get an exception instead, use `checked((byte)300)`.</details>

**Q9.** Why is `int.Parse("3.14")` an error?
<details><summary>Answer</summary>`int.Parse` only parses integers. Use `double.Parse` (or `decimal.Parse`) for decimals, and cast: `(int)double.Parse("3.14")` = 3.</details>

**Q10.** What's the difference between `int.Parse` and `int.TryParse`?
<details><summary>Answer</summary>`Parse` throws `FormatException` (and friends) on bad input. `TryParse` returns `bool` for success and provides the value via `out` — never throws. Use `TryParse` for any input you don't fully trust.</details>

---

## Strings

**Q11.** Why is this O(n²)?
```csharp
string s = "";
for (int i = 0; i < n; i++) s += i.ToString();
```
<details><summary>Answer</summary>Strings are immutable. Each `+=` allocates a **new** string copying the old contents plus the addition. Total work is `O(1 + 2 + 3 + ... + n) = O(n²)`. Use `StringBuilder` for incremental construction.</details>

**Q12.** What's the difference between `"path\\file"` and `@"path\file"`?
<details><summary>Answer</summary>Same result. The first uses standard escape, doubling the backslash. The second is a verbatim string — backslashes are literal.</details>

**Q13.** What's the difference between `string s = "hi"; s == "hi"` and `string s = new string(...)`?
<details><summary>Answer</summary>For strings, `==` is overloaded to compare by **value** (one of the few exceptions for reference types). So both expressions return true for equal contents. `ReferenceEquals` would distinguish them.</details>

**Q14.** Predict the output:
```csharp
string emoji = "😀";
Console.WriteLine(emoji.Length);
```
<details><summary>Answer</summary>**2**. Emojis (and many other codepoints) are surrogate pairs in UTF-16, occupying two `char`s. Use `EnumerateRunes()` to iterate by codepoint, or `StringInfo` for grapheme clusters.</details>

**Q15.** Why use `StringComparison.OrdinalIgnoreCase` instead of `ToUpper()`?
<details><summary>Answer</summary>(1) `ToUpper()` allocates a new string. (2) Culture-aware operations have subtle traps (Turkish I, German ß). Ordinal comparison is faster, doesn't allocate, and is culture-independent — right for machine-readable data.</details>

---

## Operators

**Q16.** Why does `if (5)` not compile?
<details><summary>Answer</summary>C# has **no implicit int-to-bool conversion**. Conditions must be of type `bool`. This rules out the C-style `if (n)` meaning "if n != 0" and prevents the `if (x = 5)` (assignment instead of comparison) bug class.</details>

**Q17.** What's the result?
```csharp
bool b = false;
if (b && CrashIfCalled()) { ... }
```
<details><summary>Answer</summary>Doesn't crash. `&&` short-circuits — when the left is false, the right is never evaluated. Same with `||` when the left is true.</details>

**Q18.** Modern null-safe one-liner — make this safe:
```csharp
int len = user.Profile.Name.Length;
```
<details><summary>Answer</summary>`int? len = user?.Profile?.Name?.Length;` — null-conditional `?.` short-circuits the chain.</details>

**Q19.** What does `??=` do?
<details><summary>Answer</summary>Null-coalescing assignment. `x ??= y` is equivalent to `if (x == null) x = y;` (and the right side is only evaluated when needed).</details>

**Q20.** What's new about `?.=` in C# 14?
<details><summary>Answer</summary>Null-conditional assignment. `obj?.Prop = value` assigns only if `obj` is non-null; nothing happens if it is. The right side is **not evaluated** when target is null.</details>

---

## Control flow

**Q21.** What's printed?
```csharp
int n = 7;
string s = n switch {
    < 0 => "neg",
    0 => "zero",
    < 10 => "small",
    _ => "big"
};
Console.WriteLine(s);
```
<details><summary>Answer</summary>`"small"`. The switch expression matches the first arm whose pattern fits. `< 10` matches before `_`.</details>

**Q22.** Why does this fail?
```csharp
var list = new List<int> { 1, 2, 3 };
foreach (var x in list) {
    if (x % 2 == 0) list.Remove(x);
}
```
<details><summary>Answer</summary>`InvalidOperationException`. The `foreach` enumerator detects collection modification mid-iteration. Solutions: iterate a copy (`list.ToList()`), iterate backwards with `for`, or filter with LINQ and reassign.</details>

**Q23.** When would you use a switch expression vs a switch statement?
<details><summary>Answer</summary>Expression: returning a value, exhaustiveness matters, prefer functional style. Statement: side effects per case, `break`/`continue`/`return` from inside, multiple statements per case.</details>

**Q24.** What's the difference between `while (...)` and `do { ... } while (...)`?
<details><summary>Answer</summary>`while` tests at the **top** — body may run zero times. `do-while` tests at the **bottom** — body runs **at least once**.</details>

---

## Methods

**Q25.** What's the difference between `ref`, `out`, and `in`?
<details><summary>Answer</summary>
- `ref` — alias; caller must initialize first; method can read/write.
- `out` — alias; caller doesn't need to initialize; method **must** assign before returning.
- `in` — alias; method can read but **not** write; useful for large structs to avoid copies.
</details>

**Q26.** What does `params` do?
<details><summary>Answer</summary>Lets a method accept any number of arguments of one type, packaged as a collection. `void Sum(params int[] nums)` can be called `Sum(1, 2, 3)` or `Sum()`. C# 13 generalized to any collection; C# 14 introduces `params ReadOnlySpan<T>` for zero-allocation.</details>

**Q27.** Why is `throw ex;` worse than `throw;`?
<details><summary>Answer</summary>`throw ex;` re-throws but **resets** the stack trace — you lose the original throw location. `throw;` rethrows with the original stack intact.</details>

**Q28.** Predict and explain:
```csharp
void Greet(string greeting = "Hello") => Console.WriteLine(greeting);
Greet();
Greet("Hi");
```
<details><summary>Answer</summary>"Hello", "Hi". Optional parameters use their default when omitted.</details>

---

## Arrays

**Q29.** Difference between `int[,]` and `int[][]`?
<details><summary>Answer</summary>`int[,]` is **rectangular** — single contiguous block, fixed shape, indexed `[i, j]`. `int[][]` is **jagged** — array of array references, each row can be a different length, indexed `[i][j]`. Jagged is usually faster in practice due to JIT optimizations.</details>

**Q30.** What happens?
```csharp
string[] strings = { "a", "b" };
object[] objs = strings;
objs[0] = 42;
```
<details><summary>Answer</summary>`ArrayTypeMismatchException` at runtime. C# arrays are covariant (legacy Java-style decision), so the assignment to `object[]` compiles. But the underlying type is still `string[]`, so storing an int fails at runtime. Use `List<T>` (invariant) for type safety.</details>

**Q31.** Cheapest way to slice an array without copying?
<details><summary>Answer</summary>`Span<T>`: `arr.AsSpan(1, 3)` gives a window into the original memory. `arr[1..4]` allocates a new array.</details>

---

## Exceptions

**Q32.** When should you catch `Exception`?
<details><summary>Answer</summary>Almost never in business logic. Use it (a) at the very top of an app to log and possibly recover, (b) in a "catch, log, rethrow" pattern at module boundaries. Catching `Exception` and swallowing is an anti-pattern.</details>

**Q33.** What does this do differently from a regular `if` inside the catch?
```csharp
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) { ... }
```
<details><summary>Answer</summary>Exception filter (`when`). If the filter returns false, the stack **isn't unwound** — debugging sees the original throw point. Also, if `when` evaluates to false, the exception isn't logged as "caught and rethrown."</details>

**Q34.** What's the modern (`.NET 6+`) replacement for this?
```csharp
if (arg == null) throw new ArgumentNullException(nameof(arg));
```
<details><summary>Answer</summary>`ArgumentNullException.ThrowIfNull(arg);`. Uses `[CallerArgumentExpression]` so the parameter name is captured automatically.</details>

**Q35.** Throwing in a `finally` block — what happens?
<details><summary>Answer</summary>The new exception replaces the original (which is lost — no inner exception chaining). Avoid throwing in finally. If you must, catch within the finally and log.</details>

---

## Comments / XML docs

**Q36.** What does `/// <inheritdoc/>` do?
<details><summary>Answer</summary>Tells doc tools to copy the documentation from the base/interface member being overridden/implemented. Saves duplication.</details>

**Q37.** What does setting `<GenerateDocumentationFile>true</GenerateDocumentationFile>` in csproj do?
<details><summary>Answer</summary>The build produces an XML file next to the DLL containing all XML doc comments. Consumers and tooling pick it up for IntelliSense and doc generation. Also enables CS1591 warnings for missing summaries on public APIs.</details>

---

## Mixed / tricky

**Q38.** What does this print?
```csharp
var funcs = new List<Func<int>>();
for (int i = 0; i < 3; i++) funcs.Add(() => i);
foreach (var f in funcs) Console.Write(f());
```
<details><summary>Answer</summary>`333`. All lambdas capture the same `i` variable; by the time you call them, `i == 3`. Fix: introduce a fresh local inside the loop (`int copy = i;`) or use `foreach` which gives a new variable per iteration.</details>

**Q39.** Why prefer `IReadOnlyList<T>` over `T[]` for return types?
<details><summary>Answer</summary>Returning an array exposes the internal storage — callers can mutate it. `IReadOnlyList<T>` signals "you can read but not modify." For value types it's equivalent in performance; the abstraction protects invariants.</details>

**Q40.** What does this output and why?
```csharp
checked {
    int n = int.MaxValue;
    int doubled = n * 2;
}
```
<details><summary>Answer</summary>Throws `OverflowException`. `int.MaxValue * 2` overflows; `checked` makes it explicit.</details>

---

→ [Coding.md](Coding.md) — hands-on problems
