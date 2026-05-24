# Chapter 10 — Questions

> 30+ drilling questions on advanced language features: NRT, deep patterns, records, required, file types, top-level, collection expressions, raw strings, u8 literals, interpolated handlers, caller info.

---

## Nullable Reference Types

**Q1.** What's the runtime difference between `string` and `string?`?
<details><summary>Answer</summary>None at runtime. Both are `System.String`. NRT is metadata + compile-time analysis only. The compiler emits `Nullable*` attributes that other compilers + tools read.</details>

**Q2.** When does `var n = obj.Length` after `if (obj is not null)` not emit a warning?
<details><summary>Answer</summary>The compiler's flow analysis recognizes the null check and narrows `obj`'s null state to "not null" inside the if branch. After the branch, state reverts. The narrowing is intra-method only — across method boundaries you'd need NotNullWhen-style attributes.</details>

**Q3.** What's `[NotNullWhen(true)]` for?
<details><summary>Answer</summary>On an `out` parameter, says "when the method returns true, this out is not null." Lets `if (TryGet(out var x)) { x.Method(); }` compile without warnings — the compiler trusts the contract.</details>

---

## Pattern Matching

**Q4.** What's a slice pattern?
<details><summary>Answer</summary>`..` inside a list pattern — matches zero or more elements. `arr is [1, .., 3]` means "starts with 1, ends with 3, any number of elements between." At most one `..` per list pattern. Can capture: `arr is [_, .. var middle, _]`.</details>

**Q5.** Predict:
```csharp
return arr switch {
    [] => "empty",
    [_] => "one",
    [.., _] => "two or more"
};
```
For `arr = new[] { 1 }`:
<details><summary>Answer</summary>"one". `[_]` matches a single-element array first. Arms try top-to-bottom.</details>

**Q6.** What does `when` do that property patterns don't?
<details><summary>Answer</summary>Arbitrary boolean expressions, not just property matches. Useful when the condition can't be expressed structurally — e.g., `when n > someVariable` or `when CallExpensiveCheck(x)`. But `when` is opaque to the compiler's exhaustiveness analysis.</details>

---

## Records

**Q7.** What does `EqualityContract` do?
<details><summary>Answer</summary>A virtual property returning the runtime Type. Used in synthesized record equality to ensure two records of different types (e.g., Animal and Dog) aren't equal even if their members match. Avoids the asymmetric-equality trap in class hierarchies.</details>

**Q8.** What's `<Clone>$`?
<details><summary>Answer</summary>The compiler-generated method behind `with` expressions. It calls the copy constructor to produce a new instance with all members copied. `with { X = 5 }` calls `<Clone>$` then assigns X on the copy. The illegal-in-source name (`<>$`) ensures no conflict with user code.</details>

**Q9.** Why doesn't this work?
```csharp
public record Cart(List<string> Items);
new Cart([..]) == new Cart([..]);   // both empty
```
<details><summary>Answer</summary>`List<T>` doesn't override Equals. Record's synthesized equality uses `EqualityComparer<T>.Default`, which for List is reference equality. Two different list instances → records not equal even with same content. Use `ImmutableArray<string>` (structural equality) or override Equals.</details>

---

## Required Members

**Q10.** Why does this fail to compile?
```csharp
public class Person {
    public required string Name { get; init; }
}
new Person();
```
<details><summary>Answer</summary>`required` enforces that `Name` must be set via object initializer (or a `[SetsRequiredMembers]` constructor). `new Person()` doesn't set Name → compile error CS9035.</details>

**Q11.** When would you use `[SetsRequiredMembers]`?
<details><summary>Answer</summary>On a constructor that initializes all required members — tells the compiler "this ctor handles them." Allows callers to use `new Person("Alice", 30)` instead of `new Person { Name = "...", Age = ... }`. Trust-based: the compiler doesn't verify the body actually sets them.</details>

---

## File-scoped types

**Q12.** Where is `file class X` visible?
<details><summary>Answer</summary>Only within the same source file. Even other files in the same assembly / namespace can't see it. Primary use: source generators emitting per-file helpers without name collisions.</details>

---

## Top-level statements

