# 21 — Deployment, Performance & Tooling — Coding Questions

> Find the bug / pick the tool. (Concepts: [21-Deployment-Perf-Tooling.md](21-Deployment-Perf-Tooling.md))

---

### Q1 — Misleading benchmark
```csharp
var sw = Stopwatch.StartNew();
var result = Compute();          // result unused
Console.WriteLine(sw.ElapsedMilliseconds);
```
<details><summary>Answer</summary>

Two problems: (1) **JIT warmup** — first call includes compilation time. (2) **Dead-code elimination** — if `result` is unused, the JIT may delete the work entirely (measuring ~0). **Fix:** use **BenchmarkDotNet** (warmup, multiple iterations, consumes results) in **Release**, not a hand-rolled Stopwatch in Debug.
</details>

---

### Q2 — Allocation in a hot loop
```csharp
string Build(int[] data) {
    string s = "";
    foreach (var x in data) s += x + ",";    // ?
    return s;
}
```
<details><summary>Answer</summary>

**O(n²) allocations** — `s += ...` creates a new string each iteration (strings are immutable). For large arrays this dominates GC. **Fix:** `StringBuilder` or `string.Join(",", data)`. Confirm with allocation profiling (`dotnet-trace` / `[MemoryDiagnoser]`).
</details>

---

### Q3 — Which tool?
```
"Memory keeps climbing in production on a Linux container and never comes back down."
```
<details><summary>Answer</summary>

**`dotnet-dump`** (EventPipe-based, works on Linux): `dotnet-dump collect`, then `dumpheap -stat` (what's eating the heap) → `gcroot <addr>` (what's keeping it alive — static/event/cache). A growing, unreclaimed heap = a **leak** (unintended reachability). Capture two dumps over time to see what's growing.
</details>

---

### Q4 — Which tool?
```
"The app is unresponsive under load but CPU is low."
```
<details><summary>Answer</summary>

**`dotnet-counters`** first — check **ThreadPool Queue Length + Thread Count**. Climbing both with low CPU = **thread-pool starvation** (likely sync-over-async blocking pool threads). Confirm with a **`dotnet-dump`** showing threads stuck on `.Result`/`Wait`. Fix: async all the way. ([12](12-Concurrent-Parallel-AsyncBugs-Coding.md))
</details>

---

### Q5 — AOT breaks at runtime
```csharp
// Published with PublishAot=true; uses reflection-based JSON:
var obj = JsonSerializer.Deserialize<Order>(json);
```
<details><summary>Answer</summary>

May **fail at runtime** (type/members trimmed) — reflection-based serialization is AOT-hostile. **Fix:** System.Text.Json **source generation** (`[JsonSerializable(typeof(Order))]` + a `JsonSerializerContext`). Always **test the AOT/trimmed build**; heed trim warnings. ([14](14-Runtime-CLR-JIT-Coding.md))
</details>

---

### Q6 — Dockerfile layer caching
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
COPY . .                 # (a) copy everything
RUN dotnet restore       # (b)
RUN dotnet publish -o /app
```
<details><summary>Answer</summary>

Poor caching — copying **all** source before `restore` means **any** code change busts the restore layer → re-downloads all NuGet packages every build. **Fix:** copy the `.csproj` first, `restore`, *then* copy the rest:
```dockerfile
COPY *.csproj ./
RUN dotnet restore
COPY . .
RUN dotnet publish -o /app
```
</details>

---

### Q7 — Stateful app + scale-out
```csharp
// In-memory session dictionary, app scaled to 5 K8s replicas behind a load balancer.
static Dictionary<string, Session> _sessions = new();
```
<details><summary>Answer</summary>

**Breaks across replicas** — a user's requests can hit different pods, which don't share `_sessions` (and a pod restart loses them). **Fix:** externalize state to a **distributed cache** (Redis) or DB; keep app instances **stateless** so they scale horizontally. ([18](18-Caching-Resilience-Http-Coding.md))
</details>

---

### Q8 — BenchmarkDotNet correctness (senior)
```csharp
[Benchmark] public void DoWork() { var x = Fib(30); }   // ?
```
<details><summary>Answer</summary>

The result is **unused** → may be eliminated by the JIT (benchmark measures nothing). **Fix:** **return** it — `[Benchmark] public int DoWork() => Fib(30);` (BenchmarkDotNet consumes returned values). Also add `[MemoryDiagnoser]` (allocations often matter more than time) and run in Release, not attached.
</details>
