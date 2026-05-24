# BenchmarkDotNet

## What it is

BenchmarkDotNet is the standard .NET library for **rigorous micro-benchmarks**. Naively timing code with `Stopwatch` gives misleading numbers (JIT warmup, GC, CPU scaling, dead-code elimination). BenchmarkDotNet handles all of that — warmup, multiple iterations, statistical analysis, and outlier detection — and reports trustworthy results with allocations.

```csharp
using BenchmarkDotNet.Attributes;
using BenchmarkDotNet.Running;

[MemoryDiagnoser]
public class StringBenchmarks {
    private readonly int[] _data = Enumerable.Range(0, 100).ToArray();

    [Benchmark(Baseline = true)]
    public string Concatenation() {
        string r = "";
        foreach (var n in _data) r += n;
        return r;
    }

    [Benchmark]
    public string StringBuilder() {
        var sb = new System.Text.StringBuilder();
        foreach (var n in _data) sb.Append(n);
        return sb.ToString();
    }
}

// Program.cs
BenchmarkRunner.Run<StringBenchmarks>();
```

```bash
dotnet run -c Release    # MUST be Release
```

---

## Why not just use Stopwatch?

```csharp
// ✗ — naive timing is wrong
var sw = Stopwatch.StartNew();
DoWork();
sw.Stop();
Console.WriteLine(sw.ElapsedMilliseconds);
```

Problems:
- **JIT warmup** — the first calls run unoptimized (Tier 0) then get re-JIT'd (Tier 1). You'd measure compilation, not steady-state.
- **GC interference** — a collection mid-measurement skews results.
- **Dead-code elimination** — the JIT may delete work whose result is unused.
- **No statistics** — one run is noise; you need many + variance analysis.
- **CPU frequency scaling** — turbo/throttling distorts short measurements.

BenchmarkDotNet warms up, runs many iterations across multiple processes, discards outliers, reports mean/median/stddev, and prevents dead-code elimination.

---

## Key attributes

```csharp
[MemoryDiagnoser]                    // report allocations + GC counts per op
[Benchmark]                          // mark a method as a benchmark
[Benchmark(Baseline = true)]         // the reference for relative comparison
[Params(10, 100, 1000)]              // run for each value of a field
[GlobalSetup]                        // run once before all benchmarks (not measured)
[GlobalCleanup]                      // run once after
[IterationSetup]                     // before each iteration (use sparingly)
[Arguments(1, 2)]                    // pass specific args to a benchmark method
```

### `[Params]` — measure across input sizes

```csharp
public class SortBench {
    [Params(100, 10_000, 1_000_000)]
    public int N;

    private int[] _data = null!;

    [GlobalSetup]
    public void Setup() => _data = Enumerable.Range(0, N)
        .Select(_ => Random.Shared.Next()).ToArray();

    [Benchmark]
    public int[] ArraySort() {
        var copy = (int[])_data.Clone();
        Array.Sort(copy);
        return copy;
    }
}
```

This produces a result row for each `N` — revealing how performance scales (O(n log n) etc.).

---

## Reading the output

```
| Method        | N      | Mean       | Error    | StdDev   | Ratio | Gen0   | Allocated |
|-------------- |------- |-----------:|---------:|---------:|------:|-------:|----------:|
| Concatenation | 100    | 1,234.5 ns | 12.3 ns  | 11.5 ns  | 1.00  | 2.5000 |   10480 B |
| StringBuilder | 100    |   234.5 ns |  2.3 ns  |  2.1 ns  | 0.19  | 0.1000 |     456 B |
```

| Column | Meaning |
|---|---|
| **Mean** | Average time per operation |
| **Error** | Half of the 99.9% confidence interval |
| **StdDev** | Standard deviation across runs |
| **Ratio** | Relative to the `Baseline` (0.19 = 5× faster) |
| **Gen0/1/2** | GC collections per 1000 ops |
| **Allocated** | Bytes allocated per op (`[MemoryDiagnoser]`) |

Focus on **Mean** (and its Error — overlapping intervals mean "no significant difference") and **Allocated** (often the real story — fewer allocations = less GC pressure). The Ratio makes comparisons obvious.

---

## Preventing dead-code elimination

The JIT removes computations whose results aren't used. BenchmarkDotNet defeats this if you **return** the result:

