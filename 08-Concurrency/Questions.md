# Chapter 08 — Questions

> The big chapter — 40+ drilling questions across threads, async/await, locks, semaphores, channels, parallelism, memory model.

---

## Threads vs Tasks

**Q1.** Why do `Task`s scale much better than raw `Thread`s for thousands of concurrent operations?
<details><summary>Answer</summary>Each Thread is an OS-level construct with ~1 MB stack and kernel scheduling overhead. Creating 10,000 = ~10 GB memory + slow context switching. Tasks run on the thread pool — a small set of recycled threads. Async I/O doesn't even need a thread while waiting — the same handful of threads service thousands of concurrent operations.</details>

**Q2.** When should you create a `Thread` directly instead of using `Task.Run`?
<details><summary>Answer</summary>Long-running dedicated work that shouldn't share the thread pool (game loop, dedicated I/O thread, COM apartment requirement, custom stack size, foreground thread that keeps the process alive). For everyday parallel/async work, Task.Run.</details>

**Q3.** What's wrong with this?
```csharp
public async Task GetAsync() {
    return await Task.Run(() => httpClient.GetAsync(url));
}
```
<details><summary>Answer</summary>"Async-over-sync" — `httpClient.GetAsync` is already async. Wrapping it in `Task.Run` grabs a pool thread just to immediately await another pool thread. Net waste. Just call `await httpClient.GetAsync(url)`.</details>

---

## async/await

**Q4.** What does the compiler turn `async Task<int> M()` into?
<details><summary>Answer</summary>A state machine struct implementing `IAsyncStateMachine`, plus an `AsyncTaskMethodBuilder<int>` that produces the Task. The method body becomes a switch over states; each await suspends and registers a continuation. On suspension, the struct gets boxed to the heap.</details>

**Q5.** Why doesn't this run concurrently?
```csharp
var a = await GetA();
var b = await GetB();
var c = await GetC();
```
<details><summary>Answer</summary>Each `await` waits for completion before starting the next call. Sequential. To make them concurrent, start first, then await: `var ta = GetA(); var tb = GetB(); var tc = GetC(); var results = await Task.WhenAll(ta, tb, tc);`.</details>

**Q6.** What's wrong with `async void` outside event handlers?
<details><summary>Answer</summary>The caller can't await it. Exceptions propagate to the SynchronizationContext (crashing the process on the thread pool). No way to know when it finishes. Use `async Task` instead. The only legal `async void` is for event handlers (signature requires void).</details>

**Q7.** What's the difference between `Task` and `Task<T>`?
<details><summary>Answer</summary>`Task` represents an async operation that completes (with no value). `Task<T>` is one that produces a value of type T. Awaiting `Task` returns void; awaiting `Task<T>` returns T.</details>

**Q8.** Predict the output:
```csharp
Console.WriteLine("1");
var t = M();
Console.WriteLine("2");
await t;
Console.WriteLine("3");

async Task M() {
    Console.WriteLine("A");
    await Task.Delay(100);
    Console.WriteLine("B");
}
```
<details><summary>Answer</summary>`1 A 2 B 3`. M() runs synchronously until the first await. Then it returns the (incomplete) Task. After "2", we await — suspend. ~100ms later, M's continuation runs ("B"), then our continuation ("3").</details>

---

## Task / Task<T>

**Q9.** When should you return `ValueTask<T>` instead of `Task<T>`?
<details><summary>Answer</summary>When the method **often completes synchronously** (cache hit, polling that finds it immediately). For sync-completing path: zero allocation. For async path: same as Task. Use in hot library code where allocations matter. For typical app code, Task is simpler and safer.</details>

**Q10.** What are the rules of using `ValueTask<T>`?
<details><summary>Answer</summary>(1) Await at most once. (2) Don't access `.Result` unless completed. (3) Don't store / pass around / share. Violating these is undefined behavior. If you need to store or multi-await, convert to Task via `.AsTask()`.</details>

**Q11.** What does `TaskCompletionSource<T>` do?
<details><summary>Answer</summary>Lets you create a Task whose completion you control manually. Use to bridge non-async APIs (events, callbacks) to a Task. Always use `TaskCreationOptions.RunContinuationsAsynchronously` to avoid running continuations on the thread that calls SetResult.</details>

---

## Cancellation

