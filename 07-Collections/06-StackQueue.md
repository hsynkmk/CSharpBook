# Stack&lt;T&gt; and Queue&lt;T&gt;

## What they are

Two classic ADTs (abstract data types) with simple, restricted access:

- **`Stack<T>`** — LIFO (Last-In, First-Out). Push/Pop at the top. Useful for depth-first traversal, undo history, evaluator stacks.
- **`Queue<T>`** — FIFO (First-In, First-Out). Enqueue at the back, Dequeue from the front. Useful for breadth-first traversal, scheduling, work pipelines.

Both back onto a `T[]` (circular buffer for Queue), so they're efficient and cache-friendly.

```csharp
// Stack — LIFO
var s = new Stack<int>();
s.Push(1); s.Push(2); s.Push(3);
Console.WriteLine(s.Pop());   // 3
Console.WriteLine(s.Peek());   // 2
Console.WriteLine(s.Count);    // 2

// Queue — FIFO
var q = new Queue<int>();
q.Enqueue(1); q.Enqueue(2); q.Enqueue(3);
Console.WriteLine(q.Dequeue());   // 1
Console.WriteLine(q.Peek());       // 2
Console.WriteLine(q.Count);        // 2
```

---

## Why they exist

Algorithms often need these specific access patterns:
- **DFS** (depth-first search) — stack.
- **BFS** (breadth-first search) — queue.
- **Recursive→iterative conversion** — stack.
- **Producer/consumer** — queue.
- **Undo/redo** — two stacks.
- **Backtracking** — stack.

Both could be done with `List<T>` (Add + RemoveAt 0 for FIFO, Add + RemoveAt Count-1 for LIFO) — but Queue's circular-buffer Dequeue is O(1) where `RemoveAt(0)` on a List is O(n).

Use the dedicated collections for clarity and correct asymptotic behavior.

---

## `Stack<T>` API

```csharp
var s = new Stack<int>();
var sized = new Stack<int>(capacity: 100);
var from = new Stack<int>(new[] { 1, 2, 3 });   // top is 3 (pushed last)

s.Push(42);
int top = s.Peek();         // does not remove
int popped = s.Pop();        // removes and returns
bool ok = s.TryPeek(out int val);   // .NET Core 2.0+
bool ok2 = s.TryPop(out int v);

bool has = s.Contains(42);   // O(n)
s.Clear();
int count = s.Count;
int[] arr = s.ToArray();      // top is index 0

s.TrimExcess();
```

Iteration is top-to-bottom (LIFO order):

```csharp
foreach (var x in s) Console.WriteLine(x);   // 3, 2, 1 if pushed 1, 2, 3
```

This is the order callers expect — most recently pushed first.

---

## `Queue<T>` API

```csharp
var q = new Queue<int>();
var sized = new Queue<int>(capacity: 100);
var from = new Queue<int>(new[] { 1, 2, 3 });   // front is 1 (enqueued first)

q.Enqueue(42);
int front = q.Peek();
int dequeued = q.Dequeue();
bool ok = q.TryPeek(out int val);
bool ok2 = q.TryDequeue(out int v);

bool has = q.Contains(42);
q.Clear();
int count = q.Count;
int[] arr = q.ToArray();

q.TrimExcess();
```

Iteration is front-to-back (FIFO order):

```csharp
foreach (var x in q) Console.WriteLine(x);   // 1, 2, 3 if enqueued 1, 2, 3
```

---

## Common patterns

### Iterative DFS

```csharp
public IEnumerable<TreeNode> DepthFirst(TreeNode root) {
    var stack = new Stack<TreeNode>();
    stack.Push(root);
    while (stack.TryPop(out var node)) {
        yield return node;
        foreach (var child in node.Children.Reverse())
            stack.Push(child);
    }
}
```

`Reverse` because Stack reverses order — push children in reverse so left child gets visited first.

### Iterative BFS

```csharp
public IEnumerable<TreeNode> BreadthFirst(TreeNode root) {
    var queue = new Queue<TreeNode>();
    queue.Enqueue(root);
    while (queue.TryDequeue(out var node)) {
        yield return node;
        foreach (var child in node.Children)
            queue.Enqueue(child);
    }
}
```

### Undo stack

```csharp
public class UndoManager {
    private readonly Stack<Action> _undo = new();
    private readonly Stack<Action> _redo = new();

    public void Do(Action action, Action undo) {
        action();
        _undo.Push(undo);
        _redo.Clear();
    }
    public void Undo() {
        if (_undo.TryPop(out var u)) { u(); _redo.Push(/* the redo */); }
    }
}
```

(Real undo systems track both directions; this is the skeleton.)

### Producer/consumer (single-threaded)

```csharp
var work = new Queue<WorkItem>();
work.Enqueue(item1);
work.Enqueue(item2);
while (work.TryDequeue(out var item)) {
    Process(item);
    // Process might enqueue more items
}
```

