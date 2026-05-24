# Chapter 00 — Coding Problems

> Hands-on. Install, build, run, inspect. Every problem can be done on your machine in 10 minutes or less. Solutions are hidden — try first.

---

## Problem 1 — Install and verify

Install the .NET 10 SDK on your machine. Then verify it.

**Goal**: see "10.0.x" in `dotnet --version`.

<details><summary>Solution</summary>

Pick the install method for your OS (see [03-DevelopmentSetup.md](03-DevelopmentSetup.md)).

Then:
```bash
dotnet --version
# expected: 10.0.100 (or newer patch)

dotnet --list-sdks
# expected: list including 10.0.100

dotnet --list-runtimes
# expected: list including Microsoft.NETCore.App 10.0.0
```

If `dotnet --version` shows an older version even after install, you might have a `global.json` somewhere up the directory tree pinning to an older SDK. Run `dotnet --info` to see which SDK is being selected and why.

</details>

---

## Problem 2 — Three ways to Hello World

Write the same "Hello, World!" program three times — classic Main, top-level statements, file-based — and run each.

<details><summary>Solution</summary>

**Classic:**
```bash
mkdir hello-classic && cd hello-classic
dotnet new console --use-program-main
dotnet run
```
The generated `Program.cs` has `static void Main`.

**Top-level statements:**
```bash
mkdir ../hello-toplevel && cd ../hello-toplevel
dotnet new console
dotnet run
```
The generated `Program.cs` has a single `Console.WriteLine` line.

**File-based:**
```bash
mkdir ../hello-file && cd ../hello-file
echo 'Console.WriteLine("Hello, World!");' > hello.cs
dotnet run hello.cs
```
No project, no folders — works because we're on .NET 10 + C# 14.

</details>

---

## Problem 3 — Show me the IL

Compile this method and look at its IL:

```csharp
public static int Add(int a, int b) => a + b;
```

**Goal**: see the IL instructions for `Add`.

<details><summary>Solution</summary>

**Easiest** — paste into <https://sharplab.io> and select "IL" view. You'll see something like:
```
ldarg.0   // load 'a'
ldarg.1   // load 'b'
add
ret
```

**Local tool** — `dotnet ildasm`:
```bash
dotnet new classlib
# replace generated code with the Add method, then:
dotnet build
dotnet ildasm bin/Debug/net10.0/ClassLib.dll
```
Look for the `Add` method in the output.

</details>

---

## Problem 4 — Inspect a NuGet package

Add Serilog to a project. Find where it lives on disk.

<details><summary>Solution</summary>

```bash
dotnet new console -n SerilogExperiment
cd SerilogExperiment
dotnet add package Serilog
dotnet add package Serilog.Sinks.Console

# Update Program.cs:
cat > Program.cs <<'EOF'
using Serilog;

Log.Logger = new LoggerConfiguration()
    .WriteTo.Console()
    .CreateLogger();

Log.Information("Hello from Serilog!");
Log.CloseAndFlush();
EOF

dotnet run
```

The packages are downloaded to your global packages folder, default `~/.nuget/packages` (Linux/macOS) or `%USERPROFILE%\.nuget\packages` (Windows).

```bash
ls ~/.nuget/packages/serilog/
# 4.1.0/  (or whichever version)
ls ~/.nuget/packages/serilog/4.1.0/lib/
# net6.0/  net7.0/  net8.0/  netstandard2.0/  netstandard2.1/
```

The package ships multiple target frameworks. NuGet picks the most compatible one for your project — for `net10.0`, that's `net8.0` (highest matching).

</details>

---

## Problem 5 — Pin an older SDK

Create a project that uses .NET 8 SDK even though you have .NET 10 installed.

<details><summary>Solution</summary>

```bash
mkdir net8-project && cd net8-project
cat > global.json <<'EOF'
{
  "sdk": {
    "version": "8.0.400",
    "rollForward": "latestPatch"
  }
}
EOF

dotnet --version
# 8.0.400 — even though .NET 10 is installed!

dotnet new console
# uses the 8.0 SDK's templates
```

`global.json` affects the entire directory tree from where it's placed downward. Delete it (or run `dotnet --version` from another folder) and you're back to the default SDK.

