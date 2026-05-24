# The .NET Memory Model and `volatile`

> When two threads access the same field without locks, what does each see? The answer involves CPU memory ordering, compiler optimizations, and the .NET memory model. This is the deepest topic in this chapter.

For most code, locks and `Interlocked` make this irrelevant. But knowing the underlying model helps you understand why those primitives are needed and when subtle bugs lurk.

---

## The problem

```csharp
private bool _ready;
private int _data;

// Thread A
_data = 42;
_ready = true;   // signal that data is set

// Thread B
while (!_ready) { }      // wait
Console.WriteLine(_data);  // expect 42 — but might see 0!
```

Possibilities for what B observes:
- `_ready == false` forever (compiler hoisted out of the loop).
- `_ready == true` AND `_data == 42` (the obvious case).
- `_ready == true` BUT `_data == 0` (writes reordered).

Compilers and CPUs can **reorder** reads and writes for performance, as long as the observable behavior on a **single thread** is preserved. Between threads, anything goes.

The fix: synchronize. Either:
- Use `lock` (full memory barrier, semantic guarantee).
- Use `Interlocked` (atomic + memory barrier).
- Mark the field `volatile` (acquire/release semantics).
- Use `Volatile.Read` / `Volatile.Write` (manual barriers).

---

## What `volatile` does

`volatile` is a field modifier that:
- Prevents the compiler / JIT from caching the value in a register across uses.
- Adds memory barriers around reads (acquire) and writes (release).

```csharp
private volatile bool _ready;
private int _data;

// Thread A
_data = 42;
_ready = true;   // release: ensures _data write completes before _ready

// Thread B
while (!_ready) { }    // acquire: each read sees current value; subsequent reads ordered after
Console.WriteLine(_data);   // safe — guaranteed to see 42
```

The semantics:
- **Volatile write** (release): all PRIOR writes complete before this write becomes visible.
- **Volatile read** (acquire): all SUBSEQUENT reads see values consistent with this read.

Pair them: writer writes data, then volatile-writes the flag. Reader volatile-reads the flag, then reads data. Safe.

### Limitations of `volatile`

- Only certain types: `bool`, `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `char`, `float`, `IntPtr`, `UIntPtr`, references. NOT `long`, `double`, `decimal`, `Nullable<T>` of these.
- For atomic 64-bit operations on 32-bit hardware, use `Interlocked`.
- It's a **hint to the compiler**, not a synchronization mechanism. Doesn't make compound operations atomic — `_volatile++` is still NOT atomic.

---

## `Volatile.Read` / `Volatile.Write`

Static methods that apply volatile semantics at a single call site, without marking the field:

```csharp
private int _flag;

// Thread A
_data = 42;
Volatile.Write(ref _flag, 1);     // release barrier

// Thread B
if (Volatile.Read(ref _flag) == 1) {   // acquire barrier
    Console.WriteLine(_data);
}
```

Equivalent to using `volatile` field, but explicit per-use. Useful for `long` fields (where `volatile` is illegal) or when you want the semantic to be visible at the call site.

`Volatile.Read<T>(ref long location)` — supports more types than the `volatile` modifier.

---

## `Interlocked.MemoryBarrier()`

A full barrier: no reads or writes can be reordered across it.

```csharp
Interlocked.MemoryBarrier();
```

Stronger than volatile (which is only half-barriers). Rarely needed directly — locks and Interlocked operations include barriers.

---

## The .NET memory model — informal

For atomic-size types (reference, 32-bit value, etc.):
- **All reads/writes are atomic** — never a torn value.
- **Reads/writes are NOT necessarily ordered** between threads without barriers.

For 64-bit types on 32-bit CPUs:
- Reads/writes are NOT atomic — use `Interlocked.Read` / `Interlocked.Exchange`.

For multi-field state:
- Always synchronize — locks, Interlocked, or explicit barriers.

The official spec is in **ECMA-335** (CLI standard). It's quite weak — meaning hardware (especially ARM, with weaker memory ordering than x64) can reorder more. Code that "works on x64" may fail on ARM.

---

## x86/x64 vs ARM

**x86/x64** has a strong memory model:
- Writes are observed in the order they're issued.
- Reads can be reordered with later writes (StoreLoad).

For most simple producer-consumer patterns, x64 hardware "just works" without explicit barriers — but the compiler/JIT might still reorder. Hence `volatile` and locks remain necessary in source code, even though the hardware mostly cooperates.

**ARM** has a weaker memory model:
- Reads and writes can be reordered freely.
- Without explicit barriers, even producer-consumer patterns break.

This is why code "works on Intel" but breaks on Apple Silicon / ARM64 servers. The .NET memory model abstracts the differences — write portable code using lock/Interlocked/volatile correctly, and it works everywhere.

---

## Double-checked locking — the classic example

```csharp
private SomeService? _instance;
private readonly object _lock = new();

