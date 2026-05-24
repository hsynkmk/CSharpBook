# The .csproj File

## What it is

The `.csproj` is an MSBuild XML file describing how to build a project: target framework, dependencies, compile settings, and output. Modern .NET uses the **SDK-style** project format — concise, with sensible defaults and implicit file inclusion.

```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Serilog" Version="4.0.0" />
  </ItemGroup>

</Project>
```

That's a complete project. Compare to the old-style csproj which listed every `.cs` file and GUID — SDK-style includes files implicitly.

---

## SDK-style vs legacy

| | SDK-style (modern) | Legacy (.NET Framework) |
|---|---|---|
| First line | `<Project Sdk="...">` | `<Project ...><Import .../>` |
| File inclusion | Implicit (all `*.cs`) | Explicit `<Compile Include>` |
| Size | ~10 lines | ~100s of lines |
| Package refs | `<PackageReference>` | `packages.config` |

The `Sdk` attribute selects the build logic:
- `Microsoft.NET.Sdk` — console/library.
- `Microsoft.NET.Sdk.Web` — ASP.NET Core.
- `Microsoft.NET.Sdk.Worker` — worker services.
- `Microsoft.NET.Sdk.Razor` — Razor class libraries.

---

## `<PropertyGroup>` — build settings

Common properties:

```xml
<PropertyGroup>
  <!-- Target framework(s) -->
  <TargetFramework>net10.0</TargetFramework>

  <!-- Output -->
  <OutputType>Exe</OutputType>            <!-- or Library (default) -->
  <AssemblyName>MyApp</AssemblyName>
  <RootNamespace>MyCompany.MyApp</RootNamespace>

  <!-- Language -->
  <LangVersion>14</LangVersion>           <!-- usually implicit from TFM -->
  <Nullable>enable</Nullable>             <!-- NRT on -->
  <ImplicitUsings>enable</ImplicitUsings> <!-- global usings for common namespaces -->

  <!-- Quality gates -->
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <WarningsAsErrors>nullable</WarningsAsErrors>
  <AnalysisLevel>latest-recommended</AnalysisLevel>
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>

  <!-- Debugging / symbols -->
  <DebugType>portable</DebugType>
  <DebugSymbols>true</DebugSymbols>

  <!-- AOT / trimming / publish -->
  <PublishAot>true</PublishAot>
  <IsAotCompatible>true</IsAotCompatible>
  <InvariantGlobalization>true</InvariantGlobalization>

  <!-- Determinism / reproducible builds -->
  <Deterministic>true</Deterministic>
  <ContinuousIntegrationBuild>true</ContinuousIntegrationBuild>
</PropertyGroup>
```

### `Nullable` and `ImplicitUsings`

- `<Nullable>enable</Nullable>` turns on Nullable Reference Types — flow analysis for `null` (see [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md)). Strongly recommended for new code.
- `<ImplicitUsings>enable</ImplicitUsings>` auto-imports common namespaces (`System`, `System.Collections.Generic`, `System.Linq`, etc.) so you don't write them. See [Chapter 10 §07](../10-AdvancedLanguage/07-GlobalAndImplicitUsings.md).

---

## `<ItemGroup>` — references and files

### Package references

```xml
<ItemGroup>
  <PackageReference Include="Serilog" Version="4.0.0" />
  <PackageReference Include="Microsoft.Extensions.Hosting" Version="10.0.0" />

  <!-- Analyzer/source-gen package — not a runtime dependency -->
  <PackageReference Include="MyAnalyzer" Version="1.0.0" PrivateAssets="all" />
</ItemGroup>
```

`PrivateAssets="all"` means the dependency isn't exposed transitively to consumers (used for analyzers, build-only tools, source generators).

### Project references

```xml
<ItemGroup>
  <ProjectReference Include="..\Core\Core.csproj" />
</ItemGroup>
```

### Explicit file control (rarely needed)

```xml
<ItemGroup>
  <Compile Remove="Generated\**\*.cs" />        <!-- exclude -->
  <None Include="appsettings.json" CopyToOutputDirectory="PreserveNewest" />
  <EmbeddedResource Include="Resources\*.json" />
  <Content Include="wwwroot\**" CopyToOutputDirectory="Always" />
</ItemGroup>
```

`CopyToOutputDirectory` (`PreserveNewest`/`Always`/`Never`) controls whether non-code files are copied to `bin`.

