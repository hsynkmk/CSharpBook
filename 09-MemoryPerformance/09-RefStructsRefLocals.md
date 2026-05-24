# ref struct, ref locals, ref returns

## What it is

C# 7.2+ added **byref** features for high-performance scenarios:

- **`ref struct`** — a struct that can only live on the stack (like `Span<T>`).
- **`ref` locals** — a local variable that's an alias for a memory location.
- **`ref` returns** — a method returning a byref to a field/slot.
- **`ref readonly`** — a read-only byref.
- **`scoped`** (C# 11) — limits a ref's allowed escape.

```csharp
int[] arr = { 1, 2, 3 };
ref int slot = ref arr[1];   // ref local — alias for arr[1]
slot = 99;
Console.WriteLine(arr[1]);    // 99
```

These features are the foundation of zero-allocation code in modern .NET. You'll use them implicitly through `Span<T>`. Direct use is for advanced libraries.

---

## Why they exist

For high-performance code, copying data is expensive. A 64-byte struct passed by value is a 64-byte memcpy per call. References (heap allocations) have lifetime overhead.

`ref` features give you C-style "alias to memory" without sacrificing managed safety:
- Aliased access (read/write the same memory) — no copy.
- Compile-time lifetime tracking — no dangling references.

`ref struct` is the marker that "this type is so tied to its memory that it must not escape the stack" — used to ensure Span<T> remains safe.

---

## `ref` locals

```csharp
int[] arr = { 1, 2, 3, 4, 5 };
ref int slot = ref arr[2];   // alias for arr[2]
slot = 99;
Console.WriteLine(arr[2]);    // 99

// Reassigning the ref (C# 7.3+)
slot = ref arr[3];
slot = 88;
Console.WriteLine(arr[3]);    // 88
Console.WriteLine(arr[2]);    // still 99 (slot now refers to [3])
```

`ref int slot = ref arr[2]` — `slot` IS arr[2], not a copy. Writing to slot writes to arr[2].

`slot = ref arr[3]` — rebinds slot to a different slot. (Without `ref` on the RHS, it would just write to the current slot.)

### `ref readonly`

For read-only aliases:

```csharp
ref readonly int slot = ref arr[2];
int x = slot;     // read OK
slot = 99;        // ❌ — read-only
```

Useful for large structs where you want by-ref to avoid copying but don't want mutation.

---

## `ref` parameters and returns

```csharp
public ref int Find(int[] arr, int target) {
    for (int i = 0; i < arr.Length; i++) {
        if (arr[i] == target) return ref arr[i];   // alias to the array slot
    }
    throw new InvalidOperationException();
}

int[] data = { 1, 2, 3, 4 };
ref int slot = ref Find(data, 3);
slot = 99;
Console.WriteLine(data[2]);   // 99
```

`Find` returns a `ref int` — an alias for the array slot. The caller can read and write it. No copy.

Used in:
- `Span<T>.GetReference()` — returns ref to the first element.
- `Dictionary` with `CollectionsMarshal.GetValueRefOrAddDefault` (.NET 6+).
- `List<T>` with `CollectionsMarshal.AsSpan` then indexing.
- Performance-critical libraries.

### `ref readonly` returns

```csharp
public ref readonly Point Origin => ref _origin;

// Caller can read, not write
ref readonly Point o = ref obj.Origin;
int x = o.X;     // OK
o.X = 5;          // ❌
```

Used for exposing internal large structs without copying.

---

## `ref` parameters

```csharp
public void Increment(ref int n) { n++; }

int x = 5;
Increment(ref x);
Console.WriteLine(x);   // 6
```

Already familiar from [Chapter 01](../01-Fundamentals/06-Methods.md). The parameter is an alias for the caller's variable.

`ref readonly` parameter (C# 12+):

```csharp
public void Process(ref readonly BigStruct s) {
    Console.WriteLine(s.X);
    // s.X = 5;   // ❌ — read-only
}

BigStruct b = ...;
Process(ref b);   // pass by readonly reference — no copy
```

Same effect as `in` but the caller explicitly says `ref readonly` (vs `in` which is implicit). For new code, prefer `in` for back-compat; `ref readonly` for explicit by-ref intent in API design.

---

## `ref struct`

A struct that can ONLY live on the stack:

```csharp
public ref struct StackBuffer {
    public Span<byte> Data;
}
```

Restrictions:
- Cannot be a field of a class.
- Cannot be a field of a non-ref struct.
- Cannot be boxed.
- Cannot be captured by lambda / local function.
- Cannot cross `await` / `yield`.
- Cannot be used as generic type argument (unless `allows ref struct`).

Examples in BCL:
- `Span<T>`, `ReadOnlySpan<T>`.
- `Utf8JsonReader`.
- `SpanReader<T>`, `SpanWriter<T>`.

The restrictions exist to keep the data tied to its stack frame. The compiler ensures it never escapes — no dangling references possible.

---

## Why ref struct can't escape

```csharp
public ref struct StackBuf { public byte X; }

public StackBuf MakeBad() {
    StackBuf b = default;
    return b;   // ⚠ if it escaped, dangling pointer
}
```

Compile error. The ref struct is glued to its method's stack frame. Returning it (or storing it elsewhere) would leave a hanging reference.

The compiler is strict: any operation that could let a `ref struct` escape is forbidden.

---

## `scoped` modifier (C# 11+)

`scoped` is an annotation that says "this ref / ref struct cannot escape this method via this parameter / local."

```csharp
public void Process(scoped ReadOnlySpan<byte> data) {
    // I won't store data anywhere
    // Compiler enforces this
}
```

Most usage of `scoped` is implicit (the compiler infers it). You write it explicitly to override default escape rules — usually to be more restrictive.

Used by libraries that want to make their lifetime rules explicit.

---

## Common patterns

### `ref` for performance-critical state mutation

```csharp
public class Bag {
    private int[] _items = new int[100];

    public ref int this[int i] => ref _items[i];   // ref indexer
}

var bag = new Bag();
bag[5] = 42;
ref int slot = ref bag[5];
slot++;
Console.WriteLine(bag[5]);   // 43
```

Allows in-place mutation without copying. Same as direct array access but works for custom collections.

### CollectionsMarshal.GetValueRefOrAddDefault

```csharp
var dict = new Dictionary<string, int>();
ref int count = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, "key", out bool existing);
if (!existing) count = 1; else count++;
```

Atomic-ish (within single-threaded use) — gets a ref to the dict's slot, can read/write directly. Faster than `TryGetValue + indexer set` (one lookup vs two).

### Aliasing for swap

```csharp
public static void Swap<T>(ref T a, ref T b) {
    (a, b) = (b, a);
}

int x = 1, y = 2;
Swap(ref x, ref y);
Console.WriteLine($"{x} {y}");   // 2 1
```

Generic byref swap. Common building block.

### Pass-by-readonly for large structs

```csharp
public struct BigMatrix { /* lots of data */ }

public void Process(in BigMatrix m) {   // 'in' is implicit ref readonly
    Console.WriteLine(m.M11);
}
```

Without `in`, the caller would copy BigMatrix on each call. With `in`, pass by reference, read-only — no copy.

---

## Internals — how `ref` is represented

A `ref T` is a **managed pointer** — a pointer the GC knows about. When the underlying object moves (during GC compaction), the GC updates the pointer.

For ref to an array element: the pointer points into the array's data area. The GC updates if the array is relocated.

For ref to a stack local: the pointer points into the stack frame. No GC involvement (stack memory doesn't move).

For ref returns: the caller receives the managed pointer. Same GC tracking.

The runtime representation is similar to a pointer, but the C# compiler enforces lifetime rules at the source level.

### Lifetime rules — the compiler's job

```csharp
public ref int Bad() {
    int x = 5;
    return ref x;   // ❌ — x lives only in this stack frame
}
```

The compiler tracks: can this ref escape its source? If returning from a function, the source must outlive the call (must be a field, parameter, or alias of one).

For ref struct, the same rules but applied to all the struct's contents transitively.

---

## C# 13 — `allows ref struct`

A generic constraint that lets a generic parameter accept ref structs:

```csharp
public T Process<T>(T input) where T : allows ref struct {
    return input;
}

Process<Span<byte>>(stackalloc byte[100]);   // ✓
```

Pre-C# 13, this was illegal — `Span<T>` couldn't be a generic type argument because the framework assumed Ts could escape. The constraint relaxes this for code that genuinely keeps the ref struct local.

---

## When to use these features

### Use `ref` locals when:
- Repeatedly accessing the same field/slot — saves on indexing.
- Building algorithms that conceptually need aliases.

### Use `ref` returns when:
- Exposing a slot for in-place mutation (rare).
- Building high-performance containers.

### Use `ref struct` when:
- Building a Span-like type.
- The type holds pointers to stack memory.
- You need to prevent escape for safety.

### Use `in` / `ref readonly` parameters when:
- Passing large structs by read-only reference (avoid copy).
- API design wants explicit "I won't mutate this."

### When NOT:
- Everyday code where copies are cheap.
- Premature optimization.

---

## Common bugs

### Returning ref to a local

```csharp
public ref int Bad() {
    int x = 5;
    return ref x;   // ❌ compile error
}
```

Compiler stops you. Good.

### Storing ref struct as a field

```csharp
public class Container {
    private Span<byte> _buf;   // ❌ compile error
}
```

Use `Memory<byte>` for class fields.

### Async with ref struct

```csharp
public async Task Process() {
    Span<byte> buf = stackalloc byte[100];
    await Task.Delay(1);
    buf[0] = 1;   // ❌ — Span can't cross await
}
```

The compiler catches this. Use Memory if you need this pattern.

### Capturing ref struct in lambda

```csharp
Span<byte> buf = stackalloc byte[10];
Action a = () => buf[0] = 1;   // ❌ — can't capture Span
```

Don't. Use Memory or rewrite the design.

### Reassigning the wrong way

```csharp
ref int slot = ref arr[2];
slot = arr[3];   // doesn't reassign the ref — writes arr[3]'s VALUE to arr[2]
slot = ref arr[3];   // ✓ — actually reassigns the alias
```

The `ref` keyword on RHS is required for rebinding.

---

## Performance

- `ref` local / parameter: essentially free.
- `ref` return: same as returning an int (the pointer).
- `ref struct` type: same as a regular struct, with the safety restrictions enforced at compile time only.
- `in` parameter for large struct: O(1) vs O(sizeof) copy. Big win for 100+ byte structs.

For typical methods, no measurable difference. For tight inner loops or large-struct APIs, real wins.

---

## Summary

- `ref` features give you C-style aliasing with managed safety.
- `ref locals` / `ref returns` — aliases for memory locations; no copy.
- `ref struct` — stack-only types (like Span); compiler enforces no escape.
- `ref readonly` / `in` — read-only by-reference; for large struct params.
- `scoped` — additional lifetime tightening (C# 11+).
- `allows ref struct` — generic constraint allowing ref struct args (C# 13+).
- Foundation for `Span<T>`, modern BCL high-performance APIs.

→ Next: [10-UnsafeCode.md](10-UnsafeCode.md)
