# Chapter 03 — The Type System

> The deep model: how C# distinguishes value from reference types, what structs really are, what records add, how boxing works, how the modern pattern-matching system unifies all of it.

**Prerequisites**: [Chapter 02 (OOP)](../02-OOP/README.md). You need classes before structs make sense.

**Time to read**: ~8-10 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-ValueVsReference.md](01-ValueVsReference.md) | The fundamental split: where values live, copy semantics, the consequences for parameter passing and equality. |
| [02-Structs.md](02-Structs.md) | Defining structs, when they make sense, `readonly struct`, `ref struct`, `record struct`, the no-default-ctor rule (pre-C# 10) and what changed. |
| [03-Records.md](03-Records.md) | Records (C# 9), positional vs nominal, synthesized members, value equality, `with` expressions, when records beat classes. |
| [04-Enums.md](04-Enums.md) | Defining enums, underlying types, `[Flags]`, conversion to/from int, `Enum.TryParse`, common patterns. |
| [05-Tuples.md](05-Tuples.md) | `ValueTuple`, named members, deconstruction, when tuples replace tiny classes. |
| [06-NullableTypes.md](06-NullableTypes.md) | `Nullable<T>` for value types (`int?`), the NRT system for references (`string?`), `null` propagation, `??`, `!`. |
| [07-BoxingUnboxing.md](07-BoxingUnboxing.md) | When boxing happens, why it costs, how to spot it in IL, generic constraints that prevent it. |
| [08-TypeConversions.md](08-TypeConversions.md) | Implicit, explicit, user-defined conversions, `as`, `is`, `checked`/`unchecked`, operator overloading. |
| [09-PatternMatching.md](09-PatternMatching.md) | Every pattern: type, constant, var, discard, property, positional, list (C# 11), relational, logical. |
| [10-AnonymousTypes.md](10-AnonymousTypes.md) | `new { X = 1, Y = 2 }`, scope and limitations, why LINQ uses them. |
| [Questions.md](Questions.md) | ~25 questions. |
| [Coding.md](Coding.md) | ~15 problems including the boxing-count trick everyone gets wrong. |

---

## Learning objectives

- Predict whether a variable lives on the stack or heap from its declaration alone.
- Decide between class / struct / record / record struct without hesitation.
- Read modern pattern-matching code (switch expressions with property patterns) fluently.
- Avoid boxing on hot paths.
- Use NRT to eliminate `NullReferenceException` bugs at compile time.

---

## The Big Idea

Most languages have one kind of type. C# has two — value types and reference types — and the entire type system flows from that split. Once you've internalized it, everything else (boxing, copy semantics, structs, equality, generics) follows logically.

→ Begin: [01-ValueVsReference.md](01-ValueVsReference.md)
