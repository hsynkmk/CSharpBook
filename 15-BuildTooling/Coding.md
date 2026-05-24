# Chapter 15 — Build & Tooling — Coding Problems

---

## Problem 1: Bootstrap a solution from the CLI

Create a solution with an API project, a core library, and a test project, all wired up.

<details><summary>Solution</summary>

```bash
dotnet new sln -n Shop
dotnet new web      -o src/Shop.Api
dotnet new classlib -o src/Shop.Core
dotnet new xunit    -o tests/Shop.Tests

dotnet sln add src/Shop.Api src/Shop.Core tests/Shop.Tests
dotnet add src/Shop.Api  reference src/Shop.Core
dotnet add tests/Shop.Tests reference src/Shop.Core

dotnet build
dotnet test
```

No IDE needed — the CLI does everything. This is also the basis of a CI script.

</details>

---

## Problem 2: Multi-target a library

Configure a library to build for `net10.0` and `netstandard2.0`, using a newer API only where available.

<details><summary>Solution</summary>

```xml
<PropertyGroup>
  <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
  <Nullable>enable</Nullable>
  <LangVersion>14</LangVersion>
</PropertyGroup>
```

```csharp
public static string Describe(ReadOnlySpan<char> input) {
#if NET10_0_OR_GREATER
    return string.Concat("len=", input.Length);   // newest APIs
#else
    return "len=" + input.Length.ToString();        // netstandard2.0 fallback
#endif
}
```

Note plural `TargetFrameworks`. `#if NET10_0_OR_GREATER` guards version-specific code.

</details>

---

## Problem 3: Centralize package versions

A solution has 8 projects all referencing xunit and Serilog. Centralize versions.

<details><summary>Solution</summary>

```xml
<!-- Directory.Packages.props (repo root) -->
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.0.0" />
    <PackageVersion Include="xunit" Version="2.9.0" />
    <PackageVersion Include="xunit.runner.visualstudio" Version="2.8.0" />
  </ItemGroup>
</Project>
```

```xml
<!-- each project -->
<ItemGroup>
  <PackageReference Include="Serilog" />   <!-- no version here -->
  <PackageReference Include="xunit" />
</ItemGroup>
```

Now a version bump happens in one place. Override per-project with `VersionOverride` if truly needed.

</details>

---

## Problem 4: Repo-wide build settings

Apply Nullable, ImplicitUsings, warnings-as-errors, and LangVersion 14 to every project without editing each csproj.

<details><summary>Solution</summary>

```xml
<!-- Directory.Build.props (repo root) -->
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <LangVersion>14</LangVersion>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>
    <AnalysisLevel>latest-recommended</AnalysisLevel>
    <Deterministic>true</Deterministic>
  </PropertyGroup>
</Project>
```

MSBuild imports this automatically for every project below. Individual csproj files stay minimal.

</details>

---

## Problem 5: Custom MSBuild target with incrementality

Add a build step that runs a code generator over `*.schema.json` files, but only when they change.

<details><summary>Solution</summary>

```xml
<ItemGroup>
  <SchemaFiles Include="schemas/*.schema.json" />
</ItemGroup>

<Target Name="GenerateModels"
        BeforeTargets="CoreCompile"
        Inputs="@(SchemaFiles)"
        Outputs="@(SchemaFiles -> '$(IntermediateOutputPath)%(Filename).g.cs')">
  <Exec Command="codegen %(SchemaFiles.FullPath) -o $(IntermediateOutputPath)%(SchemaFiles.Filename).g.cs" />
  <ItemGroup>
    <Compile Include="$(IntermediateOutputPath)%(SchemaFiles.Filename).g.cs" />
  </ItemGroup>
</Target>
```

`Inputs`/`Outputs` make it incremental — MSBuild skips the target when generated files are newer than schemas. Adding the generated files to `@(Compile)` includes them in the build.

</details>

---

## Problem 6: Make a NuGet package

Turn a class library into a publishable NuGet package with metadata and a README.

<details><summary>Solution</summary>

```xml
<PropertyGroup>
  <PackageId>MyCompany.Json.Extensions</PackageId>
  <Version>1.0.0</Version>
  <Authors>Jane Dev</Authors>
  <Description>Helpful extensions for System.Text.Json.</Description>
  <PackageLicenseExpression>MIT</PackageLicenseExpression>
  <PackageReadmeFile>README.md</PackageReadmeFile>
  <RepositoryUrl>https://github.com/jane/json-ext</RepositoryUrl>
  <PackageTags>json;extensions;stj</PackageTags>
  <IncludeSymbols>true</IncludeSymbols>
  <SymbolPackageFormat>snupkg</SymbolPackageFormat>
</PropertyGroup>

<ItemGroup>
  <None Include="README.md" Pack="true" PackagePath="\" />
  <PackageReference Include="Microsoft.SourceLink.GitHub" Version="8.0.0" PrivateAssets="all" />
</ItemGroup>
```

```bash
dotnet pack -c Release
dotnet nuget push bin/Release/MyCompany.Json.Extensions.1.0.0.nupkg \
  --api-key $KEY --source https://api.nuget.org/v3/index.json
```

SourceLink + symbols let consumers debug into your source.

</details>

---

## Problem 7: Configure naming conventions in .editorconfig

Enforce: private fields use `_camelCase`, constants use `PascalCase`, both as warnings.

<details><summary>Solution</summary>

