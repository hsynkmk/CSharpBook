# Chapter 15 — Build & Tooling — Q & A

---

### Q1. SDK vs runtime?

The **runtime** runs .NET apps (end users need it for framework-dependent apps). The **SDK** builds them — includes the runtime plus Roslyn compilers, MSBuild, CLI, and templates. Developers install the SDK.

---

### Q2. What does `dotnet run` do that `dotnet build` doesn't?

`dotnet run` builds **and executes** the project (dev convenience). `dotnet build` only compiles to `bin/`. For deployment, `publish` then run the output directly to skip the build step each launch. Pass app args after `--`.

---

### Q3. Global vs local tools?

Global tools (`dotnet tool install -g`) are available everywhere, one version per machine. Local tools are pinned per-repo in a `.config/dotnet-tools.json` manifest, so the whole team and CI use the same version (`dotnet tool restore`). Prefer local for projects.

---

### Q4. How do you check for vulnerable dependencies?

`dotnet list package --vulnerable --include-transitive` lists known CVEs (including transitive). Enable `<NuGetAudit>true</NuGetAudit>` to warn at restore. Essential CI hygiene.

---

### Q5. SDK-style vs legacy csproj?

SDK-style (`<Project Sdk="...">`) includes source files implicitly, uses `<PackageReference>`, and is ~10 lines. Legacy listed every file and used `packages.config`. All modern .NET uses SDK-style.

---

### Q6. `TargetFramework` vs `TargetFrameworks`?

Singular `<TargetFramework>` takes one TFM. Plural `<TargetFrameworks>` (note the s) takes a `;`-separated list for multi-targeting, producing an output per framework. Using singular with a list silently misbehaves.

---

### Q7. What is `netstandard2.0` for?

A common API surface implemented by both .NET Framework 4.6.1+ and .NET Core, giving libraries maximum reach across runtimes. Still relevant for widely-consumed libraries; apps and modern libraries target `net10.0`.

---

### Q8. What do `Directory.Build.props` and `.targets` do?

MSBuild auto-imports them for all projects below their directory — `.props` first (defaults), `.targets` last (overrides/added targets). They apply repo-wide settings (Nullable, LangVersion, analyzers) without repeating them in each `.csproj`.

---

### Q9. Why `PrivateAssets="all"` on a package reference?

It prevents the dependency from flowing transitively to consumers — used for analyzers, source generators, and build-only tools that aren't runtime dependencies.

---

### Q10. What is a NuGet lock file and why use it?

`packages.lock.json` (enabled via `RestorePackagesWithLockFile`) pins exact resolved versions including transitive. Commit it and restore with `--locked-mode` in CI for reproducible builds — no surprise transitive version bumps.

---

### Q11. What is Central Package Management?

`Directory.Packages.props` with `<ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>` declares versions once for the whole solution via `<PackageVersion>`; projects reference packages without versions. Keeps versions consistent across many projects.

---

### Q12. How does NuGet resolve conflicting transitive versions?

It picks the **lowest version that satisfies all constraints** (nearest-wins + lowest-applicable). This can surprise you; inspect with `--include-transitive` and force a version with an explicit `<PackageReference>` if needed.

---

### Q13. What is Package Source Mapping and why does it matter?

Configuration in `nuget.config` that restricts which feed a package pattern comes from (e.g., `MyCompany.*` only from your private feed). Defends against dependency-confusion attacks where a malicious public package shadows an internal name.

---

### Q14. The four MSBuild building blocks?

**Properties** (scalar variables, `$(...)`), **Items** (file lists, `@(...)`, with `%(...)` metadata), **Targets** (named, ordered steps), **Tasks** (units of work like `Copy`, `Exec`, `Csc`).

---

### Q15. How do you make a custom MSBuild target incremental?

Declare `Inputs` and `Outputs`. If outputs are newer than inputs, MSBuild skips the target. Without them, the target runs on every build, slowing things down.

---

### Q16. Best way to debug why a build did something unexpected?

The **binary log**: `dotnet build -bl` produces `msbuild.binlog`; open it in the MSBuild Structured Log Viewer to see every property, item, target, and task. Also `/pp` to preprocess the fully-expanded project.

---

### Q17. What are Roslyn analyzers?

Compiler plugins that inspect code during compilation and report diagnostics (with optional code fixes), shown in the IDE and CI. Built-in `CAxxxx`/`IDExxxx` rules plus NuGet packages (StyleCop, Roslynator) enforce correctness, style, performance, and security.

---

### Q18. Where do you configure analyzer severity?

In `.editorconfig`: `dotnet_diagnostic.CA2007.severity = error`. The modern replacement for `.ruleset` files; lets you set severity per-rule and per-scope (e.g., relax rules for tests).

---

### Q19. How do you enforce code style at build time (not just IDE)?

Set `<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>` so `IDExxxx` style rules fire during `dotnet build`. Gate CI with `dotnet format --verify-no-changes`.

---

### Q20. How does `.editorconfig` discovery work?

Editors/compiler walk up from a file's directory applying all `.editorconfig` files until one with `root = true`. Closer files override farther ones, so you can relax rules for subfolders (tests, generated code).

---

### Q21. What's the cardinal rule of profiling?

**Measure, don't guess.** Intuition about hot paths is usually wrong. Profile (in Release, with a realistic workload) before optimizing — you'll often find the cost is somewhere unexpected (I/O, a query, a lock).

---

### Q22. Which tool do you reach for first when an app is slow/heavy?

`dotnet-counters monitor` — near-zero overhead, shows CPU, GC heap size, allocation rate, time-in-gc, thread-pool queue, exception count. It tells you the *direction* (CPU vs GC vs thread-pool vs exceptions) to investigate.

---

### Q23. How do you diagnose a managed memory leak in production?

`dotnet-gcdump collect` snapshots over time; open in Visual Studio and compare to find what's growing. In a dump, `dumpheap -stat` + `gcroot <address>` shows object counts and retention paths (what keeps them alive).

---

### Q24. How do you diagnose a hang or deadlock in production?

`dotnet-stack report -p <pid>` (or `dotnet-dump` + `clrthreads`/`pstacks`) prints managed stacks of all threads. Look for threads blocked on `.Result`/`.Wait()` — the sync-over-async deadlock pattern.

---

### Q25. Why must you profile/benchmark in Release?

Debug builds disable optimizations (no inlining, extra checks), so timings are meaningless for perf decisions. Use Release; BenchmarkDotNet refuses to run optimized benchmarks in Debug for this reason.

---

### Q26. What does `[MemoryDiagnoser]` add to a BenchmarkDotNet benchmark?

It reports bytes allocated per operation and GC collections per generation — crucial for spotting allocation/GC pressure that raw timing alone hides.

---

→ Next: [Coding.md](Coding.md)
