# 11 — Synchronization & Memory Model — Coding Questions ⭐⭐

> Predict the output / find the race or deadlock. (Concepts: [11-Synchronization-and-MemoryModel.md](11-Synchronization-and-MemoryModel.md))

---

### Q1 — The race condition
```csharp
int counter = 0;
Parallel.For(0, 100_000, _ => counter++);
Console.WriteLine(counter);
```
<details><summary>Answer</summary>

**Less than 100,000** (non-deterministic). `counter++` is **read-modify-write** — not atomic; threads overwrite each other's updates. **Fix:** `Interlocked.Increment(ref counter)` (or `lock`). This is *the* race-condition question.
</details>

---

### Q2 — Does volatile fix Q1?
```csharp
volatile int counter = 0;
Parallel.For(0, 100_000, _ => counter++);
```
<details><summary>Answer</summary>

**No.** `volatile` ensures visibility/ordering of reads/writes but does **not** make `counter++` atomic (it's still read-modify-write). You still lose updates. Use `Interlocked.Increment`. `volatile` is for a *flag* you set in one thread and read in another, not for compound ops.
</details>

---

### Q3 — Deadlock: spot it
```csharp
object A = new(), B = new();
var t1 = Task.Run(() => { lock (A) { Thread.Sleep(50); lock (B) { } } });
var t2 = Task.Run(() => { lock (B) { Thread.Sleep(50); lock (A) { } } });
await Task.WhenAll(t1, t2);
```
<details><summary>Answer</summary>

**Deadlock.** `t1` holds A waits B; `t2` holds B waits A → circular wait. **Fix:** acquire locks in a **consistent global order** (both lock A then B). This is the dining-philosophers cycle.
</details>

---

### Q4 — await inside lock
```csharp
private readonly object _gate = new();
async Task DoAsync() {
    lock (_gate) {
        await Task.Delay(100);   // ?
    }
}
```
<details><summary>Answer</summary>

**Compile error** — you can't `await` inside a `lock` block. `Monitor` (lock) is thread-affine and must be released by the acquiring thread, but `await` may resume on a different thread. **Fix:** use `SemaphoreSlim.WaitAsync()`/`Release()` as an async lock.
</details>

---

### Q5 — What's wrong with the lock object?
```csharp
public class Cache {
    public void Add() { lock (this) { /* ... */ } }   // ?
}
```
<details><summary>Answer</summary>

**`lock(this)` is dangerous** — external code holding your instance can `lock` on the same object, causing unexpected contention/deadlock. (Same problem with `lock(typeof(X))` or interned strings.) **Fix:** a **private readonly** dedicated lock object: `private readonly object _gate = new();`.
</details>

---

### Q6 — Spin loop never exits
```csharp
bool _stop = false;           // not volatile
var t = Task.Run(() => { while (!_stop) { } });   // worker
Thread.Sleep(100);
_stop = true;                  // main thread
await t;                       // ?
```
<details><summary>Answer</summary>

May **never exit** (especially in Release on some CPUs) — the JIT can cache `_stop` in a register, so the worker never sees the update. **Fix:** mark `_stop` `volatile` (or use `Volatile.Read`, or a `CancellationToken`). Visibility, not atomicity, is the issue here.
</details>

---

### Q7 — Interlocked CAS pattern
```csharp
int _max = 0;
void UpdateMax(int candidate) {
    int current;
    do { current = _max; if (candidate <= current) return; }
    while (Interlocked.CompareExchange(ref _max, candidate, current) != current);
}
```
<details><summary>Answer</summary>

**Correct lock-free max.** CAS loop: read `current`, only swap if `_max` is still `current` (nobody changed it); if it changed, retry. `CompareExchange` returns the *original* value; loop until the swap succeeds. This is how lock-free algorithms are built.
</details>

---

### Q8 — SemaphoreSlim leak
```csharp
var sem = new SemaphoreSlim(1, 1);
await sem.WaitAsync();
await DoWorkAsync();        // may throw
sem.Release();             // ?
```
<details><summary>Answer</summary>

If `DoWorkAsync` throws, `Release()` is **skipped** → the permit leaks → next `WaitAsync` blocks forever. **Fix:** `await sem.WaitAsync(); try { await DoWorkAsync(); } finally { sem.Release(); }`.
</details>

---

### Q9 — Lazy thread-safe init
```csharp
// Hand-rolled double-checked locking — what's safer?
private static Config? _config;
private static readonly object _lock = new();
Config Get() {
    if (_config == null) lock (_lock) { if (_config == null) _config = Load(); }
    return _config;
}
```
<details><summary>Answer</summary>

It works (double-checked locking) but is **easy to get subtly wrong** (memory ordering). The idiomatic, safe alternative: **`Lazy<Config>`** — `private static readonly Lazy<Config> _config = new(Load);` then `_config.Value`. Thread-safe by default, no hand-rolled locking.
</details>

---

### Q10 — SemaphoreSlim is not reentrant (senior)
```csharp
var sem = new SemaphoreSlim(1, 1);
async Task Outer() { await sem.WaitAsync(); try { await Inner(); } finally { sem.Release(); } }
async Task Inner() { await sem.WaitAsync(); try { } finally { sem.Release(); } }   // ?
```
<details><summary>Answer</summary>

**Deadlock** — `SemaphoreSlim` is **not reentrant**. `Outer` holds the single permit; `Inner` waits for it on the same logical flow → never released until Inner proceeds, but Inner can't. (`Monitor`/`lock` *is* reentrant on the same thread, but you can't `await` in it.) **Fix:** don't re-acquire; restructure so only one level locks.
</details>
