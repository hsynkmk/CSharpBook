# Chapter 01 — Fundamentals

> The grammar of C#: variables, types, operators, control flow, methods, strings, arrays, exceptions. Everything you need to write working code before you touch object-orientation.

**Prerequisites**: [Chapter 00](../00-Introduction/README.md) — you should be able to build and run a console app.

**Time to read**: ~6-8 hours if new to programming; ~2 hours if you know Java/C++/JavaScript.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-Variables.md](01-Variables.md) | Declaration, `var`, scope, constants, naming conventions, the difference between declaration and assignment. |
| [02-PrimitiveTypes.md](02-PrimitiveTypes.md) | `int`, `long`, `float`, `double`, `decimal`, `bool`, `char`, `byte`, ranges, overflow, when to use each. |
| [03-StringsAndChars.md](03-StringsAndChars.md) | `string` immutability, interpolation `$""`, raw strings `"""..."""`, UTF-8 literals `"..."u8`, `StringBuilder`, common operations. |
| [04-Operators.md](04-Operators.md) | Arithmetic, comparison, logical, bitwise, null-coalescing `??`, null-conditional `?.`, null-conditional assignment `?.=` (C# 14), precedence rules. |
| [05-ControlFlow.md](05-ControlFlow.md) | `if`/`else`, classic `switch`, `switch` expressions, `for`/`while`/`do-while`/`foreach`, `break`/`continue`/`goto`, ternary, pattern matching in switch. |
| [06-Methods.md](06-Methods.md) | Method signatures, parameters (`ref`, `out`, `in`, `params`), named/optional args, expression-bodied methods, `params ReadOnlySpan<T>` (C# 14). |
| [07-Arrays.md](07-Arrays.md) | 1D, multi-dimensional, jagged arrays, ranges `..` and indices `^`, `Array.Sort`, iteration. |
| [08-ExceptionsBasics.md](08-ExceptionsBasics.md) | `try`/`catch`/`finally`, `throw`, common exception types, exception filters `when`, when to throw vs return error. |
| [09-CommentsAndXmlDocs.md](09-CommentsAndXmlDocs.md) | `//`, `/* */`, XML doc comments `///`, what goes into a good comment. |
| [Questions.md](Questions.md) | 25+ drilling questions. |
| [Coding.md](Coding.md) | 12-15 coding problems: predict output, fix bugs, refactor, optimize. |

---

## Learning objectives

After this chapter you should be able to:
- Write any non-OO procedural program in C# without looking anything up.
- Pick the right primitive type for a job (`int` vs `long` vs `decimal`, `float` vs `double`).
- Use modern string features confidently: interpolation, raw strings, and UTF-8 literals.
- Handle exceptions correctly without overusing try/catch.
- Read and write methods with `ref`/`out`/`in` parameters without confusion.
- Understand the difference between `==` for value types vs reference types.

---

## Style note

C# is verbose by some standards but very expressive. Embrace verbosity for clarity early; learn the shortcuts later. `var` is fine when the type is obvious from context; spelling it out explicitly is also fine.

→ Begin: [01-Variables.md](01-Variables.md)
