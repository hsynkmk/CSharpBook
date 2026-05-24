# The .NET Ecosystem

## What it is

C# is a language. **.NET** is the platform that runs it. Understanding the platform is just as important as knowing the language — the runtime decides how your program is loaded, JIT-compiled, executed, and cleaned up.

The platform has many pieces. Let's name them.

---

## The four big pieces

```
┌────────────────────────────────────────────────────────┐
│  Your application                                      │
│  (source code: .cs files compiled to IL)               │
└─────────────────────┬──────────────────────────────────┘
                      │ depends on
┌─────────────────────┴──────────────────────────────────┐
│  BCL — Base Class Library                              │
│  (System.*, System.Collections.*, System.IO.*, ...)    │
└─────────────────────┬──────────────────────────────────┘
                      │ executed by
┌─────────────────────┴──────────────────────────────────┐
│  CLR — Common Language Runtime                         │
│  (JIT, GC, type system, exception handler, threading)  │
└─────────────────────┬──────────────────────────────────┘
                      │ runs on
┌─────────────────────┴──────────────────────────────────┐
│  Operating System (Windows, Linux, macOS, ...)         │
└────────────────────────────────────────────────────────┘
```

1. **CLR (Common Language Runtime)** — The virtual machine. In modern .NET this is called **CoreCLR**. It loads assemblies, JIT-compiles IL to machine code, manages memory via the GC, handles exceptions, and enforces type safety.

2. **BCL (Base Class Library)** — The standard library. Every type in the `System.*` namespace, plus `Microsoft.*` core types. `int`, `string`, `List<T>`, `Dictionary<K,V>`, `File`, `HttpClient`, `Task` — all live here. The BCL is shipped *with* the runtime.

3. **SDK (Software Development Kit)** — The build tools. Compiler (`csc`), CLI (`dotnet`), MSBuild, NuGet client, templates. The SDK is what you install to *write* C#; the runtime is what you install to *run* compiled C#.

4. **Frameworks** built on top: ASP.NET Core (web), EF Core (database), MAUI (cross-platform UI), Blazor (web UI), etc. These are NuGet packages that depend on the BCL and runtime.

---

## .NET Framework vs .NET Core vs modern .NET

A lot of confusion lives here. Let's untangle it.

### .NET Framework (1.0 – 4.8.1, **legacy**)
- The original .NET, **Windows-only**.
- Latest version: **4.8.1** (Aug 2022). No new features after that.
- Still gets security patches.
- Used in enterprise apps that haven't been migrated.
- **Don't use for new code.**

### .NET Core (1.0 – 3.1, **superseded**)
- Microsoft's rewrite for cross-platform (Windows, Linux, macOS).
- Versions 1.0 (2016) through 3.1 (2019).
- Officially superseded.

### "Modern .NET" (5+, **current**)
Starting with version 5 (Nov 2020), the "Core" name was dropped. There's just **.NET** now. Versions:

| Version | Released | Support type | EOL |
|---|---|---|---|
| .NET 5 | Nov 2020 | Current (non-LTS) | May 2022 (gone) |
| .NET 6 | Nov 2021 | **LTS** | Nov 2024 (gone) |
| .NET 7 | Nov 2022 | STS | May 2024 (gone) |
| .NET 8 | Nov 2023 | **LTS** | Nov 2026 |
| .NET 9 | Nov 2024 | STS | May 2026 |
| **.NET 10** | **Nov 2025** | **LTS** | **Nov 2028** |

The pattern: **even-numbered versions are LTS** (3 years of support); odd are STS (Short-Term Support, 18 months). Always prefer LTS for production unless you specifically want a feature in an STS release.

**.NET 10 is the current LTS as of this book's writing.** Examples in the book target it.

---

## How a program runs

Here's what happens when you `dotnet run` a console app:

1. **Compile-time** (only when source changes)
   - `csc` (the C# compiler, part of Roslyn) reads your `.cs` files.
   - It produces an assembly: a `.dll` (or `.exe`) containing **IL** (Intermediate Language) bytecode + metadata + resources.
   - Errors here are *compiler* errors: type mismatches, missing references, syntax.

2. **Load** (every time the program starts)
   - The CLR is bootstrapped.
   - Your assembly is loaded along with all referenced assemblies (System.Private.CoreLib, your NuGet packages, ...).
   - The CLR walks metadata to set up the type system.

3. **JIT** (just-in-time compilation)
   - When a method is first called, the JIT compiles its IL to native machine code.
   - **Tiered compilation** (default since .NET Core 3.0): methods start at **Tier 0** (fast compile, slow code) and get promoted to **Tier 1** (full optimization) once they're "hot" (called many times).
   - **OSR (On-Stack Replacement)**: a long-running loop in Tier 0 code can switch to Tier 1 *mid-loop*, without restarting the method.
   - **PGO (Profile-Guided Optimization)**: tier 0 collects profiling data; tier 1 uses it to inline hot paths and lay out branches.

4. **Execute**
   - JIT'd machine code runs natively, full speed.
   - The CLR continues to manage memory (GC), threads, exceptions.

5. **Garbage collection**
   - Periodically, the GC pauses the program to reclaim unreachable heap memory.
   - .NET 10 uses **DATAS** (Dynamically Adapting To Application Sizes) by default — the GC auto-tunes its heap targets based on observed allocation patterns.

6. **Shutdown**
   - Finalizers run for objects with `~Dtor`.
   - The CLR unloads, the process exits.

### Native AOT alternative

If you publish with `PublishAot=true`:
- The compiler + ILCompiler produce a **single native binary** at publish time.
- No JIT at runtime. Startup is instant (~10ms vs ~80ms typical).
- Trade-off: no dynamic code generation, no reflection emit, smaller available API surface.
- See [Chapter 14 — Native AOT](../14-InteropAOT/04-NativeAOT.md).

---

## IL: a peek behind the curtain

When you write:

```csharp
public int Add(int a, int b) => a + b;
```

The compiler produces IL roughly like:

```il
.method public hidebysig instance int32 Add(int32 a, int32 b) cil managed
{
  IL_0000: ldarg.1    // load 'a' onto the evaluation stack
  IL_0001: ldarg.2    // load 'b'
  IL_0002: add        // pop two, push sum
  IL_0003: ret        // return the value on the stack
}
```

IL is **stack-based** — instructions push and pop a virtual evaluation stack. The JIT translates this to register-based machine code for your CPU.

You don't normally need to read IL, but knowing it exists demystifies some things — `delegate.Invoke`, generics, async state machines all become more concrete when you've seen the IL once.

Tool: **`dotnet ildasm`** or **SharpLab.io** (online) to inspect IL.

---

## The framework / runtime / SDK relationship

When you install ".NET" you can install:

- **Runtime only** — enough to *run* compiled apps. Used in production servers and Docker images.
- **ASP.NET Core runtime** — runtime + ASP.NET Core dependencies. For web servers.
- **SDK** — runtime + compiler + CLI + templates. For developers.

The SDK *includes* multiple runtimes. Check what you have:

```bash
dotnet --info
```

Output lists installed runtimes and SDKs.

---

## NuGet: the package ecosystem

.NET's package manager is **NuGet**. NuGet packages contain:
- Compiled `.dll`(s).
- Metadata (`.nuspec`).
- Optionally: source code, MSBuild props/targets, content files.

You reference packages in your `.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="Serilog" Version="4.1.0" />
</ItemGroup>
```

When you `dotnet restore`, NuGet downloads the package from a feed (default: `nuget.org`) and resolves its transitive dependencies.

Modern .NET ships with thousands of free, high-quality packages. The BCL covers fundamentals; NuGet covers everything else.

---

## Cross-platform reality

.NET runs on:
- Windows (x86, x64, ARM64)
- Linux (x64, ARM64, several distros officially supported)
- macOS (x64, ARM64 / Apple Silicon)
- iOS, Android (via MAUI)
- Browser (via Blazor WebAssembly — compiles IL to WASM)

Cross-platform code is the default. The runtime team works hard to make I/O, threading, and platform-specific APIs work the same everywhere. A few APIs are platform-specific (e.g., `System.Drawing.Common` is Windows-only; the registry APIs only do anything on Windows) — these are guarded by `[SupportedOSPlatform]` attributes that warn you at compile time.

---

## Common confusions

**"Is .NET still Windows-only?"** — No. Hasn't been since .NET Core 1.0 in 2016.

**"What's the difference between .NET Framework 4.8 and .NET 8?"** — They're different runtimes. .NET Framework is Windows-only legacy; modern .NET (5+) is cross-platform.

**"What does 'runtime' mean exactly?"** — Two things, confusingly:
1. *The CLR* — the executable that runs IL.
2. *A runtime package* — a NuGet-style bundle of (1) plus the BCL.

**"What's the difference between CLR and JIT?"** — JIT is a component of the CLR. CLR = the whole VM (JIT + GC + type system + ...). JIT = the part that compiles IL → machine code.

**"Mono? Xamarin? Are those .NET?"** — Yes, kind of. Mono was the original open-source .NET implementation (started 2001). Xamarin built mobile tooling on Mono. Microsoft acquired Xamarin in 2016 and gradually merged everything. **In modern .NET (6+), the Mono runtime and CoreCLR runtime live side-by-side** in the same product — CoreCLR for servers/desktops, Mono for mobile/Wasm/embedded.

---

## What's new in .NET 10 runtime

.NET 10 brings several runtime-level improvements that affect *every* C# program, not just code using new language features:

| Improvement | Effect |
|---|---|
| **DATAS GC default** | GC auto-tunes per workload; better for cloud + containers |
| **Escape analysis** | JIT promotes some heap allocations to stack |
| **JIT block reordering** | Hot paths laid out for better i-cache use (3-opt TSP) |
| **Reduced write barriers** | Less GC bookkeeping in allocation-heavy code |
| **Async state-machine box elision** | Common async patterns allocate less |
| **`Task.WhenEach`** | Stream completing tasks as they finish |

Chapter 11 §08 has the full deep-dive. For now: upgrading the runtime gives you free performance improvements with no code changes.

---

## Summary

- C# the language → compiled to IL → run by the CLR → with the BCL → built and managed by the SDK + NuGet.
- Modern .NET (6+) is cross-platform; .NET Framework is Windows-only legacy.
- LTS releases (.NET 6, 8, 10) get 3 years of support; STS releases get 18 months.
- JIT compiles IL on demand; Native AOT compiles ahead of time.
- The platform improves itself every November release — your code gets faster for free.

→ Next: [03-DevelopmentSetup.md](03-DevelopmentSetup.md)
