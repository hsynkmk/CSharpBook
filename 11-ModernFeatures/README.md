# Chapter 11 — Modern Features (Version by Version)

> A chronological tour from C# 7 through C# 14, plus the .NET 10 runtime improvements. If you're modernizing legacy code or interviewing about "what's new," this is your reference.

**Prerequisites**: most of the earlier chapters — features here often refer back to fundamentals covered there.

**Time to read**: ~4-6 hours for a thorough pass.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-CSharp7-8-History.md](01-CSharp7-8-History.md) | C# 7.0-7.3 (tuples, out vars, local funcs, pattern matching v1) and C# 8 (DIM, NRT, async streams, ranges, indices, `using` declaration). |
| [02-CSharp9.md](02-CSharp9.md) | Records, init-only setters, top-level statements, target-typed `new()`, `with` expressions, function pointers, source generators GA. |
| [03-CSharp10.md](03-CSharp10.md) | Global usings, file-scoped namespaces, record struct, parameterless struct ctors, extended property patterns. |
| [04-CSharp11.md](04-CSharp11.md) | Required members, raw string literals, generic math, list patterns, UTF-8 string literals, `file` modifier, `nameof` for parameters. |
| [05-CSharp12.md](05-CSharp12.md) | Primary constructors for classes/structs, collection expressions `[1,2,3]`, `using` alias for any type, ref readonly parameters. |
| [06-CSharp13.md](06-CSharp13.md) | `params` collections (any IEnumerable, Span, ReadOnlySpan), partial properties/indexers, `lock` object type, escape sequence `\e`, indexer `^` in initializers. |
| [07-CSharp14.md](07-CSharp14.md) | **Extension members** (properties/operators/statics), **`field` keyword**, **null-conditional assignment `?.=`**, first-class `Span<T>` conversions, **`params ReadOnlySpan<T>`**, unbound generics in `nameof`, **partial constructors and events**, user-defined compound assignment, **file-based apps**. |
| [08-DotNet10Runtime.md](08-DotNet10Runtime.md) | **DATAS GC** default, **escape analysis** + stack promotion, **JIT block reordering** (3-opt TSP), **reduced write barriers**, **`Task.WhenEach`**, async state-machine box elision. |
| [Questions.md](Questions.md) | ~25 questions across all versions. |
| [Coding.md](Coding.md) | ~12 problems: refactor C# 7 code to C# 14, identify which version introduced what. |

→ Begin: [01-CSharp7-8-History.md](01-CSharp7-8-History.md)
