# Source Generators

## What they are

Source generators are **compile-time code-emitting plugins** to the Roslyn compiler. They inspect your code (and other inputs) during compilation and **add new source files** to the compilation. The generated code compiles alongside yours.

```csharp
// You write:
[GeneratedRegex(@"^\d{3}-\d{4}$")]
public static partial Regex PhonePattern();

// A source generator emits the implementation at compile time:
public static partial Regex PhonePattern() => _phoneRegex;
private static readonly Regex _phoneRegex = new(@"^\d{3}-\d{4}$", ...);
```

Source generators are the modern replacement for **runtime reflection** in performance-critical and AOT scenarios. They move work from runtime to compile time.

Used by:
- `[GeneratedRegex]` — compiled regex without runtime IL emit.
- `System.Text.Json` source-gen — serialize without reflection.
- `[LibraryImport]` — P/Invoke marshalling code (replaces `DllImport`).
- `LoggerMessage` source-gen — high-perf logging.
- MVVM Toolkit — `[ObservableProperty]` generates INPC boilerplate.

---

## Why they exist

Reflection at runtime is:
- **Slow** — member lookup, boxing, no inlining.
- **AOT-hostile** — the trimmer can't see reflective usage.
- **Startup-heavy** — JIT compiles reflection expressions on first use.

Source generators do the equivalent work at **compile time**, emitting direct code. Result: zero runtime reflection, full AOT compatibility, faster startup.

```
Reflection model:  [Runtime] inspect type → build delegate → invoke (slow, AOT-unsafe)
SourceGen model:   [Compile] inspect type → emit direct code → (fast, AOT-safe)
```

---

## The two generator interfaces

### `IIncrementalGenerator` (modern, preferred)

```csharp
[Generator]
public class HelloGenerator : IIncrementalGenerator {
    public void Initialize(IncrementalGeneratorInitializationContext context) {
        context.RegisterPostInitializationOutput(ctx => {
            ctx.AddSource("Hello.g.cs", """
                namespace Generated;
                public static class Hello {
                    public static string Greeting => "Hello from generated code!";
                }
                """);
        });
    }
}
```

**Incremental** generators cache intermediate results and only re-run the parts that changed. Essential for IDE performance — they run on every keystroke.

### `ISourceGenerator` (legacy, deprecated)

The original v1 API (`Execute`/`Initialize`). Re-runs fully on every change — slow in IDE. **Don't write new ones** with this API.

---

## The incremental pipeline

The core pattern is a LINQ-like pipeline over compilation inputs:

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context) {
    // 1. Find syntax nodes of interest (fast, runs constantly)
    IncrementalValuesProvider<ClassDeclarationSyntax> classes = context.SyntaxProvider
        .CreateSyntaxProvider(
            predicate: static (node, _) => node is ClassDeclarationSyntax c && c.AttributeLists.Count > 0,
            transform: static (ctx, _) => (ClassDeclarationSyntax)ctx.Node)
        .Where(static c => c is not null);

    // 2. Combine with compilation (semantic model)
    var compilationAndClasses = context.CompilationProvider.Combine(classes.Collect());

    // 3. Generate
    context.RegisterSourceOutput(compilationAndClasses, static (spc, source) => {
        var (compilation, classList) = source;
        foreach (var cls in classList) {
            // emit code per class
            spc.AddSource($"{cls.Identifier}.g.cs", GenerateFor(cls));
        }
    });
}
```

The two-stage `predicate` (cheap syntactic filter) + `transform` (semantic extraction) is key: the predicate runs on every node fast, and the expensive semantic work runs only on filtered candidates.

**Caching rule**: each stage's output should be an **equatable value type** (record or struct). Roslyn compares outputs; if unchanged, downstream stages are skipped. Never pass `ISymbol` / `SyntaxNode` through the cached pipeline — they break caching and hold compilations alive. Extract the data you need into a record.

---

## A complete example: generating a `ToString`

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class AutoToStringAttribute : Attribute {}

[Generator]
public class ToStringGenerator : IIncrementalGenerator {
    public void Initialize(IncrementalGeneratorInitializationContext context) {
        var classes = context.SyntaxProvider.ForAttributeWithMetadataName(
            "MyLib.AutoToStringAttribute",
            predicate: static (_, _) => true,
            transform: static (ctx, _) => {
                var symbol = (INamedTypeSymbol)ctx.TargetSymbol;
                var props = symbol.GetMembers()
                    .OfType<IPropertySymbol>()
                    .Where(p => p.DeclaredAccessibility == Accessibility.Public)
                    .Select(p => p.Name)
                    .ToArray();
                // Return an equatable record, NOT the symbol
                return new Model(symbol.ContainingNamespace.ToDisplayString(), symbol.Name, props);
            });

        context.RegisterSourceOutput(classes, static (spc, model) => {
            var body = string.Join(" + \", \" + ", model.Props.Select(p => $"\"{p}=\" + {p}"));
            spc.AddSource($"{model.Name}.ToString.g.cs", $$"""
                namespace {{model.Namespace}};
                partial class {{model.Name}} {
                    public override string ToString() => "{{model.Name}} { " + {{body}} + " }";
                }
                """);
        });
    }

    private record Model(string Namespace, string Name, string[] Props);
}
```

