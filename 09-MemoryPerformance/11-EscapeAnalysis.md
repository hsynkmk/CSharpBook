# Escape Analysis (.NET 10)

## What it is

**Escape analysis** is a JIT optimization that determines whether an allocated object's references "escape" the method. If they don't (the object is purely local), the JIT can **promote the allocation from the heap to the stack** — eliminating the heap object entirely.

```csharp
public int Compute(int x, int y) {
    var p = new Point(x, y);   // would normally allocate on the heap
    return p.X + p.Y;           // but p doesn't escape → JIT may stack-allocate it
}
```

.NET 10 dramatically improved escape analysis. The JIT now eliminates many small heap allocations that previously triggered GC pressure. Free performance win — your code without changes.

This optimization isn't visible in C# source. Profile to see the difference (allocations in `dotMemory` / `dotnet-counters`).

---

## Why it matters

A `new` of a small reference-type object: ~10-50 ns + GC pressure. Multiplied across millions of operations, this dominates many hot paths.

If the JIT can prove the object **never escapes** (no return value, no field store, no captured by lambda, etc.), allocating on the stack is essentially free. The object lives in the stack frame; freed when the method returns.

This is what languages like Go and Java's HotSpot have done for years. .NET added it gradually; .NET 10 made it actually useful for most code.

---

## What "escapes" means

An object escapes if its reference can outlive the current method:

- **Returned** from the method.
- **Stored in a field** of `this` or a static field.
- **Passed to a method that might store it**.
- **Captured by a lambda or local function**.
- **Used as part of a state machine** (async, iterator).
- **Boxed** to object / interface.
- **Pinned** via fixed or stackalloc-like.

If NONE of these happen, the object is "non-escaping" — the JIT may eliminate the heap allocation.

```csharp
// Non-escaping — JIT can stack-allocate
public int M() {
    var list = new List<int>();   // doesn't escape; never returned, stored, or captured
    for (int i = 0; i < 10; i++) list.Add(i);
    return list.Sum();
}

// Escaping — must allocate on heap
public List<int> M2() {
    var list = new List<int>();   // escapes (returned)
    return list;
}
```

In M, the JIT might eliminate the allocation. In M2, it must allocate.

---

## What escape analysis does

When the JIT compiles a method:

1. **Analyze every allocation** in the method (`newobj`).
2. For each, **track usage**: does the reference escape via the criteria above?
3. If non-escaping AND the object size is reasonable AND fields don't include exotic stuff (finalizers, etc.):
4. **Allocate on the stack** instead. The object's fields become stack-local data.
5. Optionally, **scalarize**: break the object into individual stack locals (e.g., a Point becomes two ints on the stack).

The optimization is silent. You don't see it in source. The behavior is identical (object semantics preserved). The runtime difference: no heap allocation, no GC tracking.

---

## What .NET 10 added

.NET 10's JIT improvements:
- **More aggressive non-escape detection** — sees through method calls if the callee doesn't escape (inlining + analysis combined).
- **Scalarization** — replaces small structs/classes with individual register values.
- **Async state machine box elision** — common async patterns no longer box the state machine if it doesn't escape.
- **`stackalloc` automatic upgrade** — some `new T[]` patterns are converted to `stackalloc T[]`.

Net result: significant allocation reductions in microbenchmarks. Real-world apps see 5-15% fewer allocations on hot paths.

---

## What the JIT can elide

### Local object that's never returned

```csharp
public int Sum(int[] arr) {
    var enumerator = arr.GetEnumerator();   // returns an iterator object
    int total = 0;
    while (enumerator.MoveNext()) total += (int)enumerator.Current;
    return total;
}
```

If the JIT can inline `GetEnumerator()` and see the iterator never escapes → stack allocation.

### `IEnumerable.GetEnumerator` boxing

```csharp
foreach (var x in list) { ... }
```

Pre-.NET 10, sometimes boxed the enumerator. Now usually elided.

### Async state machine

```csharp
public async Task<int> SimpleAsync() {
    await Task.Yield();
    return 42;
}
```

The state machine struct (containing local state) traditionally boxed on suspension. With box elision, simple cases avoid the box entirely.

### LINQ chains with non-escaping intermediates

```csharp
int sum = items.Where(x => x > 0).Sum();
```

The `Where` returns an iterator; `Sum` consumes it. If the iterator doesn't escape `Sum`'s frame, JIT may eliminate it.

The wins compound: LINQ chains that previously allocated 3-4 iterator objects now allocate 0 or 1.

---

## What still escapes (and allocates)

Even with .NET 10's improvements, some patterns still allocate:

```csharp
// Returned — must escape
return new Item();

// Stored in field
_field = new Item();

// Passed to opaque method
SomeFunc(new Item());   // can JIT see that SomeFunc doesn't store it? Maybe not.

// Captured by closure
list.ForEach(x => list2.Add(item));   // closure captures the lambda's state

// Used across await
async Task M() {
    var x = new Item();
    await Task.Yield();
    Use(x);   // x's state must survive the await — heap-needed
}

// Implements an interface and called via the interface
ISomething s = new Implementation();   // requires heap (boxed-ish for interfaces)
```

The JIT is conservative — when in doubt, it allocates. Profile to see.

---

## How to encourage escape elision