**Q13.** Why can only one file in a project have top-level statements?
<details><summary>Answer</summary>They synthesize a `Program.Main` method. Two files → two synthesized Mains → ambiguity for the entry point. The compiler enforces one.</details>

**Q14.** Can you write a `class Program { static void Main }` alongside top-level statements?
<details><summary>Answer</summary>No. The compiler synthesizes Program when top-level statements exist; explicit `class Program` conflicts. Compile error.</details>

---

## Global / implicit usings

**Q15.** What does `<ImplicitUsings>enable</ImplicitUsings>` do?
<details><summary>Answer</summary>The SDK injects a project-type-specific set of global usings (System, System.Linq, System.IO, etc., plus Web-specific ones for Web SDKs). Reduces boilerplate at the top of every file.</details>

**Q16.** How would you make `Math.Sqrt(x)` callable as just `Sqrt(x)` everywhere?
<details><summary>Answer</summary>
```xml
<Using Include="System.Math" Static="True" />
```
in csproj, or `global using static System.Math;` in a source file. Adds `using static System.Math` globally — all static members of Math accessible unqualified.
</details>

---

## Collection Expressions

**Q17.** Predict the type of `[1, 2, 3]` in this context:
```csharp
ReadOnlySpan<int> s = [1, 2, 3];
```
<details><summary>Answer</summary>`ReadOnlySpan<int>`. The compiler targets the destination type and picks efficient construction — often `stackalloc int[3] { 1, 2, 3 }` for spans. Zero heap allocation.</details>

**Q18.** What's `[..a, x, ..b]`?
<details><summary>Answer</summary>A collection expression with spread — concatenates `a`'s elements, then `x`, then `b`'s. Result type determined by context. The `..` is the spread operator.</details>

**Q19.** Does `var x = [1, 2, 3];` compile?
<details><summary>Answer</summary>No. `var` can't infer a type from a collection expression alone (could be `int[]`, `List<int>`, etc.). Specify: `int[] x = [1, 2, 3];` or `var x = (int[])[1, 2, 3];`.</details>

---

## Raw String Literals

**Q20.** What determines indentation in a multi-line raw string?
<details><summary>Answer</summary>The column position of the closing `"""`. That column's worth of leading whitespace is stripped from each line. Lines with less indent than the baseline = compile error.</details>

**Q21.** What's `$$"""...{{x}}..."""`?
<details><summary>Answer</summary>Interpolated raw string with TWO `$` symbols → uses `{{ }}` for placeholders (so single `{` and `}` are literal). Useful for embedded JSON, CSS, etc., where curly braces are common literals.</details>

---

## UTF-8 String Literals

**Q22.** What's the type of `"hello"u8`?
<details><summary>Answer</summary>`ReadOnlySpan<byte>`. The bytes are UTF-8 encoded and embedded in the assembly. Zero allocation at runtime.</details>

**Q23.** Why use `"GET "u8` instead of `Encoding.UTF8.GetBytes("GET ")`?
<details><summary>Answer</summary>The `u8` literal is compile-time (bytes baked into the assembly, zero allocation). `GetBytes` allocates a new byte array per call — fine occasionally, disastrous in a hot loop (HTTP server comparing methods). For high-throughput byte-level work, `u8` is the right tool.</details>

**Q24.** Why can't `"x"u8` be a class field?
<details><summary>Answer</summary>`ReadOnlySpan<byte>` is a `ref struct` — can't be a class field. Solution: use a static getter property:
```csharp
private static ReadOnlySpan<byte> Greeting => "Hello"u8;
```
Each call returns a fresh span over the same embedded bytes — no allocation, no field-storage constraint.</details>

---

## Interpolated String Handlers

**Q25.** Why is `_logger.LogDebug($"Slow: {ExpensiveOp()}")` faster than the pre-handler version?
<details><summary>Answer</summary>The interpolated handler defers argument evaluation. If debug isn't enabled, `ExpensiveOp()` isn't called AND no string is built. Pre-handler: arguments evaluated even when logging is disabled.</details>

**Q26.** Does `DefaultInterpolatedStringHandler` allocate?
<details><summary>Answer</summary>For small interpolated strings, uses `stackalloc char[]` — zero allocation. For larger, falls back to `ArrayPool<char>`. The final string is allocated once via `ToStringAndClear`. Net: usually 1 allocation (the result), vs `string.Format` which allocates more.</details>