```ini
[*.cs]
# Private fields → _camelCase
dotnet_naming_symbols.private_fields.applicable_kinds = field
dotnet_naming_symbols.private_fields.applicable_accessibilities = private
dotnet_naming_style.underscore_camel.capitalization = camel_case
dotnet_naming_style.underscore_camel.required_prefix = _
dotnet_naming_rule.private_fields_rule.symbols = private_fields
dotnet_naming_rule.private_fields_rule.style = underscore_camel
dotnet_naming_rule.private_fields_rule.severity = warning

# Constants → PascalCase
dotnet_naming_symbols.consts.applicable_kinds = field
dotnet_naming_symbols.consts.required_modifiers = const
dotnet_naming_style.pascal.capitalization = pascal_case
dotnet_naming_rule.consts_rule.symbols = consts
dotnet_naming_rule.consts_rule.style = pascal
dotnet_naming_rule.consts_rule.severity = warning
```

Each rule needs all three parts (symbol + style + rule). A typo in any key makes the rule silently inert — test by introducing a violation.

</details>

---

## Problem 8: Write a custom analyzer

Write an analyzer that flags `DateTime.Now` (suggesting `DateTimeOffset.UtcNow` for testability/correctness).

<details><summary>Solution</summary>

```csharp
using System.Collections.Immutable;
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;
using Microsoft.CodeAnalysis.Operations;

[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class NoDateTimeNowAnalyzer : DiagnosticAnalyzer {
    private static readonly DiagnosticDescriptor Rule = new(
        "MY0002", "Avoid DateTime.Now",
        "Use DateTimeOffset.UtcNow (or an injected clock) instead of DateTime.Now",
        "Reliability", DiagnosticSeverity.Warning, isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context) {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterOperationAction(Analyze, OperationKind.PropertyReference);
    }

    private static void Analyze(OperationAnalysisContext ctx) {
        var op = (IPropertyReferenceOperation)ctx.Operation;
        if (op.Property.Name == "Now" &&
            op.Property.ContainingType.ToDisplayString() == "System.DateTime") {
            ctx.ReportDiagnostic(Diagnostic.Create(Rule, op.Syntax.GetLocation()));
        }
    }
}
```

Using `RegisterOperationAction` + the **semantic model** (checking `ContainingType` is `System.DateTime`) is more accurate than matching the text `"Now"`, which would false-positive on any `.Now` property. Analyzer project targets `netstandard2.0`.

</details>

---

## Problem 9: Optimize an allocation-heavy method (profile-driven)

Profiling shows this is allocating heavily. Identify why and fix it.

```csharp
public string Join(IEnumerable<int> numbers) {
    string result = "";
    foreach (var n in numbers)
        result += n + ",";        // hot spot per allocation profiler
    return result;
}
```

<details><summary>Solution</summary>

The profiler (VS Allocation tool / `[MemoryDiagnoser]`) shows quadratic string allocations: each `+=` creates a new string. Fix with `StringBuilder` (or `string.Join`):

```csharp
public string Join(IEnumerable<int> numbers) {
    var sb = new StringBuilder();
    foreach (var n in numbers) {
        sb.Append(n);
        sb.Append(',');
    }
    return sb.ToString();
}

// Or, simplest:
public string Join(IEnumerable<int> numbers) => string.Join(",", numbers);
```

Benchmark to confirm:

```csharp
[MemoryDiagnoser]
public class JoinBench {
    private readonly int[] _data = Enumerable.Range(0, 1000).ToArray();
    [Benchmark(Baseline = true)] public string Concat() { /* original */ }
    [Benchmark] public string Builder() { /* StringBuilder */ }
    [Benchmark] public string Linq() => string.Join(",", _data);
}
```

Expect `Concat` to allocate O(n²) bytes; `Builder`/`Linq` allocate O(n). Always confirm the win with a measurement, not intuition.

</details>

---

## Problem 10: A CI build script

Write a CI script that restores, builds, tests, checks formatting, scans for vulnerabilities, and publishes — efficiently (no redundant work).

<details><summary>Solution</summary>

```bash
#!/usr/bin/env bash
set -euo pipefail

# Restore once (locked for reproducibility)
dotnet restore --locked-mode

# Build without re-restoring
dotnet build -c Release --no-restore /p:TreatWarningsAsErrors=true

# Test on the existing build
dotnet test -c Release --no-build --logger "trx;LogFileName=results.trx" \
  --collect:"XPlat Code Coverage"

# Style/format gate
dotnet format --verify-no-changes

# Security: fail on vulnerable dependencies
dotnet list package --vulnerable --include-transitive 2>&1 | tee audit.log
grep -q "no vulnerable packages" audit.log || (echo "Vulnerable packages found"; exit 1)

# Publish on the existing build
dotnet publish src/Api -c Release --no-build -o ./out
```

Key efficiencies: `--locked-mode` (reproducible restore), `--no-restore`/`--no-build` to avoid repeating work across steps, warnings-as-errors and format-verify as quality gates, vulnerability scan as a security gate.

</details>

---

These problems cover the full tooling loop: scaffolding, multi-targeting, centralized/repro builds, custom MSBuild + analyzers, `.editorconfig` enforcement, profile-driven optimization, and a real CI pipeline.

→ Back to [Chapter 15 README](README.md). Next chapter: [Chapter 16 — Testing](../16-Testing/README.md).
