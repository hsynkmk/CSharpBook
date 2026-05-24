# MSBuild

## What it is

MSBuild is the build engine behind .NET. `dotnet build` is a thin wrapper over it. An MSBuild project (`.csproj`) is an XML document describing **properties** (variables), **items** (file lists), **targets** (named build steps), and **tasks** (units of work). MSBuild evaluates this graph to produce output.

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
  </PropertyGroup>
</Project>
```

The `Sdk="Microsoft.NET.Sdk"` attribute imports a large set of predefined targets/properties — that's why a 5-line csproj can build a full app.

---

## The four building blocks

### Properties — scalar variables

```xml
<PropertyGroup>
  <MyVar>hello</MyVar>
  <OutputName>$(MyVar)-app</OutputName>   <!-- referenced with $(...) -->
</PropertyGroup>
```

Referenced as `$(PropertyName)`. Properties are strings. Later definitions override earlier ones.

```xml
<MyVar Condition="'$(Configuration)' == 'Release'">release-value</MyVar>
```

### Items — lists of things (usually files)

```xml
<ItemGroup>
  <Compile Include="**/*.cs" />
  <MyFiles Include="data/*.json" />
</ItemGroup>
```

Referenced as `@(ItemName)`. Each item can have **metadata**:

```xml
<None Include="config.json">
  <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
</None>
<!-- access metadata: %(None.CopyToOutputDirectory) -->
```

### Targets — named, ordered build steps

```xml
<Target Name="SayHello" BeforeTargets="Build">
  <Message Text="Building $(MSBuildProjectName)..." Importance="high" />
</Target>
```

`BeforeTargets`/`AfterTargets`/`DependsOnTargets` control ordering. The SDK defines targets like `Restore`, `Build`, `Compile`, `Publish`, `Pack`.

### Tasks — the actual work

Tasks are .NET classes that do something (compile, copy, exec). Built-in tasks include `Message`, `Copy`, `Delete`, `Exec`, `MakeDir`, `Csc`.

```xml
<Target Name="CopyExtras" AfterTargets="Build">
  <Copy SourceFiles="@(MyFiles)" DestinationFolder="$(OutDir)/data" />
  <Exec Command="echo Build done" />
</Target>
```

---

## Evaluation order

MSBuild runs in two phases:
1. **Evaluation** — read all properties/items (top to bottom, imports included). Properties get final values; items are gathered.
2. **Execution** — run the requested target and its dependencies.

This is why a property defined later can override one defined earlier, and why `Condition` is evaluated during the appropriate phase. Order matters.

---

## Common built-in properties

```
$(MSBuildProjectName)        project name
$(MSBuildProjectDirectory)   project folder
$(MSBuildThisFileDirectory)  folder of the current .props/.targets
$(Configuration)             Debug / Release
$(TargetFramework)           net10.0
$(OutputPath) / $(OutDir)    bin/<config>/<tfm>/
$(IntermediateOutputPath)    obj/...
$(MSBuildProjectFullPath)    full path to .csproj
```

---

## Custom build steps

### Run a command before/after build

```xml
<Target Name="GenerateVersion" BeforeTargets="CoreCompile">
  <Exec Command="git rev-parse --short HEAD &gt; $(IntermediateOutputPath)version.txt" />
</Target>

<Target Name="Notify" AfterTargets="Build">
  <Message Text="Built $(AssemblyName) v$(Version)" Importance="high" />
</Target>
```

### Conditional logic

```xml
<Target Name="OnlyInRelease" AfterTargets="Build" Condition="'$(Configuration)' == 'Release'">
  <Message Text="Release build complete" Importance="high" />
</Target>
```

### Inputs/Outputs for incremental builds

```xml
<Target Name="GenerateCode"
        Inputs="@(SchemaFiles)"
        Outputs="@(SchemaFiles -> '$(IntermediateOutputPath)%(Filename).g.cs')">
  <Exec Command="codegen %(SchemaFiles.FullPath)" />
</Target>
```

If outputs are newer than inputs, MSBuild **skips** the target — incremental build. Always declare `Inputs`/`Outputs` for custom codegen so it doesn't run every build.

---

## `Directory.Build.props` / `.targets` — repo-wide build logic

MSBuild walks up from each project looking for these files and imports them automatically:

```xml
<!-- Directory.Build.props — imported at the START of every project -->
<Project>
  <PropertyGroup>
    <LangVersion>14</LangVersion>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

