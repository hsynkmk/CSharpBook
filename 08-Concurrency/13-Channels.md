# Channels — System.Threading.Channels

## What it is

`Channel<T>` is the modern **async producer-consumer** primitive. Producers write items via `ChannelWriter<T>.WriteAsync`; consumers read via `ChannelReader<T>.ReadAsync` or `ReadAllAsync` (returns `IAsyncEnumerable<T>`).

```csharp
var channel = Channel.CreateBounded<int>(100);   // bounded capacity 100

// Producer
_ = Task.Run(async () => {
    for (int i = 0; i < 1000; i++) {
        await channel.Writer.WriteAsync(i);   // backpressure: blocks (asynchronously) if full
    }
    channel.Writer.Complete();
});

// Consumer
await foreach (var item in channel.Reader.ReadAllAsync()) {
    Console.WriteLine(item);
}
```

Channels were added in .NET Core 2.1 (2018) as the async-first replacement for `BlockingCollection<T>`. Used widely:
- Background work queues.
- Producer-consumer pipelines.
- Decoupling fast producers from slow consumers (with backpressure).
- Internal queues in ASP.NET Core's rate limiter, gRPC, SignalR.

---

## Why they exist

Pre-Channel, async producer-consumer was awkward:
- `BlockingCollection<T>` blocks threads (not async-friendly).
- `ConcurrentQueue<T>` + polling wastes CPU.
- Hand-rolled `TaskCompletionSource` based queues — error-prone.

Channel offers:
- **True async semantics** — `WriteAsync` and `ReadAsync` release the thread while waiting.
- **Backpressure** — bounded channels block the writer (asynchronously) when full.
- **Completion signaling** — producers call `Complete()`; consumers observe end-of-stream.
- **Multiple readers / writers** — configurable for single or multiple.
- **Cancellation** — every async method takes a CancellationToken.

It's the right tool for any async producer-consumer pattern.

---

## Creating channels

### Unbounded

```csharp
var channel = Channel.CreateUnbounded<int>();
```

No capacity limit — producer never blocks. Risk: memory grows without bound if consumer is slow.

### Bounded

```csharp
var channel = Channel.CreateBounded<int>(100);
```

Capacity 100. Producer blocks (asynchronously) when full. Backpressure.

### Options

```csharp
var channel = Channel.CreateBounded<int>(new BoundedChannelOptions(100) {
    FullMode = BoundedChannelFullMode.Wait,    // default — block writer
    SingleReader = true,                        // optimization if only one consumer
    SingleWriter = true,                        // optimization if only one producer
    AllowSynchronousContinuations = false,      // safer default
});
```

`SingleReader` / `SingleWriter` enable internal optimizations — use them when accurate.

`FullMode`:
- `Wait` (default) — writer awaits until space available.
- `DropOldest` — silently drop the oldest item to make room.
- `DropNewest` — silently drop the newest in the channel.
- `DropWrite` — drop the new item being written (Write returns false).

`AllowSynchronousContinuations = false` (recommended) — writes don't run readers' continuations on the writer's thread. Avoids surprise re-entrance.

---

## Writing

```csharp
ChannelWriter<int> writer = channel.Writer;

// Async write — awaits if bounded and full
await writer.WriteAsync(42, ct);

// Try without waiting
if (writer.TryWrite(42)) { /* succeeded */ }
else { /* would block (or rejected by FullMode) */ }

// Wait until space is available
if (await writer.WaitToWriteAsync(ct)) {
    writer.TryWrite(42);   // now there's space — fast path
}

// Done producing
writer.Complete();           // no more writes
writer.Complete(new Exception("..."));   // signal an error
```

After Complete, further writes throw.

---

## Reading

```csharp
ChannelReader<int> reader = channel.Reader;

// Async read — awaits until item available or channel completes
int item = await reader.ReadAsync(ct);   // throws ChannelClosedException when done

// Try without waiting
if (reader.TryRead(out var item2)) { /* got one */ }

// Wait until item available
if (await reader.WaitToReadAsync(ct)) {
    reader.TryRead(out var item3);   // fast path
}

// Iterate everything
await foreach (var x in reader.ReadAllAsync(ct)) {
    Process(x);
}

// Completion task
await reader.Completion;   // completes when channel done + drained
```

