# What is C#?

## What it is

C# (pronounced "see sharp") is a general-purpose, statically-typed, object-oriented, multi-paradigm programming language designed by Anders Hejlsberg at Microsoft. The first version shipped in **2002** alongside the .NET Framework. The latest stable version, **C# 14**, ships with **.NET 10** (November 2025, LTS).

C# combines:
- **Static typing** with strong **type inference** — the compiler checks your types, but you rarely have to type them out.
- **Object orientation** as the default paradigm — everything is a class, struct, record, or interface.
- **Functional features** — lambdas, LINQ, pattern matching, records with value equality.
- **Memory safety by default** — garbage collection, no buffer overruns, no use-after-free. You can opt into unsafe pointer code where you need it.
- **First-class async** — `async`/`await` baked into the language since 2012 (C# 5), continuously refined.

It compiles to **IL** (Intermediate Language), a portable bytecode that runs on the **CLR** (Common Language Runtime). The CLR JIT-compiles IL to native machine code at runtime — or you can opt for **Native AOT** to compile directly to a self-contained native binary.

---

## Why it exists

When C# was conceived in the late 1990s, the dominant systems language was C++ — powerful but unforgiving — and the dominant managed language was Java, which Microsoft was unhappy with (literal and legal reasons). Microsoft wanted a language that:

1. Felt familiar to C/C++/Java developers.
2. Took the best ideas from Delphi (Hejlsberg's previous work) and added what C++/Java got wrong.
3. Ran on a managed runtime with a quality garbage collector.
4. Could grow over time without breaking compatibility.

That last point matters: C# has gone through 14 major versions and **every release is backward-compatible**. Code you wrote in 2002 compiles today (with deprecation warnings here and there). Few mainstream languages can claim that.

---

## What you can build with it

C# is comfortable in almost every domain:

| Domain | Stack |
|---|---|
| Web APIs / sites | ASP.NET Core, Minimal APIs, Blazor |
| Desktop apps | WPF, WinUI 3, Avalonia (cross-platform), .NET MAUI |
| Mobile apps | .NET MAUI (iOS, Android) |
| Game development | Unity (C# as scripting), Godot (.NET version), Stride |
| Cloud / microservices | ASP.NET Core, Azure Functions, AWS Lambda (.NET runtime) |
| Data + ML | ML.NET, TensorFlow.NET, ONNX Runtime |
| CLI tools | `dotnet tool`, System.CommandLine, native AOT for fast startup |
| Embedded / IoT | nanoFramework, .NET IoT libraries |
| Backend services | gRPC, SignalR, EF Core |

It's NOT the typical choice for:
- Systems programming where you need to control every byte (use Rust or C++).
- Quick scripting (Python or PowerShell are faster to spin up — though C# 14 file-based apps narrow this gap).
- Browser-side rendering (Blazor WebAssembly works, but JavaScript/TypeScript still dominate).

---

## Where C# sits in the landscape

A quick comparison with neighbors:

| | C# | Java | Python | JavaScript / TS | C++ | Go | Rust |
|---|---|---|---|---|---|---|---|
| Static types | yes | yes | optional | optional (TS yes) | yes | yes | yes |
| GC | yes | yes | yes | yes | no | yes | no (ownership) |
| Native compile | optional (AOT) | optional (GraalVM) | no | no | yes | yes | yes |
| async/await | yes (excellent) | virtual threads (21+) | yes | yes | coroutines (20+) | goroutines | yes |
| Pattern matching | excellent | improving (21+) | basic | no | basic | no | excellent |
| Generics | excellent (variance, constraints) | type erasure | duck-typed | generics (TS only) | templates | (since 1.18) | excellent |
| Records / data classes | yes (C# 9+) | yes (16+) | dataclass | TS interfaces | structs | structs | structs |
| Module system | namespaces + assemblies | packages + modules | imports | ES modules | headers | packages | modules |

C#'s sweet spot: large-scale backend and desktop applications where you want strong typing, fast iteration, and excellent runtime performance without writing C++.

---

## The C# language philosophy

The C# language design team — led by Mads Torgersen, with contributions from a public proposal repository (`github.com/dotnet/csharplang`) — operates under a few rules:

1. **Backwards compatibility is non-negotiable.** New keywords are usually *contextual* (e.g., `field` is a keyword only inside property accessors). Existing code keeps compiling.
2. **Pay-for-play.** Features cost nothing if you don't use them. NRT (`?`-marked references) only kicks in if you turn on the warning context.
3. **Pit of success.** Make the default easy and correct. `string` is non-null by default; you must opt into nullable.
4. **Empower performance-conscious code without forcing it.** `Span<T>`, `ref struct`, source generators are available when you need them.
5. **Listen to the community.** Proposals are public, discussed, and frequently rejected with detailed reasoning.

C# 14 is the latest expression of this philosophy: features like the `field` keyword, extension members, and `params ReadOnlySpan<T>` smooth common patterns without breaking anything.

---

## Quick tour: a feel for modern C#

A flavor of how C# looks in 2025, drawing on features from many versions:

```csharp
// Top-level statements (C# 9): no Main(), no class wrapper
using System;
using System.Linq;

// Records (C# 9): immutable, value equality
public record Employee(string Name, decimal Salary, string Department);

var team = new List<Employee> {
    new("Alice", 95_000m, "Engineering"),
    new("Bob",   72_000m, "Marketing"),
    new("Carol", 110_000m,"Engineering"),
};

// Pattern matching + switch expressions (C# 8+)
string Category(Employee e) => e switch {
    { Salary: > 100_000m } => "Senior",
    { Department: "Engineering", Salary: > 80_000m } => "Mid Engineer",
    _ => "Other"
};

// LINQ + records + interpolated raw strings (C# 11)
var summary = team
    .GroupBy(e => e.Department)
    .Select(g => $"""
        Department: {g.Key}
        Members:    {g.Count()}
        Total pay:  {g.Sum(e => e.Salary):C0}
        """);

foreach (var line in summary) Console.WriteLine(line);
```

If parts of that feel mysterious — `record`, `switch` expressions, raw strings, top-level statements — they'll all be explained in detail in the chapters ahead.

---

## A short history of the language

| Year | Version | .NET version | Headline features |
|---|---|---|---|
| 2002 | C# 1.0 | .NET Framework 1.0 | Initial release |
| 2005 | C# 2.0 | .NET 2.0 | Generics, nullable value types, iterators, anonymous methods |
| 2007 | C# 3.0 | .NET 3.5 | LINQ, lambdas, extension methods, anonymous types, `var` |
| 2010 | C# 4.0 | .NET 4.0 | `dynamic`, named/optional args, COM improvements |
| 2012 | C# 5.0 | .NET 4.5 | `async` / `await` |
| 2015 | C# 6.0 | .NET 4.6 | String interpolation, `?.`, expression-bodied members, `nameof` |
| 2017 | C# 7.0-7.3 | .NET Core 2.x | Tuples, pattern matching v1, `out var`, local functions, `Span<T>` |
| 2019 | C# 8.0 | .NET Core 3.x | Nullable reference types, default interface methods, async streams, `using` declarations |
| 2020 | C# 9.0 | .NET 5 | Records, init-only setters, top-level statements, target-typed `new()`, source generators |
| 2021 | C# 10.0 | .NET 6 (LTS) | Global usings, file-scoped namespaces, record struct, parameterless struct ctors |
| 2022 | C# 11.0 | .NET 7 | Required members, raw strings, generic math, list patterns, UTF-8 literals |
| 2023 | C# 12.0 | .NET 8 (LTS) | Primary constructors, collection expressions, alias any type |
| 2024 | C# 13.0 | .NET 9 | `params` collections, partial properties, `lock` type, `\e` |
| **2025** | **C# 14.0** | **.NET 10 (LTS)** | **Extension members, `field` keyword, `?.=`, partial ctors/events, file-based apps** |

We're going to cover every feature in detail. Chapter 11 has a version-by-version reference if you want the chronological view.

---

## What you'll read in this book

In rough order:
- **Chapters 0–2**: setup, syntax, and the OO model. Beginner level.
- **Chapters 3–7**: deep type system, generics, delegates, LINQ, collections. Intermediate.
- **Chapters 8–10**: concurrency, memory/performance, advanced language features. Advanced.
- **Chapters 11–14**: modern features (version-by-version), reflection, I/O, native interop. Expert.
- **Chapters 15–17**: tooling, testing, best practices. Mastery.

Each chapter has a `Questions.md` for drilling and a `Coding.md` with full-solution problems. The goal is for you to put this book down and **know** C# — not just have read about it.

→ Next: [02-DotNetEcosystem.md](02-DotNetEcosystem.md)
