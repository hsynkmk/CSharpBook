# Roslyn Analyzers

## What they are

Roslyn analyzers are compiler plugins that **inspect your code during compilation** and report diagnostics (warnings/errors/info) — and optionally provide automatic **code fixes**. They enforce correctness, style, performance, and security rules continuously, in the IDE and in CI.

```csharp
// Analyzer flags this:
string s = null;
int len = s.Length;        // CA1062 / CS8602: possible null dereference

// And CA1822 suggests:
public int Compute() => 42;   // → "mark as static" (doesn't use instance state)
```

They run on the same Roslyn syntax/semantic model the compiler uses — fast, accurate, and IDE-integrated (squiggles + lightbulb fixes).

---

## Where analyzers come from

1. **Built-in .NET analyzers** — ship with the SDK (`CAxxxx` rules from `Microsoft.CodeAnalysis.NetAnalyzers`). On by default in modern projects.
2. **Roslyn compiler diagnostics** — `CSxxxx` (language rules).
3. **NuGet analyzer packages** — e.g., `StyleCop.Analyzers`, `Roslynator`, `Meziantou.Analyzer`, `SonarAnalyzer.CSharp`, security analyzers.
4. **Custom analyzers** — ones you write for project-specific rules.

```xml
<ItemGroup>
  <PackageReference Include="Roslynator.Analyzers" Version="4.x" PrivateAssets="all" />
  <PackageReference Include="Meziantou.Analyzer" Version="2.x" PrivateAssets="all" />
</ItemGroup>
```

`PrivateAssets="all"` keeps the analyzer out of your runtime dependencies.

---

## Enabling and configuring built-in analyzers

```xml
<PropertyGroup>
  <EnableNETAnalyzers>true</EnableNETAnalyzers>       <!-- on by default for net5.0+ -->
  <AnalysisLevel>latest-recommended</AnalysisLevel>    <!-- or latest-all, latest-minimum -->
  <AnalysisMode>All</AnalysisMode>                     <!-- enable more rules -->
  <EnforceCodeStyleInBuild>true</EnforceCodeStyleInBuild>  <!-- run IDExxxx style rules at build -->
  <CodeAnalysisTreatWarningsAsErrors>true</CodeAnalysisTreatWarningsAsErrors>
</PropertyGroup>
```

`AnalysisLevel` / `AnalysisMode` control how many rules fire. `latest-recommended` is a sensible baseline; `All` is aggressive (expect noise).

---

## Severities and per-rule configuration

Each diagnostic has a severity: `error`, `warning`, `suggestion` (info), `silent`, or `none`. Configure per-rule in `.editorconfig`:

```ini
# .editorconfig
[*.cs]
# Treat a specific rule as an error
dotnet_diagnostic.CA2007.severity = error      # ConfigureAwait

# Downgrade a noisy rule
dotnet_diagnostic.CA1031.severity = none       # catch general exception

# Style rule severity
dotnet_diagnostic.IDE0090.severity = warning   # use 'new(...)'
```

`.editorconfig` is the modern way to configure analyzer severity (the old `.ruleset` files still work but are legacy). See [06-EditorConfig.md](06-EditorConfig.md).

---

## Common useful rules

| Rule | What it catches |
|---|---|
| CA1062 | Validate public method arguments for null |
| CA1822 | Member can be static (no instance state used) |
| CA2007 | Missing `ConfigureAwait` on awaited task (library code) |
| CA1849 | Call async method instead of blocking |
| CA1848 | Use `LoggerMessage` delegates (perf logging) |
| CA1854 | Prefer `TryGetValue` over double dictionary lookup |
| CA2016 | Forward `CancellationToken` to methods that take one |
| CA1304/1305/1310 | Specify culture / StringComparison |
| CA5394 | Insecure randomness |
| IDE0011 | Add braces |
| IDE0090 | Simplify `new` expression |

These encode hard-won best practices (many covered in [Chapter 17](../17-BestPractices/README.md)).

---

## Suppressing diagnostics

When a rule genuinely doesn't apply:

```csharp
// Inline (preferred — local and documented)
#pragma warning disable CA1031 // justification
try { ... } catch (Exception) { ... }
#pragma warning restore CA1031

// Attribute-based (for a whole member)
[SuppressMessage("Reliability", "CA2007:ConfigureAwait", Justification = "ASP.NET Core has no SyncContext")]
public async Task DoWork() { ... }
```

Or globally in `GlobalSuppressions.cs`. Always include a **justification** — silent suppression hides intent. Prefer fixing over suppressing.

---

## Treating warnings as errors

```xml
<PropertyGroup>
  <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
  <!-- or selectively -->
  <WarningsAsErrors>nullable;CA2007;CS8602</WarningsAsErrors>
  <!-- exclude specific ones -->
  <WarningsNotAsErrors>CA1031</WarningsNotAsErrors>
</PropertyGroup>
```

In CI, treating warnings as errors prevents diagnostic rot. Start with `WarningsAsErrors` for a curated set, then broaden once the codebase is clean.

---

## Writing a custom analyzer

