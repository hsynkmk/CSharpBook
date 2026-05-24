# Global Usings and Implicit Usings

## What it is

C# 10 added two related features that reduce `using` boilerplate:

- **Global usings** — declare `using` once for the whole project.
- **Implicit usings** — the SDK auto-includes common namespaces.

```csharp
// GlobalUsings.cs (anywhere in your project)
global using System;
global using System.Linq;
global using System.Collections.Generic;
```

```csharp
// Any other .cs file in the project
// No "using" needed!
var list = new List<int> { 1, 2, 3 };
var sum = list.Sum();
```

Plus, with `<ImplicitUsings>enable</ImplicitUsings>` in csproj, the SDK adds a default set of usings — System, System.IO, System.Linq, System.Threading.Tasks, etc.

Net effect: most files no longer need any `using` directives. The SDK + global usings cover the common namespaces.

---

## Why it exists

Every C# file traditionally started with 5-10 `using` lines:

```csharp
using System;
using System.Collections.Generic;
using System.IO;
using System.Linq;
using System.Threading.Tasks;
// ... 5 more ...

namespace MyApp;

public class Service { ... }
```

Repetitive. Annoying. Sometimes a file needs unusual usings; sometimes just the basics. Global / implicit usings consolidate the basics out of every file.

For ASP.NET Core projects, the implicit usings include MVC, routing, hosting — most of what you'd write for an API.

---

## Implicit usings

In your `.csproj`:

```xml
<PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
</PropertyGroup>
```

When enabled, the SDK injects a set of global usings based on the project SDK.

For `Microsoft.NET.Sdk` (Console, ClassLib):
- `System`
- `System.Collections.Generic`
- `System.IO`
- `System.Linq`
- `System.Net.Http`
- `System.Threading`
- `System.Threading.Tasks`

For `Microsoft.NET.Sdk.Web` (ASP.NET Core):
- All the above, PLUS
- `Microsoft.AspNetCore.Builder`
- `Microsoft.AspNetCore.Hosting`
- `Microsoft.AspNetCore.Http`
- `Microsoft.AspNetCore.Routing`
- `Microsoft.Extensions.Configuration`
- `Microsoft.Extensions.DependencyInjection`
- `Microsoft.Extensions.Hosting`
- `Microsoft.Extensions.Logging`

For `Microsoft.NET.Sdk.Worker`:
- Common stuff + the hosting / DI namespaces.

For `dotnet new console` and other modern templates, `<ImplicitUsings>enable</ImplicitUsings>` is set by default.

---

## Custom implicit usings

You can add to (or remove from) the implicit set:

```xml
<ItemGroup>
    <Using Include="MyApp.Domain" />
    <Using Include="System.Math" Static="True" />
    <Using Remove="System.Net.Http" />
</ItemGroup>
```

- `<Using Include="..." />` — add a global using.
- `<Using Include="..." Static="True" />` — `using static`, lets you call `Sqrt(4)` instead of `Math.Sqrt(4)`.
- `<Using Include="MyCustom" Alias="X" />` — `using X = MyCustom`.
- `<Using Remove="..." />` — exclude one of the SDK defaults.

Customize per project's domain. For a team standard, set in `Directory.Build.props`.

---

## Global usings in source

You can also declare global usings in any `.cs` file:

```csharp
// GlobalUsings.cs (convention: one file in the project)
global using System;
global using System.Linq;
global using MyApp.Common;
global using static System.Math;
global using AppLogger = MyApp.Logging.MyLogger;
```

These apply to ALL `.cs` files in the project as if each had `using ...`.

By convention: put them in a single file like `GlobalUsings.cs` or `Usings.cs`. Some teams use `_Imports.cs` (Blazor convention).

---

## `using static` globally

```xml
<Using Include="System.Math" Static="True" />
```

Or:
```csharp
global using static System.Math;
```

Now:
```csharp
var r = Sqrt(2);    // no Math. prefix
var p = PI;
```

For domain code with lots of math, this can be cleaner. For most code, keep the prefix to clarify origin.

---

## Aliases globally

```csharp
global using DateOnly = System.DateOnly;
global using MyJson = System.Text.Json.JsonSerializer;
```

Aliases globally — every file sees them. Useful for resolving ambiguities once.

---

## When global / implicit usings hurt

### Hidden dependencies

Pre-global-using, a `using` at the top of a file documented its dependencies. Now they're invisible:

```csharp
public class Service {
    public void M() {
        var list = new List<int>();          // where's List from?
        var users = list.Where(...);         // where's Where from?
    }
}
```

A reader has to know "they're implicit" or check `Usings.cs`. Marginal cognitive cost.

