# Abstract Class vs Interface

> One of the most common interview questions. The honest answer: they overlap a lot. The art is choosing the right one for the design.

---

## The contracts

|  | Abstract class | Interface |
|---|---|---|
| Multiple inheritance | No (single base) | Yes (any number) |
| Fields | Yes (any access) | No instance fields; static OK since C# 8 |
| Constructors | Yes — derived must call | No |
| State | Yes | No |
| Access modifiers on members | Yes (public, protected, etc.) | Public by default; explicit form has no modifier |
| Default implementations | Yes (concrete methods alongside abstract) | Default methods since C# 8, with caveats |
| Static abstract members | No (regular static members fine) | Yes since C# 11 (key for generic math) |
| Can be instantiated | No | No |
| Inherits from | One class (chain to `Object`) | Multiple interfaces |
| Versioning safe | Adding members → breaks subclasses | Adding members → breaks implementers (unless default-method) |

---

## The basic mental model

> **Interface** answers: "what can this thing do?"
> **Abstract class** answers: "what is this thing, and what does it share?"

```csharp
public interface IPrintable {            // capability
    void Print();
}

public abstract class Document {          // shared identity
    public string Title { get; init; } = "";
    public DateTime CreatedAt { get; init; }
    public abstract void Render();
}

public class Pdf : Document, IPrintable {
    public override void Render() { /* ... */ }
    public void Print() { /* ... */ }
}
```

`Document` defines *what* Pdf is — title, creation date, virtual render method. `IPrintable` is a *capability* Pdf chooses to support.

---

## When to choose which

### Use an interface when...

