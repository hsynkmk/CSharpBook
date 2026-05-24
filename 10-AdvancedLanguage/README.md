# Chapter 10 — Advanced Language Features

> The features that don't fit cleanly into "OO" or "type system": nullable reference types, deep pattern matching, raw strings, UTF-8 literals, interpolated string handlers, caller-info attributes, and more.

**Prerequisites**: Chapters 02 (OOP), 03 (Type System), 04 (Generics), 05 (Delegates).

**Time to read**: ~8-10 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-NullableReferenceTypes.md](01-NullableReferenceTypes.md) | The NRT system (C# 8+), annotation vs warning context, flow analysis, the `!` operator, `[NotNullWhen]` etc. |
| [02-PatternMatchingDeep.md](02-PatternMatchingDeep.md) | All patterns again, deeper: nested property patterns, list patterns with slice, recursive patterns, when expressions. |
| [03-RecordsDeep.md](03-RecordsDeep.md) | Synthesized members in detail, `EqualityContract`, inheritance with records, equality across hierarchies. |
| [04-RequiredMembers.md](04-RequiredMembers.md) | `required` (C# 11), `SetsRequiredMembers`, how it interacts with init-only and constructors. |
| [05-FileScopedTypes.md](05-FileScopedTypes.md) | `file class` (C# 11), scope, primary use case (source generators). |
| [06-TopLevelStatements.md](06-TopLevelStatements.md) | C# 9 top-level statements, the implicit `Main`, capturing args, why source generators love them. |
| [07-GlobalAndImplicitUsings.md](07-GlobalAndImplicitUsings.md) | `global using`, `<ImplicitUsings>enable</ImplicitUsings>`, what the SDK adds for free. |
| [08-CollectionExpressions.md](08-CollectionExpressions.md) | C# 12 `[1, 2, 3]`, spread `..`, target types, where it can be used. |
| [09-RawStringLiterals.md](09-RawStringLiterals.md) | `"""..."""`, indentation rules, interpolation with `$"""...{x}..."""`. |
| [10-UTF8StringLiterals.md](10-UTF8StringLiterals.md) | `"hello"u8` producing a `ReadOnlySpan<byte>`, allocation-free constants. |
| [11-InterpolatedStringHandlers.md](11-InterpolatedStringHandlers.md) | C# 10+ custom handlers, the `[InterpolatedStringHandlerArgument]` attribute, `Debug.Assert` and `ILogger` optimizations. |
| [12-CallerInfoAttributes.md](12-CallerInfoAttributes.md) | `CallerMemberName`, `CallerFilePath`, `CallerLineNumber`, `CallerArgumentExpression` for great error messages. |
| [13-CheckedOperators.md](13-CheckedOperators.md) | User-defined `checked` operators, `checked` blocks, when you'd write one. |
| [Questions.md](Questions.md) | ~25 questions. |
| [Coding.md](Coding.md) | ~12 problems. |

→ Begin: [01-NullableReferenceTypes.md](01-NullableReferenceTypes.md)