- **Keep objects local** — don't return them, don't store them, don't pass to opaque APIs.
- **Inline hot helpers** — use `[MethodImpl(MethodImplOptions.AggressiveInlining)]` so the JIT can analyze across the call boundary.
- **Prefer structs for small data** — struct allocations are already stack-based, no escape analysis needed.
- **Use `Span<T>` / `stackalloc`** — explicit stack usage, no JIT magic needed.
- **Use static methods where possible** — fewer hidden captures.

Most of these are already "good code" advice — encouraging escape elision is a side benefit, not the primary driver.

---

## Profile to verify

Use `dotnet-counters` or BenchmarkDotNet to compare:

```csharp
[MemoryDiagnoser]
public class Bench {
    [Benchmark]
    public int SimpleSum() {
        var list = new List<int> { 1, 2, 3, 4, 5 };
        return list.Sum();
    }
}
```

In .NET 8 vs .NET 10, this might show different allocations. .NET 10's escape analysis may stack-allocate the List (size-bounded), bringing allocs to 0.

For your real hot paths: run the benchmark, look at `Allocated` column. If it dropped or zeroed after upgrading to .NET 10, escape analysis kicked in.

---

## Common patterns

### Local builder pattern

```csharp
public string BuildName(string first, string last) {
    var sb = new StringBuilder();
    sb.Append(first);
    sb.Append(' ');
    sb.Append(last);
    return sb.ToString();
}
```

`sb` doesn't escape (we only call `ToString` which returns a different string). With escape analysis, the StringBuilder allocation might be elided.

For modern code, interpolated strings (`$"{first} {last}"`) often skip StringBuilder entirely.

### Iterator chain

```csharp
int total = arr.Where(x => x > 0).Select(x => x * 2).Sum();
```

In .NET 10, each of those LINQ operators creates a small struct enumerator that the JIT often stack-allocates. The chain executes with near-zero allocation for primitives.

### Async fast path

```csharp
public async Task<int> Get(int key) {
    if (_cache.TryGetValue(key, out var v)) return v;
    return await _slow.GetAsync(key);
}
```

If `TryGetValue` returns true, the async method completes synchronously. Pre-.NET 10, the state machine box still allocated. .NET 10 often elides it when the sync path is taken.

For best results, use `ValueTask` — same goals, explicit.

---

## Limitations

Escape analysis isn't magic:
- The JIT must INLINE the relevant methods to see escape. Inlining decisions can vary.
- Reference-type fields of escape-eligible objects might force heap allocation anyway.
- Generic / virtual dispatch can defeat analysis (the JIT can't see the actual callee).
- Allocations inside loops are harder to analyze.

Don't rely on escape elision for correctness or for fundamental optimizations. Use it as a "bonus" when profiling shows it helped.

---

## Internals — how the JIT analyzes

The JIT walks each allocation:

1. **Reaching definition** — track where the reference flows.
2. **Sink analysis** — find all uses; check if any are "escape" sites.
3. **Inline-aware** — if the reference is passed to a callee, inline if possible and continue tracking.
4. **Stackalloc decision** — if non-escaping and small enough, replace the heap allocation with stack memory.
5. **Scalarization** — sometimes replace the object's fields with separate stack locals.

This is similar to what HotSpot (Java) and modern C++ compilers do. .NET joined the club gradually; .NET 10 is the most capable yet.

### Why it took so long

C# / .NET has more dynamic features than C++:
- Reflection can find any object.
- Virtual dispatch makes inlining harder.
- Try/catch + exception unwinding constrains escape analysis.
- GC + finalizers add complexity.

Each version of .NET chips away at the conservative assumptions. .NET 10's analysis is what other languages had earlier; the .NET team caught up.

---

## What's coming

.NET 11+ will likely:
- More aggressive scalarization.
- Whole-method allocation elimination via dataflow.
- AOT-aware analysis (Native AOT can do more aggressive optimization because there's no reflection at runtime).

For .NET 10: enjoy what's here. Upgrade your runtime; benchmark to see the wins.

---

## Common bugs / non-issues

There aren't really "bugs" with escape analysis — it's an optimization, not a feature. If it's missing, the worst case is "no improvement" (still allocates as before).

Things to keep in mind:
- Don't rely on identity for stack-allocated objects (`ReferenceEquals` may behave oddly if internal scalarization happened — but the runtime hides this from your code).
- For debugging, the optimization isn't visible — the IL still has `newobj`. The JIT replaces it during compilation.

---

## When to think about escape analysis

For most code: don't. Write clean code; trust the JIT.

For hot paths in libraries:
- Profile with BenchmarkDotNet's `[MemoryDiagnoser]`.
- If allocations are high and seemingly transient, consider patterns that help escape analysis (small structs, local-only usage).
- Compare .NET 8 vs .NET 10 numbers — the latter often wins.

For library authors: design APIs to encourage non-escape (e.g., callbacks / Span params instead of returning small objects).

---

## Summary

- Escape analysis is a JIT optimization that promotes non-escaping heap allocations to the stack.
- .NET 10 made it significantly more capable.
- "Escape" means: return, store in field, capture, pass to opaque method, cross async, etc.
- Free performance — no source changes needed.
- Profile to confirm — `dotnet-counters` or BenchmarkDotNet `[MemoryDiagnoser]`.
- For specific allocation control: use Span, stackalloc, structs, ArrayPool — those are explicit and predictable.

→ Next: [12-StringInterning.md](12-StringInterning.md)
