# Chapter 02 — Questions

> Drilling questions for every section of the OOP chapter. Answer first, then check.

---

## Classes and objects

**Q1.** What's the difference between a class and an instance of that class?
<details><summary>Answer</summary>A class is a **type** — a blueprint describing fields, methods, and behavior. An instance is a **value** of that type — a specific runtime object with its own field storage. The class lives in metadata; instances live on the heap.</details>

**Q2.** Where do instances of `class Foo` live?
<details><summary>Answer</summary>On the managed heap. Reference types are always heap-allocated; the variable holding `Foo` is a reference (pointer-sized) that can sit on the stack, on the heap (as a field of another reference type), or in a register.</details>

**Q3.** Predict the output:
```csharp
class Box { public int Value; }
var a = new Box { Value = 5 };
var b = a;
b.Value = 10;
Console.WriteLine(a.Value);
```
<details><summary>Answer</summary>`10`. `a` and `b` are two references to the **same** heap object. Mutating through one is observed by the other.</details>

**Q4.** What does `this` mean inside an instance method?
<details><summary>Answer</summary>The reference to the current instance. Compiled as an implicit first argument to the method. Inside a `struct` method, `this` is a `ref` to the struct, not a copy.</details>

---

## Constructors and initialization

**Q5.** What's the difference between a field initializer and a constructor assignment?
<details><summary>Answer</summary>Field initializers run **before** the constructor body but **after** the base constructor. Assignments inside the constructor body run **after** any base constructor and after field initializers, in textual order.</details>

**Q6.** What order do these run in?
```csharp
class A {
    int x = 1;       // (1)
    public A() { x = 2; }  // (2)
}
class B : A {
    int y = 10;      // (3)
    public B() { y = 20; } // (4)
}
new B();
```
<details><summary>Answer</summary>(1) → (2) → (3) → (4). Base field inits, base ctor, derived field inits, derived ctor. Field initializers always run before any constructor body in the *same* class, and base completes before derived begins.</details>

**Q7.** What does `: base(...)` do?
<details><summary>Answer</summary>Calls a specific base-class constructor. Required when the base has no parameterless constructor. Without it, the compiler implicitly inserts `: base()`.</details>

**Q8.** What is target-typed `new()`?
<details><summary>Answer</summary>C# 9 syntax letting the compiler infer the type from the target: `List<int> list = new();` instead of `new List<int>()`. Useful when the type name is already on the left.</details>

**Q9.** What's an `init` accessor?
<details><summary>Answer</summary>A property setter that can only be called during object construction — including object initializers, primary constructor parameters, and `with`-expressions. After construction it behaves like `readonly`.</details>

