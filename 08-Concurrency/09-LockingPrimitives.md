# Locking Primitives — `lock`, `Monitor`, `Mutex`, `ReaderWriterLockSlim`

## What it is

When multiple threads access shared state, you need **mutual exclusion** — only one (or in some cases, a controlled set) can be inside a critical section at a time. The .NET toolbox:

- **`lock`** — the C# keyword, syntactic sugar over `Monitor.Enter/Exit`. Default choice for most cases.
- **`Monitor`** — the underlying API. Use when you need `TryEnter` with timeout.
- **`Mutex`** — kernel-level lock, slow, but works **across processes**.
- **`ReaderWriterLockSlim`** — many readers OR one writer.
- **`SpinLock`** — short-spin lock, niche perf scenarios.
- **`System.Threading.Lock`** (C# 13 / .NET 9+) — a dedicated lock type that's slightly faster than `lock(object)`.

```csharp
private readonly object _gate = new();
private int _count;

public void Increment() {
    lock (_gate) {
        _count++;
    }
}
```

Locking is **the** fundamental synchronization primitive — the building block for safe shared state.

---

## Why locking exists

Without synchronization, two threads racing on the same data produces undefined behavior:

```csharp
int counter = 0;
Parallel.For(0, 1_000_000, _ => counter++);
Console.WriteLine(counter);   // ⚠ usually < 1,000,000
```

`counter++` is **three** operations: read, add, write. Two threads can both read the same value, both add 1, both write back — losing one increment. Race condition.

Locking serializes access:

```csharp
int counter = 0;
object gate = new();
Parallel.For(0, 1_000_000, _ => { lock (gate) counter++; });
Console.WriteLine(counter);   // 1,000,000
```

For pure increment, `Interlocked.Increment(ref counter)` is faster (lock-free atomic) — see [§11](11-Interlocked.md). But for compound state, locking is the answer.

---

## `lock` statement

```csharp
private readonly object _gate = new();

public void M() {
    lock (_gate) {
        // critical section
    }
}
```

Equivalent to:

```csharp
bool taken = false;
try {
    Monitor.Enter(_gate, ref taken);
    // critical section
} finally {
    if (taken) Monitor.Exit(_gate);
}
```

`lock` ensures the lock is released even if the body throws.

### Rules

- **Lock object MUST be a reference type**. Locking a value type boxes it → each box is a different object → no mutual exclusion. The compiler warns.
- **Lock object should be `readonly`** — reassigning it breaks the lock.
- **Lock object should be `private`** — public locks can be grabbed by external code, causing surprise deadlocks.
- **Prefer a dedicated lock object** (`new object()`) over locking `this` or a type. Locking `this` lets callers grab the lock from outside.

```csharp
public class Counter {
    private readonly object _gate = new();   // ✓ dedicated, private, readonly
    private int _count;

    public void Increment() {
        lock (_gate) _count++;
    }
}
```

### Reentrant

`lock` is **reentrant** — the same thread can take it multiple times:

```csharp
lock (_gate) {
    lock (_gate) {   // OK — same thread can re-enter
        // ...
    }
}
```

Internally, Monitor tracks the count. Each `Enter` increments; each `Exit` decrements. The lock is released when the count reaches zero.

Used to call from one locked method into another that also locks — saves you from "is this called locked or not?" complications.

---

## `System.Threading.Lock` (C# 13+)

C# 13 (.NET 9) added a dedicated `System.Threading.Lock` type:

```csharp
private readonly Lock _gate = new();

public void Increment() {
    lock (_gate) {
        _count++;
    }
}
```

Why a new type?
- **Slightly faster** — no per-object lock-word indirection.
- **Type-safe** — you can't accidentally `lock` a value-type instance.
- **Clearer intent** — the field's purpose is obvious from its type.

The compiler recognizes `Lock`-typed expressions and uses optimized lock primitives. For new code targeting .NET 9+, prefer `Lock` over `object`.

Old `lock(someObject)` still works — backward compatible.

---

## `Monitor` — the underlying API

`lock` is sugar for `Monitor`. Use the API directly for advanced scenarios:

```csharp
bool taken = false;
try {
    Monitor.Enter(_gate, ref taken);
    // ...
} finally {
    if (taken) Monitor.Exit(_gate);
}

// TryEnter — non-blocking
if (Monitor.TryEnter(_gate, TimeSpan.FromMilliseconds(100))) {
    try { /* ... */ }
    finally { Monitor.Exit(_gate); }
} else {
    // couldn't get the lock in time
}
```

`Monitor.Wait` / `Monitor.Pulse` / `Monitor.PulseAll` implement condition-variable semantics — release the lock while waiting, reacquire on wake. Used in classic producer-consumer patterns. We'll see them in [§13 Channels](13-Channels.md) (the modern replacement).

---

## `lock` and async — DON'T

```csharp
public async Task BadAsync() {
    lock (_gate) {
        await Task.Delay(100);   // ❌ compile error
    }
}
```

You can't `await` inside a `lock` block. The reason: `lock` is tied to the OS thread (via Monitor.Enter). After `await`, the continuation may run on a different thread — which doesn't own the lock — and `Monitor.Exit` would throw.

For async-safe mutual exclusion, use `SemaphoreSlim`:

```csharp
private readonly SemaphoreSlim _lock = new(1, 1);

public async Task M() {
    await _lock.WaitAsync();
    try {
        await DoWorkAsync();   // OK — semaphore not thread-bound
    } finally {
        _lock.Release();
    }
}
```

[§10 Semaphores](10-Semaphores.md) goes deep.

---

## Mutex — cross-process locking

`Mutex` is a kernel-level lock. Much slower than `lock` (involves OS calls), but it can be **named** and shared across processes:

```csharp
using var mutex = new Mutex(false, "MyApp_SingletonGuard");
if (!mutex.WaitOne(TimeSpan.FromSeconds(1))) {
    Console.WriteLine("Another instance is running.");
    return;
}
try {
    // do work
} finally {
    mutex.ReleaseMutex();
}
```

Used for:
- Single-instance applications (only one copy of the app running).
- File locks across processes.
- Coordinating with non-.NET processes.

For intra-process locking, use `lock` (1000× faster).

---

## `ReaderWriterLockSlim` — many readers, one writer

If your shared state is **read-heavy** (many reads, few writes), `ReaderWriterLockSlim` lets multiple readers proceed in parallel while writers exclude everyone:

```csharp
private readonly ReaderWriterLockSlim _rwLock = new();
private readonly Dictionary<int, string> _cache = new();

public string? Get(int key) {
    _rwLock.EnterReadLock();
    try {
        return _cache.TryGetValue(key, out var v) ? v : null;
    } finally {
        _rwLock.ExitReadLock();
    }
}

public void Set(int key, string value) {
    _rwLock.EnterWriteLock();
    try {
        _cache[key] = value;
    } finally {
        _rwLock.ExitWriteLock();
    }
}
```

Multiple readers can hold the read lock simultaneously. A writer takes the write lock alone, blocking all readers and other writers.

### When it's worth it

- Read-mostly workloads (90%+ reads).
- Long-held read locks (real work inside the lock).

For short locks with mixed access, plain `lock` is often faster — RWLockSlim has more overhead per operation, and most readers don't benefit if the operations are short.

For **read-only** snapshots, prefer immutable collections + atomic swap — no lock at all:

```csharp
private ImmutableDictionary<int, string> _cache = ImmutableDictionary<int, string>.Empty;

public string? Get(int key) =>
    _cache.TryGetValue(key, out var v) ? v : null;   // lock-free read

public void Set(int key, string value) =>
    ImmutableInterlocked.Update(ref _cache, c => c.SetItem(key, value));
```

Readers never lock. Writers use a CAS loop. See [§09 ImmutableCollections](../07-Collections/09-ImmutableCollections.md).

---

## `SpinLock` — niche

A lock that **spins** (busy-waits) briefly before blocking — useful when the lock is held for nanoseconds:

```csharp
private SpinLock _spin = new(false);   // false = no thread tracking, faster

bool taken = false;
try {
    _spin.Enter(ref taken);
    // very brief critical section
} finally {
    if (taken) _spin.Exit();
}
```

Use cases:
- High-frequency micro-operations where blocking + waking up is more expensive than spinning.
- Lock-free algorithms where SpinLock is part of a CAS loop.

For most code, `lock` is the right answer. SpinLock is a sharp tool for measured hot paths.

---

## Deadlock — the classic two-lock problem

```csharp
private readonly object _a = new();
private readonly object _b = new();

// Thread 1
lock (_a) { Thread.Sleep(10); lock (_b) { ... } }

// Thread 2
lock (_b) { Thread.Sleep(10); lock (_a) { ... } }
```

Thread 1 holds A, waits for B. Thread 2 holds B, waits for A. Forever stuck.

**Prevention**:
- **Always acquire locks in the same global order.** If A always comes before B in your codebase, deadlock impossible.
- **Avoid holding multiple locks** — design to need only one at a time.
- **Don't call out to unknown code while holding a lock** (events, virtual methods, callbacks). The callee might lock and call back into you, causing reentrance issues or deadlock.

---

## Lock granularity

Too coarse:
```csharp
lock (_gate) {
    Process(data1);
    Process(data2);   // unrelated work also serialized
}
```

Too fine:
```csharp
lock (_gate) { _count++; }
lock (_gate) { _total += x; }   // two acquisitions for related work
```

The right grain is one lock per **invariant** — protect the related fields together, but don't bundle unrelated state.

---

## Lock convoy — performance pitfall

When many threads contend on the same lock, they queue up — even if each only needs the lock briefly. The kernel schedules each in turn, often with context switches between.

Symptoms:
- High CPU but low throughput.
- "Lock contention" on profiler reports.

Fixes:
- Reduce time inside the lock.
- Partition the lock (shard the data — one lock per shard).
- Replace with lock-free or atomic operations.
- Use `ConcurrentDictionary` (lock per bucket).

---

## Don't lock on...

### Public state

```csharp
public class Bad {
    public readonly object Lock = new();   // public — external code can grab it
}

// Outside:
lock (myBad.Lock) { ... }   // hijacks the lock
```

### `this`

```csharp
public void M() {
    lock (this) { ... }   // external code might also lock(myInstance) → conflict
}
```

### Type objects

```csharp
lock (typeof(MyClass)) { ... }   // Type objects are AppDomain-wide
```

### Strings

```csharp
lock ("my-key") { ... }   // strings are interned — different code locking on the same literal collides
```

**Always use a dedicated private readonly object.** Or, in .NET 9+, a `Lock` field.

---

## Internals — Monitor's lock word

Every object on the heap has a **sync block** field in its header (8 bytes on 64-bit, the slot before the MT pointer).

For uncontended locks, the runtime uses **thin locks** — the sync block holds the lock state directly:
- Owner thread ID.
- Recursion count.

When contention happens, the runtime "inflates" the sync block to a full lock structure with a wait queue. After contention subsides, it can be deflated back.

`lock (obj)` is therefore essentially **free for uncontended cases** — a few atomic compare-exchange instructions. Contention is where the cost grows.

### `Lock` type (C# 13+) optimization

The dedicated `System.Threading.Lock` type stores its state directly in fields, no sync-block lookup. Avoids the indirection. Slightly faster for tight critical sections.

For ultra-hot lock paths, this small win can matter. For everyday code, the difference is negligible.

---

## Common patterns

### Lazy initialization with lock

```csharp
private object? _instance;
private readonly object _lock = new();

public object Instance {
    get {
        if (_instance is null) {
            lock (_lock) {
                _instance ??= CreateExpensive();
            }
        }
        return _instance;
    }
}
```

Double-checked locking. Volatile read for the first check is a classic concern; modern .NET memory model makes it safe with the field marked `volatile` or wrapped via `Volatile.Read`. Or use `Lazy<T>`:

```csharp
private readonly Lazy<object> _instance = new(() => CreateExpensive());
public object Instance => _instance.Value;
```

`Lazy<T>` handles concurrency correctly.

### State machine with multiple fields

```csharp
public class BankAccount {
    private readonly object _lock = new();
    private decimal _balance;
    private List<Transaction> _history = new();

    public void Deposit(decimal amount, Transaction t) {
        lock (_lock) {
            _balance += amount;
            _history.Add(t);
        }
    }

    public (decimal balance, int count) Snapshot() {
        lock (_lock) {
            return (_balance, _history.Count);
        }
    }
}
```

One lock guards multiple correlated fields. Snapshot returns both consistently.

### TryEnter pattern

```csharp
public bool TryProcess() {
    if (!Monitor.TryEnter(_lock, TimeSpan.FromMilliseconds(100))) {
        return false;   // someone else has it; skip
    }
    try {
        // ...
        return true;
    } finally {
        Monitor.Exit(_lock);
    }
}
```

"Try to acquire; give up if busy" — useful for non-critical work.

---

## Common bugs

### Locking a different object accidentally

```csharp
private object _lock = new();

public void M() {
    lock (_lock) { /* ... */ }
    _lock = new();   // ⚠ — now subsequent locks use a different object!
}
```

Make `_lock` readonly to prevent this.

### Forgetting `Monitor.Exit` on TryEnter

```csharp
if (Monitor.TryEnter(_lock)) {
    // ... no try/finally
    Monitor.Exit(_lock);   // if anything throws above, exit is skipped
}
```

Always wrap in try/finally with TryEnter.

### Async inside lock

Already covered — compile error. Compiler protects you here.

### Lock convoy from unprotected hot field

```csharp
public class Counter {
    private readonly object _lock = new();
    public int Count {
        get { lock (_lock) return _count; }
        set { lock (_lock) _count = value; }
    }
    private int _count;
}
```

For pure int reads/writes, prefer `Interlocked.CompareExchange` / volatile fields. Locks here are overkill and prone to contention.

### Lock in a finalizer

```csharp
~Foo() {
    lock (_gate) { /* ... */ }   // ⚠ — finalizer runs on the finalizer thread
}
```

Finalizers run on a dedicated thread. They can deadlock if the lock is held by a different finalizer. Generally, finalizers should NOT lock.

---

## Performance summary

| Primitive | Uncontended cost | Contended cost | Use when |
|---|---|---|---|
| `lock (obj)` | ~10-20 ns | depends on contention | Default mutual exclusion |
| `Lock` (C# 13+) | ~5-10 ns | similar | Best for new code |
| `Monitor.TryEnter` | ~10-20 ns | timeout | Avoid blocking |
| `Mutex` | ~1000+ ns | slow | Cross-process |
| `ReaderWriterLockSlim` | ~30-50 ns per read | lower contention | Read-mostly |
| `SpinLock` | ~5-10 ns | depends | Very short critical sections |
| `Interlocked` | ~5-10 ns | varies | Atomic single-variable ops |

---

## When to use which

| Need | Use |
|---|---|
| Default mutual exclusion | `lock (obj)` or `Lock` (C# 13+) |
| Try-to-acquire | `Monitor.TryEnter` |
| Async-friendly mutex | `SemaphoreSlim(1, 1)` |
| Many readers, few writers | `ReaderWriterLockSlim` OR `ImmutableDictionary` |
| Atomic single-field ops | `Interlocked.*` |
| Across processes | `Mutex` |
| Lazy init | `Lazy<T>` |
| Short, high-contention | profile; maybe `SpinLock` or lock-free |

---

## The bigger picture

Locks are correct but slow under contention. The modern playbook minimizes locking:
- Immutable data + atomic swaps.
- Concurrent collections (per-bucket locking).
- Channels for message passing (locks hidden inside).
- Per-thread state + aggregation.
- Lock-free with `Interlocked.CompareExchange`.

When you do lock, keep critical sections short and avoid blocking inside them.

→ Next: [10-Semaphores.md](10-Semaphores.md)