</details>

---

## Problem 6 — Inspect what ImplicitUsings adds

Write a console program that uses `List<int>`, `Task`, and `File.ReadAllText` — all without writing a single `using` directive. Then disable ImplicitUsings and see what breaks.

<details><summary>Solution</summary>

```bash
dotnet new console -n ImplicitsDemo
cd ImplicitsDemo
```

Replace `Program.cs`:
```csharp
var list = new List<int> { 1, 2, 3 };
var task = Task.FromResult(42);
var text = File.Exists("missing.txt") ? File.ReadAllText("missing.txt") : "n/a";
Console.WriteLine($"{list.Count} {task.Result} {text}");
```

`dotnet run` — works fine.

Now edit `ImplicitsDemo.csproj`:
```xml
<ImplicitUsings>disable</ImplicitUsings>
```

`dotnet build`:
```
error CS0246: The type or namespace name 'List<>' could not be found
error CS0103: The name 'Task' does not exist in the current context
error CS0103: The name 'File' does not exist in the current context
error CS0103: The name 'Console' does not exist in the current context
```

ImplicitUsings was adding `System`, `System.Collections.Generic`, `System.IO`, `System.Threading.Tasks` for free. With it off, you'd need every `using` explicitly.

</details>

---

## Problem 7 — Self-contained publish

Publish a console app as a single-file self-contained Linux binary, even if you're on Windows.

<details><summary>Solution</summary>

```bash
dotnet new console -n MyTool
cd MyTool
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true
```

Look in `bin/Release/net10.0/linux-x64/publish/`:
```
MyTool       ← single executable, ~70-80 MB (includes the runtime)
MyTool.pdb   ← debug symbols
```

You can `scp` `MyTool` to a Linux machine and run it without installing .NET.

To make it smaller, add trimming (more on this in Chapter 14):
```bash
dotnet publish -c Release -r linux-x64 --self-contained -p:PublishSingleFile=true -p:PublishTrimmed=true
```
Now it's ~20-30 MB.

Or go all the way:
```bash
dotnet publish -c Release -r linux-x64 -p:PublishAot=true
```
~5-10 MB, instant startup, but no JIT/reflection emit.

</details>

---

## Problem 8 — Multi-target a library

Create a class library that compiles against both .NET 10 and .NET Standard 2.0.

<details><summary>Solution</summary>

```bash
dotnet new classlib -n MyLib
cd MyLib
```

Edit `MyLib.csproj` — change `<TargetFramework>` to `<TargetFrameworks>` (plural):

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFrameworks>net10.0;netstandard2.0</TargetFrameworks>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
  </PropertyGroup>
</Project>
```

`dotnet build`:
```
ls bin/Debug/
# net10.0/  netstandard2.0/  ← two outputs from one build
```

Now, what if you need to use a .NET 10-only API but still target netstandard2.0? Use a `#if`:

```csharp
public string Format(int n) {
#if NET10_0_OR_GREATER
    return $"Value: {n:N0}";
#else
    return "Value: " + n.ToString("N0");   // older API
#endif
}
```

The `NET10_0_OR_GREATER` symbol is defined automatically by the SDK when targeting that framework.

</details>

---

## Problem 9 — A polyglot file-based script

Write a file-based app that uses a NuGet package via the `#:package` directive.

<details><summary>Solution</summary>

Save as `colors.cs`:
```csharp
#:package Spectre.Console@0.49.1

using Spectre.Console;

AnsiConsole.MarkupLine("[green]Hello[/], [yellow]colorful[/] [red]world[/]!");

var table = new Table();
table.AddColumn("Language");
table.AddColumn("Version");
table.AddRow("C#", "14");
table.AddRow(".NET", "10 LTS");
AnsiConsole.Write(table);
```

```bash
dotnet run colors.cs
```

First run: ~10-15 seconds (resolving NuGet, compiling). Subsequent runs: instant (cached).

Compare to setting up a real `dotnet new console` + `dotnet add package` for the same output — file-based apps are dramatically faster for one-offs.

</details>

---

## Problem 10 — Convert a file-based app to a project

