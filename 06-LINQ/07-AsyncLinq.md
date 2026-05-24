# Async LINQ — `IAsyncEnumerable<T>`

## What it is

`IAsyncEnumerable<T>` is the async cousin of `IEnumerable<T>`. Each `MoveNext` returns a `ValueTask<bool>`. The whole sequence can yield items asynchronously — pulling from a database, a network stream, or a long-running computation — without buffering everything in memory.

```csharp
public async IAsyncEnumerable<int> CountAsync() {
    for (int i = 0; i < 5; i++) {
        await Task.Delay(100);
        yield return i;
    }
}

await foreach (var n in CountAsync()) {
    Console.WriteLine(n);   // 0, 1, 2, 3, 4 — one per 100ms
}
```

LINQ operators for `IAsyncEnumerable<T>` live in the **`System.Linq.Async`** NuGet package (or the `Microsoft.Extensions.Linq.Async` extensions). They're not in the BCL by default.

`IAsyncEnumerable<T>` arrived in C# 8 (2019); EF Core, gRPC streaming, and `Channel<T>` all expose data as `IAsyncEnumerable<T>`.

---

## Why it exists

A long-running data source (database streaming, large API result, file with backpressure) needs:
- **Async** — don't block a thread while waiting for I/O.
- **Sequence** — items arrive one at a time, possibly endlessly.

`Task<IEnumerable<T>>` doesn't capture this — you'd have to materialize everything before returning. `IObservable<T>` is push-based and harder to compose with sequential code. `IAsyncEnumerable<T>` is the pull-based async sequence.

---

## Producing an async sequence

```csharp
public async IAsyncEnumerable<string> ReadLinesAsync(string path) {
    using var reader = new StreamReader(path);
    string? line;
    while ((line = await reader.ReadLineAsync()) is not null) {
        yield return line;
    }
}
```

`async IAsyncEnumerable<T>` with `yield return` and `await` — combine an iterator with async.

The method:
1. Returns immediately when called (no work done yet).
2. The body runs when consumer calls `MoveNextAsync`.
3. Yields one item per iteration; awaits between as needed.

---

## Consuming with `await foreach`

```csharp
await foreach (var line in ReadLinesAsync("big.txt")) {
    Process(line);
}
```

The `await foreach` is sugar for:

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

Sequential async consumption, one item at a time. Backpressure flows naturally — producer doesn't run faster than consumer.

---

## Cancellation: `[EnumeratorCancellation]`

The standard pattern for cancellable async iterators:

```csharp
public async IAsyncEnumerable<int> CountAsync(
    [EnumeratorCancellation] CancellationToken cancellationToken = default)
{
    for (int i = 0; i < 100; i++) {
        await Task.Delay(100, cancellationToken);
        yield return i;
    }
}

// Consumer passes a token:
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(1));
await foreach (var n in CountAsync(cts.Token)) {
    Console.WriteLine(n);
}
// Cancels after ~1 second.
```

The `[EnumeratorCancellation]` attribute tells the compiler "use this parameter as the iterator's cancellation token." When the consumer calls `WithCancellation(token)` on the enumerable, the attribute-marked parameter is replaced.

Without the attribute, `WithCancellation` calls have no effect on the iterator's behavior — the consumer's token is ignored.

```csharp
// Caller pattern — supply a token without the producer signature explicitly taking one:
await foreach (var n in CountAsync().WithCancellation(cts.Token)) { /* ... */ }
```

The framework combines that token with the parameter-marked one.

---

## Async LINQ operators

Install `System.Linq.Async` (or use `Microsoft.Extensions.Linq.Async`):

```csharp
using System.Linq;   // namespaces extend it

await foreach (var x in source
    .Where(x => x.IsActive)
    .Select(x => x.Name)
    .OrderBy(x => x)) {
    Console.WriteLine(x);
}
```

You get LINQ over async sequences: `Where`, `Select`, `Take`, `Skip`, `GroupBy`, `Join`, `Aggregate`, etc. The package adds equivalents for all the standard LINQ operators.

There are also "Await" variants for predicates that are themselves async:

```csharp
await foreach (var u in users.WhereAwait(async u => await IsActiveAsync(u))) { ... }
```

vs the sync predicate form:
```csharp
await foreach (var u in users.Where(u => u.IsActive)) { ... }
```

Use `WhereAwait` when the predicate is async; `Where` when it isn't.

---

## Materializing async sequences

```csharp
List<int> all = await source.ToListAsync();
int[] arr = await source.ToArrayAsync();
Dictionary<int, string> dict = await source.ToDictionaryAsync(x => x.Id, x => x.Name);

int total = await source.CountAsync();
int sum = await source.SumAsync(x => x.Value);
bool any = await source.AnyAsync();
int first = await source.FirstAsync();
```

These force enumeration to complete. After `ToListAsync()`, you have a regular `List<T>` you can iterate synchronously.

---

## `IAsyncEnumerable<T>` in EF Core

EF Core's `IQueryable<T>` exposes both sync and async sequences:

