# Polymorphism

## What it is

**Polymorphism** ("many forms") lets a single piece of code work with values of multiple types, and have the **right** behavior automatically chosen for each.

In C#, polymorphism comes in several flavors:
1. **Subtype polymorphism** — `virtual` + `override` (runtime dispatch).
2. **Ad-hoc polymorphism** — overloading (compile-time dispatch on argument types).
3. **Parametric polymorphism** — generics (one body, many type parameters).
4. **Interface polymorphism** — calling code via an interface; dispatch to whatever implements it.

This file focuses on (1) — runtime polymorphism via virtual dispatch — because it's the most subtle and the one that creates the famous "interview question of override vs new."

```csharp
Animal a = new Dog();
a.Speak();          // calls Dog.Speak — runtime polymorphism in action
```

---

## The classic example

```csharp
public class Animal {
    public virtual string Speak() => "generic sound";
}

public class Dog : Animal {
    public override string Speak() => "woof";
}

public class Cat : Animal {
    public override string Speak() => "meow";
}

void TellEach(IEnumerable<Animal> animals) {
    foreach (var a in animals)
        Console.WriteLine(a.Speak());
}

TellEach(new Animal[] { new Dog(), new Cat(), new Dog() });
// woof, meow, woof
```

`TellEach` doesn't know about Dog or Cat. It calls `a.Speak()`; the runtime picks the right override for each.

---

## `override` vs `new` — the most common trap

```csharp
public class Base {
    public virtual string M() => "Base.M";
}

public class DerivedOverride : Base {
    public override string M() => "DerivedOverride.M";
}

public class DerivedNew : Base {
    public new string M() => "DerivedNew.M";
}

Base b1 = new DerivedOverride();
Base b2 = new DerivedNew();

Console.WriteLine(b1.M());   // "DerivedOverride.M"
Console.WriteLine(b2.M());   // "Base.M"  ← !
```

The difference:
- **`override`** — derived class's method goes into the same vtable slot. Runtime dispatch picks it.
- **`new`** — a brand-new method. The base's vtable slot is untouched. The new method is only visible when the variable's declared type is `DerivedNew`.

```csharp
DerivedNew d = new DerivedNew();
Console.WriteLine(d.M());   // "DerivedNew.M" — declared type is DerivedNew now
```

**Rule**: use `override` 99% of the time. `new` is a code smell unless you have a specific reason (rare backward-compat scenarios in libraries).

---

## How virtual dispatch works at runtime

(This is the "internals" payoff.)

Each class with virtual methods has a **method table** (MT). The MT contains a **vtable**: an array of pointers to the method implementations for each virtual slot.

```
Base's vtable:                  DerivedOverride's vtable:
[0] Object.Equals (inherited)   [0] Object.Equals (inherited)
[1] Object.GetHashCode          [1] Object.GetHashCode
[2] Object.ToString             [2] Object.ToString
[3] Base.M                       [3] DerivedOverride.M    ← override replaced slot
```

When you call `b1.M()`:
1. The compiler emits `callvirt instance string Base::M()`.
2. The runtime loads `b1`'s MT pointer (offset 8 in the object header on 64-bit).
3. The MT points to **DerivedOverride's MT** (because `b1` is actually a DerivedOverride).
4. The runtime reads vtable slot 3 → `DerivedOverride.M`.
5. Calls it.

For `b2.M()` (where `b2` is a `DerivedNew`):
1. Same compiled instruction: `callvirt instance string Base::M()`.
2. `b2`'s MT is `DerivedNew`'s MT.
3. DerivedNew's MT has the same `Base.M` in slot 3 — DerivedNew didn't override it.
4. Slot 3 → `Base.M`.
5. Calls it.

DerivedNew.M is in DerivedNew's vtable somewhere else (its own slot), reachable only when the declared type is `DerivedNew`.

---

## Polymorphism with abstract methods

Abstract methods are virtual with no body. Subclasses must override:

```csharp
public abstract class Shape {
    public abstract double Area();
}

public class Circle : Shape {
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
}

public class Square : Shape {
    public double Side { get; init; }
    public override double Area() => Side * Side;
}

Shape[] shapes = { new Circle { Radius = 1 }, new Square { Side = 2 } };
foreach (var s in shapes)
    Console.WriteLine(s.Area());
// 3.14159...
// 4
```

