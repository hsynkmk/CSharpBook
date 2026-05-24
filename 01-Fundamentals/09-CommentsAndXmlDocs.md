# Comments and XML Docs

## What it is

Three kinds of comments in C#:
- **Single-line** `// ...`
- **Multi-line** `/* ... */`
- **XML doc comments** `/// ...` — machine-readable, used by tooling.

```csharp
// This is a single-line comment.

/* This is
   a multi-line comment. */

/// <summary>
/// XML doc — read by IntelliSense and documentation generators.
/// </summary>
public void DoStuff() { }
```

---

## Single-line `//`

The most common form. Use for short notes, "todo" markers, or commenting out one line.

```csharp
int retries = 3;   // arbitrary but seems plenty

// TODO: revisit this once we have proper telemetry
DoWork();

// var legacy = OldMethod();   // commented out while we migrate
```

Style notes:
- Leave one space after `//`.
- Put end-of-line comments at column ~60-80 to give code room.
- "TODO:", "HACK:", "FIXME:", and "NOTE:" are common conventions IDEs highlight.

---

## Multi-line `/* */`

Useful for long block explanations or temporarily disabling several lines:

```csharp
/*
 * The algorithm here is Boyer-Moore-Horspool:
 *  1. Build a "shift" table from the pattern.
 *  2. Compare against text from right to left.
 *  3. On mismatch, shift by the table amount.
 *
 * Not the fastest substring search in .NET (use String.IndexOf
 * which uses SIMD), but useful to study.
 */
```

Note: **multi-line comments don't nest**:

```csharp
/* outer
   /* inner */ <-- this closes the comment
   still outside */ <-- this is a syntax error
```

`//` works inside `/* */`, so you can nest `//` chunks freely.

---

## XML doc comments `///`

The big one. Three slashes turn the comment into structured documentation:

```csharp
/// <summary>
/// Calculates the area of a circle with the given radius.
/// </summary>
/// <param name="radius">The radius in meters. Must be non-negative.</param>
/// <returns>The area in square meters.</returns>
/// <exception cref="ArgumentOutOfRangeException">
/// Thrown when <paramref name="radius"/> is negative.
/// </exception>
public double CircleArea(double radius) {
    ArgumentOutOfRangeException.ThrowIfNegative(radius);
    return Math.PI * radius * radius;
}
```

These show up in:
- **IntelliSense** — hovering over `CircleArea` in your IDE shows the summary, parameters, exceptions.
- **`docs/`** — tools like DocFX, Sandcastle, or doc-generating CI produce HTML/PDF API references from them.
- **NuGet packages** — when packing, the SDK includes `.xml` files alongside DLLs so consumers see your docs.

---

## Common XML tags

### `<summary>`
A one-paragraph description. The most important tag.

```csharp
/// <summary>
/// Loads a user by ID from the underlying repository.
/// Returns <see langword="null"/> if not found.
/// </summary>
```

### `<param>`
One per parameter:

```csharp
/// <param name="userId">The numeric identifier.</param>
/// <param name="includeDeleted">Whether to include soft-deleted records.</param>
```

### `<returns>`
What the method returns:

```csharp
/// <returns>The matching user, or <see langword="null"/> if none.</returns>
```

For `void` methods, skip `<returns>`.

### `<exception>`
Exceptions the method may throw:

```csharp
/// <exception cref="ArgumentException">The ID was invalid.</exception>
/// <exception cref="UnauthorizedAccessException">No permission.</exception>
```

`cref` (code reference) is verified at compile time — if you typo the type name you get a warning.

### `<remarks>`
Extended discussion that doesn't fit in summary:

```csharp
/// <summary>Loads a user by ID.</summary>
/// <remarks>
/// This method uses a per-request cache; calling it twice in the same scope
/// will return the same instance. For a fresh fetch, call <see cref="Reload"/>.
/// </remarks>
```

### `<example>`
Code samples:

```csharp
/// <example>
/// <code>
/// var user = await repo.LoadAsync(42);
/// Console.WriteLine(user?.Name);
/// </code>
/// </example>
```

### `<see>` and `<seealso>`
Cross-references:

```csharp
/// See <see cref="SaveAsync(User)"/> for the inverse operation.
/// <seealso cref="Repository"/>
```

`<see langword="..."/>` formats keywords:

```csharp
/// Returns <see langword="true"/> on success, <see langword="false"/> otherwise.
```

### `<paramref>` and `<typeparamref>`
Reference parameters and type parameters from within prose:

```csharp
/// Throws when <paramref name="radius"/> is negative.
/// Operates on values of type <typeparamref name="T"/>.
```

### `<list>`
Bulleted or numbered lists:

```csharp
/// <list type="bullet">
///   <item><description>First point</description></item>
///   <item><description>Second point</description></item>
/// </list>
```

`type` can be `"bullet"`, `"number"`, or `"table"`.

### `<typeparam>`
For generic types/methods:

```csharp
/// <typeparam name="T">The element type.</typeparam>
public class Stack<T> { }
```

### `<inheritdoc/>`
Inherits documentation from the base class/interface:

```csharp
public class Logger : ILogger {
    /// <inheritdoc/>
    public void Log(string msg) { }
}
```

Saves duplication. Most modern doc generators understand it.

---

## A complete example

```csharp
/// <summary>
/// A thread-safe LRU cache with bounded capacity.
/// </summary>
/// <typeparam name="TKey">The key type. Must be non-null and implement <see cref="IEquatable{T}"/>.</typeparam>
/// <typeparam name="TValue">The cached value type.</typeparam>
/// <remarks>
/// Implementation uses a doubly-linked list for O(1) eviction and a
/// <see cref="Dictionary{TKey, TValue}"/> for O(1) lookup. Lookups
/// promote items to the front of the LRU order.
/// </remarks>
public sealed class LruCache<TKey, TValue> where TKey : notnull {

    /// <summary>
    /// Creates a new cache with the specified maximum capacity.
    /// </summary>
    /// <param name="capacity">Maximum number of entries. Must be positive.</param>
    /// <exception cref="ArgumentOutOfRangeException">
    /// Thrown when <paramref name="capacity"/> is not greater than zero.
    /// </exception>
    public LruCache(int capacity) { }

    /// <summary>
    /// Attempts to retrieve a cached value.
    /// </summary>
    /// <param name="key">The lookup key.</param>
    /// <param name="value">When this returns <see langword="true"/>, the cached value.</param>
    /// <returns><see langword="true"/> if the key was present; otherwise <see langword="false"/>.</returns>
    public bool TryGetValue(TKey key, out TValue value) { ... }
}
```

---

## Generating docs

To produce an XML doc file at build time, add to your `.csproj`:

```xml
<PropertyGroup>
    <GenerateDocumentationFile>true</GenerateDocumentationFile>
</PropertyGroup>
```

Now `dotnet build` produces a `.xml` next to the `.dll`. Tools like **DocFX** (Microsoft's static-site doc generator) consume these and produce documentation websites.

The compiler also warns about missing summaries on public APIs by default once this flag is set. Silence with `<NoWarn>CS1591</NoWarn>` or comment more thoroughly.

---

## When to comment

Comments are a controversial topic. The pragmatic view:

**DO** comment:
- **The "why"** — not what the code does (the code says that), but why it does it. "Cap retries at 5 to match the upstream service's rate limit."
- **Surprises** — counter-intuitive choices, workarounds for bugs in libraries, magic numbers with non-obvious origins.
- **Public APIs** — yes, every public type and method should have a `<summary>` at minimum.
- **Complex algorithms** — point to the paper, explain the invariant.

**DON'T** comment:
- **The obvious** — `// increment i` next to `i++` is noise.
- **Bad names** — rename the variable instead of explaining what it holds.
- **Outdated information** — comments rot when code changes. Better to write self-documenting code.
- **Commented-out code** — delete it. Git remembers.

A good rule: if you find yourself writing a long comment to explain a function, consider whether the function should be refactored into smaller, well-named pieces.

---

## Modern alternatives — analyzers and naming

Many things comments used to express are now better expressed in code:
- **`[Obsolete]`** attribute — replaces "don't use this anymore" comments.
- **`[Required]`**, `required` keyword (C# 11+) — replaces "you must set this" comments.
- **NRT annotations (`?`)** — replaces "this can return null" comments.
- **Method names** — replaces "what does this do" comments.
- **Unit tests** — replace "use it like this" comments.

---

## Common bugs and style issues

- **`/// <summary>This sentence doesn't end.</summary>`** — finish summaries with periods.
- **Outdated `<param>` after rename** — happens often. IDE warnings catch most of these.
- **Cargo-cult `///` on private members** — debatable; some teams require it, others say public-only.
- **Long doc blocks for trivial properties** — properties usually need only a one-line summary.
- **Forgetting `///` on the first line** — the IDE often inserts a single-line `///` then nothing for follow-up lines. Be careful.

---

## Summary

- `//` for short notes, `/* */` for blocks, `///` for documentation.
- XML doc tags integrate with IDE, NuGet, and doc generators.
- Comment the why, not the what. Let code be self-documenting.
- `<summary>`, `<param>`, `<returns>`, `<exception>`, `<remarks>` cover 90% of needs.
- `<inheritdoc/>` saves duplication; `cref` cross-refs are verified by the compiler.

→ Continue to: [Questions.md](Questions.md)