```xml
<!-- Directory.Build.targets — imported at the END (override, add targets) -->
<Project>
  <Target Name="StampBuild" AfterTargets="Build">
    <Message Text="Stamped $(MSBuildProjectName)" Importance="high" />
  </Target>
</Project>
```

- `.props` → first (defaults you might override in the csproj).
- `.targets` → last (final overrides and shared targets).

This is how you apply consistent settings to every project in a repo without copy-pasting. See [Chapter 15 §02](02-Csproj.md).

---

## Import and shared logic

```xml
<Import Project="../build/common.props" />
```

Factor shared MSBuild logic into `.props`/`.targets` files and import them. The SDK itself is a giant set of imported targets.

---

## Inspecting and debugging the build

```bash
# Detailed log
dotnet build -v diag > build.log

# Binary log (best for analysis) — view with the MSBuild Structured Log Viewer
dotnet build -bl              # produces msbuild.binlog
dotnet build /bl:mybuild.binlog

# Preprocess: see the fully expanded project (all imports inlined)
dotnet msbuild /pp:full.xml MyProject.csproj

# Print a property's value
dotnet msbuild MyProject.csproj -getProperty:TargetFramework
```

The **binary log** (`-bl`) + Structured Log Viewer is the single best tool for understanding why a build did (or didn't) do something — every target, property, and task is recorded.

---

## Passing properties from the CLI

```bash
dotnet build -p:Configuration=Release -p:Version=2.1.0
dotnet build /p:DefineConstants=FEATURE_X
dotnet publish -p:PublishAot=true -p:InvariantGlobalization=true
```

`-p:Name=Value` (or `/p:`) overrides any property. Useful in CI to inject versions, feature flags, etc.

---

## Writing a custom task (advanced)

For logic beyond `Exec`, write a task class:

```csharp
using Microsoft.Build.Framework;
using Microsoft.Build.Utilities;

public class GenerateBuildInfo : Microsoft.Build.Utilities.Task {
    [Required] public string OutputPath { get; set; } = "";
    public override bool Execute() {
        File.WriteAllText(OutputPath, $"Built at {DateTime.UtcNow:O}");
        Log.LogMessage(MessageImportance.High, $"Wrote {OutputPath}");
        return !Log.HasLoggedErrors;
    }
}
```

```xml
<UsingTask TaskName="GenerateBuildInfo" AssemblyFile="$(MyTasksDll)" />
<Target Name="BuildInfo" BeforeTargets="Build">
  <GenerateBuildInfo OutputPath="$(IntermediateOutputPath)buildinfo.txt" />
</Target>
```

Most needs are met by `Exec` + scripts or source generators; custom tasks are for reusable, build-integrated logic.

---

## Common bugs / gotchas

### Custom target runs every build

Forgetting `Inputs`/`Outputs` means no incremental skip — the target runs every time, slowing builds. Declare them.

### Property used before it's set

Because evaluation is top-to-bottom, a property referenced in an import before it's defined sees the empty/old value. Mind ordering; `.props` (early) vs `.targets` (late).

### `Condition` quoting

```xml
<PropertyGroup Condition="'$(Configuration)' == 'Release'">   <!-- ✓ quoted -->
<PropertyGroup Condition="$(Configuration) == Release">       <!-- ⚠ fails if value is empty -->
```

Always quote both sides so an empty value doesn't break parsing.

### Wrong target ordering

`BeforeTargets="Build"` may run earlier than you expect (before `Compile`). For "after the DLL exists," use `AfterTargets="Build"` or `AfterTargets="Compile"` appropriately.

---

## Summary

- MSBuild is the engine; `.csproj` is an MSBuild file of **properties**, **items**, **targets**, **tasks**.
- Reference properties with `$(...)`, items with `@(...)`, metadata with `%(...)`.
- Custom targets via `BeforeTargets`/`AfterTargets`/`DependsOnTargets`; declare `Inputs`/`Outputs` for incremental builds.
- `Directory.Build.props`/`.targets` apply repo-wide settings automatically (props early, targets late).
- Debug builds with the **binary log** (`-bl`) + Structured Log Viewer; preprocess with `/pp`.
- Override properties from CLI with `-p:Name=Value`. Write custom tasks only when `Exec`/scripts/source-gen won't do.

→ Next: [05-RoslynAnalyzers.md](05-RoslynAnalyzers.md)
