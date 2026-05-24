# Stack vs Heap

## What it is

Memory in a .NET process is organized into several regions; the two you care about most are the **stack** and the **heap**.

- **Stack** — fast, per-thread, automatically reclaimed when methods return. Holds locals, parameters, return addresses, and value-type instances.
- **Heap** — slower, GC-managed, shared across threads. Holds all reference-type objects (and boxed value types).

```csharp
int x = 5;                           // x lives on the stack
string s = "hello";                  // s (the reference) lives on the stack; the string object on the heap
var list = new List<int>();           // list (the reference) on stack; List<int> object on heap
```

Understanding the split is foundational. Every performance discussion eventually comes back to it.

---

## Why the split exists

Two memory regions, two different jobs:

- **Stack** is dirt cheap: allocating means decrementing the stack pointer; freeing means incrementing it back. ~1 ns per op. Per-thread, so no synchronization needed.
- **Heap** is slow: the allocator hands out blocks from a managed pool; the GC reclaims unreferenced ones. Per-allocation cost is small but adds up; the **collection** phase is the real cost.

Value types (int, struct) and short-lived locals get the stack's speed. Long-lived shared objects, polymorphic instances, and anything needing finalization go on the heap.

---

## What lives where — the typical mental model

```
┌─────────────────────────────────────────┐
│           STACK (per thread)            │
│ ┌─────────────────────────────────────┐ │
│ │ Method's locals                      │ │
│ │ Method's parameters                  │ │
│ │ Return address                       │ │
│ │ Value-type instances declared local  │ │
│ │ References to heap objects           │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘

┌─────────────────────────────────────────┐
│            HEAP (process-wide)          │
│ ┌─────────────────────────────────────┐ │
│ │ All reference-type objects           │ │
│ │ Boxed value-type instances           │ │
│ │ Strings                              │ │
│ │ Arrays                               │ │
│ │ Closures / async state machine boxes │ │
│ └─────────────────────────────────────┘ │
└─────────────────────────────────────────┘
```

When you write:

```csharp
int n = 42;                       // n is on the stack: 4 bytes containing 42
string s = "hello";                // s on stack: 8-byte pointer; "hello" string on heap
List<int> list = new();            // list on stack: 8-byte pointer; List<int> on heap
```

**But this mental model is simplified.** Real layout is more nuanced — see "Internals" below.

---

## The big consequences

### Copy semantics

Value-type assignment copies the bits. Reference-type assignment copies the pointer.

```csharp
int a = 5;
int b = a;
b = 10;
Console.WriteLine(a);   // 5 — independent copy

class Box { public int X; }
Box ba = new() { X = 5 };
Box bb = ba;
bb.X = 10;
Console.WriteLine(ba.X);  // 10 — same heap object
```

### Parameter passing

Value types pass by value (copied). Reference types pass the reference (the heap object is shared).

```csharp
void Inc(int n) { n++; }
int x = 5; Inc(x); Console.WriteLine(x);   // 5

void Clear(List<int> lst) { lst.Clear(); }
var lst = new List<int> { 1, 2, 3 };
Clear(lst); Console.WriteLine(lst.Count);   // 0
```

### Allocation cost

Stack allocation is essentially **free** — pointer bump.
Heap allocation is cheap but real — ~10-50 ns per object on a hot allocator.
**Collection** (GC) is the bigger cost — every Gen0 collection involves scanning the heap, finding unreachable objects, and reclaiming them.

For high-throughput code: minimize heap allocations on the hot path. Reuse buffers (`ArrayPool`), use structs where appropriate, avoid boxing.

### Cache friendliness

Stack data is hot in CPU caches — recently used, contiguous. Heap data is scattered — pointer chasing causes cache misses. For numeric crunching, an `int[]` (heap) iterated in order is cache-friendly; an `object[]` of int boxes is a cache nightmare.

---

## Where the mental model breaks down

"Value types are on the stack, reference types on the heap" is a useful starting point but **not always literally true**:

### Value-type fields of a class

```csharp
class Container {
    public int X;
    public DateTime When;
    public Point P;
}
```

