# Chapter 07 — Questions

> Drilling for everything in Chapter 07. Collections show up in every interview and every codebase.

---

## Picking the right collection

**Q1.** Which collection for: "I need to find an item by name, fast"?
<details><summary>Answer</summary>`Dictionary<string, T>` (or `FrozenDictionary<string, T>` if read-only and built once). O(1) average lookup.</details>

**Q2.** Which for: "I need unique items and 'is X in here?' lookups"?
<details><summary>Answer</summary>`HashSet<T>`. O(1) Contains / Add / Remove.</details>

**Q3.** Which for: "Process tasks in order of arrival, FIFO"?
<details><summary>Answer</summary>`Queue<T>`. O(1) Enqueue / Dequeue.</details>

**Q4.** Which for: "Process highest-priority task next"?
<details><summary>Answer</summary>`PriorityQueue<TElement, TPriority>`. O(log n) Enqueue / Dequeue. Min-heap by default; reverse comparer for max-heap.</details>

**Q5.** Which for: "Read-only lookup table built at startup, accessed millions of times"?
<details><summary>Answer</summary>`FrozenDictionary<TKey, TValue>` (.NET 8+). Pays construction cost upfront for fastest possible reads.</details>

**Q6.** Which for: "Many threads might read; occasionally one updates"?
<details><summary>Answer</summary>`ImmutableDictionary<K, V>` with `ImmutableInterlocked.Update`, OR `ConcurrentDictionary<K, V>`. Immutable wins for very read-heavy; Concurrent for balanced read/write.</details>

---

## Array

**Q7.** Why is this a runtime bug?
```csharp
string[] strs = { "a", "b" };
object[] objs = strs;
objs[0] = 42;
```
<details><summary>Answer</summary>Array covariance — `string[]` flows as `object[]` at compile time. But the runtime type is still `string[]`. Storing an int triggers `ArrayTypeMismatchException`. Generic `List<T>` is invariant — same bug class doesn't compile.</details>

**Q8.** What's the difference between `int[,]` and `int[][]`?
<details><summary>Answer</summary>`int[,]` is rectangular — one contiguous block, fixed shape, indexed `[i, j]`. `int[][]` is jagged — array of array references, each row can differ in length, indexed `[i][j]`. Jagged is usually faster in practice (JIT optimizes `int[]` aggressively); rectangular wins for storage layout.</details>

---

## List

**Q9.** What happens internally when you `Add` to a List that's at capacity?
<details><summary>Answer</summary>The backing array is doubled (or set to 4 if it was 0). All items are copied. Then the new item is added. Amortized O(1) per Add — the resize is O(n) but happens log₂(n) times over many Adds.</details>

**Q10.** Why might `list.RemoveAll(predicate)` be much faster than calling `list.Remove` in a loop?
<details><summary>Answer</summary>`Remove(item)` is O(n) — finds + shifts. Calling it n times = O(n²). `RemoveAll(pred)` is a single O(n) pass — collects survivors, copies them once.</details>

**Q11.** What's the fastest way to get a `Span<int>` over a `List<int>`?
<details><summary>Answer</summary>`CollectionsMarshal.AsSpan(list)` (.NET 5+). No copy — direct view over the backing array. Invalid after any mutation to the list.</details>

---

## Dictionary

**Q12.** What's the equality contract for Dictionary keys?
<details><summary>Answer</summary>If `a.Equals(b)` is true, then `a.GetHashCode() == b.GetHashCode()`. Both must be stable while the key is in the dictionary (no mutation of fields used in equality/hash).</details>

**Q13.** Why does this leak the entry?
```csharp
var d = new Dictionary<MyKey, string>();
var k = new MyKey { X = 5 };
d[k] = "five";
k.X = 99;
d.Remove(k);
```
<details><summary>Answer</summary>`k.X = 99` changes the hash. `d.Remove(k)` hashes 99 → looks in bucket for hash(99), but the entry lives in bucket for hash(5). Remove returns false; entry remains, unreachable except by enumeration. **Don't mutate dictionary keys after insertion.**</details>

