# PriorityQueue&lt;TElement, TPriority&gt;

## What it is

A **min-heap-based priority queue** added in .NET 6. Each element has a priority; Dequeue removes the element with the **smallest** priority. Backed by an array-based binary heap.

```csharp
var pq = new PriorityQueue<string, int>();
pq.Enqueue("medium", 2);
pq.Enqueue("low", 3);
pq.Enqueue("high", 1);

while (pq.Count > 0) Console.WriteLine(pq.Dequeue());
// high, medium, low
```

Two generic params: the **element** (the value you store) and the **priority** (the key used for ordering).

This was a long-awaited addition — pre-.NET 6, you had to roll your own heap or use a third-party library.

---

## Why it exists

Many algorithms need "give me the smallest-priority pending item next":
- **Dijkstra's algorithm** — process the nearest unvisited node.
- **A\* search** — process the most-promising candidate.
- **Job scheduling** — run the highest-priority task next.
- **Top-N selection** — keep the K largest, drop the smallest.
- **Event simulation** — fire the next event in time order.

Without a priority queue, you'd sort the queue after every insert (O(n log n) per op) or scan for the minimum (O(n) per op). A heap makes both O(log n).

---

## Basics

```csharp
var pq = new PriorityQueue<string, int>();

// Min-priority dequeue (lower priority = dequeued first)
pq.Enqueue("first", 1);
pq.Enqueue("third", 3);
pq.Enqueue("second", 2);

pq.Count;             // 3
pq.Peek();             // "first" — the element with min priority
pq.PeekPriority();     // 1

pq.Dequeue();          // "first"
pq.TryDequeue(out var elem, out var prio);   // try variant
pq.TryPeek(out var e2, out var p2);
```

Default ordering is **smallest first** (min-heap). For largest-first, use a reversed comparer:

```csharp
var maxHeap = new PriorityQueue<string, int>(Comparer<int>.Create((a, b) => b - a));
maxHeap.Enqueue("third", 3);
maxHeap.Enqueue("first", 1);
maxHeap.Dequeue();   // "third" — largest first
```

Or, more clearly, negate priorities when enqueueing if you want max-heap semantics.

---

## Custom priorities

If priority is a complex type, supply an `IComparer<TPriority>`:

```csharp
var pq = new PriorityQueue<Task, DateTime>(Comparer<DateTime>.Default);
pq.Enqueue(taskA, DateTime.UtcNow.AddSeconds(10));
pq.Enqueue(taskB, DateTime.UtcNow.AddSeconds(5));
pq.Dequeue();   // taskB — earliest first
```

For tuple priorities (multi-level ordering):

```csharp
var pq = new PriorityQueue<Job, (int Priority, int Order)>(
    Comparer<(int, int)>.Create((a, b) =>
        a.Item1 != b.Item1 ? a.Item1 - b.Item1 : a.Item2 - b.Item2)
);

pq.Enqueue(job, (priority: 5, order: 100));
```

---

## `EnqueueRange` and `EnqueueDequeue`

For bulk insertion:

```csharp
pq.EnqueueRange(new[] { ("a", 1), ("b", 2), ("c", 3) });   // items with priorities
pq.EnqueueRange(new[] { "x", "y", "z" }, defaultPriority: 0);   // all at same priority
```

For "swap-in" semantics — useful when keeping a bounded "top K":

```csharp
// "Add this item, but only keep it if it would dequeue last"
pq.EnqueueDequeue("new", priority: 5);   // enqueue + immediate dequeue, returns either the new item or the min
```

---

## Common patterns

### Top-K largest items

To keep the **K largest** from a stream:

```csharp
public static IEnumerable<T> TopK<T>(IEnumerable<T> source, int k, IComparer<T>? comparer = null) {
    comparer ??= Comparer<T>.Default;
    var minHeap = new PriorityQueue<T, T>(comparer);   // min-heap by value

    foreach (var item in source) {
        if (minHeap.Count < k) minHeap.Enqueue(item, item);
        else if (comparer.Compare(item, minHeap.Peek()) > 0) {
            // item is larger than the smallest in heap — swap in
            minHeap.DequeueEnqueue(item, item);
        }
    }
    while (minHeap.Count > 0) yield return minHeap.Dequeue();
}

TopK(numbers, 5);   // 5 largest
```

A min-heap of size K — we only keep the K largest items. Each insertion is O(log K), so total time is O(n log K). Beats sorting (O(n log n)) when K << n.

### Dijkstra's shortest path

```csharp
public Dictionary<TNode, int> Dijkstra<TNode>(TNode start, Func<TNode, IEnumerable<(TNode, int)>> neighbors)
    where TNode : notnull
{
    var dist = new Dictionary<TNode, int> { [start] = 0 };
    var pq = new PriorityQueue<TNode, int>();
    pq.Enqueue(start, 0);

    while (pq.TryDequeue(out var u, out var distU)) {
        if (distU > dist[u]) continue;   // stale entry, skip
        foreach (var (v, w) in neighbors(u)) {
            int alt = distU + w;
            if (!dist.TryGetValue(v, out var curr) || alt < curr) {
                dist[v] = alt;
                pq.Enqueue(v, alt);
            }
        }
    }
    return dist;
}
```

