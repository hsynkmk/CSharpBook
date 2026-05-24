# Concurrent Collections

> `ConcurrentDictionary<K, V>`, `ConcurrentQueue<T>`, `ConcurrentStack<T>`, `ConcurrentBag<T>`, `BlockingCollection<T>`. Thread-safe collections from `System.Collections.Concurrent`.

These let multiple threads add, remove, and read **without external locking** — the collections handle synchronization internally, usually with fine-grained per-bucket locks or lock-free algorithms.

```csharp
var dict = new ConcurrentDictionary<string, int>();

// Many threads can do this safely:
dict["a"] = 1;
dict.TryAdd("b", 2);
dict.TryGetValue("a", out var v);
dict.AddOrUpdate("c", 1, (k, old) => old + 1);
```

For the read-mostly, write-mostly, and mixed scenarios of high-concurrency code, these collections are the right tool.

---

## ConcurrentDictionary&lt;K, V&gt;

The most-used concurrent collection. Thread-safe drop-in for `Dictionary<K, V>` (mostly).

### API

```csharp
var d = new ConcurrentDictionary<string, int>();

// Single ops
d.TryAdd(key, value);                                // false if key exists
d.TryGetValue(key, out var value);
d.TryRemove(key, out var value);
d.TryUpdate(key, newValue, comparisonValue);          // CAS-style conditional update
d[key] = value;                                       // unconditional set
int v = d[key];                                       // throws if missing

// Atomic compound ops
int v1 = d.GetOrAdd(key, k => Expensive(k));
int v2 = d.AddOrUpdate(key, addValue, (k, old) => old + 1);

// Inspection
int count = d.Count;
bool has = d.ContainsKey(key);
foreach (var (k, v) in d) { ... }  // snapshot-style iteration
```

### Atomic compound operations

The big wins over `Dictionary` + `lock`:

```csharp
// Atomic "increment by key" (count words)
counts.AddOrUpdate("hello", 1, (k, old) => old + 1);
```

Equivalent locking would be:
```csharp
lock (gate) {
    if (counts.TryGetValue("hello", out var v)) counts["hello"] = v + 1;
    else counts["hello"] = 1;
}
```

The concurrent version uses fine-grained locks (one per bucket) — far more parallel.

### `GetOrAdd` — lazy factory

```csharp
var conn = pool.GetOrAdd(host, h => OpenConnection(h));
```

If `host` is in the dict, return its value. If not, call the factory, add the result, return it.

**Gotcha**: the factory **may run multiple times** under contention. ConcurrentDictionary only guarantees one **stored** value, not one **factory invocation**. If your factory has side effects, this matters:

```csharp
var ud = data.GetOrAdd(key, k => {
    Connection.Open();   // ⚠ might run multiple times
    return result;
});
```

For factory-must-run-once, use `Lazy<T>`:

```csharp
private readonly ConcurrentDictionary<string, Lazy<Connection>> _pool = new();
var lazy = _pool.GetOrAdd(host, h => new Lazy<Connection>(() => OpenConnection(h)));
var conn = lazy.Value;   // factory runs exactly once, thread-safe via Lazy
```

`Lazy<T>` ensures the factory runs only once even if multiple GetOrAdd calls win the race.

### `AddOrUpdate`

```csharp
counts.AddOrUpdate(
    key,
    addValueFactory: k => 1,                        // if absent, what to add
    updateValueFactory: (k, old) => old + 1          // if present, what to set
);
```

Same race caveat as GetOrAdd — the update factory may run multiple times under contention. For counters, that's usually fine because the update is idempotent on retry.

### Enumeration

```csharp
foreach (var (k, v) in dict) { ... }
```

Enumeration is **snapshot-style** — not guaranteed to see updates that happen during iteration. Doesn't throw "collection modified" — items may appear or disappear, but iteration won't fail.

For a **consistent snapshot**, call `dict.ToArray()` (returns array of KeyValuePair, captured at a moment in time).

### Performance

ConcurrentDictionary uses **bucket-level locking** (default 1 lock per ~4 buckets, scaled to processor count). Multiple threads working on different buckets don't contend.

For reads, lock-free fast path — atomic checks of the bucket state. Writes take a per-bucket lock.

Vastly more scalable than `Dictionary` + global lock, especially with many writers.

---

## ConcurrentQueue&lt;T&gt;

Thread-safe FIFO.

```csharp
var q = new ConcurrentQueue<int>();
q.Enqueue(1);
q.Enqueue(2);
if (q.TryDequeue(out var v)) { /* ... */ }
if (q.TryPeek(out var v2)) { /* ... */ }
int count = q.Count;
```

No `Dequeue` (only `TryDequeue`) — race-friendly API.

