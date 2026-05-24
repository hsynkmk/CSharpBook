# Cancellation — CancellationToken

## What it is

A **cooperative** mechanism for canceling async (or long-running sync) work. The caller creates a `CancellationTokenSource`, hands a `CancellationToken` to the work, and signals cancellation when needed. The work checks the token and bails out gracefully.

```csharp
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(5));   // auto-cancel after 5s

try {
    var result = await DoWorkAsync(cts.Token);
} catch (OperationCanceledException) {
    Console.WriteLine("Canceled or timed out");
}

public async Task<int> DoWorkAsync(CancellationToken ct) {
    for (int i = 0; i < 100; i++) {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct);
        // ...
    }
    return 42;
}
```

Cancellation is **cooperative**: the work decides when (and if) to check the token. No forced termination — that's deliberate, because forced cancellation leaves shared state in unpredictable states.

---

## Why it exists

Long-running async operations need:
- A user-canceled "Cancel" button.
- A timeout when the operation takes too long.
- A "shut down" signal during application stop.
- A "skip this request" decision when the client disconnected.

The CancellationToken pattern handles all of these uniformly. Every BCL async method accepts a token; the convention is universal.

---

## The pieces

### CancellationTokenSource (CTS) — the producer

```csharp
var cts = new CancellationTokenSource();
cts.Cancel();             // signal cancellation
cts.CancelAfter(5000);    // signal after 5 seconds
cts.IsCancellationRequested;   // true after Cancel
cts.Dispose();            // clean up timer + registrations
```

Use `using var cts = new CancellationTokenSource();` to ensure disposal.

### CancellationToken — the consumer

```csharp
public async Task Work(CancellationToken ct) {
    ct.ThrowIfCancellationRequested();    // throw if canceled
    bool canceled = ct.IsCancellationRequested;   // check flag
    ct.Register(() => Cleanup());          // register callback
    await SomeAsync(ct);                    // pass to inner async work
}
```

The token is a value type (struct). Cheap to pass around. Holds a reference to the CTS internally.

`CancellationToken.None` is a token that's never canceled (the default for optional parameters).

---

## Pattern: accept and forward

The single most important pattern. Every async method should:

```csharp
public async Task<int> DoWorkAsync(CancellationToken ct = default) {
    await Task.Delay(1000, ct);              // forward to inner
    await _httpClient.GetAsync(url, ct);      // forward to inner
    ct.ThrowIfCancellationRequested();        // explicit check before expensive sync work
    return Compute();
}
```

- Last parameter, named `cancellationToken` (or `ct` shorthand), with default `default(CancellationToken)` = `CancellationToken.None`.
- Pass to every inner async call.
- Check `ThrowIfCancellationRequested` before expensive operations the inner calls might not check.

Tokens **propagate down the call chain**. The top-level caller controls cancellation; everyone in the middle just forwards.

---

## CancelAfter — timeout

```csharp
using var cts = new CancellationTokenSource();
cts.CancelAfter(TimeSpan.FromSeconds(10));

try {
    var data = await httpClient.GetStringAsync(url, cts.Token);
} catch (OperationCanceledException) {
    Console.WriteLine("Timed out");
}
```

The CTS sets an internal timer. After the delay, it cancels itself. The async work sees the canceled token and throws `OperationCanceledException`.

For per-request timeouts in HTTP, `HttpClient.Timeout` exists but it's clumsier; `CancelAfter` is the modern pattern.

---

## Linked tokens

Combine multiple cancellation sources:

```csharp
public async Task Process(CancellationToken outerToken) {
    using var timeoutCts = new CancellationTokenSource(TimeSpan.FromSeconds(30));
    using var linkedCts = CancellationTokenSource.CreateLinkedTokenSource(outerToken, timeoutCts.Token);

    try {
        await DoSlowWork(linkedCts.Token);
    } catch (OperationCanceledException) when (timeoutCts.IsCancellationRequested) {
        throw new TimeoutException();
    }
}
```

The linked token is canceled when **any** of its sources is canceled. Useful for combining caller cancellation + a local timeout.

The `when (timeoutCts.IsCancellationRequested)` exception filter distinguishes "we timed out" from "caller canceled" — same OperationCanceledException type otherwise.

---

## Register — callback on cancellation

For non-Task-based work that needs to clean up on cancel:

```csharp
var registration = ct.Register(() => {
    _socket.Close();
});

try {
    var data = _socket.Read();
} finally {
    registration.Dispose();   // unregister to avoid memory leak
}
```

The callback runs **synchronously** on the thread that called `Cancel`. Be quick — don't block, don't throw.