`X`, `When`, and `P` are value-type fields of a heap-allocated class. They live **inside** the class's heap allocation — not on the stack. Same memory region as the class itself.

### Boxed value types

```csharp
int x = 5;
object o = x;   // boxing — heap allocation
```

`o` (the reference) is on the stack; the boxed int (a heap object wrapping 5) is on the heap.

### Captured locals in closures

```csharp
void M() {
    int x = 5;
    Action a = () => Console.WriteLine(x);   // captures x
}
```

`x` is "hoisted" to a heap-allocated **closure class**. Even though it looks like a local, it lives on the heap because the lambda needs to reference it after `M` returns.

### Async state machine

```csharp
async Task<int> M() {
    int local = 5;
    await Task.Delay(100);
    return local;   // local captured across the await
}
```

When `M` suspends on the await, the locals can't stay on the stack (the stack frame is gone). They're hoisted into a **state machine struct**. On first suspension, the struct is boxed to the heap.

### Stack-allocated reference objects (.NET 10!)

The JIT's **escape analysis** can determine that some heap-allocated objects don't escape the method. In .NET 10, these can be promoted to the stack — eliminating the heap allocation. See [§11 EscapeAnalysis](11-EscapeAnalysis.md).

### Large value types

A struct with many fields is still "on the stack" when it's a local, but copying it can be expensive. For ≥16 bytes, consider passing by `in` reference (read-only by-ref):

```csharp
struct Big { /* 100 bytes */ }
void Process(in Big b) { /* read b, can't write */ }
```

Compiler passes a pointer, no copy.

### Stackalloc

```csharp
Span<byte> buffer = stackalloc byte[256];   // 256-byte buffer ON THE STACK
```

`stackalloc` literally puts the buffer on the stack. Useful for short-lived buffers in hot loops — no heap allocation. See [§08 Stackalloc](08-Stackalloc.md).

---

## Stack characteristics

- **Per-thread**: each thread has its own stack (default ~1 MB on 64-bit).
- **LIFO**: methods push frames as they're called; pop when they return.
- **Limited size**: deep recursion or huge `stackalloc` blows the stack → `StackOverflowException` (process kills).
- **Auto-cleanup**: when a method returns, its stack frame is reclaimed — no GC needed.
- **Cache-friendly**: contiguous, recently-touched memory.

---

## Heap characteristics

- **Process-wide**: one heap shared by all threads.
- **GC-managed**: garbage collector tracks reachability, reclaims unreferenced objects.
- **Generational**: Gen0 (new), Gen1 (survived once), Gen2 (long-lived), LOH (large objects), POH (pinned).
- **Possibly compacted**: most generations are compacted (objects moved to defrag); LOH isn't (by default).
- **Allocations cost a few ns + the cost of eventual collection**.

See [§02 GarbageCollection](02-GarbageCollection.md) for the deep version.

---

## The cost of heap allocations

Every `new` of a reference type:
1. Increments the allocator (bump pointer) — ~10 ns.
2. Zeros the memory (most allocations).
3. Eventually triggers a Gen0 GC (when budget exceeded) — ~milliseconds.

For typical app code: invisible. For hot paths (parsing, serialization, math kernels): can be the dominant cost.

Symptoms of allocation pressure:
- High CPU but low throughput (lots of GC work).
- Periodic latency spikes (Gen2 collections).
- `gc-heap-size` keeps growing in `dotnet-counters`.

Mitigations:
- Use structs for short-lived values (Point, Range, etc.).
- Use `ArrayPool<T>.Shared` for temporary buffers.
- Use `Span<T>` to slice without allocation.
- Avoid boxing.
- Use `Memory<T>` for async paths.
- Pre-allocate when sizes are known.

---

## Internals — process memory layout

A .NET process on x64 Linux/Windows roughly:

```
0xFFFFFFFF... (top)
┌────────────────────────────────┐
│ Kernel space                    │
├────────────────────────────────┤
│ Stack (thread N)                │
│ Stack (thread 1)                │
│ ...                             │
│ Memory-mapped files              │
│ Native heaps (P/Invoke)         │
│ ...                             │
│ Managed heaps (Gen0/1/2/LOH/POH)│
│ ...                             │
│ Loaded assemblies (code + data) │
│ ...                             │
│ Process startup code            │
└────────────────────────────────┘
0x00000000... (bottom)
```

