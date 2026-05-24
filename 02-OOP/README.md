# Chapter 02 — Object-Oriented Programming

> Classes, objects, properties, inheritance, polymorphism, interfaces. C# is OO at its core — this chapter is the longest in the Beginner section because every later chapter assumes you understand it.

**Prerequisites**: [Chapter 01](../01-Fundamentals/README.md) — methods and basic types.

**Time to read**: ~10-12 hours for a beginner.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-ClassesAndObjects.md](01-ClassesAndObjects.md) | Defining a class, instantiating with `new`, fields, methods, `this`, the difference between a class and an instance. |
| [02-ConstructorsAndInit.md](02-ConstructorsAndInit.md) | Constructors, constructor chaining (`: this(...)`, `: base(...)`), object initializers, target-typed `new()`, `init` accessors. |
| [03-Properties.md](03-Properties.md) | Auto-properties, computed properties, getters/setters, `init`-only, `required` (C# 11), and the **`field` keyword (C# 14)**. |
| [04-FieldsAndAccess.md](04-FieldsAndAccess.md) | Access modifiers: `public`, `private`, `protected`, `internal`, `protected internal`, `private protected`, `file` (C# 11). `readonly` fields, `const` vs `static readonly`. |
| [05-MethodsAdvanced.md](05-MethodsAdvanced.md) | `static`, `virtual`, `override`, `sealed`, `new` (method hiding), `abstract`, `partial`. |
| [06-Inheritance.md](06-Inheritance.md) | Single inheritance, `base`, constructor inheritance, `protected` members, the inheritance vs composition trade-off. |
| [07-Polymorphism.md](07-Polymorphism.md) | Runtime dispatch, the vtable mental model, why `override` differs from `new`, devirtualization by `sealed`. |
| [08-Interfaces.md](08-Interfaces.md) | Defining and implementing interfaces, explicit implementation, default interface methods (C# 8), static abstract members (C# 11). |
| [09-AbstractVsInterface.md](09-AbstractVsInterface.md) | When to use each, layered design (interface + abstract base), shared state. |
| [10-Encapsulation.md](10-Encapsulation.md) | Hiding state, exposing behavior, why public fields are an anti-pattern, defensive copying. |
| [11-NestedAndPartial.md](11-NestedAndPartial.md) | Nested types, partial classes/methods, partial constructors/events (C# 14), when these are useful. |
| [12-PrimaryConstructors.md](12-PrimaryConstructors.md) | Primary constructors for classes and structs (C# 12), how they differ from records. |
| [Questions.md](Questions.md) | ~25 drilling questions. |
| [Coding.md](Coding.md) | ~15 coding problems including the famous "override vs new" output prediction. |

---

## Learning objectives

After this chapter you should be able to:
- Design a class hierarchy that follows SOLID principles (without yet knowing the acronym).
- Decide between class / abstract class / interface for a given problem.
- Explain *why* `virtual` dispatch costs more than a direct call and when the JIT can eliminate that cost.
- Use the C# 14 `field` keyword to write properties with custom logic but no manual backing field.
- Read other people's OO code without surprise — predict output, spot polymorphism bugs.

---

## The Three Concepts That Trip Everyone Up

1. **`override` vs `new`**. They look almost identical at a call site but behave completely differently. Polymorphism (override) → runtime dispatch by actual object type. Hiding (new) → compile-time dispatch by declared variable type. We'll predict outputs in [§07](07-Polymorphism.md) and [Coding.md](Coding.md).

2. **Abstract vs Interface**. Both can have abstract members. Default interface methods (C# 8+) blur the line further. The book takes a strong stance: use interfaces for *capabilities*, abstract classes for *shared implementation*. [§09](09-AbstractVsInterface.md).

3. **Properties are methods**. `public int X { get; set; }` is sugar for `private int _x; public int get_X() {...} public void set_X(int) {...}`. This matters when you reflect, override, or wonder why properties can be virtual. [§03](03-Properties.md).

→ Begin: [01-ClassesAndObjects.md](01-ClassesAndObjects.md)