For async cleanup, use `RegisterAsync` or check the token inside an async method.

---

## ThrowIfCancellationRequested

```csharp
ct.ThrowIfCancellationRequested();
```

If the token is canceled, throws `OperationCanceledException` with the token as a property. Cheap if not canceled (single check).

Use **before** expensive synchronous work that the inner async calls won't check the token for:

```csharp
public async Task ProcessAsync(CancellationToken ct) {
    var data = await LoadAsync(ct);

    ct.ThrowIfCancellationRequested();   // before expensive CPU work
    var transformed = HeavyCompute(data);

    await SaveAsync(transformed, ct);
}
```

---

## OperationCanceledException

When canceled work throws, it's an `OperationCanceledException`. The exception's `CancellationToken` property tells you **which** token caused it.

```csharp
try {
    await DoWorkAsync(token);
} catch (OperationCanceledException ex) when (ex.CancellationToken == token) {
    // canceled by our token
}
```

`Task.Delay(time, token)` throws `TaskCanceledException` — a subclass of OperationCanceledException — when the token cancels. Same handling.

```csharp
try { await Task.Delay(10_000, token); }
catch (OperationCanceledException) { /* canceled */ }
```

You usually catch the base class.

---

## Long-running sync work

Synchronous work doesn't `await` — but it can still check the token:

```csharp
public int Compute(IEnumerable<int> data, CancellationToken ct) {
    int sum = 0;
    foreach (var x in data) {
        ct.ThrowIfCancellationRequested();
        sum += x * x;
    }
    return sum;
}
```

For loops with many cheap iterations, check every N iterations to keep the check cost negligible:

```csharp
for (int i = 0; i < data.Length; i++) {
    if ((i & 0xFFF) == 0) ct.ThrowIfCancellationRequested();   // every 4096
    // ...
}
```

---

## Cancellation in producer-consumer

```csharp
public async IAsyncEnumerable<int> ProduceAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    for (int i = 0; i < 100; i++) {
        ct.ThrowIfCancellationRequested();
        await Task.Delay(100, ct);
        yield return i;
    }
}

// Consumer
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await foreach (var x in producer.WithCancellation(cts.Token)) {
    Process(x);
}
```

`[EnumeratorCancellation]` (.NET Core 3.0+) is critical — without it, `WithCancellation(token)` is a no-op on the iterator's internal awaits. See [§07 AsyncStreams](07-AsyncStreams.md).

---

## CancellationToken in ASP.NET Core

ASP.NET Core gives every request a `HttpContext.RequestAborted` — canceled when the client disconnects:

```csharp
app.MapGet("/slow", async (HttpContext ctx) => {
    await Task.Delay(10_000, ctx.RequestAborted);
    return "done";
});
```

Forward `RequestAborted` to all your async calls. When the client closes the connection, your work stops within the next check.

This is essential for production servers — without it, you keep churning on requests no one's reading.

---

## Disposing the CTS

```csharp
using var cts = new CancellationTokenSource();
// ... use cts ...
```

`Dispose` is **important** if you used `CancelAfter` — it cancels the timer. Without disposal, the timer keeps a strong reference to the CTS until it fires, even if no one cares anymore.

Also disposes registered callbacks. Lots of registrations + no disposal = memory leak.

---

## Internals — what's in a CancellationTokenSource

The CTS holds:
- A `_state` field (0 = not canceled, 1 = canceling, 2 = canceled).
- A list of registered callbacks.
- Optionally, a Timer for `CancelAfter`.
- A `ManualResetEvent` for `WaitHandle` (legacy interop).

`Cancel()` (or the timer firing):
1. Atomically sets state to canceling.
2. Runs all registered callbacks (synchronously by default; can be async with `RegisterCallback`).
3. Sets state to canceled.
4. Wakes any threads waiting on `WaitHandle`.

`CancellationToken` is a struct wrapping a reference to the CTS. The reference can be null (the `default` / `None` token, never cancels). Field access is fast.

`ThrowIfCancellationRequested` is essentially:
```csharp
if (_source?._state >= 1) throw new OperationCanceledException(this);
```

Very cheap when not canceled — single load + branch.

### Memory leak risk

If your CTS lives a long time and many short-lived tokens register callbacks, the registrations stack up. Each registration is held until the CTS is disposed or you `Dispose()` the returned registration.

```csharp
var ctr = ct.Register(() => Cleanup());
try { /* ... */ }
finally { ctr.Dispose(); }
```

For most code, the registration's lifetime matches the using-scope of the work. But long-running tokens (a "shut down" token held by the app) can accumulate stale registrations if you forget to dispose.

