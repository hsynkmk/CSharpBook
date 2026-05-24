# List&lt;T&gt;

## What it is

`List<T>` is .NET's go-to **dynamic array** — random access by index, automatic resizing, and amortized O(1) Add. Internally it's a regular `T[]` (the "backing array") plus a count, growing as needed.

```csharp
var nums = new List<int> { 1, 2, 3 };
nums.Add(4);
nums.Insert(0, 0);                  // { 0, 1, 2, 3, 4 }
Console.WriteLine(nums[2]);          // 2
Console.WriteLine(nums.Count);        // 5

nums.Remove(2);                      // remove first occurrence
nums.RemoveAt(0);                    // remove at index
```

If you're holding a sequence of items and you're not sure which collection to use, **start with `List<T>`**.

---

## Why it exists

Arrays are fixed-size; manually resizing them via `Array.Resize` is tedious and error-prone. `List<T>` automates:
- Growing the backing array when you Add and it's full.
- Tracking the logical Count separately from the Capacity.
- Common operations (`Insert`, `Remove`, `Contains`, `IndexOf`, etc.).

It's a thin abstraction over an array, optimized for typical Add-mostly access patterns.

---

## API surface

```csharp
var list = new List<int>();
var sized = new List<int>(capacity: 100);   // pre-allocate
var from = new List<int>(new[] { 1, 2, 3 }); // from any IEnumerable

// Mutation
list.Add(42);
list.AddRange(otherList);
list.Insert(0, 99);
list.InsertRange(1, otherList);
bool removed = list.Remove(42);     // first occurrence
list.RemoveAt(2);
list.RemoveAll(x => x < 0);          // by predicate
list.RemoveRange(2, 3);              // range
list.Clear();

// Inspection
int c = list.Count;
int cap = list.Capacity;
list.TrimExcess();                   // shrink Capacity to Count

// Access
int x = list[0];                     // indexer
list[0] = 99;

// Find
int idx = list.IndexOf(42);          // -1 if not found
int last = list.LastIndexOf(42);
bool has = list.Contains(42);
int first = list.Find(x => x > 0);
List<int> all = list.FindAll(x => x > 0);
int firstIdx = list.FindIndex(x => x > 0);
bool any = list.Exists(x => x > 0);
bool all2 = list.TrueForAll(x => x > 0);

// Sort
list.Sort();                                          // default comparer
list.Sort((a, b) => b - a);                            // custom Comparison
list.Sort(Comparer<int>.Create((a, b) => b - a));      // IComparer

// Other
list.Reverse();                                        // in place
list.ToArray();                                        // → T[]
int[] arr = list.ToArray();
list.CopyTo(targetArray);
List<int> slice = list.GetRange(1, 3);                  // copy of items 1..3

// ForEach (legacy — use foreach instead)
list.ForEach(x => Console.WriteLine(x));
```

The API is large but consistent. Most of it has a LINQ equivalent — use whichever reads better.

---

## Indexed access

O(1). Identical performance to `T[]`:

```csharp
list[3] = 99;
int x = list[3];
```

Bounds checked. Out of range throws `ArgumentOutOfRangeException`.

For Span access without copies (.NET 5+):

```csharp
Span<int> span = CollectionsMarshal.AsSpan(list);
// Use the span; valid only while list isn't mutated.
```

---

## Capacity and growth

`List<T>` distinguishes:
- **Count** — the logical number of items.
- **Capacity** — the size of the backing array.

When you Add and Count would exceed Capacity, the backing array is replaced with one **double the size** (or 4 if it was 0). The old contents are copied. This makes Add **amortized O(1)** — most Adds are constant-time; occasional resize is O(n) but happens log₂(n) times.

```csharp
var list = new List<int>();
for (int i = 0; i < 17; i++) list.Add(i);

// Capacity history during growth:
// 0 → 4 → 8 → 16 → 32
```

### Pre-sizing

If you know you'll add ~N items, pre-size:

```csharp
var list = new List<int>(capacity: 10_000);
for (int i = 0; i < 10_000; i++) list.Add(i);
```

Avoids resize overhead — single allocation.

### `EnsureCapacity` (.NET 6+)

```csharp
list.EnsureCapacity(1000);   // grow if needed; no shrink
```

### `TrimExcess`

After bulk removal:

```csharp
list.TrimExcess();   // shrink to Count if Capacity > Count * 1.something
```

Useful when you've made a small list out of a large one and want to release memory.

---

## Insertion and removal

Indexed access is O(1). Inserting or removing at position **i** is **O(n)** because items after `i` shift:

```csharp
list.Insert(0, 99);    // O(n) — shift everything right by one
list.Add(99);          // O(1) amortized — append
list.RemoveAt(5);      // O(n) — shift everything left
list.Remove(42);       // O(n) — find + shift
```

For frequent mid-insertion / mid-removal, consider `LinkedList<T>` (O(1) at a known node). For tail-only operations, `List<T>` and `Queue<T>` excel.

### `RemoveAll` is efficient for bulk removal

```csharp
list.RemoveAll(x => x < 0);    // single pass, in place
```

Don't repeatedly call `Remove` in a loop — that's O(n) per removal. `RemoveAll` is one O(n) pass.

---

## Iteration patterns

```csharp
// foreach — most common
foreach (var x in list) { ... }

// for — when you need index or want to mutate
for (int i = 0; i < list.Count; i++) { ... }

// Span — fast iteration when you don't mutate
foreach (var x in CollectionsMarshal.AsSpan(list)) { ... }
```

`foreach` over `List<T>` uses a **struct enumerator** (`List<T>.Enumerator`) — no heap allocation in optimized code. Same speed as indexed `for`.

**Don't modify a List while foreach-ing it**:

```csharp
foreach (var x in list) {
    if (x < 0) list.Remove(x);   // ❌ InvalidOperationException
}
```

The enumerator detects the version change and throws. Solutions:
- Use `RemoveAll(pred)`.
- Materialize first: `foreach (var x in list.ToList()) { ... }`.
- Iterate by index and shift manually.

---

## Sorting

```csharp
list.Sort();                                       // default Comparer<T>.Default
list.Sort((a, b) => a.CompareTo(b));                // Comparison<T> delegate
list.Sort(Comparer<MyType>.Create(...));            // IComparer<T>
list.Sort(0, 10, comparer);                         // sort range
```

`List<T>.Sort()` uses **introsort** (quicksort that falls back to heapsort on bad inputs) — O(n log n) average, O(n log n) worst case. **Not stable** — equal-key items may swap order.

For a stable sort, use LINQ's `OrderBy` (stable since .NET 6):

```csharp
list = list.OrderBy(x => x.Key).ToList();
```

---

## Searching

```csharp
list.IndexOf(42);          // first occurrence, -1 if absent
list.LastIndexOf(42);       // from end
list.Contains(42);          // bool
list.Find(x => x > 0);      // first match, default(T) if absent
list.FindAll(x => x > 0);   // List<T> of matches
list.BinarySearch(42);      // requires sorted; returns index or ~insertionPoint
```

All linear except `BinarySearch`, which is O(log n) but requires the list to be sorted by the comparer.

LINQ alternatives:

```csharp
list.FirstOrDefault(x => x > 0);   // same as Find for class types; differs for value types
list.Where(x => x > 0).ToList();    // same as FindAll, more allocation
```

Use whichever reads clearer. The `Find*` methods don't go through delegates → marginally faster.

---

## Conversion

```csharp
int[] array = list.ToArray();      // copy to T[]
List<int> fromArr = arr.ToList();   // copy from T[]
List<int> fromSeq = seq.ToList();   // copy from IEnumerable<T>
```

For zero-copy:

```csharp
Span<int> span = CollectionsMarshal.AsSpan(list);
ReadOnlySpan<int> rspan = CollectionsMarshal.AsSpan(list);
```

The span is valid until you next modify the list (Add, Insert, Remove, etc.). After mutation, the backing array might have moved.

---

## Generic constraints aren't needed

```csharp
public T First<T>(List<T> list) => list[0];   // T can be anything
```

`List<T>` has **no constraints** on T. It works for any type — value or reference, struct or class, nullable or not.

If T is a value type, the items live inline in the backing array — no boxing, no extra heap objects.

---

## Internals — how List grows

The backing array starts at 0 (until first Add). On first Add, it's grown to **4**. Each subsequent overflow doubles:

```
Count→Capacity:
    1 → 4
    5 → 8
    9 → 16
   17 → 32
   33 → 64
```

This is the standard amortized-O(1) dynamic-array growth.

In IL, `List<T>.Add(T item)`:

```csharp
public void Add(T item) {
    if (_size == _items.Length) {
        // Grow
        T[] newItems = new T[_items.Length == 0 ? 4 : _items.Length * 2];
        Array.Copy(_items, newItems, _size);
        _items = newItems;
    }
    _items[_size++] = item;
    _version++;   // for enumerator invalidation
}
```

(Simplified — the actual code is more careful about Capacity vs Length.)

The `_version` counter is what enables the "collection modified" detection in enumerators.

### Enumerator

`List<T>.Enumerator` is a struct:

```csharp
public struct Enumerator : IEnumerator<T> {
    private readonly List<T> _list;
    private int _index;
    private readonly int _version;
    private T _current;
    // ...
    public bool MoveNext() {
        if (_version != _list._version)
            throw new InvalidOperationException("Collection modified");
        if (_index < _list._size) {
            _current = _list._items[_index++];
            return true;
        }
        return false;
    }
}
```

A struct (not class) — no heap allocation per foreach. The compiler optimizes `foreach (var x in list)` to use the concrete `Enumerator` type, avoiding interface dispatch.

For `foreach (var x in (IEnumerable<int>)list)`, you'd lose this — the iteration goes through `IEnumerator<T>` which is boxed.

---

## Common patterns

### Builder

```csharp
var users = new List<User>(capacity: knownSize);
foreach (var row in dataSource) {
    users.Add(MapToUser(row));
}
return users;
```

Pre-sized list, single allocation, populate, return.

### Filter and remove

```csharp
list.RemoveAll(x => !x.IsActive);
```

Single O(n) pass. Faster than `list = list.Where(x => x.IsActive).ToList();` (allocates a new list).

### Convert to array efficiently

```csharp
int[] arr = list.ToArray();      // O(n) copy
ReadOnlySpan<int> span = list.AsSpan();   // O(1), no copy

// .NET 5+:
Span<int> mutSpan = CollectionsMarshal.AsSpan(list);
```

If callers can accept `ReadOnlySpan<T>` or `IReadOnlyList<T>`, you avoid the array allocation entirely.

### Bulk operations

```csharp
list.AddRange(other);          // efficient — adjusts Capacity once
list.InsertRange(0, other);    // efficient — single shift
list.RemoveRange(0, 100);      // efficient — single shift
```

Range methods are O(n) but with one allocation/shift, not n.

### Sort-by-key with stable LINQ

```csharp
list = list.OrderBy(x => x.Key).ThenBy(x => x.SecondaryKey).ToList();
```

LINQ's OrderBy is stable (.NET 6+). For multi-key sort, ThenBy chains.

---

## Common bugs

- **Modifying while iterating** — InvalidOperationException via the version check.
- **`Remove` in a loop** — O(n²). Use `RemoveAll(pred)`.
- **Not pre-sizing for known-large lists** — many reallocations + copies.
- **`list.Capacity = 0` to "clear"** — doesn't work; sets capacity to 0 but throws if Count > 0. Use `Clear()` or `Capacity = list.Count` after clearing.
- **Treating `List<T>` as thread-safe** — it isn't. For concurrent scenarios, use `ConcurrentBag<T>`, `ConcurrentQueue<T>`, or lock externally.
- **Boxing struct items via interface** — `IList<int> il = list; foreach (object o in il)` boxes each int.

---

## Performance vs alternatives

| Operation | `List<T>` | `T[]` | `LinkedList<T>` |
|---|---|---|---|
| Indexed access | O(1) | O(1) | O(n) |
| Add at end | O(1) amortized | (resize) | O(1) |
| Insert at i | O(n) | (resize) | O(1) at node |
| Remove at i | O(n) | (resize) | O(1) at node |
| Find | O(n) | O(n) | O(n) |
| Memory overhead per item | ~0 | 0 | high (next/prev pointers + node object) |
| Cache friendliness | high | highest | low |

`List<T>` is the right default for almost everything. `T[]` for fixed-size buffers; `LinkedList<T>` for frequent mid-list insertions where you already have a node reference.

---

## Thread safety

`List<T>` is **not thread-safe**. Multiple readers are safe if no one writes; any concurrent write is UB.

For concurrent scenarios:
- `ConcurrentBag<T>` — unordered, optimized for same-thread add/remove.
- `ConcurrentQueue<T>` / `ConcurrentStack<T>` — ordered.
- `ImmutableList<T>` — persistent collection.
- `ReaderWriterLockSlim` + `List<T>` — manual concurrency.

[CSharpBook chapter 08](../08-Concurrency/12-ConcurrentCollections.md) covers concurrent collections.

---

## When to use List vs other collections

✓ Sequential data you'll iterate and add to.
✓ Random access by index.
✓ Default when in doubt.

✗ Frequent insert/remove in the middle — consider `LinkedList<T>` or a different design.
✗ Multi-threaded access — use a concurrent collection.
✗ Massive append-only logs — consider `Channel<T>` or a chunked approach.
✗ When you need value equality between lists — use `ImmutableArray<T>` or compare with `SequenceEqual`.

→ Next: [03-LinkedList.md](03-LinkedList.md)
