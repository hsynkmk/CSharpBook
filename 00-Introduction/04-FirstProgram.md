# Your First C# Program

## What we'll build

Three different "Hello, World" programs, in order of how C# evolved:

1. **Classic Main()** — what every C# program looked like before 2020.
2. **Top-level statements** — what `dotnet new console` produces now (C# 9+).
3. **File-based apps** — single-file `app.cs` you run directly (C# 14+).

By the end, you'll know exactly what's going on under each style.

---

## 1. Classic Main()

The traditional form. Until C# 9, every program needed it. You'll still see it in legacy code:

`Program.cs`:
```csharp
using System;

namespace HelloApp
{
    public class Program
    {
        public static void Main(string[] args)
        {
            Console.WriteLine("Hello, World!");
        }
    }
}
```

Let's decode every piece:

- **`using System;`** — imports the `System` namespace. Without it you'd have to write `System.Console.WriteLine`. (In modern projects with `<ImplicitUsings>enable</ImplicitUsings>`, the `System` namespace is already imported for you — but you can still write the using explicitly.)
- **`namespace HelloApp`** — a logical grouping. Stops naming collisions across libraries.
- **`public class Program`** — declares a class. `public` makes it accessible to other assemblies; you could also use `internal` (the default).
- **`public static void Main(string[] args)`** — the entry point.
  - `public` — accessibility.
  - `static` — doesn't need an instance to call.
  - `void` — returns nothing (you can also return `int` for an exit code, or `Task` / `Task<int>` for `async Main`).
  - `Main` — the name the runtime looks for.
  - `string[] args` — command-line arguments. Optional; you can also declare `Main()` with no parameters.
- **`Console.WriteLine("Hello, World!");`** — calls `WriteLine` on the static `Console` class. Statements end with `;`.

To run:
```bash
dotnet new console --use-program-main
dotnet run
```

`--use-program-main` tells the template to generate the classic style.

### Valid Main signatures

The runtime accepts any of these:
```csharp
static void Main()
static void Main(string[] args)
static int Main()
static int Main(string[] args)
static async Task Main()                  // C# 7.1+
static async Task Main(string[] args)
static async Task<int> Main()
static async Task<int> Main(string[] args)
```

Returning `int` lets you signal exit codes to the OS (0 = success by convention).

---

## 2. Top-level statements (C# 9+)

What `dotnet new console` produces today. The class and Main are still there — the compiler generates them — but you don't write them.

`Program.cs`:
```csharp
Console.WriteLine("Hello, World!");
```

Yes, that's the whole file. Where did everything go?

### What the compiler does

Behind the scenes, the compiler wraps your top-level code in a synthetic class and Main method. Roughly equivalent to:

```csharp
internal static class Program
{
    static async Task<int> <Main>$(string[] args)
    {
        Console.WriteLine("Hello, World!");
        return 0;
    }
}
```

(The actual generated name `<Main>$` is illegal as a C# identifier — it's there to avoid clashing with your code.)

### Rules

- **Only one file per project** can have top-level statements. Other files use ordinary classes.
- **Order matters**: declarations of helper classes/methods after the top-level code is fine, but they must come **after** the executable statements in the same file, OR be in a different file:

  ```csharp
  Greet("World");                   // top-level statements first
  Greet("Friend");

  void Greet(string name) =>        // local function (or could be a class) after
      Console.WriteLine($"Hi, {name}!");
  ```
- **`args` is implicit** — it's a `string[]` in scope without declaring it.
- **`return` is allowed** to exit early or return an int exit code.
- **`await` is allowed** at the top level (the compiler generates `async Task<int> Main`).

### Why this exists

Top-level statements lower the barrier for beginners and small scripts. Teaching C# in 2025 means you can show someone:

```csharp
Console.WriteLine("Hello, World!");
```

…and that's a complete, working program. No "explanation deferred until later" about classes, static, void, or string[].

For larger projects, the ceremony returns once you add more files (each of those is a normal class).

---

## 3. File-based apps (C# 14, new in November 2025)

You can now run a `.cs` file **without a `.csproj`**:

`hello.cs`:
```csharp
Console.WriteLine("Hello, World!");
```

Run it:
```bash
dotnet run hello.cs
```

Output: `Hello, World!`.

That's it. No project, no restore, no folder structure.

### How does this work?

Behind the scenes the SDK silently creates a temporary project, restores, builds, and runs your file. The first run is slower (compilation); subsequent runs are cached.

### Adding dependencies

You can reference NuGet packages right inside the file using a special comment directive:

```csharp
#:package Serilog@4.1.0
using Serilog;

Log.Logger = new LoggerConfiguration().WriteTo.Console().CreateLogger();
Log.Information("Hello from a one-file app!");
```

The `#:package` directive tells the SDK to add that NuGet reference. Other directives:

```csharp
#:sdk Microsoft.NET.Sdk          // change the SDK
#:property TargetFramework=net10.0  // set MSBuild properties
```

### When to use file-based apps

- Quick experiments — testing a language feature, trying a NuGet package.
- Educational examples — like the snippets in this book.
- Small scripts that grew out of "let me just check…".
- Anywhere Python or Bash would be reached for.

### When NOT to use them

- Real applications with multiple files, configuration, or test suites — switch to a real project: `dotnet new console`.
- Anything you want to publish as a NuGet package or executable.
- Production code. File-based apps are for ergonomics, not deployment.

### Converting to a project

When your file-based app outgrows itself:
```bash
dotnet project convert hello.cs
```

This generates a `.csproj` alongside the file.

---

## What's behind `Console.WriteLine`?

`Console` is a static class in `System` (BCL). `WriteLine` is one overload of many:

```csharp
public static void WriteLine();
public static void WriteLine(string? value);
public static void WriteLine(int value);
public static void WriteLine(object? value);
public static void WriteLine(string format, params object?[] arg);
public static void WriteLine(string format, params ReadOnlySpan<object?> arg);  // C# 14
// ... and more
```

The compiler picks the overload that matches your arguments. For:
```csharp
Console.WriteLine("Hello");        // → WriteLine(string?)
Console.WriteLine(42);             // → WriteLine(int)
Console.WriteLine($"x={x}");       // → WriteLine(string?) after interpolation
```

Internally, `Console.WriteLine` writes to **stdout** (file descriptor 1) using the encoding configured for the console (`Console.OutputEncoding`).

You'll see `Console` everywhere in this book — it's the default I/O channel for examples.

---

## Reading input

The mirror of `WriteLine` is `ReadLine`:

```csharp
Console.Write("What is your name? ");      // no newline
string? name = Console.ReadLine();          // returns string? (nullable — could be null on EOF)
Console.WriteLine($"Hello, {name ?? "stranger"}!");
```

Notes:
- `Write` doesn't append a newline; `WriteLine` does.
- `ReadLine` returns `string?` — could be `null` if stdin is closed. The `??` operator gives a fallback.

---

## String interpolation

You've seen `$"..."` already. It's how modern C# does string formatting:

```csharp
int n = 5;
Console.WriteLine($"There are {n} items.");
```

Beats:
```csharp
Console.WriteLine("There are " + n + " items.");
Console.WriteLine(string.Format("There are {0} items.", n));
```

You can format inside the braces:
```csharp
double price = 19.95;
Console.WriteLine($"Price: {price:F2}");      // → Price: 19.95
Console.WriteLine($"Date:  {DateTime.Now:yyyy-MM-dd}");
```

More on strings in [Chapter 01 §03](../01-Fundamentals/03-StringsAndChars.md).

---

## Comments

Three styles:

```csharp
// Single-line. Most common.

/* Multi-line.
   Spans lines. Used sparingly. */

/// <summary>
/// XML doc comment. Read by tooling for IntelliSense and doc generators.
/// </summary>
public void DoSomething() { }
```

[Chapter 01 §09](../01-Fundamentals/09-CommentsAndXmlDocs.md) covers documentation style.

---

## Putting it together: a slightly bigger example

```csharp
// Top-level statement style — modern C# default
using System;

Console.Write("Enter a number: ");
string? input = Console.ReadLine();

if (int.TryParse(input, out int n))
{
    Console.WriteLine($"Squared: {n * n}");
}
else
{
    Console.WriteLine("That's not a valid number.");
    return 1;            // exit with non-zero code
}
```

You don't yet know what `int.TryParse` does in detail (Chapter 01 covers it), but you can read the gist: ask for a number, parse it, square it, handle failure.

This is real C#. You can save it as `square.cs` and run `dotnet run square.cs`.

---

## Summary

- Three valid ways to write your entry point: classic `Main`, top-level statements, file-based apps.
- All three compile to roughly the same thing — different syntactic doors into the same building.
- Top-level statements (C# 9+) are the modern default.
- File-based apps (C# 14, late 2025) let you skip projects entirely for quick scripts.
- Behind everything: a `static Main` method the runtime looks for.

→ Next: [05-ProjectStructure.md](05-ProjectStructure.md)