- Multiple unrelated types need to satisfy the same contract.
- You want testability (DI + mocks).
- You're crossing assembly boundaries with versioning concerns.
- The "thing" is a capability (`IDisposable`, `IComparable`, `ICloneable`).
- You want generic math or static abstract members (C# 11+).

### Use an abstract class when...

- You have **shared state** (fields) that all subclasses need.
- You have **shared method implementations** that subclasses can override or call.
- The relationship is genuine "is-a" — not "can-do."
- Default behavior should be reusable, with hooks for specialization.
- You're building a Template Method pattern.

### Use both when...

A common, healthy layered design:
- **Interface** for the public contract — that's what callers depend on.
- **Abstract base class** that implements the interface + provides reusable defaults — subclasses inherit it for free.

```csharp
public interface IRepository<T> {
    Task<T?> GetAsync(int id);
    Task SaveAsync(T item);
}

public abstract class RepositoryBase<T> : IRepository<T> {
    protected DbContext Db { get; }
    protected RepositoryBase(DbContext db) { Db = db; }

    public abstract Task<T?> GetAsync(int id);
    public virtual Task SaveAsync(T item) {
        Db.Add(item);
        return Db.SaveChangesAsync();
    }
}

public class UserRepository : RepositoryBase<User> {
    public UserRepository(DbContext db) : base(db) { }
    public override Task<User?> GetAsync(int id) => Db.Users.FindAsync(id).AsTask();
}
```

Callers depend on `IRepository<User>`. Implementations get free `SaveAsync` from the base, override what they need.

---

## A worked example: shapes

### Option A — interface only

```csharp
public interface IShape {
    double Area();
    double Perimeter();
}

public class Circle : IShape {
    public double Radius { get; init; }
    public double Area() => Math.PI * Radius * Radius;
    public double Perimeter() => 2 * Math.PI * Radius;
}

public class Square : IShape {
    public double Side { get; init; }
    public double Area() => Side * Side;
    public double Perimeter() => 4 * Side;
}
```

Pros: minimal, every shape is self-contained.
Cons: shared logic (e.g., `Describe()`) has to be reimplemented or made an extension method.

### Option B — abstract class only

```csharp
public abstract class Shape {
    public string Name { get; init; } = "";
    public abstract double Area();
    public abstract double Perimeter();
    public virtual string Describe() => $"{Name}: area={Area():F2}, perim={Perimeter():F2}";
}

public class Circle : Shape {
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
    public override double Perimeter() => 2 * Math.PI * Radius;
}
```

Pros: shared `Describe`, shared `Name`. No duplication.
Cons: can't say `interface IFlattenable` or `IDrawable` simultaneously.

### Option C — both (recommended for non-trivial)

```csharp
public interface IShape {
    double Area();
    double Perimeter();
}

public abstract class ShapeBase : IShape {
    public string Name { get; init; } = "";
    public abstract double Area();
    public abstract double Perimeter();
    public override string ToString() => $"{Name}: area={Area():F2}, perim={Perimeter():F2}";
}

public class Circle : ShapeBase {
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
    public override double Perimeter() => 2 * Math.PI * Radius;
}
```

Now:
- Callers can depend on `IShape` (no inheritance lock-in).
- Subclasses inherit shared `Name` and `ToString`.
- Future shapes can implement `IShape` directly or inherit `ShapeBase`.

---

## Default interface methods (DIM) — does it change the calculus?

C# 8 introduced default methods on interfaces. Do they replace abstract classes?

**Short answer: no.** Default methods exist to **let interface authors evolve their contracts** without breaking existing implementers. They aren't a substitute for an abstract base class.

Differences vs abstract class:
- DIM has **no instance state** (interfaces don't have instance fields).
- DIM dispatch goes through the interface — only callable through the interface, not via the implementing class directly (without explicit interface call).
- Reasoning about DIM with multiple inheritance is harder (ambiguity rules).

DIM is great for:
- Adding a convenience overload (`LogError(string)` that calls `Log(level, string)`).
- Backward-compatible API evolution.
- Cross-cutting concerns where there's no state.

Not great for:
- A genuine "base class" with state, fields, complex internal helpers.

---

## Static abstract members vs abstract base

C# 11+ lets interfaces have **static abstract members**:

```csharp
public interface IParseable<T> {
    static abstract T Parse(string s);
}

public class MyNumber : IParseable<MyNumber> {
    public static MyNumber Parse(string s) => new();
}

// Generic use
T ParseAndUse<T>(string s) where T : IParseable<T> => T.Parse(s);
```

This lets you put **static factory methods, operators, and constants** in a contract. Pre-C# 11, that required a separate factory pattern.

For an abstract class, you can have static members but they're not abstract — they can't be required of subclasses. Static abstract on interfaces fills that gap.

---

## Versioning consideration

This is subtle and often overlooked.

**Adding a member to an abstract class:**
- A new non-abstract method? Subclasses keep compiling. ✓
- A new abstract method? Subclasses break — they must implement it.
- A new field? OK.

**Adding a member to an interface:**
- A new member without a default? Every implementer breaks. ❌
- A new member **with** a default? Implementers keep compiling. ✓

So if you're shipping a library and adding a method:
- To an abstract class — add a `virtual` method with a default body.
- To an interface — add a default method.

In both cases, the goal is "non-breaking change."

---

## Choosing — a decision tree

```
Do you need to mix many capabilities into one type?
├── Yes → use INTERFACES
└── No → next question

Do you have shared state (fields) across subtypes?
├── Yes → use ABSTRACT CLASS
└── No → next question

Do you have shared *behavior* across subtypes?
├── Yes, identical for all: maybe a sealed base helper
├── Yes, varies: virtual + override in abstract class
└── No → INTERFACE only

Does the contract need to be testable / DI-friendly?
└── Always → expose an INTERFACE on top

Generic math / static contracts?
└── INTERFACE with static abstract (C# 11+)
```

---

## Internals — what's actually different at runtime

### Abstract class

A regular class with the `abstract` flag in metadata. The CLR throws if you try to instantiate one (`newobj` on an abstract type). Subclasses inherit fields, methods, the whole shape.

A virtual call on an abstract method works the normal way — via the vtable. The abstract slot just doesn't have an implementation; the subclass fills it.

### Interface

Doesn't have a "vtable" per se. Each implementing type has an **interface map** in its metadata. The runtime resolves interface calls by walking that map.

Multiple interface implementation is cheap — the map can hold many entries; instance memory doesn't grow.

### Performance

- Direct calls on a class: fastest.
- Virtual calls on a sealed class: devirtualized (direct).
- Virtual calls on an abstract class hierarchy: vtable lookup.
- Interface calls: interface-map lookup (slower than vtable, but the JIT optimizes hot paths, and PGO helps).
- Default interface method calls: interface-map lookup + redirect to interface body.

For most code, these differences don't matter. For tight inner loops, prefer non-virtual or sealed.

---

## Common bugs and anti-patterns

- **"Header" interfaces** — interfaces declared just to mirror one specific class. If only one implementer exists and ever will, the interface is overhead. Use the class directly until you actually need polymorphism.
- **Interface with one method named after the class** — `IOrderService` with `PlaceOrder`. Often clearer to just use a delegate or function.
- **Abstract class with no abstract members** — should probably be `sealed` (or just a normal class).
- **Choosing abstract class then needing multiple inheritance later** — refactor to interface + helper class. Avoid by exposing capability through interfaces from the start.
- **Default methods that depend on each other** — ambiguity hell. Keep DIMs simple and orthogonal.

---

## A pragmatic rule of thumb

For most application code:
- **Public-facing contracts** → interface.
- **Internal layered behavior** → abstract class (or just a regular class if not extension-friendly).
- **Capability mixins** → small interfaces (`IDisposable`, `IClonable`, etc.).
- **Don't double-interface** — if one interface and one class is enough, don't add a second interface just because.

For library design (NuGet packages, frameworks):
- Be more generous with interfaces — consumers will extend / mock more.
- Pair with default methods or abstract base implementations to make implementing easy.
- Think about versioning. Adding non-default interface members breaks the world.

→ Next: [10-Encapsulation.md](10-Encapsulation.md)
