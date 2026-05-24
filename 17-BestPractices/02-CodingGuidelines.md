# Coding Guidelines

## The goal

Code is read far more often than it's written. Guidelines optimize for the reader: predictable structure, minimal surprise, and intent that's obvious without comments. The .NET ecosystem has well-established conventions (Microsoft's Framework Design Guidelines, the dotnet/runtime coding style); following them makes your code feel native.

These are defaults, not laws — deviate when there's a concrete reason, and do so consistently.

---

## File organization

- **One top-level type per file**, file named after the type (`OrderService.cs` → `class OrderService`).
- **File-scoped namespaces** (C# 10+) — less indentation:

```csharp
namespace MyCompany.Orders;   // file-scoped — no braces, whole file in this namespace

public class OrderService { ... }
```

- `using` directives **outside** the namespace, `System.*` first, then sorted. Prefer global usings + implicit usings for ubiquitous namespaces (see [Chapter 10 §07](../10-AdvancedLanguage/07-GlobalAndImplicitUsings.md)).

### Member ordering

A common, readable order within a type:

```csharp
public class Example {
    // 1. Constants and static readonly fields
    private const int MaxRetries = 3;

    // 2. Fields
    private readonly ILogger _logger;
    private int _state;

    // 3. Constructors
    public Example(ILogger logger) => _logger = logger;

    // 4. Properties
    public int Count { get; private set; }

    // 5. Public methods
    public void Process() { ... }

    // 6. Private methods
    private void Helper() { ... }

    // 7. Nested types
    private sealed class Node { ... }
}
```

Within each group, order by accessibility (public → private). Consistency matters more than the exact order — pick one and apply it.

---

## Use `var` judiciously

```csharp
var orders = new List<Order>();           // ✓ — type obvious from RHS
var customer = repository.GetById(id);     // ✓ — descriptive method name
var count = 5;                             // debatable — int is short

int count = 5;                             // ✓ — explicit primitive
var result = Process();                    // ⚠ — what type? unclear without IDE
Dictionary<string, List<int>> map = Get(); // ✓ — explicit when RHS type isn't apparent
```

The common rule: use `var` when the type is **apparent from the right-hand side** (constructors, casts, descriptive methods); use explicit types when it aids clarity (especially for non-obvious return types and primitives). Configure your team's preference in `.editorconfig`.

---

## Braces and formatting

```csharp
// Allman braces (default C# style) — opening brace on its own line
public void Method()
{
    if (condition)
    {
        DoThing();
    }
}
```

The dominant C# style uses **Allman braces** (brace on a new line). Whatever you choose, **always use braces** even for single statements — prevents the "goto fail" class of bugs:

```csharp
// ✗ — fragile; adding a line silently breaks logic
if (x) DoOne();

// ✓ — braces always
if (x) { DoOne(); }
```

Enforce with `csharp_prefer_braces = true:warning` in `.editorconfig`.

### Expression-bodied members

```csharp
public int Square(int x) => x * x;                  // ✓ — concise one-liner
public string Name => $"{First} {Last}";            // ✓ — computed property
public override string ToString() => $"Order {Id}";  // ✓

public int Complex() => a > b ? Foo(a) : Bar(b) + Baz();  // ⚠ — too dense; use a block
```

Use expression bodies for genuinely simple members. Don't cram complex logic into one line at the cost of readability.

---

## Comments — explain *why*, not *what*

```csharp
// ✗ — restates the code
i++;   // increment i

// ✓ — explains intent the code can't
// Retry once: the gateway occasionally returns a transient 503 on the first call.
if (attempt == 0) Retry();
```

Good code is mostly self-documenting through names. Reserve comments for:
- **Why** a non-obvious decision was made.
- Workarounds and their reasons (link the issue).
- Warnings about non-obvious constraints.
- Complex algorithms' high-level approach.

Delete commented-out code (that's what version control is for). Update comments when code changes — a stale comment is worse than none.

### XML doc comments for public APIs

```csharp
/// <summary>Charges the customer's card for the given amount.</summary>
/// <param name="amount">The amount in the account's currency. Must be positive.</param>
/// <returns>True if the charge succeeded; false if declined.</returns>
/// <exception cref="ArgumentOutOfRangeException">If <paramref name="amount"/> is not positive.</exception>
public bool Charge(decimal amount) { ... }
```

Public/published APIs deserve XML docs (drives IntelliSense and generated docs). Internal code usually doesn't need them — clear names suffice.

---

## Keep methods small and focused

- One method, one responsibility. If you need "and" to describe it, split it.
- A method that doesn't fit on a screen is a smell — extract helpers.
- Few parameters (≤ 3–4); if more, the params probably belong in a parameter object.
- Avoid deep nesting; use guard clauses to flatten (see [08-DefensiveProgramming.md](08-DefensiveProgramming.md)).

```csharp
// ✗ — nested
public void Process(Order o) {
    if (o != null) {
        if (o.Items.Count > 0) {
            if (o.IsValid) {
                // real work, 3 levels deep
            }
        }
    }
}

// ✓ — guard clauses, flat
public void Process(Order o) {
    ArgumentNullException.ThrowIfNull(o);
    if (o.Items.Count == 0) return;
    if (!o.IsValid) return;
    // real work, top level
}
```

---

## Prefer modern language features

Apply the idioms from earlier chapters:

```csharp
// Pattern matching over type checks
var area = shape switch { Circle c => Math.PI * c.R * c.R, Square s => s.Side * s.Side, _ => 0 };

// Target-typed new
private readonly Dictionary<int, string> _map = new();

// Collection expressions
int[] data = [1, 2, 3];

// Null-coalescing / conditional
var name = customer?.Name ?? "Unknown";

// Records for data
public record Money(decimal Amount, string Currency);

// Switch expressions over long if/else chains
```

Newer C# is usually clearer and safer. Don't, however, use a feature just because it's new — readability wins.

---

## Common bugs / gotchas

### Inconsistent style within a codebase

Mixed brace styles, `var` usage, member ordering. Pick conventions, codify in `.editorconfig`, enforce with `dotnet format` in CI.

### Over-commenting trivial code

Comments that restate code add noise and rot. Comment intent, not mechanics.

### God methods

A 300-line method doing everything. Extract cohesive helpers; aim for methods that read top-to-bottom like prose.

### Ignoring analyzer/style warnings

They encode these guidelines. Treat curated warnings as errors (see [Chapter 15 §05](../15-BuildTooling/05-RoslynAnalyzers.md)).

---

## Summary

- One type per file; file-scoped namespaces; consistent member ordering (constants → fields → ctors → properties → methods → nested).
- Use `var` when the type is apparent from the RHS; explicit types otherwise.
- Allman braces, **always braces** even for one statement; expression bodies for simple members only.
- Comments explain **why**, not what; XML docs for public APIs; delete dead code.
- Small, single-responsibility methods; guard clauses over deep nesting; few parameters.
- Prefer modern idioms (patterns, records, target-typed `new`) for clarity; enforce style with `.editorconfig` + analyzers.

→ Next: [03-PerformanceIdioms.md](03-PerformanceIdioms.md)