```csharp
// ✗ — JIT may delete this entirely (result unused)
[Benchmark]
public void Compute() { var x = ExpensiveCalc(); }

// ✓ — return the value so it can't be eliminated
[Benchmark]
public int Compute() => ExpensiveCalc();
```

Always return the result (or assign it to a `[Benchmark]`-consumed field). For multiple values, combine them or use `Consumer`.

---

## Avoiding common measurement traps

### Don't include setup in the measurement

```csharp
// ✗ — allocating the array is measured too
[Benchmark]
public int Sum() {
    var data = new int[10000];   // allocation skews the result
    return data.Sum();
}

// ✓ — allocate in GlobalSetup
private int[] _data = null!;
[GlobalSetup] public void Setup() => _data = new int[10000];
[Benchmark] public int Sum() => _data.Sum();
```

### Cold cache vs warm cache

A benchmark that reads the same small data repeatedly runs from L1 cache — unrealistically fast vs. production cache-miss patterns. Be aware of working-set size; vary it with `[Params]` if cache effects matter.

### Constant folding

```csharp
[Benchmark]
public int Add() => 2 + 3;   // ✗ — JIT folds to a constant at compile time; measures nothing
```

Use non-constant inputs (fields, `[Arguments]`) so the work actually happens at runtime.

### Inlining hiding the work

Tiny methods get inlined, so you may measure the harness, not the method. For measuring call overhead specifically, `[MethodImpl(MethodImplOptions.NoInlining)]` — but usually you want the realistic inlined behavior.

---

## Comparing strategies

```csharp
[MemoryDiagnoser]
public class LookupBench {
    private Dictionary<int, string> _dict = null!;
    private (int, string)[] _array = null!;

    [GlobalSetup]
    public void Setup() {
        _dict = Enumerable.Range(0, 1000).ToDictionary(i => i, i => i.ToString());
        _array = _dict.Select(kv => (kv.Key, kv.Value)).ToArray();
    }

    [Benchmark(Baseline = true)]
    public string DictLookup() => _dict[500];

    [Benchmark]
    public string LinearScan() {
        foreach (var (k, v) in _array) if (k == 500) return v;
        return "";
    }
}
```

The output quantifies the dictionary's O(1) advantage over the O(n) scan — turning "obviously faster" into a number.

---

## Diagnosers and exporters

```csharp
[MemoryDiagnoser]                          // allocations
[DisassemblyDiagnoser(printSource: true)]  // show the JIT asm
[ThreadingDiagnoser]                       // lock contention, completed work items
[EventPipeProfiler(EventPipeProfile.GcVerbose)]  // GC events for trace tools
```

Results export to Markdown, HTML, CSV, JSON automatically (in `BenchmarkDotNet.Artifacts/`). The `DisassemblyDiagnoser` is great for verifying the JIT did (or didn't) vectorize/inline.

---

## Common bugs / gotchas

### Running in Debug

BenchmarkDotNet **refuses** to run optimized benchmarks in Debug and warns loudly — Debug numbers are meaningless. Always `dotnet run -c Release`.

### Benchmarking inside the test project

Benchmarks are a separate concern from unit tests; put them in their own console project (`Microsoft.NET.Sdk`, `OutputType=Exe`). Running them via `dotnet test` doesn't work the same way.

### Trusting tiny differences

If the Mean intervals (Mean ± Error) overlap, the difference is **not statistically significant** — don't claim one is faster. Look at the Error/StdDev.

### Micro-benchmark not reflecting real use

A method fast in isolation may be slow in context (contention, cache misses, larger data). Validate with whole-app profiling (see [Chapter 15 §08](../15-BuildTooling/08-Profiling.md)) — micro-benchmarks complement, don't replace, profiling.

---

## Summary

- BenchmarkDotNet gives **rigorous** micro-benchmarks: warmup, many iterations, statistics, allocation reporting — far more reliable than `Stopwatch`.
- Always run in **Release** in a dedicated console project.
- Use `[MemoryDiagnoser]` (allocations), `[Benchmark(Baseline=true)]` (Ratio), `[Params]` (scaling), `[GlobalSetup]` (un-measured setup).
- **Return** results to prevent dead-code elimination; avoid constant folding and measuring setup.
- Read **Mean** (with Error — overlapping = no real difference) and **Allocated** (often the real story).
- Micro-benchmarks complement profiling; validate wins at the whole-app level.

→ Next: [Questions.md](Questions.md)
