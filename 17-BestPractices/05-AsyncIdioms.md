# Async Idioms

## The async contract

Async done well is invisible; async done wrong causes deadlocks, thread-pool starvation, and swallowed exceptions. These idioms encode the hard-won rules. The deep mechanics are in [Chapter 08](../08-Concurrency/README.md); this is the practitioner's checklist.

---

## Name async methods with the `Async` suffix

```csharp
public async Task<Order> GetOrderAsync(int id, CancellationToken ct = default);
public async Task SaveAsync(CancellationToken ct = default);
```

Task-returning methods get an `Async` suffix — it signals "await me" and disambiguates from sync overloads. (ASP.NET Core actions are a framework exception.)

---

## `async all the way` — never block on async

The cardinal rule: **don't mix sync and async.** Blocking on an async call (`.Result`, `.Wait()`, `.GetAwaiter().GetResult()`) can deadlock and always wastes a thread.

```csharp
// ✗ — sync-over-async: deadlock risk (UI/ASP.NET classic context), thread blocked
var result = GetOrderAsync(1).Result;
GetOrderAsync(1).Wait();

// ✓ — await all the way up the call chain
var result = await GetOrderAsync(1);
```

If a method calls async code, it must itself be async, and so on up to the entry point (`Main` can be `async Task`). Breaking the chain with `.Result` is where deadlocks live. See [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md).

---

## Never `async void`

```csharp
// ✗ — exceptions can't be caught by the caller; can crash the process
public async void Process() { await DoWorkAsync(); }

// ✓ — async Task is awaitable and observable
public async Task ProcessAsync() { await DoWorkAsync(); }
```

`async void` methods can't be awaited, so exceptions escape to the synchronization context (crashing the app) and callers can't know when they finish. The **only** legitimate use is event handlers (which must be `void`), and even then keep the body tiny and wrap in try/catch.

```csharp
// Acceptable: event handler (must be void), with internal exception handling
private async void Button_Click(object sender, EventArgs e) {
    try { await SaveAsync(); }
    catch (Exception ex) { ShowError(ex); }
}
```

---

## Accept `CancellationToken`, pass it through, last parameter

```csharp
public async Task<Data> FetchAsync(int id, CancellationToken ct = default) {
    var response = await _http.GetAsync($"/api/{id}", ct);   // forward it
    return await response.Content.ReadFromJsonAsync<Data>(ct);  // and again
}
```

- Async methods that do I/O should accept a `CancellationToken`, defaulting to `default`.
- Put it **last** in the parameter list (convention).
- **Forward it** to every async call you make — a token you accept but don't pass does nothing.
- Honor it: long CPU loops should `ct.ThrowIfCancellationRequested()`.

Analyzer CA2016 flags tokens you forget to forward. See [Chapter 08 §05](../08-Concurrency/05-Cancellation.md).

---

## `ConfigureAwait(false)` in library code

```csharp
// Library code — don't capture/resume on the caller's context
public async Task<Data> LoadAsync() {
    var raw = await _http.GetStringAsync(url).ConfigureAwait(false);
    return Parse(raw);
}
```

In **library** code, `ConfigureAwait(false)` avoids capturing the synchronization context — improving performance and preventing the classic UI/legacy-ASP.NET deadlock. In **application** code (especially ASP.NET Core, which has no sync context) it's usually unnecessary, and in UI app code you often *want* to resume on the UI thread (so omit it there). Rule of thumb: libraries → `ConfigureAwait(false)` everywhere; apps → usually omit. See [Chapter 08 §06](../08-Concurrency/06-ConfigureAwait.md).

---

## Return `Task` directly when you don't need `await`

```csharp
// ✗ — unnecessary async state machine for a pass-through
public async Task<Data> GetAsync(int id) => await _repo.LoadAsync(id);

// ✓ — return the task directly (no state machine overhead)
public Task<Data> GetAsync(int id) => _repo.LoadAsync(id);
```

If a method just forwards a task without doing work after the await, return the task directly to skip the state-machine overhead. **Caveat**: you lose the `await` if you have a `using`/`try` — the disposal/catch happens at the wrong time. When in doubt (resources, exception handling), keep `await`.