Classic shortest-path. `PriorityQueue<T, P>` doesn't support "decrease-key" directly — the canonical workaround is to enqueue a new entry and skip stale ones (the `if (distU > dist[u]) continue;` check).

### Event simulation

```csharp
var events = new PriorityQueue<Action, DateTime>();
events.Enqueue(() => Console.WriteLine("e1"), DateTime.Now.AddSeconds(5));
events.Enqueue(() => Console.WriteLine("e2"), DateTime.Now.AddSeconds(2));

while (events.TryDequeue(out var action, out var time)) {
    var delay = time - DateTime.Now;
    if (delay > TimeSpan.Zero) Thread.Sleep(delay);
    action();
}
```

For real schedulers, use `Task.Delay` + cancellation, not Thread.Sleep.

---

## Internals — the binary heap

A min-heap stored in an array. For position `i`:
- Parent: `(i - 1) / 2`
- Left child: `2*i + 1`
- Right child: `2*i + 2`

The heap invariant: every parent ≤ its children. Therefore the root (`array[0]`) is always the minimum.

```csharp
// Approximate implementation
public class PriorityQueue<TElement, TPriority> {
    private (TElement Element, TPriority Priority)[] _nodes;
    private int _count;
    private readonly IComparer<TPriority> _comparer;

    public void Enqueue(TElement element, TPriority priority) {
        if (_count == _nodes.Length) Grow();
        _nodes[_count] = (element, priority);
        MoveUp(_count);
        _count++;
    }

    public TElement Dequeue() {
        var root = _nodes[0];
        _count--;
        if (_count > 0) {
            _nodes[0] = _nodes[_count];
            MoveDown(0);
        }
        return root.Element;
    }

    private void MoveUp(int i) {
        while (i > 0) {
            int parent = (i - 1) / 2;
            if (_comparer.Compare(_nodes[i].Priority, _nodes[parent].Priority) >= 0) break;
            (_nodes[i], _nodes[parent]) = (_nodes[parent], _nodes[i]);
            i = parent;
        }
    }

    private void MoveDown(int i) {
        while (true) {
            int left = 2 * i + 1;
            int right = 2 * i + 2;
            int smallest = i;
            if (left < _count && _comparer.Compare(_nodes[left].Priority, _nodes[smallest].Priority) < 0)
                smallest = left;
            if (right < _count && _comparer.Compare(_nodes[right].Priority, _nodes[smallest].Priority) < 0)
                smallest = right;
            if (smallest == i) break;
            (_nodes[i], _nodes[smallest]) = (_nodes[smallest], _nodes[i]);
            i = smallest;
        }
    }
}
```

Both Enqueue and Dequeue are O(log n) — they do at most log₂(n) swaps walking up or down the heap.

### Memory layout

Stored in a `T[]` (where T is `(TElement, TPriority)` — a tuple value type). Each slot holds both the element and priority directly. No node objects, no pointers — just a flat array.

This makes PriorityQueue **dramatically more memory-efficient** than tree-based heaps (which `SortedSet<T>` or hand-coded BSTs would use).

---

## What it doesn't have

`PriorityQueue<TElement, TPriority>` is intentionally minimal. It lacks:
- **Update key / Decrease key** — no direct way to "change the priority of an enqueued item." Workaround: enqueue a new entry, skip stale ones on dequeue.
- **Remove by element** — no `Remove(elem)`. Workaround: mark the element as deleted (lazy deletion), skip on dequeue.
- **Iterate in priority order** — `foreach` gives you elements in heap-array order, NOT sorted. To iterate sorted, copy to an array + sort, or repeatedly Dequeue (destructive).

These omissions keep the API small and the implementation tight. For more features, third-party heap libraries exist.

---

## Common bugs

- **Assuming foreach is sorted** — it isn't. Heap-array order is NOT priority order.
- **Trying to update an enqueued priority** — not directly supported. Use the "enqueue new + skip stale" trick.
- **Calling Dequeue on empty** — throws. Use `TryDequeue`.
- **Thread safety** — not thread-safe.

---

## Performance summary

| Operation | Time |
|---|---|
| Enqueue | O(log n) |
| Dequeue | O(log n) |
| Peek / PeekPriority | O(1) |
| Count | O(1) |
| EnqueueRange (k items) | O(k + log n) via heapify |
| Contains / Remove | not supported (use a separate set) |

For top-K from a stream of N: O(N log K) — much faster than sort-and-take when K is small.

---

## When to use PriorityQueue

✓ Dijkstra, A\*, MST, other graph algorithms.
✓ Top-K problems.
✓ Event simulation / scheduling.
✓ Job queues with priority.

✗ Need to update priorities frequently — consider Fibonacci heaps or `SortedSet` + manual management.
✗ Need ordered iteration of all elements — use `SortedDictionary<TKey, TValue>` or `SortedSet<T>`.
✗ Multi-threaded — wrap externally or use a different design.

→ Next: [08-SortedCollections.md](08-SortedCollections.md)