Each subclass provides its own Area; the loop iterates polymorphically.

---

## Interface polymorphism

Interfaces let you swap implementations without inheritance:

```csharp
public interface IGreeter {
    string Greet(string name);
}

public class FormalGreeter : IGreeter {
    public string Greet(string name) => $"Good day, {name}";
}

public class CasualGreeter : IGreeter {
    public string Greet(string name) => $"Hey, {name}!";
}

void TestGreeter(IGreeter g) {
    Console.WriteLine(g.Greet("Alice"));
}
TestGreeter(new FormalGreeter());
TestGreeter(new CasualGreeter());
```

Internally, interface dispatch is similar to virtual dispatch but uses an **interface dispatch map** instead of the regular vtable. The runtime walks the implemented interfaces, finds the matching slot, and calls. Slightly more expensive than virtual dispatch on a class, but the JIT can devirtualize many cases.

[Chapter 02 §08](08-Interfaces.md) covers interfaces deeply.

---

## Covariant return types

An override may return a more-derived type:

```csharp
public class Animal {
    public virtual Animal Clone() => new Animal();
}

public class Dog : Animal {
    public override Dog Clone() => new Dog();
}

Dog d = new Dog().Clone();    // Returns Dog — no cast needed
```

Useful for fluent factory methods. Pre-C# 9 this required casting.

---

## Operator overloading (ad-hoc polymorphism)

User-defined operators that work like polymorphism, but at compile time based on operand types:

```csharp
public readonly record struct Money(decimal Amount, string Currency) {
    public static Money operator +(Money a, Money b) {
        if (a.Currency != b.Currency) throw new InvalidOperationException();
        return new(a.Amount + b.Amount, a.Currency);
    }
}

var sum = new Money(10m, "USD") + new Money(20m, "USD");
```

The compiler picks the right `+` based on operand types. Common operators: `+`, `-`, `*`, `/`, `==`, `!=`, `<`, `>`, `<=`, `>=`, `++`, `--`, `!`, `~`.

C# 11+ adds:
- **Static abstract members in interfaces** — interfaces can require static methods/operators, used by generic math.
- **Checked operators** — `checked operator +` for explicit overflow handling.

C# 14 adds **user-defined compound assignment operators** — `+=` can be overridden separately from `+`.

[Chapter 04 §05](../04-Generics/05-StaticAbstractMembers.md) and [Chapter 04 §06](../04-Generics/06-GenericMath.md) cover the modern arithmetic patterns.

---

## Common polymorphism patterns

### Visitor pattern (manual)

Inverts the dispatch — the caller's type drives behavior:

```csharp
public abstract class Shape {
    public abstract T Accept<T>(IShapeVisitor<T> visitor);
}
public class Circle : Shape {
    public override T Accept<T>(IShapeVisitor<T> v) => v.VisitCircle(this);
}
public class Square : Shape {
    public override T Accept<T>(IShapeVisitor<T> v) => v.VisitSquare(this);
}

public interface IShapeVisitor<T> {
    T VisitCircle(Circle c);
    T VisitSquare(Square s);
}
```

Used in compilers (AST traversal) and parsers. With modern pattern matching, often replaced by a switch expression.

### Strategy via interfaces

```csharp
public interface IPricingStrategy {
    decimal Apply(decimal subtotal);
}

public class NoDiscount : IPricingStrategy {
    public decimal Apply(decimal s) => s;
}
public class TenPercentOff : IPricingStrategy {
    public decimal Apply(decimal s) => s * 0.9m;
}

decimal Pay(decimal subtotal, IPricingStrategy strat) => strat.Apply(subtotal);
```

Swap strategies at runtime by passing a different implementation.

### Method dispatch via pattern matching

For when polymorphism feels too heavy:

```csharp
double Area(Shape s) => s switch {
    Circle c => Math.PI * c.Radius * c.Radius,
    Square q => q.Side * q.Side,
    _ => throw new InvalidOperationException()
};
```

