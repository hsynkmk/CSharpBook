# Chapter 05 — Delegates & Events

> Functions as first-class values: delegates, lambdas, closures, events, expression trees. The bridge between OO and functional programming in C#.

**Prerequisites**: [Chapter 02 (OOP)](../02-OOP/README.md), [Chapter 04 (Generics)](../04-Generics/README.md).

**Time to read**: ~4-5 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Delegates.md](01-Delegates.md) | What a delegate is, declaring custom delegates, multicast, `Invoke` vs direct call. |
| [02-Lambdas.md](02-Lambdas.md) | Lambda syntax, statement vs expression form, attributes on lambdas (C# 10), `static` lambdas. |
| [03-FuncActionPredicate.md](03-FuncActionPredicate.md) | Built-in generic delegates, when to use which, the 17-parameter limit. |
| [04-Closures.md](04-Closures.md) | What captures, how the compiler implements them (closure class), capture bugs in loops, allocation cost. |
| [05-Events.md](05-Events.md) | `event` keyword, the classic pattern, `EventHandler` / `EventHandler<T>`, raising thread-safely, the subscription-leak trap. |
| [06-LocalFunctions.md](06-LocalFunctions.md) | Nested named functions, when they beat lambdas, allocation-free closure capture. |
| [07-AnonymousMethods.md](07-AnonymousMethods.md) | The old `delegate(...)` syntax — modern code uses lambdas instead, but legacy code still has it. |
| [08-ExpressionTrees.md](08-ExpressionTrees.md) | `Expression<Func<...>>`, how LINQ providers (EF Core) translate C# to SQL, building/compiling expressions at runtime. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~12 problems, heavy on closure-capture predictions and event-leak fixes. |

→ Begin: [01-Delegates.md](01-Delegates.md)
