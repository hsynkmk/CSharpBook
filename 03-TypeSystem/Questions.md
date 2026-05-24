# Chapter 03 — Questions

> Drilling for everything in Chapter 03. The type system is foundational — every later chapter assumes you understand these.

---

## Value vs reference

**Q1.** What's the difference between a value type and a reference type in one sentence?
<details><summary>Answer</summary>A value-type variable holds the value directly; a reference-type variable holds a pointer to a heap-allocated object.</details>

**Q2.** Predict the output:
```csharp
class Box { public int X; }
var a = new Box { X = 5 };
var b = a;
b.X = 10;
Console.WriteLine(a.X);
```
<details><summary>Answer</summary>`10`. `b = a` copied the reference, so both point at the same object. Mutation via b is visible via a.</details>

**Q3.** Predict the output:
```csharp
struct Point { public int X; }
var a = new Point { X = 5 };
var b = a;
b.X = 10;
Console.WriteLine(a.X);
```
<details><summary>Answer</summary>`5`. Struct assignment copies the value. a and b are independent.</details>

**Q4.** What's the difference between passing a `List<int>` by value vs by `ref`?
<details><summary>Answer</summary>By value: a copy of the **reference** is passed. The method can mutate the list contents (visible to caller) but can't reassign caller's variable to a new list. By ref: the parameter is an alias for the caller's variable; the method can reassign it to a different list.</details>

---

## Structs

**Q5.** What does `readonly struct` enforce?
<details><summary>Answer</summary>All instance fields must be readonly, auto-property setters must be `init`, and instance methods can't mutate `this`. The compiler can also elide defensive copies when calling methods through readonly fields.</details>

