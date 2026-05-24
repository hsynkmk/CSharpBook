# Native AOT

## What it is

Native AOT (Ahead-Of-Time) compiles your .NET app to a **single, self-contained native executable** at publish time. No JIT, no IL, no .NET runtime installed on the target — just machine code, like a C++ program.

```xml
<PropertyGroup>
  <PublishAot>true</PublishAot>
</PropertyGroup>
```

```bash
dotnet publish -r win-x64 -c Release
# produces a single native .exe, no runtime dependency
```

The output is a native binary with the GC and minimal runtime statically linked in. There's no JIT compiler in the final image.

---

## Why it exists — the benefits

| Benefit | Detail |
|---|---|
| **Fast startup** | No JIT warmup. Starts in ~milliseconds vs ~tens-hundreds ms. |
| **Low memory** | No JIT, smaller runtime footprint. |
| **Small size** | Trimmed + AOT: tens of MB (or less), self-contained. |
| **No runtime install** | Target machine needs nothing — single file. |
| **Predictable perf** | No JIT/tiering jitter; consistent latency. |
| **Harder to decompile** | Native code, no IL. |

Ideal for: CLI tools, serverless/Lambda functions (cold-start sensitive), microservices, containers, and anywhere a small, fast-starting, dependency-free binary matters.

---

## The cost — limitations

Native AOT trades flexibility for these wins. Things that **don't work**:

1. **No runtime code generation**:
   - `Reflection.Emit` / `DynamicMethod` → throws.
   - `Expression.Compile()` → throws `PlatformNotSupportedException` (it interprets in some cases but can't JIT).
   - `dynamic` (DLR) → mostly broken.
   - Runtime assembly loading (`Assembly.LoadFile` of new code) → not supported.

2. **Limited reflection**:
   - Reflection works only on what the AOT compiler can statically determine is used. Unbounded reflection (`Type.GetType(userString)`, `MakeGenericType` over unseen types) may throw at runtime.
   - Requires `[DynamicallyAccessedMembers]` annotations to preserve members.

3. **Trimming is mandatory** — unused code is removed (see [05-Trimming.md](05-Trimming.md)). Code that relies on reflection over trimmed members breaks.

4. **No `C++/CLI`, limited COM** — use `[GeneratedComInterface]` for COM.

5. **Platform-specific** — you publish per RID (`win-x64`, `linux-arm64`, etc.); no "build once run anywhere."

---

## What works well

- **`[LibraryImport]`** P/Invoke (source-generated marshalling).
- **Source generators** — STJ source-gen, `[GeneratedRegex]`, logging source-gen, `[GeneratedComInterface]`.
- **Statically-known generics**.
- **Most of the BCL** (collections, LINQ, async, networking).
- **ASP.NET Core** minimal APIs (with the AOT-friendly subset).

The recipe for AOT-compatible code: **replace runtime reflection with source generators.** This is why STJ, regex, logging, and config binding all ship source generators.

---

## Building an AOT app

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>   <!-- optional, smaller -->
    <StackTraceSupport>false</StackTraceSupport>            <!-- optional, smaller -->
  </PropertyGroup>
</Project>
```

```bash
dotnet publish -r linux-x64 -c Release
```

Requirements:
- Platform native toolchain (clang/gcc on Linux, MSVC build tools on Windows, Xcode on macOS).
- A Runtime Identifier (`-r`).

The build emits **trim and AOT warnings** (`IL2xxx`, `IL3xxx`) for code that may not work. Treat them seriously — they predict runtime failures.

---

## AOT warnings

```
IL2026: Using member 'X' which has 'RequiresUnreferencedCodeAttribute'
IL3050: Using member 'Y' which has 'RequiresDynamicCodeAttribute'
```

- **`RequiresUnreferencedCode`** — uses reflection that trimming can break.
- **`RequiresDynamicCode`** — needs runtime codegen (Expression.Compile, MakeGenericType in some cases).

Annotate your own APIs that have these requirements so callers get warned:

```csharp
[RequiresDynamicCode("Uses Expression.Compile")]
public Func<T> BuildFactory<T>() => Expression.Lambda<Func<T>>(...).Compile();
```

---

## Diagnosing what breaks

Enable AOT analyzers even before publishing AOT (catches issues during normal builds):

```xml
<IsAotCompatible>true</IsAotCompatible>   <!-- for libraries: enables AOT/trim analyzers -->
```

For apps, `<PublishAot>true</PublishAot>` enables the analyzers. Fix all `IL2xxx`/`IL3xxx` warnings before relying on the AOT build.

---

## .NET 10 AOT improvements

- Smaller binaries (~30% smaller than .NET 9 for typical APIs).
- Faster AOT compilation.
- More BCL paths annotated as AOT-safe.
- Better error messages pointing to the offending code.
- ASP.NET Core's AOT-compatible surface expanded.

See [Chapter 11 §08](../11-ModernFeatures/08-DotNet10Runtime.md).

---

## Size comparison (typical "hello world" web API)

| Mode | Size | Startup | Runtime needed |
|---|---|---|---|
| Framework-dependent | ~few hundred KB | ~100-200 ms | Yes (.NET installed) |
| Self-contained | ~70 MB | ~100-200 ms | No |
| Self-contained + trimmed | ~25 MB | ~80 ms | No |
| ReadyToRun (R2R) | larger | faster startup than pure JIT | depends |
| **Native AOT** | **~10-15 MB** | **~10-30 ms** | **No** |

AOT wins on startup and self-contained size. Pure JIT may win peak throughput on long-running servers (more aggressive runtime optimizations like dynamic PGO), so AOT isn't always faster at steady state.

---

## AOT vs JIT throughput

- **Startup**: AOT wins big (no warmup).
- **Steady-state throughput**: JIT can win — Dynamic PGO (.NET 10) re-optimizes hot paths at runtime using real profiles; AOT optimizes at build time only.

So: AOT for short-lived processes (CLI, serverless, frequent restarts). JIT (possibly with R2R) for long-running high-throughput servers where startup is amortized.

---

## Common bugs / gotchas

### Reflection over user-supplied type names

```csharp
Type? t = Type.GetType(configValue);   // ⚠ — t may be null under AOT (type trimmed)
```

The AOT compiler can't see that `configValue` will name a type, so it may trim it. Use source generators or explicitly preserve types.

### Expression.Compile in a dependency

A NuGet package using `Expression.Compile` (older serializers, mappers, validators) throws under AOT. Check dependencies for AOT compatibility; prefer source-generated alternatives (e.g., Mapperly over AutoMapper, STJ source-gen over reflection).

### Generic virtual methods / unbounded generics

`MakeGenericType(runtimeType)` over a type combination not seen at compile time can throw. Constrain generics to statically-known instantiations.

### Assuming stack traces

`<StackTraceSupport>false</StackTraceSupport>` (sometimes default for size) removes detailed stack traces. Keep it on if you need diagnostics.

---

## When to use Native AOT

- CLI tools (fast startup, single binary, no install).
- Serverless functions (cold-start sensitive — Lambda, Azure Functions).
- Containers/microservices (small image, fast scale-up).
- Constrained/embedded environments.

When **not** to:
- Apps relying on reflection-heavy frameworks (some ORMs, older DI, plugins).
- Long-running servers where JIT + PGO peak throughput matters more than startup.
- Rapid development (AOT publish is slower than JIT debug builds — develop on JIT, publish AOT).

---

## Summary

- Native AOT compiles to a single self-contained native binary — no JIT, no runtime install.
- Wins: fast startup (~ms), low memory, small size, predictable latency. Great for CLI/serverless/containers.
- Costs: no runtime codegen (`Reflection.Emit`, `Expression.Compile`, `dynamic`), limited reflection, mandatory trimming, per-RID builds.
- The fix for AOT compatibility is **source generators** instead of runtime reflection.
- Heed `IL2xxx`/`IL3xxx` warnings; annotate APIs with `RequiresDynamicCode`/`RequiresUnreferencedCode`.
- JIT + Dynamic PGO may beat AOT on steady-state throughput; AOT wins startup.

→ Next: [05-Trimming.md](05-Trimming.md)