`ReadAllAsync` is the modern way to consume — clean async foreach syntax. It naturally stops when the channel is complete and empty.

---

## A complete example

```csharp
public async Task RunPipelineAsync(IEnumerable<string> urls, CancellationToken ct = default) {
    var channel = Channel.CreateBounded<string>(new BoundedChannelOptions(100) {
        SingleWriter = true,
        SingleReader = false,
    });

    // Producer
    var producer = Task.Run(async () => {
        try {
            foreach (var url in urls) {
                await channel.Writer.WriteAsync(url, ct);
            }
        } finally {
            channel.Writer.Complete();
        }
    }, ct);

    // Consumers (4 workers)
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () => {
        await foreach (var url in channel.Reader.ReadAllAsync(ct)) {
            await ProcessAsync(url, ct);
        }
    }, ct));

    await Task.WhenAll(consumers.Append(producer));
}
```

Pattern:
1. Create a bounded channel.
2. One producer task writes URLs, calls `Complete()` at the end (in `finally` so it runs on exception too).
3. Multiple consumer tasks read concurrently — `ReadAllAsync` distributes items across them.
4. WhenAll waits for everyone.

Backpressure: if consumers are slow, the channel fills, producer awaits in `WriteAsync`. No memory growth.

---

## Channel patterns

### Fan-out (one producer, many consumers)

```csharp
var channel = Channel.CreateBounded<Work>(100);

// 1 producer
_ = Task.Run(async () => {
    foreach (var work in source) await channel.Writer.WriteAsync(work);
    channel.Writer.Complete();
});

// N consumers
var consumers = Enumerable.Range(0, Environment.ProcessorCount).Select(_ => Task.Run(async () => {
    await foreach (var work in channel.Reader.ReadAllAsync()) ProcessWork(work);
}));

await Task.WhenAll(consumers);
```

### Fan-in (many producers, one consumer)

```csharp
var channel = Channel.CreateBounded<Event>(1000);

// N producers
var producers = Enumerable.Range(0, 4).Select(workerId => Task.Run(async () => {
    foreach (var e in source.Skip(workerId).TakeEvery(4))
        await channel.Writer.WriteAsync(e);
}));

// 1 consumer that aggregates
_ = Task.Run(async () => {
    await Task.WhenAll(producers);
    channel.Writer.Complete();   // wait for all producers, then complete
});

await foreach (var e in channel.Reader.ReadAllAsync()) Aggregate(e);
```

The completion coordination is the tricky bit — only one of the producers should call Complete after all are done. A separate task awaits them and signals.

### Pipeline (chained channels)

```csharp
var step1Out = Channel.CreateBounded<Item>(100);
var step2Out = Channel.CreateBounded<Result>(100);

// Step 1: read from source, transform, write to step1Out
_ = Task.Run(async () => {
    foreach (var s in source) {
        var t = Transform1(s);
        await step1Out.Writer.WriteAsync(t);
    }
    step1Out.Writer.Complete();
});

// Step 2: read from step1Out, transform, write to step2Out
_ = Task.Run(async () => {
    await foreach (var t in step1Out.Reader.ReadAllAsync()) {
        var r = await Transform2Async(t);
        await step2Out.Writer.WriteAsync(r);
    }
    step2Out.Writer.Complete();
});

// Step 3: read from step2Out, write to output
await foreach (var r in step2Out.Reader.ReadAllAsync()) Output(r);
```

Like Unix pipes: `source | step1 | step2 | sink`. Each step concurrent. Bounded channels provide backpressure between stages.

---

## Completion semantics

`Writer.Complete()` signals "no more writes coming." Items in flight finish flowing through. Readers' `ReadAllAsync` exits naturally when the channel is empty AND complete.

`reader.Completion` is a Task that completes when:
- The writer called Complete (or Complete(exception)) AND
- All items have been read.

```csharp
await reader.Completion;   // wait until done
```

Catches errors from `Complete(exception)`.

If you forget to call Complete, readers wait forever (until cancellation). Always call Complete in a finally block:

```csharp
try {
    foreach (var x in source) await writer.WriteAsync(x);
} finally {
    writer.Complete();
}
```

For exceptions:
```csharp
try { /* writer logic */ }
catch (Exception ex) { writer.Complete(ex); throw; }
finally { writer.TryComplete(); }   // safe — does nothing if already completed
```

---

## Backpressure vs FullMode

Bounded channels default to `FullMode.Wait` — producer awaits when full. This gives **backpressure**: the producer goes only as fast as the consumer.

Alternatives:
- `DropOldest` — drop the oldest item in the channel. Good for "latest data wins" scenarios (sensor readings, market data).
- `DropNewest` — drop the newest item still in the channel. Less common.
- `DropWrite` — `WriteAsync` returns immediately without writing the new item. `TryWrite` returns false.

```csharp
var ch = Channel.CreateBounded<Sensor>(new BoundedChannelOptions(10) {
    FullMode = BoundedChannelFullMode.DropOldest
});
// If consumer is slow, oldest data is dropped — we always have the freshest 10 readings
```

For "I don't want to lose data, slow the producer" — default Wait is correct.

---

## Cancellation

Every async method accepts a CancellationToken:

```csharp
using var cts = new CancellationTokenSource(TimeSpan.FromSeconds(5));
await channel.Writer.WriteAsync(item, cts.Token);   // throws OCE if canceled
await foreach (var x in channel.Reader.ReadAllAsync(cts.Token)) { ... }
```

When the token cancels:
- Pending WriteAsync calls throw OperationCanceledException.
- Pending ReadAsync / ReadAllAsync throws OperationCanceledException.
- The channel itself isn't completed — others can keep using it.

For "shut down the channel entirely," call `Writer.Complete()`.

---

## Internals — how Channels are built

Channel uses:
- A **queue** (deque) for items.
- A **lock** for short critical sections (or lock-free where possible).
- **`AsyncOperation<T>` pools** for waiters — reusable state machines for awaiting reads and writes.
- **`ValueTask<T>`** for the async results — zero allocation in the synchronous fast path.

When a Read finds the channel non-empty, it returns the item synchronously (no Task allocation). When empty, it allocates an AsyncOperation, registers as a waiter, returns its ValueTask.

When a Write to an empty channel finds a pending reader, it directly hands the item to the reader (skipping the queue) — fastest path.

Bounded channels track a count and apply the FullMode policy when at capacity.

The result: **very fast** per-operation cost. Channels are competitive with hand-tuned lock-free queues for many workloads.

### SingleReader / SingleWriter optimizations

When you promise only one reader (or writer), the runtime can skip some internal locking and use faster code paths. Always set these correctly — it's a measurable win.

---

## Channels vs alternatives

| | Channel | BlockingCollection | ConcurrentQueue + Semaphore | TPL Dataflow |
|---|---|---|---|---|
| Async-friendly | ✓ | sync only | partial | ✓ |
| Backpressure | ✓ | ✓ (bounded) | manual | ✓ |
| Multiple readers | ✓ | ✓ | yes | ✓ |
| Completion signaling | ✓ | ✓ | manual | ✓ |
| Composability | bare bones | bare bones | bare bones | high (linked blocks) |
| Memory cost | low | medium | low | medium |
| API complexity | simple | medium | manual | complex |

**Channels are the modern default** for async producer-consumer. TPL Dataflow (`System.Threading.Tasks.Dataflow`) is more featureful (linked blocks, fork/join, transform stages) but more complex. Reach for it if Channels feel too low-level.

---

## Common patterns

### Long-running background queue

```csharp
public class BackgroundQueue {
    private readonly Channel<Func<CancellationToken, Task>> _queue =
        Channel.CreateBounded<Func<CancellationToken, Task>>(new BoundedChannelOptions(1000) {
            FullMode = BoundedChannelFullMode.Wait
        });

    public ValueTask EnqueueAsync(Func<CancellationToken, Task> work, CancellationToken ct = default) =>
        _queue.Writer.WriteAsync(work, ct);

    public async Task RunAsync(CancellationToken ct) {
        await foreach (var work in _queue.Reader.ReadAllAsync(ct)) {
            try { await work(ct); }
            catch (Exception ex) { _log.LogError(ex, "background work failed"); }
        }
    }
}
```