**Q6.** Why does this fail to compile?
```csharp
List<Point> list = new() { new() { X = 1 } };
list[0].X = 99;
```
<details><summary>Answer</summary>`list[0]` returns a **copy** of the struct (indexers don't return mutable references for value types in `List<T>`). The compiler refuses the assignment because it would modify a transient copy. Fix: read into a local, modify, write back; or use a `readonly` struct + with-expression; or use a class.</details>

**Q7.** When should a struct be `readonly record struct`?
<details><summary>Answer</summary>Almost always — for small value-like types you'd otherwise hand-write. It gives you immutability, value equality, deconstruction, `with` expressions, and good defaults for free.</details>

**Q8.** What's wrong with this struct?
```csharp
public struct Money(decimal amount, string currency) {
    public decimal Amount { get; } = amount;
    public string Currency { get; } = currency;
}

Money m = default;
Console.WriteLine(m.Currency.ToUpper());
```
<details><summary>Answer</summary>`default(Money)` produces a struct with `Currency = null` (the default for a reference type). The `.ToUpper()` then throws NullReferenceException. You can't prevent `default` for structs. Either use a class (where you can validate in ctor) or design the struct to tolerate the default value.</details>

---

## Records

**Q9.** What does the compiler synthesize for `public record Person(string Name, int Age);`?
<details><summary>Answer</summary>Init-only properties `Name` and `Age`, a constructor, a `Deconstruct` method, value-based `Equals` (with `EqualityContract` checking runtime type), `GetHashCode`, `ToString`, `==` / `!=` operators, a copy constructor, and a `<Clone>$` method backing `with` expressions.</details>

**Q10.** Predict:
```csharp
public record Cart(int UserId, List<string> Items);
var a = new Cart(1, new() { "milk" });
var b = new Cart(1, new() { "milk" });
Console.WriteLine(a == b);
```
<details><summary>Answer</summary>`false`. Records use `EqualityComparer<T>.Default` per member, and `List<T>` doesn't override Equals → reference comparison. Two distinct list instances → not equal.</details>

**Q11.** When would you use a `record class` vs a `record struct`?
<details><summary>Answer</summary>`record class` (the default `record`): reference type, larger objects, supports inheritance among records. `record struct`: value type, small data (~16 bytes), no heap allocation, no inheritance. Use `readonly record struct` for small value-like types; `record` for larger DTOs or hierarchies.</details>

**Q12.** What does `with` do?
<details><summary>Answer</summary>Creates a new record by copying the original (via the synthesized copy ctor) and then applying the listed property assignments. The original is untouched.</details>

---

## Enums

**Q13.** Why does this print "999" instead of throwing?
```csharp
enum Severity { Info, Warning, Error }
Severity s = (Severity)999;
Console.WriteLine(s);
```
<details><summary>Answer</summary>The cast doesn't validate. Enums in C# are just typed integers. Use `Enum.IsDefined<Severity>(value)` to validate.</details>

**Q14.** What does `[Flags]` change about an enum?
<details><summary>Answer</summary>Mainly `ToString` behavior — it composes member names with " | ". Doesn't change runtime semantics. Convention is to assign each member a distinct power of two so bitwise OR / AND combine them cleanly.</details>

**Q15.** Difference between `p.HasFlag(Permissions.Read)` and `(p & Permissions.Read) != 0`?
<details><summary>Answer</summary>Modern (.NET Core 2.1+) `HasFlag` is a JIT intrinsic — same speed. Pre-2.1, `HasFlag` boxed both operands; the bitwise form was faster. Either is fine now; the bitwise form is still slightly more explicit about intent.</details>

---

## Tuples

**Q16.** Predict:
```csharp
(int X, int Y) a = (1, 2);
(int Width, int Height) b = (1, 2);
Console.WriteLine(a == b);
```
<details><summary>Answer</summary>`true`. Tuple names are compile-time metadata; both are `ValueTuple<int, int>` at runtime. Equality is element-by-element.</details>

**Q17.** What's the practical limit on tuple size?
<details><summary>Answer</summary>Compiler supports up to 7 named elements directly; 8+ uses a recursive "rest" tuple. Beyond 3-4, switch to a record or class for readability.</details>

**Q18.** When does a tuple allocate on the heap?
<details><summary>Answer</summary>When boxed (`object o = (1, 2);`) or stored as `dynamic`. Otherwise it's a struct — stack, register, or inline in another type.</details>

---

## Nullable

**Q19.** What's the runtime type difference between `int?` and `string?`?
<details><summary>Answer</summary>`int?` is `Nullable<int>` — a real wrapping struct holding `HasValue` and `Value`. `string?` is just `string` with compile-time nullability metadata; the runtime type is identical to non-nullable `string`.</details>

**Q20.** Predict:
```csharp
int? n = 5;
object o = n;
Console.WriteLine(o.GetType());
```
<details><summary>Answer</summary>`System.Int32`. Boxing `Nullable<T>` with `HasValue == true` produces a boxed `T`, not a boxed `Nullable<T>`. Boxing one with `HasValue == false` produces null.</details>

**Q21.** What does the `!` (null-forgiving) operator do at runtime?
<details><summary>Answer</summary>Nothing. It's purely a compile-time hint to suppress NRT warnings. The runtime sees no difference.</details>

**Q22.** What's the C# 14 `?.=` operator?
<details><summary>Answer</summary>Null-conditional assignment. `obj?.Prop = value` assigns to `obj.Prop` only if `obj` is non-null; otherwise it's a no-op. The right side is NOT evaluated when target is null.</details>

---

## Boxing

**Q23.** Where does boxing happen in this code?
```csharp
int n = 5;
Console.WriteLine($"n = {n}");
ArrayList list = new();
list.Add(n);
IComparable c = n;
```
<details><summary>Answer</summary>
- `Console.WriteLine($"n = {n}")` — C# 10+ interpolated string handlers avoid the box for primitives, so likely NO box. (Pre-handler era: would box.)
- `list.Add(n)` — boxes (ArrayList stores objects).
- `IComparable c = n` — boxes (assigning value type to interface).
</details>

**Q24.** How much memory does a boxed `int` take on 64-bit?
<details><summary>Answer</summary>24 bytes: 8 sync block + 8 MT pointer + 4 int + 4 padding.</details>

**Q25.** How does implementing `IEquatable<T>` reduce boxing for a struct used as a dictionary key?
<details><summary>Answer</summary>Without `IEquatable<T>`, the default `Equals(object?)` accepts an object — meaning the other key gets boxed for each comparison. With `IEquatable<T>`, the dictionary's `EqualityComparer<T>.Default` uses the strongly-typed `Equals(T)` overload — no boxing.</details>

---

## Conversions

**Q26.** Predict:
```csharp
double d = 3.7;
int i = (int)d;
Console.WriteLine(i);
```
<details><summary>Answer</summary>`3`. Casting double to int **truncates toward zero**, not rounds. Use `Math.Round(d)` if you want rounding.</details>

**Q27.** What does `checked` do?
<details><summary>Answer</summary>Enables overflow detection for integer arithmetic. Throws `OverflowException` instead of silently wrapping. Default for arithmetic is `unchecked` (for performance); checked is opt-in per expression or block.</details>

**Q28.** Difference between `(string)obj` and `obj as string`?
<details><summary>Answer</summary>`(string)obj` throws `InvalidCastException` if obj isn't a string (or null). `obj as string` returns null on mismatch. Use `as` (or `is string s`) when failure is recoverable; use the cast when failure is a bug.</details>

**Q29.** When would you define a user-defined `implicit` operator?
<details><summary>Answer</summary>Only when the conversion is genuinely safe and natural — no surprise to callers, no data loss, no obvious "view" implication. Examples: `Celsius → double`, `Money → decimal` (if amount is the natural numeric form), `Duration → TimeSpan`. When in doubt, use `explicit` so callers see the cast.</details>

---

## Pattern matching

**Q30.** Predict:
```csharp
return n switch {
    > 10 => "big",
    > 0 => "small",
    0 => "zero",
    _ => "negative"
};
```
For n = 5?
<details><summary>Answer</summary>`"small"`. Arms are evaluated top-down; `> 0` matches before `0`.</details>

**Q31.** What's a list pattern? Give an example.
<details><summary>Answer</summary>C# 11 feature for matching elements of array-like types. Example: `arr is [1, _, .., var last]` matches arrays starting with 1, followed by any element, then any number of middle elements, ending with something bound to `last`.</details>

**Q32.** What's wrong here?
```csharp
return shape switch {
    Circle => Math.PI * 5 * 5,
    Square => 4
};
```
<details><summary>Answer</summary>(a) No `_` fallback — non-exhaustive, may give a warning. (b) `Circle` and `Square` are type patterns but unused — you'd want `Circle c => Math.PI * c.Radius * c.Radius` to use the bound variable. (c) Missing default `_` would throw on unexpected input.</details>

---

## Anonymous types

**Q33.** Predict:
```csharp
var a = new { X = 1, Y = 2 };
var b = new { X = 1, Y = 2 };
Console.WriteLine(a == b);
Console.WriteLine(a.Equals(b));
```
<details><summary>Answer</summary>`==` is **false** (anonymous types don't overload `==`, defaults to reference equality). `Equals` is **true** (synthesized value equality). Subtle inconsistency.</details>

**Q34.** Why can't you return an anonymous type from a public method easily?
<details><summary>Answer</summary>You can't name the type. The compiler-generated name is illegal in C# source. Workarounds: return `dynamic` (loose), `object` (caller reflects), or use a generic-helper template (rarely useful). The clean answer: use a record or named class.</details>

---

## Synthesis

**Q35.** Walk through what happens at runtime when you do this:
```csharp
record Person(string Name);
Person a = new("Alice");
Person b = a with { Name = "Alice2" };
```
<details><summary>Answer</summary>
1. `new("Alice")` allocates a `Person` on the heap with Name = "Alice".
2. `a with { Name = "Alice2" }`:
   - Calls the synthesized `<Clone>$()` method, which calls the synthesized copy constructor.
   - Copy ctor allocates a new Person and copies all fields/properties from `a`.
   - Then sets `Name = "Alice2"` on the copy.
3. `b` points to the new heap object; `a` is unchanged.
</details>

**Q36.** You have a 1M-element `List<Point>` (struct). Why is iterating with `for (int i; i < list.Count; i++) Process(list[i])` faster than with `foreach`?
<details><summary>Answer</summary>Usually they're the same speed because `List<T>`'s enumerator is a struct and gets inlined. BUT: `foreach (Point p in list)` copies each element (struct), same as `list[i]`. If `Process(in Point p)` takes by `in` reference and you call `Process(list[i])`, there's still a copy because the indexer returns by value. There's no way to avoid the copy without `Span<T>` or `CollectionsMarshal.AsSpan(list)`.</details>

**Q37.** You add `IEquatable<Coord>` to a struct used as a dictionary key. What changes?
<details><summary>Answer</summary>Dictionary lookups go from reflection-based field-by-field comparison (slow, may box) to a direct call to your `bool Equals(Coord other)` method. Often 10-50× faster on hot paths.</details>

**Q38.** What does this print and why?
```csharp
int? n = null;
int sum = n + 5 ?? 0;
Console.WriteLine(sum);
```
<details><summary>Answer</summary>`0`. `n + 5` is lifted: `null + 5 = null`. Then `null ?? 0 = 0`. Result: 0. Watch operator precedence — `?? ` is very low, so this parses fine. With parens for clarity: `(n + 5) ?? 0`.</details>

**Q39.** Sorted from fastest to slowest equality check on a value-type struct `MyStruct`:
- a) `s1.Equals(s2)` where MyStruct implements `IEquatable<MyStruct>`.
- b) `s1.Equals((object)s2)`.
- c) `EqualityComparer<MyStruct>.Default.Equals(s1, s2)` where MyStruct implements `IEquatable<MyStruct>`.
- d) `s1.Equals(s2)` where MyStruct does NOT implement IEquatable.

