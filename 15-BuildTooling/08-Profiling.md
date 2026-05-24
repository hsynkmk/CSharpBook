# Profiling

## What it is

Profiling measures **where your program spends time and memory** so you optimize the right things. The cardinal rule: **measure, don't guess.** Intuition about hot paths is frequently wrong; a profiler shows the truth.

```
Profiling answers:
- Which methods consume the most CPU?  (CPU/sampling profiler)
- What's allocating and triggering GC? (memory/allocation profiler)
- Why is latency spiky?                (GC pauses, lock contention, async stalls)
```

---

## The diagnostic CLI tools

Cross-platform, low-overhead, production-safe. Install as global tools:

```bash
dotnet tool install -g dotnet-counters
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-gcdump
```

### `dotnet-counters` — live metrics

```bash
dotnet-counters monitor -p <pid>
dotnet-counters monitor -p <pid> System.Runtime Microsoft.AspNetCore.Hosting
```

Real-time counters, near-zero overhead:
- `cpu-usage`, `working-set`, `gc-heap-size`
- `gen-0/1/2-gc-count`, `time-in-gc`
- `alloc-rate` (bytes allocated/sec)
- `threadpool-thread-count`, `threadpool-queue-length`
- `exception-count`

The **first** tool to reach for: is the problem CPU, memory/GC, thread-pool starvation, or exceptions? `dotnet-counters` tells you which direction to investigate.

```bash
dotnet-counters collect -p <pid> --format json   # export for later analysis
```

### `dotnet-trace` — CPU and event traces

```bash
dotnet-trace collect -p <pid>                       # default profile (CPU sampling)
dotnet-trace collect -p <pid> --profile gc-verbose  # detailed GC events
dotnet-trace collect -p <pid> --providers Microsoft-DotNETCore-SampleProfiler
```

Produces a `.nettrace` (convert with `--format speedscope` for the speedscope viewer). Analyze in PerfView, Visual Studio, or speedscope.app. Low overhead — safe to run against production.

### `dotnet-gcdump` — managed heap snapshot

```bash
dotnet-gcdump collect -p <pid>   # lightweight; pauses briefly
```

Open the `.gcdump` in Visual Studio to see object counts, sizes, and **retention paths** (what keeps objects alive). The go-to for managed memory leaks — compare two snapshots over time to find what's growing. See [Chapter 09 §13](../09-MemoryPerformance/13-MemoryLeaks.md).

---

## PerfView

The deep, free Windows profiler from Microsoft. Heavy but powerful: ETW-based CPU stacks, GC analysis, allocation tick stacks, JIT stats, exceptions.

Typical use:
- **CPU Stacks** — flame-graph-like view of where CPU goes (find hot methods).
- **GC Heap Alloc Stacks** — which call stacks allocate the most (find allocation hot spots).
- **GCStats** — pause times, gen sizes, allocation rates.

PerfView's learning curve is steep, but it's the most thorough free tool for diving into a `.nettrace`/ETW capture. On non-Windows, use `dotnet-trace` + speedscope / Visual Studio.

---

## Visual Studio Diagnostic Tools

Built into VS (Debug → Performance Profiler):
- **CPU Usage** — sampling profiler with a call tree and flame graph.
- **.NET Object Allocation** — every allocation by type and call stack.
- **Memory Usage** — heap snapshots with diff (find growth).
- **Database / async / events** tools.

Integrated and approachable — best starting point on Windows for app-level profiling during development.

---

## JetBrains dotTrace / dotMemory

Commercial, polished:
- **dotTrace** — CPU profiling (sampling, tracing, timeline). Timeline mode correlates CPU, GC, threads, locks, async over time — excellent for latency analysis.
- **dotMemory** — memory profiling with snapshot diffs, retention graphs, automatic leak detection inspections.

If you do regular performance work, these are worth the license for their ergonomics.

---

## BenchmarkDotNet — micro-benchmarks

For measuring the performance of a specific method (not a whole app), use BenchmarkDotNet. It handles warmup, multiple iterations, statistical rigor, and reports allocations.

