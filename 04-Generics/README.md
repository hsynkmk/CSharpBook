# Chapter 04 — Generics

> Type parameterization: `List<T>`, `Dictionary<K,V>`, and the constraint system that makes them powerful. Plus the modern features (static abstract members, generic math) that turned generics into something C++ templates can almost envy.

**Prerequisites**: [Chapter 02 (OOP)](../02-OOP/README.md), [Chapter 03 (Type System)](../03-TypeSystem/README.md).

**Time to read**: ~5-6 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-GenericsBasics.md](01-GenericsBasics.md) | Why generics exist, syntax for generic classes/methods/interfaces, type inference. |
| [02-GenericMethods.md](02-GenericMethods.md) | Standalone generic methods, when type inference works and when you must specify. |
| [03-Constraints.md](03-Constraints.md) | `where T : class / struct / new() / notnull / unmanaged / interface / base class`, multiple constraints. |
| [04-Variance.md](04-Variance.md) | `out T` (covariance), `in T` (contravariance), why `List<string>` isn't a `List<object>`. |
| [05-StaticAbstractMembers.md](05-StaticAbstractMembers.md) | C# 11+ static abstract members in interfaces — the foundation of generic math. |
| [06-GenericMath.md](06-GenericMath.md) | `INumber<T>`, `IAdditionOperators<T,T,T>`, writing math algorithms that work on any numeric type. |
| [07-DefaultLiteralAndT.md](07-DefaultLiteralAndT.md) | `default(T)`, `default` literal, what it returns for unconstrained T. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~12 problems including variance prediction and boxing-prevention via constraints. |

---

## Learning objectives

- Write generic types and methods with appropriate constraints.
- Predict whether code compiles when assigning `IFoo<Derived>` to `IFoo<Base>` or vice versa.
- Use C# 11+ generic math to write algorithms reusable across `int`, `double`, `decimal`.
- Know the boxing cost of generic-vs-non-generic interface calls.

→ Begin: [01-GenericsBasics.md](01-GenericsBasics.md)