public SomeService Instance {
    get {
        if (_instance is null) {
            lock (_lock) {
                if (_instance is null) _instance = new SomeService();
            }
        }
        return _instance;
    }
}
```

The outer null check avoids taking the lock when the instance is already created. The inner check (inside the lock) handles the race where two threads passed the outer check.

**The subtle bug**: between "create the SomeService" and "assign to _instance," another thread might read _instance and see the non-null reference but an UNINITIALIZED object (constructor not yet visible).

`volatile` on the field would fix this in theory:

```csharp
private volatile SomeService? _instance;
```

Or use `Lazy<T>`:

```csharp
private readonly Lazy<SomeService> _instance = new(() => new SomeService());
public SomeService Instance => _instance.Value;
```

Lazy is the modern idiom. Handles the memory model correctly.

The lesson: double-checked locking without volatile is **broken on weak-memory hardware**. Use Lazy.

---

## When to actually use `volatile`

In practice: rarely. The cases:

1. **Boolean flags for graceful shutdown** between threads.
2. **Spin loops** waiting for a value to change.
3. **Lock-free algorithms** combined with Interlocked.

For everything else: use locks, Interlocked, or `ImmutableX` + atomic swap.

```csharp
// Shutdown flag pattern
private volatile bool _shutdownRequested;

public void RequestShutdown() => _shutdownRequested = true;

public async Task WorkerLoop() {
    while (!_shutdownRequested) {
        await DoWorkAsync();
    }
}
```

Worker reads _shutdownRequested every iteration — volatile prevents the JIT from hoisting it out of the loop (caching the read).

---

## Reordering examples

### Out-of-thin-air read

```csharp
// Thread A
x = 1;

// Thread B
int read = x;   // could be 0 (initial) or 1 — never anything else
```

The .NET memory model forbids "out of thin air" reads — you can't see a value that was never written. So you'll see 0 or 1, not garbage.

### Reordering of two writes

```csharp
// Thread A
a = 1;
b = 2;

// Thread B reading
if (b == 2 && a == 0) { /* possible? */ }
```

**Yes, possible** on weak memory. Thread B might see `b == 2` (the write happened) but `a == 0` (the write hasn't propagated). Use `Volatile.Write` on the publishing write or a lock.

Modern x64: this specific reordering doesn't happen (x64 preserves store-store order). But your code should be correct regardless of hardware.

### Loop hoisting

```csharp
private bool _done;

public void Worker() {
    while (!_done) {   // ⚠ JIT might hoist: bool d = _done; while (!d) {}
        DoWork();
    }
}
```

Without `volatile`, the JIT can hoist the read out of the loop. The loop becomes infinite even if another thread sets _done = true.

Fix:
```csharp
private volatile bool _done;
```

Now each iteration re-reads from memory.

---

## Interlocked vs Volatile vs lock — a comparison

| | Atomicity | Reorder barrier | Cost |
|---|---|---|---|
| Plain field | atomic for ≤32-bit | none — JIT free to reorder | free |
| `volatile` field | atomic for supported types | acquire / release | tiny |
| `Volatile.Read/Write` | atomic | acquire / release | tiny |
| `Interlocked.*` | atomic + RMW | full barrier | ~10-50 ns |
| `lock` | (none directly) | full barrier (Enter/Exit) | ~10-20 ns uncontended |
| `Interlocked.MemoryBarrier` | (none) | full barrier | tiny |

For most code: just use `lock`. The performance difference of volatile vs lock for simple flag patterns is unmeasurable, and locks are easier to reason about.

For lock-free algorithms: Interlocked + volatile, very carefully.

---

## Common bugs

### Using volatile to "make code thread-safe"

```csharp
private volatile int _counter;
public void Increment() => _counter++;   // ⚠ NOT atomic!
```

`_counter++` is read-modify-write. Volatile only ensures memory ordering, not atomicity. Use `Interlocked.Increment(ref _counter)`.

### Believing x86 = x64 behavior

```csharp
// Tested fine on Intel, ships, breaks on Apple Silicon production
```

Always synchronize. Don't assume the hardware will save you.

### Volatile on long/double

```csharp
private volatile long _bigCounter;   // ❌ compile error
```

Use `Interlocked.Read/Exchange/Add` instead.

### Skipping locks "because writes are atomic"

```csharp
private int _x, _y;