**Lock-free** implementation. Both Enqueue and Dequeue are O(1) amortized. Scales well under contention.

Used for producer-consumer where readers and writers run concurrently.

Modern alternative: `Channel<T>` for async producer-consumer with backpressure. See [§13](13-Channels.md).

---

## ConcurrentStack&lt;T&gt;

Thread-safe LIFO.

```csharp
var s = new ConcurrentStack<int>();
s.Push(1);
s.Push(2);
if (s.TryPop(out var v)) { /* ... */ }
if (s.TryPeek(out var v2)) { /* ... */ }
```

Lock-free Treiber stack. Same shape as ConcurrentQueue but LIFO.

Less commonly used than ConcurrentQueue — most concurrent producer-consumer wants FIFO.

---

## ConcurrentBag&lt;T&gt;

A thread-safe **unordered** bag. Optimized for **same-thread add/remove**.

```csharp
var bag = new ConcurrentBag<int>();
bag.Add(1);
if (bag.TryTake(out var v)) { /* ... */ }
```

Each thread has its own local list (work-stealing). When a thread Adds, it's free — no contention. When a thread Takes, it tries its own list first; if empty, steals from others.

**Use case**: parallel work where each thread accumulates results, then drains at the end (e.g., parallel aggregation).

For the typical "producer adds, consumer takes," `ConcurrentQueue` is faster.

---

## BlockingCollection&lt;T&gt;

A wrapper around any concurrent collection (default: ConcurrentQueue) that adds **blocking** semantics — Take blocks when empty, Add blocks when bounded and full.

```csharp
var bc = new BlockingCollection<int>(boundedCapacity: 100);

// Producer
_ = Task.Run(() => {
    foreach (var item in source) {
        bc.Add(item);   // blocks if full
    }
    bc.CompleteAdding();
});

// Consumer
foreach (var item in bc.GetConsumingEnumerable()) {
    Process(item);
}
```

Older pattern (pre-Channel). Modern code uses `Channel<T>` instead — it's async-friendly, doesn't block threads.

Use BlockingCollection if you have sync code that needs blocking semantics. For async, `Channel<T>` wins.

---

## When to use which

| Need | Use |
|---|---|
| Thread-safe key-value lookup | `ConcurrentDictionary<K, V>` |
| FIFO producer-consumer | `ConcurrentQueue<T>` (sync) or `Channel<T>` (async) |
| LIFO concurrent stack | `ConcurrentStack<T>` |
| Per-thread accumulation | `ConcurrentBag<T>` |
| Blocking producer-consumer | `BlockingCollection<T>` (sync) or `Channel<T>` (async) |
| Read-only after build | `FrozenDictionary<K, V>` + atomic swap |
| Snapshot-style immutability | `ImmutableDictionary<K, V>` + atomic swap |

---

## Common patterns

### Word counter

```csharp
var counts = new ConcurrentDictionary<string, int>();
Parallel.ForEach(words, w => {
    counts.AddOrUpdate(w, 1, (_, old) => old + 1);
});
```

Atomic per-word increment without locks.

For even faster (no closure allocation per increment):
```csharp
counts.AddOrUpdate(w, k => 1, (k, old) => old + 1);
```

Or with `CollectionsMarshal.GetValueRefOrAddDefault` (.NET 6+) for a regular Dictionary + lock-per-shard if you're optimizing.

### Connection pool

```csharp
private readonly ConcurrentDictionary<string, Lazy<Connection>> _pool = new();

public Connection Get(string host) {
    var lazy = _pool.GetOrAdd(host, h => new Lazy<Connection>(() => OpenConnection(h)));
    return lazy.Value;
}
```

Per-host connection, opened exactly once via Lazy.

### Fan-out collect

```csharp
var results = new ConcurrentBag<Result>();
await Parallel.ForEachAsync(items, async (item, ct) => {
    var r = await ProcessAsync(item, ct);
    results.Add(r);
});
return results.ToArray();
```

Each task adds to its local bag slot. ToArray drains all slots into an array. Lock-free fan-out collection.

For ordered results, use `ConcurrentDictionary<int, Result>` with index as key.

### Producer/consumer (sync)

```csharp
var queue = new ConcurrentQueue<Work>();

// Multiple producers
Parallel.ForEach(items, item => queue.Enqueue(Build(item)));

// Multiple consumers
Parallel.For(0, 4, _ => {
    while (queue.TryDequeue(out var work)) Process(work);
});
```

For one-shot producer-consumer, this works. For continuous, use `BlockingCollection<T>` or `Channel<T>`.

---

## Common bugs

### Treating ConcurrentDictionary like Dictionary