For most code, the implicit set is universal (System, System.Linq, etc.) — readers know these are everywhere. No problem.

### Ambiguity surprises

If two libraries both define `Foo`, and both are in your global usings, you have a conflict everywhere. Pre-globals, only files that explicitly `using` both saw the conflict.

Solution: avoid putting potentially-conflicting namespaces in globals.

---

## File-scoped namespace

Often combined with global usings:

```csharp
// Old
namespace MyApp {
    public class Service { ... }
}

// New (C# 10+)
namespace MyApp;

public class Service { ... }
```

`namespace MyApp;` (with semicolon) applies to the entire file. One less indentation level. Combined with global usings: cleaner files.

```csharp
namespace MyApp.Services;

public class Service {
    public void Do() {
        var list = new List<int>();
        // implicit using System.Collections.Generic
    }
}
```

That's a full file. Three lines of "actual code" beyond the namespace declaration.

---

## Common patterns

### Per-project standardization

In `Directory.Build.props`:

```xml
<PropertyGroup>
    <ImplicitUsings>enable</ImplicitUsings>
    <Nullable>enable</Nullable>
</PropertyGroup>

<ItemGroup>
    <Using Include="MyApp.Common" />
    <Using Include="MyApp.Domain" />
</ItemGroup>
```

All projects in the repo get the same defaults. New files can skip 5+ using lines.

### Test projects with shared utilities

```csharp
// tests/GlobalUsings.cs
global using Xunit;
global using FluentAssertions;
global using MyApp;
global using static Xunit.Assert;
```

Now test files are pure test logic — no using overhead.

### Blazor _Imports.razor

Blazor has its own version: `_Imports.razor` is a file with `@using` directives applied to all `.razor` files in the folder (and subfolders).

```razor
@using Microsoft.AspNetCore.Components.Forms
@using MyApp.Models
```

Equivalent concept for components.

---

## Internals — what's emitted

Implicit usings result in the SDK generating a hidden file like:

```csharp
// obj/Debug/net10.0/MyApp.GlobalUsings.g.cs (auto-generated)
global using System;
global using System.Collections.Generic;
global using System.IO;
global using System.Linq;
global using System.Net.Http;
global using System.Threading;
global using System.Threading.Tasks;
```

This file is part of the compilation. The compiler treats it like any other source.

Custom `<Using>` items in the csproj add lines to this generated file.

Once compiled, there's no trace — the usings only affect compile-time symbol resolution. Runtime sees no difference.

---

## When to use / when to avoid

✓ Most modern projects — enable both implicit and a few custom globals.
✓ Standardize across a repo via `Directory.Build.props`.
✓ ASP.NET Core projects (the implicit set covers most needs).
✓ Test projects (assertion library + test framework can be global).

✗ Library projects where readers benefit from explicit dependencies — case-by-case.
✗ Anywhere file-local usings document important context.

For consumer-facing libraries: be more conservative. The user reading your code might find missing usings confusing.

For application code: enable freely.

---

## Migration

Existing project without implicit usings:

1. Set `<ImplicitUsings>enable</ImplicitUsings>`.
2. Build — observe warnings or remove redundant `using` lines.
3. Roslyn has a code fix: "Remove unnecessary usings" — runs across the project, deletes ones now provided implicitly.

For new projects: default is enabled.

---

## Common bugs

### Using two namespaces that have the same type name

```csharp
// GlobalUsings.cs
global using System.Drawing;       // has Color
global using MyApp.Themes;          // has Color
```

In every file, `Color` is now ambiguous. Compile errors propagate.

Fix: don't globally use both; or alias one (`global using ThemeColor = MyApp.Themes.Color;`).

### Removing a global using and breaking many files

Once a global using is established, removing it triggers errors in every file using its symbols. Caution when changing globals.

### Forgetting that implicit usings don't add references

Implicit `using System.Net.Http` doesn't add the assembly reference — that's handled separately by the SDK. For most BCL namespaces, references are automatic. For NuGet packages, you still need `<PackageReference>`.

---

## Performance

Zero runtime cost. Usings are compile-time only. Larger / more usings = slightly longer compile time (rarely measurable).

---

## Summary

- `<ImplicitUsings>enable</ImplicitUsings>` — SDK adds default usings (System, Linq, etc.).
- `global using ...;` — declare a using once, applies to all files.
- Custom implicits in csproj via `<Using Include="..." />`.
- Combined with `namespace MyApp;` (file-scoped namespace), files are concise.
- Centralize in `Directory.Build.props` for repo-wide consistency.
- Trade-off: less boilerplate, slightly more implicit context to read.

→ Next: [08-CollectionExpressions.md](08-CollectionExpressions.md)
