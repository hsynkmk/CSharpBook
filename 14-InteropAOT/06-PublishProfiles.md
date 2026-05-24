# Publish Profiles & Deployment Modes

## What it is

`dotnet publish` produces a deployable artifact. The mode determines size, startup speed, whether the .NET runtime must be installed, and whether the output is portable or platform-specific.

```bash
dotnet publish -c Release -r linux-x64 --self-contained
```

The main axes:
1. **Framework-dependent** vs **Self-contained** — is the runtime bundled?
2. **Portable** vs **RID-specific** — one build for all, or per-platform?
3. **Single-file**, **Trimmed**, **ReadyToRun**, **AOT** — optimizations layered on top.

---

## Framework-dependent deployment (FDD)

```bash
dotnet publish -c Release
```

The output contains **only your app's DLLs** — the .NET runtime must be installed on the target machine.

```xml
<SelfContained>false</SelfContained>   <!-- default -->
```

| Pros | Cons |
|---|---|
| Tiny output (just your code) | Requires matching .NET runtime installed |
| Shared runtime across apps (less disk) | Version mismatches possible |
| Portable (with `--no-self-contained`, runs on any RID with the runtime) | Slower first-run if runtime not cached |

Use for: server environments where you control the installed runtime (Docker base images with .NET, dev machines).

---

## Self-contained deployment (SCD)

```bash
dotnet publish -c Release -r win-x64 --self-contained
```

Bundles the **.NET runtime + your app** into the output. The target needs nothing pre-installed.

```xml
<SelfContained>true</SelfContained>
<RuntimeIdentifier>win-x64</RuntimeIdentifier>
```

| Pros | Cons |
|---|---|
| No runtime install needed | Large output (~70 MB+ before trimming) |
| Version-locked (your exact runtime) | Per-RID build; not portable |
| Predictable | Bigger; each app carries its own runtime |

Use for: distributing to machines you don't control, kiosks, locked-down environments.

---

## Runtime Identifiers (RID)

A RID targets a specific OS + architecture:

```
win-x64    win-arm64    win-x86
linux-x64  linux-arm64  linux-musl-x64 (Alpine)
osx-x64    osx-arm64    (Apple Silicon)
```

Self-contained, single-file, R2R, and AOT all require a RID (`-r`). Framework-dependent can be portable (no RID).

```xml
<RuntimeIdentifiers>win-x64;linux-x64;osx-arm64</RuntimeIdentifiers>   <!-- multi -->
```

---

## Single-file

```bash
dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true --self-contained
```

```xml
<PublishSingleFile>true</PublishSingleFile>
```

Packs everything into **one executable**. At runtime, it extracts (or memory-maps) the bundled assemblies. Simplifies distribution (one file to copy).

```xml
<IncludeNativeLibrariesForSelfExtract>true</IncludeNativeLibrariesForSelfExtract>
<EnableCompressionInSingleFile>true</EnableCompressionInSingleFile>   <!-- compress, smaller, slightly slower start -->
```

Single-file is packaging, not AOT — the runtime/JIT is still present inside the bundle.

---

## ReadyToRun (R2R)

```bash
dotnet publish -c Release -r win-x64 -p:PublishReadyToRun=true
```

```xml
<PublishReadyToRun>true</PublishReadyToRun>
```

R2R **pre-compiles IL to native code** at publish time (AOT compilation), but keeps the IL and JIT too. At startup, methods run from precompiled native code (no JIT warmup); the JIT can still re-optimize hot paths later (and combine with Dynamic PGO).

| Pros | Cons |
|---|---|
| Faster startup (no cold JIT) | Larger output (native + IL) |
| Keeps JIT for peak optimization | Per-RID build |
| Good middle ground | Not as small/fast-start as AOT |

Use for: long-running server apps where you want faster startup but also peak steady-state throughput (JIT + PGO).

---

## Native AOT

```bash
dotnet publish -c Release -r linux-x64   # with <PublishAot>true</PublishAot>
```

Full native compilation — no JIT, no IL, smallest fastest-start self-contained binary. See [04-NativeAOT.md](04-NativeAOT.md) for the full treatment and limitations.