**Q10.** What does `required` (C# 11) do?
<details><summary>Answer</summary>Forces callers to set the property during initialization. Constructors that satisfy `required` properties must be marked `[SetsRequiredMembers]`. Object initializers must set every `required` member or it's a compile error.</details>

---

## Properties

**Q11.** What does the compiler do with `public int Age { get; set; }`?
<details><summary>Answer</summary>Generates a hidden backing field (mangled name like `<Age>k__BackingField`), a `get_Age()` method that reads it, and a `set_Age(int)` method that writes it.</details>

**Q12.** What's the `field` keyword (C# 14)?
<details><summary>Answer</summary>Inside a property body, `field` refers to the compiler-generated backing field — you don't have to declare it manually. Lets you add custom getter/setter logic without losing auto-property convenience.

```csharp
public string Name {
    get => field ?? "(none)";
    set => field = value?.Trim();
}
```
</details>

**Q13.** Predict:
```csharp
class C { public int X { get; } = 5; }
var c = new C();
Console.WriteLine(c.X);
```
<details><summary>Answer</summary>`5`. Auto-property initializers run as part of the implicit constructor. The `get`-only auto-property is readable; nobody can assign to it after construction.</details>

**Q14.** Why are `public fields` an anti-pattern and `public properties` not?
<details><summary>Answer</summary>Properties are methods. You can add validation, change the backing storage, fire change events, or replace with computed logic without breaking the binary contract. A public field locks the storage layout into the public API.</details>

---

## Fields and access modifiers

**Q15.** Rank from most to least restrictive: `internal`, `private`, `protected`, `protected internal`, `private protected`, `public`.
<details><summary>Answer</summary>From most restrictive to least: `private` → `private protected` → `protected` / `internal` (incomparable: assembly vs hierarchy) → `protected internal` → `public`.</details>

**Q16.** What's the difference between `internal` and `protected internal`?
<details><summary>Answer</summary>`internal` = same assembly only. `protected internal` = same assembly **OR** derived class (in any assembly). It's an OR — broader, not narrower.</details>

**Q17.** What's `private protected`?
<details><summary>Answer</summary>Same assembly **AND** derived class — the intersection. Narrower than `protected` (which permits any-assembly derived classes) and narrower than `internal`.</details>

**Q18.** When does `readonly` actually freeze a field?
<details><summary>Answer</summary>The field reference can't be reassigned after construction. But if the field is a mutable reference type (e.g., `readonly List<int>`), the contents can still be mutated through it. `readonly` is about the binding, not the value.</details>

**Q19.** Difference between `const` and `static readonly`?
<details><summary>Answer</summary>`const` is compile-time and inlined into every consumer. `static readonly` is set at runtime (field init or static ctor) and read at runtime. Changing a `const` requires recompiling every consumer; changing a `static readonly` does not.</details>

---

## Methods, virtual, override, new

**Q20.** What does `virtual` mean for the JIT?
<details><summary>Answer</summary>The call goes through a virtual method table (vtable) lookup at runtime — the actual method is chosen by the object's runtime type, not the declared variable type. JIT may devirtualize when it can prove the actual type (sealed class, sealed method, struct, or known concrete type).</details>

**Q21.** Predict and explain:
```csharp
class A { public virtual void Print() => Console.WriteLine("A"); }
class B : A { public override void Print() => Console.WriteLine("B"); }
class C : A { public new void Print() => Console.WriteLine("C"); }

A b = new B(); b.Print();
A c = new C(); c.Print();
```
<details><summary>Answer</summary>`B` then `A`. `override` participates in the vtable, so the call to `b.Print()` dispatches to B. `new` hides at compile-time — when called through the base reference, the base's version is selected. `c.Print()` only prints "C" if you call it as `((C)c).Print()`.</details>

**Q22.** When is `sealed` useful on a method?
<details><summary>Answer</summary>To stop further overriding partway down the hierarchy: `public sealed override void Foo()`. Lets the JIT devirtualize calls in subclasses and signals "this branch of behavior is closed."</details>

**Q23.** What is method hiding (`new`) good for? When is it dangerous?
<details><summary>Answer</summary>Useful when the base method's signature is identical but you want a completely different behavior in your derived class, without participating in polymorphism. Dangerous because callers holding a base reference get the base behavior — surprising if they expect overriding semantics. Almost always a code smell.</details>

**Q24.** What does `abstract` on a method imply?
<details><summary>Answer</summary>No body, no implementation. The enclosing class must also be `abstract`. Subclasses must override the abstract method (unless they're also `abstract`). Implicit `virtual`.</details>

**Q25.** What does `partial` on a method mean (original form)?
<details><summary>Answer</summary>A method declared in one partial-class part and optionally defined in another. If never defined, the call site is erased at compile time — arguments not evaluated. Implicit `private`, `void` return, no `out` parameters.</details>

---

## Inheritance

**Q26.** Why does C# disallow multiple class inheritance?
<details><summary>Answer</summary>Avoids the "diamond problem" (ambiguous member resolution when two base classes share an ancestor with different overrides) and keeps the vtable model simple. Use multiple **interface** inheritance to compose capabilities.</details>

**Q27.** What can you put in a `: base(...)` call?
<details><summary>Answer</summary>Any expression evaluated in the context of the constructor's parameters — including method calls. The base call runs **before** any field initializer of the derived class.</details>

**Q28.** Should you prefer inheritance or composition? Why?
<details><summary>Answer</summary>Composition by default. Inheritance is permanent — the base class becomes part of your public contract. Composition is replaceable. Inheritance is right when the relationship is genuinely "is-a" and the base is stable; composition is right when it's "has-a" or "uses-a."</details>

**Q29.** Predict:
```csharp
class Animal { public Animal() => Speak(); public virtual void Speak() {} }
class Dog : Animal {
    string sound;
    public Dog() { sound = "woof"; }
    public override void Speak() => Console.WriteLine(sound ?? "null");
}
new Dog();
```
<details><summary>Answer</summary>`null`. The base constructor calls the virtual `Speak`, which dispatches to `Dog.Speak`. But the `Dog` constructor hasn't run yet — `sound` is still default (`null`). The classic "don't call virtual from constructors" pitfall.</details>

---

## Polymorphism

**Q30.** What is the "vtable" model?
<details><summary>Answer</summary>Each polymorphic class has a virtual method table — an array of method pointers, one slot per virtual method. Every instance carries an implicit reference to its class's vtable (via the type handle). A virtual call loads the vtable pointer, indexes into it, and jumps. The CLR uses a slightly more elaborate scheme (interface dispatch goes through a separate map), but the mental model is the same: an indirection through type-specific tables.</details>

**Q31.** When is a virtual call "devirtualized"?
<details><summary>Answer</summary>When the JIT can prove the exact runtime type. Common triggers: the class or method is `sealed`, the type is a `struct` (value types use static dispatch), or the JIT has inlined enough context to know the concrete type. Once devirtualized, the call becomes a direct call — same cost as a non-virtual method.</details>

**Q32.** Predict:
```csharp
class Base { public virtual string Name => "Base"; }
class Mid : Base { public override string Name => "Mid"; }
class Leaf : Mid { public new string Name => "Leaf"; }

Base b = new Leaf();
Mid m = new Leaf();
Leaf l = new Leaf();
Console.WriteLine($"{b.Name} {m.Name} {l.Name}");
```
<details><summary>Answer</summary>`Mid Mid Leaf`. `Base.Name` and `Mid.Name` form a virtual chain (Mid overrides Base), so calls through `b` and `m` dispatch to `Mid.Name`. `Leaf.Name` is `new` — it hides, doesn't override, so it's only visible when the static type is `Leaf`.</details>

**Q33.** What's the cost of a virtual call vs a direct call?
<details><summary>Answer</summary>On modern hardware: one extra indirection (read the vtable slot) plus a branch the predictor can't always nail. A few cycles at most, **but** the bigger cost is that the JIT can't inline a virtual call across the dispatch. Devirtualization restores inlining and is the real perf win.</details>

---

## Interfaces

**Q34.** What can an interface contain in modern C# (8+)?
<details><summary>Answer</summary>Abstract methods (default), default-implementation methods (C# 8), properties (auto-property syntax is not allowed — only `get/set` declarations), events, indexers, static members, **static abstract** members (C# 11), constants, nested types. Instance fields are **not** allowed.</details>

**Q35.** What's the difference between implicit and explicit interface implementation?
<details><summary>Answer</summary>**Implicit**: ordinary public method matching the interface signature; callable through both the class reference and the interface reference. **Explicit**: declared as `void IFoo.M() {}`, only callable through the interface — invisible on the class type. Used to (a) disambiguate two interfaces with the same method, (b) hide an interface member that doesn't fit the class's main API.</details>

**Q36.** What is a "default interface method"?
<details><summary>Answer</summary>C# 8 feature: an interface method with a body. Implementors **may** override, but if they don't, they inherit the default. Designed for evolving interfaces without breaking existing implementations. Default methods are dispatched virtually but live on the interface, not the class.</details>

**Q37.** What is `static abstract` on an interface member?
<details><summary>Answer</summary>C# 11. Lets generic code call static methods through a type parameter constrained to an interface: `where T : INumber<T>`. Underpins the generic-math feature — `T.Parse`, `T.Zero`, `T.operator +`.</details>

**Q38.** Why does `IDisposable` not have a default `Dispose` implementation in C# 8+?
<details><summary>Answer</summary>Default implementations are a tool for **evolving** an interface without breaking implementors. `IDisposable.Dispose` has always required a real implementation — there's no sensible default. Default methods aren't a license to make interfaces stateful.</details>

---

## Abstract vs interface

**Q39.** When should you reach for an abstract class instead of an interface?
<details><summary>Answer</summary>When you have shared **state** or shared implementation that subclasses inherit, and the relationship is a clear "is-a". Use interfaces for **capabilities** ("can be disposed", "can be compared"). The two compose — a common pattern is `interface IFoo` plus `abstract class FooBase : IFoo` providing a default implementation.</details>

**Q40.** Can an abstract class be instantiated?
<details><summary>Answer</summary>No — direct instantiation is a compile error. Only concrete subclasses can be `new`-ed. The abstract class can still have constructors, called from derived constructors via `: base(...)`.</details>

**Q41.** Can an interface have a constructor?
<details><summary>Answer</summary>No. Interfaces have no instance state, so there's nothing to construct. (They can have static constructors via static members in C# 11+.)</details>

---

## Encapsulation

**Q42.** Why is exposing a `List<T>` from a class often a mistake?
<details><summary>Answer</summary>The caller can `Add`, `Remove`, `Clear` — bypassing every invariant your class was meant to enforce. Either return `IReadOnlyList<T>`, return a copy, or expose specific operations (`Add`, `Remove`) that go through your validation.</details>

**Q43.** What does "defensive copying" mean?
<details><summary>Answer</summary>Making a private copy of mutable input (or output) so external code can't mutate it after the fact. Example: a constructor receives a `byte[]`; you `Array.Copy` it into a private field. The caller can scramble their original array; your invariant survives.</details>

**Q44.** What's the trade-off in using `internal` instead of `public` for a type?
<details><summary>Answer</summary>`internal` limits the type to your assembly — no consumer can take a dependency on it. Wins: refactor freely, narrower API surface. Losses: tests in another assembly need `[InternalsVisibleTo]`; the type isn't reusable from outside.</details>

---

## Nested and partial

**Q45.** When does it make sense to nest one type inside another?
<details><summary>Answer</summary>When the nested type is genuinely only meaningful to the outer (e.g., a builder, a state machine, a private state enum). The nesting communicates the relationship and grants privileged access to the outer's private members.</details>

**Q46.** Can a nested type access the outer's private members?
<details><summary>Answer</summary>Yes — and vice-versa. Two nested types **sibling** to each other also see each other's privates. This is a compile-time rule; the runtime treats them as ordinary types.</details>

**Q47.** What's the original form of a partial method, and why might it have zero overhead?
<details><summary>Answer</summary>Implicit `private`, `void` return, no `out` parameters. If never implemented, **all call sites are removed** at compile time — including their arguments. Source generators use this to emit "hook" calls that vanish unless the user opts in.</details>

**Q48.** What's the C# 9+ extended form of a partial method?
<details><summary>Answer</summary>When the partial method has an access modifier or a non-void return type, an implementation is **required**. Used by `[GeneratedRegex]`, `[LoggerMessage]`, and JSON source generation.</details>

---

## Primary constructors

**Q49.** Where do primary-constructor parameters live in the type?
<details><summary>Answer</summary>In scope for the entire type body. If any instance member references them, the compiler emits a private capture field; if no member outside the constructor uses them, no field is generated.</details>

**Q50.** Are primary-constructor parameters readonly?
<details><summary>Answer</summary>No — they're mutable. The capture field is `initonly` only if the parameter is never reassigned anywhere in the body. Mutating one mutates the shared capture.</details>

**Q51.** How does a primary constructor differ on a `class` vs a `record`?
<details><summary>Answer</summary>On a `class`, the compiler synthesizes only the constructor (plus capture fields on demand). On a `record`, the compiler additionally synthesizes public `init`-only properties for each parameter, value-based `Equals`/`GetHashCode`, `ToString`, `Deconstruct`, copy constructor for `with`-expressions, and `==`/`!=` operators.</details>

**Q52.** Predict:
```csharp
class Box(int size) {
    public Box() : this(0) { }
    public int Size => size;
}
Console.WriteLine(new Box().Size);
```
<details><summary>Answer</summary>`0`. The parameterless constructor must chain to the primary via `: this(0)`. The primary captures `size = 0` into the hidden field, and `Size` returns it.</details>

**Q53.** Why does the compiler warn (CS9124) on this code?
```csharp
class Person(string name) {
    private string name = name;
}
```
<details><summary>Answer</summary>Double storage. The compiler emits a capture field for the parameter (anywhere `name` is used elsewhere) **and** the explicit field — two slots for what looks like the same value. Rename to `_name` to make the intent clear.</details>

---

## Mixed / tricky

**Q54.** Predict and explain:
```csharp
class A { public A() => Console.Write("A"); }
class B : A { public B() => Console.Write("B"); }
class C : B { public C() => Console.Write("C"); }
new C();
```
<details><summary>Answer</summary>`ABC`. Base constructors run before derived. Construction order is most-base to most-derived, regardless of inheritance depth.</details>

**Q55.** Predict:
```csharp
interface IA { void M() => Console.Write("IA"); }
interface IB : IA { void IA.M() => Console.Write("IB"); }
class C : IB { }
((IA)new C()).M();
```
<details><summary>Answer</summary>`IB`. `IB` provides an explicit override of `IA.M`. When called through `IA`, runtime dispatch picks the most-derived interface implementation — that's `IB`. Default interface methods support inheritance and override.</details>

**Q56.** What's wrong with this design?
```csharp
public class User {
    public string Name { get; set; }
    public List<Order> Orders { get; set; } = new();
}
```
<details><summary>Answer</summary>Two encapsulation leaks: (1) `Name` has a public setter — anyone can blank it. Prefer `init` or no setter plus a method like `Rename(...)`. (2) `Orders` exposes a mutable list. Consumers can `Clear` or `Add` directly. Use `IReadOnlyList<Order>` for the public surface and a private `AddOrder(...)` for controlled mutation.</details>

**Q57.** How is interface dispatch implemented in the CLR (high level)?
<details><summary>Answer</summary>Two-step lookup. Each object has a type handle pointing to its method table. For interface calls, the runtime locates the type's "interface map" — an array recording which class slots implement each interface slot. The first call typically populates a per-call-site cache (a "stub"). Subsequent calls hit the cache directly. Slower than virtual class dispatch, but JIT optimizations narrow the gap.</details>

**Q58.** When does the JIT inline across a virtual call?
<details><summary>Answer</summary>When it can devirtualize — i.e., prove the concrete type. Triggers: `sealed` class or method, value types (static dispatch), tiered compilation seeing only one type at the call site (PGO-guided devirtualization). Once devirtualized, normal inlining heuristics apply.</details>

**Q59.** Predict:
```csharp
abstract class Shape { public abstract double Area { get; } }
class Square(double side) : Shape { public override double Area => side * side; }
Shape s = new Square(3);
Console.WriteLine(s.Area);
```
<details><summary>Answer</summary>`9`. Primary-constructor parameter `side` is captured in a private field; `Area` reads it. Virtual property dispatched through the `Shape` reference resolves to `Square.Area`.</details>

**Q60.** What's the difference between `class Foo {}` and `sealed class Foo {}` for the JIT?
<details><summary>Answer</summary>`sealed` tells the JIT no derived type exists. Virtual calls on `Foo` references can be devirtualized — and once devirtualized, inlined. For hot leaf types, sealing is a real performance optimization, not just a design hint.</details>

---

→ [Coding.md](Coding.md) — hands-on problems
