# Project Structure

## The .csproj file

Every .NET project (except file-based apps) has a **project file** with the extension `.csproj`. It tells the SDK what to build, what to reference, and how. Modern projects use the **SDK-style** format, which is dramatically simpler than the .NET Framework era.

A fresh `dotnet new console` produces:

`HelloApp.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">

  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>

</Project>
```

That's it. Let's understand every element.

---

## `<Project Sdk="..."`>

Tells MSBuild which **SDK** to use. The SDK provides default targets (what to build, where to put output), default item includes (which files are sources), and default properties.

| SDK | Used for |
|---|---|
| `Microsoft.NET.Sdk` | console apps, class libraries, most non-web projects |
| `Microsoft.NET.Sdk.Web` | ASP.NET Core apps and Web APIs |
| `Microsoft.NET.Sdk.BlazorWebAssembly` | Blazor WebAssembly apps |
| `Microsoft.NET.Sdk.Worker` | hosted services / background workers |
| `Microsoft.NET.Sdk.Razor` | Razor class libraries |
| `MSBuild.Sdk.Extras` | community SDK for multi-targeting NuGet packages |

You can usually leave this on the default.

---

## `<PropertyGroup>` — settings

Inside `<PropertyGroup>` you set **MSBuild properties** that configure the build. The common ones:

### `OutputType`
- `Exe` → produces an executable (`.exe` on Windows, an entry-point binary on Linux/macOS).
- `Library` → produces a `.dll` only.

Default for `Microsoft.NET.Sdk` is `Library`. The `console` template overrides to `Exe`.

### `TargetFramework`
Which framework version your code targets:
- `net10.0` → .NET 10
- `net8.0` → .NET 8 LTS
- `netstandard2.1` → .NET Standard 2.1 (for libraries that need to run on multiple runtimes including .NET Framework)
- `net48` → .NET Framework 4.8 (legacy)
- `net10.0-windows10.0.19041.0` → Windows-specific APIs available

For new projects: **target the latest LTS** (currently `net10.0`).

### `TargetFrameworks` (plural) — multi-targeting
For libraries that should compile against multiple frameworks:
```xml
<TargetFrameworks>net10.0;net8.0;netstandard2.0</TargetFrameworks>
```
The compiler runs once per framework, producing one DLL per target. NuGet packages can ship all of them.

### `ImplicitUsings`
- `enable` → the SDK adds common `using` directives for free (System, System.IO, System.Collections.Generic, System.Linq, System.Threading.Tasks, etc.).
- `disable` → you must type every using.

For console apps the implicit list also includes `System.Console`; for web apps it adds `Microsoft.AspNetCore.Builder` and similar.

You can fine-tune in code via `<Using>` items (see §"Adding usings via csproj" below).

### `Nullable`
Controls **Nullable Reference Types** (NRT) — see [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md).

- `enable` → reference types are non-nullable by default, must annotate with `?` for nullable.
- `disable` → pre-C# 8 behavior; no null-state analysis.
- `annotations` → annotations honored, no warnings emitted.
- `warnings` → warnings emitted, annotations ignored.

**Always `enable` for new code.**

### `LangVersion`
Forces a specific C# language version:
```xml
<LangVersion>14.0</LangVersion>
```

Default: the latest version that matches your target framework. .NET 10 = C# 14 by default.

### Other useful properties

```xml
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>      <!-- CI safety -->
<EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>  <!-- run analyzers in build, not just IDE -->
<NoWarn>CS1591</NoWarn>                                   <!-- silence specific warnings -->
<RootNamespace>MyCompany.MyProduct</RootNamespace>       <!-- override namespace prefix -->
<AssemblyName>my-tool</AssemblyName>                      <!-- override output DLL name -->
<Version>1.2.3</Version>                                  <!-- package version -->
<Authors>You</Authors>
<Description>What this thing does.</Description>
```

---

## `<ItemGroup>` — what to include

Items are "things in the project": source files, references, content.

By default, **all `.cs` files in the project folder and subfolders are compiled**. You don't need to list them.

### Adding NuGet packages
```xml
<ItemGroup>
  <PackageReference Include="Serilog" Version="4.1.0" />
  <PackageReference Include="Serilog.Sinks.Console" Version="6.0.0" />
</ItemGroup>
```

After editing, run `dotnet restore` (or just `dotnet build` — it restores automatically).

### Referencing another project in the same solution
```xml
<ItemGroup>
  <ProjectReference Include="..\MyLibrary\MyLibrary.csproj" />
</ItemGroup>
```

### Referencing a raw assembly (legacy)
```xml
<ItemGroup>
  <Reference Include="..\libs\Legacy.dll" />
</ItemGroup>
```

Avoid this — prefer NuGet packages.

### Adding implicit usings
```xml
<ItemGroup>
  <Using Include="Microsoft.Extensions.Logging" />
  <Using Include="MyApp.Common" />
</ItemGroup>
```

Now every file behaves as if `using Microsoft.Extensions.Logging;` and `using MyApp.Common;` were at the top.

### Adding global using statics
```xml
<ItemGroup>
  <Using Include="System.Math" Static="true" />
</ItemGroup>
```

Now you can write `Sqrt(4)` instead of `Math.Sqrt(4)`.

### Excluding files
```xml
<ItemGroup>
  <Compile Remove="GeneratedCode/**/*.cs" />
</ItemGroup>
```

---

## The folder layout

A typical solo project:
```
MyApp/
├── MyApp.csproj
├── Program.cs
├── Models/
│   ├── User.cs
│   └── Order.cs
├── Services/
│   └── UserService.cs
└── bin/                ← compiled output (gitignored)
    └── Debug/
        └── net10.0/
            ├── MyApp.dll
            ├── MyApp.exe (Windows) or MyApp (Linux)
            └── *.pdb     ← debug symbols
└── obj/                ← intermediate build files (gitignored)
```

