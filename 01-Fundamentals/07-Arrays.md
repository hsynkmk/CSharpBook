# Arrays

## What it is

An **array** is a fixed-size, indexed sequence of elements of the same type. Arrays are reference types in C# — even though they hold value types like `int[]`, the array itself lives on the heap.

```csharp
int[] nums = { 1, 2, 3, 4, 5 };
Console.WriteLine(nums[0]);     // 1
Console.WriteLine(nums.Length); // 5
nums[0] = 99;                   // mutable
```

---

## Declaring and creating

```csharp
// Inline initialization
int[] nums = { 1, 2, 3 };
int[] nums2 = new int[] { 1, 2, 3 };
int[] nums3 = new[] { 1, 2, 3 };     // type inferred
var nums4 = new[] { 1, 2, 3 };

// Default values
int[] zeros = new int[5];            // [0, 0, 0, 0, 0]
string?[] nulls = new string?[3];    // [null, null, null]
bool[] flags = new bool[3];          // [false, false, false]

// From a collection
int[] arr = list.ToArray();

// Single-element array
int[] one = { 42 };
```

Array length is **fixed at creation**. To resize, allocate a new one:

```csharp
Array.Resize(ref nums, 10);   // creates new array, copies elements, reassigns
```

---

## Indexing

Zero-based. Reading or writing out of bounds throws `IndexOutOfRangeException`:

```csharp
int[] arr = { 10, 20, 30 };
Console.WriteLine(arr[0]);    // 10
Console.WriteLine(arr[2]);    // 30
Console.WriteLine(arr[3]);    // 💥 IndexOutOfRangeException
```

### Range and from-end indexing (C# 8+)

```csharp
int[] arr = { 10, 20, 30, 40, 50 };
arr[^1];       // 50 — last element
arr[^2];       // 40
arr[1..4];     // { 20, 30, 40 } — start incl, end excl
arr[..3];      // { 10, 20, 30 }
arr[2..];      // { 30, 40, 50 }
arr[..];       // full copy
arr[1..^1];    // { 20, 30, 40 }
```

These produce **a new array** (a copy of the slice). For a no-copy slice over an array, use `Span<T>`:

```csharp
Span<int> slice = arr.AsSpan(1, 3);   // { 20, 30, 40 }, no allocation
```

---

## Iterating

```csharp
int[] arr = { 1, 2, 3, 4, 5 };

// foreach — read-only access typical
foreach (var x in arr) {
    Console.WriteLine(x);
}

// for — when you need indices
for (int i = 0; i < arr.Length; i++) {
    Console.WriteLine($"{i}: {arr[i]}");
}

// reverse iteration
for (int i = arr.Length - 1; i >= 0; i--) {
    Console.WriteLine(arr[i]);
}
```

The JIT eliminates bounds checks in `for` loops when it can prove the index is in range (e.g., `for (int i = 0; i < arr.Length; i++)`). `foreach` on arrays is the same speed.

---

## Multi-dimensional arrays

### Rectangular (2D and higher)

```csharp
int[,] grid = new int[3, 4];      // 3 rows, 4 columns, all zeros
grid[0, 0] = 1;
grid[2, 3] = 99;

// Inline init
int[,] mat = {
    { 1, 2, 3 },
    { 4, 5, 6 },
    { 7, 8, 9 }
};

// Length and dimensions
mat.GetLength(0);    // 3 (rows)
mat.GetLength(1);    // 3 (cols)
mat.Length;          // 9 (total elements)
mat.Rank;            // 2 (number of dimensions)

// Iterate
for (int i = 0; i < mat.GetLength(0); i++) {
    for (int j = 0; j < mat.GetLength(1); j++) {
        Console.Write($"{mat[i,j]} ");
    }
    Console.WriteLine();
}
```

Storage is **contiguous** (row-major) — one big block of memory.

3D, 4D, etc. work the same:
```csharp
int[,,] cube = new int[2, 3, 4];
```

### Jagged arrays (array of arrays)

```csharp
int[][] jagged = new int[3][];
jagged[0] = new int[] { 1, 2, 3 };
jagged[1] = new int[] { 4, 5 };
jagged[2] = new int[] { 6, 7, 8, 9, 10 };

// Inline
int[][] jagged2 = {
    new[] { 1, 2, 3 },
    new[] { 4, 5 },
    new[] { 6, 7, 8, 9, 10 }
};

// Iterate
foreach (var row in jagged) {
    foreach (var x in row) Console.Write(x + " ");
    Console.WriteLine();
}

jagged[1].Length;     // 2 — each row has its own length
```

### Rectangular vs jagged — which?

| Rectangular `int[,]` | Jagged `int[][]` |
|---|---|
| Fixed rectangular shape | Each row can be any length |
| Single allocation (contiguous) | One allocation per row + outer array |
| Slightly faster `Length` and `[i,j]` access in some cases | Often **faster** in practice due to JIT optimizations on the simpler `int[]` |
| Can't slice rows efficiently | `arr[i]` gives you a regular `int[]` |
| Indexer is `[i, j]` | `[i][j]` |

In most real code, **jagged is preferred** unless you specifically want the storage layout of rectangular. For numeric/scientific work, libraries like `Memory<T>` or `MathNet.Numerics` provide proper matrix types.

---

## Array methods

