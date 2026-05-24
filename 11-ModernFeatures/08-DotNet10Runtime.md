# .NET 10 Runtime — what changed under the hood

> DATAS GC by default, escape analysis with stack promotion, async box elision, smarter PGO and inlining, faster startup. .NET 10 is the LTS through November 2028.

The language features in [§07](07-CSharp14.md) get the headlines. The runtime improvements in .NET 10 are equally important — they change perf and memory behavior without any code changes.

---

## DATAS — Dynamically Adapting To Application Sizes

```xml
<!-- .csproj — DATAS is the default in .NET 10 -->
<PropertyGroup>
  <GarbageCollectionAdaptationMode>1</GarbageCollectionAdaptationMode>
</PropertyGroup>
```

DATAS (introduced as opt-in in .NET 8, default in .NET 9, refined in .NET 10) is a GC mode that:
- Sizes heap regions **per application's working set** rather than a fixed default.
- Adapts as the app's allocation pattern changes (startup vs steady-state).
- Reduces memory by 20-40% for small/medium apps; comparable throughput for large ones.

Before DATAS, server GC reserved large heap chunks based on machine RAM. A small ASP.NET app on a 64-GB server would reserve gigabytes. With DATAS:
- Heap regions are 4-MB chunks (vs older single-large-heap segments).
- The GC adds/removes regions as needed.
- Total heap mirrors actual working set + headroom.

### When to keep / disable

- **Keep on** (default) for: ASP.NET Core APIs, microservices, console tools, anything memory-conscious.
- **Disable** for: throughput-critical apps with predictable large allocations (analytics engines, scientific computing) where DATAS's adaptiveness has measurable cost.

Disable:
```xml
<GarbageCollectionAdaptationMode>0</GarbageCollectionAdaptationMode>
```

See [Chapter 09 §02](../09-MemoryPerformance/02-GarbageCollection.md).

---

## Escape analysis and stack promotion

```csharp
void Hot() {
    var buffer = new byte[256];   // ⚡ .NET 10: JIT may stack-allocate this
    Fill(buffer);
    Use(buffer);
}
```

If the JIT can prove a reference-type allocation **doesn't escape** the method (not stored in fields, not returned, not captured by lambdas surviving the call), it allocates the object on the **stack** instead of the heap. Zero GC pressure.

### Coverage in .NET 10

Stack-promotable today:
- Local `new T[]` (sized constants).
- Local `new SomeClass()` for small types that don't escape.
- Local boxing operations (limited).
- Some lambda closures (when capture lifetime is local).

Not yet:
- Reference-counted shared objects.
- Anything passed to a virtual method (the JIT must devirtualize first).
- Anything passed to async methods that hold across awaits.

The JIT logs (with `DOTNET_JitDisasm`) show `[stack alloc]` annotations for promoted objects.

### Implications

You no longer need to manually use `stackalloc` / `ArrayPool` for many simple cases. Just write idiomatic code; the JIT optimizes.

For **hot, allocation-sensitive paths**, explicit `Span<T>` and `ArrayPool` still beat the JIT (predictable, no fallback to heap). Benchmark both.

See [Chapter 09 §11](../09-MemoryPerformance/11-EscapeAnalysis.md).

---

## Async state machine box elision

Async methods compile to a state machine struct. If the method ever awaits, the runtime boxes the struct to the heap. .NET 10 skips boxing when:
- The method completes synchronously.
- All awaited tasks are already completed.

```csharp
async Task<int> Cached(string key) {
    if (_cache.TryGetValue(key, out var v))
        return v;     // ⚡ no box allocation in .NET 10
    return await Fetch(key);
}
```

For "fast path" returns, allocations drop to zero. ASP.NET Core middleware and caching APIs see large wins.

Combined with `ValueTask<T>`, fast-path async is now nearly free.

See [Chapter 08 §02](../08-Concurrency/02-AsyncAwaitFundamentals.md).

---

## Improved PGO (Profile-Guided Optimization)

**Dynamic PGO** instruments tier-0 code, observes hot paths, and recompiles them with profile-informed optimizations:
- Virtual call → direct call (devirtualization).
- Hot blocks moved earlier.
- Cold blocks (e.g., exception throws) moved out of the main path.

.NET 10 makes Dynamic PGO **default** for all release configurations. .NET 8 had it opt-in; .NET 9 made it default for some scenarios; .NET 10 universal.

