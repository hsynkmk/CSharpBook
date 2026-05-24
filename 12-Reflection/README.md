# Chapter 12 — Reflection & Metaprogramming

> Looking at code from the outside: inspecting types at runtime, generating code at compile time, attaching metadata via attributes, and replacing reflection with source generators when performance matters.

**Prerequisites**: Chapters 02 (OOP), 04 (Generics).

**Time to read**: ~5-7 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-ReflectionBasics.md](01-ReflectionBasics.md) | `Type`, `typeof`, `GetType()`, `MethodInfo`, `PropertyInfo`, `FieldInfo`, invoking members dynamically. |
| [02-Activator.md](02-Activator.md) | `Activator.CreateInstance`, generic `Activator.CreateInstance<T>()`, performance characteristics, JIT optimizations (.NET 6+). |
| [03-Attributes.md](03-Attributes.md) | Defining and querying attributes, `AttributeUsage`, retrieval via reflection, common BCL attributes. |
| [04-DynamicAndExpando.md](04-DynamicAndExpando.md) | The `dynamic` keyword, `ExpandoObject`, the DLR (Dynamic Language Runtime), when (rarely) it's the right tool. |
| [05-SourceGenerators.md](05-SourceGenerators.md) | `IIncrementalGenerator`, when source gen replaces reflection, the regex/`System.Text.Json` source generators, writing your own. |
| [06-CompileTimeReflection.md](06-CompileTimeReflection.md) | `nameof`, `typeof`, `CallerMemberName`, `CallerArgumentExpression` — reflection that costs nothing. |
| [07-PerformanceConcerns.md](07-PerformanceConcerns.md) | Caching `MethodInfo`, `Expression.Compile` vs `Reflection.Emit` vs source generators, when to AOT-compile reflection. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~12 problems: build a custom JSON serializer with reflection, then with a source generator. |

→ Begin: [01-ReflectionBasics.md](01-ReflectionBasics.md)
