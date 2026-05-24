# LinkedList&lt;T&gt;

## What it is

`LinkedList<T>` is a **doubly-linked list** — each item lives in a node holding `Value`, `Next`, and `Previous` references. Constant-time insertion and removal at known nodes; no random access.

```csharp
var ll = new LinkedList<int>();
ll.AddLast(1);
ll.AddLast(2);
ll.AddLast(3);
ll.AddFirst(0);                          // 0, 1, 2, 3

LinkedListNode<int>? node = ll.Find(2);
if (node is not null) {
    ll.AddBefore(node, 99);              // 0, 1, 99, 2, 3
    ll.Remove(node);                     // 0, 1, 99, 3
}

foreach (var x in ll) Console.WriteLine(x);
```

Used sparingly. `List<T>` is faster in 95% of cases due to cache locality. `LinkedList<T>` shines when you have many mid-list insertions/removals AND you already hold a node reference.

---

## Why it exists

Some algorithms genuinely need:
- O(1) insertion/removal at a known position (LRU caches, free lists).
- Stable iterator (a node reference stays valid through unrelated modifications).
- No backing-array resize overhead.

The classic case: an **LRU cache** combining a `Dictionary<TKey, LinkedListNode<TValue>>` with the linked list — O(1) lookup AND O(1) eviction.

For "I need a list-ish thing" use cases, `List<T>` wins. Don't reach for `LinkedList<T>` casually.

---

## API surface

```csharp
var ll = new LinkedList<int>();
ll.AddFirst(1);                  // returns LinkedListNode<int>
ll.AddLast(2);
ll.AddBefore(node, 0);
ll.AddAfter(node, 5);

ll.RemoveFirst();
ll.RemoveLast();
ll.Remove(node);                 // O(1) given the node
ll.Remove(value);                // O(n) — finds first matching value
ll.Clear();

LinkedListNode<int>? first = ll.First;
LinkedListNode<int>? last = ll.Last;
int count = ll.Count;

bool has = ll.Contains(42);       // O(n)
LinkedListNode<int>? found = ll.Find(42);            // first match
LinkedListNode<int>? lastFound = ll.FindLast(42);    // last match

// Each node:
node.Value
node.Next
node.Previous
node.List   // the owning list (or null if detached)
```

---

## The node abstraction

```csharp
LinkedListNode<int>? n = ll.First;
while (n is not null) {
    Console.WriteLine(n.Value);
    n = n.Next;
}
```

You can traverse, splice, and store node references. Each node is a heap object (a separate allocation) with two reference fields plus the value.

---

## Inserting and removing

Given a node, both ops are O(1):

```csharp
LinkedListNode<int> node = ll.Find(42)!;
ll.AddBefore(node, 99);    // O(1)
ll.AddAfter(node, 100);    // O(1)
ll.Remove(node);           // O(1)
```

But **finding the node is O(n)**:

```csharp
LinkedListNode<int>? node = ll.Find(42);   // O(n) walk
```

So `LinkedList<T>` only wins when you **already have** the node reference — typically because you stored it in a separate dictionary or iterated and remembered it.

---

## The LRU cache pattern — when LinkedList shines

```csharp
public class LruCache<TKey, TValue> where TKey : notnull {
    private readonly int _capacity;
    private readonly Dictionary<TKey, LinkedListNode<(TKey Key, TValue Value)>> _map;
    private readonly LinkedList<(TKey Key, TValue Value)> _order;

    public LruCache(int capacity) {
        _capacity = capacity;
        _map = new(capacity);
        _order = new();
    }

    public bool TryGet(TKey key, out TValue value) {
        if (_map.TryGetValue(key, out var node)) {
            // Move to front (most recently used)
            _order.Remove(node);
            _order.AddFirst(node);
            value = node.Value.Value;
            return true;
        }
        value = default!;
        return false;
    }

    public void Set(TKey key, TValue value) {
        if (_map.TryGetValue(key, out var node)) {
            // Update + move to front
            _order.Remove(node);
            node.Value = (key, value);
            _order.AddFirst(node);
            return;
        }

        if (_map.Count == _capacity) {
            // Evict least-recently-used (tail)
            var oldest = _order.Last!;
            _map.Remove(oldest.Value.Key);
            _order.RemoveLast();
        }

        var newNode = _order.AddFirst((key, value));
        _map[key] = newNode;
    }
}
```

