# Chapter 09 — Questions

> 35+ drilling questions on memory layout, GC, IDisposable, Span/Memory, and modern performance APIs.

---

## Stack vs Heap

**Q1.** Where does a `List<int>` instance live?
<details><summary>Answer</summary>The List object itself is on the heap (it's a reference type). The variable referring to it is on the stack (if local). The List's internal array is a separate heap object.</details>

**Q2.** Is "value types are on the stack" always true?
<details><summary>Answer</summary>No. Value types are on the stack as locals. As fields of a class, they're inline within the class's heap allocation. When boxed, on the heap. When captured by a closure or async state machine, hoisted to the heap. The simple model is a useful starting point, not a literal truth.</details>

**Q3.** What's `stackalloc`?
<details><summary>Answer</summary>Allocates memory on the current method's stack frame. Returns a Span. The buffer lives until the method returns. Fastest allocation possible. Limited to ≤~1 KB and unmanaged types.</details>

---

## Garbage Collection

**Q4.** What are the GC generations?
<details><summary>Answer</summary>Gen0 (newly allocated), Gen1 (survived 1 GC), Gen2 (long-lived), LOH (Large Object Heap, ≥85K bytes), POH (Pinned Object Heap, since .NET 5). Gen0 collections are frequent and cheap; Gen2 are rare and expensive.</details>

**Q5.** What's the generational hypothesis?
<details><summary>Answer</summary>Most objects die young. Collecting Gen0 frequently is cheap and reclaims most garbage. Gen2 collections are rare because long-lived objects don't churn. Justifies the cost/benefit of generational design.</details>

**Q6.** What's DATAS and why does .NET 10 make it default?
<details><summary>Answer</summary>DATAS = Dynamically Adapting To Application Sizes. The GC auto-tunes its heap targets based on the running app's allocation pattern. Pre-DATAS Server GC pre-allocated huge heaps based on CPU count — wasteful in containers. DATAS starts small, grows as needed. .NET 10 enables by default — better memory efficiency for cloud / container workloads.</details>

**Q7.** What's the LOH and why does it matter?
<details><summary>Answer</summary>Large Object Heap, for allocations ≥85,000 bytes. Not compacted by default. Only collected during Gen2 GC. Allocating/freeing large objects of varying sizes causes fragmentation. Use ArrayPool to avoid churning the LOH.</details>

**Q8.** What does `GC.SuppressFinalize(this)` do?
<details><summary>Answer</summary>Removes `this` from the finalization queue. Used in Dispose to skip the finalizer (since cleanup is done). Avoids the two-cycle overhead of finalization.</details>

---

## IDisposable

**Q9.** What's the difference between `using (var x = ...)` and `using var x = ...`?
<details><summary>Answer</summary>Block form (`using (...)`) scopes the disposal to the brace block. Declaration form (`using var x = ...`, C# 8+) scopes to the enclosing block (typically the method). Same semantics; declaration form is cleaner with less indentation.</details>

**Q10.** What's the full Dispose Pattern?
<details><summary>Answer</summary>Public `Dispose()` calls `Dispose(true)` + `GC.SuppressFinalize(this)`. Protected virtual `Dispose(bool disposing)` does the work: if `disposing` true, clean up managed resources; always clean up unmanaged. Finalizer `~MyClass()` calls `Dispose(false)`. The `_disposed` flag makes it idempotent.</details>

**Q11.** Why use SafeHandle?
<details><summary>Answer</summary>SafeHandle handles unmanaged resource cleanup with critical finalization. Survives even hard process tear-down. Has built-in ref-counting during P/Invoke. Lets you avoid writing your own finalizer + complex Dispose pattern. Always prefer SafeHandle for native handles over `IntPtr` + manual finalizer.</details>

---

## Finalizers

**Q12.** Why are finalizers expensive?
<details><summary>Answer</summary>Finalizable objects survive two GC cycles (queued for finalization, then collected). They typically promote to Gen2. The finalizer thread is single-threaded — slow finalizers block all others. Net: high allocation of finalizable objects pressures the GC badly.</details>

**Q13.** What should you NOT do in a finalizer?
<details><summary>Answer</summary>Don't touch other managed objects (they may already be finalized). Don't throw (process risk pre-.NET 5; swallowed in modern .NET). Don't take long (blocks the finalizer thread). Don't block on async / locks (deadlock risk). Don't assume it runs (process kill skips it).</details>

---

## Span / Memory

**Q14.** What's the difference between `Span<T>` and `Memory<T>`?
<details><summary>Answer</summary>Both are typed views over contiguous memory. `Span<T>` is a `ref struct` — stack-only, fastest access, can't cross await or be a field. `Memory<T>` is a regular struct — can be stored in fields, cross await; slightly slower access. Pattern: store as Memory, get .Span for fast work.</details>

**Q15.** Why can't `Span<T>` be a class field?
<details><summary>Answer</summary>Span often points to stack-allocated memory (via `stackalloc`). If a Span outlived its stack frame (by being on the heap as a class field), it would dangle. The `ref struct` restriction prevents this at compile time.</details>

**Q16.** Predict the allocations:
```csharp
ReadOnlySpan<char> s = "hello".AsSpan(0, 3);
```
<details><summary>Answer</summary>Zero. `"hello"` is a string literal (interned, already exists). `.AsSpan(0, 3)` creates a ReadOnlySpan struct on the stack pointing into the string. No copy, no heap allocation.</details>

---

## ArrayPool

**Q17.** What's `ArrayPool<T>.Shared.Rent(1000)` going to return?
<details><summary>Answer</summary>An array of length **at least** 1000 — possibly larger (e.g., 1024, the next bucket size). Always use the requested size (1000) for your logical work, not `buf.Length`.</details>

**Q18.** Why use ArrayPool over `new byte[size]` in a hot loop?
<details><summary>Answer</summary>Avoids per-call heap allocation. The pool reuses arrays. Especially valuable for large arrays (≥85K bytes) that would otherwise hit the LOH. For 1M iterations, ArrayPool turns GB of garbage into a handful of arrays reused.</details>

**Q19.** Why use try/finally with ArrayPool?
<details><summary>Answer</summary>Ensures `Return` is called even if the work throws. Without it, the array is "leaked" from the pool's perspective (the pool just allocates fresh ones). Always pair Rent with try/finally Return.</details>

---

## Stackalloc

**Q20.** What's the size limit on stackalloc?
<details><summary>Answer</summary>The stack is ~1 MB total per thread. Practical limit: a few KB per stackalloc. Larger risks StackOverflowException (uncatchable, kills the process). For large temporary buffers, use ArrayPool.Rent.</details>

**Q21.** What's `[SkipLocalsInit]`?
<details><summary>Answer</summary>Skips the zero-initialization of stackalloc'd memory. Saves a few ns per byte. Use only when the code is guaranteed to fill the buffer before reading. Application-wide via assembly-level attribute, or per-method.</details>

**Q22.** Can you stackalloc a `Span<Box>` where `Box` is a class?
<details><summary>Answer</summary>No. Stackalloc requires `unmanaged` types — value types with no reference-type fields anywhere. Can't stackalloc a Span of class references. For generic stackalloc: `where T : unmanaged`.</details>

---

## ref features

**Q23.** What does `ref readonly` mean on a parameter?
<details><summary>Answer</summary>Passes the parameter by reference (no copy), but the method can't write to it. Used for large structs to avoid copying without allowing mutation. Same effect as `in` (which is `ref readonly` implicit), but `ref readonly` at the call site is explicit (since C# 12).</details>

**Q24.** Why is `ref struct` restricted?
<details><summary>Answer</summary>ref structs (like Span) often hold references to stack memory or pinned data. Restricting them to stack-only prevents dangling references. Can't be class fields, can't cross await, can't be boxed, can't be captured by lambdas.</details>

**Q25.** What's `scoped` for?
<details><summary>Answer</summary>C# 11+ keyword saying a `ref` or `ref struct` parameter / local won't escape the method. Lets the compiler enforce a tighter lifetime than the default. Used by library authors to make lifetime contracts explicit.</details>

---

## Unsafe

**Q26.** When is `unsafe` justified in modern C#?
<details><summary>Answer</summary>Native interop with APIs requiring pointer parameters. SIMD intrinsics that need pointer arguments. Custom native heap allocation. Hot-path bit-packing where MemoryMarshal isn't enough. For most code, Span / MemoryMarshal / Unsafe.* cover the use cases safely.</details>

**Q27.** What does `fixed` do?
<details><summary>Answer</summary>Pins a heap object (array, string) so the GC won't move it during collection. Required when taking a pointer to a heap object that you'll use across a potential GC point (typically a native call). Excessive pinning fragments the heap; POH (.NET 5+) helps for long-lived pins.</details>

---

## Escape Analysis

**Q28.** What's escape analysis and what does .NET 10 add?
<details><summary>Answer</summary>JIT optimization that determines whether an allocated object's references "escape" the method. If not, the JIT can stack-allocate it instead of the heap. .NET 10 made it significantly more capable — many small allocations that previously hit the heap now stay on the stack. Free perf with no code changes.</details>

**Q29.** Does an object that's returned from a method escape?
<details><summary>Answer</summary>Yes. Return = caller can use it = it lives past the method. Must allocate on the heap. Same for: stored in a field, captured by lambda, used across await, boxed, passed to opaque methods.</details>

---

## String interning

**Q30.** Are string literals interned automatically?
<details><summary>Answer</summary>Yes. The compiler emits literals to assembly metadata; at load time, equal literals across all assemblies share the same heap object. Process-wide intern pool. `"hello" == "hello"` references the same instance.</details>

**Q31.** Why is `string.Intern(userInput)` a memory leak?
<details><summary>Answer</summary>The intern pool is a GC root. Interned strings live forever (until process exit). Interning unbounded user data grows the pool indefinitely. Prefer `StringPool` (bounded) or `FrozenSet` / `Dictionary` for dedup lookups.</details>

---

## Memory Leaks

**Q32.** Why does this leak?
```csharp
public class Subscriber {
    public Subscriber(Publisher pub) {
        pub.Event += OnEvent;
    }
    void OnEvent(object? s, EventArgs e) { }
}
```
<details><summary>Answer</summary>`pub.Event` holds a delegate referencing `this`. As long as Publisher is alive, the Subscriber is reachable through the event's invocation list. If Publisher is long-lived and Subscribers come and go, they accumulate. Fix: implement IDisposable, unsubscribe in Dispose.</details>

**Q33.** What tool would you use to find what's keeping an object alive in a leak diagnosis?
<details><summary>Answer</summary>`dotnet-dump collect` + `dotnet-dump analyze`, then use the `gcroot` command on the suspected object's address. Shows the chain of references from a GC root to the object. Identifies the culprit (typically a static field, event subscription, timer, or captured closure).</details>

**Q34.** When should you use `WeakReference<T>`?
<details><summary>Answer</summary>When you want to keep a reference IF the object is alive elsewhere, but don't want your reference to keep it alive. Used in caches where stale entries are acceptable, in pub-sub patterns, and event-listener registries. For most caches, bounded `MemoryCache` is simpler and more predictable.</details>

---

## Modern .NET memory APIs

**Q35.** When to use `MemoryPool<T>` vs `ArrayPool<T>`?
<details><summary>Answer</summary>MemoryPool returns IMemoryOwner with `Memory<T>` — async-friendly, integrates with Span/Memory patterns. ArrayPool returns raw arrays. For sync code wanting an array: ArrayPool. For async / API surface using Memory: MemoryPool (or ArrayPool + `.AsMemory()`).</details>

**Q36.** What does `MemoryMarshal.Cast<T1, T2>` do?
<details><summary>Answer</summary>Reinterprets a `Span<T1>` as `Span<T2>`. Lets you view a byte span as an int span (or similar), no copy. Same memory, different type. Used for binary protocol parsing, struct-over-buffer access. Beware alignment + size differences.</details>

**Q37.** Predict the allocations in `await foreach (var x in someAsyncEnumerable) { ... }`:
<details><summary>Answer</summary>Depends on the iterator. `IAsyncEnumerable<T>`'s state machine is a class — typically allocates on first use (the enumerator). With .NET 10's box elision, simple iterators may avoid the allocation. ValueTask&lt;bool&gt; returns make per-iteration overhead near-zero for sync-completing items.</details>

---

## Synthesis

**Q38.** A web API endpoint is taking 200 MB of memory for every request and not releasing. You're told "GC isn't running." What's your debugging plan?
<details><summary>Sample answer</summary>
First, GC IS running — the user means heap isn't shrinking. So it's a leak: objects allocated per request are being held by something.

1. **dotnet-counters** to confirm: gc-heap-size growing over hours.
2. **dotnet-gcdump** at startup and again after 100 requests. Diff.
3. Look at top retained types in the diff. Probably some collection growing.
4. **gcroot** on a specific instance: shows the reference chain. Likely a static cache, an event handler chain, or a captured closure.
5. Apply the fix (bound the cache, unsubscribe, etc.).
6. Repeat the diagnostic to verify.

Most common: a static cache like `Dictionary<RequestKey, ResponseData>` that grows per request.
</details>

**Q39.** Why is `Span<T>` not a class?
<details><summary>Answer</summary>If Span were a class, you could store it in fields, cross await, capture it in lambdas — exposing it to heap and async lifetimes. But Span often points to stack memory (stackalloc) or pinned native memory. The reference would dangle.

Making it a `ref struct` enforces stack-only at compile time. No way for the lifetime to be violated. The cost: restrictions on use. The benefit: safety.

For span-like behavior that CAN cross those boundaries, use `Memory<T>` (which IS a regular struct, slightly slower, no restrictions).
</details>

**Q40.** Sort by allocation cost (cheapest to most expensive): `new byte[1024]`, `stackalloc byte[1024]`, `ArrayPool<byte>.Shared.Rent(1024)`, `new int[2]`, `Marshal.AllocHGlobal(1024)`.
<details><summary>Answer</summary>
- `stackalloc byte[1024]`: ~1 ns (just a stack pointer adjustment). Cheapest.
- `new int[2]`: ~10-30 ns (very small Gen0 allocation).
- `new byte[1024]`: ~20-50 ns (small Gen0 allocation; could hit LOH if larger).
- `ArrayPool<byte>.Shared.Rent(1024)`: ~50-100 ns (hash + bucket lookup), but no GC pressure.
- `Marshal.AllocHGlobal(1024)`: ~100-200 ns (native heap call), no GC tracking; manual free required.

Per-op cost varies by hardware. The big picture: stack < small Gen0 < pool < native. For high-frequency allocations, the pool wins on total cost (allocation + GC), even though per-op it's slower.
</details>

---

→ [Coding.md](Coding.md)
