# Development Setup

## What you need

To write and run C#, you need three things:

1. **The .NET SDK** — the compiler, CLI, and templates.
2. **An editor or IDE** — VS Code, Visual Studio, JetBrains Rider, or even Notepad.
3. **(Optional but recommended) Git** — for source control. Not strictly needed for writing code.

That's it. No JDK setup ceremony, no compiler dance — `winget`, `apt`, `brew`, or the official installer and you're done.

---

## Installing the .NET 10 SDK

### Windows

**Option A — winget (recommended):**
```powershell
winget install Microsoft.DotNet.SDK.10
```

**Option B — installer:**
Download from <https://dotnet.microsoft.com/download/dotnet/10.0> → SDK x64 (or ARM64 for Surface Pro X / WSL2 on ARM).

### macOS

**Option A — Homebrew:**
```bash
brew install --cask dotnet-sdk
```
(This installs the latest LTS — currently .NET 10.)

**Option B — installer:**
Download `.pkg` from the official link above.

### Linux

**Ubuntu/Debian:**
```bash
# Add Microsoft package signing key + feed (once)
wget https://packages.microsoft.com/config/ubuntu/24.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb
sudo dpkg -i packages-microsoft-prod.deb
rm packages-microsoft-prod.deb

# Install
sudo apt update
sudo apt install -y dotnet-sdk-10.0
```

**Fedora/RHEL:**
```bash
sudo dnf install dotnet-sdk-10.0
```

**Arch:**
```bash
sudo pacman -S dotnet-sdk
```

### Verify the install

In any terminal:
```bash
dotnet --info
```

You should see output like:
```
.NET SDK:
 Version:           10.0.100
 Commit:            ...
 Workload version:  ...

Runtime Environment:
 OS Name:     ubuntu
 OS Version:  24.04
 OS Platform: Linux
 ...

.NET SDKs installed:
 10.0.100 [/usr/share/dotnet/sdk]

.NET runtimes installed:
 Microsoft.AspNetCore.App 10.0.0
 Microsoft.NETCore.App 10.0.0
 ...
```

If you see "Command not found" or no version output, the install didn't finish — re-run or restart your shell.

---

## Choosing an editor

| Editor | Cost | Best for | Pros | Cons |
|---|---|---|---|---|
| **Visual Studio Code** | Free | Most developers | Cross-platform, fast, huge extension ecosystem, lightweight | Less powerful debugger than full Visual Studio |
| **Visual Studio 2022/2026** | Free (Community) / paid (Pro, Enterprise) | Windows-first, large solutions | Best debugger, profilers, designers (WinForms/WPF) | Windows only, heavyweight |
| **JetBrains Rider** | Paid (free for students/OSS) | Cross-platform, refactoring-heavy work | Excellent refactoring, debugger, navigation, runs on macOS/Linux | Costs money |
| **Visual Studio for Mac** | — | — | **Discontinued** August 2024. Use Rider or VS Code instead. |
| **Vim / Neovim / Emacs** | Free | Existing terminal users | Works if you really want it; OmniSharp or `csharp-ls` | More setup; debugger story is rough |
| **Notepad** | Free | Demonstrations only | None | Nope |

For this book's exercises, **VS Code is sufficient**. If you're doing professional Windows development, **Visual Studio** has the deepest toolset. For cross-platform pro work, **Rider** is fantastic.

---

## Setting up VS Code for C#

1. Install VS Code: <https://code.visualstudio.com>.
2. Install the **C# Dev Kit** extension (from Microsoft). It bundles:
   - The base **C#** extension (powered by Roslyn LSP).
   - **C# Dev Kit** itself — solution explorer, test runner, AI features.
   - **IntelliCode for C# Dev Kit** — context-aware completions.
3. Open your project folder: `code my-project/`.
4. The first time you open a `.cs` file, the extension downloads .NET tooling. Wait ~30 seconds.

### Useful VS Code shortcuts for C#

| Action | Shortcut (Win/Linux) | Shortcut (macOS) |
|---|---|---|
| Go to definition | F12 | F12 |
| Find references | Shift+F12 | Shift+F12 |
| Rename symbol | F2 | F2 |
| Quick fix | Ctrl+. | Cmd+. |
| Format document | Shift+Alt+F | Shift+Opt+F |
| Run/Debug | F5 | F5 |
| Start without debug | Ctrl+F5 | Cmd+F5 |

---

## The dotnet CLI

The most important tool you'll learn first. `dotnet` is the gateway to everything:

```bash
dotnet new <template>    # Create a new project from a template
dotnet restore           # Download package dependencies
dotnet build             # Compile
dotnet run               # Build and run
dotnet test              # Run unit tests
dotnet publish           # Produce a deployable output
dotnet pack              # Produce a NuGet package
dotnet tool install ...  # Install global CLI tools
dotnet --info            # Show SDK/runtime info
dotnet --list-sdks       # List installed SDKs
dotnet --list-runtimes   # List installed runtimes
```

You'll use `dotnet new`, `dotnet run`, and `dotnet test` daily. Chapter 15 §01 covers the full CLI.

### Available templates

```bash
dotnet new list
```

A taste:
- `console` — console application
- `classlib` — class library (DLL)
- `web` — empty ASP.NET Core app
- `webapi` — Web API project
- `mvc` — ASP.NET Core MVC
- `blazor` — Blazor app
- `worker` — long-running background service
- `xunit` — xUnit test project

Example:
```bash
mkdir HelloWorld
cd HelloWorld
dotnet new console
dotnet run
```

Output: `Hello, World!`

---

## Optional but recommended tools

### Global CLI tools

These are installed once and available everywhere on your machine.

```bash
# Diagnostic tools (very useful)
dotnet tool install --global dotnet-counters
dotnet tool install --global dotnet-trace
dotnet tool install --global dotnet-dump
dotnet tool install --global dotnet-gcdump

# CSV-style code formatter for CI (built into dotnet 7+, but explicit install pins the version)
dotnet tool install --global dotnet-format

# EF Core migrations
dotnet tool install --global dotnet-ef
```

After installing, run them with their bare name: `dotnet-counters monitor -p 1234`.

### Git

If you don't have it:
- Windows: `winget install Git.Git`
- macOS: `brew install git`
- Linux: it's almost certainly already installed.

Configure once:
```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"
```

### Recommended VS Code extensions (beyond C# Dev Kit)

- **GitLens** — supercharged Git annotations in the editor.
- **Error Lens** — surfaces errors and warnings inline.
- **GitHub Pull Requests** — review and create PRs from VS Code.
- **REST Client** — send HTTP requests from `.http` files (very handy for testing APIs).
- **markdownlint** — lints Markdown.

---

## Side-by-side SDK installs

You can have multiple .NET SDKs installed at once. To pick which one a folder uses, drop a `global.json` in the folder:

```json
{
  "sdk": {
    "version": "10.0.100",
    "rollForward": "latestPatch"
  }
}
```

`rollForward` options: `disable`, `patch`, `feature`, `minor`, `major`, `latestPatch`, `latestFeature`, `latestMinor`, `latestMajor`. The most common is `latestPatch` — use exactly the major.minor specified, but accept newer patches.

`dotnet --version` from inside that folder will report the pinned version.

---

## Verifying your setup with a real program

Create a folder, run:

```bash
dotnet new console
dotnet run
```

You should see `Hello, World!`.

Open `Program.cs` — modern templates use **top-level statements** (C# 9+):

```csharp
Console.WriteLine("Hello, World!");
```

That's the whole program. No `class`, no `Main`. We'll explain how this works in [04-FirstProgram.md](04-FirstProgram.md).

---

## Common setup gotchas

- **PATH issues**: if `dotnet` works in one terminal but not another, restart the terminal — installers don't refresh existing shell environments.
- **Multiple .NET Framework vs .NET installations on Windows**: they coexist. The 32-bit .NET Framework dir is `C:\Windows\Microsoft.NET\Framework`. The modern .NET is at `C:\Program Files\dotnet`. Run `where dotnet` (Windows) or `which dotnet` (macOS/Linux) to see which one your shell picks.
- **`dotnet new console` complains about "no templates"**: SDK isn't installed correctly, or your shell is finding a runtime-only install.
- **macOS Gatekeeper / Linux missing libssl**: rare, but `dotnet --info` will tell you what's wrong.
- **VS Code says "OmniSharp" instead of using the modern extension**: the new C# Dev Kit replaced OmniSharp. Update the extension and check that "Use Modern .NET" is enabled in settings.

---

## What you should have now

- `dotnet --info` works and shows .NET 10 SDK + runtime.
- An editor of choice with C# support installed.
- The ability to `dotnet new console`, `dotnet run`, and see "Hello, World!".

That's everything you need for the rest of the book.

→ Next: [04-FirstProgram.md](04-FirstProgram.md)