**All operations O(1)**:
- Lookup: Dictionary O(1).
- Move-to-front: Remove(node) + AddFirst O(1).
- Evict: RemoveLast O(1).

This is the canonical use case. Without `LinkedList<T>` (or a manual doubly-linked structure), you'd lose either the O(1) reordering or the O(1) lookup.

---

## Iteration

```csharp
foreach (var x in ll) Console.WriteLine(x);

// Or by node:
for (var n = ll.First; n is not null; n = n.Next) {
    Console.WriteLine(n.Value);
}

// Backwards:
for (var n = ll.Last; n is not null; n = n.Previous) {
    Console.WriteLine(n.Value);
}
```

Same "collection modified during iteration" detection as `List<T>` — the enumerator throws if you mutate the list while iterating.

---

## Internals — memory layout

Each `LinkedListNode<T>` is a class:

```csharp
public sealed class LinkedListNode<T> {
    internal LinkedList<T>? list;
    internal LinkedListNode<T>? next;
    internal LinkedListNode<T>? prev;
    internal T item;
    public T Value { get; set; }
    public LinkedListNode<T>? Next { get; }
    public LinkedListNode<T>? Previous { get; }
    public LinkedList<T>? List { get; }
}
```

Per-node memory on 64-bit:
- Header: 16 bytes (sync block + MT pointer).
- list ref: 8 bytes.
- next ref: 8 bytes.
- prev ref: 8 bytes.
- item: sizeof(T).

For `LinkedList<int>`, each node is ~48 bytes (40 + int + padding). The actual data is 4 bytes; **44 bytes is overhead**. For a list of 1M ints, that's ~48 MB vs ~4 MB for a `List<int>` (where ints live inline).

This is why `List<T>` is preferred almost always: better memory density, better cache behavior.

### Traversal cost

`List<T>` iteration accesses contiguous memory — every CPU cache line load gives you ~16 ints (or 8 longs, etc.). Branch predictor and prefetcher love it.

`LinkedList<T>` iteration follows pointers — each step might be a cache miss. For 1M items, this can be 100× slower than the equivalent List iteration.

So `LinkedList<T>` only beats `List<T>` for **insertion/removal-heavy** workloads where the order matters. For read-mostly or append-mostly, `List<T>` wins.

---

## When to use LinkedList

✓ LRU cache, free list, or similar where you need O(1) insertion/removal **at known nodes**.
✓ Sliding-window algorithms where items leave from the front and arrive at the back.
✓ Stable iterator across unrelated mutations.

✗ Indexed access — slow O(n).
✗ Iteration-heavy workloads — cache-unfriendly.
✗ Small lists — overhead dominates.
✗ "I might need to insert in the middle someday" — usually a YAGNI. Start with List<T>.

---

## Common bugs

- **Storing a node reference across collection mutations from other code** — if anyone calls Clear or removes your node, it becomes orphaned (`node.List == null`).
- **Comparing nodes with `==`** — compares references. For value comparison, compare `node.Value`.
- **Adding a node from one list to another** — throws if the node already belongs to a list. Detach first.
- **Treating it like a List<T>** — its API isn't IList<T>. No indexer, no `Add(item)` with appending semantics (use `AddLast`).

---

## Performance summary

| Operation | LinkedList<T> | List<T> |
|---|---|---|
| AddLast | O(1) | O(1) amortized |
| AddFirst | O(1) | O(n) |
| AddBefore/After known node | O(1) | n/a |
| Remove known node | O(1) | n/a |
| Remove by value | O(n) | O(n) |
| Index access | O(n) | O(1) |
| Iteration | O(n), cache-unfriendly | O(n), cache-friendly |

**Heuristic**: if you're not sure, use `List<T>`. Reach for `LinkedList<T>` only when you have measured a specific workload where it wins.

→ Next: [04-Dictionary.md](04-Dictionary.md)