```csharp
[MemoryDiagnoser]
public class StringBench {
    private readonly string[] _parts = Enumerable.Range(0, 100).Select(i => i.ToString()).ToArray();

    [Benchmark(Baseline = true)]
    public string Concat() {
        string r = "";
        foreach (var p in _parts) r += p;   // O(n^2) allocations
        return r;
    }

    [Benchmark]
    public string Builder() {
        var sb = new StringBuilder();
        foreach (var p in _parts) sb.Append(p);
        return sb.ToString();
    }
}

// Program.cs
BenchmarkRunner.Run<StringBench>();
```

`[MemoryDiagnoser]` reports allocations per op — crucial for spotting GC pressure. Always benchmark in **Release**. See [Chapter 16 §07](../16-Testing/07-BenchmarkDotNet.md) for full coverage.

---

## A profiling workflow

1. **Reproduce** the problem with a representative workload.
2. **Triage with `dotnet-counters`** — CPU-bound? GC-heavy? Thread-pool starved? Exception storm?
3. **CPU-bound** → `dotnet-trace`/PerfView CPU stacks → find hot methods → optimize the actual hot path.
4. **Allocation/GC-heavy** → allocation profiler (VS / dotMemory / PerfView alloc stacks) → reduce allocations (pooling, `Span`, struct, avoid LINQ in hot loops).
5. **Memory growing** → `dotnet-gcdump` snapshots over time → `gcroot` retention → fix the leak.
6. **Verify** with a micro-benchmark (BenchmarkDotNet) that the change actually helps.
7. **Re-measure** the whole app — local wins can be globally irrelevant.

---

## Reading allocation data

Common allocation culprits the profiler reveals:
- **Boxing** — value types passed as `object` (incl. some old APIs, `string.Format` args).
- **LINQ in hot paths** — closures, iterators, delegate allocations.
- **String concatenation** in loops — use `StringBuilder`.
- **`async` state machines** that don't fast-path (mitigated by .NET 10 box elision — see [Chapter 11 §08](../11-ModernFeatures/08-DotNet10Runtime.md)).
- **Large arrays on the LOH** — use `ArrayPool`.

The fix toolbox: `Span<T>`/`Memory<T>`, `ArrayPool<T>`, `stackalloc`, `struct`, `readonly struct`, source generators, and caching. See [Chapter 09](../09-MemoryPerformance/README.md).

---

## Common bugs / gotchas

### Profiling a Debug build

Debug builds aren't optimized — numbers are meaningless for perf decisions. Always profile **Release**.

### Optimizing without measuring

The root sin. You'll often optimize a method that's 0.1% of runtime while the real cost is elsewhere (I/O, a chatty DB query, a lock). Profile first.

### Micro-benchmark that doesn't reflect reality

A method fast in isolation may be slow in context (cache effects, contention). Validate whole-app impact, not just the micro-benchmark.

### Observer effect

Heavy profilers (especially instrumenting/tracing modes) slow the program and can distort relative timings. Prefer low-overhead sampling (`dotnet-trace`, `dotnet-counters`) for production; reserve heavy instrumentation for dev.

### Ignoring GC pause time

High throughput with long GC pauses still means bad tail latency. Watch `time-in-gc` and pause durations, not just averages — p99 matters for services.

---

## Summary

- **Measure, don't guess.** Profile in Release with a realistic workload.
- Start with **`dotnet-counters`** to triage (CPU vs GC vs thread-pool vs exceptions).
- **`dotnet-trace`** / PerfView / VS for CPU hot spots; allocation profilers (VS / dotMemory / PerfView) for GC pressure.
- **`dotnet-gcdump`** snapshots + `gcroot` for memory leaks.
- **BenchmarkDotNet** (`[MemoryDiagnoser]`) to verify micro-optimizations rigorously.
- Reduce allocations with `Span`/`ArrayPool`/`stackalloc`/structs; then re-measure the whole app.
- Watch tail latency and GC pauses, not just averages.

→ Next: [Questions.md](Questions.md)