`ForAttributeWithMetadataName` (added in .NET 7-era Roslyn) is the fast path for attribute-driven generators — far cheaper than scanning all syntax.

Note: the `Model` record uses `string[]` which doesn't have value equality by default. In real generators wrap arrays in an equatable collection (e.g., `ImmutableArray<T>` with a custom comparer) for proper caching.

---

## Built-in generators worth knowing

### `[GeneratedRegex]`

```csharp
public partial class Validators {
    [GeneratedRegex(@"^[\w.]+@[\w.]+$", RegexOptions.IgnoreCase)]
    public static partial Regex Email();
}

bool ok = Validators.Email().IsMatch("a@b.com");
```

Generates a fully compiled regex at build time — faster than `new Regex(...)` and AOT-safe.

### System.Text.Json source generation

```csharp
[JsonSerializable(typeof(Person))]
public partial class AppJsonContext : JsonSerializerContext {}

// Usage — no reflection, AOT-safe
string json = JsonSerializer.Serialize(person, AppJsonContext.Default.Person);
var p = JsonSerializer.Deserialize(json, AppJsonContext.Default.Person);
```

The generator emits serialization/deserialization code per type. Faster startup, AOT-compatible, trimming-safe.

### `LoggerMessage`

```csharp
public static partial class Log {
    [LoggerMessage(Level = LogLevel.Information, Message = "User {UserId} logged in")]
    public static partial void UserLoggedIn(ILogger logger, int userId);
}

Log.UserLoggedIn(logger, 42);
```

Generates an allocation-free, structured logging method. No boxing of `userId`, no format-string parsing at runtime.

### `[LibraryImport]` (C# 11)

```csharp
public static partial class Native {
    [LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBox(IntPtr hWnd, string text, string caption, uint type);
}
```

