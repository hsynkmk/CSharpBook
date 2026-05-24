# Top-Level Statements (C# 9+)

## What it is

C# 9 (2020) introduced **top-level statements** — you can write statements directly at the top of a `.cs` file, without wrapping them in a `Main` method or class. The compiler generates the boilerplate.

```csharp
// Program.cs
using System;

Console.WriteLine("Hello, World!");
```

That's the whole program. No `class Program`, no `static void Main`. The compiler synthesizes them.

`dotnet new console` produces this style by default since .NET 6. It's the modern entry-point form.

---

## Why it exists

For teaching C#: showing newcomers `class Program { static void Main(string[] args) { Console.WriteLine("..."); } }` is a lot of ceremony before the first line of "real" code.

Top-level statements:

```csharp
Console.WriteLine("Hello, World!");
```

…lets you start with the thing you actually want to teach. Same for scripts, tools, small demos.

For larger projects, you still organize code into classes. Top-level statements give the entry point cleaner syntax.

---

## Rules

### One file per project

Only **one file** in the project can have top-level statements. The compiler uses it as the entry point.

If you try to have two:
```
error CS8802: Only one compilation unit can have top-level statements.
```

### Statements first, declarations after

```csharp
Console.WriteLine("Hello");          // top-level statement
int n = 5;                            // top-level statement
SomeMethod();                          // calls method declared below

void SomeMethod() => Console.WriteLine("invoked");   // local function declaration

class Helper { ... }                   // type declarations after the statements
```

The order matters. Statements come first; declarations (local functions, types) come after.

### `args` is implicit

```csharp
Console.WriteLine($"Got {args.Length} args");
foreach (var a in args) Console.WriteLine(a);
```

`args` is a `string[]` implicitly available, equivalent to the `string[] args` parameter of a traditional Main.

### `return` for exit code

```csharp
if (failed) return 1;
return 0;
```

The implicit Main has a return type of `int` if you use `return <int>`. Otherwise `void`.

### `await` is allowed

```csharp
using System.Net.Http;
using var client = new HttpClient();
var response = await client.GetStringAsync("https://example.com");
Console.WriteLine(response);
```

The compiler generates an `async Task<int> Main()` (or `Task` if no return). You can `await` at the top level.

---

## Generated form

For top-level statements:

```csharp
Console.WriteLine("hi");
int n = 5;
Console.WriteLine(n);
return 0;
```

The compiler generates:

```csharp
internal static class Program {
    private static int <Main>$(string[] args) {
        Console.WriteLine("hi");
        int n = 5;
        Console.WriteLine(n);
        return 0;
    }
}
```

The class is named `Program` (you can't add another `Program` class — it'd conflict). The method is named `<Main>$` (illegal in C# source). The `string[] args` parameter is exposed as `args`.

For async:

```csharp
await Task.Delay(100);
return 0;
```

```csharp
internal static class Program {
    private static async Task<int> <Main>$(string[] args) {
        await Task.Delay(100);
        return 0;
    }
}
```

---

## Co-existing with class definitions

```csharp
using System;

Console.WriteLine("From top level");
var d = new Demo();
d.Run();

public class Demo {
    public void Run() => Console.WriteLine("From class");
}
```

Top-level statements + a class in the same file is allowed. Types declared after the top-level code are normal types — accessible from the rest of the project.

If you mix: top-level statements use the file's own classes; the classes can also be referenced from other files normally.

---

## When to use top-level statements

✓ Small console apps, scripts, demos.
✓ The main entry point of an ASP.NET Core / Worker / Console app (modern default).
✓ Tools (`dotnet tool` global commands).
✓ Education / learning materials.

✗ Library projects (no entry point needed; top-level statements not relevant).
✗ Code that fits naturally in classes (most real applications, beyond Main).

For Main, top-level is shorter. For everything else, classes.

---

## ASP.NET Core minimal hosting

The biggest impact: ASP.NET Core's Minimal APIs use top-level statements:

```csharp
var builder = WebApplication.CreateBuilder(args);
builder.Services.AddSingleton<MyService>();
var app = builder.Build();

app.MapGet("/", () => "Hello, World!");
app.MapGet("/users/{id}", (int id, MyService svc) => svc.GetUser(id));

app.Run();
```

A complete web API in ~10 lines. The boilerplate of Program.cs + Startup.cs (old style) is gone.

For larger apps, you'd extract endpoint definitions to separate files / classes — but the entry point stays compact.

---

## File-based apps (.NET 10+, file extension `.cs`)

.NET 10 + C# 14 introduced `dotnet run file.cs` — run a single C# file as a complete program without a .csproj.

```csharp
// hello.cs
#:package Newtonsoft.Json@13.0.0
using Newtonsoft.Json;
Console.WriteLine(JsonConvert.SerializeObject(new { msg = "hi" }));
```

```bash
$ dotnet run hello.cs
{"msg":"hi"}
```

File-based apps combine top-level statements + package directives + automatic project synthesis. Great for scripts.

For multi-file projects, you still want a .csproj. For single-file utilities, file-based is the simplest.

---

## Internals — what's emitted

The generated `Program` class is just a regular class. Decompiling shows:

```csharp
internal static class Program {
    private static void <Main>$(string[] args) {
        // your top-level statements here
    }
}
```

The `args` parameter is bound to whatever was passed on the command line. The compiler-synthesized method is the entry point (the assembly's `[STAThread]` attribute target).

If you want to test the entry-point logic without launching the process, refactor into a public method that the synthesized Main calls. Top-level statements aren't directly testable as-is.

---

## Common bugs

### Multiple files with top-level statements

```csharp
// File A
Console.WriteLine("A");

// File B
Console.WriteLine("B");
```

Compile error: only one file can have top-level statements per project.

### Mixed with `class Program { static void Main(...) }`

```csharp
// Top-level
Console.WriteLine("top");

class Program {
    static void Main(string[] args) { Console.WriteLine("class main"); }
}
```

Compile error: top-level statements imply a synthesized Program/Main; you can't also declare your own.

### Trying to write functions before statements

```csharp
void Helper() => Console.WriteLine("hi");   // ⚠ — local functions go AFTER statements
Helper();
```

Wrong order. Statements first, then declarations:

```csharp
Helper();
void Helper() => Console.WriteLine("hi");
```

### Forgetting `await` in async top-level

```csharp
SomethingAsync();   // ⚠ — task discarded; method may not finish before Main returns
```

Always `await` top-level async calls.

---

## Performance

Zero overhead. The synthesized Program is just a regular class. Top-level vs classic Main: identical IL semantics (modulo the synthesized name).

---

## When NOT to use

- Library projects — no entry point needed.
- Apps where the entry-point logic deserves its own file in a `Services/` folder.

For Main-like entry points, top-level is the modern default. For everything else, prefer class organization.

---

## Summary

- Top-level statements let you write a program's main code without `class Program { static void Main }`.
- One file per project can have them.
- Statements first, declarations after.
- `args` is implicit; `return <int>` for exit codes; `await` works.
- Default for `dotnet new console` since .NET 6.
- ASP.NET Core Minimal APIs are built on this.
- File-based apps (C# 14 + .NET 10) extend the idea further.

→ Next: [07-GlobalAndImplicitUsings.md](07-GlobalAndImplicitUsings.md)
