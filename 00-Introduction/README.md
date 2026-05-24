# Chapter 00 — Introduction

> Orientation. By the end of this chapter you'll know what C# is, what the .NET runtime does for you, how to install the SDK, and how to write and run your first program.

**Prerequisites**: none. This is page 1.

**Time to read**: ~30 minutes.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-WhatIsCSharp.md](01-WhatIsCSharp.md) | The language: history, philosophy, where C# sits in the language landscape, what kind of programs you can write with it. |
| [02-DotNetEcosystem.md](02-DotNetEcosystem.md) | The platform: CLR vs BCL vs SDK, .NET Framework vs .NET Core vs modern .NET, what JIT means, what the runtime does at startup. |
| [03-DevelopmentSetup.md](03-DevelopmentSetup.md) | Getting set up: installing the .NET 10 SDK, choosing an IDE (Visual Studio, Rider, VS Code), what the `dotnet` CLI is. |
| [04-FirstProgram.md](04-FirstProgram.md) | Hello, world: classic Program.Main, top-level statements (C# 9), file-based apps (`dotnet run app.cs`, C# 14). |
| [05-ProjectStructure.md](05-ProjectStructure.md) | The .csproj file: SDK-style projects, target frameworks, restore/build/run/publish lifecycle. |
| [Questions.md](Questions.md) | Drilling questions on everything in this chapter. |
| [Coding.md](Coding.md) | Hands-on coding problems — install, build, run, modify. |

---

## Learning objectives

After this chapter you should be able to:
- Explain what a managed runtime is and why it matters.
- Install the .NET SDK and verify the install with `dotnet --info`.
- Create, build, and run a console application three different ways: with a `.csproj`, with top-level statements, and with a single `.cs` file (C# 14).
- Read a `.csproj` and know what every line does.
- Pronounce "C-Sharp" with confidence at a job interview.

---

## How to read this chapter

Read the sub-files in order. If you already have C# experience and you're skimming, at minimum read:
- §04 (the C# 14 file-based apps feature is brand new),
- §05 (modern SDK-style projects are different from .NET Framework era).

Then attempt `Questions.md` — if you score 80%+ you can move to Chapter 01.

→ Begin: [01-WhatIsCSharp.md](01-WhatIsCSharp.md)
