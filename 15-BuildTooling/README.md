# Chapter 15 — Build & Tooling

> The CLI, csproj, NuGet, MSBuild, analyzers, EditorConfig, debugging, profiling. The stuff around the code that makes a productive developer.

**Prerequisites**: Chapter 00 (Introduction).

**Time to read**: ~6-8 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-DotnetCLI.md](01-DotnetCLI.md) | `dotnet new`, `restore`, `build`, `run`, `test`, `publish`, `pack`, `tool`, what each does and when to use it. |
| [02-Csproj.md](02-Csproj.md) | The SDK-style `.csproj`: `<PropertyGroup>`, `<ItemGroup>`, `<PackageReference>`, target frameworks, multi-targeting. |
| [03-NuGet.md](03-NuGet.md) | Consuming packages, publishing your own, central package management, `.nuspec` vs csproj-as-package, lock files. |
| [04-MSBuild.md](04-MSBuild.md) | What MSBuild is, targets/tasks/properties, custom build steps, `Directory.Build.props`/`.targets`. |
| [05-RoslynAnalyzers.md](05-RoslynAnalyzers.md) | Analyzers, code-fix providers, ruleset configuration, writing your own analyzer, treating warnings as errors. |
| [06-EditorConfig.md](06-EditorConfig.md) | `.editorconfig` for C# style: naming rules, formatting, severity per-rule, repo-wide consistency. |
| [07-Debugging.md](07-Debugging.md) | Breakpoints, conditional breakpoints, the watch window, debugger visualizers, `dotnet-dump`, `dotnet-trace`. |
| [08-Profiling.md](08-Profiling.md) | `dotnet-counters`, PerfView, JetBrains dotMemory/dotTrace, finding allocations and hot spots. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~10 problems: write a custom analyzer, optimize an allocation-heavy method using profiler data. |

→ Begin: [01-DotnetCLI.md](01-DotnetCLI.md)
