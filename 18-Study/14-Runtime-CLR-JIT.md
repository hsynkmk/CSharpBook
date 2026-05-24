# 14 — Runtime, CLR & JIT

## ⚡ 30-second answer

Your C# compiles to **IL** (Intermediate Language) + metadata in an assembly. At runtime the **CLR** (CoreCLR) loads it and the **JIT** compiles IL → native machine code **just before each method first runs**. The CLR provides the GC, type system, exception handling, and thread pool. Modern .NET uses **tiered compilation** (quick tier-0 first for fast startup, then optimized tier-1 for hot methods) and can do **Native AOT** (compile everything to native ahead of time — fast startup, no JIT, but restrictions). One runtime, many app models, cross-platform.

---

## Core mechanics

**Compilation pipeline**:
```
C# source ──Roslyn──▶ IL + metadata (.dll) ──JIT (at runtime, per method)──▶ native code
```

**Tiered JIT**:
- **Tier 0** — compiles fast with minimal optimization → fast startup.
- **Tier 1** — recompiles **hot** methods (called often) with full optimization in the background.
- **OSR** (On-Stack Replacement) lets long-running loops jump to optimized code mid-execution.
- **Dynamic PGO** — profiles tier-0 runs and uses the data to optimize tier-1 (e.g., devirtualize the common type).

**Ahead-of-time options**:
- **ReadyToRun (R2R)** — precompiles IL to native *alongside* the IL at publish → faster startup, JIT still available for hot paths. No restrictions.
- **Native AOT** — compiles the whole app to a native binary, **no JIT, no IL** at runtime → fastest startup, low memory, small self-contained binary. But: **no runtime codegen** (`Reflection.Emit`), limited reflection, no plugins, implies trimming.

**CLR pieces**: type loader, JIT, **GC** ([13](13-MemoryAndGC.md)), exception handler, **thread pool** ([09](09-Threading-and-Tasks.md)), metadata/reflection, assembly loader.

**Assemblies & loading**: a `.dll` carries IL + metadata + a manifest. **`AssemblyLoadContext`** allows isolated/unloadable loading (plugins).

**P/Invoke**: call native (C) functions via `[LibraryImport]` (source-generated, AOT-friendly) / `[DllImport]` — marshals managed↔native data (blittable types pass without copying).

---

## Comparison tables

| Mode | Startup | Runtime perf | Restrictions | Use |
|---|---|---|---|---|
| **JIT** (default) | medium (tiered) | best (re-optimizes hot) | none | most apps |
| **ReadyToRun** | fast | best (JIT still runs) | none | startup-sensitive servers |
| **Native AOT** | fastest | good (no re-opt) | no reflection-emit/dynamic, trim-only | CLIs, serverless, containers |

| | .NET Framework | .NET (Core / 5+) |
|---|---|---|
| Platform | Windows-only | cross-platform |
| Releases | in-box, legacy | side-by-side, LTS/STS cadence |
| Status | maintenance | active (current: .NET 10 LTS) |

---

## 🪤 Traps & gotchas

- **"C# is interpreted"** — wrong. It compiles to IL, then JITs to **native** code; it's not bytecode-interpreted like default Python.
- **Cold-start cost**: first call to a method pays JIT time → noticeable in serverless. R2R/AOT mitigate.
- **AOT breaks reflection-heavy code**: serializers/DI/plugins using runtime reflection or `Reflection.Emit` fail under AOT — use source generators (System.Text.Json source-gen) and heed trim warnings ([21](21-Deployment-Perf-Tooling.md)).
- **`.NET Standard` vs `.NET`**: `.NET Standard` is a compatibility *contract* for sharing libraries across Framework and Core; for new code target `net10.0`.
- **Assembly version/binding mismatches** cause `FileLoadException`/`TypeLoadException` at load time — not compile time.
- **Trimming removes reflection-reached code** silently → runtime `MissingMethod` in the trimmed build (test it) ([21](21-Deployment-Perf-Tooling.md)).

---

## ❓ Likely questions

**Q: What happens from C# source to execution?**
A: Roslyn compiles C# to IL + metadata in an assembly. At runtime the CLR loads it and the JIT compiles each method's IL to native code on first use; the GC, type system, and thread pool support execution.

**Q: What is the JIT and tiered compilation?**
A: The Just-In-Time compiler turns IL into native code at runtime. Tiered: tier-0 compiles fast for startup, tier-1 re-optimizes hot methods in the background (with OSR and dynamic PGO).

**Q: JIT vs AOT vs ReadyToRun?**
A: JIT compiles at runtime (best steady-state). AOT compiles everything ahead of time (fastest startup, no JIT, but restricts reflection/dynamic). R2R precompiles alongside IL for fast startup while keeping the JIT — no restrictions.

**Q: Is C# compiled or interpreted?**
A: Compiled — to IL, then JIT-compiled to native machine code. Not interpreted.

**Q: What does the CLR provide?**
A: Garbage collection, the type system, JIT, exception handling, the thread pool, metadata/reflection, and assembly loading.

**Q: When would you use Native AOT?**
A: Cold-start-sensitive or resource-constrained scenarios (CLI tools, serverless, high-density microservices) where its restrictions (no reflection-emit, limited reflection, trimming) are acceptable.

**Q: .NET Framework vs .NET (Core)?**
A: Framework is Windows-only/legacy/maintenance; modern .NET is cross-platform, side-by-side, actively developed (current .NET 10 LTS). New code targets modern .NET.

---

## 🎓 Senior Extra

- **Dynamic PGO** (default .NET 8+): tier-0 instruments execution; tier-1 uses the profile to devirtualize/inline the common case (guarded) — "the JIT learns your workload."
- **Generic code sharing**: one native body for all reference-type `T`; a specialized body per value-type `T` (monomorphization) → `List<int>` is box-free and fast ([01](01-TypeSystem.md)).
- **`AssemblyLoadContext`** enables plugin isolation and **unloading** (collectible contexts); leaks happen if something outside the context references loaded types.
- **`SuppressGCTransition`** / blittable types make P/Invoke calls cheaper by skipping the GC transition for fast native calls — advanced interop perf.
- **Method tables / vtables** back virtual dispatch ([02](02-OOP.md)); `sealed` enables devirtualization.
- **Startup tuning**: R2R + trimming for servers; AOT for serverless; `TieredPGO`, `TieredCompilationQuickJit` knobs in runtimeconfig.
- **Metadata & reflection**: types/members are described by metadata tokens; reflection reads them at runtime — powerful but slow (cache `MethodInfo`) and AOT-hostile (prefer source generators).

→ Deeper: [`../DotNetBook/01-Runtime/`](../DotNetBook/01-Runtime/README.md)
