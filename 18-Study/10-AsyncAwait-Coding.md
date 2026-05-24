# 10 — async/await — Coding Questions ⭐⭐

> Predict the output / find the deadlock. (Concepts: [10-AsyncAwait.md](10-AsyncAwait.md))

---

### Q1 — The classic deadlock
```csharp
// In a UI / legacy ASP.NET context (has a SynchronizationContext):
public string GetData() {
    return GetDataAsync().Result;   // blocks
}
async Task<string> GetDataAsync() {
    await Task.Delay(100);          // default: capture context
    return "done";
}
```
<details><summary>Answer</summary>

**Deadlock.** `GetData` blocks the UI thread on `.Result`. `GetDataAsync`'s continuation (after `await`) needs to resume on that **captured** UI thread — which is blocked. Both wait forever. **Fixes:** await all the way (`await GetDataAsync()`), or `ConfigureAwait(false)` inside the library method, or never block. (In ASP.NET **Core**, no sync context → no deadlock, but you get thread-pool starvation instead.)
</details>

---

### Q2 — Does await run on a new thread?
```csharp
Console.WriteLine($"Before: {Environment.CurrentManagedThreadId}");
await Task.Delay(100);
Console.WriteLine($"After:  {Environment.CurrentManagedThreadId}");
```
<details><summary>Answer</summary>

The IDs **may differ**, but the point: `await Task.Delay` uses **no thread while waiting** — it suspends and resumes on a pool thread (no sync context) when the timer fires. `await` ≠ "run on another thread"; it's "suspend, then continue". For `Task.Delay`, *zero* threads are consumed during the wait.
</details>

---

### Q3 — When does an async method start running?
```csharp
async Task<int> WorkAsync() { Console.WriteLine("A"); await Task.Delay(10); Console.WriteLine("B"); return 1; }

Console.WriteLine("start");
var t = WorkAsync();
Console.WriteLine("called");
await t;
```
<details><summary>Answer</summary>

```
start
A
called
B
```
An async method runs **synchronously until the first incomplete await** — so "A" prints *before* `WorkAsync()` returns the task. Then control returns to the caller ("called"), and "B" runs after the delay completes.
</details>

---

### Q4 — async void can't be caught
```csharp
async void DoAsync() { await Task.Delay(10); throw new Exception("boom"); }
try { DoAsync(); await Task.Delay(100); }
catch { Console.WriteLine("caught"); }
```
<details><summary>Answer</summary>

**"caught" does NOT print** — `async void` exceptions can't be observed by the caller; they're raised on the sync context and typically **crash the process**. Use `async Task` so the exception lands on the returned task and can be awaited/caught.
</details>

---

### Q5 — ValueTask double-await bug
```csharp
ValueTask<int> vt = GetAsync();
int a = await vt;
int b = await vt;     // ?
```
<details><summary>Answer</summary>

**Undefined behavior / may throw.** A `ValueTask` must be awaited **once**. Awaiting twice (or reading `.Result` before completion) is illegal because its backing source may be pooled/reused. If you need to await multiple times, call `.AsTask()` once and await that.
</details>

---

### Q6 — Sequential vs parallel awaits
```csharp
async Task<int> Sum() {
    var a = await GetAsync(1);   // 1s
    var b = await GetAsync(2);   // 1s
    return a + b;                 // total time?
}
```
<details><summary>Answer</summary>

**~2 seconds** (sequential). To overlap (~1s): start both, then await — `var ta = GetAsync(1); var tb = GetAsync(2); return await ta + await tb;` (or `Task.WhenAll`). Awaiting immediately serializes independent work.
</details>

---

### Q7 — OperationCanceledException is expected
```csharp
var cts = new CancellationTokenSource();
cts.CancelAfter(50);
try { await Task.Delay(1000, cts.Token); Console.WriteLine("done"); }
catch (OperationCanceledException) { Console.WriteLine("cancelled"); }
```
<details><summary>Answer</summary>

**`cancelled`** — `Task.Delay` with a token throws `OperationCanceledException` (a subtype, `TaskCanceledException`) when cancelled at ~50ms. Cancellation is **expected control flow**, not an error — don't log it as a failure; let it propagate.
</details>

---

### Q8 — ConfigureAwait(false) — where matters?
```csharp
// In a class library:
public async Task<int> ComputeAsync() {
    var x = await FetchAsync().ConfigureAwait(false);
    return x * 2;
}
```
<details><summary>Answer</summary>

Correct for **library** code — it doesn't capture the caller's sync context, avoiding deadlocks/overhead if a UI/legacy-ASP.NET caller blocks on it. It's a no-op in ASP.NET Core (no context) but still correct for portability. **Don't** use it in UI code that needs to touch UI after the await.
</details>

---

### Q9 — Fire-and-forget swallows the error
```csharp
public void Handle() {
    _ = ProcessAsync();    // not awaited
}
async Task ProcessAsync() { await Task.Delay(10); throw new Exception("lost"); }
```
<details><summary>Answer</summary>

The exception is **silently lost** (unobserved task). `Handle` returns immediately; nobody observes `ProcessAsync`'s failure. **Fix:** await it, or route to managed background processing that logs failures, and hook `TaskScheduler.UnobservedTaskException` in tests. ([12-Concurrent-Parallel-AsyncBugs.md](12-Concurrent-Parallel-AsyncBugs.md))
</details>

---

### Q10 — Synchronous completion path (senior)
```csharp
async Task<int> MaybeAsync(bool cached) {
    if (cached) return 42;            // completes synchronously
    await Task.Delay(100);
    return 99;
}
```
<details><summary>Answer</summary>

When `cached` is true, the method **completes synchronously** (no real await happens) — the returned `Task` is already completed, no thread switch, no continuation. This is why **`ValueTask`** is useful for hot paths that often complete synchronously (cache hits) — it avoids allocating a `Task` object in that common case.
</details>
