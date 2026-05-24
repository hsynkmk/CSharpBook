# Array

## What it is

A `T[]` (array) is a **fixed-size, contiguous block of elements** of the same type. Arrays are reference types — even `int[]` is allocated on the heap — but the elements live inline within the array's storage. Indexed by `int` (or `Index` / `Range` since C# 8).

```csharp
int[] nums = new int[5];           // length 5, all zeros
int[] inline = { 1, 2, 3 };         // initialized
int[] sized = new[] { 1, 2, 3 };    // inferred type
nums[0] = 42;
Console.WriteLine(nums.Length);     // 5
Console.WriteLine(nums[^1]);         // 0 (last element)
```

Arrays are **the most primitive collection** in .NET. Every other indexed collection (`List<T>`, `StringBuilder`, `Span<T>`, etc.) wraps an array internally.

---

## Why it exists

Arrays are the **closest thing to raw memory** that managed code touches:
- O(1) random access.
- Cache-friendly contiguous layout.
- No per-element overhead.
- Direct interop with native code (with pinning).

Every higher-level collection trades some of those properties for resizability, sorted access, or other features. When you actually need raw indexed storage, `T[]` is the choice.

---

## Declaring and creating

```csharp
// Single-dimensional
int[] a1 = new int[5];                 // length 5
int[] a2 = new int[5] { 1, 2, 3, 4, 5 };
int[] a3 = new[] { 1, 2, 3 };          // type inferred
int[] a4 = { 1, 2, 3 };                 // implicit array

// Multi-dimensional (rectangular)
int[,] grid = new int[3, 4];           // 3 rows, 4 columns, all zeros
int[,] inline = { { 1, 2 }, { 3, 4 }, { 5, 6 } };   // 3x2

// Jagged (array of arrays)
int[][] jagged = new int[3][];          // 3 rows, sizes set later
jagged[0] = new[] { 1, 2 };
jagged[1] = new[] { 3, 4, 5 };
jagged[2] = new[] { 6 };

// Collection expression (C# 12+)
int[] modern = [1, 2, 3, 4, 5];
```

Arrays are fixed-size at construction. To "resize", allocate a new array (or use `Array.Resize`):

```csharp
Array.Resize(ref nums, 10);   // copies old elements; rest are zeros
```

---

## Indexing

Zero-based. Out-of-range throws `IndexOutOfRangeException`:

```csharp
int[] a = { 10, 20, 30 };
a[0];     // 10
a[2];     // 30
a[3];     // 💥 IndexOutOfRangeException
a[-1];    // 💥 IndexOutOfRangeException
```

### Index / Range (C# 8+)

```csharp
int[] a = { 1, 2, 3, 4, 5 };
a[^1];         // 5 — last element
a[^2];         // 4
a[..3];        // { 1, 2, 3 } — start (incl) to end (excl)
a[2..];        // { 3, 4, 5 }
a[1..^1];      // { 2, 3, 4 }
a[..];         // full copy
```

`a[..3]` **allocates a new array** (a copy). For a no-copy view, use `Span<T>`:

```csharp
Span<int> slice = a.AsSpan(1, 3);   // no allocation, view into a
slice[0] = 99;                       // modifies a[1]
```

---

## Iterating

```csharp
int[] a = { 1, 2, 3, 4, 5 };

// foreach — most common
foreach (var n in a) Console.WriteLine(n);

// for — when you need the index
for (int i = 0; i < a.Length; i++) Console.WriteLine($"{i}: {a[i]}");

// Reverse
for (int i = a.Length - 1; i >= 0; i--) Console.WriteLine(a[i]);

// Span-based for fast slicing
foreach (var n in a.AsSpan(1, 3)) Console.WriteLine(n);
```

The JIT eliminates bounds checks in `for (int i = 0; i < arr.Length; i++)` patterns. Both `foreach` and the indexed `for` are equally fast on arrays.

---

## Multi-dimensional vs jagged

### Rectangular `int[,]` — single block

```csharp
int[,] grid = new int[3, 4];
grid[0, 0] = 1;
grid[2, 3] = 99;

grid.GetLength(0);    // 3 (rows)
grid.GetLength(1);    // 4 (cols)
grid.Length;          // 12 (total)
grid.Rank;            // 2 (number of dimensions)

for (int i = 0; i < grid.GetLength(0); i++)
    for (int j = 0; j < grid.GetLength(1); j++)
        Console.Write(grid[i, j]);
```

One contiguous heap allocation; storage is row-major.

### Jagged `int[][]` — array of arrays

```csharp
int[][] jagged = new int[3][];
jagged[0] = new[] { 1, 2, 3 };
jagged[1] = new[] { 4, 5 };
jagged[2] = new[] { 6, 7, 8, 9 };

foreach (var row in jagged)
    foreach (var n in row)
        Console.Write(n);
```

Multiple allocations (one per row + the outer array). Rows can have different lengths.

### Which to use?

