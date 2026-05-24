# Chapter 00 — Questions

> Drilling questions for everything in Chapter 00. Aim for 80%+ before moving on.

---

## Quick-fire (one-line answers)

**Q1.** What does the acronym CLR stand for?
<details><summary>Answer</summary>Common Language Runtime.</details>

**Q2.** What does the acronym BCL stand for?
<details><summary>Answer</summary>Base Class Library.</details>

**Q3.** What does the SDK include that the runtime doesn't?
<details><summary>Answer</summary>The C# compiler (Roslyn), the `dotnet` CLI, project templates, MSBuild, and NuGet client. The runtime only contains what's needed to execute compiled code.</details>

**Q4.** What is IL?
<details><summary>Answer</summary>Intermediate Language — the stack-based bytecode the C# compiler produces, executed (after JIT compilation) by the CLR.</details>

**Q5.** What does the JIT do?
<details><summary>Answer</summary>The Just-In-Time compiler translates IL to native machine code when a method is first called.</details>

**Q6.** What does Native AOT skip?
<details><summary>Answer</summary>The JIT. Native AOT compiles everything to machine code at publish time; no JIT happens at runtime.</details>

**Q7.** What's the current LTS version of .NET?
<details><summary>Answer</summary>**.NET 10** (November 2025). It will receive support until November 2028.</details>

**Q8.** Which versions of .NET are LTS, and how long do they get support?
<details><summary>Answer</summary>Even-numbered versions (.NET 6, 8, 10) get 3 years of LTS support. Odd-numbered (7, 9) get 18 months of Standard-Term Support (STS).</details>

**Q9.** What's the difference between .NET Framework 4.8 and .NET 10?
<details><summary>Answer</summary>They're different runtimes. .NET Framework is Windows-only legacy (no new features after 4.8.1). Modern .NET (5+) is cross-platform and actively developed.</details>

**Q10.** What does `dotnet --info` show?
<details><summary>Answer</summary>The installed SDK version(s), the OS, all installed runtimes, and tool/global.json info.</details>

---

## Conceptual

**Q11.** What are the four "big pieces" of the .NET platform stack?
<details><summary>Answer</summary>Operating system → CLR (runtime) → BCL (base library) → your application. (Frameworks like ASP.NET Core sit between BCL and your app.)</details>

**Q12.** Explain tiered compilation in your own words.
<details><summary>Answer</summary>The JIT compiles methods in two passes. Tier 0 is fast and produces unoptimized code so startup is quick. When the runtime detects a method is "hot" (called frequently), it re-compiles to Tier 1 with full optimization. OSR can swap a tier-0 method to tier-1 mid-loop without restarting.</details>

**Q13.** Why is backwards compatibility so emphasized in C#?
<details><summary>Answer</summary>Code written 20 years ago should still compile. This forces new features to be additive — usually via contextual keywords, opt-in flags (like NRT), or compiler-version-gated syntax — so existing code never breaks.</details>

**Q14.** What's `global.json` and when do you use it?
<details><summary>Answer</summary>A file that pins a specific .NET SDK version (or roll-forward policy) for a folder. Useful when you have multiple SDKs installed and want a particular one for one project.</details>

**Q15.** Name three pieces a NuGet package can contain.
<details><summary>Answer</summary>Compiled DLLs, metadata (.nuspec), MSBuild props/targets files, content files, source code, native libraries.</details>

---

## Comparison

**Q16.** What's the practical difference between "framework-dependent" and "self-contained" publish?
<details><summary>Answer</summary>Framework-dependent: small output (your DLLs only); target machine must have the .NET runtime installed. Self-contained: the runtime is bundled; runs on any matching OS without a separate install, but the output is much larger (~70MB+).</details>

**Q17.** When would you choose Native AOT over JIT?
<details><summary>Answer</summary>Fast startup (CLI tools, serverless), smaller deployment, deterministic memory use, and contexts that disallow dynamic code generation (some embedded scenarios). Trade-offs: no reflection emit, no dynamic loading, more setup work for reflection.</details>

**Q18.** When would you target `netstandard2.0` instead of `net10.0`?
<details><summary>Answer</summary>When building a library that needs to run on both .NET Framework and modern .NET. netstandard is a common API surface they both implement. New code rarely needs it — most libraries target net8.0 or net10.0 directly today.</details>

**Q19.** What's the difference between `dotnet build` and `dotnet publish`?
<details><summary>Answer</summary>`build` produces a debug-ready binary in `bin/`. `publish` produces a deploy-ready output: optimized, sometimes self-contained, optionally single-file or AOT.</details>

**Q20.** What's the difference between `Microsoft.NET.Sdk` and `Microsoft.NET.Sdk.Web`?
<details><summary>Answer</summary>Different MSBuild SDKs. `.Web` adds defaults for ASP.NET Core projects: implicit usings for AspNetCore namespaces, package references to the shared framework, content folders (wwwroot), etc.</details>

---

## Three ways to write Main

**Q21.** Write the same "Hello, World" program three different ways: classic Main, top-level statements, and a file-based app.
<details><summary>Answer</summary>

