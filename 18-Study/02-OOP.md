# 02 — OOP (polymorphism, override vs new, abstract vs interface)

## ⚡ 30-second answer

The four pillars: **encapsulation** (hide state behind a controlled surface), **inheritance** (reuse via an is-a hierarchy), **polymorphism** (one call site, many runtime behaviors via virtual dispatch), **abstraction** (program to abstractions). The classic trap is **`override` vs `new`**: `override` participates in **runtime virtual dispatch** (the actual object's type decides), while `new` just **hides** the base member (the *variable's* compile-time type decides). Prefer **interfaces** for capability/contract and **abstract classes** when you need shared state/implementation.

---

## Core mechanics

**Virtual dispatch** — `virtual`/`override` is resolved at runtime via the object's **method table (vtable)**:

```csharp
class Animal { public virtual string Speak() => "..."; }
class Dog : Animal { public override string Speak() => "Woof"; }

Animal a = new Dog();
a.Speak();   // "Woof"  — runtime type (Dog) wins
```

**`new` (method hiding)** — resolved at **compile time** by the variable's declared type:

```csharp
class Base { public void M() => Console.WriteLine("Base"); }
class Derived : Base { public new void M() => Console.WriteLine("Derived"); }

Base b = new Derived();
b.M();              // "Base"   — declared type Base wins (NOT virtual)
((Derived)b).M();   // "Derived"
```

**Modifiers**: `virtual` (overridable), `override` (replaces base virtual), `sealed override` (stop further overriding), `abstract` (no body, must override), `new` (hide). A method is **non-virtual by default** in C#.

**Abstract class** — can have state, constructors, and both abstract & concrete members; single inheritance.
**Interface** — a contract; supports **multiple** implementation, **default interface methods** (C# 8), and **static abstract members** (C# 11, for generic math).

---

## Comparison tables

| | `override` | `new` (hiding) |
|---|---|---|
| Dispatch | **runtime** (vtable) | **compile-time** (declared type) |
| Requires base `virtual`? | yes | no |
| Polymorphic? | **yes** | **no** |
| Use when | true poly behavior | rarely — usually a mistake |

| | Abstract class | Interface |
|---|---|---|
| Multiple inheritance | no (one base) | **yes** (many) |
| State / fields | yes | no (only static fields) |
| Constructors | yes | no |
| Default implementation | yes | yes (DIM, C# 8) |
| Use for | shared base + state, "is-a" | capability/contract, "can-do" |
| Versioning | add members freely | adding a member breaks implementers (unless default) |

---

## 🪤 Traps & gotchas

- **`new` instead of `override`** → no polymorphism; behavior depends on the variable type, not the object. Almost always a bug; the compiler warns ("hides inherited member; use `new`").
- **Calling virtual methods from a constructor** → runs the **derived** override before the derived ctor body has initialized fields → operating on half-built state. Avoid.
- **Non-virtual by default** (unlike Java/C++) — forgetting `virtual` means a derived "override" silently hides.
- **Interface re-implementation / explicit implementation** can change which method runs depending on the reference type — subtle.
- **Liskov violation**: a derived type that throws on a method the base supports, or narrows accepted input, breaks polymorphism (LSP — [22](22-Architecture-Patterns.md)).
- **Protected ≠ accessible to unrelated code in the same assembly** — `protected` is for derived types; `internal` is for the assembly; `protected internal` = either; `private protected` = derived **and** same assembly.

---

## ❓ Likely questions

**Q: `override` vs `new`?**
A: `override` does runtime virtual dispatch (object's type wins, requires `virtual` base). `new` hides — compile-time, the declared variable type wins, not polymorphic.

**Q: Abstract class vs interface — when each?**
A: Abstract class when you need shared state/implementation and an "is-a" base (single inheritance). Interface for a contract/capability that many unrelated types can implement (multiple), or for DI seams.

**Q: Why are methods non-virtual by default in C#?**
A: Performance and safety — virtual calls cost a vtable lookup and prevent inlining; making it opt-in means you only pay for polymorphism you intend, and base authors control extensibility.

**Q: What's wrong with calling a virtual method in a constructor?**
A: The derived override runs before the derived fields are initialized → it sees default/uninitialized state.

**Q: Can an interface have implementation now?**
A: Yes — default interface methods (C# 8) and static abstract members (C# 11). But interfaces still hold no instance state.

**Q: Access modifiers, briefly?**
A: `public` (everywhere), `private` (type), `protected` (type + derived), `internal` (assembly), `protected internal` (assembly OR derived), `private protected` (assembly AND derived), `file` (file).

**Q: What are the four OOP pillars?**
A: Encapsulation, inheritance, polymorphism, abstraction.

---

## 🎓 Senior Extra

- **Vtable / method table**: each type has a method table; `override` slots replace the base entry, so a virtual call is an indirect jump through the table. The JIT can **devirtualize** (and inline) when it can prove the concrete type (sealed types, `Dynamic PGO` guessing the common type with a guard).
- **`sealed`** on classes/methods enables devirtualization and inlining → a real perf lever in hot paths; also expresses design intent.
- **Default interface methods** changed the versioning story: you can add a method to a published interface with a default body without breaking implementers — but they introduce diamond/most-specific-override rules; use sparingly.
- **Composition over inheritance**: deep inheritance hierarchies are fragile (base changes ripple). Favor composing behavior (inject collaborators) — this is the bias behind DI and strategy patterns ([22](22-Architecture-Patterns.md)).
- **Explicit interface implementation** hides a member from the class's public surface (only callable via the interface) — used to resolve name clashes or to keep an implementation detail off the type's API.
- **Static abstract interface members** (C# 11) enable **generic math** (`INumber<T>`) — operators/static factory methods constrained generically.
- **Covariance/contravariance** of interfaces (`out`/`in`) interacts with polymorphism — see [03](03-Generics-Delegates-Events.md).

→ Deeper: [`../CSharpBook/02-OOP/`](../CSharpBook/02-OOP/README.md)