The CLR sets up the managed heap as a series of large segments. Each thread gets its own stack region.

For a 64-bit process, the virtual address space is effectively unlimited (256 TB on Windows, similar on Linux), but **committed** memory is what counts — that's what consumes physical RAM.

### How allocation works (Gen0)

Gen0 is implemented as a **bump allocator**:
- A free pointer marks the next available address in the Gen0 region.
- New object → atomically advance the pointer by the object's size.
- ~10 ns per allocation.

Each thread has a **thread-local allocation context** to avoid contention on the bump pointer. Threads each have a small reserved Gen0 chunk; they bump within that chunk; rare central refill.

When Gen0 is "full" (its budget exceeded), a Gen0 GC kicks in. See next file.

### How stack allocation works

The CPU has a dedicated **stack pointer register** (RSP on x64). Function prologue subtracts from RSP to make room; epilogue adds back. Locals are addressed as offsets from RSP. Sub-nanosecond — typically free relative to the function call itself.

For value-type structs, allocation is just "reserve some bytes in the current stack frame."

`stackalloc N` extends the stack frame by N bytes, returns a pointer. Same mechanism; just dynamic-sized.

---

## When to think about it

You should think about stack vs heap:
- Designing a struct vs class.
- Hot loops with millions of iterations.
- Performance-critical libraries.
- Reducing GC pressure in long-running services.
- Span-based APIs.

You should NOT obsess:
- Application-level code where I/O dominates.
- Once-per-request allocations.
- Anywhere `dotMemory` shows no allocation pressure.

Profile first. Optimize what's slow.

---

## Common bugs and confusions

### "Structs are always faster than classes"

False. Small structs (≤16 bytes, immutable) are fast. Large structs cost a lot on every copy (assignment, parameter pass, return). For 100-byte struct passed by value in a hot loop, the copy dominates. Use `in` parameters or just use a class.

### "I should put my big array on the stack"

`stackalloc` is for small (≤1 KB), short-lived buffers. Big stackalloc → StackOverflowException. For big buffers, use `ArrayPool<byte>.Shared.Rent(size)`.

### "Local string is on the stack"

No. `string` is a reference type. The string OBJECT is on the heap. The reference variable is on the stack.

### Closure surprises

```csharp
public Func<int> Counter() {
    int n = 0;
    return () => n++;
}
```

`n` is "hoisted" to a heap-allocated closure object. Even though it's syntactically a local, it lives on the heap because the returned lambda references it.

---

## Performance summary

| Operation | Approx cost |
|---|---|
| Stack allocation (function entry) | ~1 ns (sub-nanosecond per local) |
| Stack-allocated read/write | ~1 ns |
| Heap allocation (Gen0) | ~10-50 ns |
| Heap-allocated read/write | ~1-2 ns (after first cache miss) |
| Gen0 collection | ~ms |
| Gen2 collection | ~tens of ms |
| Box a value type | heap allocation + memcpy |

For tight loops, the per-allocation cost is real. For typical code, allocation patterns matter more than per-call cost.

---

## When to use stack vs heap

| Want | Use |
|---|---|
| Small short-lived data (≤16 bytes, immutable) | struct → stack |
| Object that shares state | class → heap |
| Buffer for one method | `stackalloc` → stack |
| Buffer used across calls / async | `ArrayPool<T>` → heap pool |
| Local of a lambda's body | depends; static lambda → no allocation |
| Closure (capturing locals) | heap (compiler hoists) |

---

## Summary

- Stack is fast, per-thread, auto-reclaimed; holds locals and value types.
- Heap is GC-managed, shared, slower; holds reference types and boxed values.
- The simple "value type on stack, reference on heap" model is mostly right but has exceptions: fields of classes, boxes, closures, async state machines, and (in .NET 10) escape-analyzed objects.
- Allocation cost is small per object but adds up — minimize on hot paths.
- For performance: prefer structs for small immutable values; use Span and ArrayPool; profile to find real hot spots.

→ Next: [02-GarbageCollection.md](02-GarbageCollection.md)
