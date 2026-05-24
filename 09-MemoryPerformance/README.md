# Chapter 09 — Memory & Performance

> The runtime under your code: stack vs heap, the garbage collector, `IDisposable`, `Span<T>`, `Memory<T>`, `ArrayPool`, ref structs, unsafe code, the .NET 10 escape-analysis breakthrough, and how to find memory leaks.

**Prerequisites**: [Chapter 03 (Type System)](../03-TypeSystem/README.md). You need value/reference and structs.

**Time to read**: ~10-12 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-StackVsHeap.md](01-StackVsHeap.md) | The two memory regions, why it matters for performance, what the JIT does to choose between them. |
| [02-GarbageCollection.md](02-GarbageCollection.md) | Generations (0/1/2/LOH/POH), mark-and-sweep, compaction, server vs workstation GC, **DATAS (.NET 10 default)**, ephemeral collections, write barriers. |
| [03-IDisposable.md](03-IDisposable.md) | The full Dispose pattern, `using` statement, `using` declaration, `SafeHandle`. |
| [04-Finalizers.md](04-Finalizers.md) | `~ClassName()`, the finalizer queue, two-generation cost, when to use (almost never), `GC.SuppressFinalize`. |
| [05-Span.md](05-Span.md) | `Span<T>`, `ReadOnlySpan<T>`, stack-only restrictions, why it eliminates allocations on hot paths. |
| [06-Memory.md](06-Memory.md) | `Memory<T>` for heap-friendly span-like use, `Memory<T>.Pin`, when to convert to `Span`. |
| [07-ArrayPool.md](07-ArrayPool.md) | `ArrayPool<T>.Shared`, `Rent`/`Return`, the `clearArray` parameter, when pooling matters. |
| [08-Stackalloc.md](08-Stackalloc.md) | `stackalloc` for temporary buffers, the safe `Span<T>` form, size limits. |
| [09-RefStructsRefLocals.md](09-RefStructsRefLocals.md) | `ref struct`, `ref` locals/returns, `scoped`, the safety analysis the compiler does. |
| [10-UnsafeCode.md](10-UnsafeCode.md) | `unsafe`, pointers, `fixed`, when (rarely) it's worth it, `/unsafe` compiler flag. |
| [11-EscapeAnalysis.md](11-EscapeAnalysis.md) | The .NET 10 JIT improvement: objects that don't escape allocate on the stack. How to encourage it. |
| [12-StringInterning.md](12-StringInterning.md) | The string intern pool, `string.Intern`, `string.IsInterned`, when to use. |
| [13-MemoryLeaks.md](13-MemoryLeaks.md) | Common leaks (static caches, event subscriptions, timers, closures) and a diagnostic playbook. |
| [Questions.md](Questions.md) | ~30 questions. |
| [Coding.md](Coding.md) | ~15 problems: predict allocations, zero-alloc parsing, leak hunting, buffer pool. |

→ Begin: [01-StackVsHeap.md](01-StackVsHeap.md)