---

## Multi-targeting

Build one project for multiple frameworks:

```xml
<PropertyGroup>
  <TargetFrameworks>net10.0;net8.0;netstandard2.0</TargetFrameworks>
</PropertyGroup>
```

Note the plural **`TargetFrameworks`**. The build produces an output per TFM. Use conditional compilation for framework-specific code:

```csharp
#if NET10_0_OR_GREATER
    // use newest APIs
#else
    // fallback for older targets
#endif
```

Conditional package references per TFM:

```xml
<ItemGroup Condition="'$(TargetFramework)' == 'netstandard2.0'">
  <PackageReference Include="System.Text.Json" Version="10.0.0" />
</ItemGroup>
```

Libraries multi-target to reach more consumers; apps usually target a single TFM.

---

## Target framework monikers (TFMs)

| TFM | Meaning |
|---|---|
| `net10.0` | .NET 10 (current) |
| `net8.0` | .NET 8 (LTS) |
| `netstandard2.0` | API surface implemented by both .NET Framework 4.6.1+ and .NET Core — max library reach |
| `net10.0-windows` | .NET 10 with Windows-specific APIs (WinForms/WPF) |
| `net10.0-android`, `-ios` | Platform-specific (MAUI) |

For libraries aiming at the widest audience, `netstandard2.0` is still relevant. For apps and modern libraries, target `net10.0` (and maybe `net8.0` for LTS reach).

---

## `Directory.Build.props` and `Directory.Build.targets`

Repo-wide MSBuild settings without repeating them in every `.csproj`. Place at the repo root; MSBuild auto-imports them for all projects below.

```xml
<!-- Directory.Build.props — imported BEFORE each project -->
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>14</LangVersion>
    <Authors>My Company</Authors>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Build.targets — imported AFTER each project (override/add) -->
<Project>
  <ItemGroup>
    <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="all" />
  </ItemGroup>
</Project>
```

`.props` loads first (defaults), `.targets` loads last (overrides/additions). Centralizing settings here keeps individual `.csproj` files minimal and consistent. See [04-MSBuild.md](04-MSBuild.md).

---

## Packaging metadata (when building a NuGet)

```xml
<PropertyGroup>
  <PackageId>MyCompany.MyLib</PackageId>
  <Version>1.2.3</Version>
  <Authors>Jane Dev</Authors>
  <Description>A useful library.</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <RepositoryUrl>https://github.com/me/mylib</RepositoryUrl>
  <PackageTags>utility;helpers</PackageTags>
  <GeneratePackageOnBuild>true</GeneratePackageOnBuild>
</PropertyGroup>
```

See [03-NuGet.md](03-NuGet.md).

---

## Common bugs / gotchas

### Forgetting the plural for multi-targeting

`<TargetFramework>` (singular) takes one TFM; `<TargetFrameworks>` (plural) takes a list. Using the singular with a `;` list silently builds nothing useful.

### Files unexpectedly included/excluded

SDK-style includes all `*.cs` under the project dir by default. Stray `.cs` files in subfolders get compiled. Use `<Compile Remove>` or `<DefaultItemExcludes>` to exclude.

### Missing CopyToOutputDirectory

Config/data files won't be in `bin` unless marked `CopyToOutputDirectory`. Runtime "file not found" often traces to this.

### Analyzer packages leaking as dependencies

Forgetting `PrivateAssets="all"` on analyzer/source-gen packages exposes them transitively. Always set it for build-only packages.

### LangVersion vs TFM mismatch

Setting `<LangVersion>14</LangVersion>` on an old TFM enables syntax but may reference runtime APIs that don't exist on that target. Language features that need runtime support require the matching framework.

---

## Summary

- SDK-style `.csproj` is concise: implicit file inclusion, `<PackageReference>`, sensible defaults.
- `<PropertyGroup>` for build settings (TFM, Nullable, ImplicitUsings, quality gates, AOT).
- `<ItemGroup>` for packages, project references, and explicit file handling.
- Multi-target with `<TargetFrameworks>` (plural) + `#if` for per-framework code.
- Centralize repo-wide settings in `Directory.Build.props`/`.targets`.
- Use `PrivateAssets="all"` for analyzer/build-only packages; `CopyToOutputDirectory` for data files.

→ Next: [03-NuGet.md](03-NuGet.md)