Trade-off: data and behavior split. Polymorphism keeps them together (Area method on Shape). Switch can be cleaner when behavior lives outside the type hierarchy.

---

## When polymorphism shines vs hurts

**Shines when**:
- You have several types sharing a contract, with different implementations.
- The contract is **stable** — adding methods to the base affects all subclasses.
- Callers want to write loops/operations that don't care which subtype they have.

**Hurts when**:
- The "shared" behavior keeps being overridden — that's a sign of LSP violation.
- The hierarchy is brittle — adding a type requires modifying many places.
- You're using polymorphism for **type discrimination** — `if (obj is Dog) ... else if (obj is Cat) ...` — that's a code smell. Use proper dispatch or pattern matching.

---

## Internals — the full vtable picture

Let's trace one more example.

```csharp
class Shape {
    public virtual double Area() => 0;
    public virtual string Describe() => $"Shape, area={Area()}";
}

class Square : Shape {
    public double Side { get; init; }
    public override double Area() => Side * Side;
    // Inherits Describe()
}
```

Shape's vtable (showing virtual slots only):
```
[0] System.Object.Equals
[1] System.Object.GetHashCode
[2] System.Object.ToString
[3] Shape.Area
[4] Shape.Describe
```

Square's vtable:
```
[0] System.Object.Equals             (inherited)
[1] System.Object.GetHashCode        (inherited)
[2] System.Object.ToString           (inherited)
[3] Square.Area                      (override)
[4] Shape.Describe                   (inherited)
```

When you do `Square s = new(); s.Describe();`:
1. `callvirt Shape::Describe()` → uses Square's MT → slot 4 → `Shape.Describe`.
2. Inside `Shape.Describe`, the IL says `Area()`. That's `callvirt this.Area()`.
3. `this` is the Square, so it goes to slot 3 → `Square.Area`.

So `Shape.Describe()` running on a `Square` calls `Square.Area()` automatically — even though `Describe` is the Shape version. That's polymorphism.

---

## JIT optimizations

The JIT looks for opportunities to **devirtualize** — replace `callvirt` with a direct `call`:

1. **Sealed types**: `class Dog : Animal { ... } sealed class GoldenRetriever : Dog { ... }` — calls on a `GoldenRetriever` are devirtualized.
2. **Value types**: structs are implicitly sealed; calls are direct.
3. **Generic constraints**: `where T : Animal` doesn't help; `where T : struct` does (devirtualized per T).
4. **PGO**: if profiling sees one subtype dominate, the JIT emits a fast path for it.
5. **Inlining**: small methods called once may be inlined entirely.

These optimizations are why **`sealed` matters for hot code**: it gives the JIT proof.

---

## Common bugs

- **Using `new` to "fix" a base method** — the base reference still sees the old behavior. Use `override`.
- **Calling virtual methods in constructors** — the derived override runs before derived fields are initialized.
- **Polymorphism without LSP** — `Square : Rectangle` with overridden setters breaks `Test(Rectangle)` callers.
- **Catching too broadly** — `catch (Exception)` polymorphically catches everything including bugs.
- **Equality breaking** — overriding `Equals` for derived classes can violate symmetry. Use records for value equality, or `GetType() == other.GetType()` guard.

---

## Polymorphism vs generic dispatch

```csharp
// Polymorphism (virtual)
double TotalArea(IEnumerable<Shape> shapes) =>
    shapes.Sum(s => s.Area());     // s.Area() is virtual

// Generic dispatch (static, monomorphized for value types)
double TotalArea<T>(IEnumerable<T> items) where T : Shape =>
    items.Sum(t => t.Area());      // still virtual — T is a class
```

Both work polymorphically. Generics shine more for **structs** and primitives where the JIT can specialize and eliminate boxing.

---

## When to use which polymorphism

| Need | Use |
|---|---|
| Loops over different subtypes | `virtual` / abstract methods |
| Dispatch on arg types at compile time | Overloading |
| Type-safe containers / algorithms | Generics |
| Swap implementations at runtime | Interfaces + DI |
| Behavior-rich data | Polymorphism on the type |
| Behavior-poor data | Pattern matching in a switch |

→ Next: [08-Interfaces.md](08-Interfaces.md)