---

## Caller Info Attributes

**Q27.** What does `[CallerArgumentExpression(nameof(arg))]` do?
<details><summary>Answer</summary>The compiler injects the source text of the named argument as a string at the call site. `ThrowIfNull(user.Name)` becomes `ThrowIfNull(user.Name, "user.Name")`. Used by `ArgumentNullException.ThrowIfNull` and other guards for precise error messages.</details>

**Q28.** When does `[CallerMemberName]` give you "Item"?
<details><summary>Answer</summary>When called from an indexer setter — the synthesized member name for indexers is "Item". Unusual; you'd typically call it from regular method bodies or property setters where the name is more meaningful.</details>

---

## Checked Operators

**Q29.** What's the default arithmetic behavior in C#?
<details><summary>Answer</summary>`unchecked` — silent wraparound on overflow. `int.MaxValue + 1` becomes `int.MinValue`. Use `checked` (per-expression, per-block, or project-wide) to throw `OverflowException` instead.</details>

**Q30.** Does `checked` affect floating-point arithmetic?
<details><summary>Answer</summary>No. Float / double produce infinity or NaN on overflow regardless. `decimal` already throws on overflow regardless. `checked` is specific to integer types.</details>

**Q31.** What's a user-defined checked operator?
<details><summary>Answer</summary>C# 11+. A separate operator overload that runs in `checked` contexts: `public static T operator checked +(T a, T b)`. Lets custom numeric types behave differently when overflow checking is requested. Used by `INumber<T>` implementations.</details>

---

## Synthesis

**Q32.** Why prefer interpolated strings over `string.Format`?
<details><summary>Answer</summary>
- Compile-time type checking of arguments.
- Compiler optimizes via DefaultInterpolatedStringHandler — uses stackalloc + ArrayPool.
- More readable.
- Conditional evaluation when consumed by handler-aware APIs (logging, asserts).
`string.Format` is fine for legacy contexts, dynamic format strings, or when you specifically need its parameter array form.
</details>

**Q33.** Build a guard method using all relevant features:
<details><summary>Solution</summary>

```csharp
public static T NotNull<T>(
    [NotNull] T? value,
    [CallerArgumentExpression(nameof(value))] string? expr = null,
    [CallerMemberName] string? caller = null,
    [CallerLineNumber] int line = 0) where T : class
{
    if (value is null) {
        throw new ArgumentNullException(
            expr,
            $"Null detected for '{expr}' in {caller} at line {line}");
    }
    return value;
}

// Use
var user = NotNull(repo.FindById(id));
// On null: throws with rich context — "Null detected for 'repo.FindById(id)' in MyMethod at line 42"
```

Combines NotNull (for the compiler to track non-null after the call) + CallerArgumentExpression (capture the calling expression) + CallerMemberName / LineNumber (context).
</details>

**Q34.** A coworker writes:
```csharp
ReadOnlySpan<byte> data = "x"u8;
field = data;   // ⚠
```
What's wrong?
<details><summary>Answer</summary>If `field` is a class field (instance or static), this won't compile — Span is a ref struct, can't be a class field. Convert to byte[] (`"x"u8.ToArray()`, allocates) or use a static getter property. Or use ReadOnlyMemory<byte> if you need it field-storable.</details>

**Q35.** Open-ended: why is the modern C# entry point cleaner than the classic one?
<details><summary>Sample answer</summary>
Modern (.NET 6+ default):
```csharp
// Program.cs
Console.WriteLine("hi");
```

Classic:
```csharp
using System;
namespace MyApp {
    class Program {
        static void Main(string[] args) {
            Console.WriteLine("hi");
        }
    }
}
```

Modern wins via:
- **Top-level statements**: no class wrapper, no Main signature.
- **Implicit usings**: no manual `using System;`.
- **File-scoped namespace** (if you add one): no indentation.
- **Args implicit**: just use `args` if needed.

Result: the first thing the reader sees is the actual code, not boilerplate. Better for teaching, scripts, demos, and Minimal API entry points.
</details>

---

→ [Coding.md](Coding.md)
