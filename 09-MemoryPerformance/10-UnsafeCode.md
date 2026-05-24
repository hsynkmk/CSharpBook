# Unsafe Code and Pointers

## What it is

C# supports raw pointer arithmetic and unmanaged memory access through the **`unsafe`** keyword. You opt out of GC safety to gain C-style memory control — direct pointer manipulation, native memory allocation, fixed pinning of heap objects.

```csharp
unsafe {
    int x = 42;
    int* p = &x;
    *p = 99;
    Console.WriteLine(x);   // 99
}
```

99% of C# code never touches unsafe. The cases that do: native interop, SIMD primitives, bit-twiddling algorithms, custom memory managers. Modern .NET has reduced the need — `Span<T>`, `Vector<T>`, and `MemoryMarshal` cover most "I need to be unsafe" cases safely.

---

## Why it exists

Some scenarios genuinely need pointer-level access:
- **Native interop** — passing pointers to OS APIs.
- **SIMD intrinsics** — `Vector128<T>` operations often need pointer alignment.
- **Custom memory allocation** — talking to `NativeMemory.Alloc` or unmanaged heaps.
- **Bit-packing / serialization** — reinterpreting bytes as different types.
- **High-performance crypto** — avoiding bounds checks in inner loops.

For the rest of C#, safety wins. `Span<T>` and `MemoryMarshal` provide most of the perf benefits of unsafe with the safety of the type system.

---

## Enabling unsafe code

Add to your csproj:

```xml
<PropertyGroup>
    <AllowUnsafeBlocks>true</AllowUnsafeBlocks>
</PropertyGroup>
```

Without this, the compiler rejects all unsafe syntax. Opting in is project-wide.

---

## Pointer syntax

```csharp
unsafe {
    int x = 42;
    int* p = &x;      // p is a pointer to x
    int y = *p;        // dereference — y = 42
    *p = 99;            // write through pointer

    int[] arr = { 1, 2, 3 };
    fixed (int* ap = arr) {     // pin the array; ap is a pointer to its first element
        ap[0] = 99;              // direct array access via pointer
        Console.WriteLine(arr[0]);   // 99
    }
}
```

### Operations on pointers

- `&x` — address-of (returns `T*`).
- `*p` — dereference (returns the T at the pointer).
- `p->member` — dereference + member access.
- `p[i]` — same as `*(p + i)`.
- `p + i` — pointer arithmetic, advances by `i * sizeof(T)` bytes.
- `p - q` — pointer difference (in T-sized steps).

Native pointer types: `int*`, `byte*`, `MyStruct*`. `void*` for untyped.

### `fixed` — pinning

The GC moves heap objects during compaction. If you take the address of a heap object, the address could become invalid mid-operation. `fixed` **pins** the object, telling the GC "don't move this":

```csharp
byte[] data = new byte[1000];
fixed (byte* p = data) {
    // Inside this block, data won't move. p is valid.
    SomeNativeFunc(p, data.Length);
}
// Outside: data is unpinned; GC may move it next collection.
```

Pinning interferes with GC compaction — too much/long pinning fragments the heap. Pinned-Object-Heap (POH) helps for long-lived pins (see [§02 GC](02-GarbageCollection.md)).

Common use: passing arrays to native APIs that expect raw pointers.

For strings (immutable):
```csharp
string s = "hello";
fixed (char* p = s) {
    Console.WriteLine((int)*p);   // 104 = 'h'
}
```

---

## stackalloc (already covered)

```csharp
unsafe {
    byte* buf = stackalloc byte[256];
    // pointer to stack memory; lives until method returns
}
```

In modern code, prefer the safe `Span<byte> buf = stackalloc byte[256];` form. The unsafe pointer is only needed for interop or specific low-level work.

---

## NativeMemory — unmanaged heap

`System.Runtime.InteropServices.NativeMemory` (since .NET 6) provides allocation on the unmanaged heap:

```csharp
unsafe {
    void* ptr = NativeMemory.Alloc(1024);
    Span<byte> span = new Span<byte>(ptr, 1024);
    // ... use span ...
    NativeMemory.Free(ptr);
}
```

vs the older `Marshal.AllocHGlobal` — NativeMemory is faster, more semantic.

Use cases:
- Buffers that need to outlive a method but not be GC-managed.
- Interop with native libraries that own their own heap.
- Avoiding GC pressure in extreme high-perf scenarios.

The `Span<byte>(ptr, length)` constructor wraps a native pointer in a Span — safe iteration, indexed access, but you own the lifetime. Free explicitly.

---

## `sizeof` and `Unsafe.SizeOf<T>()`

```csharp
unsafe {
    Console.WriteLine(sizeof(int));     // 4
    Console.WriteLine(sizeof(Point));    // 8 (or whatever)
}
```