For most use, **jagged wins** for performance — the inner `int[]` is just a regular array, and the JIT optimizes it aggressively (SIMD, bounds-check elision). Rectangular `[,]` has its own access pattern that the JIT optimizes less well.

For mathematical matrices with fixed dimensions, libraries like `Math.NET` or `System.Numerics.Vector<T>` are better than raw arrays.

---

## Static `Array` helpers

`System.Array` provides static methods that work on any array:

```csharp
int[] a = { 3, 1, 4, 1, 5, 9, 2, 6 };

Array.Sort(a);                       // in place
Array.Reverse(a);                    // in place
Array.IndexOf(a, 5);                 // index of first 5, or -1
Array.LastIndexOf(a, 1);             // index of last 1
Array.Find(a, x => x > 3);            // first matching item
Array.FindAll(a, x => x > 3);         // all matching items, as array
Array.FindIndex(a, x => x > 3);       // first matching index
Array.Exists(a, x => x > 8);          // any?
Array.TrueForAll(a, x => x > 0);      // all?

Array.BinarySearch(a, 5);            // requires sorted; returns index or negative bitwise complement of insertion point

int[] copy = (int[])a.Clone();        // shallow copy
Array.Copy(src, 0, dst, 0, count);    // copy a range
Array.Clear(a, 0, a.Length);          // zero out a range
Array.Fill(a, -1);                    // fill range with value
```

Most have LINQ equivalents (`.OrderBy`, `.Where`, `.Any`, `.All`) — use whichever reads better.

---

## Array covariance — the legacy mistake

C# arrays are **covariant at runtime** — `T[] : object[]` if T is a reference type. This was inherited from Java (which inherited it from older OOP designs):

```csharp
string[] strs = { "a", "b" };
object[] objs = strs;        // legal — array covariance
objs[0] = 42;                // ⚠ ArrayTypeMismatchException at runtime
```

The compile-time type says "object[], can hold any object." The runtime type is still `string[]`. Storing an int triggers a per-write check that fails.

**Modern lesson**: avoid `object[]` parameters when generics work. `List<T>` is invariant (compile-time error instead of runtime) — safer.

For value-type arrays, covariance doesn't apply: `int[]` is NOT `object[]` at runtime. (Boxing would be needed; no implicit conversion.)

---

## Length is fixed

You can't add or remove elements:

```csharp
int[] a = new int[5];
a.Add(99);   // ❌ — no such method
```

To "grow":

```csharp
Array.Resize(ref a, 10);   // allocates new array, copies, reassigns ref
```

For dynamic growth, use `List<T>`. For frequent growth, `List<T>` is the right tool; for fixed-size buffers, `T[]`.

---

## Filling at creation

```csharp
int[] zeros = new int[1000];                   // all zeros (default for int)
int[] all99 = Enumerable.Repeat(99, 1000).ToArray();   // less efficient

int[] filled = new int[1000];
Array.Fill(filled, 99);                         // SIMD-accelerated
```

