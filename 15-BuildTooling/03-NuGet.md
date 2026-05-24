# NuGet

## What it is

NuGet is .NET's package manager. Packages (`.nupkg` files — zipped assemblies + metadata) are published to feeds (nuget.org by default) and consumed via `<PackageReference>`. It handles dependency resolution, versioning, and restore.

```xml
<PackageReference Include="Serilog" Version="4.0.0" />
```

```bash
dotnet add package Serilog --version 4.0.0
dotnet restore
```

---

## Consuming packages

```bash
dotnet add package Newtonsoft.Json              # latest stable
dotnet add package Serilog --version 4.0.0      # pinned
dotnet add package MyPreview --prerelease       # include prerelease
dotnet remove package Serilog
dotnet list package                             # what's installed
dotnet list package --outdated                  # updates available
dotnet list package --vulnerable                # known CVEs
dotnet list package --include-transitive        # full tree
```

Restored packages land in the global cache `~/.nuget/packages` (shared across projects), not in your project folder.

---

## Version ranges

```xml
<PackageReference Include="Lib" Version="4.0.0" />        <!-- 4.0.0 or higher that resolver picks; floats up via transitive -->
<PackageReference Include="Lib" Version="[4.0.0]" />      <!-- exactly 4.0.0 -->
<PackageReference Include="Lib" Version="[4.0,5.0)" />    <!-- >=4.0, <5.0 -->
<PackageReference Include="Lib" Version="4.*" />          <!-- latest 4.x -->
<PackageReference Include="Lib" Version="4.0.0-*" />      <!-- latest 4.0.0 prerelease -->
```

A bare version like `4.0.0` is a **minimum** ("4.0.0 or the nearest available ≥"). For reproducible builds, prefer exact versions or a lock file.

### NuGet version resolution

When multiple packages depend on different versions of the same transitive dependency, NuGet picks the **lowest version that satisfies all constraints** (nearest-wins + lowest-applicable). This can surprise you — use `dotnet list package --include-transitive` to inspect.

---

## Lock files — reproducible restore

```xml
<PropertyGroup>
  <RestorePackagesWithLockFile>true</RestorePackagesWithLockFile>
</PropertyGroup>
```

This generates `packages.lock.json` pinning the **exact resolved versions** (including transitive). Commit it. In CI, enforce it:

```bash
dotnet restore --locked-mode    # fail if lock file is out of date
```

Lock files make restores reproducible — the same versions everywhere, no surprise transitive bumps.

---

## Central Package Management (CPM)

Manage versions for the **whole solution** in one file instead of per-project. Add `Directory.Packages.props` at the repo root:

```xml
<!-- Directory.Packages.props -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.0.0" />
    <PackageVersion Include="xunit" Version="2.9.0" />
  </ItemGroup>
</Project>
```

Then projects reference **without** a version:

```xml
<!-- Project.csproj -->
<ItemGroup>
  <PackageReference Include="Serilog" />   <!-- version comes from CPM -->
</ItemGroup>
```

CPM keeps versions consistent across many projects — change once, applies everywhere. Essential for large solutions. Override per-project if needed with `VersionOverride`.

---

## Creating your own package

### csproj-as-package (modern)

```xml
<PropertyGroup>
  <PackageId>MyCompany.Utils</PackageId>
  <Version>1.2.3</Version>
  <Authors>Jane Dev</Authors>
  <Company>MyCompany</Company>
  <Description>Utility helpers for X.</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <PackageProjectUrl>https://github.com/me/utils</PackageProjectUrl>
  <RepositoryUrl>https://github.com/me/utils</RepositoryUrl>
  <RepositoryType>git</RepositoryType>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <PackageTags>utility;helpers</PackageTags>
  <PackageIcon>icon.png</PackageIcon>
</PropertyGroup>

<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="\" />
  <None Include="icon.png" Pack="true" PackagePath="\" />
</ItemGroup>
```

```bash
dotnet pack -c Release           # → bin/Release/MyCompany.Utils.1.2.3.nupkg
```