```csharp
if (!dict.ContainsKey(key)) {
    dict[key] = value;   // ⚠ — TOCTOU race
}
```

Two threads can both pass the ContainsKey check, both add. Use atomic operations:

```csharp
dict.TryAdd(key, value);   // atomic
// OR
dict.AddOrUpdate(key, value, (_, old) => old);
```

### Side effects in GetOrAdd / AddOrUpdate factories

```csharp
dict.GetOrAdd(key, k => {
    LogToFile($"Loading {k}");   // ⚠ might log multiple times under contention
    return Load(k);
});
```

The factory can run multiple times. Don't put non-idempotent side effects in it.

### Enumerating without expecting changes

```csharp
foreach (var (k, v) in dict) {
    // While iterating, other threads add/remove
    // The iteration won't throw, but you may see partial state
}
```

If you need a consistent view, `dict.ToArray()` first.

### `Count` is a snapshot

```csharp
if (dict.Count == 0) { /* dict might already have entries */ }
```

`Count` is a moment-in-time value. Don't make decisions based on it that race-related operations depend on.

### Adding to ConcurrentBag and expecting order

```csharp
bag.Add(1); bag.Add(2); bag.Add(3);
// Order undefined; ConcurrentBag is unordered
```

If order matters, use ConcurrentQueue or a list with a lock.

---

## Internals — ConcurrentDictionary's design

```csharp
private class Tables {
    public Node[] buckets;
    public object[] locks;
    public int[] countPerLock;
    // ...
}

private volatile Tables _tables;
```

Multiple buckets (like a Dictionary). Multiple **locks** (default: ~ProcessorCount or 4 per bucket, scaled).

Lookup:
1. Hash → bucket index.
2. Lock-free read of the bucket's chain (linked list of nodes).
3. Walk and find.

Mutation:
1. Hash → bucket index → lock index.
2. Take lock for that bucket group.
3. Mutate the chain.

Multiple threads writing to different buckets don't contend. Reads never lock (except during resize).

On resize: the entire structure is rebuilt in a new `Tables`. The `volatile` field publishes the new one atomically. In-flight readers see either old or new — never a partial state.

### Performance scaling

For uncontested access: ~50-100 ns per op (slightly slower than Dictionary due to atomic checks).
For contended access: scales linearly with cores up to the number of locks. With 4 locks and 16 threads, you'll see some contention but nothing catastrophic.

For ultra-high contention (one hot key), even ConcurrentDictionary slows down — all writers to that key serialize on the bucket's lock. Solutions:
- Shard keys further (per-key sub-dictionaries).
- Use Interlocked.Increment on a value-type counter field directly.

---

## ConcurrentDictionary vs alternatives

| | ConcurrentDict | Dict + lock | ImmutableDict + Interlocked |
|---|---|---|---|
| Reads | mostly lock-free | always lock | always lock-free |
| Writes | per-bucket lock | global lock | full copy + CAS |
| Memory | ~bucket count overhead | low | high (per-version trees) |
| Best for | mixed read/write | low contention | read-heavy, occasional write |
| Snapshot reads | weak | yes (under lock) | yes (just read the ref) |

For most concurrent maps, ConcurrentDictionary is the right answer.

---

## Performance summary

- ConcurrentDictionary: ~50-100 ns/op uncontested; scales with bucket-level locks.
- ConcurrentQueue: ~50-100 ns/op; lock-free.
- ConcurrentStack: ~50-100 ns/op; lock-free Treiber.
- ConcurrentBag: ~30-50 ns/op same-thread; slower cross-thread.
- BlockingCollection: thread blocking + condition variables; ~microsecond per op under contention.
- All vastly better than `Dictionary<K, V> + lock` under contention.

---

## When NOT to use concurrent collections

✗ Single-threaded code — overhead for no benefit.
✗ Read-only after build — `FrozenDictionary` is faster.
✗ Async producer/consumer with backpressure — `Channel<T>` is async-friendly.
✗ "Just one writer" scenarios — `ImmutableDictionary` + Interlocked is elegant.

---

## Summary

- `ConcurrentDictionary<K, V>` — thread-safe map. Use for mixed read/write under contention.
- `ConcurrentQueue<T>` — thread-safe FIFO. For sync producer-consumer.
- `ConcurrentStack<T>` — thread-safe LIFO. Rare.
- `ConcurrentBag<T>` — thread-safe unordered. For per-thread accumulation.
- `BlockingCollection<T>` — sync blocking wrapper. Modern code prefers `Channel<T>`.

These collections handle synchronization for you. They don't replace the need for understanding concurrency — race conditions in your USE of them are still possible (e.g., "check then act" anti-pattern).

→ Next: [13-Channels.md](13-Channels.md)
