# Chapter 17 — Best Practices & Idioms

> The opinions that come with experience. Style, design principles, patterns, API design, error handling, async and collection idioms, immutability, equality, defensive programming, and anti-patterns to avoid. The final chapter; read it after everything else.

**Prerequisites**: the rest of the book.

**Time to read**: ~8-10 hours.

---

## What's in this chapter

The files are grouped below by theme in a sensible reading order. (File numbers reflect the order they were added; follow the `→ Next` links to read straight through.)

### Style & conventions
| File | What it covers |
|---|---|
| [01-NamingConventions.md](01-NamingConventions.md) | PascalCase / camelCase / `_camelCase` — when to use each, acronym rules, async suffix, framework conventions. |
| [02-CodingGuidelines.md](02-CodingGuidelines.md) | File organization, member ordering, `var` usage, braces, comment philosophy, small methods, modern idioms. |

### Design principles & patterns
| File | What it covers |
|---|---|
| [10-SOLID.md](10-SOLID.md) | The five SOLID principles in idiomatic C#, with examples — and when *not* to over-apply them. |
| [11-DesignPatterns.md](11-DesignPatterns.md) | GoF patterns in modern C#; which the language/BCL made obsolete (`yield`, `Lazy<T>`, `event`, `record`) and which still earn their keep. |
| [12-DependencyInjection.md](12-DependencyInjection.md) | DI as a design principle: constructor injection, composition root, pure DI, lifetimes, service-locator anti-pattern. |
| [04-ApiDesign.md](04-ApiDesign.md) | Public-API rules, breaking changes, exception design, parameter ordering, Try-pattern, return/accept types. |

### Correctness
| File | What it covers |
|---|---|
| [13-ErrorHandling.md](13-ErrorHandling.md) | Strategy: exceptions vs result types, exception design, where to catch, stack preservation, validation, logging once. |
| [14-Equality.md](14-Equality.md) | `Equals`/`GetHashCode` correctness, records, `IEquatable<T>`, `IEqualityComparer<T>`, immutable keys, hashing pitfalls. |
| [08-DefensiveProgramming.md](08-DefensiveProgramming.md) | `ArgumentNullException.ThrowIfNull`, `ObjectDisposedException.ThrowIf`, guard clauses, fail-fast, NRT as a tool. |

### Idioms
| File | What it covers |
|---|---|
| [05-AsyncIdioms.md](05-AsyncIdioms.md) | The `*Async` suffix, CancellationToken last + forwarded, ConfigureAwait in libraries, avoiding sync-over-async. |
| [06-CollectionIdioms.md](06-CollectionIdioms.md) | Accept `IEnumerable<T>`, return `IReadOnlyList<T>`, empty-not-null, right collection for the job, `TryGetValue`. |
| [07-ImmutabilityPatterns.md](07-ImmutabilityPatterns.md) | Records vs `init`/`required` vs builders, `ImmutableArray` vs `ImmutableList`, shallow-immutability traps, when it's overkill. |
| [03-PerformanceIdioms.md](03-PerformanceIdioms.md) | Measure first; `Span<T>`/`stackalloc`/`ArrayPool`, avoid boxing/LINQ in hot paths, struct enumerators, JIT-friendly code. |

### Pulling it together
| File | What it covers |
|---|---|
| [09-CommonAntiPatterns.md](09-CommonAntiPatterns.md) | God classes, primitive obsession, anemic models, magic strings, service locator, `async void`, sync-over-async, swallowed exceptions, `throw ex;`, and the fixes. |
| [Questions.md](Questions.md) | ~35 questions across all topics. |
| [Coding.md](Coding.md) | Refactoring problems — take bad code, make it good. |

→ Begin: [01-NamingConventions.md](01-NamingConventions.md)
