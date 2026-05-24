# 08 — Modern C# — Coding Questions

> Predict the output / find the bug. (Concepts: [08-ModernCSharp.md](08-ModernCSharp.md))

---

### Q1 — record value equality
```csharp
record Person(string Name, int Age);
var a = new Person("Ann", 30);
var b = new Person("Ann", 30);
Console.WriteLine(a == b);
Console.WriteLine(ReferenceEquals(a, b));
```
<details><summary>Answer</summary>

**`True`** then **`False`**. Records generate value-based `==` (same Name+Age → equal), but they're still distinct heap objects (different references).
</details>

---

### Q2 — Records are shallow-immutable
```csharp
record Order(List<string> Items);
var a = new Order(new List<string> { "x" });
var b = a with { };
b.Items.Add("y");
Console.WriteLine(a.Items.Count);
```
<details><summary>Answer</summary>

**`2`**. `with` does a **shallow** copy — `a` and `b` share the *same* `List`. Records protect their own properties from reassignment, not the contents of mutable members. Use immutable collections for true immutability.
</details>

---

### Q3 — Switch expression non-exhaustive
```csharp
string Classify(int n) => n switch {
    > 0 => "positive",
    < 0 => "negative"
};
Console.WriteLine(Classify(0));
```
<details><summary>Answer</summary>

**Throws `SwitchExpressionException`** at runtime — `0` matches neither arm (compiler warns it's not exhaustive). Add a `_ => "zero"` or `0 => ...` arm.
</details>

---

### Q4 — NRT is compile-time only
```csharp
#nullable enable
string name = GetName();   // returns string (non-nullable annotation)
Console.WriteLine(name.Length);
static string GetName() => null!;   // lies to the compiler
```
<details><summary>Answer</summary>

**Throws `NullReferenceException`** at runtime. NRT is **compile-time analysis only** — `null!` suppresses the warning, but `null` still flows at runtime. Annotations are warnings, not guarantees; keep guard clauses for external input.
</details>

---

### Q5 — Pattern matching: list pattern
```csharp
int[] a = { 1, 2, 3, 4 };
var r = a switch {
    [var first, .., var last] => $"{first}..{last}",
    [] => "empty",
    _ => "other"
};
Console.WriteLine(r);
```
<details><summary>Answer</summary>

**`1..4`**. The list pattern `[var first, .., var last]` binds the first and last elements, with `..` matching the middle. (C# 11 list patterns.)
</details>

---

### Q6 — Property pattern
```csharp
record Person(string Name, int Age);
string Describe(Person p) => p switch {
    { Age: >= 18 } => "adult",
    { Age: > 0 }   => "minor",
    _              => "unknown"
};
Console.WriteLine(Describe(new Person("X", 18)));
```
<details><summary>Answer</summary>

**`adult`** — `Age: >= 18` matches first (patterns checked top to bottom). Property patterns test members; relational patterns (`>=`) compare.
</details>

---

### Q7 — required members
```csharp
class Config { public required string Url { get; init; } public int Timeout { get; init; } }
var c = new Config { Timeout = 30 };   // ?
```
<details><summary>Answer</summary>

**Compile error** — `Url` is `required` but not set in the initializer. `required` (C# 11) enforces "must be assigned at initialization" without a constructor, pairing with object initializers.
</details>

---

### Q8 — Target-typed new + collection expression
```csharp
List<int> a = new() { 1, 2 };
int[] b = [3, 4];
List<int> c = [..a, ..b, 5];
Console.WriteLine(string.Join(",", c));
```
<details><summary>Answer</summary>

**`1,2,3,4,5`**. `[..a, ..b, 5]` is a **collection expression** with the **spread** operator (`..`) — concatenates the elements of `a` and `b`, then appends `5`. (C# 12.)
</details>

---

### Q9 — Primary constructor capture
```csharp
class Counter(int start) {
    private int _value = start;
    public int Next() => ++_value;   // can it see 'start'?
}
var c = new Counter(10);
Console.WriteLine(c.Next());
```
<details><summary>Answer</summary>

**`11`**. Primary-constructor parameters (`start`) are in scope for the whole class; here `start` initializes `_value`. (C# 12.) Note: capturing a primary-ctor param directly in a method would create a hidden backing field.
</details>

---

### Q10 — C# 14 `field` keyword (senior)
```csharp
// What does the 'field' keyword let you do without declaring a backing field?
public string Name {
    get => field;
    set => field = value?.Trim() ?? "";
}
```
<details><summary>Answer</summary>

The **`field` keyword** (C# 14) gives the property accessor direct access to its **compiler-generated backing field** — so you can add logic (here, trimming/null-handling on set) **without manually declaring** a private field. Reduces boilerplate for "auto-property with a little logic".
</details>