For thread-safe variants, see `ConcurrentQueue<T>` and `Channel<T>` ([CSharpBook chapter 08](../08-Concurrency/13-Channels.md)).

### Recursion → iteration

A recursive algorithm with state at each level can be rewritten with an explicit stack — useful when recursion depth might exceed stack size:

```csharp
// Recursive
int SumDeep(TreeNode n) =>
    n.Value + n.Children.Sum(SumDeep);

// Iterative
int SumDeepIterative(TreeNode root) {
    int sum = 0;
    var stack = new Stack<TreeNode>();
    stack.Push(root);
    while (stack.TryPop(out var n)) {
        sum += n.Value;
        foreach (var c in n.Children) stack.Push(c);
    }
    return sum;
}
```

The iterative version uses the heap (where the stack object lives) instead of the call stack. Survives deep trees.

---

## Internals

### Stack<T>

Backed by a `T[]`. The "top" is at index `Count - 1`. Push appends; Pop removes from the end. Resizing on Push works just like `List<T>` (double when full).

```csharp
// Approximate Stack<T> implementation
public class Stack<T> {
    private T[] _array = Array.Empty<T>();
    private int _size = 0;
    public int Count => _size;
    public void Push(T item) {
        if (_size == _array.Length) Grow();
        _array[_size++] = item;
    }
    public T Pop() {
        if (_size == 0) throw new InvalidOperationException("Stack empty");
        T item = _array[--_size];
        _array[_size] = default!;   // clear reference for GC
        return item;
    }
    public T Peek() { /* ... */ }
}
```

Per-Push: O(1) amortized. Per-Pop: O(1).

### Queue<T>

Backed by a `T[]` used as a **circular buffer**. Two indices:
- `_head` — where Dequeue reads from.
- `_tail` — where Enqueue writes to.

When `_tail` reaches `_array.Length`, it wraps to 0 (modulo). If `_head == _tail` and Count > 0, the buffer is full and grows.

```csharp
// Approximate Queue<T> implementation
public class Queue<T> {
    private T[] _array;
    private int _head, _tail, _size;
    public void Enqueue(T item) {
        if (_size == _array.Length) Grow();
        _array[_tail++] = item;
        if (_tail == _array.Length) _tail = 0;
        _size++;
    }
    public T Dequeue() {
        if (_size == 0) throw new InvalidOperationException();
        T item = _array[_head];
        _array[_head++] = default!;
        if (_head == _array.Length) _head = 0;
        _size--;
        return item;
    }
}
```

O(1) per operation. **Crucial** — a naive queue using `List<T>.RemoveAt(0)` is O(n) per Dequeue.

When growing, the array is copied with `_head` becoming 0 of the new array — un-wrapping the circular layout.

### Memory

Both have the same per-item cost as `List<T>` — just `sizeof(T)` per slot, plus a small fixed overhead for the structure.

---

## Iteration semantics

```csharp
foreach (var x in stack) { ... }   // top to bottom (LIFO order — same as Pop order)
foreach (var x in queue) { ... }    // front to back (FIFO order — same as Dequeue order)
```

Don't mutate the collection during iteration — same "collection modified" check as other BCL collections.

For inspection without mutation: `.ToArray()` to snapshot.

---

## Common bugs

- **Pop / Dequeue on empty** — `InvalidOperationException`. Use `TryPop` / `TryDequeue`.
- **`List<T>.RemoveAt(0)` for FIFO** — O(n) per Dequeue. Use `Queue<T>`.
- **Using a List for LIFO** — works but less clear. `Stack<T>` signals intent.
- **Iterating then trying to Dequeue** — the foreach doesn't consume. Use `while (TryDequeue)` if you want both.
- **Thread safety** — neither is thread-safe. Use `ConcurrentStack<T>` / `ConcurrentQueue<T>` / `Channel<T>`.

---

## Performance summary

| Operation | Stack<T> | Queue<T> | LinkedList<T> | List<T> |
|---|---|---|---|---|
| Push / AddLast / Add | O(1) amortized | O(1) amortized | O(1) | O(1) |
| Pop / RemoveLast | O(1) | n/a | O(1) | O(1) |
| Enqueue | n/a | O(1) amortized | O(1) | O(1) |
| Dequeue / RemoveFirst | n/a | O(1) | O(1) | O(n) |
| Contains | O(n) | O(n) | O(n) | O(n) |
| Indexed access | n/a | n/a | n/a | O(1) |

For pure LIFO/FIFO, the dedicated types are the right call.

---

## When to use

✓ **Stack**: DFS, recursion-to-iteration, undo, expression evaluation.
✓ **Queue**: BFS, scheduling, single-threaded producer/consumer.

✗ Random access — use `List<T>`.
✗ Concurrent — use `ConcurrentStack/Queue` or `Channel<T>`.
✗ Priority-based dequeue — use `PriorityQueue<TElement, TPriority>`.

→ Next: [07-PriorityQueue.md](07-PriorityQueue.md)
