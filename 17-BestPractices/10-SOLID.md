# SOLID Principles

## What they are

SOLID is five object-oriented design principles that make code easier to change, test, and extend. They're not C#-specific, but C# features (interfaces, DI, generics, records) make them natural to apply. The goal isn't dogma — it's reducing the cost of change.

| Letter | Principle |
|---|---|
| **S** | Single Responsibility Principle |
| **O** | Open/Closed Principle |
| **L** | Liskov Substitution Principle |
| **I** | Interface Segregation Principle |
| **D** | Dependency Inversion Principle |

Apply them with judgment — over-applying (especially abstraction-everywhere) produces its own complexity. SOLID is a guide, not a checklist to maximize.

---

## S — Single Responsibility Principle

> A class should have one reason to change.

A class should do one thing. When a class handles persistence *and* validation *and* notification, a change to any one concern risks the others, and the class becomes hard to test.

```csharp
// ✗ — three responsibilities: persistence, email, formatting
public class OrderService {
    public void PlaceOrder(Order o) {
        // validate
        // save to database
        // send confirmation email
        // write to log file
    }
}

// ✓ — each class has one reason to change
public class OrderValidator { public ValidationResult Validate(Order o) { ... } }
public class OrderRepository { public void Save(Order o) { ... } }
public class OrderNotifier { public void SendConfirmation(Order o) { ... } }

public class OrderService(OrderValidator validator, OrderRepository repo, OrderNotifier notifier) {
    public void PlaceOrder(Order o) {
        validator.Validate(o);
        repo.Save(o);
        notifier.SendConfirmation(o);   // orchestrates; doesn't implement each concern
    }
}
```

The orchestrating class (`OrderService`) still exists — SRP doesn't mean "tiny classes everywhere," it means **cohesion**: things that change together live together; things that change for different reasons are separated.

---

## O — Open/Closed Principle

> Open for extension, closed for modification.

You should be able to add new behavior without editing existing, tested code. The enemy is the growing `switch`/`if` chain that you must edit for every new case.

```csharp
// ✗ — every new shape forces editing this method (and re-testing it)
public decimal Area(Shape s) {
    switch (s.Type) {
        case "circle": return Math.PI * s.Radius * s.Radius;
        case "square": return s.Side * s.Side;
        // add triangle → edit here → risk breaking circle/square
    }
}

// ✓ — add a new shape by adding a class; existing code untouched
public abstract record Shape { public abstract decimal Area(); }
public record Circle(double Radius) : Shape { public override decimal Area() => (decimal)(Math.PI * Radius * Radius); }
public record Square(double Side) : Shape { public override decimal Area() => (decimal)(Side * Side); }
public record Triangle(double Base, double Height) : Shape { public override decimal Area() => (decimal)(0.5 * Base * Height); }
// New shape = new file; Area() callers unchanged.
```

Polymorphism (virtual methods, interfaces) and the strategy pattern are the usual tools. **Caveat**: don't pre-abstract for hypothetical future cases — apply OCP where variation is real or expected. A `switch` over a *closed, stable* set (e.g., an enum that won't grow) is fine and often clearer; switch expressions with exhaustiveness are a legitimate alternative.

---

## L — Liskov Substitution Principle

> Subtypes must be substitutable for their base types without breaking correctness.

A derived class must honor the contract of its base — same expectations, no surprises. If code using a base type breaks when given a subtype, LSP is violated.

```csharp
// ✗ — the classic violation: Square "is-a" Rectangle breaks the contract
public class Rectangle {
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
}
public class Square : Rectangle {
    public override int Width { set { base.Width = base.Height = value; } }  // surprises callers
    public override int Height { set { base.Width = base.Height = value; } }
}

void Test(Rectangle r) {
    r.Width = 5; r.Height = 4;
    Assert.Equal(20, r.Width * r.Height);   // fails for Square (gives 16) — LSP broken
}
```

Symptoms of LSP violations:
- Overrides that throw `NotSupportedException` for inherited members.
- Subtypes that strengthen preconditions or weaken postconditions.
- Callers that type-check (`if (x is SpecificType)`) to handle subtypes specially.

