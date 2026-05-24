# 03 — Generics, Delegates, Events — Coding Questions

> Predict the output / find the bug. (Concepts: [03-Generics-Delegates-Events.md](03-Generics-Delegates-Events.md))

---

### Q1 — The famous closure-in-loop bug
```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++)
    actions.Add(() => Console.Write(i));
foreach (var a in actions) a();
```
<details><summary>Answer</summary>

**`333`**. All three lambdas capture the **same variable** `i` (one `i` for the whole `for` loop); by the time they run, `i == 3`. **Fix:** copy to a per-iteration local — `int copy = i; actions.Add(() => Console.Write(copy));` → `012`.
</details>

---

### Q2 — Same loop, but foreach
```csharp
var actions = new List<Action>();
foreach (var x in new[] { 1, 2, 3 })
    actions.Add(() => Console.Write(x));
foreach (var a in actions) a();
```
<details><summary>Answer</summary>

**`123`**. Since C# 5, `foreach` introduces a **fresh loop variable each iteration**, so each closure captures its own `x`. (Only the `for` loop shares one variable — Q1.)
</details>

---

### Q3 — What's the output?
```csharp
Func<int> f;
int n = 10;
f = () => n;
n = 20;
Console.WriteLine(f());
```
<details><summary>Answer</summary>

**`20`**. Closures capture the **variable**, not its value at capture time. When `f()` runs, it reads the current `n` (20).
</details>

---

### Q4 — Multicast delegate return value
```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;
Console.WriteLine(f());
```
<details><summary>Answer</summary>

**`3`**. A multicast delegate invokes all in order, but only the **last** return value is observed. (The 1 and 2 are computed and discarded.) Don't rely on returns from multicast delegates.
</details>

---

### Q5 — Will this compile?
```csharp
IEnumerable<string> strings = new List<string>();
IEnumerable<object> objects = strings;        // (a)
List<string> ls = new();
List<object> lo = ls;                          // (b)
```
<details><summary>Answer</summary>

**(a) compiles** — `IEnumerable<out T>` is **covariant** (T only comes out). **(b) fails** — `List<T>` is **invariant** (you could add an `object` that isn't a `string`). Variance applies to interfaces/delegates, not concrete generic classes.
</details>

---

### Q6 — Event leak: spot it
```csharp
class Publisher { public event Action? OnTick; public void Tick() => OnTick?.Invoke(); }
class Subscriber {
    public Subscriber(Publisher p) => p.OnTick += Handle;   // ?
    void Handle() { }
}
```
<details><summary>Answer</summary>

**Memory leak risk**: the long-lived `Publisher` holds a reference to each `Subscriber` via the event delegate. If subscribers are created/discarded but never `-=`, they **can't be garbage-collected** (rooted by the publisher). **Fix:** unsubscribe (`p.OnTick -= Handle`) in `Dispose`, or use weak events. ([13-MemoryAndGC.md](13-MemoryAndGC.md))
</details>

---

### Q7 — Why box? Fix it.
```csharp
T Max<T>(T a, T b) where T : IComparable    // non-generic IComparable
    => a.CompareTo(b) >= 0 ? a : b;
Max(3, 5);   // any hidden cost?
```
<details><summary>Answer</summary>

`where T : IComparable` (non-generic) **boxes** value types on the `CompareTo(object)` call. **Fix:** `where T : IComparable<T>` — the generic interface avoids boxing.
</details>

---

### Q8 — What's the output?
```csharp
Action<string> log = Console.WriteLine;
Action<string> log2 = s => Console.WriteLine($"[{s}]");
var combined = log + log2;
combined("hi");
```
<details><summary>Answer</summary>

```
hi
[hi]
```
Both delegates run in order (Action returns void, so no return-value issue). Multicast invokes each subscriber.
</details>

---

### Q9 — Static lambda (C# 9): what's the point?
```csharp
Func<int,int> f = static x => x * 2;     // vs  x => x * factor;
```
<details><summary>Answer</summary>

`static` lambdas **cannot capture** any enclosing variable (compile error if you try). The benefit: guarantees **no closure allocation** and prevents accidental captures — useful in hot paths. A non-static lambda capturing `factor` allocates a closure object.
</details>

---

### Q10 — Generic specialization (senior)
```csharp
// Conceptual: how many native method bodies does the JIT generate?
List<int>  a = new();
List<long> b = new();
List<string> c = new();
List<object> d = new();
```
<details><summary>Answer</summary>

Roughly: **one specialized body each** for `int` and `long` (value types are monomorphized — different layouts), and **one shared** body for `string` **and** `object` (all reference-type `T` share code since references are uniform pointers). This is why value-type generics are box-free but produce more JIT'd code.
</details>
