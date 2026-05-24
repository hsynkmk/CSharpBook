# Chapter 07 — Collections

> Every data structure in the BCL: arrays, lists, dictionaries, sets, queues, stacks, priority queues, sorted variants, immutable variants, the frozen variants (.NET 8). Plus the equality contract that makes hash-based collections work.

**Prerequisites**: [Chapter 04 (Generics)](../04-Generics/README.md).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Array.md](01-Array.md) | `System.Array`, single/multi-dimensional, jagged, `Array.Sort`, `Array.BinarySearch`, ranges/indices. |
| [02-List.md](02-List.md) | `List<T>` internals (backing array, capacity, growth), `AddRange`, `Insert`, `RemoveAt`, common idioms. |
| [03-LinkedList.md](03-LinkedList.md) | `LinkedList<T>`, when (rarely) it beats `List<T>`. |
| [04-Dictionary.md](04-Dictionary.md) | `Dictionary<K,V>` deep dive: hashing, buckets, chaining, resize, the collision-attack mitigation. |
| [05-HashSet.md](05-HashSet.md) | `HashSet<T>` — semantics of a math set, `UnionWith`, `IntersectWith`, custom `IEqualityComparer`. |
| [06-StackQueue.md](06-StackQueue.md) | `Stack<T>`, `Queue<T>`, common algorithm uses, when to reach for them. |
| [07-PriorityQueue.md](07-PriorityQueue.md) | `PriorityQueue<TElement,TPriority>` (.NET 6+), heap basics, top-K patterns. |
| [08-SortedCollections.md](08-SortedCollections.md) | `SortedDictionary<K,V>`, `SortedSet<T>` (both red-black trees), `SortedList<K,V>` (sorted array), perf characteristics. |
| [09-ImmutableCollections.md](09-ImmutableCollections.md) | `System.Collections.Immutable`: `ImmutableList`, `ImmutableArray`, `ImmutableDictionary`, builders, structural sharing. |
| [10-FrozenCollections.md](10-FrozenCollections.md) | `FrozenDictionary`, `FrozenSet` (.NET 8) — build-once, read-many, fastest lookup in BCL. |
| [11-EqualityContract.md](11-EqualityContract.md) | The `Equals` / `GetHashCode` contract, why mutating a key destroys a dictionary entry, `IEqualityComparer<T>`. |
| [12-IEnumerableHierarchy.md](12-IEnumerableHierarchy.md) | `IEnumerable<T>` → `ICollection<T>` → `IList<T>` → `IReadOnlyList<T>`, choosing the right interface for parameters and returns. |
| [Questions.md](Questions.md) | ~25 questions. |
| [Coding.md](Coding.md) | ~15 problems: implement LRU cache, predict dictionary behavior with mutable keys, choose the right collection. |

→ Begin: [01-Array.md](01-Array.md)
