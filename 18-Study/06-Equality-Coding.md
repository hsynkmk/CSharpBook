# 06 — Equality — Coding Questions

> Predict the output / find the bug. (Concepts: [06-Equality.md](06-Equality.md))

---

### Q1 — class vs record equality
```csharp
class CPoint { public int X; public int Y; }
record RPoint(int X, int Y);

Console.WriteLine(new CPoint{X=1,Y=2}.Equals(new CPoint{X=1,Y=2}));
Console.WriteLine(new RPoint(1,2).Equals(new RPoint(1,2)));
```
<details><summary>Answer</summary>

**`False`** then **`True`**. A plain `class` uses **reference** equality by default (two different instances). A **record** generates **value** equality from its members. This is records' headline feature.
</details>

---

### Q2 — The == vs Equals dispatch trap
```csharp
object a = "hello";
object b = "hel" + "lo";        // (compile-time concat → also "hello")
Console.WriteLine(a == b);
Console.WriteLine(a.Equals(b));
```
<details><summary>Answer</summary>

Both **`True`** here, but for different reasons — and the subtlety: `==` on `object` is **reference** equality (not string `==`!). It's `True` only because the compiler **interns** both literals to the same instance. Change `b` to a runtime-built string and `a == b` would be **`False`** while `a.Equals(b)` stays **`True`**. Lesson: `==` binds to the **compile-time type** (`object` → reference), `Equals` is virtual (→ string value compare).
</details>

---

### Q3 — Override Equals but forget GetHashCode
```csharp
class Money { public int Amount;
    public override bool Equals(object? o) => o is Money m && m.Amount == Amount;
    // no GetHashCode override
}
var set = new HashSet<Money>();
set.Add(new Money{Amount=5});
Console.WriteLine(set.Contains(new Money{Amount=5}));
```
<details><summary>Answer</summary>

**`False`** (and a compiler warning). Without overriding `GetHashCode`, the two equal objects get **different** (reference-based) hash codes → land in different buckets → never compared. **Always override `GetHashCode` with `Equals`** (e.g., `=> Amount.GetHashCode();`).
</details>

---

### Q4 — Struct default equality cost
```csharp
struct Big { public int A, B, C; }
// In a tight loop comparing millions of Big values via .Equals — any concern?
```
<details><summary>Answer</summary>

Default `ValueType.Equals` uses **reflection** over fields (and can box) → slow. **Fix:** implement `IEquatable<Big>` (or use `record struct`) for a fast, allocation-free comparison.
</details>

---

### Q5 — Record inheritance equality
```csharp
record Animal(string Name);
record Dog(string Name) : Animal(Name);

Animal a = new Animal("Rex");
Animal d = new Dog("Rex");
Console.WriteLine(a.Equals(d));
```
<details><summary>Answer</summary>

**`False`**. Records compare the **`EqualityContract`** (runtime type) too — an `Animal` and a `Dog` with the same data are **not** equal. This sidesteps the inheritance symmetry problem but can surprise.
</details>

---

### Q6 — `with` and equality
```csharp
record P(int X, int Y);
var a = new P(1, 2);
var b = a with { Y = 2 };
Console.WriteLine(ReferenceEquals(a, b));
Console.WriteLine(a == b);
```
<details><summary>Answer</summary>

**`False`** then **`True`**. `with` creates a **new** instance (different reference) but with identical values, so value-equality `==` is `True`.
</details>

---

### Q7 — Bad GetHashCode
```csharp
class Box { public int A, B;
    public override int GetHashCode() => A ^ B;   // ?
    public override bool Equals(object? o) => o is Box x && x.A==A && x.B==B; }
```
<details><summary>Answer</summary>

`A ^ B` is a **poor** hash: `Box{1,2}` and `Box{2,1}` collide (XOR is commutative), and `Box{3,3}` hashes to 0. Lots of collisions → degraded O(n) dictionary perf. **Fix:** `HashCode.Combine(A, B)` (well-mixed, order-sensitive).
</details>

---

### Q8 — Float equality
```csharp
double x = 0.1 + 0.2;
Console.WriteLine(x == 0.3);
```
<details><summary>Answer</summary>

**`False`** — `0.1 + 0.2` is `0.30000000000000004` in binary floating point. Never compare floats/doubles with `==`; compare within a tolerance (`Math.Abs(x - 0.3) < 1e-9`).
</details>

---

### Q9 — IEqualityComparer to the rescue
```csharp
var names = new[] { "Ann", "ann", "BOB" };
Console.WriteLine(names.Distinct().Count());
Console.WriteLine(names.Distinct(StringComparer.OrdinalIgnoreCase).Count());
```
<details><summary>Answer</summary>

**`3`** then **`2`**. Default `Distinct` is case-sensitive ("Ann" ≠ "ann"); supplying `StringComparer.OrdinalIgnoreCase` treats them as equal → 2 distinct. Inject a comparer to customize equality without changing the type.
</details>

---

### Q10 — Senior: equality vs ordering consistency
```csharp
class V : IEquatable<V>, IComparable<V> {
    public int Major, Minor;
    public bool Equals(V? o) => o!=null && Major==o.Major && Minor==o.Minor;
    public int CompareTo(V? o) => Major.CompareTo(o!.Major);   // only Major!
    public override int GetHashCode() => HashCode.Combine(Major, Minor);
}
```
<details><summary>Answer</summary>

**Inconsistent**: `CompareTo` ignores `Minor`, so two values with same `Major` but different `Minor` are `CompareTo == 0` (ordering says "equal") yet `Equals` says **not** equal. Sorted collections / `Distinct`-by-order may behave surprisingly. Keep `CompareTo == 0` consistent with `Equals` (compare `Minor` too).
</details>