| Pros | Cons |
|---|---|
| Fastest startup (~ms) | No reflection-heavy code / runtime codegen |
| Smallest self-contained | Mandatory trimming |
| Lowest memory | Per-RID; slower publish |
| No runtime install | May lose steady-state throughput vs JIT+PGO |

---

## Comparison table

| Mode | Runtime needed | Size (typical web API) | Startup | Steady-state | Reflection |
|---|---|---|---|---|---|
| Framework-dependent | Yes | ~hundreds KB | medium | full (JIT+PGO) | full |
| Self-contained | No | ~70 MB | medium | full | full |
| SC + trimmed | No | ~25 MB | medium | full | limited |
| Single-file | No (SC) | ~70 MB (1 file) | medium | full | full |
| ReadyToRun | depends | larger | fast | full (JIT+PGO) | full |
| **Native AOT** | No | **~10-15 MB** | **fastest** | good (no PGO) | **limited** |

---

## Choosing per scenario

| Scenario | Recommended mode |
|---|---|
| Server you control (Docker w/ .NET base image) | Framework-dependent |
| Distribute to user machines | Self-contained + single-file + trimmed |
| CLI tool | Native AOT (fast start, single binary) |
| Serverless / Lambda (cold-start sensitive) | Native AOT |
| Long-running high-throughput server | Framework-dependent + ReadyToRun (JIT+PGO at steady state) |
| Container microservice (fast scale-up) | Native AOT or trimmed SC |
| Plugin host / reflection-heavy app | Framework-dependent (avoid trimming/AOT) |

---

## Publish profiles (`.pubxml`)

For repeatable publishing, define a profile under `Properties/PublishProfiles/`:

```xml
<!-- Properties/PublishProfiles/Production.pubxml -->
<Project>
  <PropertyGroup>
    <PublishProtocol>FileSystem</PublishProtocol>
    <Configuration>Release</Configuration>
    <RuntimeIdentifier>linux-x64</RuntimeIdentifier>
    <SelfContained>true</SelfContained>
    <PublishSingleFile>true</PublishSingleFile>
    <PublishTrimmed>true</PublishTrimmed>
    <PublishReadyToRun>true</PublishReadyToRun>
  </PropertyGroup>
</Project>
```

```bash
dotnet publish -p:PublishProfile=Production
```

Profiles keep deployment settings out of the main `.csproj` and let you maintain multiple targets (dev, staging, prod).

---

## Common bugs

### Forgetting `-r` for self-contained

Self-contained/single-file/R2R/AOT all need a RID. Without it you get a framework-dependent build or an error.

### Trimming breaks reflection at runtime

Covered in [05-Trimming.md](05-Trimming.md). Test the trimmed output, not just the debug build.

### Native libraries not bundled in single-file

By default some native libs extract separately. Set `IncludeNativeLibrariesForSelfExtract` if you need a truly single file.

### Wrong RID for the target

Publishing `win-x64` and deploying to ARM (or musl/Alpine Linux) fails. Match the RID to the actual target (`linux-musl-x64` for Alpine).

### Assuming AOT is always faster

AOT wins startup but may lose steady-state throughput to JIT+PGO. Benchmark for your workload.

---

## Performance notes

- **Startup ranking** (fastest→slowest): AOT < R2R < single-file/SC ≈ FDD.
- **Steady-state throughput**: JIT + Dynamic PGO (FDD/SC/R2R) ≥ AOT for long-running servers.
- **Size ranking** (smallest→largest self-contained): AOT < trimmed SC < SC.
- Compression in single-file reduces size but adds a small extraction cost on start.

---

## Summary

- Choose **framework-dependent** when you control the runtime; **self-contained** when you don't.
- Layer **single-file** (one artifact), **trimmed** (smaller), **ReadyToRun** (faster startup, keeps JIT), or **Native AOT** (fastest start, smallest, but limited reflection).
- Self-contained/single-file/R2R/AOT require a RID (`-r`).
- AOT for CLI/serverless/containers (startup-sensitive); R2R + FDD for long-running throughput servers; FDD for reflection-heavy apps.
- Use `.pubxml` publish profiles for repeatable, multi-target deployments.
- Always test trimmed/AOT output — debug builds hide trimming failures.

→ Next: [Questions.md](Questions.md)
