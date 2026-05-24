# Span&lt;T&gt; and ReadOnlySpan&lt;T&gt;

## What it is

`Span<T>` is a **stack-only struct** that gives you a typed view over a contiguous region of memory — could be a portion of an array, a `stackalloc` buffer, a substring, or unmanaged memory. **Slicing is free** (no copy); reads and writes are array-fast.

```csharp
int[] arr = { 1, 2, 3, 4, 5 };
Span<int> all = arr;                  // implicit conversion
Span<int> middle = arr.AsSpan(1, 3);   // view of indices 1..4 — zero copy
middle[0] = 99;
Console.WriteLine(arr[1]);             // 99 — written through the span

ReadOnlySpan<char> hello = "Hello, World!".AsSpan(0, 5);   // view into the string
```

Added in .NET Core 2.1 (2018). Used pervasively in modern BCL for high-performance parsing, formatting, and I/O.

`ReadOnlySpan<T>` is the immutable variant — read-only, used widely for string slicing without allocation.

---

## Why it exists

Many APIs need "a chunk of memory" without caring whether it came from an array, the stack, native memory, or a slice of a larger buffer. Pre-Span, the options were:

- `T[]` + offset + length parameters everywhere (ugly).
- `ArraySegment<T>` (limited; only wraps arrays).
- Copy into a new array (slow, allocates).

Span unifies all of them under one type. It's a pointer + length. Works the same for any backing memory.

The killer feature: **slicing without allocation**. `arr.AsSpan(0, 5)` is just `(arr_ptr, 5)` — no new object. `substring.Substring(0, 5)` allocates a new string; `substring.AsSpan(0, 5)` does not.

---

## Basics

### Creating a Span

```csharp
// From array
int[] arr = { 1, 2, 3, 4, 5 };
Span<int> s1 = arr;
Span<int> s2 = arr.AsSpan();
Span<int> s3 = arr.AsSpan(1, 3);     // index, length

// From string (ReadOnlySpan<char>)
ReadOnlySpan<char> s = "Hello".AsSpan();
ReadOnlySpan<char> s2 = "Hello".AsSpan(0, 3);   // "Hel"

// From stackalloc
Span<byte> buffer = stackalloc byte[256];

// From another Span
Span<int> slice = s1.Slice(2, 2);   // or s1[2..4]
```

### Indexing and iteration

```csharp
Span<int> s = arr.AsSpan(1, 3);
int x = s[0];          // 2 (which is arr[1])
s[1] = 99;             // mutates arr[2]
int len = s.Length;
foreach (var n in s) Console.WriteLine(n);
```

Bounds-checked like arrays. JIT often eliminates the checks in clean for loops.

### Slicing

```csharp
Span<int> first = s[..2];     // first 2 elements (range form)
Span<int> last = s[^2..];      // last 2 elements
Span<int> middle = s[1..^1];   // exclude first and last
Span<int> all = s[..];          // full view (no-op)
```

All O(1) — adjust pointer and length, no copy.

---

## ReadOnlySpan&lt;T&gt;

The read-only variant. Cannot modify the contents.

```csharp
ReadOnlySpan<int> s = arr.AsSpan();
int x = s[0];   // OK
s[0] = 99;      // ❌ compile error
```

For strings: `string` implicitly converts to `ReadOnlySpan<char>` — the BCL's parsers (`int.TryParse`, `DateTime.TryParse`, etc.) accept `ReadOnlySpan<char>` overloads, enabling allocation-free parsing.

```csharp
ReadOnlySpan<char> input = "12345".AsSpan();
int.TryParse(input, out int n);   // no allocation; no substring needed
```

---

## The "ref struct" restriction

`Span<T>` is a `ref struct` — stack-only. It **cannot**:

- Be a field of a class.
- Be a field of a non-`ref struct`.
- Be boxed (assigned to `object`, `dynamic`).
- Be captured by a lambda or local function (unless `static` and not capturing).
- Cross an `await`, `yield return`, or other state-machine boundary.
- Be used as a generic type argument (unless the constraint `allows ref struct` is present, C# 13+).

```csharp
public class C {
    private Span<byte> _buf;   // ❌ — Span as class field forbidden
}

public async Task M() {
    Span<byte> s = stackalloc byte[10];
    await Task.Delay(100);
    s[0] = 1;   // ❌ — Span can't cross await
}
```

These restrictions exist because Span often points to **stack memory** (`stackalloc` buffer) or pinned memory. Letting it escape to the heap or another method's stack frame could leave a dangling pointer.

For span-like that **can** escape, use `Memory<T>` — see [§06](06-Memory.md).

---

## Common patterns

### Parse without allocating

```csharp
string input = "name=value;count=42";
ReadOnlySpan<char> span = input;
int sep = span.IndexOf('=');
ReadOnlySpan<char> key = span[..sep];
ReadOnlySpan<char> rest = span[(sep + 1)..];
int semi = rest.IndexOf(';');
ReadOnlySpan<char> value = semi < 0 ? rest : rest[..semi];

if (int.TryParse(value, out int n)) {
    // n = 42 — no substring allocations along the way
}
```

For large-scale parsing (logs, CSV, protocols), Span eliminates GB of garbage compared to substring-based code.

### Build buffers in place

```csharp
public string Build(int a, int b) {
    Span<char> buf = stackalloc char[32];
    int written = 0;
    a.TryFormat(buf, out int aLen);
    written += aLen;
    buf[written++] = ',';
    b.TryFormat(buf[written..], out int bLen);
    written += bLen;
    return new string(buf[..written]);
}
```

Format two integers + a comma without intermediate strings. `string` constructor copies the chars; one allocation total.

### Read-only constant data

```csharp
ReadOnlySpan<byte> Magic => "HTTP/1.1"u8;   // C# 11 UTF-8 literal, no allocation

void Check(byte[] data) {
    if (data.AsSpan(0, 8).SequenceEqual(Magic)) {
        Console.WriteLine("HTTP");
    }
}
```

`"..."u8` is a UTF-8 literal — `ReadOnlySpan<byte>` pointing to readonly data in the assembly. No heap allocation.

### Slicing a List

```csharp
List<int> list = new() { 1, 2, 3, 4, 5 };
Span<int> span = CollectionsMarshal.AsSpan(list);   // .NET 5+
foreach (var x in span) { ... }
```

Span over `List<T>`'s internal array. Valid until the list mutates (Add, Remove, etc.).

---

## Span methods

`Span<T>` has methods like an array:

```csharp
span.Length;
span.IsEmpty;
span.Clear();              // zero out
span.Fill(value);           // set all elements
span.CopyTo(otherSpan);
span.TryCopyTo(otherSpan);   // returns false if dst too small
span.Reverse();
span.Sort();                // .NET 5+
span.Slice(start, length);
```

Plus extension methods in `MemoryExtensions`:

```csharp
span.IndexOf(x);
span.IndexOf(otherSpan);
span.SequenceEqual(otherSpan);
span.StartsWith(otherSpan);
span.EndsWith(otherSpan);
span.Contains(x);            // .NET 7+
span.Trim(), TrimStart(), TrimEnd();   // for char spans
```

`MemoryExtensions` provides LINQ-like operators on spans without allocating.

---

## String + Span = power combo

The BCL added many `Span`-based string methods:

```csharp
string s = "Hello, World!";

// Allocation-free length check
if (s.AsSpan().StartsWith("Hello".AsSpan())) { ... }

// Parse a number from middle of a string
ReadOnlySpan<char> numSpan = s.AsSpan(7, 5);
int.TryParse(numSpan, out int n);

// Format into a stack buffer
Span<char> buf = stackalloc char[32];
n.TryFormat(buf, out int written);
```

For high-throughput text processing (JSON parsing, CSV, network protocols), `Span` is the difference between MB/s and GB/s.

---

## Internals — what Span actually is

`Span<T>` is roughly:

```csharp
public readonly ref struct Span<T> {
    private readonly ref T _reference;   // managed pointer to T
    private readonly int _length;
    // ...
}
```

Two fields: a **managed reference** (essentially a pointer the GC knows about) and a **length**. 16 bytes on 64-bit.

`ref T` is a special "ref" — the GC tracks it during collections, updating the pointer if the referenced object moves (e.g., array compaction). This is what makes Span safe even when its target is on the heap.

For unmanaged data (stackalloc, native memory), the ref is just a raw pointer.

### Why "ref struct"

A regular struct can be boxed, stored in a class, passed across `await`. If Span pointed to a stack-allocated buffer and was boxed to the heap, the heap reference would outlive the stack frame — dangling pointer.

The `ref struct` restriction makes this impossible at compile time. The compiler refuses any operation that could let the Span escape its source's lifetime.

### Slicing is just pointer math

`span.Slice(start, length)` returns a new Span with:
- `_reference = original._reference + start * sizeof(T)`
- `_length = length`

No allocation. The original and the slice can both be used (as long as neither outlives the backing memory).

### JIT inlining and bounds check elimination

For a clean loop:

```csharp
for (int i = 0; i < span.Length; i++) {
    Process(span[i]);
}
```

The JIT recognizes the pattern and eliminates per-iteration bounds checks. The generated code is essentially as fast as raw pointer iteration.

`foreach (var x in span)` gets the same treatment.

---

## When Span shines

- **Parsing**: protocols, log files, CSV, JSON — anywhere you'd substring repeatedly.
- **Formatting**: building strings without intermediate allocations.
- **Cryptography / hashing**: chunking data without copies.
- **Network**: reading from a buffer without allocating per-frame.
- **File I/O**: `Stream.Read(Span<byte>)` is allocation-free.
- **High-throughput libraries**: System.Text.Json, System.IO.Pipelines, Kestrel.

---

## When Span doesn't help

- **Code that doesn't allocate much anyway** — premature optimization.
- **Code that needs to escape its scope** (async, fields) — use Memory.
- **Tiny allocations rarely on the hot path** — Span complexity isn't worth it.

For most application code, Span is overkill. For libraries / kernels / hot paths, it's transformative.

---

## Modern BCL APIs that take Span

```csharp
Stream.Read(Span<byte>)
Stream.Write(ReadOnlySpan<byte>)
int.TryParse(ReadOnlySpan<char>, out int)
DateTime.TryParseExact(ReadOnlySpan<char>, ...)
Convert.TryFromBase64Chars(ReadOnlySpan<char>, Span<byte>, out int)
Encoding.UTF8.GetBytes(ReadOnlySpan<char>, Span<byte>)
HashAlgorithm.ComputeHash(Span<byte>)
JsonSerializer.Deserialize<T>(ReadOnlySpan<byte>)
HttpResponseMessage.Content.CopyToAsync(Stream, byte[])  // older
```

If you see a Span overload alongside an array overload, prefer Span when you have a Span — avoids copying.

---

## Common bugs

### Span captured across await

```csharp
public async Task M() {
    Span<byte> buf = stackalloc byte[10];
    await Task.Delay(100);   // ❌ compile error
}
```

Span can't cross await. Use Memory if you need this.

### Slice over mutated List

```csharp
List<int> list = new() { 1, 2, 3 };
Span<int> s = CollectionsMarshal.AsSpan(list);
list.Add(4);   // might resize internal array → s now points to freed memory!
int x = s[0];   // ⚠ undefined behavior
```

Don't mutate the source while a Span over it is alive. Span doesn't notify or invalidate.

### Slicing past the end

```csharp
Span<int> s = arr.AsSpan(0, 3);
var bad = s.Slice(0, 5);   // throws ArgumentOutOfRangeException
```

Bounds checked. Will throw, not silently corrupt.

### Span<T> as a field

```csharp
public class Container {
    private Span<byte> _data;   // ❌ compile error — Span as class field forbidden
}
```

Use `Memory<byte>` for storage; convert to Span only inside synchronous code.

### Allocating a stackalloc bigger than fits

```csharp
Span<byte> huge = stackalloc byte[10_000_000];   // StackOverflowException
```

Stack is ~1 MB. Stackalloc beyond that crashes the process. For big buffers, use `ArrayPool<byte>.Shared.Rent(size)`.

---

## Performance

- Span creation from array: ~1 ns (struct construction).
- Slice: ~1 ns (pointer math).
- Indexed read/write: same as array.
- Bounds check elimination in clean for loops: identical to raw pointer code.
- vs Substring: substring allocates a new string + copy; AsSpan slice doesn't.
- vs new array: massive win — no allocation, no GC.

For libraries that process millions of items, the win is order-of-magnitude.

---

## When to use Span vs Memory vs T[]

| Need | Use |
|---|---|
| Slice a buffer in sync code | `Span<T>` |
| Slice a string allocation-free | `ReadOnlySpan<char>` |
| Slice a buffer that crosses async / is stored | `Memory<T>` |
| Constant binary data | `ReadOnlySpan<byte>` via `"..."u8` |
| Stack-allocated short-term buffer | `Span<T>` from `stackalloc` |
| Long-lived owned buffer | `T[]` from `ArrayPool<T>.Shared.Rent` |

---

## Summary

- `Span<T>` is a stack-only struct: pointer + length over contiguous memory.
- Slicing is free (no copy); reads/writes are array-fast.
- Used pervasively in modern BCL for parsing, formatting, I/O.
- Restrictions (`ref struct`): can't be a class field, can't cross await, can't be boxed.
- For span-like across boundaries, use `Memory<T>` (next file).
- For high-throughput libraries, Span is the difference between MB/s and GB/s.

→ Next: [06-Memory.md](06-Memory.md)
