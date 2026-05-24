# 11 — Synchronization & Memory Model ⭐⭐ (high-frequency)

## ⚡ 30-second answer

When multiple threads share **mutable** state, you need synchronization to prevent **race conditions**. The workhorse is **`lock`** (sugar over `Monitor`) for mutual exclusion of a critical section. For simple atomic numeric ops use **`Interlocked`** (lock-free, faster). For *visibility* of a flag across threads use **`volatile`** (prevents the compiler/CPU from caching/reordering reads). For async or cross-process you need **`SemaphoreSlim`** (async-friendly) or **`Mutex`** (cross-process). The two killers to discuss: **race conditions** (unsynchronized shared mutation) and **deadlocks** (circular lock waits).

---

## Core mechanics

**`lock`** — mutual exclusion; only one thread in the block at a time:
```csharp
private readonly object _gate = new();   // dedicated private lock object
lock (_gate) { _balance += amount; }      // critical section
```
`lock(x)` ≡ `Monitor.Enter/Exit(x)` with try/finally. (C# 13: a dedicated `System.Threading.Lock` type is faster and clearer.)

**`Interlocked`** — atomic, lock-free single operations:
```csharp
Interlocked.Increment(ref _counter);
Interlocked.Add(ref _total, n);
Interlocked.CompareExchange(ref _state, newVal, expected);   // CAS — the basis of lock-free
```

**`volatile`** — visibility/ordering for a single field (no caching, no reordering around it):
```csharp
private volatile bool _stop;     // one thread sets it, another sees it promptly
```
`volatile` does **not** make compound operations atomic (`_x++` is still a race).

**Async-friendly / specialized**:
```csharp
await _semaphore.WaitAsync(ct); try { … } finally { _semaphore.Release(); }  // async "lock"/throttle
using var mutex = new Mutex(false, "Global\\MyApp");                          // cross-process
var rw = new ReaderWriterLockSlim();   // many readers OR one writer
```

**Memory model**: without barriers, the CPU/JIT may **reorder** reads/writes and cache values in registers, so one thread may not see another's update. `lock`, `Interlocked`, and `volatile` insert the necessary **memory barriers**.

---

## Comparison tables

| Primitive | Scope | Async? | Use |
|---|---|---|---|
| `lock` / `Monitor` | in-process | **no** (can't `await` inside) | guard a critical section |
| `Interlocked` | in-process | n/a | atomic counter / CAS, lock-free |
| `volatile` | in-process | n/a | visibility of a single flag |
| `SemaphoreSlim` | in-process | **yes** (`WaitAsync`) | async lock, throttle to N |
| `Mutex` | **cross-process** | no | single-instance app, machine-wide lock |
| `ReaderWriterLockSlim` | in-process | no | many readers, rare writers |

| Problem | What it is | Fix |
|---|---|---|
| **Race condition** | unsynchronized shared mutation → torn/lost updates | lock / Interlocked / immutability |
| **Deadlock** | threads wait on each other's locks in a cycle | consistent lock ordering, timeouts, fewer locks |
| **Livelock** | threads keep reacting, no progress | backoff/jitter |
| **Starvation** | a thread never gets the lock/CPU | fair locks, avoid long critical sections |

---

## 🪤 Traps & gotchas

- **`lock` on the wrong object**: `lock(this)` or `lock(typeof(X))` or a public/string object lets *outside* code lock the same monitor → accidental deadlock. Use a **private readonly object**.
- **Can't `await` inside `lock`**: the monitor is thread-affine; awaiting could resume on a different thread → can't release. Use **`SemaphoreSlim.WaitAsync`** for an async critical section.
- **`volatile` ≠ atomic**: `volatile int x; x++;` is still a race (read-modify-write). Use `Interlocked.Increment`.
- **Deadlock by lock ordering**: thread A locks X then Y; thread B locks Y then X → cycle. Always acquire multiple locks in a **global consistent order** (or use a single lock).
- **Forgetting to release**: always `lock`/`try-finally`; for `SemaphoreSlim`, `Release()` in `finally` (a throw without release leaks a permit).
- **Double-checked locking** done wrong (without proper barriers) is broken — prefer `Lazy<T>` for thread-safe lazy init.
- **Over-locking** kills throughput (serializes everything); **under-locking** corrupts data. Lock the smallest correct region.
- **Reading a non-volatile flag in a spin loop** may never see the update (cached in a register) — use `volatile` or `Volatile.Read`.

---

## ❓ Likely questions

**Q: `lock` vs `Interlocked` vs `volatile` — when each?**
A: `lock` for a multi-statement critical section (mutual exclusion). `Interlocked` for a single atomic numeric op (faster, lock-free). `volatile` for visibility/ordering of one flag — it doesn't provide atomicity for compound ops.

**Q: What's a race condition?**
A: Two+ threads access shared mutable state without synchronization and at least one writes, so the result depends on timing — lost updates, torn reads. Fix with locking, Interlocked, or immutability.

**Q: What's a deadlock and how do you prevent it?**
A: Threads each hold a lock the other needs, in a cycle, so none proceed. Prevent with consistent lock ordering, fewer/coarser locks, lock timeouts (`Monitor.TryEnter`), or lock-free designs.

**Q: Why can't you `await` inside a `lock`?**
A: `Monitor` is thread-affine — it must be released by the acquiring thread, but `await` may resume on a different thread. Use `SemaphoreSlim.WaitAsync()` instead.

**Q: What does `lock` actually compile to?**
A: `Monitor.Enter(obj)` in a try, `Monitor.Exit(obj)` in finally — mutual exclusion plus memory barriers.

**Q: Is `volatile bool` enough for a counter?**
A: No — increment is read-modify-write (not atomic). Use `Interlocked.Increment`. `volatile` only ensures the latest value is read, not that compound ops are atomic.

**Q: What object should you lock on?**
A: A private, readonly, dedicated object (`private readonly object _gate = new();`). Never `this`, a `Type`, or a public/interned object.

**Q: SemaphoreSlim vs Mutex?**
A: `SemaphoreSlim` is in-process, async-capable, allows N concurrent holders (throttling). `Mutex` is OS-level/cross-process (e.g., single-instance app), heavier, no async.

---

## 🎓 Senior Extra

- **CAS & lock-free**: `Interlocked.CompareExchange` is the building block — read a value, compute a new one, swap only if unchanged, else retry (a CAS loop). Powers `ConcurrentQueue`, lazy init, lock-free counters. Beware the **ABA problem**.
- **Memory barriers / fences**: `lock`/`Interlocked` imply full barriers; `Volatile.Read`/`Write` give acquire/release semantics. The .NET memory model is *stronger* than the ECMA minimum on x86 (strongly-ordered) but **ARM is weakly-ordered** — code that "works" on x64 can break on ARM without proper volatile/barriers.
- **`Lazy<T>`** with `LazyThreadSafetyMode` is the correct, simple thread-safe lazy init (avoids hand-rolled double-checked locking).
- **`ReaderWriterLockSlim`** helps only when reads vastly outnumber writes and the critical section is non-trivial; otherwise a plain `lock` is simpler and often faster. Avoid recursion unless configured.
- **Lock contention** is measurable (`dotnet-counters` "Monitor Lock Contention Count" — [21](21-Deployment-Perf-Tooling.md)); high contention means rethink granularity (sharding, lock-free, immutable snapshots, `ConcurrentDictionary`).
- **Immutability as a strategy**: the easiest "synchronization" is no shared mutable state — pass immutable data / use `Immutable*` collections / per-thread state, eliminating locks.
- **Thread-affine reentrancy**: `Monitor`/`lock` is reentrant (same thread can re-enter); `SemaphoreSlim` is **not** (re-acquiring on the same logical flow deadlocks).
- **`SpinLock`/`SpinWait`** for ultra-short critical sections to avoid context-switch cost — niche; measure.

→ Deeper: [`../CSharpBook/08-Concurrency/09-LockingPrimitives.md`](../CSharpBook/08-Concurrency/README.md), [`../CSharpBook/08-Concurrency/16-MemoryModelVolatile.md`](../CSharpBook/08-Concurrency/README.md)