Classic Main:
```csharp
using System;
namespace HelloApp {
    public class Program {
        public static void Main(string[] args) {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

Top-level statements (C# 9+):
```csharp
Console.WriteLine("Hello, World!");
```

File-based app (C# 14+), saved as `hello.cs`, run via `dotnet run hello.cs`:
```csharp
Console.WriteLine("Hello, World!");
```

(The contents of the file-based and top-level versions are identical — the difference is whether a `.csproj` exists.)
</details>

**Q22.** What are all the valid `Main` signatures?
<details><summary>Answer</summary>
```csharp
static void Main()
static void Main(string[] args)
static int Main()
static int Main(string[] args)
static async Task Main()
static async Task Main(string[] args)
static async Task<int> Main()
static async Task<int> Main(string[] args)
```
</details>

**Q23.** Can you have multiple files with top-level statements in one project?
<details><summary>Answer</summary>No — only one file per project. Other files use regular classes.</details>

---

## csproj

**Q24.** What property in the csproj controls Nullable Reference Types? What values can it take?
<details><summary>Answer</summary>`<Nullable>`. Values: `enable` (NRT on), `disable` (NRT off, pre-C# 8 behavior), `annotations` (honor `?` but don't emit warnings), `warnings` (emit warnings but ignore annotations).</details>

**Q25.** What does `<ImplicitUsings>enable</ImplicitUsings>` do?
<details><summary>Answer</summary>The SDK adds a default set of `using` directives (System, System.IO, System.Linq, System.Collections.Generic, System.Threading.Tasks, etc.) to every file in the project, so you don't have to type them.</details>

**Q26.** How do you add a NuGet package to a project from the command line?
<details><summary>Answer</summary>`dotnet add package PackageName` (optionally `--version 1.2.3`). This edits the csproj and runs restore.</details>

**Q27.** What's `Directory.Build.props`?
<details><summary>Answer</summary>A file at (or above) project level that's auto-imported into every csproj in the subtree. Used to share common properties and items across many projects in one repo.</details>

**Q28.** What's the purpose of Central Package Management?
<details><summary>Answer</summary>To declare NuGet package *versions* in one central file (`Directory.Packages.props`) so individual csproj files don't need version numbers. Avoids drift across projects when updating.</details>

---

## Tooling

**Q29.** What's the `dotnet-counters` global tool used for?
<details><summary>Answer</summary>Live performance counters from a running .NET process: GC stats, exceptions, lock contention, thread-pool queue length, etc. Great first-line diagnostic.</details>

**Q30.** What's the difference between the C# extension and C# Dev Kit in VS Code?
<details><summary>Answer</summary>"C# extension" is the base Roslyn-powered language server. "C# Dev Kit" is the higher-level toolset (solution explorer, test runner, AI features) that builds on top of it. Installing Dev Kit pulls in both.</details>

**Q31.** What's the recommended way to install the .NET 10 SDK on Linux?
<details><summary>Answer</summary>Use the official Microsoft package feed: `wget` the `packages-microsoft-prod.deb` (Ubuntu/Debian) and `apt install dotnet-sdk-10.0`. On Fedora: `dnf install dotnet-sdk-10.0`.</details>

---

## Identifying problems

**Q32.** A coworker says "I installed .NET 10 SDK but `dotnet --version` still says 8.0". What are the likely causes?
<details><summary>Answer</summary>(1) They restarted the wrong terminal; PATH wasn't refreshed. (2) A `global.json` in the folder pins to 8.0. (3) Multiple SDKs are installed and an older one is being picked due to PATH order. Run `dotnet --list-sdks` to see all installed, and `where dotnet` / `which dotnet` to see which is being used.</details>

**Q33.** Why does `dotnet new console` produce different `Program.cs` content than it did 5 years ago?
<details><summary>Answer</summary>The default template was updated to use top-level statements (C# 9+). You can still opt into the classic style with `dotnet new console --use-program-main`.</details>

**Q34.** Why might `dotnet run hello.cs` not work?
<details><summary>Answer</summary>File-based apps are a C# 14 / .NET 10 feature (Nov 2025). On older SDKs the command fails. Check with `dotnet --version`.</details>

---

## Open-ended

**Q35.** Explain in one paragraph why a managed runtime (like the CLR) trades some performance for safety.
<details><summary>Sample answer</summary>A managed runtime garbage-collects memory automatically, checks array bounds, type-safety enforces, and provides exception unwinding. Each of those checks costs a few CPU cycles compared to raw C, but eliminates entire categories of bugs (use-after-free, buffer overruns, type confusion) that produce security vulnerabilities and crashes. Modern JITs claw back most of the gap by optimizing hot paths, eliminating redundant checks, and inlining aggressively, leaving the safety benefits dominant for the vast majority of applications.</details>

**Q36.** A junior on your team asks "what's the difference between .NET, .NET Core, and .NET Framework?" Give them a 30-second answer.
<details><summary>Sample answer</summary>".NET Framework is the original Windows-only platform from 2002 — version 4.8 is the last one and it's not getting new features. .NET Core was the cross-platform rewrite that started in 2016. In 2020 Microsoft dropped 'Core' from the name; now there's just '.NET'. The current version is .NET 10, it runs everywhere, and it's the one you should use for all new code."</details>

---

→ [Coding.md](Coding.md) — hands-on problems