```csharp
// MUST await here — using disposes too early if you return the task
public async Task<Data> GetAsync(int id) {
    using var conn = Open();
    return await _repo.LoadAsync(conn, id);   // await keeps conn alive until done
}
```

---

## `ValueTask` for hot sync-completing paths

```csharp
public ValueTask<Data> GetAsync(int id) {
    if (_cache.TryGetValue(id, out var cached))
        return ValueTask.FromResult(cached);   // no Task allocation on the hot path
    return new ValueTask<Data>(LoadAsync(id));
}
```

`ValueTask<T>` avoids a `Task` allocation when the method often completes synchronously (cache hits). But it has rules: **await it exactly once**, don't block on it, don't store it. Use it only on measured-hot paths; default to `Task<T>`. See [Chapter 08 §04](../08-Concurrency/04-ValueTask.md).

---

## Parallelism: `WhenAll` / `WhenEach`

```csharp
// Run independent async ops concurrently
var tasks = ids.Select(id => FetchAsync(id));
Data[] results = await Task.WhenAll(tasks);

// Process results as they complete (.NET 9+)
await foreach (var task in Task.WhenEach(tasks))
    Handle(await task);
```

For independent operations, start them all and `await Task.WhenAll` rather than awaiting sequentially (which serializes them). `Task.WhenEach` streams completions. Beware unbounded concurrency — throttle with `SemaphoreSlim` or `Parallel.ForEachAsync`. See [Chapter 08 §15](../08-Concurrency/15-TaskWhenAllWhenAny.md).

```csharp
// ✗ — serial: each awaited before the next starts
foreach (var id in ids) results.Add(await FetchAsync(id));

// ✓ — concurrent
var results = await Task.WhenAll(ids.Select(FetchAsync));
```

---

## Async streams for sequences

```csharp
public async IAsyncEnumerable<Order> StreamOrdersAsync(
    [EnumeratorCancellation] CancellationToken ct = default) {
    await foreach (var row in _db.QueryAsync(ct))
        yield return Map(row);
}

// Consume
await foreach (var order in StreamOrdersAsync(ct)) { ... }
```

`IAsyncEnumerable<T>` streams items asynchronously without buffering the whole sequence — ideal for paged/streamed data. Note `[EnumeratorCancellation]` to wire the token. See [Chapter 08 §07](../08-Concurrency/07-AsyncStreams.md).

---

## Common bugs / gotchas

### Sync-over-async deadlock

`.Result`/`.Wait()` on a context-capturing task deadlocks. Await all the way; if you must bridge, do it only at the true top (e.g., a console `Main`).

### `async void` swallowing exceptions

Exceptions in `async void` crash the process or vanish. Use `async Task` everywhere except event handlers.

### Forgetting to pass the token

Accepting `CancellationToken ct` but not forwarding it makes cancellation a no-op. Forward to every async call.

### `await` in a loop that could be concurrent

Sequential `await` in a loop serializes independent work. Use `WhenAll` for independent operations (mind throttling).

### Fire-and-forget without observation

```csharp
DoWorkAsync();   // ⚠ — unobserved task; exceptions lost, no completion tracking
```

Either `await` it, or deliberately track it (store the task, attach a continuation that logs failures). Don't silently drop tasks.

---

## Summary

- `Async` suffix; **await all the way** — never `.Result`/`.Wait()` (deadlock + blocked thread).
- **Never `async void`** except event handlers (with try/catch).
- Accept `CancellationToken` (last param, default), **forward it** everywhere, honor it.
- `ConfigureAwait(false)` in **libraries**; usually omit in apps/UI.
- Return the task directly for pass-throughs (but `await` when a `using`/`try` is involved).
- `ValueTask` for measured sync-completing hot paths (await once); `Task.WhenAll`/`WhenEach` for concurrency; `IAsyncEnumerable` for streamed sequences.
- Don't fire-and-forget without observing exceptions.

→ Next: [06-CollectionIdioms.md](06-CollectionIdioms.md)
