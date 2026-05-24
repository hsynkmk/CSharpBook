# The dotnet CLI

## What it is

`dotnet` is the command-line driver for the .NET SDK. It creates, builds, runs, tests, packs, and publishes projects, and manages tools. Everything an IDE does, it does via `dotnet` (or MSBuild) under the hood — so knowing the CLI means you can build anywhere (CI, containers, scripts).

```bash
dotnet --version          # SDK version
dotnet --info             # SDKs, runtimes, RIDs installed
dotnet --list-sdks
dotnet --list-runtimes
```

---

## SDK vs runtime

- **Runtime** — runs .NET apps. End users need only the runtime (for framework-dependent apps).
- **SDK** — builds .NET apps. Includes the runtime + compilers (Roslyn) + MSBuild + CLI + templates. Developers install the SDK.

```bash
dotnet --list-sdks       # 10.0.100 [C:\Program Files\dotnet\sdk]
dotnet --list-runtimes   # Microsoft.NETCore.App 10.0.0, Microsoft.AspNetCore.App 10.0.0, ...
```

A `global.json` pins the SDK version for a repo:

```json
{ "sdk": { "version": "10.0.100", "rollForward": "latestfeature" } }
```

---

## Core commands

### `dotnet new` — create from templates

```bash
dotnet new list                    # show available templates
dotnet new console -o MyApp        # console app in MyApp/
dotnet new classlib -o MyLib
dotnet new web                     # ASP.NET Core minimal API
dotnet new webapi
dotnet new xunit -o MyTests
dotnet new sln -n MySolution       # solution file
dotnet new gitignore               # .gitignore for .NET
dotnet new editorconfig            # .editorconfig
```

Add projects to a solution:

```bash
dotnet sln add MyApp/MyApp.csproj
dotnet sln add MyLib/MyLib.csproj
```

Add a project reference:

```bash
dotnet add MyApp reference MyLib/MyLib.csproj
```

### `dotnet restore` — fetch dependencies

```bash
dotnet restore
```

Downloads NuGet packages declared in `.csproj`/`Directory.Packages.props` into the global package cache (`~/.nuget/packages`) and writes `obj/project.assets.json`. Most other commands restore implicitly first (use `--no-restore` to skip).

### `dotnet build` — compile

```bash
dotnet build                       # Debug by default
dotnet build -c Release
dotnet build --no-restore
dotnet build /p:TreatWarningsAsErrors=true
```

