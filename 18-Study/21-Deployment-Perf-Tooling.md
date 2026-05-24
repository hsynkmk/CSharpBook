# 21 — Deployment, Performance & Tooling

## ⚡ 30-second answer

**Deployment**: publish **framework-dependent** (needs .NET installed, small) or **self-contained** (bundles the runtime, runs anywhere); ship in **Docker** (multi-stage build, small **chiseled** base image) to **Kubernetes** (Deployment + Service + liveness/readiness probes). **AOT/trimming/R2R** trade flexibility for startup/size. **Performance**: the rule is **measure, don't guess** — profile to find the real bottleneck, then fix it. Use **BenchmarkDotNet** for micro-comparisons and **`dotnet-counters/trace/dump`** for live diagnosis. The usual culprits: excessive **allocations** (GC pressure), **N+1 queries**, and **sync-over-async** (thread-pool starvation).

---

## Core mechanics

**Publish modes**:
```bash
dotnet publish -c Release                                # framework-dependent (needs runtime)
dotnet publish -c Release -r linux-x64 --self-contained  # bundles the runtime (needs a RID)
dotnet publish -c Release -r linux-x64 -p:PublishReadyToRun=true   # faster startup, full JIT
dotnet publish -c Release -r linux-x64 -p:PublishAot=true          # native, fastest startup, restricted
```

**Docker** — multi-stage (build on SDK image, run on small runtime image):
```dockerfile
FROM mcr.microsoft.com/dotnet/sdk:10.0 AS build
... dotnet restore (cache) ... dotnet publish -o /app
FROM mcr.microsoft.com/dotnet/aspnet:10.0-noble-chiseled   # tiny, no shell, non-root, low CVEs
COPY --from=build /app .
```

**Kubernetes** maps your app to a **Deployment** (replicas, rolling updates), **Service** (stable endpoint), and **probes**:
- **liveness** `/alive` → restart if unhealthy; **readiness** `/health` → stop routing if not ready ([20](20-Observability-Messaging-Background.md)).
- Config/secrets via env vars (`__` for nesting); apps must be **stateless** to scale out + honor SIGTERM.

**Performance methodology**: define the metric → **baseline** → **profile** → fix one thing → **re-measure** → stop.

**Tools**:
| Tool | Use |
|---|---|
| **BenchmarkDotNet** | micro "is A faster than B?" (handles warmup/GC, reports allocations) |
| **dotnet-counters** | live triage — GC, allocation rate, **thread-pool queue/threads**, exceptions |
| **dotnet-trace** | CPU/allocation hot spots (flame graph) |
| **dotnet-dump** | heap snapshot — `dumpheap`/`gcroot` to find **leaks** |
| **PerfView / dotMemory / dotTrace** | deep GC/CPU/memory analysis |

---

## Comparison tables

| Mode | Startup | Perf | Restrictions |
|---|---|---|---|
| JIT (default) | medium | best (re-opt) | none |
| **ReadyToRun** | fast | best | none (JIT still runs) |
| **Native AOT** | fastest | good | no reflection-emit/dynamic, trim-only ([14](14-Runtime-CLR-JIT.md)) |

| Symptom (counters) | Likely cause |
|---|---|
| high allocation rate / frequent Gen2 | allocations / GC pressure |
| thread-pool queue + thread count climbing | **sync-over-async starvation** ([12](12-Concurrent-Parallel-AsyncBugs.md)) |
| high CPU | hot method → capture a trace |
| memory grows, never reclaimed | **leak** → dump + `gcroot` ([13](13-MemoryAndGC.md)) |

---

## 🪤 Traps & gotchas