Combined with **Static PGO** (compile-time profiles), the JIT now matches the perf of C++ for many workloads.

Real-world: ASP.NET Core's Plaintext benchmark gained another ~10% in .NET 10 vs .NET 9.

---

## JIT improvements

- **Loop cloning and unrolling** for `Span<T>` indexing.
- **Branch elimination** based on inlined constants.
- **Vector256/Vector512** automatic vectorization for `Span<T>` ops in many BCL methods.
- **`Math.X` intrinsics** for AVX-512 where available.

Net effect: hot loops over spans run 2-3× faster on modern CPUs without writing intrinsics.

---

## ReadyToRun and crossgen2

- **R2R** (Ready-to-Run) AOT-compiles assemblies to native code at publish time. JIT skips startup compilation.
- **crossgen2** is the cross-platform R2R compiler.

.NET 10 improvements:
- Smaller R2R blobs.
- R2R + Dynamic PGO combine: startup uses R2R native code, then DPO recompiles hot methods to optimized code.

Result: ASP.NET Core cold start drops from ~500 ms to ~150 ms in many configurations.

---

## NativeAOT improvements

NativeAOT compiles the full application to a single native executable. No JIT, no runtime codegen.

.NET 10:
- Smaller binaries (-30% for typical web APIs vs .NET 9).
- Faster compile times.
- Better trimming compatibility — more BCL paths annotated for trimming.
- Improved diagnostics (better error messages for AOT-incompatible code).

See [Chapter 14 §04](../14-InteropAOT/04-NativeAOT.md).

---

## BCL additions worth knowing

- **`Task.WhenEach`** (introduced .NET 9, stable in 10): stream tasks as they complete via `IAsyncEnumerable<Task>`.
- **`OrderedDictionary<TKey,TValue>`**: long-overdue generic ordered dictionary.
- **`System.Numerics.Tensors`**: native-acceleration tensor primitives.
- **`System.Text.Json`** source generation faster, supports more polymorphic scenarios.
- **`Lock` type** (C# 13) now used everywhere in BCL internally.
- **Channels v2**: lower-overhead `Channel<T>` for high-throughput pipelines.

---

## Networking and HTTP

- HTTP/3 stable and on by default.
- QUIC client and server improvements (lower latency, better congestion control).
- `SocketsHttpHandler` connection pooling optimizations.

---

## Cryptography

- Post-quantum algorithms (ML-KEM, ML-DSA) supported via `System.Security.Cryptography`.
- Faster `Span<byte>`-based hashing APIs.
- AES-GCM and ChaCha20-Poly1305 hardware acceleration when available.

---

## Diagnostics and observability

- **`dotnet-counters`** improved metric names and dimensions.
- **`dotnet-trace`** can capture managed thread stack samples without breakpoints.
- **EventPipe** has lower overhead — production-safe profiling.
- **OpenTelemetry** native integration with reduced overhead.

See [Chapter 15 §07-08](../15-BuildTooling/07-Debugging.md).

---

## LTS commitment

.NET 10 is **LTS** — supported through **November 2028** (3 years).

Pattern:
- Even-numbered .NET versions (6, 8, 10) → LTS, 3-year support.
- Odd-numbered (7, 9, 11) → STS, 18 months.

For production projects, target LTS. .NET 10 is the recommended baseline today.

---

## Migration from .NET 8 / 9

For most apps:
1. Bump `<TargetFramework>net10.0</TargetFramework>`.
2. Update NuGet packages to .NET 10-compatible versions.
3. Run analyzers — fix warnings.
4. Benchmark hot paths — you'll likely see gains for free.

Watch for:
- Breaking changes in trimming / AOT (more code paths now flagged as incompatible).
- DATAS heap behavior — memory usage might decrease but heap shape is different. Tools like dotMemory may show different snapshots.

---

## Summary of .NET 10

**Big runtime wins**:
- DATAS GC default — smaller heaps.
- Escape analysis + stack promotion — fewer heap allocations.
- Async box elision — fast-path async is free.
- Dynamic PGO universal — JIT matches C++ for hot code.
- HTTP/3 stable.

**Tools**:
- Better R2R and NativeAOT.
- Faster dotnet CLI startup.
- Improved diagnostics.

**Support**:
- LTS through Nov 2028.

Combined with C# 14 language features ([§07](07-CSharp14.md)), .NET 10 is the strongest .NET release to date. New projects should start here.

→ Next: [Questions.md](Questions.md)