For uninitialized arrays (where you'll fill every slot before reading), `GC.AllocateUninitializedArray<T>` skips the zero-fill:

```csharp
byte[] buffer = GC.AllocateUninitializedArray<byte>(1_000_000);
// Contents undefined! Fill before reading.
```

Used in performance-critical paths where you know you'll overwrite everything.

---

## Span and Memory interop

```csharp
int[] a = { 1, 2, 3, 4, 5 };
Span<int> span = a;                  // implicit conversion
ReadOnlySpan<int> readOnly = a;       // also fine
Span<int> slice = a.AsSpan(1, 3);

slice[0] = 99;
Console.WriteLine(a[1]);   // 99 — slice writes through to the array

Memory<int> mem = a.AsMemory();
// Memory<T> can be stored in fields and cross await; Span<T> can't
```

`Span<T>` is the modern way to pass array slices around without copying. See CSharpBook chapter 09.

---

## Internals — memory layout

A `T[]` on 64-bit .NET:

```
offset 0:  sync block (8 bytes)
offset 8:  method table pointer (8 bytes)
offset 16: length (4 bytes — int32)
offset 20: padding (4 bytes — alignment)
offset 24: element[0]
offset 24 + sizeof(T): element[1]
...
```

For `int[5]`:
- Header: 16 bytes
- Length: 4 bytes
- Padding: 4 bytes
- Elements: 5 × 4 = 20 bytes
- Total: 44 bytes (often rounded up to 48 for alignment)

For `string[5]`:
- Same header + length.
- Elements: 5 × 8 = 40 bytes (each is a reference / pointer)
- Total: 64 bytes (the actual string objects are elsewhere on the heap)

For `byte[1024]`:
- Header: 16 bytes
- Length: 4 bytes  
- Padding: 4 bytes
- Elements: 1024 bytes
- Total: 1048 bytes

### Large Object Heap

Arrays ≥ 85,000 bytes go to the **Large Object Heap (LOH)** — not compacted, slower to allocate, only collected in Gen2 GCs. For arrays you'll churn through, prefer `ArrayPool<T>` over `new T[size]`:

```csharp
byte[] buf = ArrayPool<byte>.Shared.Rent(100_000);
try { /* use buf */ }
finally { ArrayPool<byte>.Shared.Return(buf); }
```

See [Chapter 09 §07](../09-MemoryPerformance/07-ArrayPool.md).

### IL for indexed access

```csharp
int[] a = { 1, 2, 3 };
int x = a[1];
```

In IL:
```il
ldloc.0       // a
ldc.i4.1      // 1
ldelem.i4     // load int element
stloc.1
```

`ldelem.i4` is a single CPU instruction (after bounds check) — direct memory access. Fastest possible element load.

For `a[1] = 99`:
```il
ldloc.0
ldc.i4.1
ldc.i4 99
stelem.i4
```

`stelem` includes a runtime type check for reference-type arrays (the covariance check). For value-type arrays, it's elided.

### Bounds checks

Every `a[i]` access has an implicit `if (i < 0 || i >= a.Length) throw IndexOutOfRangeException()` check. The JIT eliminates this check when it can prove the index is in range — typically inside `for (int i = 0; i < a.Length; i++)` loops.

For unchecked access (rare, unsafe):
```csharp
ref int slot = ref MemoryMarshal.GetArrayDataReference(a);
// Direct memory access without bounds check. ⚠ unsafe — caller must validate.
```

Used in highly optimized math/SIMD code.

---

## Common patterns

### Two-pointer technique

```csharp
public static int[] Reverse(int[] a) {
    int i = 0, j = a.Length - 1;
    while (i < j) {
        (a[i], a[j]) = (a[j], a[i]);
        i++; j--;
    }
    return a;
}
```

Classic algorithm style — fast and allocation-free.

### Sort with custom comparer

```csharp
Array.Sort(arr, (a, b) => b.CompareTo(a));   // descending
Array.Sort(arr, Comparer<int>.Create((a, b) => b.CompareTo(a)));
```

For more complex sorting, use `Array.Sort` with a `Comparison<T>` delegate or an `IComparer<T>` instance.

### Binary search

```csharp
Array.Sort(a);
int idx = Array.BinarySearch(a, target);
if (idx < 0) {
    int insertionPoint = ~idx;   // bitwise complement gives insertion index
    // ...
}
```

`BinarySearch` requires sorted input. Returns the index of the target, or a negative bitwise-complement of the insertion point.

### Fast clone

```csharp
int[] copy = (int[])a.Clone();    // shallow copy — for value types, sufficient
int[] copy2 = a.ToArray();         // LINQ — same effect, slightly more allocation overhead
Buffer.BlockCopy(a, 0, copy, 0, a.Length * 4);   // byte-level copy, fastest for primitives
```

---

## Common bugs

- **Off-by-one** — `for (i = 0; i <= a.Length; i++)` reads past the end → IndexOutOfRangeException.
- **Array covariance trap** — `object[] o = strings; o[0] = 42;` throws at runtime.
- **`arr.Clone()` is shallow** — for arrays of reference types, the references point to the same objects.
- **`Array.Sort` mutates in place** — surprising if you wanted to keep the original. Clone first.
- **Comparing arrays with `==`** — compares references, not contents. Use `arr1.SequenceEqual(arr2)` or `arr1.AsSpan().SequenceEqual(arr2)`.
- **`new int[5] { 5 }`** — error; the inline initializer must match length. `new int[] { 5 }` is length-1.

---

## Performance summary

- Indexed access: O(1), single CPU instruction.
- Length: O(1), cached field.
- Iteration: O(n), highly optimized (bounds-check elision).
- Sort: O(n log n) (introsort hybrid).
- BinarySearch: O(log n).
- IndexOf: O(n).
- Resize: O(n) (allocate new + copy).

### When to use vs alternatives

| Need | Array `T[]` | `List<T>` | Other |
|---|---|---|---|
| Fixed size | ✓ | over-allocates | |
| Dynamic size | new array per change | ✓ | |
| Multi-dim | `int[,]` or jagged | nested `List<List<T>>` | |
| Highest perf indexing | ✓ | ✓ (same speed) | |
| Span integration | ✓ | via `CollectionsMarshal.AsSpan` | |
| LINQ | ✓ | ✓ | |

---

## When to use arrays

✓ Fixed-size buffers, especially in I/O paths.
✓ Multi-dimensional data (jagged arrays).
✓ Native interop (with pinning).
✓ Hot loops where every cycle matters.
✓ When you need `Span<T>` interop.

✗ Dynamic-size collections that grow — use `List<T>`.
✗ Frequent insertion/removal in the middle — use `LinkedList<T>` or `List<T>` (different trade-offs).
✗ When you'd benefit from named indexing — use `Dictionary<TKey, TValue>`.

→ Next: [02-List.md](02-List.md)