- **Optimizing without measuring** — intuition about bottlenecks is usually wrong. Profile first; the hot path is rarely where you think.
- **Benchmarking in Debug / with `Stopwatch`** — Debug disables optimizations; naive timing ignores JIT warmup and dead-code elimination. Use BenchmarkDotNet in Release.
- **AOT breaks reflection** — serializers/DI/plugins using runtime reflection fail; use source generators and heed trim warnings; test the trimmed/AOT build.
- **`new HttpClient()` per request** / **sync-over-async** / **N+1** — the recurring whole-app killers ([18](18-Caching-Resilience-Http.md), [12](12-Concurrent-Parallel-AsyncBugs.md), [17](17-EFCore.md)).
- **Full-resolution images / huge buffers** — allocation/LOH pressure; pool with `ArrayPool<T>` ([13](13-MemoryAndGC.md)).
- **Stateful app + scale-out** — in-memory state breaks across K8s replicas; externalize to cache/DB.
- **Liveness probe depending on the DB** → restart loops when the DB blips; liveness should be cheap, readiness checks dependencies.
- **Shipping Debug builds / no trimming** — slow/large. Ship Release; trim/AOT where compatible.

---

## ❓ Likely questions

**Q: Framework-dependent vs self-contained?**
A: Framework-dependent needs .NET installed (small, runtime patched by host). Self-contained bundles the runtime (larger, runs anywhere for a target RID). Choose by environment control.

**Q: What's the #1 performance rule?**
A: Measure, don't guess. Baseline → profile to find the real bottleneck → fix → re-measure. Optimizing by intuition wastes effort and adds complexity.

**Q: Which tool for which problem?**
A: BenchmarkDotNet for micro comparisons; dotnet-counters for live triage; dotnet-trace for CPU/allocation hot spots; dotnet-dump for heap/leaks. These work cross-platform (Linux/containers).

**Q: How do you find a memory leak?**
A: `dotnet-dump` → `dumpheap -stat` (what's eating the heap) → `gcroot <addr>` (what's keeping it alive — static, event, cache). A .NET leak is unintended reachability.

**Q: JIT vs AOT vs ReadyToRun for deployment?**
A: JIT = best steady-state, medium startup. R2R = fast startup, full JIT, no restrictions. AOT = fastest startup/smallest, but no runtime codegen/limited reflection.

**Q: Liveness vs readiness probes?**
A: Liveness = is the process alive (restart if not). Readiness = can it serve traffic now incl. dependencies (stop routing if not). Don't tie liveness to dependencies.

**Q: What are the most common .NET performance issues?**
A: Excessive allocations (GC pressure), N+1 database queries, and sync-over-async causing thread-pool starvation. Caching misses and reflection in hot paths follow.

---

## 🎓 Senior Extra

- **Allocation is the GC lever**: cost scales with allocation *rate*, not heap size. Reduce with `Span<T>`/`stackalloc`/`ArrayPool<T>`, avoid LINQ/closures/boxing in hot loops, use `ValueTask` for sync-completing hot paths ([13](13-MemoryAndGC.md)).
- **Server GC + DATAS** for throughput on multi-core servers; tune `GCLatencyMode` for latency-critical sections ([13](13-MemoryAndGC.md)).
- **ETW vs EventPipe**: PerfView (deepest, Windows, ETW) vs the cross-platform `dotnet-*` tools (EventPipe) — both read `EventSource` events the runtime emits (GC/JIT/threadpool/HTTP/EF).
- **CI/CD**: build → test → containerize → push (pinned tag, not `latest`) → deploy (Helm/`azd`/kubectl); gate on tests + vulnerability scans; secrets via the CI store/OIDC.
- **Rollout strategies**: rolling (default), blue/green (instant rollback), canary (% traffic + watch metrics) — driven by observability ([20](20-Observability-Messaging-Background.md)).
- **Trimming** removes unused IL (smaller) but can strip reflection-reached members → runtime failures; annotate (`DynamicallyAccessedMembers`) or prefer source generators; **always test the trimmed artifact**.
- **Chiseled images**: distroless-style (no shell/package manager, non-root, minimal CVEs) — production default; pair `runtime-deps` chiseled with self-contained/AOT for the smallest image.
- **BenchmarkDotNet rigor**: `[MemoryDiagnoser]` (allocations often matter more than time), `[Params]` for scaling, baseline ratios; return results to avoid dead-code elimination; Release only.

→ Deeper: [`../DotNetBook/19-Deployment/`](../DotNetBook/19-Deployment/README.md), [`../DotNetBook/21-Performance/`](../DotNetBook/21-Performance/README.md)
