# 10 — async / await ⭐⭐ (highest-frequency topic)

## ⚡ 30-second answer

`async`/`await` lets you write asynchronous code that *reads* sequentially. The compiler rewrites an `async` method into a **state machine**: at each `await`, if the awaited operation isn't already complete, the method **returns to its caller** and registers a continuation to resume when the operation finishes — **no thread is blocked while waiting**. This is why async scales for **I/O**: the request thread is freed to serve others instead of sitting on a blocking call. `await` does **not** mean "run on another thread"; it means "suspend until done, then continue."

---

## Core mechanics

**What `await` actually does**:
1. Evaluate the awaitable (e.g., `Task`). If **already complete**, continue synchronously (no suspension).
2. If not, **return** to the caller (the method's `Task` is still pending), attaching a **continuation**.
3. When the awaited op completes, the continuation resumes the method — by default on the **captured synchronization context** (UI thread / none in ASP.NET Core).

```csharp
async Task<string> GetAsync() {
    var res = await _http.GetAsync(url);     // suspend here if not done; thread is freed
    return await res.Content.ReadAsStringAsync();
}
```

**The state machine** (conceptual): the compiler turns locals into fields, captures the current position, and uses an `AsyncTaskMethodBuilder` to manage the returned `Task` and continuations. You don't write it, but *know* it exists (explains why locals survive across `await`, and why `async` adds a little overhead).

**`ConfigureAwait(false)`** — don't capture the context; resume on any pool thread:
```csharp
await SomethingAsync().ConfigureAwait(false);   // libraries: avoid deadlocks/overhead
```
- **ASP.NET Core has no sync context** → `ConfigureAwait(false)` is effectively a no-op there.
- **UI apps (WPF/Blazor/MAUI)** *do* — without it, continuations resume on the UI thread (needed to touch UI; harmful if a caller blocks → deadlock).

**Cancellation**:
```csharp
async Task DoAsync(CancellationToken ct) {
    await Task.Delay(1000, ct);              // throws OperationCanceledException if cancelled
    ct.ThrowIfCancellationRequested();
}
```
Flow the token **through every async call** ([22](22-Architecture-Patterns.md) async-at-scale).

---

## Comparison tables

| | `Task` | `ValueTask` |
|---|---|---|
| Allocates | yes (heap object) | no, when completes synchronously |
| Await twice / `.Result` early | ok | **not allowed** (single consumption) |
| Use | general | hot paths that often complete sync (cache hit) |

| Context | Sync context? | `ConfigureAwait(false)` effect |
|---|---|---|
| ASP.NET Core | none | no-op (still correct in libraries) |
| WPF/WinForms/Blazor/MAUI UI | yes (UI thread) | resume off UI thread (don't touch UI after) |
| Console | none | no-op |

---

## 🪤 Traps & gotchas

- **Sync-over-async deadlock**: blocking on async with `.Result`/`.Wait()` in a context with a sync context (UI, legacy ASP.NET). The blocked thread *is* the one the continuation needs → deadlock. Fix: **await all the way**; never `.Result`/`.Wait()`.
- **`async void`**: can't be awaited, its exceptions escape and **crash the process**. Only for event handlers. Use `async Task` everywhere else ([07](07-Exceptions-Idioms.md)).
- **`await` ≠ background thread**: `await ReadFileAsync()` doesn't spin a thread — it suspends. For CPU work you *do* need `Task.Run` ([09](09-Threading-and-Tasks.md)).
- **Forgetting to await** (fire-and-forget): the method runs detached, exceptions are lost, ordering breaks. Await it (or deliberately manage it).
- **`ValueTask` misuse**: awaiting it twice or calling `.Result` before completion is undefined. Await once; convert with `.AsTask()` if you need to store it.
- **Capturing context unnecessarily in libraries** → overhead and deadlock risk for UI consumers. Use `ConfigureAwait(false)` in library code.
- **`Task.Delay` vs `Thread.Sleep`**: `Thread.Sleep` blocks a thread; `await Task.Delay` doesn't. Never `Thread.Sleep` in async code.
- **Not flowing `CancellationToken`**: work continues after the caller gave up — wasted resources at scale.

---

## ❓ Likely questions

**Q: What does `await` do under the hood?**
A: The compiler builds a state machine. At an `await`, if the task isn't done, the method returns to its caller and registers a continuation to resume when the task completes — no thread is blocked meanwhile.

**Q: Does `await` run code on another thread?**
A: No. It suspends until the awaited operation completes. For I/O, no thread is used while waiting. Only `Task.Run` moves CPU work to another thread.

**Q: Why does `.Result` cause a deadlock?**
A: In a context with a single-threaded sync context, the continuation must resume on that captured thread — but you've blocked it waiting on `.Result`. Both wait forever. Await instead.

**Q: What is `ConfigureAwait(false)` and when use it?**
A: It tells the await not to capture/resume on the sync context. Use it in **library** code to avoid overhead and UI deadlocks. It's a no-op in ASP.NET Core (no context) but still correct.

**Q: `Task` vs `ValueTask`?**
A: `ValueTask` avoids allocation when the result is often available synchronously (e.g., cache hit). But it must be awaited once and not have `.Result` accessed before completion. Use `Task` by default.

**Q: Why is `async void` bad?**
A: It can't be awaited and its exceptions can't be caught by the caller — they crash the process. Use `async Task` (except UI event handlers).

**Q: How does cancellation work with async?**
A: Pass a `CancellationToken`; awaited operations throw `OperationCanceledException` when it's cancelled. Flow the token through the whole call chain and check `ThrowIfCancellationRequested` for CPU loops.

**Q: What does an `async` method return and when does it start?**
A: It returns a `Task`/`Task<T>`/`ValueTask` and runs **synchronously until the first incomplete await**, then returns the pending task to the caller.

---

## 🎓 Senior Extra

- **State machine cost**: each `async` method has a small overhead (state machine + builder); for value types the machine can be a struct (no alloc) until it suspends. .NET 10 elides some async boxing. Don't make trivial sync methods async.
- **Synchronization context internals**: `await` captures `SynchronizationContext.Current` (or the current `TaskScheduler`). UI contexts post continuations to the message loop; ASP.NET Core removed its context (better throughput, and `ConfigureAwait(false)` unnecessary there).
- **`IValueTaskSource`** lets high-perf libraries pool the async machinery (e.g., `Socket`/pipelines) for zero-allocation awaits — advanced.
- **`IAsyncEnumerable<T>` + `await foreach`** for async streams with backpressure; annotate the token with `[EnumeratorCancellation]` so it flows.
- **`IAsyncDisposable` / `await using`** for async cleanup (flush a stream, dispose a DB connection asynchronously).
- **Async coordination primitives**: `SemaphoreSlim.WaitAsync` (async lock/throttle), `Channel<T>` (async producer/consumer), `Task.WhenEach` (.NET 9) — there is **no `await` inside `lock`** (use `SemaphoreSlim`) ([11](11-Synchronization-and-MemoryModel.md)).
- **Exception/cancellation semantics**: `await` unwraps `AggregateException` to the first exception; `OperationCanceledException` is "success-ish" (expected) — don't log as an error; `TaskCanceledException` derives from it.
- **Diagnosing async issues**: thread-pool starvation shows as growing queue length + thread count in `dotnet-counters`; a dump shows threads stuck on `.Result` ([21](21-Deployment-Perf-Tooling.md)).

→ Deeper: [`../CSharpBook/08-Concurrency/02-AsyncAwaitFundamentals.md`](../CSharpBook/08-Concurrency/README.md)