Fix: model the relationship correctly (a `Square` isn't a substitutable `Rectangle`; use composition or a common `IShape`). Prefer **composition over inheritance** when "is-a" doesn't truly hold. Records and sealed types reduce accidental LSP traps.

---

## I — Interface Segregation Principle

> Clients shouldn't depend on methods they don't use.

Prefer many small, focused interfaces over one fat interface. A fat interface forces implementers to provide (or stub) members they don't need, and couples clients to methods they ignore.

```csharp
// ✗ — fat interface; a read-only consumer is forced to depend on write methods
public interface IRepository<T> {
    T Get(int id);
    IReadOnlyList<T> GetAll();
    void Add(T item);
    void Update(T item);
    void Delete(int id);
}

// ✓ — segregated; clients depend only on what they use
public interface IReadRepository<T> {
    T Get(int id);
    IReadOnlyList<T> GetAll();
}
public interface IWriteRepository<T> {
    void Add(T item);
    void Update(T item);
    void Delete(int id);
}
// A reporting service depends on IReadRepository<T> only.
```

Small interfaces also make mocking and testing easier (fewer members to set up). The BCL models this well: `IReadOnlyList<T>`, `ICollection<T>`, `IList<T>` are a capability ladder, not one giant interface (see [Chapter 07 §12](../07-Collections/12-IEnumerableHierarchy.md)).

---

## D — Dependency Inversion Principle

> Depend on abstractions, not concretions. High-level modules shouldn't depend on low-level details.

Business logic should depend on an interface, not a concrete database/email/HTTP class. This decouples policy from mechanism and makes both testable and swappable.

```csharp
// ✗ — high-level OrderService directly depends on a concrete SQL class
public class OrderService {
    private readonly SqlOrderRepository _repo = new();   // hard dependency on a detail
}

// ✓ — depends on an abstraction; the concrete type is injected
public interface IOrderRepository { void Save(Order o); }
public class SqlOrderRepository : IOrderRepository { ... }      // a detail
public class InMemoryOrderRepository : IOrderRepository { ... } // for tests

public class OrderService(IOrderRepository repo) {              // depends on abstraction
    public void Place(Order o) => repo.Save(o);
}
```

DIP is the foundation of dependency injection — the abstraction (`IOrderRepository`) is owned by the high-level module, and the concrete implementation is supplied from outside (the composition root). See [12-DependencyInjection.md](12-DependencyInjection.md). It's also what makes the code testable: inject a fake repository in tests (see [Chapter 16 §03](../16-Testing/03-Mocking.md)).

---

## SOLID in practice — and when to relax

SOLID principles reinforce each other: DIP needs interfaces (ISP), substitutability (LSP) keeps polymorphism (OCP) honest, and SRP keeps each piece small enough to do all this.

But **don't over-apply**:
- Not every class needs an interface. Introduce an abstraction when you have a real second implementation, a test seam, or a stable contract — not speculatively. Premature interfaces add indirection for no benefit.
- A small, cohesive class with multiple closely-related methods isn't an SRP violation.
- A stable, closed `switch`/enum is fine; don't force polymorphism on a set that won't grow.
- "One class, one method" is a misreading of SRP that produces fragmented, hard-to-follow code.

The metric that matters: **does the design make change cheap and safe?** SOLID is a means to that end, not the end itself.

---

## Common bugs / gotchas

### Over-abstraction ("interface for everything")

Wrapping every class in a one-implementation interface adds indirection, hurts navigability, and provides no benefit. Abstract when there's a reason (test seam, real variation, contract boundary).

### LSP violations via `NotSupportedException`

A subtype that throws for an inherited member breaks substitutability. Re-model the hierarchy (composition, narrower interfaces).

### Fat interfaces that grow forever

An `IService` with 30 methods couples everyone to everything. Segregate by role/capability.

### "SRP" used to justify anemic models

Splitting all behavior out of an entity into "services" can produce an anemic domain model (see [09-CommonAntiPatterns.md](09-CommonAntiPatterns.md)). Cohesive behavior belongs with its data; SRP is about *reasons to change*, not removing all logic from entities.

---

## Summary

- **SRP** — one reason to change; separate concerns, but keep cohesive things together.
- **OCP** — extend without modifying; use polymorphism/strategy where variation is real (a stable enum `switch` is fine).
- **LSP** — subtypes must honor the base contract; prefer composition when "is-a" doesn't truly hold.
- **ISP** — many small interfaces over one fat one; depend only on what you use.
- **DIP** — depend on abstractions you own; inject concretions — the basis of DI and testability.
- Apply with judgment: SOLID's purpose is cheap, safe change — don't over-abstract for hypothetical needs.

→ Next: [11-DesignPatterns.md](11-DesignPatterns.md)