---

## Common patterns

### Method with default token

```csharp
public async Task<int> Get(CancellationToken ct = default) {
    return await _db.SelectAsync(ct);
}

// Caller can pass or omit:
var v = await svc.Get();
var v = await svc.Get(myToken);
```

`default` is `CancellationToken.None` — never cancels. Safe default.

### Combining caller token with internal timeout

```csharp
public async Task<int> GetWithTimeout(CancellationToken ct = default) {
    using var timeout = new CancellationTokenSource(TimeSpan.FromSeconds(10));
    using var linked = CancellationTokenSource.CreateLinkedTokenSource(ct, timeout.Token);
    return await DoWorkAsync(linked.Token);
}
```

Tracks both signals.

### Polling with cancellation

```csharp
public async Task PollAsync(CancellationToken ct) {
    while (!ct.IsCancellationRequested) {
        var done = await CheckCompleteAsync(ct);
        if (done) return;
        await Task.Delay(TimeSpan.FromSeconds(1), ct);
    }
}
```

`!ct.IsCancellationRequested` lets the loop exit gracefully. `Task.Delay(..., ct)` short-circuits if canceled.

### Timeout per attempt

```csharp
foreach (var attempt in Enumerable.Range(1, 3)) {
    using var attemptCts = CancellationTokenSource.CreateLinkedTokenSource(outerCt);
    attemptCts.CancelAfter(TimeSpan.FromSeconds(5));
    try {
        return await DoAsync(attemptCts.Token);
    } catch (OperationCanceledException) when (attempt < 3) {
        // try again
    }
}
```

Linked outer cancellation + per-attempt timeout.

---

## Common bugs

### Not forwarding the token

```csharp
public async Task<int> M(CancellationToken ct) {
    await Task.Delay(10_000);   // ⚠ — doesn't honor ct
    return 0;
}
```

Should be `Task.Delay(10_000, ct)`. If you don't forward, the inner await ignores cancellation.

### Catching OperationCanceledException too broadly

```csharp
try { /* ... */ }
catch (Exception ex) { _log.LogError(ex, "failed"); }   // ⚠ catches cancellation as "error"
```

Cancellation is usually expected — not an error. Catch it separately:

```csharp
try { /* ... */ }
catch (OperationCanceledException) when (ct.IsCancellationRequested) { /* expected */ }
catch (Exception ex) { _log.LogError(ex, "failed"); }
```

### Forgetting to dispose CTS

```csharp
var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));   // ⚠ — no using
await DoAsync(cts.Token);
// CTS lives until GC. Timer keeps it alive.
```

Use `using var cts = new ...`.

### Token from a different source

```csharp
public async Task Outer(CancellationToken outer) {
    using var inner = new CancellationTokenSource();
    await DoAsync(inner.Token);   // ⚠ — outer cancellation ignored
}
```

Forward outer:
```csharp
using var linked = CancellationTokenSource.CreateLinkedTokenSource(outer);
await DoAsync(linked.Token);
```

### `ct.IsCancellationRequested` without throwing

```csharp
if (ct.IsCancellationRequested) return;   // returns success
```

Sometimes correct (you successfully partial-completed). Often you want to throw — `ct.ThrowIfCancellationRequested()` is the convention. Document the semantic.

---

## Performance

- `CancellationToken` is a struct — passing it is essentially free.
- `ct.IsCancellationRequested` — single field load + branch.
- `ct.ThrowIfCancellationRequested()` — same as above plus the exception path (rare).
- `ct.Register(callback)` — allocates a registration (~80 bytes). Dispose to free.
- `CancellationTokenSource.CancelAfter` — allocates a Timer internally. Dispose the CTS to dispose the timer.

For ultra-hot paths, you might pre-check `ct.CanBeCanceled` (false for `CancellationToken.None`) to avoid bothering when no cancellation is possible:

```csharp
if (ct.CanBeCanceled) ct.ThrowIfCancellationRequested();
```

Usually not worth the micro-optimization.

---

## Summary

- Every async method should accept a `CancellationToken ct = default`.
- Forward it to every inner async call.
- Check `ThrowIfCancellationRequested` before expensive sync work.
- Caller controls cancellation via `CancellationTokenSource`.
- Use `CancelAfter` for timeouts.
- Use `CreateLinkedTokenSource` to combine multiple cancellation reasons.
- Catch `OperationCanceledException` to handle cancellation cleanly.
- `using var cts = ...` to clean up timers and registrations.

→ Next: [06-ConfigureAwait.md](06-ConfigureAwait.md)