**Q12.** Why does every async method should accept a `CancellationToken`?
<details><summary>Answer</summary>Cancellation is cooperative. If a method doesn't accept the token, callers can't tell it to stop. The convention: `CancellationToken cancellationToken = default` as the last parameter. Forward to every inner async call.</details>

**Q13.** What's the difference between `ct.ThrowIfCancellationRequested()` and `ct.IsCancellationRequested`?
<details><summary>Answer</summary>`ThrowIfCancellationRequested` throws `OperationCanceledException` if the token is canceled (the idiomatic way to bail out). `IsCancellationRequested` just returns a bool — useful for "check and return success" patterns or non-throwing cleanup paths.</details>

**Q14.** What's `CancellationTokenSource.CreateLinkedTokenSource`?
<details><summary>Answer</summary>Creates a CTS whose token is canceled when ANY of the source tokens is canceled. Used to combine multiple cancellation reasons (e.g., caller's token + local timeout). Common in libraries that want to add a timeout to their work.</details>

**Q15.** What's the proper pattern for adding a 5-second timeout to an existing token?
<details><summary>Answer</summary>
```csharp
using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(5));
using var linked = CancellationTokenSource.CreateLinkedTokenSource(callerCt, timeout.Token);
await DoAsync(linked.Token);
```
Or with `Task.WaitAsync` (.NET 6+): `await DoAsync(ct).WaitAsync(TimeSpan.FromSeconds(5), ct);`.
</details>

---

## ConfigureAwait

**Q16.** Why do libraries use `ConfigureAwait(false)` but ASP.NET Core controllers usually don't?
<details><summary>Answer</summary>Libraries don't know who calls them. If called from UI code, they'd capture the UI SynchronizationContext on every await — slow and deadlock-prone. ConfigureAwait(false) says "I don't need the original context." ASP.NET Core has NO SynchronizationContext — ConfigureAwait(false) is a no-op. So app code in ASP.NET Core doesn't need it.</details>

**Q17.** Does `ConfigureAwait(false)` help if the calling code does `.Result`?
<details><summary>Answer</summary>It helps prevent the SPECIFIC deadlock where the continuation needs the captured context. But the real fix is "don't call .Result." Async-all-the-way removes the deadlock class entirely. ConfigureAwait is a defense in depth, not the primary cure.</details>

---

## Async streams

**Q18.** What does `[EnumeratorCancellation]` do?
<details><summary>Answer</summary>On the CancellationToken parameter of an `async IAsyncEnumerable<T>` method, this attribute hooks the parameter to whatever token the consumer passes via `WithCancellation(token)`. Without it, the consumer's cancellation is silently ignored inside the iterator.</details>

**Q19.** What's the difference between `IEnumerable<T>` and `IAsyncEnumerable<T>`?
<details><summary>Answer</summary>`IEnumerable<T>` iterates synchronously — each MoveNext returns a bool synchronously. `IAsyncEnumerable<T>` iterates asynchronously — MoveNextAsync returns a `ValueTask<bool>`, can await between items. Consume with `await foreach`.</details>

---

## Locking primitives

**Q20.** Why doesn't `lock` work with `await`?
<details><summary>Answer</summary>`lock` is implemented via `Monitor.Enter/Exit` — tied to the OS thread. After await, the continuation may run on a different thread, which doesn't own the lock — `Monitor.Exit` would throw "synchronization lock not owned." The compiler flags this as a compile error inside a lock block.</details>

**Q21.** What's the async-friendly mutex?
<details><summary>Answer</summary>`SemaphoreSlim(1, 1)` — initialized with 1 permit and max 1. WaitAsync acquires it; Release releases. Not thread-bound, so safe to span awaits.</details>

**Q22.** What does `ReaderWriterLockSlim` add over `lock`?
<details><summary>Answer</summary>Multiple readers can hold the read lock concurrently; writers exclude everyone. Useful for read-mostly workloads (90%+ reads). Overhead is higher than plain lock, so it's worth it only when reads are frequent and contend.</details>

**Q23.** Why should you NEVER `lock (this)` or `lock (typeof(MyClass))`?
<details><summary>Answer</summary>Both expose the lock to external code. Anyone can `lock(myInstance)` and conflict with internal locking — deadlock risk or surprising serialization. Always use a private readonly object (or `Lock` in C# 13+).</details>

---

## Semaphores

**Q24.** Show how to limit concurrent HTTP calls to 5.
<details><summary>Answer</summary>
```csharp
using var sem = new SemaphoreSlim(5);
var tasks = urls.Select(async url => {
    await sem.WaitAsync();
    try { await client.GetAsync(url); }
    finally { sem.Release(); }
});
await Task.WhenAll(tasks);
```
Or more cleanly with `Parallel.ForEachAsync`:
```csharp
await Parallel.ForEachAsync(urls, new() { MaxDegreeOfParallelism = 5 }, async (url, ct) => {
    await client.GetAsync(url, ct);
});
```
</details>

**Q25.** Is `SemaphoreSlim` re-entrant?
<details><summary>Answer</summary>No. Unlike `lock` (Monitor) which is re-entrant on the same thread, SemaphoreSlim isn't. Taking it twice without releasing causes a deadlock. Use a library like Nito.AsyncEx's `AsyncLock` for re-entrant async locking if needed.</details>

---

## Interlocked

**Q26.** Why is `_counter++` not thread-safe?
<details><summary>Answer</summary>It's read-modify-write: load `_counter`, add 1, store. Two threads can both load the same value, both add 1, both store — losing one increment. Use `Interlocked.Increment(ref _counter)` for atomic compound operations.</details>

**Q27.** What's `Interlocked.CompareExchange`?
<details><summary>Answer</summary>Atomic Compare-And-Swap. `Interlocked.CompareExchange(ref field, newValue, expectedValue)` — if field equals expected, set to new; return original value. Building block for lock-free CAS loops: read; compute new; CAS-swap; if interfered, retry.</details>

**Q28.** When to use Interlocked vs lock?
<details><summary>Answer</summary>Interlocked for **single-variable** atomic operations (increment, swap, CAS). Lock for **multi-field invariants** or longer critical sections. Interlocked is faster but covers a narrower set of operations.</details>

---

## Concurrent collections

**Q29.** What's the difference between `ConcurrentDictionary` and `Dictionary` + lock?
<details><summary>Answer</summary>ConcurrentDictionary has fine-grained per-bucket locking (4 locks per CPU by default). Multiple threads accessing different buckets don't contend. Dictionary + global lock serializes everyone. For high-contention workloads, ConcurrentDictionary scales much better.</details>

**Q30.** Why might `dict.GetOrAdd(key, factory)` call the factory multiple times under contention?
<details><summary>Answer</summary>ConcurrentDictionary guarantees only one **stored** value per key, not one factory invocation. Multiple threads can both call the factory; the first to commit wins. For factory-must-run-once, wrap in `Lazy<T>`: `Lazy<TValue>` ensures one-time invocation.</details>

---

## Channels

**Q31.** What's the modern async producer-consumer primitive?
<details><summary>Answer</summary>`Channel<T>` (System.Threading.Channels). Bounded channels provide backpressure (writer awaits when full). Async-friendly throughout. Replaces `BlockingCollection<T>` (which blocks threads).</details>

**Q32.** What does `BoundedChannelFullMode.Wait` do?
<details><summary>Answer</summary>Default for bounded channels. When the channel is full, the writer awaits in `WriteAsync` until a slot frees up. Produces natural backpressure — producer slows to match consumer. Alternatives: DropOldest (lose the oldest item), DropNewest, DropWrite (reject the new write).</details>

---

## TPL / Parallel

**Q33.** When should you use `Parallel.For` vs `Task.Run`?
<details><summary>Answer</summary>`Parallel.For` for CPU-bound parallel iteration over a known range. The TPL partitions the work across cores. `Task.Run` for a single long-running CPU task, or for one-off offloading from a UI thread. For async I/O with concurrency limits, `Parallel.ForEachAsync` (.NET 6+).</details>

**Q34.** What's `Parallel.ForEachAsync` and when was it added?
<details><summary>Answer</summary>Added in .NET 6. Async-aware parallel iteration with `MaxDegreeOfParallelism` throttling. Modern replacement for SemaphoreSlim + WhenAll pattern. Body is async; concurrency limited; cancellation supported.</details>

**Q35.** What's wrong with `Parallel.For(0, n, async i => await DownloadAsync(i))`?
<details><summary>Answer</summary>`Parallel.For` is sync — it calls the body but doesn't await its returned Task. Sees the Task return value, considers the iteration done — even though work is still happening. Use `Parallel.ForEachAsync` for async bodies.</details>

---

## WhenAll / WhenAny

**Q36.** What does `Task.WhenAll` do when one of the tasks throws?
<details><summary>Answer</summary>Waits for ALL tasks to complete first. Then the result task is faulted with an AggregateException containing all exceptions. Awaiting it unwraps only the FIRST exception; the rest are on `.Exception.InnerExceptions`.</details>

**Q37.** What's `Task.WhenEach` and what does it do?
<details><summary>Answer</summary>.NET 9+. Returns an `IAsyncEnumerable<Task<T>>` that yields each input task in COMPLETION order. Let you process results as they arrive (vs WhenAll which gives them all at once at the end). Useful for showing progress, early termination, etc.</details>

---

## Memory model

**Q38.** When do you actually need `volatile`?
<details><summary>Answer</summary>Rarely. Most concurrent state uses lock or Interlocked. Volatile is for simple flag patterns (e.g., shutdown signal) or spin loops where you don't want the JIT to hoist the read out of the loop. For 64-bit fields on 32-bit hardware, use Interlocked.Read (volatile doesn't support long).</details>

**Q39.** What's wrong with this on weak-memory architectures (ARM)?
```csharp
private bool _ready;
private int _data;

// Thread A
_data = 42;
_ready = true;

// Thread B
while (!_ready) { }
Console.WriteLine(_data);
```
<details><summary>Answer</summary>Without volatile/lock, the writes can be reordered between threads — B might see `_ready == true` but `_data == 0`. On x64 the specific reordering is rare due to strong memory model; on ARM it happens. Fix: `volatile bool _ready` (release barrier on write; acquire on read).</details>

---

## Common async bugs

**Q40.** Why is `task.Result` bad in async code?
<details><summary>Answer</summary>Blocks the calling thread until the task completes. Wastes a thread pool slot. Can deadlock if a captured SynchronizationContext is the thread being blocked. Wraps exceptions in AggregateException. Use `await` instead — async all the way.</details>

**Q41.** What does `_ = SomeAsync();` mean?
<details><summary>Answer</summary>Explicit fire-and-forget. The `_ =` tells the compiler "I know I'm not awaiting; please shut up about it." Useful for genuine fire-and-forget (background work), but you should wrap in a try/catch + logging to handle unobserved exceptions.</details>

**Q42.** A coworker writes:
```csharp
public Task<int> GetCachedAsync() => Task.Run(() => _cache.Get("key"));
```
What's wrong?
<details><summary>Answer</summary>`_cache.Get` is presumably sync and fast. Wrapping in Task.Run grabs a pool thread for nanoseconds of work — overhead beats the gain. If the method is supposed to be async-shaped for an interface, return `Task.FromResult(_cache.Get("key"))`. If you genuinely have async cache, use that.</details>

---

## Synthesis

**Q43.** Build a method that downloads N URLs concurrently, limited to 5 at a time, with per-request 10-second timeout and overall cancellation.
<details><summary>Solution</summary>
```csharp
public async Task<string[]> DownloadAllAsync(IList<string> urls, CancellationToken ct = default) {
    var results = new string[urls.Count];
    await Parallel.ForEachAsync(
        urls.Select((url, idx) => (url, idx)),
        new ParallelOptions { MaxDegreeOfParallelism = 5, CancellationToken = ct },
        async (item, token) => {
            using var timeout = CancellationTokenSource.CreateLinkedTokenSource(token);
            timeout.CancelAfter(TimeSpan.FromSeconds(10));
            results[item.idx] = await _http.GetStringAsync(item.url, timeout.Token);
        });
    return results;
}
```
Uses Parallel.ForEachAsync for throttling, linked CTS for per-request timeout combining outer cancellation.
</details>

**Q44.** Explain why `ConcurrentDictionary` is faster than `Dictionary + lock` at high concurrency.
<details><summary>Answer</summary>ConcurrentDictionary uses fine-grained bucket-level locking. Each operation locks only one bucket (or just one lock per ~ProcessorCount buckets). Multiple threads operating on different keys mostly don't contend. Dictionary + global lock serializes everyone — under high concurrency, throughput collapses. For low-contention workloads, the difference is small.</details>

---

→ [Coding.md](Coding.md)
