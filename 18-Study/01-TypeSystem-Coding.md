# 01 — Type System — Coding Questions

> Predict the output / find the bug. Cover the answer, decide, then reveal. (Concepts: [01-TypeSystem.md](01-TypeSystem.md))

---

### Q1 — What's the output?
```csharp
struct Point { public int X; }
var a = new Point { X = 1 };
var b = a;
b.X = 99;
Console.WriteLine(a.X);
```
<details><summary>Answer</summary>

**`1`**. `Point` is a **value type** — `b = a` copies the bytes, so `b` is independent. Mutating `b.X` doesn't touch `a`. (If `Point` were a `class`, the answer would be `99` — both reference the same object.)
</details>

---

### Q2 — What's the output?
```csharp
int i = 5;
object o = i;     // box
i = 10;
Console.WriteLine(o);
```
<details><summary>Answer</summary>

**`5`**. Boxing **copies** the value into a new heap object at the moment of assignment. Later changing `i` doesn't affect the boxed copy.
</details>

---

### Q3 — Reference type, but...
```csharp
string s = "hello";
string t = s;
t += " world";
Console.WriteLine(s);
```
<details><summary>Answer</summary>

**`hello`**. `string` is a reference type but **immutable** — `t += ...` creates a *new* string and reassigns `t`; `s` still points at the original. (Mutating a normal mutable class object *would* be visible through both references.)
</details>

---

### Q4 — Find the performance bug
```csharp
var list = new System.Collections.ArrayList();
for (int i = 0; i < 1_000_000; i++)
    list.Add(i);          // ?
```
<details><summary>Answer</summary>

Every `Add(i)` **boxes** the `int` (ArrayList stores `object`) → a million heap allocations + GC pressure. **Fix:** use `List<int>` (generic, no boxing).
</details>

---

### Q5 — What's the output?
```csharp
struct S { public int X; }
S s = default;
Console.WriteLine(s.X is 0);
Console.WriteLine((default(string)) is null);
```
<details><summary>Answer</summary>

**`True`** then **`True`**. `default` of a struct is a real instance with all fields zeroed (`X == 0`); `default` of a reference type is `null`. A struct is never "null".
</details>

---

### Q6 — Does this allocate on the heap?
```csharp
int? a = 5;
int? b = null;
```
<details><summary>Answer</summary>

**No.** `int?` is `Nullable<int>` — a **struct** (`HasValue` + value), stored inline. (Boxing an `int?` *would* allocate — it boxes the underlying `int`, or yields `null` if no value.)
</details>

---

### Q7 — What's the output?
```csharp
object x = 1;
object y = 1;
Console.WriteLine(ReferenceEquals(x, y));
Console.WriteLine(x.Equals(y));
```
<details><summary>Answer</summary>

**`False`** then **`True`**. Boxing `1` twice creates **two separate** heap objects (different references → `ReferenceEquals` false). But `Equals` compares the boxed **values** → `True`.
</details>

---

### Q8 — Why won't this compile / what's the smell?
```csharp
struct Counter { public int Count; public void Inc() => Count++; }
var list = new List<Counter> { new Counter() };
list[0].Inc();   // ?
```
<details><summary>Answer</summary>

**Compile error** — `list[0]` returns a **copy** of the struct (the indexer returns by value), so mutating it would be pointless; the compiler forbids calling a mutating method on it. This is the "mutable structs are evil" trap. **Fix:** make the struct immutable (return a new one), or use a class.
</details>

---

### Q9 — Predict the output
```csharp
Console.WriteLine(default(int));
Console.WriteLine(default(bool));
Console.WriteLine(default(DateTime));
Console.WriteLine(default(int?) is null);
```
<details><summary>Answer</summary>

`0` / `False` / `01/01/0001 00:00:00` (DateTime.MinValue) / `True`. Value types default to zeroed; `int?` (Nullable) defaults to "no value" → null.
</details>

---

### Q10 — Senior: what does `in` change here?
```csharp
readonly struct Big { public readonly long A, B, C, D; }
void Use(in Big b) { /* ... */ }    // vs void Use(Big b)
```
<details><summary>Answer</summary>

`in` passes the large struct **by readonly reference** (no 32-byte copy per call) — a perf win for large readonly structs. Because `Big` is a **`readonly struct`**, the compiler avoids **defensive copies** on member access. (For a *non-readonly* struct passed by `in`, each member call would defensively copy it, defeating the purpose.)
</details>
