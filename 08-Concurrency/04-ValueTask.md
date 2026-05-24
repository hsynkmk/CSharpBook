# ValueTask and ValueTask&lt;T&gt;

## What it is

`ValueTask<T>` is a lightweight `Task<T>` alternative for hot paths where most calls complete **synchronously**. It's a `struct` that holds either:
- A `T` (when the operation completed synchronously), OR
- A `Task<T>` (when it had to suspend), OR
- An `IValueTaskSource<T>` (when using pooled state machines for max performance).

```csharp
public ValueTask<int> GetAsync(int id) {
    if (_cache.TryGetValue(id, out var value)) {
        return new ValueTask<int>(value);   // ← no allocation
    }
    return new ValueTask<int>(LoadFromDbAsync(id));   // ← wraps Task<int>
}

int n = await GetAsync(1);
```

For the cache-hit path: zero allocation. For the cache-miss path: same as Task. Best of both worlds for cache-like APIs.

ValueTask was added in .NET Core 2.0 (2017) for this exact scenario.

---

## Why it exists

Returning a `Task<T>` from an async method allocates the Task (~100-200 bytes). Usually fine. But for **very hot paths** that almost always complete synchronously (cache hits, polling that finds the value immediately, await on already-completed work), that allocation per call adds up:

- Method called 10 million times → ~1-2 GB of garbage.
- Pressure on Gen0 GC → throughput collapse.

`ValueTask` is the answer: a struct that holds the result inline if available, only allocating a Task if it actually has to suspend.

The trade-off: stricter usage rules. You **can't** await a `ValueTask` more than once, store it, or pass it around freely. It's optimized for "create, await, done."

---

## When to use

✅ **Use `ValueTask` / `ValueTask<T>` when:**
- The method **often** completes synchronously (cache hits, fast-path returns).
- It's called millions of times — saving allocations matters.
- You're writing low-level library code optimized for performance.

✅ **Use `Task` / `Task<T>` when:**
- The default for everything else.
- You'll await multiple times, store, or share.
- The method is usually genuinely async.
- You're writing application code (the allocation savings rarely matter).

**Rule of thumb**: prefer `Task` until profiling shows allocation pressure from a specific hot method. Then upgrade that method to `ValueTask`.

---

## Usage

### Producing

```csharp
// From a synchronous result — no allocation
return new ValueTask<int>(42);
return new ValueTask<int>();           // default for value types (0)

// From a Task<T> — wraps it
return new ValueTask<int>(LoadFromDbAsync(id));

// From an async method
public async ValueTask<int> LookupAsync(int id) {
    if (_cache.TryGetValue(id, out var v)) return v;   // sync — no Task
    return await LoadFromDbAsync(id);
}
```

When the async method returns synchronously (taking the cache hit path), no Task is allocated. The state machine produces a `ValueTask<T>` wrapping the immediate value.

### Consuming

```csharp
int v = await GetAsync(1);   // works just like Task<int>
```

Looks identical to consuming a Task<T>. Difference is in what's allocated.

---

## The strict rules

**ValueTask has restrictions that Task doesn't:**

### 1. Await at most ONCE

```csharp
var vt = GetAsync(1);
int a = await vt;
int b = await vt;   // ⚠ UNDEFINED behavior
```

ValueTask is one-shot. Awaiting a second time might return a different value, throw, or corrupt state. If you need the result multiple times, **assign it to a variable**:

```csharp
int v = await GetAsync(1);
int a = v; int b = v;
```

Or, if you really need the Task, convert:

```csharp
Task<int> t = vt.AsTask();   // promotes to Task, then can await multiple times
```

### 2. Don't access `.Result` unless completed

```csharp
var vt = GetAsync(1);
int v = vt.Result;   // ⚠ may throw or return wrong value if not complete
```

Use `if (vt.IsCompleted) ... vt.Result ...` carefully, or convert to Task. In general, just await.

### 3. Don't store ValueTask in a field, collection, or pass around

```csharp
List<ValueTask<int>> tasks = new();   // ⚠ — defeats the purpose
```

ValueTask is meant to be created, awaited, and dropped. If you need a collection of tasks, use `Task<T>`:

```csharp
List<Task<int>> tasks = new();
tasks.Add(vt.AsTask());   // promote to Task to store
```

### 4. Don't concurrently consume

Don't pass a ValueTask to two consumers. Pick one or convert to Task.

These rules let the runtime safely reuse internal state. Violating them = undefined behavior.

---

## ValueTask (non-generic) and the value-task version of Task

`ValueTask` (no generic) is the void-equivalent — for methods that return no value but might complete synchronously:

```csharp
public async ValueTask FlushAsync() {
    if (_bufferedSize == 0) return;       // synchronous fast path
    await _stream.WriteAsync(_buffer);
}
```

Same rules.

---

## Performance comparison

For a method called 10M times with a 90% cache hit rate:

| Return type | Allocations (10M calls) | Time |
|---|---|---|
| `Task<int>` | ~10M (one per call) | baseline |
| `Task<int>` with `Task.FromResult` caching | ~10M (FromResult allocates) | similar |
| `ValueTask<int>` | ~1M (one per cache miss) | ~2× faster |
| Just returning T directly (not async) | 0 | ~5× faster |

ValueTask saves the 9M allocations from cache hits. For methods called billions of times in libraries (logging, parsing, etc.), this matters.

For typical application code called thousands of times per request, the difference is in microseconds and not worth the complexity.

---

## Where the BCL uses ValueTask

The BCL adopted ValueTask in many places:

- `Stream.ReadAsync(Memory<byte>, CancellationToken)` → `ValueTask<int>` (.NET Core 2.1+).
- `ChannelReader<T>.ReadAsync` → `ValueTask<T>`.
- `MemoryStream` operations that complete synchronously.
- `System.IO.Pipelines`.

When streaming, **most reads complete synchronously** (data already buffered) → ValueTask saves a Task allocation per read in tight loops.

---

## IValueTaskSource — the pool

For ultimate performance, `ValueTask<T>` can be backed by an `IValueTaskSource<T>` — a reusable, pooled state machine.

```csharp
public class MyPooled : IValueTaskSource<int> {
    // ... state ...
    public int GetResult(short token) { ... }
    public ValueTaskSourceStatus GetStatus(short token) { ... }
    public void OnCompleted(Action<object?> continuation, object? state, short token, ValueTaskSourceContinuationFlags flags) { ... }
}

public ValueTask<int> GetAsync() {
    var source = _pool.Rent();
    return new ValueTask<int>(source, source.Version);
}
```

This is **very advanced** — you'd write this for a library serving millions of QPS. Most code uses ValueTask via `async ValueTask<T>` methods, never touching IValueTaskSource directly.

`System.IO.Pipelines`, `Channel<T>`, and `Socket` use these internally to avoid per-operation allocations entirely.

---

## Common patterns

### Cache-first lookup

```csharp
public async ValueTask<User?> GetUserAsync(int id, CancellationToken ct = default) {
    if (_cache.TryGetValue(id, out User? cached)) return cached;
    var user = await _db.Users.FindAsync(new object[] { id }, ct);
    if (user is not null) _cache[id] = user;
    return user;
}
```

Most calls hit the cache — synchronous, no allocation. Cache misses fall through to async DB call.

### Streaming reader

```csharp
public async ValueTask<int> ReadAsync(Memory<byte> buffer) {
    if (_buffered > 0) {
        int copyLen = Math.Min(buffer.Length, _buffered);
        _internalBuffer.AsSpan(0, copyLen).CopyTo(buffer.Span);
        _buffered -= copyLen;
        return copyLen;   // synchronous — no Task allocated
    }
    return await _innerStream.ReadAsync(buffer);
}
```

Reads from an internal buffer when available; falls back to async only when empty.

### Conditional async

```csharp
public ValueTask SaveAsync(Item item) {
    if (item.IsDirty) return SaveImpl(item);   // returns ValueTask
    return default;   // synchronous no-op — zero allocation
}
```

For "maybe nothing to do" semantics, ValueTask + default is idiomatic.

---

## Converting to/from Task

```csharp
// ValueTask -> Task
Task t = vt.AsTask();
Task<int> tv = vtg.AsTask();

// Task -> ValueTask
ValueTask vt = new ValueTask(task);
ValueTask<int> vtv = new ValueTask<int>(task);
```

`AsTask()` materializes the underlying Task. If the ValueTask was synchronous, it allocates a new Task wrapping the value. If it was already a Task, returns the same one.

Convert when:
- You need to store/share the result.
- You need to await multiple times.
- The API requires Task.

---

## Internals — the struct layout

```csharp
public readonly struct ValueTask<TResult> {
    private readonly object? _obj;       // null, Task<TResult>, or IValueTaskSource<TResult>
    private readonly TResult? _result;
    private readonly short _token;        // for IValueTaskSource versioning
    private readonly bool _continueOnCapturedContext;
}
```

Three "modes":
1. `_obj == null`: synchronously completed. `_result` holds the value.
2. `_obj is Task<T>`: wraps a Task — defer to that.
3. `_obj is IValueTaskSource<T>`: uses pooled state machine, `_token` identifies the current operation.

On `await`, the runtime checks the mode and behaves accordingly. The struct itself is ~24 bytes (on 64-bit) — passed by value, no heap allocation.

### Async ValueTask state machine

For `async ValueTask<T>` methods, the compiler generates a state machine that uses `AsyncValueTaskMethodBuilder<T>`. If the method completes synchronously (no await actually suspends), the builder produces a `ValueTask<T>` wrapping the result — no Task object allocated.

If it does suspend, the builder allocates a Task<T> internally and wraps it. Same eventual cost as Task in that case.

---

## Common bugs

### Awaiting twice

```csharp
var vt = GetAsync();
int a = await vt;
int b = await vt;   // ⚠ undefined behavior
```

Compiler can sometimes catch this, but not always. Be disciplined.

### Returning ValueTask from a public API without thinking

If your API returns `ValueTask`, callers MUST follow the rules. Document the constraint or use Task — Task is forgiving.

### Storing in collections / fields

```csharp
private ValueTask<int> _pending;   // ⚠ violates "don't store"
```

Convert to Task if storing is unavoidable.

### Mixing await with synchronous .Result

```csharp
var vt = GetAsync();
if (vt.IsCompleted) {
    int v = vt.Result;   // OK — but you've consumed it
} else {
    int v = await vt;     // OK
}
```

Either branch is fine; both consume the ValueTask. The pattern is sometimes useful for "fast path: return immediately; slow path: await."

---

## When NOT to use ValueTask

✗ Methods that are almost always async — Task is simpler.
✗ Public APIs where callers might violate the rules.
✗ Application code where allocation isn't the bottleneck.
✗ Code you'll await multiple times or pass around — use Task.

---

## Performance summary

- Sync-completing path: zero heap allocations.
- Suspending path: same allocations as Task (Task instance + state machine box).
- await on a ValueTask: same cost as Task at runtime.
- Conversion to Task (`.AsTask()`): one allocation if ValueTask was synchronous.

For library code optimizing 1M+ QPS, ValueTask saves serious memory pressure. For everyday code, prefer Task.

→ Next: [05-Cancellation.md](05-Cancellation.md)