Compiles to `bin/<config>/<tfm>/`. Produces IL assemblies (`.dll`), not a native binary (that's `publish` with AOT).

### `dotnet run` — build + execute

```bash
dotnet run                          # build and run the project in cwd
dotnet run -c Release
dotnet run --project MyApp
dotnet run -- arg1 arg2             # args after -- go to your app
dotnet run app.cs                   # file-based app (C# 14 / .NET 10) — no project needed
```

`dotnet run` is for development. For deployment, `publish` then run the output directly (skips the build step on each launch).

### `dotnet test` — run tests

```bash
dotnet test
dotnet test --filter "Category=Unit"
dotnet test --collect:"XPlat Code Coverage"
dotnet test --logger "trx;LogFileName=results.trx"
```

Discovers and runs xUnit/NUnit/MSTest via the test SDK. See [Chapter 16](../16-Testing/README.md).

### `dotnet publish` — deployable output

```bash
dotnet publish -c Release                                  # framework-dependent
dotnet publish -c Release -r linux-x64 --self-contained    # self-contained
dotnet publish -c Release -r win-x64 -p:PublishSingleFile=true
dotnet publish -c Release -r linux-x64 -p:PublishAot=true  # native AOT
```

Produces the artifact you deploy. See [Chapter 14 §06](../14-InteropAOT/06-PublishProfiles.md).

### `dotnet pack` — create a NuGet package

```bash
dotnet pack -c Release                  # produces bin/Release/MyLib.1.0.0.nupkg
dotnet pack -c Release -p:Version=2.1.0
```

See [03-NuGet.md](03-NuGet.md).

---

## `dotnet tool` — global and local tools

CLI tools distributed as NuGet packages.

```bash
# Global (available everywhere)
dotnet tool install -g dotnet-ef
dotnet tool install -g dotnet-counters
dotnet tool update -g dotnet-ef
dotnet tool list -g

# Local (per-repo, version-pinned via manifest)
dotnet new tool-manifest                       # creates .config/dotnet-tools.json
dotnet tool install dotnet-ef                  # local
dotnet tool restore                            # restore from manifest (CI)
dotnet ef migrations add Init                  # run a local tool
```

**Local tools** are preferred for projects — the manifest pins versions so the whole team and CI use the same tool version. Common tools: `dotnet-ef`, `dotnet-counters`, `dotnet-trace`, `dotnet-dump`, `dotnet-format`.

---

## Package management commands

```bash
dotnet add package Newtonsoft.Json                 # add latest
dotnet add package Serilog --version 4.0.0
dotnet remove package Serilog
dotnet list package                                # installed packages
dotnet list package --outdated                     # available updates
dotnet list package --vulnerable                   # known CVEs
dotnet list package --include-transitive           # full dependency tree
```

`--vulnerable` and `--outdated` are essential for security hygiene in CI.

---

## Useful flags

```bash
-c|--configuration Release      # build configuration
-r|--runtime linux-x64          # target RID
-f|--framework net10.0          # target framework (multi-targeted projects)
-v|--verbosity [q|m|n|d|diag]   # log detail
--no-restore                    # skip implicit restore
--no-build                      # skip build (run/test on prior output)
-p:Property=Value               # set an MSBuild property
```

Example diagnosing a build:

```bash
dotnet build -v diag > build.log    # full diagnostic MSBuild log
```

---

## `dotnet format` — code formatting

```bash
dotnet format                       # fix formatting per .editorconfig
dotnet format --verify-no-changes   # CI: fail if not formatted
dotnet format style                 # apply code-style fixes
dotnet format analyzers             # apply analyzer fixes
```

Enforces `.editorconfig` rules across the repo. See [06-EditorConfig.md](06-EditorConfig.md).

---

## Watch mode

```bash
dotnet watch run        # rebuild + restart on file change (hot reload where possible)
dotnet watch test       # re-run tests on change
```

`dotnet watch` uses Hot Reload to apply many edits without restart — fast inner loop for development.

---

## Typical workflows

### New solution from scratch

```bash
dotnet new sln -n Shop
dotnet new web -o src/Shop.Api
dotnet new classlib -o src/Shop.Core
dotnet new xunit -o tests/Shop.Tests
dotnet sln add src/Shop.Api src/Shop.Core tests/Shop.Tests
dotnet add src/Shop.Api reference src/Shop.Core
dotnet add tests/Shop.Tests reference src/Shop.Core
dotnet build
dotnet test
```

### CI build script

```bash
dotnet restore
dotnet build -c Release --no-restore
dotnet test -c Release --no-build --logger trx
dotnet publish src/Shop.Api -c Release -o ./out --no-build
```

`--no-restore`/`--no-build` avoid redundant work across steps, speeding up CI.

---

## Common bugs / gotchas

### Wrong SDK version

If `global.json` pins an SDK you don't have, commands fail. Install the pinned SDK or adjust `rollForward`.

### Forgetting `--` for app args

```bash
dotnet run --myflag       # ⚠ — dotnet tries to interpret --myflag
dotnet run -- --myflag    # ✓ — passed to your app
```

### Stale build output

If you see odd behavior, `dotnet clean` (or delete `bin`/`obj`) and rebuild. `obj/project.assets.json` can go stale after manual edits.

### Implicit restore surprises in CI

Network-restricted CI may fail on implicit restore. Run an explicit `dotnet restore` step (with a configured feed) first, then `--no-restore` on subsequent commands.

---

## Summary

- `dotnet` drives the SDK: `new`, `restore`, `build`, `run`, `test`, `publish`, `pack`, `tool`.
- SDK builds; runtime runs. Pin the SDK with `global.json`.
- Use **local tools** (manifest) for version-consistent per-repo tooling.
- `dotnet list package --vulnerable/--outdated` for dependency hygiene.
- `dotnet format` enforces `.editorconfig`; `dotnet watch` powers the fast dev loop.
- In CI, chain `restore` → `build --no-restore` → `test --no-build` → `publish --no-build`.

→ Next: [02-Csproj.md](02-Csproj.md)