`sizeof` is restricted to unmanaged types (primitives + value-type structs containing only value types). For arbitrary `T`, use `Unsafe.SizeOf<T>()`:

```csharp
int size = System.Runtime.CompilerServices.Unsafe.SizeOf<MyStruct>();
```

Same number, no unsafe block needed.

---

## `Unsafe.*` — safe wrappers around unsafe

`System.Runtime.CompilerServices.Unsafe` provides operations that LOOK unsafe but don't require the `unsafe` keyword:

```csharp
ref int slot = ref Unsafe.AsRef<int>(in arr[0]);
ref int element = ref Unsafe.Add(ref slot, 5);   // arr[5]

// Reinterpret cast (be careful!)
ref byte byteRef = ref Unsafe.As<int, byte>(ref slot);
```

`Unsafe.As<T1, T2>` reinterprets the memory at one location as a different type. Dangerous if types don't actually match — like a `reinterpret_cast` in C++.

Used by high-performance code (Span internals, SIMD libraries). Not for everyday code.

---

## MemoryMarshal — safer span/memory manipulation

`System.Runtime.InteropServices.MemoryMarshal` provides reinterpretation between Spans of different types:

```csharp
byte[] bytes = new byte[4];
Span<byte> byteSpan = bytes;
Span<int> intSpan = MemoryMarshal.Cast<byte, int>(byteSpan);   // view as int[]

intSpan[0] = 42;   // writes 4 bytes to the byte array
```

Useful for parsing binary protocols, where you want to read a byte array as a struct or vice versa.

```csharp
byte[] header = readFromNetwork;
ref MyHeader h = ref MemoryMarshal.AsRef<MyHeader>(header);
Console.WriteLine(h.Version);
```

Direct reinterpretation. No unsafe keyword — but you still have to be careful about alignment and matching size.

---

## Common patterns

### Native function call

```csharp
[LibraryImport("mylib", EntryPoint = "process")]
private static partial unsafe int Process(byte* data, int length);

byte[] buffer = new byte[1024];
unsafe {
    fixed (byte* p = buffer) {
        int result = Process(p, buffer.Length);
    }
}
```