Register as a `BackgroundService` in ASP.NET Core. Endpoints enqueue work; the service drains.

### Multi-stage pipeline

```csharp
var input = Channel.CreateBounded<RawData>(100);
var stage1 = Channel.CreateBounded<Parsed>(100);
var stage2 = Channel.CreateBounded<Enriched>(100);

// Stages run concurrently, each transforming and forwarding
async Task Stage1Worker() {
    await foreach (var raw in input.Reader.ReadAllAsync()) {
        await stage1.Writer.WriteAsync(Parse(raw));
    }
}
// ... etc
```

Each stage on its own task; channels glue them together with backpressure.

### Buffered "every N ms" batcher

```csharp
public async IAsyncEnumerable<List<T>> Batch<T>(
    IAsyncEnumerable<T> source,
    int maxBatchSize,
    TimeSpan maxWait,
    [EnumeratorCancellation] CancellationToken ct = default)
{
    var batch = new List<T>(maxBatchSize);
    var sw = Stopwatch.StartNew();
    await foreach (var item in source.WithCancellation(ct)) {
        batch.Add(item);
        if (batch.Count >= maxBatchSize || sw.Elapsed >= maxWait) {
            yield return batch;
            batch = new(maxBatchSize);
            sw.Restart();
        }
    }
    if (batch.Count > 0) yield return batch;
}
```

Channels often feed into batchers like this.

---

## Common bugs

### Forgetting Complete

```csharp
foreach (var x in source) await writer.WriteAsync(x);
// no Complete — readers wait forever
```

Always call `writer.Complete()` (or `TryComplete()` for safety) in a finally.

### Calling Complete twice

```csharp
writer.Complete();
writer.Complete();   // ⚠ throws InvalidOperationException
```

Use `TryComplete()` if uncertain.

### Reading after Complete without checking

```csharp
var item = await reader.ReadAsync();   // throws ChannelClosedException if drained
```

Use `ReadAllAsync` which exits naturally, or wrap in try/catch.

### Misusing SingleReader / SingleWriter

```csharp
var ch = Channel.CreateBounded<int>(new BoundedChannelOptions(100) {
    SingleReader = true   // ← I said only one reader
});

// But then:
_ = Task.Run(async () => { await foreach (var x in ch.Reader.ReadAllAsync()) ... });
_ = Task.Run(async () => { await foreach (var x in ch.Reader.ReadAllAsync()) ... });   // ⚠
```

The runtime may behave incorrectly. Only set SingleReader if you literally have one reader.

### Deep nesting losing track of completion

```csharp
var ch = Channel.CreateUnbounded<int>();
_ = Task.Run(async () => { /* produce, complete */ });
_ = Task.Run(async () => { /* produce, complete */ });   // ⚠ — calling Complete twice
```

For fan-in, coordinate completion separately (see fan-in pattern earlier).

---

## Performance

- WriteAsync / ReadAsync on non-empty / non-full channels: ~50-100 ns, no allocation (uses ValueTask).
- Contended ops: faster than locks (per-bucket synchronization).
- Bounded channels with backpressure: depend on producer/consumer speed — no inherent overhead.
- For millions of items/second pipelines, Channel is competitive with hand-rolled lock-free queues.

---

## When to use Channels

✓ Async producer-consumer.
✓ Pipeline stages with backpressure.
✓ Background work queues.
✓ Decoupling fast producers from slow consumers.

✗ Single-threaded code — overkill.
✗ Simple shared collection — ConcurrentDictionary / Queue suffices.
✗ Highly complex dataflow graphs (forks, joins, transforms) — consider TPL Dataflow.

---

## Summary

- `Channel<T>` is the modern async producer-consumer primitive.
- Bounded channels give backpressure for free.
- `ReadAllAsync` returns an `IAsyncEnumerable<T>` — clean async foreach.
- Always Complete the writer when done.
- Single/Multi reader / writer options give optimization hints.

→ Next: [14-TPL.md](14-TPL.md)
