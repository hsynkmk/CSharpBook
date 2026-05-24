# Async Streams — IAsyncEnumerable&lt;T&gt;

## What it is

`IAsyncEnumerable<T>` is the async counterpart to `IEnumerable<T>`. Each item is yielded **asynchronously** — perfect for streaming data from a network, database, or any source where items arrive over time.

```csharp
public async IAsyncEnumerable<int> CountAsync([EnumeratorCancellation] CancellationToken ct = default) {
    for (int i = 0; i < 10; i++) {
        await Task.Delay(100, ct);
        yield return i;
    }
}

await foreach (var n in CountAsync()) {
    Console.WriteLine(n);   // 0, 1, 2, ... one per 100ms
}
```

Added in C# 8 (2019). The streaming counterpart of `async/await`. Used everywhere:
- EF Core's `AsAsyncEnumerable()` for row-by-row streaming.
- gRPC server streaming.
- `System.IO.Pipelines`.
- `Channel<T>.Reader.ReadAllAsync()`.

---

## Why it exists

A `Task<List<T>>` materializes the entire result before yielding anything. For "stream 1M rows" or "read file line by line," that's wasteful. `IAsyncEnumerable<T>` yields **one item at a time**, asynchronously.

Pre-C# 8, you'd use:
- A `Channel<T>` and write a producer/consumer manually.
- `IObservable<T>` from Rx.NET (push-based, very different semantics).
- A callback API.

None composed well with regular async code. `IAsyncEnumerable<T>` + `await foreach` made streaming async naturally readable.

---

## Producing an async stream

### `async IAsyncEnumerable<T>` with `yield return`

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string path,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var reader = new StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null) {
        yield return line;
    }
}
```

The method:
1. Returns immediately when called (no work done yet — like an iterator).
2. The body runs when the consumer calls `MoveNextAsync`.
3. Yields one item per iteration; awaits between as needed.

`async IAsyncEnumerable<T>` + `yield return` + `await` is the modern streaming idiom.

### Without `async` and `yield` (rarely needed)

You can implement `IAsyncEnumerable<T>` and `IAsyncEnumerator<T>` manually:

```csharp
public class Pulser : IAsyncEnumerable<int> {
    public async IAsyncEnumerator<int> GetAsyncEnumerator(CancellationToken ct = default) {
        for (int i = 0; i < 10; i++) {
            await Task.Delay(100, ct);
            yield return i;
        }
    }
}
```

(Technically still uses async + yield; the compiler generates the state machine.)

For truly bespoke streaming, implement the interfaces directly. Mostly not needed.

---

## Consuming with `await foreach`

```csharp
await foreach (var line in ReadLinesAsync("big.txt")) {
    Process(line);
}
```

Sugar for:

```csharp
var enumerator = source.GetAsyncEnumerator();
try {
    while (await enumerator.MoveNextAsync()) {
        var line = enumerator.Current;
        Process(line);
    }
} finally {
    await enumerator.DisposeAsync();
}
```

Per iteration:
1. `MoveNextAsync` advances. If sync-completion fast path, no actual await.
2. `Current` exposes the item.
3. Loop body runs.

Sequential. Backpressure: the producer waits for the consumer to ask for the next item.

---

## Cancellation — `[EnumeratorCancellation]`

The token-passing pattern for async iterators is subtle. The consumer's pattern:

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await foreach (var x in producer.WithCancellation(cts.Token)) {
    Process(x);
}
```

`.WithCancellation(token)` calls `GetAsyncEnumerator(token)` under the hood. But your iterator method has its own `CancellationToken` parameter (usually):

```csharp
public async IAsyncEnumerable<int> Produce([EnumeratorCancellation] CancellationToken ct = default) {
    // ct is the token from WithCancellation (or default if none provided)
}
```

The `[EnumeratorCancellation]` attribute tells the compiler "wire this parameter to whatever `WithCancellation` provides."

Without the attribute, your method's `ct` is whatever the caller passed in the original `Produce(token)` call — and `WithCancellation` does nothing for the iterator's internal awaits.

**Always use `[EnumeratorCancellation]`** on the token parameter of async iterators. Otherwise consumer-side cancellation is silently ignored inside the iterator.

---

## LINQ over async streams

The `System.Linq.Async` NuGet package (or `Microsoft.Extensions.Linq.Async`) provides LINQ operators for `IAsyncEnumerable<T>`:

```csharp
using System.Linq;

await foreach (var u in users
    .Where(u => u.IsActive)
    .Select(u => u.Email)
    .OrderBy(s => s)
    .Take(10))
{
    Console.WriteLine(u);
}
```

Plus materialization:
```csharp
List<string> emails = await users.Where(u => u.IsActive).Select(u => u.Email).ToListAsync();
int count = await users.CountAsync();
bool any = await users.AnyAsync();
```

For predicates that themselves are async, there are `*Await` variants:
```csharp
await foreach (var u in users.WhereAwait(async u => await IsActiveAsync(u))) { ... }
```

---

## EF Core integration

EF Core's `DbSet<T>` and `IQueryable<T>` support async iteration:

```csharp
await foreach (var user in db.Users.Where(u => u.IsActive).AsAsyncEnumerable()) {
    await Process(user);   // one row at a time from the DB
}
```

vs:

```csharp
var users = await db.Users.Where(u => u.IsActive).ToListAsync();
// All rows in memory before processing any
```

For 10M rows, the streaming form uses constant memory. The list form needs 10M-row memory.

---

## Backpressure

Async streams have **natural backpressure**: the producer is paused between yields until the consumer calls `MoveNextAsync` again.

```csharp
async IAsyncEnumerable<int> Slow() {
    for (int i = 0; i < 1000; i++) {
        await Task.Delay(100);   // simulate slow producer
        yield return i;
    }
}

await foreach (var x in Slow()) {
    await Task.Delay(2000);   // simulate slow consumer
    Process(x);
}
```

The consumer is slower; the producer waits. Total time: 1000 × 2000 = 2000 seconds (consumer-bound).

If you want a **buffered producer** (producer runs ahead, consumer catches up), use a `Channel<T>`:

```csharp
var ch = Channel.CreateBounded<int>(100);
_ = Task.Run(async () => {
    for (int i = 0; i < 1000; i++) {
        await Task.Delay(100);
        await ch.Writer.WriteAsync(i);   // blocks if channel full (100 items)
    }
    ch.Writer.Complete();
});

await foreach (var x in ch.Reader.ReadAllAsync()) {
    Process(x);
}
```

Producer can write up to 100 ahead. Then waits. See [§13 Channels](13-Channels.md).

---

## Common patterns

### Streaming HTTP body

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(
    string url,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    using var client = new HttpClient();
    using var resp = await client.GetAsync(url, HttpCompletionOption.ResponseHeadersRead, ct);
    using var stream = await resp.Content.ReadAsStreamAsync(ct);
    using var reader = new StreamReader(stream);

    string? line;
    while ((line = await reader.ReadLineAsync(ct)) is not null) {
        yield return line;
    }
}
```

`HttpCompletionOption.ResponseHeadersRead` is the key — don't buffer the whole body.

### Streaming DB rows

```csharp
public async IAsyncEnumerable<User> StreamUsersAsync(
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var user in _db.Users.AsAsyncEnumerable().WithCancellation(ct)) {
        yield return user;
    }
}
```

EF Core's underlying reader streams rows.

### Server-Sent Events (SSE)

```csharp
app.MapGet("/events", async (HttpContext ctx, CancellationToken ct) => {
    ctx.Response.Headers.ContentType = "text/event-stream";
    await foreach (var update in MonitorAsync(ct)) {
        await ctx.Response.WriteAsync($"data: {update}\n\n", ct);
        await ctx.Response.Body.FlushAsync(ct);
    }
});

async IAsyncEnumerable<string> MonitorAsync([EnumeratorCancellation] CancellationToken ct) {
    while (!ct.IsCancellationRequested) {
        await Task.Delay(1000, ct);
        yield return $"{DateTime.UtcNow:O}";
    }
}
```

Stream events to the client.

### Pagination via async stream

```csharp
public async IAsyncEnumerable<User> AllPagesAsync([EnumeratorCancellation] CancellationToken ct = default) {
    int page = 1;
    while (true) {
        var batch = await FetchPage(page, ct);
        if (batch.Count == 0) yield break;
        foreach (var u in batch) yield return u;
        page++;
    }
}

await foreach (var u in AllPagesAsync()) Process(u);
```

The consumer sees one continuous sequence; under the hood it's paginated.

### Combining with `IAsyncEnumerable` LINQ

```csharp
var top10 = await users.AllPagesAsync()
    .Where(u => u.IsActive)
    .OrderByDescending(u => u.JoinedAt)
    .Take(10)
    .ToListAsync();
