# Performance Idioms

## The first rule

**Measure before optimizing.** Most code doesn't need to be fast — it needs to be correct and clear. Profile (see [Chapter 15 §08](../15-BuildTooling/08-Profiling.md)) to find the actual hot path, then apply these idioms there. Premature micro-optimization sacrifices readability for gains that don't matter. The idioms below are for the proven-hot 5% of code.

---

## Reduce allocations

The single biggest perf lever in managed code is **allocating less** — fewer allocations means less GC pressure, fewer pauses, better cache behavior.

### `Span<T>` / `ReadOnlySpan<T>` for slicing without copying

```csharp
// ✗ — Substring allocates a new string
string token = line.Substring(0, line.IndexOf(','));

// ✓ — slice with no allocation
ReadOnlySpan<char> token = line.AsSpan(0, line.IndexOf(','));
```

`Span<T>` lets you work over array/string/stack memory without copying. Parsing, slicing, and buffer manipulation become allocation-free. See [Chapter 09 §05](../09-MemoryPerformance/05-Span.md).

### `stackalloc` for small transient buffers

```csharp
Span<byte> buffer = stackalloc byte[256];   // on the stack — no heap, no GC
int written = Encoding.UTF8.GetBytes(text, buffer);
```

For small (< ~1 KB), short-lived buffers, `stackalloc` avoids the heap entirely. Guard the size to avoid stack overflow; for larger/variable sizes use `ArrayPool`.

### `ArrayPool<T>` for reusable larger buffers

```csharp
byte[] buffer = ArrayPool<byte>.Shared.Rent(8192);
try {
    int read = stream.Read(buffer, 0, buffer.Length);
    Process(buffer.AsSpan(0, read));
} finally {
    ArrayPool<byte>.Shared.Return(buffer);   // always return, ideally in finally
}
```

Pool reusable buffers instead of allocating per call. Critical for I/O loops and serialization. Always `Return` (in `finally`); the returned array may be larger than requested — use the length you asked for. See [Chapter 09 §07](../09-MemoryPerformance/07-ArrayPool.md).

### `StringBuilder` / `string.Create` for string building

```csharp
// ✗ — O(n²) allocations in a loop
string s = ""; foreach (var p in parts) s += p;

// ✓
var sb = new StringBuilder();
foreach (var p in parts) sb.Append(p);
return sb.ToString();

// ✓ — string.Create for known-length, allocation-minimal building
string id = string.Create(10, data, (span, d) => { /* fill span */ });
```

### `params ReadOnlySpan<T>` (C# 13+)

```csharp
void Log(params ReadOnlySpan<object> args) { ... }
Log("a", 1, true);   // stack-allocated, no array per call
```

Zero-allocation variadic calls. See [Chapter 11 §06](../11-ModernFeatures/06-CSharp13.md).

---

## Choose the right type

### `struct` for small, short-lived values

```csharp
public readonly record struct Point(int X, int Y);   // no heap allocation, value semantics
```

Small immutable data (≤ 16 bytes guideline) as a `readonly struct`/`record struct` avoids heap allocation. But beware large structs (expensive copies) and defensive copies — see [Chapter 03 §02](../03-TypeSystem/02-Structs.md).

### Avoid boxing

```csharp
object o = 42;             // ✗ — boxes the int
int x = (int)o;            // unbox

// Use generics to stay unboxed
void Process<T>(T value);  // T stays as its value type, no boxing
```

Boxing allocates. It hides in `object`-typed APIs, non-generic collections (`ArrayList`), `string.Format` args, and some interface calls on structs. Use generics and `Span`-based APIs to avoid it.

### `FrozenDictionary`/`FrozenSet` for read-heavy lookups

```csharp
private static readonly FrozenSet<string> Keywords =
    new[] { "if", "else", "while" }.ToFrozenSet();   // build once, read forever
```

For lookup tables built once and read many times, Frozen collections optimize read speed at construction cost. See [Chapter 07 §10](../07-Collections/10-FrozenCollections.md).

---

## Hot-loop idioms

### Struct enumerators avoid allocation

`List<T>`, arrays, `Span<T>` expose **struct** enumerators — `foreach` over them allocates nothing. But casting to `IEnumerable<T>` boxes the enumerator:

```csharp
// ✓ — struct enumerator, no allocation
foreach (var x in list) { ... }

// ✗ — IEnumerable<T> boxes the enumerator each iteration
IEnumerable<int> seq = list;
foreach (var x in seq) { ... }
```

In hot paths, iterate the concrete type (or `Span<T>`), not the interface.

### Avoid LINQ in hot paths

```csharp
// ✗ — allocates closures, iterators, delegates per call
var sum = items.Where(x => x.Active).Sum(x => x.Value);

// ✓ — explicit loop, zero allocation
int sum = 0;
foreach (var item in items) if (item.Active) sum += item.Value;
```

LINQ is wonderful for readability but allocates (delegates, iterator state machines). In the proven-hot path, a plain loop is faster and allocation-free. **Don't** rewrite all LINQ — only the measured hot spots.

### `CollectionsMarshal` for advanced scenarios

```csharp
// Get a ref to a dictionary value to update in place (no double lookup)
ref var count = ref CollectionsMarshal.GetValueRefOrAddDefault(dict, key, out _);
count++;

// Span over a List<T>'s backing array (read-only iteration without bounds re-checks)
foreach (var x in CollectionsMarshal.AsSpan(list)) { ... }
```

`CollectionsMarshal` exposes the internals of `List<T>`/`Dictionary` for zero-overhead access. Powerful but unsafe — the spans/refs are invalidated by mutation. Use only when measured.

---

## JIT-friendly code

### `[MethodImpl(MethodImplOptions.AggressiveInlining)]`

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public static int Square(int x) => x * x;
```

Hints the JIT to inline a tiny hot method. The JIT usually inlines well on its own — only add this when profiling shows call overhead matters. Don't sprinkle it everywhere.

### Let the runtime help (.NET 10)

Modern .NET does a lot for free — escape analysis (stack-allocates non-escaping objects), Dynamic PGO (re-optimizes hot paths), async box elision. Write clear code; the JIT often optimizes incidental allocations away. See [Chapter 11 §08](../11-ModernFeatures/08-DotNet10Runtime.md). This *reduces* the need for manual micro-optimization.

---

## Async performance

- Use `ValueTask<T>` for hot async methods that often complete synchronously (cache hits) — avoids Task allocation. But it has rules (await once). See [Chapter 08 §04](../08-Concurrency/04-ValueTask.md).
- Avoid `async` for trivial pass-throughs that just `return SomethingAsync()` — return the task directly (but mind exception/using semantics).
- `ConfigureAwait(false)` in libraries avoids context capture overhead.

---

## Common bugs / gotchas

### Optimizing the wrong thing

Without profiling, you'll optimize a method that's 0.1% of runtime while the real cost is a chatty DB query or a lock. **Measure first.**

### Sacrificing correctness/readability for unmeasured gains

Cramming `Span`/`stackalloc`/`CollectionsMarshal` into cold code adds risk and complexity for zero benefit. Apply hot-path idioms only to hot paths.

### `stackalloc` overflow

Large or unbounded `stackalloc` blows the stack. Cap the size; fall back to `ArrayPool` for bigger buffers.

### Forgetting to return pooled arrays

A rented `ArrayPool` array not returned defeats the pool (and may leak). Return in `finally`.

### Holding `Span<T>` incorrectly

`Span<T>` is a ref struct — can't be stored in fields, captured by lambdas, or used across `await`. Use `Memory<T>` for those cases.

---

## Summary

- **Measure first** — these idioms are for the proven-hot 5% of code; the rest should optimize for clarity.
- Reduce allocations: `Span`/`ReadOnlySpan` (slice without copy), `stackalloc` (small buffers), `ArrayPool` (reuse larger buffers), `StringBuilder` (string building).
- Choose types well: small `readonly struct`s, avoid boxing (use generics), Frozen collections for read-heavy lookups.
- Hot loops: iterate concrete types (struct enumerators), avoid LINQ allocations, `CollectionsMarshal` when measured.
- `[AggressiveInlining]` sparingly; trust .NET 10's escape analysis + PGO.
- `ValueTask` for sync-completing hot async; `ConfigureAwait(false)` in libraries.

→ Next: [04-ApiDesign.md](04-ApiDesign.md)