**Q14.** What's `CollectionsMarshal.GetValueRefOrAddDefault` for?
<details><summary>Answer</summary>Returns a **ref** to the slot in the dictionary, doing one hash lookup. Lets you read or write directly, avoiding the double-lookup of `TryGetValue` + `dict[key] = ...`. Used in hot counters and parsers. The ref is invalidated by subsequent mutations.</details>

---

## HashSet, sets

**Q15.** What's the difference between `set.UnionWith(other)` and `set.Union(other)`?
<details><summary>Answer</summary>`UnionWith` mutates `set` in place (`ISet<T>` method). `Union` is LINQ — returns a new sequence, doesn't modify `set`. Use UnionWith for "modify this set"; Union for "give me a new sequence."</details>

**Q16.** Can a HashSet contain `null`?
<details><summary>Answer</summary>For reference type T, yes — `HashSet<string>` can contain null. For value type T, no — `null` doesn't exist unless `T?` (Nullable). Note that `Dictionary<TKey, TValue>` does NOT allow null keys (TKey has `where TKey : notnull`).</details>

---

## Priority queue

**Q17.** What's the default ordering for `PriorityQueue<T, int>`?
<details><summary>Answer</summary>**Min-priority dequeue** — smallest priority comes out first. For max-priority, use `Comparer<int>.Create((a, b) => b - a)` or negate priorities.</details>

**Q18.** What's `EnqueueDequeue` for?
<details><summary>Answer</summary>Atomically enqueues an item and immediately dequeues — returns either the new item or the previous min, whichever is smaller. Useful for "keep top K" — push a new item; if the heap has K already, pushes out the smallest.</details>

---

## Sorted collections

**Q19.** Difference between `SortedDictionary` and `SortedList`?
<details><summary>Answer</summary>Same API surface for lookup. `SortedDictionary` is a red-black tree — O(log n) Add/Remove/Lookup, higher memory. `SortedList` is parallel sorted arrays — O(n) Add/Remove (shift), O(log n) Lookup (binary search), lower memory. Use `SortedDictionary` for mutation-heavy workloads; `SortedList` for build-once + read-mostly.</details>

**Q20.** What does `SortedSet<T>.GetViewBetween(min, max)` return?
<details><summary>Answer</summary>A **live view** of the sub-range. Mutations to the view modify the underlying set; mutations to the set show up in the view. Useful for range queries that span the underlying structure.</details>

---

## Immutable

**Q21.** Why does this not modify the list?
```csharp
var list = ImmutableList.Create(1, 2, 3);
list.Add(4);
Console.WriteLine(list.Count);
```
<details><summary>Answer</summary>**3.** `list.Add(4)` returns a new list; the original is unchanged. Must capture: `list = list.Add(4);`. This is the immutable pattern.</details>

**Q22.** Why use a Builder for ImmutableList?
<details><summary>Answer</summary>Adding 1M items one-by-one to an `ImmutableList<T>` creates 1M intermediate immutable lists — slow. The Builder lets you mutate freely and finalize with `ToImmutable()`. Single allocation cost. Same total time as `List<T>` build, then freeze.</details>

**Q23.** Difference between `ImmutableArray<T>` and `ImmutableList<T>`?
<details><summary>Answer</summary>`ImmutableArray<T>` is backed by a regular array — reads are array-fast (O(1)), mutations allocate a full new array (O(n)). `ImmutableList<T>` is backed by a balanced tree — reads O(log n), mutations O(log n) with structural sharing. Array for read-mostly; List for mutation-heavy.</details>

---

## Frozen

**Q24.** When does `FrozenDictionary` beat `Dictionary`?
<details><summary>Answer</summary>When you build it once (startup, configuration load) and read it many times. Construction is slower than `Dictionary` (analyzes keys); reads are ~20-40% faster. For build-once-read-millions-of-times tables, the lifetime cost is much lower.</details>

**Q25.** What's the cost of `dict.ToFrozenDictionary()` if called once per request?
<details><summary>Answer</summary>O(n × analysis) — analyzing keys and choosing a layout each time. Worse than just using the dictionary. Frozen is for startup / one-shot use. Replacing dynamically (e.g., reloading config) is OK if rare.</details>