`System.Array` provides static methods that work on any array:

```csharp
int[] arr = { 3, 1, 4, 1, 5, 9, 2, 6 };

Array.Sort(arr);                     // mutates in place
// { 1, 1, 2, 3, 4, 5, 6, 9 }

Array.Reverse(arr);                  // also in place
Array.IndexOf(arr, 5);               // index of first 5, or -1
Array.Find(arr, x => x > 3);         // first matching element
Array.FindAll(arr, x => x > 3);      // all matching, as array
Array.Exists(arr, x => x > 8);       // true / false
Array.TrueForAll(arr, x => x > 0);   // all match?
Array.BinarySearch(arr, 5);          // requires sorted; returns index or negative bitwise-complement of insertion point

Array.Clear(arr, 0, arr.Length);     // zero out a range
Array.Copy(src, dst, 5);             // copy 5 elements
Array.Fill(arr, 0);                  // fill with a value

int[] copy = (int[])arr.Clone();     // shallow copy
```

Most of these have LINQ equivalents (`.OrderBy`, `.Where`, `.Any`, `.All`, etc.) that work on `IEnumerable<T>` and are usually preferred unless you specifically need an array result.

---

## Array covariance — a subtle danger

C# arrays are **covariant** at runtime — `Derived[]` can be assigned to `Base[]`:

```csharp
string[] strings = { "a", "b" };
object[] objs = strings;            // legal — covariance

objs[0] = 42;                       // ❌ ArrayTypeMismatchException at runtime!
```

This was a Java-inheritance design decision that hasn't aged well. The compiler accepts the assignment but every write to the array does a runtime type check. For new code, prefer generics — `List<string>` is **not** assignable to `List<object>` (invariant) so the bug is caught at compile time.

---

## Initializing large arrays

For very large fills, `Array.Fill` and `Span<T>.Fill` are fastest:

```csharp
int[] big = new int[1_000_000];
Array.Fill(big, -1);   // SIMD-optimized

// Initialize with index-based values
for (int i = 0; i < big.Length; i++) big[i] = i;
```

For uninitialized arrays (where you'll fill every slot before reading), `GC.AllocateUninitializedArray` skips zero-init:

```csharp
int[] arr = GC.AllocateUninitializedArray<int>(1_000_000);
// You MUST fill every slot before reading — undefined contents otherwise
```

Used in high-performance code that allocates and then overwrites.

---

## Arrays vs `List<T>` — which?

|  | Array `T[]` | `List<T>` |
|---|---|---|
| Size | Fixed | Resizable |
| Indexed access | Yes, fastest | Yes (via internal array) |
| Add / Remove | No (must reallocate) | Yes, amortized O(1) append |
| Memory | Just the data | Internal array, possibly larger than count |
| Multi-dimensional | Yes (`int[,]`) | No (use `List<List<int>>`) |
| Co/Contravariance | Covariant at runtime (dangerous) | Invariant |
| API surface | Static helpers via `Array.*` | Methods on the instance |

**Use `T[]` when**: the size is known and won't change, performance matters, you need multi-dim.

**Use `List<T>` when**: size varies, you need `Add`/`Remove`/`Insert` regularly, you're returning a result from a query.

---

## Span — the no-allocation view

`Span<T>` and `ReadOnlySpan<T>` provide views over array regions, plus stackalloc, plus native memory:

```csharp
int[] arr = { 1, 2, 3, 4, 5 };
Span<int> slice = arr.AsSpan(1, 3);   // points at arr[1..4], no copy
slice[0] = 99;                         // modifies arr[1]
Console.WriteLine(arr[1]);             // 99

// Conversion from string for parsing
ReadOnlySpan<char> chars = "Hello".AsSpan();
ReadOnlySpan<char> first = chars[..3];  // "Hel"
```

Spans are stack-only (`ref struct`) so they can't be stored in fields or crossed `await`. For longer-lived span-like access, see `Memory<T>` in [Chapter 09 §06](../09-MemoryPerformance/06-Memory.md).

---

## Common bugs

- **Off-by-one in `for`**: `i <= arr.Length` accesses `arr[Length]` → `IndexOutOfRangeException`.
- **Modifying array length**: arrays are fixed size. `Array.Resize` creates a new one; `nums.Length = 10;` doesn't compile.
- **Comparing arrays with `==`** : compares references, not contents. Use `arr1.SequenceEqual(arr2)` or `arr1.AsSpan().SequenceEqual(arr2)`.
- **`arr.Clone()` is shallow**: nested arrays still reference the same inner arrays.
- **`new int[5]` vs `new int[] { 5 }`**: the first is an array of 5 zeros; the second is an array with one element (5). Subtle but important.
- **Covariance trap**: storing into `object[]` that's secretly `string[]` throws at runtime.

---

## Performance

- Array element access is O(1) and one of the cheapest operations in .NET.
- `for` and `foreach` over arrays are equivalent.
- LINQ over arrays adds allocation (delegates, enumerators) — fine for general code, avoid on hot paths.
- Use `Span<T>` for hot-path slicing without allocation.
- `Array.Sort` is highly optimized (TimSort for stable sort in some cases, introspective sort otherwise).
- For arrays of primitives, `Array.Sort` uses SIMD where possible.

→ Next: [08-ExceptionsBasics.md](08-ExceptionsBasics.md)