public (int x, int y) Snapshot() => (_x, _y);   // ⚠ not atomic together
```

Two reads, not one. Another thread might update _x and _y between them. The snapshot can show "torn" state. Lock to read both consistently.

---

## When you actually need to think about the memory model

You're writing:
- Lock-free data structures (Treiber stack, MPMC queue).
- High-throughput counters across cores.
- Wait-free coordination primitives.

You're NOT writing:
- Regular business logic.
- Async I/O code.
- Most concurrent server work.

For everyday code, follow the boring rules:
1. Use `lock` for multi-field mutations.
2. Use `Interlocked` for single-variable atomics.
3. Use immutable + atomic swap for snapshot publish.
4. Use concurrent collections.
5. Use `Channel<T>` for producer-consumer.

The memory model becomes relevant when you go off this path.

---

## A concrete pattern that requires understanding

Reading shared state without locks:

```csharp
private ImmutableDictionary<int, string> _state = ImmutableDictionary<int, string>.Empty;

public string? Get(int key) {
    var state = _state;   // single read of the reference
    return state.TryGetValue(key, out var v) ? v : null;
}

public void Update(int key, string value) {
    ImmutableDictionary<int, string> current, updated;
    do {
        current = _state;
        updated = current.SetItem(key, value);
    } while (Interlocked.CompareExchange(ref _state, updated, current) != current);
}
```

The reader does a **single read** of `_state` into a local. The reference assignment is atomic (since references are pointer-sized). The local is a stable snapshot for the rest of the method.

The writer uses CAS to publish a new version. Interlocked.CompareExchange has full memory barriers, so all writes to the dictionary (which is immutable, so this is trivial) are visible to readers that see the new reference.

This pattern is **memory-model correct** on all platforms. No volatile needed because the writer's barrier ensures visibility.

---

## Internals — what the JIT generates

For a regular field read:
```il
ldfld _field
```

The JIT may optimize this into a register cache across multiple uses.

For volatile:
```il
ldfld _field
volatile.   // prefix or special opcode in newer CLR
```

The JIT preserves the read each time — no caching.

For writes:
```il
volatile.
stfld _field
```

Memory barrier emitted (`mfence` or weaker, depending on architecture).

---

## A practical mental model

**For 90%+ of concurrent code**:
- Use `lock` (or `Lock` in C# 13+) for multi-field state.
- Use `Interlocked` for atomic counters / flags.
- Use immutable + atomic swap for snapshots.
- Use `ConcurrentDictionary` for shared maps.
- Use `Channel<T>` for producer-consumer.

For the remaining 10%:
- Use `volatile` for the simplest reader/writer flag patterns.
- Use `Volatile.Read/Write` when you want barriers at specific call sites.
- Read papers (and the source code of `ConcurrentDictionary`) before writing your own lock-free data structures.

---

## Summary

- The .NET memory model allows reordering of reads/writes between threads without barriers.
- `lock` and `Interlocked` provide full barriers — safe by default.
- `volatile` provides acquire/release semantics — useful for flag patterns.
- For 64-bit on 32-bit hardware, use `Interlocked` (volatile doesn't support long).
- For ARM portability, never rely on "Intel just works."
- Most application code never needs explicit memory model awareness — use locks and high-level primitives.

→ Next: [17-CommonAsyncBugs.md](17-CommonAsyncBugs.md)