---

## Equality contract

**Q26.** What's wrong with this?
```csharp
public class Bad {
    public int X;
    public override bool Equals(object? o) => o is Bad b && b.X == X;
}
```
<details><summary>Answer</summary>Overrode `Equals` without `GetHashCode`. Default GetHashCode is identity-based — equal objects can have different hashes — violates the contract. Compiler warning CS0659. Hashed collections will malfunction.</details>

**Q27.** What does `HashCode.Combine` add over `X ^ Y ^ Z`?
<details><summary>Answer</summary>Better distribution (uses xxHash), randomized seed per process (mitigates collision attacks), correctly handles 0 / null. XOR is commutative — `Combine(1, 2)` ≠ `Combine(2, 1)`, but `1 ^ 2 == 2 ^ 1`, so XOR loses ordering. `HashCode.Combine` is the right tool.</details>

**Q28.** Why prefer records for value-equality types?
<details><summary>Answer</summary>Records auto-synthesize the correct equality contract: `IEquatable<T>`, `Equals(object?)`, `GetHashCode` via `HashCode.Combine`, `==`, `!=`. Less code, no chance of mismatches. Use `record class` or `readonly record struct`.</details>

---

## Interface hierarchy

**Q29.** What parameter type would you use for a method that just iterates?
<details><summary>Answer</summary>`IEnumerable<T>` — most flexible; accepts arrays, lists, sets, LINQ queries, custom iterators. Use looser types when you don't need more.</details>

**Q30.** What return type for "give the caller a snapshot they can iterate and index"?
<details><summary>Answer</summary>`IReadOnlyList<T>`. Snapshot (materialized), can be indexed, can be iterated, can't be mutated by caller. Materialize with `ToList()` and return as `IReadOnlyList<T>`.</details>

**Q31.** Why is `IEnumerable<out T>` covariant but `IList<T>` invariant?
<details><summary>Answer</summary>`IEnumerable<T>` only produces T (read-only iteration). `IEnumerable<string> → IEnumerable<object>` is safe — every string IS an object. `IList<T>` produces AND consumes T (indexer get + set, Add, Remove). Allowing covariance would let `IList<string>` flow as `IList<object>`; then `Add(42)` would compile but corrupt the underlying list. Mutable collections must be invariant.</details>

---

## Mixed / synthesis

**Q32.** Implement an LRU cache. What collections do you need?
<details><summary>Answer</summary>`Dictionary<TKey, LinkedListNode<TValue>>` + `LinkedList<TValue>`. Dictionary for O(1) lookup; LinkedList for O(1) move-to-front and evict-from-back. All ops O(1). See [03-LinkedList.md](03-LinkedList.md) for code.</details>

**Q33.** What collection for "count word frequencies in a stream"?
<details><summary>Answer</summary>`Dictionary<string, int>` (or `FrozenDictionary` if read-only after). For thread-safety, `ConcurrentDictionary<string, int>` with `AddOrUpdate`. For the absolute fastest single-threaded version, `CollectionsMarshal.GetValueRefOrAddDefault` + `ref count` increment.</details>

**Q34.** Why might `list.ToFrozenSet()` be a better choice than `new HashSet<T>(list)` for `Contains` checks?
<details><summary>Answer</summary>FrozenSet has specialized implementations per type (int, string, etc.) optimized for the specific set being frozen. For very hot `Contains` checks (e.g., a whitelist checked in middleware), it's measurably faster. Trade-off: slower construction.</details>

**Q35.** A coworker writes a public method returning `List<User>`. What's the concern, and what would you suggest?
<details><summary>Answer</summary>Returning `List<T>` exposes the internal storage (or allows callers to mutate the result). If you don't want callers mutating, return `IReadOnlyList<User>`. The list reference is the same — no copy — but the API restricts mutation. If you actually want a snapshot, return `users.ToList().AsReadOnly()` or use `ImmutableList<User>`.</details>

---

→ [Coding.md](Coding.md)