```csharp
var users = db.Users.Where(u => u.IsActive);

// Materialize all
var list = await users.ToListAsync();

// Stream
await foreach (var u in users.AsAsyncEnumerable()) {
    Process(u);   // one row at a time, never buffered fully
}
```

`AsAsyncEnumerable()` converts the IQueryable into an `IAsyncEnumerable<T>` that streams rows from the database connection. For large result sets where you don't want to load everything into memory.

---

## Custom async operators

Write your own iterator:

```csharp
public static async IAsyncEnumerable<T> ThrottleAsync<T>(
    this IAsyncEnumerable<T> source,
    TimeSpan delay,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var x in source.WithCancellation(ct)) {
        yield return x;
        await Task.Delay(delay, ct);
    }
}

// Yield one item per second
await foreach (var x in FastSource().ThrottleAsync(TimeSpan.FromSeconds(1))) {
    Console.WriteLine(x);
}
```

Same iterator mechanics as sync LINQ, with `await` and `await foreach` sprinkled in.

---

## Internals — what the compiler does

For:

```csharp
public async IAsyncEnumerable<int> CountAsync() {
    for (int i = 0; i < 5; i++) {
        await Task.Delay(100);
        yield return i;
    }
}
```

The compiler generates a state machine implementing **both** `IAsyncEnumerable<int>` and `IAsyncEnumerator<int>`. It tracks:

- The current state (which `yield return` or `await` it's at).
- The result of the last operation.
- The Task being awaited (if any).
- The current item (for `Current`).

`MoveNextAsync()` returns a `ValueTask<bool>`:
- `true` if there's more (and `Current` is now set).
- `false` if done.

If the iterator hits an `await`, the state machine returns the pending ValueTask without advancing. The consumer can `await` it.

Performance:
- One state-machine class per call site (shared structure, per-call instance).
- Per-MoveNextAsync: typically one allocation for the ValueTask (or zero if the underlying task is synchronous via caching).
- The state machine handles both iteration AND async — no separate Task wrapper.

For very high-throughput async sequences, write the iterator carefully and use `ValueTask` where possible.

---

## Common patterns

### Streaming HTTP response

```csharp
public async IAsyncEnumerable<string> StreamLinesAsync(
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

Streams a multi-MB response without loading it all into memory.

### Producer-consumer with Channel

```csharp
public async IAsyncEnumerable<T> ReadFromChannel<T>(
    ChannelReader<T> reader,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    await foreach (var item in reader.ReadAllAsync(ct)) {
        yield return item;
    }
}
```

`Channel<T>` (from `System.Threading.Channels`) is the canonical async producer-consumer queue. It already implements this — `reader.ReadAllAsync()` returns an `IAsyncEnumerable<T>`.

### Async aggregation

```csharp
public async Task<decimal> SumAsync(this IAsyncEnumerable<decimal> source) {
    decimal total = 0;
    await foreach (var x in source) total += x;
    return total;
}
```

Equivalent operators ship in `System.Linq.Async` for the standard cases.

---

## Async vs sync LINQ — when to choose

| Use sync LINQ when | Use async LINQ when |
|---|---|
| Data is in memory | Data is streamed from I/O |
| Operations are CPU-bound | Operations are I/O-bound |
| Latency per operator is negligible | Each operator might block on network/disk |
| Source is `List<T>`, array, etc. | Source is `IAsyncEnumerable<T>` (EF Core, Channels, etc.) |

Don't `await` for the sake of it. Don't sync-over-async (`.GetAwaiter().GetResult()`).

---

## Common bugs

- **Forgetting `[EnumeratorCancellation]`** — `WithCancellation` calls become no-ops; iterator runs to completion even on cancel.
- **Mixing sync `foreach` with `IAsyncEnumerable<T>`** — compile error. Use `await foreach`.
- **`ToListAsync` from a deferred query** — runs the entire query to completion. Use this when you want all results; otherwise stream.
- **Inadvertent capture in async iterators** — same closure caveats as sync.
- **Returning `IAsyncEnumerable<T>` from a public method** — caller may not realize it's deferred. Document clearly.

---

## Performance

- `IAsyncEnumerable<T>` is heavier than `IEnumerable<T>` — async state machine + ValueTask per MoveNext.
- For purely synchronous data, use the sync interface.
- For I/O-bound or streaming data, async is the right tool — the latency hides the overhead.
- `WhereAwait`, `SelectAwait` add per-item awaits — significant overhead. Use `Where`/`Select` when the predicate/selector is sync.

---

## When to use

✓ Streaming I/O (network, file).
✓ Database row streaming (EF Core `AsAsyncEnumerable`).
✓ gRPC server-streaming.
✓ Producer-consumer with Channel.
✓ Long-running computations that yield gradually.

✗ Pure in-memory data — use regular LINQ.
✗ Single-result async operations — just use `Task<T>`.
✗ When the producer is fast and consumer wants batched access — buffer first.

→ Next: [08-IQueryable.md](08-IQueryable.md)