Generates marshalling code at compile time (vs `DllImport`'s runtime marshalling stubs). AOT-safe. See [Chapter 14 §01](../14-InteropAOT/01-PInvoke.md).

---

## Project setup for writing a generator

```xml
<!-- Generator project -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>netstandard2.0</TargetFramework>   <!-- generators target netstandard2.0 -->
    <EnforceExtendedAnalyzerRules>true</EnforceExtendedAnalyzerRules>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.CodeAnalysis.CSharp" Version="4.x" PrivateAssets="all" />
  </ItemGroup>
</Project>

<!-- Consuming project references the generator as an analyzer -->
<ItemGroup>
  <ProjectReference Include="..\MyGenerator\MyGenerator.csproj"
                    OutputItemType="Analyzer"
                    ReferenceOutputAssembly="false" />
</ItemGroup>
```

Generators **must** target `netstandard2.0` (the Roslyn host runtime). They run inside the compiler process.

---

## Debugging generators

```csharp
public void Initialize(IncrementalGeneratorInitializationContext context) {
#if DEBUG
    // if (!Debugger.IsAttached) Debugger.Launch();   // attach a debugger
#endif
    ...
}
```

Inspect generated output:
```xml
<PropertyGroup>
  <EmitCompilerGeneratedFiles>true</EmitCompilerGeneratedFiles>
  <CompilerGeneratedFilesOutputPath>generated</CompilerGeneratedFilesOutputPath>
</PropertyGroup>
```

Generated `.g.cs` files appear under `obj/.../generated/`. In VS, they show under Dependencies → Analyzers.

---

## Common patterns

### Marker attribute + partial class

The canonical shape: user marks a `partial` type with your attribute, you emit the other half.

```csharp
// User writes:
[AutoNotify]
public partial class ViewModel {
    private string _name = "";
}

// Generator emits the property with INotifyPropertyChanged.
```

### Emit a registration table

DI frameworks generate a switch over known types instead of reflecting at startup.

### Diagnostics

```csharp
context.ReportDiagnostic(Diagnostic.Create(
    new DiagnosticDescriptor("MYGEN001", "Invalid usage", "Class must be partial",
        "Usage", DiagnosticSeverity.Error, true),
    location));
```

Generators can emit compiler errors/warnings — guide users to correct usage.

---

## Common bugs

### Breaking incrementality

```csharp
// ⚠ — passing ISymbol/SyntaxNode through pipeline holds compilation alive, breaks caching
.Select((ctx, _) => ctx.TargetSymbol)   // bad
```

Extract data into equatable records inside the `transform`. Never let symbols/syntax flow downstream.

### Forgetting `partial`

The user's type must be `partial` for the generator to add members. If not, emit a diagnostic.

### Non-deterministic output

Generated code must be stable — same input → identical output. Don't use `DateTime.Now`, random GUIDs, or unordered dictionary iteration. Breaks build caching and reproducibility.

### Targeting wrong framework

Generators not targeting `netstandard2.0` won't load in the compiler. You'll see "generator failed to load."

---

## Source generators vs reflection vs Expression.Compile

| Aspect | Reflection | Expression.Compile | Source Generator |
|---|---|---|---|
| When | Runtime | Runtime (cached) | Compile time |
| Per-call cost | ~500 ns | ~5 ns (after compile) | < 1 ns (direct code) |
| Startup cost | Low | High (compile) | None |
| AOT-safe | ✗ | ✗ (needs JIT) | ✓ |
| Trimming-safe | ✗ (annotate) | ✗ | ✓ |
| Debuggable | hard | hard | ✓ (real source) |
| Complexity to write | low | medium | high |

For libraries shipping to AOT/trimmed consumers, source generators are the gold standard.

---

## When to use source generators

- Replacing runtime reflection in hot paths.
- Enabling AOT/trimming for serialization, DI, logging.
- Eliminating boilerplate (INPC, equality, builders).
- Library authors who control consumer codegen.

When **not** to:
- One-off scripts or prototypes (reflection is simpler).
- Logic that genuinely needs runtime types (plugins loaded from disk).
- When the boilerplate is trivial and a generator's complexity isn't justified.

---

## Summary

- Source generators emit C# at compile time via Roslyn.
- Use `IIncrementalGenerator` — cache-friendly, IDE-fast.
- Pipeline: cheap syntactic predicate → semantic transform → emit. Pass equatable records, never symbols.
- Built-ins: `[GeneratedRegex]`, STJ source-gen, `LoggerMessage`, `[LibraryImport]`.
- Replace runtime reflection for performance + AOT/trimming compatibility.
- Target `netstandard2.0`; output must be deterministic.

→ Next: [06-CompileTimeReflection.md](06-CompileTimeReflection.md)