A larger project with tests:
```
MyApp.sln
├── src/
│   ├── MyApp/                ← main project
│   │   ├── MyApp.csproj
│   │   └── ...
│   └── MyApp.Domain/         ← domain library
│       ├── MyApp.Domain.csproj
│       └── ...
└── tests/
    └── MyApp.Tests/          ← xUnit tests
        ├── MyApp.Tests.csproj
        └── ...
```

---

## Solutions (.sln files)

For multi-project setups, a **solution file** groups projects:

```bash
dotnet new sln -n MyApp
dotnet sln add src/MyApp/MyApp.csproj
dotnet sln add src/MyApp.Domain/MyApp.Domain.csproj
dotnet sln add tests/MyApp.Tests/MyApp.Tests.csproj
```

`MyApp.sln` is a text file (legacy format) that just lists projects + folder organization for IDEs. The CLI doesn't really need it — `dotnet build` on a folder finds all projects. But IDEs use it.

**Modern alternative**: `*.slnx` (XML solution format, introduced in .NET 9). Cleaner format:

```bash
dotnet new sln -n MyApp --format slnx
```

---

## Restore / Build / Run / Publish lifecycle

```
                      dotnet restore                dotnet build            dotnet run
.cs + .csproj  ───┬────────────────────►  obj/ ─────────────────►  bin/ ───────────────►  executes
                   resolve dependencies      compile to IL+metadata           load + JIT
```

### `dotnet restore`
- Reads `<PackageReference>` items.
- Resolves the dependency graph.
- Downloads packages to the global packages folder (`~/.nuget/packages` by default).
- Generates `obj/project.assets.json` describing the resolved tree.

You rarely run this manually — `dotnet build` and `dotnet run` invoke restore automatically.

### `dotnet build`
- Compiles all `.cs` files in the project to IL.
- Produces `bin/<Configuration>/<TargetFramework>/MyApp.dll`.
- Default configuration is `Debug`. Use `-c Release` for optimized builds:
  ```bash
  dotnet build -c Release
  ```

### `dotnet run`
- Implicit build (unless `--no-build`).
- Launches the resulting executable.
- Passes through command-line args after `--`:
  ```bash
  dotnet run -- --my-flag value
  ```

### `dotnet publish`
- Builds + bundles for deployment.
- Many flavors:
  ```bash
  # Framework-dependent — requires .NET runtime to be installed on target
  dotnet publish -c Release

  # Self-contained — includes the .NET runtime; runs anywhere
  dotnet publish -c Release -r linux-x64 --self-contained

  # Single-file — one binary instead of many DLLs
  dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true

  # Native AOT — fully compiled native binary, fastest startup
  dotnet publish -c Release -r linux-x64 -p:PublishAot=true
  ```

[Chapter 14 §06](../14-InteropAOT/06-PublishProfiles.md) covers publish modes in detail.

---

## `bin/` and `obj/`

You'll see these folders appear after the first build. They're:
- `obj/` — intermediate build artifacts. Restore manifests, partial IL, source generator outputs.
- `bin/` — final output: DLLs, EXEs, debug symbols.

**Always add to `.gitignore`**:
```
bin/
obj/
*.user
.vs/
```

Microsoft provides a comprehensive `.gitignore` template — just `dotnet new gitignore` in your repo root generates one.

---

## `Directory.Build.props` and `Directory.Build.targets`

For multi-project repos, you often want common settings (target framework, analyzers, nullable enabled) in every project. Drop a `Directory.Build.props` at the repo root:

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <LangVersion>14.0</LangVersion>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.0.0">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

MSBuild walks up from each `.csproj` looking for this file and imports it. Project-level `.csproj` settings still override. This keeps individual project files tiny.

---

## Central Package Management (CPM)

For repos with many projects, you can centralize package versions in `Directory.Packages.props`:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>

  <ItemGroup>
    <PackageVersion Include="Serilog" Version="4.1.0" />
    <PackageVersion Include="xunit" Version="2.9.0" />
  </ItemGroup>
</Project>
```

Then in any project:
```xml
<PackageReference Include="Serilog" />
<!-- No Version= attribute; comes from Directory.Packages.props -->
```

Now bumping a version is a single edit, not a scavenger hunt.

---

## A complete starter template

For a multi-project repo with library + tests:

```
.
├── .gitignore
├── Directory.Build.props
├── Directory.Packages.props
├── MyApp.sln
├── src/
│   └── MyApp/
│       ├── MyApp.csproj
│       └── Program.cs
└── tests/
    └── MyApp.Tests/
        ├── MyApp.Tests.csproj
        └── BasicTests.cs
```

Build it:
```bash
dotnet new sln -n MyApp
mkdir -p src/MyApp tests/MyApp.Tests
cd src/MyApp && dotnet new console && cd -
cd tests/MyApp.Tests && dotnet new xunit && cd -

dotnet sln add src/MyApp tests/MyApp.Tests
dotnet add tests/MyApp.Tests reference src/MyApp

dotnet build
dotnet test
```

You now have a project structure professional teams use.

---

## Summary

- The `.csproj` is the source of truth for everything about your build.
- Modern SDK-style projects are dramatically simpler than the .NET Framework era.
- `PropertyGroup` configures the build; `ItemGroup` says what's in it.
- Default behavior covers 95% of cases — minimal project files are normal.
- `Directory.Build.props` + CPM scale to large repos.
- The restore → build → run → publish lifecycle is consistent and CLI-first.

→ Continue to: [Questions.md](Questions.md)
