# 05 — Collections (internals, Big-O, choosing the right one)

## ⚡ 30-second answer

Pick by access pattern: **`List<T>`** for ordered, index access and append (array-backed); **`Dictionary<K,V>`** for O(1) key lookup (hash table); **`HashSet<T>`** for uniqueness/membership; **`Queue`/`Stack`** for FIFO/LIFO; **`SortedDictionary`/`SortedSet`** for ordered keys (tree, O(log n)). For threads, use **`Concurrent*`** collections; for shared immutable state, **`Immutable*`**; for read-heavy lookup tables, **`Frozen*`** (.NET 8). The classic question is **Dictionary internals**: it hashes the key, buckets by `hash % capacity`, and resolves collisions via chaining — relying on a correct `GetHashCode`/`Equals`.

---

## Core mechanics

**`List<T>`** — backed by a `T[]`. Append is amortized O(1) but **doubles capacity** when full (copying the array); `Count` ≠ `Capacity`. Index access O(1); insert/remove in the middle O(n).

**`Dictionary<K,V>`** — hash table:
1. Compute `key.GetHashCode()`.
2. Map to a bucket (`hash % bucketCount`).
3. On collision (same bucket), walk a short chain comparing with `Equals`.
- Average **O(1)** get/add/remove; **O(n)** worst case if all keys collide (bad hash).
- Requires a stable, well-distributed `GetHashCode` and consistent `Equals` ([06-Equality.md](06-Equality.md)).
- **Unordered**; enumeration order is an implementation detail (don't rely on it).

**`HashSet<T>`** — same hashing, stores keys only; O(1) `Add`/`Contains`. Set ops: `UnionWith`, `IntersectWith`, `ExceptWith`.

**Sorted (`SortedDictionary`, `SortedSet`)** — balanced binary search tree (red-black). O(log n) ops, keeps keys **sorted**. `SortedList` is array-backed (less memory, slower inserts).

**`Span<T>`/`ReadOnlySpan<T>`** — a view over contiguous memory (array/stack/string) with no allocation/copy ([13-MemoryAndGC.md](13-MemoryAndGC.md)).

---

## Comparison tables

| Collection | Lookup | Insert | Ordered? | Backed by |
|---|---|---|---|---|
| `List<T>` | O(1) by index, O(n) by value | O(1)* append, O(n) middle | insertion | array |
| `Dictionary<K,V>` | **O(1)** by key | O(1)* | no | hash table |
| `HashSet<T>` | **O(1)** contains | O(1)* | no | hash table |
| `SortedDictionary<K,V>` | O(log n) | O(log n) | **by key** | red-black tree |
| `Queue<T>` / `Stack<T>` | — | O(1) enqueue/push | FIFO / LIFO | array (ring) |
| `LinkedList<T>` | O(n) | O(1) at a known node | insertion | doubly linked |
| `PriorityQueue<E,P>` | O(1) peek | O(log n) | by priority | binary heap |

\* amortized

| Family | When |
|---|---|
| `Concurrent*` (`ConcurrentDictionary`, `ConcurrentQueue`, `BlockingCollection`) | multi-threaded access without external locks ([12](12-Concurrent-Parallel-AsyncBugs.md)) |
| `Immutable*` (`ImmutableList`, `ImmutableDictionary`) | shared state, snapshot semantics, "modify" returns a new copy |
| `Frozen*` (`FrozenDictionary`, `FrozenSet`, .NET 8) | build once, read many — optimized for ultra-fast lookups |

---

## 🪤 Traps & gotchas

- **Mutable key in a Dictionary/HashSet**: if a key's hash changes after insertion (mutating a field used in `GetHashCode`), you can never find it again. Use **immutable keys**.
- **Wrong/missing `GetHashCode`**: a class without overridden `GetHashCode` uses reference identity — two "equal" objects won't match as keys. Override both `Equals` and `GetHashCode` together ([06](06-Equality.md)).
- **`Dictionary` is not thread-safe**: concurrent writes corrupt it (even concurrent read+write). Use `ConcurrentDictionary` or lock.
- **Relying on enumeration order** of `Dictionary`/`HashSet` — undefined. Use a sorted/ordered collection if order matters.
- **`List.Capacity` growth** — repeated `Add` reallocates; pre-size with `new List<T>(capacity)` if you know the count.
- **`foreach` while modifying** a collection throws `InvalidOperationException` (modified during enumeration).
- **`ConcurrentDictionary.GetOrAdd` factory** can run **more than once** under contention (only one result is kept) — don't rely on the factory running exactly once for side effects.

---

## ❓ Likely questions

**Q: How does `Dictionary<K,V>` work internally?**
A: Hash table — hash the key, map to a bucket, resolve collisions by chaining, compare candidates with `Equals`. Average O(1); needs a good `GetHashCode`.

**Q: `List<T>` vs `LinkedList<T>`?**
A: `List` is array-backed: O(1) index, cache-friendly, O(n) middle insert. `LinkedList` is O(1) insert/remove at a node but O(n) search and poor cache locality. Prefer `List` almost always.

**Q: When `HashSet` vs `List` for "contains"?**
A: `HashSet.Contains` is O(1); `List.Contains` is O(n). For membership/dedup, use `HashSet`.

**Q: What happens when a `List` exceeds capacity?**
A: It allocates a new larger array (typically doubling) and copies elements — amortized O(1) append, but a momentary O(n) copy.

**Q: How do you make a dictionary thread-safe?**
A: Use `ConcurrentDictionary` (lock-free reads, fine-grained locked writes) or guard a `Dictionary` with a `lock`. Don't share a plain `Dictionary` across writing threads.

**Q: `Concurrent` vs `Immutable` vs `Frozen`?**
A: Concurrent = safe mutation by many threads. Immutable = never changes; "edits" return new instances (great for shared snapshots). Frozen = build once then read-only, tuned for fastest lookups.

**Q: Big-O of dictionary worst case?**
A: O(n) if every key hashes to the same bucket (degenerate hash) — why a good `GetHashCode` matters.

---

## 🎓 Senior Extra

- **`ConcurrentDictionary` internals**: lock striping — multiple lock objects guard buckets so different keys can be written concurrently; reads are lock-free. `GetOrAdd`/`AddOrUpdate` are atomic *per key* but the value factory isn't guaranteed single-invocation under races.
- **Hash quality & DoS**: predictable string hashing once enabled hash-flooding attacks (many colliding keys → O(n) → CPU DoS). .NET randomizes string hash seeds per process to mitigate; for value objects, combine fields with `HashCode.Combine`.
- **`FrozenDictionary`** spends extra time at build to pick an optimal internal layout (perfect-ish hashing) so reads are faster than `Dictionary` — worth it for static lookup tables read millions of times.
- **`Span<T>`/`stackalloc`** let you process slices of arrays/strings with zero allocation; `CollectionsMarshal.AsSpan(list)` gets a span over a `List`'s backing array (advanced, careful with resizing).
- **`CollectionsMarshal.GetValueRefOrAddDefault`** lets you update a struct value in a dictionary in place without a double lookup (hot-path optimization).
- **Capacity & memory**: `EnsureCapacity`/constructor sizing avoids reallocations; `TrimExcess` reclaims unused capacity. For large short-lived buffers, `ArrayPool<T>` beats allocating arrays ([13](13-MemoryAndGC.md)).
- **Equality comparer injection**: pass an `IEqualityComparer<T>` to a dictionary/set to customize key matching (e.g., case-insensitive `StringComparer.OrdinalIgnoreCase`) without changing the type.

→ Deeper: [`../CSharpBook/07-Collections/`](../CSharpBook/07-Collections/README.md)
