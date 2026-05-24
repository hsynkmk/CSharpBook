# Interlocked — Atomic Operations

## What it is

`System.Threading.Interlocked` provides **atomic** read-modify-write operations on single variables. They execute as a single CPU instruction (or with a CAS loop), so they're thread-safe without locks.

```csharp
private int _counter;

public void Increment() => Interlocked.Increment(ref _counter);
public int Read() => Interlocked.Read(ref _counter);
```

Atomic = no other thread can see a partial update. For pure counters and flags, Interlocked is dramatically faster than `lock`. It's also the building block for lock-free algorithms via `CompareExchange`.

---

## Why it exists

A naive counter:

```csharp
private int _counter;
public void Bump() => _counter++;
```

`_counter++` is **not atomic** — it's three operations (load, add, store). Two threads racing each other can lose increments:

```
Thread A: load 5
Thread B: load 5
Thread A: store 6
Thread B: store 6     // lost A's increment!
```

Options:
1. `lock` — works but expensive for a single int update.
2. `Interlocked.Increment` — one CPU instruction, atomic, lock-free.

Interlocked is the right tool for single-variable atomicity.

---

## The basic operations

### Increment / Decrement / Add

```csharp
int newValue = Interlocked.Increment(ref _counter);   // _counter += 1, returns new value
int newValue2 = Interlocked.Decrement(ref _counter);  // _counter -= 1
int newValue3 = Interlocked.Add(ref _counter, 5);      // _counter += 5
int newValue4 = Interlocked.Add(ref _counter, -5);     // _counter -= 5
```

All atomic. Return the **new** value.

For `long`: `Interlocked.Increment(ref _longField)` works the same. On 32-bit hardware, even reading a 64-bit field isn't atomic without Interlocked.

### Read / Exchange

```csharp
long current = Interlocked.Read(ref _longField);    // atomic read of long on 32-bit
int prev = Interlocked.Exchange(ref _field, newValue);   // atomic set-and-return-old
```

`Read` matters mainly for `long` on 32-bit (on 64-bit, plain reads of long are already atomic).

`Exchange` atomically sets a new value and returns the previous one. Useful for "claim this slot" patterns.

---

## CompareExchange — the foundation of lock-free

```csharp
int original = Interlocked.CompareExchange(ref _field, newValue, expectedValue);
```

Atomically:
1. If `_field == expectedValue`, set `_field = newValue`.
2. Return the original value of `_field`.

The classic CAS (Compare-And-Swap) operation. Almost all lock-free algorithms are built on top.

The **CAS loop pattern**:

```csharp
public int IncrementBy(int x) {
    int current;
    int next;
    do {
        current = _counter;
        next = current + x;
    } while (Interlocked.CompareExchange(ref _counter, next, current) != current);
    return next;
}
```

"Read; compute new value; try to swap; if someone else changed it, retry." This is **lock-free** — no thread waits on a lock; threads make progress as long as the system makes progress.

For pure `Add`, just use `Interlocked.Add` — but the CAS loop pattern generalizes to anything (e.g., "set if greater than"):

```csharp
public bool SetIfGreater(int x) {
    int current;
    do {
        current = _max;
        if (x <= current) return false;
    } while (Interlocked.CompareExchange(ref _max, x, current) != current);
    return true;
}
```

---

## Atomic operations on references

```csharp
Interlocked.Exchange(ref _settings, newSettings);
Interlocked.CompareExchange(ref _settings, newSettings, oldSettings);
```

For reference-type fields, atomic operations replace the pointer atomically. The old object isn't modified — you publish a new instance.

Classic pattern: **immutable snapshot with atomic swap**:

```csharp
private ImmutableDictionary<int, string> _config = ImmutableDictionary<int, string>.Empty;

public void Update(int key, string value) {
    ImmutableDictionary<int, string> current, updated;
    do {
        current = _config;
        updated = current.SetItem(key, value);
    } while (Interlocked.CompareExchange(ref _config, updated, current) != current);
}

public string? Get(int key) => _config.TryGetValue(key, out var v) ? v : null;
```

Reads are **lock-free** — just read the immutable dictionary reference and use it. Writes do a CAS loop with the updated copy.