Take your `colors.cs` from Problem 9 and convert it to a full project.

<details><summary>Solution</summary>

```bash
dotnet project convert colors.cs
```

This generates:
```
colors/
├── colors.csproj
└── Program.cs   (was colors.cs)
```

with the `#:package` directives translated into `<PackageReference>` entries.

```bash
cd colors && dotnet run
```

Same output. Now your "script" is a real project — add tests, version it, ship a NuGet, whatever.

</details>

---

## Problem 11 — Find what got installed

You ran `dotnet tool install --global dotnet-counters` last week. Where did it actually go on disk?

<details><summary>Solution</summary>

Global .NET tools live in:
- **Linux/macOS**: `~/.dotnet/tools/`
- **Windows**: `%USERPROFILE%\.dotnet\tools\`

```bash
ls ~/.dotnet/tools/
# dotnet-counters  dotnet-counters.exe (on Windows)  ...
```

That folder must be on your PATH for the tools to be invokable by bare name. The first `dotnet tool install` warns if it isn't.

To list installed tools:
```bash
dotnet tool list --global
```

</details>

---

## Problem 12 — Build a simple Directory.Build.props

Create a 2-project solution where both projects share Nullable, ImplicitUsings, and a common analyzer via `Directory.Build.props`.

<details><summary>Solution</summary>

```
my-repo/
├── Directory.Build.props
├── MyApp.sln
├── src/
│   └── App/
│       └── App.csproj
└── tests/
    └── App.Tests/
        └── App.Tests.csproj
```

`Directory.Build.props`:
```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <LangVersion>14.0</LangVersion>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.NetAnalyzers" Version="9.0.0">
      <PrivateAssets>all</PrivateAssets>
    </PackageReference>
  </ItemGroup>
</Project>
```

`src/App/App.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
  </PropertyGroup>
</Project>
```

No `TargetFramework`, `Nullable`, or analyzer here — they're all inherited.

`tests/App.Tests/App.Tests.csproj`:
```xml
<Project Sdk="Microsoft.NET.Sdk">
  <ItemGroup>
    <PackageReference Include="xunit" Version="2.9.0" />
    <PackageReference Include="xunit.runner.visualstudio" Version="2.8.0" />
    <PackageReference Include="Microsoft.NET.Test.Sdk" Version="17.10.0" />
    <ProjectReference Include="..\..\src\App\App.csproj" />
  </ItemGroup>
</Project>
```

Build:
```bash
dotnet build my-repo/
```

Both projects compile with the shared settings, and updating the target framework is one line in one file.

</details>

---

## Problem 13 — Add a tool reference (project-local)

Add `dotnet-format` as a *project-local* tool (not global).

<details><summary>Solution</summary>

```bash
cd my-repo
dotnet new tool-manifest         # creates .config/dotnet-tools.json
dotnet tool install dotnet-format
```

`dotnet-tools.json` now contains:
```json
{
  "version": 1,
  "isRoot": true,
  "tools": {
    "dotnet-format": {
      "version": "5.1.0",
      "commands": ["dotnet-format"]
    }
  }
}
```

To use:
```bash
dotnet tool restore        # one-time, in case a teammate clones the repo
dotnet format              # uses the manifest's pinned version
```

Project-local tools are great for CI — pinned versions, no installation step required by users, no global state pollution.

</details>

---

## Bonus — find out how big things are

How big is a "hello world" published every way?

<details><summary>Sample numbers (Linux x64, .NET 10)</summary>

| Mode | Approx size |
|---|---|
| Framework-dependent | ~150 KB (just your DLLs) |
| Self-contained | ~75 MB (runtime bundled) |
| Self-contained, single-file | ~75 MB (same, just packaged) |
| Self-contained, single-file, trimmed | ~25 MB |
| Native AOT | ~5–10 MB |

Trimming and AOT remove huge amounts of unused BCL code. AOT goes further by also eliminating the JIT.

</details>

---

That's Chapter 00 done. You should now have a working SDK, an editor, and a sense of what's happening when `dotnet run` runs your program.

→ [Chapter 01 — Fundamentals](../01-Fundamentals/README.md)