Modern packages embed metadata in the `.csproj`. The old `.nuspec` file is only needed for advanced/custom layouts.

### Symbol packages and SourceLink

```xml
<PropertyGroup>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
</PropertyGroup>

<ItemGroup>
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="all" />
</ItemGroup>
```

SourceLink lets consumers step into your package's source while debugging — publish a `.snupkg` symbol package alongside.

---

## Publishing

```bash
# Push to nuget.org (need an API key)
dotnet nuget push bin/Release/MyCompany.Utils.1.2.3.nupkg \
  --api-key $NUGET_API_KEY \
  --source https://api.nuget.org/v3/index.json

# Push symbols
dotnet nuget push bin/Release/MyCompany.Utils.1.2.3.snupkg --api-key ... --source ...
```

### Versioning (SemVer)

Follow Semantic Versioning:
- **Major** (1.x → 2.0) — breaking changes.
- **Minor** (1.1 → 1.2) — new features, backward-compatible.
- **Patch** (1.1.1 → 1.1.2) — bug fixes.
- **Prerelease** (`1.2.0-beta.1`) — pre-stable.

Breaking a public API in a minor/patch release breaks consumers — see [Chapter 17 §04](../17-BestPractices/04-ApiDesign.md).

---

## Private and multiple feeds

`nuget.config` at repo root configures feeds:

```xml
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="company" value="https://pkgs.company.com/v3/index.json" />
  </packageSources>
  <packageSourceMapping>
    <packageSource key="company">
      <package pattern="MyCompany.*" />     <!-- company packages only from company feed -->
    </packageSource>
    <packageSource key="nuget.org">
      <package pattern="*" />
    </packageSource>
  </packageSourceMapping>
</configuration>
```

**Package Source Mapping** (security) ensures `MyCompany.*` packages only come from your trusted feed — defends against dependency-confusion attacks (a malicious public package shadowing your internal name).

---

## Security hygiene

```bash
dotnet list package --vulnerable --include-transitive   # CVEs in deps (incl. transitive)
dotnet list package --deprecated                        # deprecated packages
```

```xml
<PropertyGroup>
  <NuGetAudit>true</NuGetAudit>                  <!-- warn on vulnerable packages at restore -->
  <NuGetAuditLevel>moderate</NuGetAuditLevel>
</PropertyGroup>
```

NuGet Audit (default on in modern SDKs) flags vulnerable dependencies during restore. Treat audit warnings seriously in CI.

---

## Common bugs / gotchas

### Diamond dependency / version conflict

Two packages need different versions of a shared transitive dep. NuGet picks one (lowest-applicable). If incompatible, you get runtime `MissingMethodException`. Add an explicit `<PackageReference>` to force a compatible version.

### Floating versions break reproducibility

`Version="4.*"` can pull a new version unexpectedly, breaking a build that worked yesterday. Pin versions and use a lock file for reproducibility.

### Forgetting `PrivateAssets` on analyzers

Exposes build-only packages transitively. Set `PrivateAssets="all"`.

### Dependency confusion

Internal package name also exists publicly → restore pulls the public (possibly malicious) one. Use Package Source Mapping.

### Stale package cache

Rarely, a corrupt cache causes weird restore errors. `dotnet nuget locals all --clear` resets the global cache.

---

## Summary

- NuGet manages dependencies via `<PackageReference>`; packages restore to a shared global cache.
- Bare versions are minimums; use exact versions + `packages.lock.json` (`--locked-mode`) for reproducibility.
- **Central Package Management** (`Directory.Packages.props`) centralizes versions across a solution.
- Build packages with `dotnet pack` (metadata in `.csproj`); publish with `dotnet nuget push`. Follow SemVer.
- Use **Package Source Mapping** to prevent dependency-confusion attacks.
- Run `dotnet list package --vulnerable` and enable NuGet Audit for security.

→ Next: [04-MSBuild.md](04-MSBuild.md)