An analyzer is a class deriving from `DiagnosticAnalyzer`, registered for specific syntax/symbol kinds.

```csharp
using Microsoft.CodeAnalysis;
using Microsoft.CodeAnalysis.CSharp;
using Microsoft.CodeAnalysis.CSharp.Syntax;
using Microsoft.CodeAnalysis.Diagnostics;
using System.Collections.Immutable;

[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class NoConsoleWriteLineAnalyzer : DiagnosticAnalyzer {
    private static readonly DiagnosticDescriptor Rule = new(
        id: "MY0001",
        title: "Avoid Console.WriteLine",
        messageFormat: "Use ILogger instead of Console.WriteLine",
        category: "Usage",
        defaultSeverity: DiagnosticSeverity.Warning,
        isEnabledByDefault: true);

    public override ImmutableArray<DiagnosticDescriptor> SupportedDiagnostics => [Rule];

    public override void Initialize(AnalysisContext context) {
        context.ConfigureGeneratedCodeAnalysis(GeneratedCodeAnalysisFlags.None);
        context.EnableConcurrentExecution();
        context.RegisterSyntaxNodeAction(AnalyzeInvocation, SyntaxKind.InvocationExpression);
    }

    private static void AnalyzeInvocation(SyntaxNodeAnalysisContext ctx) {
        var invocation = (InvocationExpressionSyntax)ctx.Node;
        if (invocation.Expression is MemberAccessExpressionSyntax m &&
            m.Name.Identifier.Text == "WriteLine" &&
            m.Expression is IdentifierNameSyntax id && id.Identifier.Text == "Console") {
            ctx.ReportDiagnostic(Diagnostic.Create(Rule, invocation.GetLocation()));
        }
    }
}
```

Key points:
- `[DiagnosticAnalyzer(LanguageNames.CSharp)]`.
- `Initialize` registers callbacks (`RegisterSyntaxNodeAction`, `RegisterSymbolAction`, `RegisterOperationAction`, etc.).
- `EnableConcurrentExecution` + `ConfigureGeneratedCodeAnalysis` for performance/correctness.
- Use the **semantic model** (`ctx.SemanticModel`) for accurate symbol resolution rather than relying on names alone (the example above is simplified — a real analyzer checks the symbol is `System.Console.WriteLine`).

Analyzers target `netstandard2.0` and run in the compiler — like source generators (see [Chapter 12 §05](../12-Reflection/05-SourceGenerators.md)).

---

## Code fix providers

Pair an analyzer with a `CodeFixProvider` to offer a lightbulb auto-fix:

```csharp
[ExportCodeFixProvider(LanguageNames.CSharp)]
public class UseLoggerCodeFix : CodeFixProvider {
    public override ImmutableArray<string> FixableDiagnosticIds => ["MY0001"];
    public override FixAllProvider GetFixAllProvider() => WellKnownFixAllProviders.BatchFixer;

    public override async Task RegisterCodeFixesAsync(CodeFixContext context) {
        // Register an action that rewrites the syntax tree to replace Console.WriteLine
        context.RegisterCodeFix(
            CodeAction.Create("Use ILogger", ct => ReplaceAsync(context, ct)),
            context.Diagnostics);
    }
    // ReplaceAsync produces a new Document with the rewritten node
}
```

The fix produces a transformed `Document`; the IDE applies it on click (and `dotnet format analyzers` can apply fixes in bulk).

---

## Running analyzers from the CLI

```bash
dotnet build                          # analyzers run during build
dotnet format analyzers               # apply analyzer code fixes
dotnet format analyzers --verify-no-changes   # CI: fail if fixes pending
```

---

## Common bugs / gotchas

### Analyzer noise overwhelms the team

Turning on `AnalysisMode=All` floods the build with warnings, and people start ignoring all of them. Start with `latest-recommended`, curate, then ratchet up. Use `.editorconfig` to silence rules that don't fit.

### Suppressing without justification

`#pragma warning disable` with no comment hides intent and accumulates. Always justify; prefer fixing.

### Custom analyzer relying on names not symbols

Matching `"WriteLine"` by text flags `myObject.WriteLine()` too. Use the semantic model to confirm the actual symbol (`System.Console.WriteLine`).

### Analyzer slows the build/IDE

Heavy or non-concurrent analyzers degrade the editing experience. Register the narrowest actions, enable concurrency, and skip generated code.

---

## Summary

- Analyzers inspect code at compile time and report diagnostics (with optional code fixes), in IDE and CI.
- Built-in `CAxxxx`/`IDExxxx` rules ship with the SDK; add `StyleCop`/`Roslynator`/`Meziantou`/`Sonar` for more.
- Configure severity per-rule in `.editorconfig`; treat curated warnings as errors in CI.
- Suppress sparingly with justification; prefer fixing.
- Write custom analyzers (`DiagnosticAnalyzer`) + code fixes (`CodeFixProvider`) for project-specific rules; use the semantic model, not names.
- `dotnet format analyzers` applies fixes; `--verify-no-changes` gates CI.

→ Next: [06-EditorConfig.md](06-EditorConfig.md)
