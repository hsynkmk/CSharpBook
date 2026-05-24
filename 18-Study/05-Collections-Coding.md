# 05 — Collections — Coding Questions

> Predict the output / find the bug. (Concepts: [05-Collections.md](05-Collections.md))

---

### Q1 — Mutable key disaster
```csharp
class Key { public int Id; public override int GetHashCode() => Id; public override bool Equals(object? o) => o is Key k && k.Id == Id; }
var dict = new Dictionary<Key,string>();
var k = new Key { Id = 1 };
dict[k] = "one";
k.Id = 2;                       // mutate after insertion
Console.WriteLine(dict.TryGetValue(k, out _));
```
<details><summary>Answer</summary>

**`False`**. The key was bucketed by hash `1`; after mutating to `2`, lookup hashes to bucket `2` and can't find it. The entry is "lost". **Never mutate a field used in `GetHashCode` after using the object as a key** — use immutable keys.
</details>

---

### Q2 — Why can't it find the key?
```csharp
class Point { public int X, Y; public Point(int x,int y){X=x;Y=y;} }
var d = new Dictionary<Point,string>();
d[new Point(1,1)] = "a";
Console.WriteLine(d.ContainsKey(new Point(1,1)));
```
<details><summary>Answer</summary>

**`False`**. `Point` doesn't override `Equals`/`GetHashCode`, so it uses **reference equality** — a *different* `new Point(1,1)` instance doesn't match. **Fix:** override both (or use a `record`/`record struct`). ([06-Equality.md](06-Equality.md))
</details>

---

### Q3 — Modifying during enumeration
```csharp
var list = new List<int> { 1, 2, 3 };
foreach (var x in list) {
    if (x == 2) list.Remove(x);
}
```
<details><summary>Answer</summary>

**Throws `InvalidOperationException`** ("Collection was modified; enumeration operation may not execute"). You can't add/remove during `foreach`. **Fix:** iterate a copy (`list.ToList()`), use a `for` loop backwards, or `list.RemoveAll(x => x == 2)`.
</details>

---

### Q4 — Big-O check
```csharp
var seen = new List<int>();      // vs HashSet<int>
foreach (var x in million)
    if (!seen.Contains(x)) seen.Add(x);
```
<details><summary>Answer</summary>

This is **O(n²)** — `List.Contains` is O(n), done n times. With `HashSet<int>`, `Contains`/`Add` are O(1) → **O(n)** overall. Use `HashSet` for membership/dedup.
</details>

---

### Q5 — Capacity vs Count
```csharp
var list = new List<int>();
list.Add(1); list.Add(2); list.Add(3);
Console.WriteLine($"{list.Count} / {list.Capacity}");   // typical values?
```
<details><summary>Answer</summary>

Count is **3**; Capacity is **4** (it grows by doubling: 0→4 on first adds). `Count` = elements; `Capacity` = backing array size. Pre-size with `new List<int>(known)` to avoid reallocations.
</details>

---

### Q6 — ConcurrentDictionary GetOrAdd gotcha
```csharp
var cache = new ConcurrentDictionary<int,Guid>();
// called concurrently for the same key:
var v = cache.GetOrAdd(1, _ => { Console.WriteLine("factory"); return Guid.NewGuid(); });
```
<details><summary>Answer</summary>

Under contention, the **value factory may run more than once** (only one result is kept). So "factory" can print multiple times, and you shouldn't rely on the factory for one-time side effects. For single-shot init, store a `Lazy<Guid>` as the value.
</details>

---

### Q7 — Is this thread-safe?
```csharp
static Dictionary<int,int> _d = new();
// two threads:
void Worker() { for (int i=0;i<1000;i++) _d[i] = i; }
```
<details><summary>Answer</summary>

**No** — `Dictionary` isn't thread-safe; concurrent writes can corrupt internal state (even crash). Use `ConcurrentDictionary` or lock around access. ([11-Synchronization-and-MemoryModel.md](11-Synchronization-and-MemoryModel.md))
</details>

---

### Q8 — Enumeration order
```csharp
var d = new Dictionary<string,int> { ["c"]=3, ["a"]=1, ["b"]=2 };
foreach (var kv in d) Console.Write(kv.Key);
```
<details><summary>Answer</summary>

**Undefined / implementation-detail order** (often insertion order, but *don't rely on it*). For guaranteed ordering use `SortedDictionary` (by key) or sort explicitly. Never depend on `Dictionary`/`HashSet` enumeration order.
</details>

---

### Q9 — Case-insensitive dictionary
```csharp
var d = new Dictionary<string,int> { ["Hello"] = 1 };
Console.WriteLine(d.ContainsKey("hello"));
```
<details><summary>Answer</summary>

**`False`** (default ordinal, case-sensitive). **Fix:** pass a comparer — `new Dictionary<string,int>(StringComparer.OrdinalIgnoreCase)` → then `"hello"` matches. Inject an `IEqualityComparer<T>` to customize key matching without changing the type.
</details>

---

### Q10 — Frozen vs Concurrent vs Immutable (senior)
```csharp
// Read-only lookup table built once at startup, read millions of times. Which type?
```
<details><summary>Answer</summary>

**`FrozenDictionary`** (.NET 8) — it spends extra time at build to optimize layout so **reads are faster than `Dictionary`**. `ConcurrentDictionary` is for concurrent *mutation* (overhead you don't need). `ImmutableDictionary` is for shared snapshots with cheap "edits" (slower reads than Frozen). For build-once-read-many, Frozen wins.
</details>