P/Invoke + pinning. For modern code, `[LibraryImport]` (C# 11+) source-generates safer interop wrappers.

### SIMD vector operation

```csharp
unsafe {
    Vector256<byte> vec = Vector256.Create<byte>(0);
    byte* dst = stackalloc byte[32];
    vec.Store(dst);
}
```

SIMD intrinsics often need pointers for store/load operations.

### Custom struct + reinterpret

```csharp
struct Header {
    public uint Magic;
    public ushort Version;
    public ushort Flags;
}

byte[] data = ReadFile();
ref Header h = ref MemoryMarshal.AsRef<Header>(data);
if (h.Magic == 0xDEADBEEF) {
    Console.WriteLine($"Version {h.Version}");
}
```

Read structured data from a byte array without copying. Safe-ish — assumes the byte order and layout match.

### Custom memory allocator

```csharp
unsafe {
    void* nativePtr = NativeMemory.Alloc(8 * 1024 * 1024);   // 8 MB
    Span<byte> span = new Span<byte>(nativePtr, 8 * 1024 * 1024);
    // ... use span ...
    NativeMemory.Free(nativePtr);
}
```

Native heap, GC-free, but manually-managed lifetime. Used for very large buffers in low-latency systems where GC pauses are unacceptable.

---

## Internals — what unsafe disables

In unsafe contexts, the compiler disables:
- Bounds checks (on pointer arithmetic).
- Type safety for reinterpretations.
- Some GC safety (you can take addresses).

The runtime still does what it does (GC, JIT). Your code just has new dangers:
- Dangling pointers (writing to freed memory).
- Buffer overruns (writing past allocated bounds).
- Type confusion (treating bytes as a struct that doesn't match).

These are the classic C bugs. Unsafe C# gets you closer to them.

---

## When unsafe is the right tool

- **Native interop** where the API requires pointers.
- **Hot SIMD code** where Vector requires alignment.
- **Custom serializers** processing binary protocols byte-by-byte.
- **Memory-mapped files** with native pointers.
- **Embedded / IoT** with hardware memory layouts.

Modern alternatives (Span, MemoryMarshal, Unsafe.*) cover most cases without `unsafe`. Only reach for actual pointers when there's no managed equivalent.

---

## When unsafe is the WRONG tool

- Micro-optimizations in regular code — Span is usually as fast.
- Avoiding bounds checks — modern JIT often eliminates them anyway.
- "I want to control the bits" — usually clearer with BitConverter, MemoryMarshal, or bit operators.
- Performance theater — profile to confirm there's a real win.

Code review red flag: unsafe without a clear performance justification.

---

## Common bugs

### Pointer outliving its source

```csharp
unsafe {
    int* p;
    {
        int x = 42;
        p = &x;
    }   // x goes out of scope
    Console.WriteLine(*p);   // ⚠ undefined behavior — x is gone
}
```

The pointer points to stack memory that's been freed. Undefined behavior. The compiler doesn't always catch this.

### Forgetting `fixed`

```csharp
unsafe {
    byte[] data = new byte[100];
    byte* p = &data[0];   // ⚠ — GC could move data; p becomes invalid
    SomeNativeFunc(p, 100);   // crash possible
}
```

Always pin with `fixed` when taking the address of heap data.

### Free + use

```csharp
unsafe {
    void* p = NativeMemory.Alloc(1024);
    NativeMemory.Free(p);
    *(byte*)p = 0;   // ⚠ — use after free
}
```

Classic C bug. After Free, the pointer is dangling. Set to null after Free, defensively.

### Mismatched struct layout

```csharp
struct Mine { public int A; public short B; }   // size depends on padding

byte[] bytes = new byte[6];
ref Mine m = ref MemoryMarshal.AsRef<Mine>(bytes);
// ⚠ — Mine's layout includes 2 bytes of padding after B; you have 6 bytes for a struct that's actually 8
```

Use `[StructLayout(LayoutKind.Sequential, Pack = 1)]` to control layout exactly.

### Memory leak

```csharp
unsafe {
    void* p = NativeMemory.Alloc(1024);
    // ... use ...
    // forgot Free
}
```

The OS heap leaks. GC doesn't track it. Production memory growth.

Wrap allocation in IDisposable:
```csharp
public class NativeBuffer : IDisposable {
    private unsafe void* _ptr;
    public unsafe NativeBuffer(int size) { _ptr = NativeMemory.Alloc((nuint)size); }
    public unsafe Span<byte> Span => new Span<byte>(_ptr, /* length */);
    public void Dispose() {
        unsafe {
            if (_ptr is not null) { NativeMemory.Free(_ptr); _ptr = null; }
        }
    }
}
```

---

## Safety nets via static analysis

- `[SuppressUnmanagedCodeSecurity]` is gone in modern .NET — interop is always slightly unsafe.
- Roslyn analyzers flag obvious mistakes.
- AOT compilation often refuses unsafe code with mismatched layouts.

For production unsafe code: write extensive unit tests, use sanitizers (AddressSanitizer via Linux/macOS).

---

## Modern alternatives — use these first

| Goal | Safe alternative | Unsafe equivalent |
|---|---|---|
| Slice an array | `arr.AsSpan(i, len)` | `&arr[i]` + length |
| Get element ref | `ref arr[i]` | `&arr[i]` |
| Reinterpret bytes as struct | `MemoryMarshal.AsRef<T>(span)` | `*(T*)ptr` |
| Native allocation | `NativeMemory.Alloc/Free` | `Marshal.AllocHGlobal/FreeHGlobal` |
| sizeof | `Unsafe.SizeOf<T>()` | `sizeof(T)` in unsafe |
| Cast between Span types | `MemoryMarshal.Cast<T1, T2>(span)` | pointer cast |

Most "unsafe" work can be done safely now. Reach for actual pointers only when:
- A specific API requires `T*`.
- You're doing CPU-intrinsic SIMD that needs pointer parameters.
- You're managing native memory.

---

## Performance

- Unsafe pointer access: same speed as managed `ref` in most cases (JIT generates similar code).
- Eliminating bounds checks: ~1-5% in tight loops (modern JIT eliminates many checks automatically).
- Native allocation vs managed: zero GC pressure but no GC reclaim; manual lifetime.

The performance win of unsafe is usually small compared to alternative optimizations (caching, algorithm changes, SIMD via Vector<T> with safe APIs).

---

## When to use unsafe

✓ Native interop unavoidable.
✓ Custom SIMD intrinsics with pointer arguments.
✓ Native heap management.
✓ Bit-packing / binary protocol parsing where MemoryMarshal isn't enough.

✗ "Skip the safety checks" without measured benefit.
✗ Premature optimization.
✗ When Span / MemoryMarshal / Unsafe.* would work.

---

## Summary

- `unsafe` enables pointer arithmetic, address-of, stackalloc with pointers.
- Enable in csproj with `<AllowUnsafeBlocks>true</AllowUnsafeBlocks>`.
- `fixed` pins heap data so pointer remains valid.
- `NativeMemory.Alloc/Free` for unmanaged heap.
- `MemoryMarshal` / `Unsafe.*` cover most "I want to reinterpret memory" cases safely.
- Use sparingly — modern Span / Memory APIs cover most needs.
- Be paranoid: dangling pointers, leaks, overruns all return when you use unsafe.

→ Next: [11-EscapeAnalysis.md](11-EscapeAnalysis.md)