For a "single-writer" scenario (just one thread updates), the loop usually iterates once. For multi-writer with contention, retries can pile up — but never deadlock.

Modern API: `ImmutableInterlocked.Update`:

```csharp
ImmutableInterlocked.Update(ref _config, c => c.SetItem(key, value));
```

Same pattern, packaged.

---

## Volatile vs Interlocked

Both relate to memory visibility:

- **`volatile`** — a field modifier. Reads/writes have specific memory semantics (acquire-release), preventing reordering.
- **`Interlocked`** — atomic operations with full memory barriers.

For an int counter:
```csharp
volatile int _flag;            // simple flag; reads/writes ordered
private int _counter;
Interlocked.Increment(ref _counter);   // for read-modify-write
```

Simple field reads/writes (just storing a single int) are atomic on aligned 32-bit (and 64-bit on 64-bit CPUs). `volatile` ensures memory ordering. Interlocked is needed when the operation is **compound** (read-modify-write).

See [§16 MemoryModelVolatile](16-MemoryModelVolatile.md) for the deep version.

---

## Common patterns

### Lock-free counter

```csharp
private long _total;

public long Add(long x) => Interlocked.Add(ref _total, x);
public long Current => Interlocked.Read(ref _total);
```

For pure counters, dramatically faster than `lock`. Scales well under contention (modern CPUs have hardware atomic add).

### Lock-free max

```csharp
public void TrackMax(int x) {
    int current;
    do {
        current = _max;
        if (x <= current) return;
    } while (Interlocked.CompareExchange(ref _max, x, current) != current);
}
```

Set `_max` only if `x` is greater. CAS retries if another thread interferes.

### Lock-free flag (one-shot)

```csharp
private int _initialized;

public void EnsureInit() {
    if (Interlocked.CompareExchange(ref _initialized, 1, 0) == 0) {
        // we won the race; do initialization
        DoInit();
    }
    // else: someone else is doing it (or did)
}
```

For "exactly once" initialization, CAS the flag from 0 to 1. The winner does the work. Others observe the flag is non-zero.

For waiting on the init to complete: combine with `Lazy<T>` or a `ManualResetEventSlim`.

### Lock-free stack (Treiber stack)

```csharp
public class LockFreeStack<T> {
    private Node? _head;
    private class Node { public T Value; public Node? Next; public Node(T v) { Value = v; } }

    public void Push(T value) {
        var newNode = new Node(value);
        Node? oldHead;
        do {
            oldHead = _head;
            newNode.Next = oldHead;
        } while (Interlocked.CompareExchange(ref _head, newNode, oldHead) != oldHead);
    }

    public bool TryPop(out T value) {
        Node? oldHead;
        Node? newHead;
        do {
            oldHead = _head;
            if (oldHead is null) { value = default!; return false; }
            newHead = oldHead.Next;
        } while (Interlocked.CompareExchange(ref _head, newHead, oldHead) != oldHead);
        value = oldHead.Value;
        return true;
    }
}
```

Push: read head; set new node's Next to head; CAS new node as head. Pop: read head; CAS its Next as the new head.

Note: this stack has the **ABA problem** in theory (a head pointer could be popped and re-pushed between reads). For `Node` references in .NET, GC ensures the popped node isn't reused → ABA is mostly avoided. But in unmanaged contexts, you'd need a generation counter or hazard pointers.

For real code, use `ConcurrentStack<T>` from the BCL — already implemented correctly.

---

## When Interlocked is the right tool

✓ Atomic counters, flags.
✓ Atomic swap of a reference (publish a new immutable snapshot).
✓ Lock-free algorithms (CAS loops).
✓ Single-variable read-modify-write.

✗ Compound state involving multiple fields — use `lock`.
✗ Long critical sections — `lock` is clearer.
✗ Operations that need to be one logical step (e.g., transfer balance from A to B) — needs a lock.

---

## Internals — what makes it atomic

`Interlocked.Increment(ref x)` compiles to a single CPU instruction with the LOCK prefix (on x86/x64):

```asm
lock add dword ptr [rcx], 1
```

The LOCK prefix tells the CPU to make this operation atomic across all cores — typically by holding the cache line for the duration.

`CompareExchange` is the `CMPXCHG` instruction with LOCK:

```asm
lock cmpxchg dword ptr [rcx], edx
```

The CPU reads the value, compares with the "expected" register, conditionally writes the "new" register, and returns the original value — all atomically.

These are **fast** — ~10-50 cycles depending on cache contention. Vastly faster than a kernel lock.

### Memory barriers

Interlocked operations have **full memory barriers** — no reads/writes can be reordered across them. This is what makes them safe for synchronization beyond just the atomic operation.

For just memory ordering without atomic compute: `Volatile.Read` / `Volatile.Write` / `Interlocked.MemoryBarrier`.

---

## Common bugs

### Reading without Interlocked.Read on 32-bit

```csharp
private long _value;
long x = _value;   // ⚠ on 32-bit: torn read possible
```

On 32-bit hardware, reading a 64-bit field is two operations — another thread can update between. Use `Interlocked.Read`. On 64-bit hardware, this is a non-issue (the read is atomic).

For maximum portability, use `Interlocked.Read` or mark the field `volatile` (limits to int / IntPtr / reference types, not long).

### Read-modify-write with raw operators

```csharp
_counter++;   // ⚠ not atomic
```

Use `Interlocked.Increment(ref _counter)`. Even if the field is volatile, `++` is still three operations.

### Wrong expected value in CompareExchange

```csharp
Interlocked.CompareExchange(ref _x, newValue, 0);   // only updates if _x is 0
```

Make sure the expected value matches what you actually want to overwrite. Without a loop, this is a one-shot "if matches, update" — useful for one-time flags.

### Forgetting CAS-loop retry

```csharp
int current = _x;
Interlocked.CompareExchange(ref _x, current + 1, current);   // might fail silently
```

CompareExchange returns the original value; check whether it matched expected:
```csharp
int current;
do {
    current = _x;
} while (Interlocked.CompareExchange(ref _x, current + 1, current) != current);
```

For pure increment, just `Interlocked.Increment`.

### ABA problem (rare but real)

In unmanaged or specially-designed code, a value can change from A to B and back to A between your read and CAS — you wouldn't notice. In .NET with GC, references usually avoid this because the GC ensures freed objects aren't immediately reused as new objects with the same address. But for raw value swaps, ABA can bite.

Solution: use a version counter alongside the value, or hazard pointers, or a different design.

---

## Performance

| Operation | Approximate cost |
|---|---|
| `Interlocked.Increment` | ~10-50 ns |
| `Interlocked.CompareExchange` | ~10-50 ns |
| `lock (obj)` uncontended | ~10-20 ns |
| `lock (obj)` contended | μs+ |
| Plain int read/write | ~1 ns |

Interlocked is **slower than plain operations** (atomic instructions are CPU-expensive) but **dramatically faster than locks** under contention.

For a hot counter, Interlocked scales linearly with cores (until cache-line contention starts to bottleneck). Locks under contention serialize threads → throughput collapses.

### False sharing

Multiple threads touching adjacent atomic counters on the same cache line cause **false sharing** — the cache line bounces between cores, even though logically the operations are independent.

```csharp
class Counters {
    public long A;
    public long B;
}
```

Threads incrementing A and B contend on the same 64-byte cache line. Use `[StructLayout]` padding or `[Cache.PadCacheLine]` (or just space them out in separate objects) to avoid.

For most code, false sharing isn't a concern. For high-perf concurrent counters, profile.

---

## When to use Interlocked vs lock

| Need | Use |
|---|---|
| Atomic single-field increment | `Interlocked.Increment` |
| Atomic single-field swap | `Interlocked.Exchange` |
| Compare-and-set | `Interlocked.CompareExchange` |
| Lock-free algorithm building block | `Interlocked.CompareExchange` in a CAS loop |
| Atomic snapshot publish | `Interlocked.Exchange` (reference) |
| Multi-field invariant | `lock` |
| Long critical section | `lock` |
| Across processes | `Mutex` |

---

## Summary

- Use `Interlocked.*` for atomic operations on single variables.
- `CompareExchange` is the lock-free building block.
- Faster than locks under contention.
- Doesn't replace locks for multi-field invariants.
- Combine with immutable data structures for elegant lock-free publish-subscribe.

→ Next: [12-ConcurrentCollections.md](12-ConcurrentCollections.md)