```

(Requires `System.Linq.Async` for `OrderByDescending` over IAsyncEnumerable — it must materialize, of course.)

---

## Internals — what the compiler generates

`async IAsyncEnumerable<T>` with `yield return` generates a state machine implementing **both** `IAsyncEnumerable<T>` and `IAsyncEnumerator<T>`. Approximately:

```csharp
private sealed class StateMachine : IAsyncEnumerable<T>, IAsyncEnumerator<T>, IAsyncStateMachine {
    private int _state;
    private T _current;
    private bool _disposed;
    private ManualResetValueTaskSourceCore<bool> _moveNextSource;
    // ... captured locals ...

    public IAsyncEnumerator<T> GetAsyncEnumerator(CancellationToken ct = default) {
        if (canReuse) {
            _state = 0;
            return this;
        }
        return new StateMachine { _state = 0, /* clone */ };
    }

    public ValueTask<bool> MoveNextAsync() {
        // resume state machine; return ValueTask<bool>
    }

    public T Current => _current;

    public async ValueTask DisposeAsync() {
        // cleanup
    }
}
```

Key bits:
- `MoveNextAsync` returns `ValueTask<bool>` — `true` if there's more, `false` if done.
- The state machine handles both async semantics AND iterator state — combined.
- `Current` exposes the most recent yield.
- `DisposeAsync` is async-friendly cleanup.

The struct is heap-allocated on first iteration (the iterator IS the enumerator after the first call).

### ValueTask&lt;bool&gt; for MoveNextAsync

The use of `ValueTask` is intentional — many iterations complete synchronously (already-buffered data) and don't need to allocate a Task per step. For million-item streams, this matters.

---

## Common bugs

### Forgetting `[EnumeratorCancellation]`

```csharp
public async IAsyncEnumerable<int> Produce(CancellationToken ct = default) {
    // no attribute
}

// Consumer
await foreach (var x in producer.WithCancellation(myCts.Token)) { ... }
```

`WithCancellation` calls `GetAsyncEnumerator(token)`, but without the attribute, that token doesn't reach your `ct` parameter. Your awaits inside the iterator ignore the consumer's cancellation.

Always add the attribute.

### Mixing `IAsyncEnumerable<T>` and synchronous foreach

```csharp
foreach (var x in producer) { ... }   // ⚠ — compile error
```

`foreach` (no await) doesn't work on `IAsyncEnumerable<T>`. Must be `await foreach`.

### Returning IAsyncEnumerable from a method without async

```csharp
public IAsyncEnumerable<int> Produce() {
    return SomeQuery.AsAsyncEnumerable();   // OK
}
```

This is fine — the method itself is sync; it returns an async stream. Don't add `async` unless you actually need to use `await` or `yield return` in the method body.

### Multi-pass over an async stream

```csharp
var stream = ProduceAsync();
await foreach (var x in stream) { ... }
await foreach (var x in stream) { ... }   // ⚠ — second enumeration's behavior is iterator-dependent
```

Most async iterators are single-pass. The second `await foreach` may produce nothing or undefined behavior. Materialize first if you need multi-pass:

```csharp
var list = await stream.ToListAsync();
```

### Long-held producer state

```csharp
public async IAsyncEnumerable<int> Produce() {
    using var resource = AcquireExpensive();
    for (int i = 0; i < 1000; i++) yield return i;
}
```

`resource` is alive until the consumer either fully iterates or disposes the enumerator. If the consumer breaks out early without `await using`, you might leak.

Best practice for consumers: use `await using` if you need explicit disposal scope.

---

## Performance

- `IAsyncEnumerable<T>` adds ~one allocation per iteration in the worst case.
- `ValueTask<bool>` for `MoveNextAsync` avoids Task allocation for sync-completion fast paths.
- For purely synchronous data, prefer `IEnumerable<T>` — async stream's overhead is unnecessary.
- For I/O-bound data with possibly-cached items, async streams are excellent.
- Channels with bounded buffering can outperform async streams when producer is faster than consumer.

---

## Summary

- `IAsyncEnumerable<T>` for streaming async data, item by item.
- `await foreach` to consume sequentially.
- `[EnumeratorCancellation]` for proper cancellation propagation.
- Use for: streaming I/O, EF Core row streaming, server-sent events, paginated APIs.
- Don't use for: in-memory data (just IEnumerable), single-result async operations (just Task).

→ Next: [08-AsyncDisposable.md](08-AsyncDisposable.md)
