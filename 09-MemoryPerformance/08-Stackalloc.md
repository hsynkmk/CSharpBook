# stackalloc

## What it is

`stackalloc` allocates memory **on the current method's stack frame** instead of the heap. The "buffer" lives only for the duration of the method; freed when the method returns. Zero GC cost. Faster than even ArrayPool.

```csharp
public string Format(int n) {
    Span<char> buf = stackalloc char[32];
    n.TryFormat(buf, out int written);
    return new string(buf[..written]);
}
```

Stackalloc is for **small, short-lived buffers** — temporary scratch space used inside a method. The killer combo with `Span<T>`: no allocation, no return-to-pool, no cleanup.

---

## Why it exists

For temporary buffers, even ArrayPool has cost:
- ~100 ns to Rent + Return.
- The pool might need to allocate if you exceed its size.

stackalloc:
- ~1 ns (just adjust the stack pointer).
- No GC ever.
- The buffer disappears when the method returns.

For hot loops doing transient buffer work (formatting numbers, parsing tokens, building short strings), `stackalloc` is the absolute fastest option.

---

## Syntax

```csharp
Span<int> nums = stackalloc int[10];
Span<byte> buffer = stackalloc byte[256];
Span<char> chars = stackalloc char[64];
```

Modern syntax (C# 7.2+): `stackalloc` produces a `Span<T>` (or `ReadOnlySpan<T>` if you assign to that). The Span knows the length and points to the stack memory.

Older syntax (unsafe pointers):
```csharp
unsafe {
    byte* ptr = stackalloc byte[256];
    // raw pointer arithmetic — unsafe
}
```

Prefer the Span form. Safer, equally fast.

---

## Where it's allowed

stackalloc can be used:
- Inside a method body.
- Inside a static method.
- Inside a property accessor.
- Inside a constructor.
- Inside an iterator (`yield return`) — limited.

It **cannot** be used:
- In a field initializer.
- Across an `await` (the Span can't cross it anyway).
- For types containing references (only **unmanaged** types allowed; covered below).

### `unmanaged` constraint

Only types where ALL fields (recursively) are value types (no reference type fields) can be stackalloc'd:

```csharp
Span<int> s = stackalloc int[10];          // ✓ — int is unmanaged
Span<DateTime> d = stackalloc DateTime[5];  // ✓
Span<Point> p = stackalloc Point[3];        // ✓ if Point has only value-type fields

class HasRef { public string Name; }
Span<HasRef> r = stackalloc HasRef[3];      // ❌ — HasRef contains a reference
```

The constraint exists because stack memory isn't tracked by the GC the way heap memory is. If a struct contained a reference, the GC wouldn't know about it (might collect the referenced object prematurely).

In generic code, mark the type parameter `where T : unmanaged` to enable stackalloc:

```csharp
public void Process<T>() where T : unmanaged {
    Span<T> buf = stackalloc T[10];   // ✓
}
```

---

## Stack size limits

Threads have limited stacks (~1 MB default). Big stackalloc → **StackOverflowException** → process crash. You cannot catch it.

```csharp
Span<byte> huge = stackalloc byte[10_000_000];   // 💥
```

Rule of thumb: stackalloc up to a few KB (1-4 KB max). For larger, use ArrayPool.

A defensive pattern:

```csharp
const int MaxStack = 1024;
Span<byte> buffer = size <= MaxStack
    ? stackalloc byte[size]
    : new byte[size];   // or ArrayPool.Rent
```

But the conditional has a subtlety: the stackalloc expression in the `<= MaxStack` branch still must be a compile-time constant or fit the size. Modern C# allows:

```csharp
Span<byte> buffer = size <= 256
    ? stackalloc byte[256]
    : new byte[size];

// Then use buffer.Slice(0, size) for the logical view
```

Or:

```csharp
byte[]? rented = null;
Span<byte> buffer;
if (size <= 256) buffer = stackalloc byte[size];
else { rented = ArrayPool<byte>.Shared.Rent(size); buffer = rented.AsSpan(0, size); }

try {
    // use buffer
} finally {
    if (rented is not null) ArrayPool<byte>.Shared.Return(rented);
}
```

The BCL uses this idiom heavily for hot paths.

---

## Common patterns

### Formatting a number into a small buffer

```csharp
public string FormatTwo(int a, int b) {
    Span<char> buf = stackalloc char[32];
    int pos = 0;
    a.TryFormat(buf[pos..], out int wa);
    pos += wa;
    buf[pos++] = ',';
    b.TryFormat(buf[pos..], out int wb);
    pos += wb;
    return new string(buf[..pos]);
}
```

Numbers formatted into a fixed-size stack buffer. Result string is one allocation.

### Parsing without substring

```csharp
public bool TryParseHex(ReadOnlySpan<char> input, out int result) {
    Span<char> upper = stackalloc char[16];
    int len = Math.Min(input.Length, upper.Length);
    for (int i = 0; i < len; i++) upper[i] = char.ToUpperInvariant(input[i]);
    return int.TryParse(upper[..len], NumberStyles.HexNumber, null, out result);
}
```

Normalize to uppercase in a stack buffer, then parse. No allocations.

### Concatenating short strings

```csharp
public string Join(ReadOnlySpan<char> a, ReadOnlySpan<char> b, ReadOnlySpan<char> c) {
    int len = a.Length + b.Length + c.Length;
    Span<char> buf = stackalloc char[len];   // assumes len is small
    int pos = 0;
    a.CopyTo(buf[pos..]); pos += a.Length;
    b.CopyTo(buf[pos..]); pos += b.Length;
    c.CopyTo(buf[pos..]); pos += c.Length;
    return new string(buf);
}
```

For very short strings, stackalloc beats StringBuilder. For long strings (>1 KB), it'd blow the stack.

### Generic algorithm with stackalloc

```csharp
public static T Sum<T>(ReadOnlySpan<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}

public static T Sum<T>(IEnumerable<T> source) where T : unmanaged, INumber<T> {
    Span<T> buf = stackalloc T[256];
    int i = 0;
    foreach (var x in source) {
        buf[i++] = x;
        if (i == 256) {
            T total = T.Zero;
            for (int j = 0; j < 256; j++) total += buf[j];
            // ... handle overflow / batch ...
        }
    }
    // ... etc
}
```

Stack scratch space for batched processing.

---

## Internals — what stackalloc compiles to

```csharp
Span<int> nums = stackalloc int[10];
```

In IL, this becomes a `localloc` instruction:

```il
ldc.i4.s 10
ldc.i4.4    // sizeof(int)
mul
localloc    // allocates on the stack frame, returns native pointer
ldc.i4.s 10
newobj instance void System.Span`1<int>::.ctor(void*, int32)
stloc.0
```

`localloc` extends the current function's stack frame by N bytes, returning a pointer to the new region. When the function returns, the entire stack frame (including the localloc region) is reclaimed instantly.

No GC involvement. No heap allocation. The fastest possible allocation.

### Aligned allocation

The runtime aligns the stack pointer to ensure SIMD-friendly access. For very small allocations, alignment can waste a few bytes, but it's negligible.

### Initialization

By default, the stack memory is **zero-initialized** (the JIT emits clearing code). This matches "every other allocation is zero." 

For performance-critical paths where you'll overwrite every byte anyway, `[SkipLocalsInit]` skips the zero-init:

```csharp
[SkipLocalsInit]
public void Hot() {
    Span<byte> buf = stackalloc byte[256];
    // buf contains undefined bytes! must fill before reading
    FillFully(buf);
}
```

The attribute applies to the whole method (or assembly if applied to the assembly). Removes the zero-clear, can save several ns for big buffers.

Use only when you're certain you fill before read. Tooling like Roslyn analyzers can sometimes catch the mistake.

---

## Stackalloc vs ArrayPool

| | stackalloc | ArrayPool |
|---|---|---|
| Allocation cost | ~1 ns | ~50-100 ns (rent + return) |
| Lifetime | Method scope | Until Return |
| Size limit | KB scale | Up to ~1 MB (`Shared.Rent`) |
| Crosses await | ✗ | ✓ |
| Stored in field | ✗ | ✓ |
| Crosses methods | ✗ (Span only) | ✓ |
| GC pressure | none | minimal |

For very short transient buffers (within a single method): stackalloc.
For larger or cross-method: ArrayPool.
For async / long-held: `MemoryPool<T>` or just `new T[size]` if persistent.

---

## Common patterns

### Format with size-bounded stackalloc

```csharp
public static string FormatGuid(Guid g) {
    Span<char> buf = stackalloc char[36];   // exactly enough for "xxxxxxxx-xxxx-..."
    g.TryFormat(buf, out int written);
    return new string(buf[..written]);
}
```

Guid's max formatted size is known and small — stackalloc is perfect.

### Build a span of known small size

```csharp
public static ReadOnlySpan<char> Trim(ReadOnlySpan<char> input, char ch) {
    int start = 0;
    int end = input.Length;
    while (start < end && input[start] == ch) start++;
    while (end > start && input[end - 1] == ch) end--;
    return input[start..end];
}
```

This doesn't even need stackalloc — just span manipulation. Useful pattern.

### Combine stackalloc with `unsafe` for native interop

```csharp
unsafe {
    byte* buf = stackalloc byte[256];
    int written = SomeNativeFunc(buf, 256);
    Span<byte> span = new Span<byte>(buf, written);
    // ... use ...
}
```

For interop with native APIs that fill a buffer.

---

## Common bugs

### Returning a Span over stackalloc

```csharp
public Span<byte> Make() {
    Span<byte> s = stackalloc byte[10];   // ❌ — Span can't escape the method
    return s;
}
```

Compile error. The Span points to stack memory that will be freed when the method returns; returning it would be a dangling pointer. The compiler prevents this.

### Stackalloc with too-large size

```csharp
public void Process(int size) {
    Span<byte> buf = stackalloc byte[size];   // ⚠ unchecked — could overflow stack
}
```

Always cap the size:
```csharp
Span<byte> buf = size <= 1024 ? stackalloc byte[size] : new byte[size];
```

### Stackalloc in a loop

```csharp
for (int i = 0; i < 1000; i++) {
    Span<byte> buf = stackalloc byte[1024];   // 1 MB total stack use
    // ...
}
```

Each iteration extends the stack frame. By iteration 1000, you've used 1 MB. In a recursive call, this overflows fast.

C# detects this and emits the stackalloc OUTSIDE the loop (the IL allocates once, reuses). But the safety depends on the compiler — for very large per-iteration allocations, you can still blow the stack.

For loop-allocated buffers, prefer ArrayPool.

### stackalloc for types with refs

```csharp
class Box { public int X; }
Span<Box> s = stackalloc Box[10];   // ❌ — not unmanaged
```

Only unmanaged types. Use `Box[]` (heap) or `stackalloc int[10]` for the values.

---

## `[SkipLocalsInit]` for max speed

```csharp
[SkipLocalsInit]
public string Format(int n) {
    Span<char> buf = stackalloc char[32];
    n.TryFormat(buf, out int written);
    return new string(buf[..written]);
}
```

Skips the zero-fill of stackalloc'd memory. Saves a few ns for the clear. Use only when:
- The method always fills the buffer before reading.
- You've measured a perceptible win.

You can apply at assembly level (project-wide) too:
```xml
<PropertyGroup>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```
```csharp
// In AssemblyInfo.cs or any file:
[module: SkipLocalsInit]
```

For Kestrel-class performance code, every byte of clear-time matters. For everyday code, leave the default.

---

## Performance

- `stackalloc N`: ~1 ns + N bytes of stack space.
- Span over stackalloc: same speed as Span over heap array.
- vs `ArrayPool.Rent`: ~100× faster per allocation (but no pooling cost = no return).
- vs `new T[]`: ~10-50× faster, plus no GC pressure.

For ≤256 byte buffers in hot paths, stackalloc is the fastest option in .NET.

---

## When to use

✓ Small (≤1 KB) buffers used within a single method.
✓ Number formatting, hex encoding, simple parsing.
✓ Hot loops where allocation pressure shows up in profiling.
✓ Generic `unmanaged`-constrained code with small workspaces.

✗ Buffers that need to outlive the method.
✗ Big buffers (use ArrayPool).
✗ Buffers crossing `await`.
✗ Types containing references (struct with a string field).

---

## Summary

- `stackalloc` allocates on the stack — fastest possible.
- Produces `Span<T>` (modern, safe form) or pointer (unsafe form).
- Limited to `unmanaged` types (no reference fields).
- Limited size — ≤1 KB rule of thumb; more risks StackOverflowException.
- Can't escape the method.
- For buffers >1 KB or cross-method: use ArrayPool.
- For hot paths in hot libraries: `[SkipLocalsInit]` to skip zero-fill.

→ Next: [09-RefStructsRefLocals.md](09-RefStructsRefLocals.md)