<details><summary>Answer</summary>
Fastest to slowest:
- **a** — direct strongly-typed call, no box.
- **c** — same call after one dispatch; comparable to (a).
- **d** — falls back to reflection on ValueType.Equals; very slow.
- **b** — boxes both s1 (this) and s2 (arg). Slowest.

(c) is approximately as fast as (a) because `EqualityComparer<T>.Default` picks the `IEquatable<T>` implementation if available.
</details>

**Q40.** Open-ended: how would you model "an order is either pending, shipped, or cancelled, and each state carries different data" in modern C#?
<details><summary>Sample answer</summary>A sealed record hierarchy + pattern matching:
```csharp
public abstract record OrderState;
public sealed record Pending(DateTime CreatedAt) : OrderState;
public sealed record Shipped(DateTime ShippedAt, string TrackingNumber) : OrderState;
public sealed record Cancelled(DateTime CancelledAt, string Reason) : OrderState;

string Display(OrderState s) => s switch {
    Pending p => $"pending since {p.CreatedAt}",
    Shipped s => $"shipped {s.ShippedAt} via {s.TrackingNumber}",
    Cancelled c => $"cancelled: {c.Reason}",
    _ => throw new()
};
```
This is C#'s pragmatic discriminated union: data, value equality, exhaustiveness via pattern matching, no separate "type tag" field needed.</details>

---

→ [Coding.md](Coding.md)
