# Chapter 08 — Concurrency

> async/await, Tasks, threads, locks, semaphores, atomic operations, concurrent collections, channels, the TPL, and the .NET memory model. The biggest chapter in the book — concurrency is where careers in C# distinguish themselves.

**Prerequisites**: [Chapter 02 (OOP)](../02-OOP/README.md), [Chapter 05 (Delegates)](../05-DelegatesEvents/README.md).

**Time to read**: ~12-15 hours, plus the rest of your career.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-ThreadsVsTasks.md](01-ThreadsVsTasks.md) | OS threads vs `Task`, the thread pool, work-stealing, when to choose each. |
| [02-AsyncAwaitFundamentals.md](02-AsyncAwaitFundamentals.md) | What `async` and `await` actually do, the state machine, suspension and resumption. |
| [03-TaskAndTaskT.md](03-TaskAndTaskT.md) | `Task` lifecycle, statuses, `Task.Run`, `Task.FromResult`, `TaskCompletionSource`, continuations. |
| [04-ValueTask.md](04-ValueTask.md) | When `ValueTask` beats `Task`, the don't-await-twice rule, `IValueTaskSource`. |
| [05-Cancellation.md](05-Cancellation.md) | `CancellationToken`, `CancellationTokenSource`, linked sources, timeouts, propagating cancellation through every call. |
| [06-ConfigureAwait.md](06-ConfigureAwait.md) | `SynchronizationContext`, why ASP.NET Core doesn't have one, when ConfigureAwait(false) matters. |
| [07-AsyncStreams.md](07-AsyncStreams.md) | `IAsyncEnumerable<T>`, `await foreach`, `[EnumeratorCancellation]`, server-streaming patterns. |
| [08-AsyncDisposable.md](08-AsyncDisposable.md) | `IAsyncDisposable`, `await using`, implementing both `Dispose` and `DisposeAsync`. |
| [09-LockingPrimitives.md](09-LockingPrimitives.md) | `lock`, `Monitor`, `Mutex`, `ReaderWriterLockSlim`, `lock` with the new `System.Threading.Lock` type (C# 13). |
| [10-Semaphores.md](10-Semaphores.md) | `SemaphoreSlim`, throttling, async-friendly waits. |
| [11-Interlocked.md](11-Interlocked.md) | Atomic increment/decrement, `CompareExchange`, lock-free CAS loops, the memory model. |
| [12-ConcurrentCollections.md](12-ConcurrentCollections.md) | `ConcurrentDictionary`, `ConcurrentQueue`, `ConcurrentBag`, `BlockingCollection`, when each fits. |
| [13-Channels.md](13-Channels.md) | `System.Threading.Channels`, bounded vs unbounded, producer-consumer pipelines. |
| [14-TPL.md](14-TPL.md) | `Parallel.For`, `Parallel.ForEachAsync`, PLINQ, partitioners, when parallelization helps. |
| [15-TaskWhenAllWhenAny.md](15-TaskWhenAllWhenAny.md) | `Task.WhenAll`, `Task.WhenAny`, `Task.WhenEach` (.NET 9), exception handling for fan-out. |
| [16-MemoryModelVolatile.md](16-MemoryModelVolatile.md) | The .NET memory model, when reads/writes can be reordered, `volatile`, `Volatile.Read/Write`, what `lock` guarantees. |
| [17-CommonAsyncBugs.md](17-CommonAsyncBugs.md) | Deadlocks, `async void`, sync-over-async (`.Result`), fire-and-forget, captured-context surprises. |
| [Questions.md](Questions.md) | ~40 questions — the longest in the book. |
| [Coding.md](Coding.md) | ~20 problems: producer-consumer, throttled HTTP, deadlock prediction, custom semaphore. |

→ Begin: [01-ThreadsVsTasks.md](01-ThreadsVsTasks.md)
