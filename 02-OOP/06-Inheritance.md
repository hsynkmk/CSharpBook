# Inheritance

## What it is

A class can **inherit** from another, gaining its members. The new class is the **derived** (or child / subclass); the one inherited from is the **base** (or parent / superclass). C# supports **single inheritance** for classes — at most one direct base — plus multiple interface implementation.

```csharp
public class Animal {
    public string Name { get; init; } = "";
    public virtual string Speak() => "generic sound";
}

public class Dog : Animal {
    public string Breed { get; init; } = "";
    public override string Speak() => "woof";
}

var rex = new Dog { Name = "Rex", Breed = "Lab" };
Console.WriteLine(rex.Name);     // inherited
Console.WriteLine(rex.Speak());  // "woof"
```

`Dog : Animal` — colon syntax declares the base.

---

## Why it exists

- **Code reuse** — share fields and methods across related types.
- **Polymorphism** — handle different subtypes uniformly through a base reference.
- **Modeling "is-a" relationships** — `Dog is-a Animal`, `Manager is-a Employee`.

The trade-off: inheritance creates **tight coupling**. Many guides (including this book) now recommend **composition over inheritance** as the default, using inheritance sparingly when the "is-a" relationship is genuine and stable.

---

## The Object class

Every class implicitly inherits from `System.Object` (alias `object`):

```csharp
public class Foo { }   // implicitly Foo : object
```

`Object` provides:
- `Equals(object?)` — virtual, default reference equality.
- `GetHashCode()` — virtual, must agree with `Equals`.
- `ToString()` — virtual, default returns the type name.
- `GetType()` — non-virtual, returns the runtime `Type`.
- `ReferenceEquals(object?, object?)` — static, identity check.

You'll often override `ToString`:

```csharp
public class Point {
    public int X { get; init; }
    public int Y { get; init; }
    public override string ToString() => $"({X}, {Y})";
}
```

---

## Single inheritance for classes

A class has **one** direct base:

```csharp
class A { }
class B : A { }
class C : A, B { }     // ❌ — multiple class inheritance not allowed
```

But a class can implement **many** interfaces:

```csharp
class C : A, IDisposable, IComparable<C> { }   // ✓
```

Why single inheritance? To avoid the "diamond problem" — ambiguity when two base classes define the same method. C++ allows multiple inheritance and lives with the complexity; Java/C#/Kotlin/Swift/Rust all chose single + interfaces.

---

## Calling base members

Inside a derived class, `base` refers to the parent:

```csharp
public class Animal {
    public virtual string Speak() => "generic";
}

public class Dog : Animal {
    public override string Speak() {
        var baseImpl = base.Speak();    // call Animal.Speak()
        return $"{baseImpl} (specifically: woof)";
    }
}
```

`base.Method()` calls the **non-virtual** version of the parent's method (it bypasses vtable dispatch — useful when you want to extend rather than replace behavior).

`base` can also access fields (if `protected` or higher) and call the parent's constructor:

```csharp
public Dog(string name) : base(name) { }   // call Animal(name)
```

---

## Constructor inheritance

Constructors are **not** inherited automatically. Each class declares its own:

```csharp
public class Animal {
    public Animal(string name) { Name = name; }
    public string Name { get; }
}

public class Dog : Animal {
    public Dog(string name) : base(name) { }   // must explicitly forward
}
```

If the base has a parameterless constructor, the derived class can omit `: base()` — the compiler inserts it.

---

## Field access in derived classes

Derived classes can access **`protected`** members of the base. `private` is invisible:

```csharp
public class Animal {
    protected string _name = "";
    private int _age;
}

public class Dog : Animal {
    void M() {
        _name = "Rex";   // ✓ — protected
        _age = 5;        // ❌ — private to Animal
    }
}
```

---

## The `protected` modifier

`protected` extends access to the type **and its derived classes** (in any assembly):

```csharp
public class Base {
    protected void Helper() { }
}

public class Derived : Base {
    void M() { Helper(); }   // ✓
}

// In some other code:
Base b = new Derived();
b.Helper();   // ❌ — Helper is protected, callable only from inside the class hierarchy
```

`protected` is the standard way to expose extension points to subclasses without making them public.

---

## Designing for inheritance

Three philosophies:

### 1. Closed by default (recommended for most code)

```csharp
public sealed class Order { ... }
```

Make classes `sealed`. If someone needs to extend, they can ask, and you can design an extension point thoughtfully.

### 2. Open for extension

```csharp
public abstract class Shape {
    public abstract double Area();
    protected Shape() { /* ... */ }
}
```

Make the class abstract with a clear contract. Document what subclasses must implement.

### 3. Template Method pattern

Base defines algorithm skeleton; subclasses fill in steps:

```csharp
public abstract class ReportGenerator {
    public string Generate() {
        var data = LoadData();
        var formatted = Format(data);
        return Wrap(formatted);
    }
    protected abstract IList<object> LoadData();
    protected abstract string Format(IList<object> data);
    protected virtual string Wrap(string s) => $"<html><body>{s}</body></html>";
}
```

The base controls flow; subclasses customize specific operations.

---

## Composition vs inheritance — the modern preference

Inheritance creates a **strong, permanent** "is-a" relationship. If circumstances change, refactoring is expensive.

Composition uses **fields of other types** to reuse behavior:

```csharp
// Inheritance
public class TimestampedLogger : ConsoleLogger {
    public override void Log(string msg) => base.Log($"[{DateTime.Now}] {msg}");
}

// Composition
public class TimestampedLogger : ILogger {
    private readonly ILogger _inner;
    public TimestampedLogger(ILogger inner) { _inner = inner; }
    public void Log(string msg) => _inner.Log($"[{DateTime.Now}] {msg}");
}
```

The composition version:
- Decouples from the concrete `ConsoleLogger`.
- Lets you stack decorators: `new TimestampedLogger(new ColorLogger(new ConsoleLogger()))`.
- Doesn't fix the relationship at compile time.

Inheritance is right when you have:
- A genuine "is-a" relationship that's stable.
- Multiple subclasses sharing concrete implementation (not just an interface).

Composition is right when:
- You need flexibility (Decorator, Strategy, Composite patterns).
- The relationship might change.
- The "is-a" is fuzzy or arguable.

---

## Liskov Substitution Principle (LSP)

A core guideline: **subtypes must be substitutable for their base type without surprising the caller**.

```csharp
public class Rectangle {
    public virtual int Width { get; set; }
    public virtual int Height { get; set; }
    public int Area() => Width * Height;
}

public class Square : Rectangle {
    public override int Width { set { base.Width = value; base.Height = value; } }
    public override int Height { set { base.Width = value; base.Height = value; } }
}

void Test(Rectangle r) {
    r.Width = 5;
    r.Height = 4;
    Debug.Assert(r.Area() == 20);   // ⚠ fails when r is a Square
}
```

Square *is-a* Rectangle mathematically but **violates LSP** — its behavior breaks code that worked for Rectangle. The right model: don't inherit. Use a separate `Square` class or a common `Shape` interface.

---

## Internals — how inheritance is represented

Each type has a method table (MT) at runtime. Inheritance is encoded as:

- The MT contains a pointer to the **parent MT**.
- The vtable is **extended**: the derived class's vtable starts with the parent's slots, in order, optionally overridden, then appends new virtual slots.
- Field layout is also concatenated: parent's fields first, then derived's fields.

For:
```csharp
class Animal { public string Name; public int Age; }
class Dog : Animal { public string Breed; }
```

Memory layout of a `Dog`:
```
offset 0  : sync block
offset 8  : MT pointer (-> Dog's MT)
offset 16 : Name (from Animal)
offset 24 : Age  (from Animal)
offset 32 : Breed (from Dog)
```

Dog's MT:
```
parent MT -> Animal's MT
vtable[0] -> Animal.Equals (or overridden)
vtable[1] -> Animal.GetHashCode
vtable[2] -> Animal.ToString
vtable[3] -> Animal.Speak (or overridden by Dog)
...
```

### Casting

Casting up (`Dog → Animal`) is **free** — same reference, view through different MT pointer. Type tests (`is` / `as`) walk the MT parent chain to verify.

```csharp
Dog d = new Dog();
Animal a = d;        // free conversion
Dog d2 = (Dog)a;     // runtime check: a's MT or its ancestors == Dog's MT
```

The downcast emits a `castclass` IL instruction. Failure throws `InvalidCastException`. `as` emits `isinst` (no throw, returns null on mismatch).

### `base.Method()` in IL

```il
call instance string Animal::Speak()
```

A `call` (not `callvirt`) — explicitly bypasses virtual dispatch even on virtual methods. That's the runtime mechanism behind "base.Speak() always calls Animal.Speak()."

### Sealing and devirtualization

When the JIT can prove the receiver is `sealed` (or `Dog` and `Dog` is sealed, or a struct), it replaces `callvirt` with `call` — saving a memory load and an indirect branch.

---

## Common patterns

### Layered abstractions

```csharp
public abstract class Shape { public abstract double Area(); }
public abstract class Polygon : Shape { public abstract int Sides(); }
public class Triangle : Polygon {
    public override double Area() => /* ... */;
    public override int Sides() => 3;
}
```

Three levels — `Shape`, `Polygon`, `Triangle` — each adding a layer. Don't go deeper than 3-4 levels; gets unwieldy fast.

### Custom exception types

```csharp
public class DomainException : Exception {
    public DomainException(string msg) : base(msg) { }
}
public class NotFoundException : DomainException {
    public NotFoundException(string what) : base($"{what} not found") { }
}
```

Lets catchers be specific (`NotFoundException`) or broad (`DomainException`).

### Generic base with type-specific methods

```csharp
public abstract class Repository<T> where T : class {
    protected DbContext Db { get; }
    public Repository(DbContext db) { Db = db; }
    public abstract Task<T?> GetAsync(int id);
}

public class UserRepo : Repository<User> {
    public UserRepo(DbContext db) : base(db) { }
    public override Task<User?> GetAsync(int id) => Db.Users.FindAsync(id);
}
```

---

## Anti-patterns

### Inheritance for code reuse alone

```csharp
public class BaseService {
    protected ILogger Log => /* ... */;
    protected string CommonHelper() => /* ... */;
}

public class UserService : BaseService { /* ... */ }
public class OrderService : BaseService { /* ... */ }
```

If `UserService` and `OrderService` aren't really "kinds of" BaseService — they just happen to use the same helper — they shouldn't inherit. Use composition or extension methods.

### Deep hierarchies

```csharp
Animal → Mammal → DomesticatedMammal → Pet → Dog → Labrador → ChocolateLab
```

7 levels deep. Each adds little, the topmost layers can't change without breaking everyone, and "is-a" gets philosophically fuzzy. Prefer flat hierarchies + composition.

### Inheriting just to override one method

```csharp
public class CustomDictionary<K,V> : Dictionary<K,V> {
    public new V this[K key] => /* custom logic */;
}
```

Probably a mistake. Compose, decorate, or use a custom interface. Inheriting from BCL collection types specifically is rarely the right call.

---

## Common bugs

- **Calling virtual methods in the constructor** — derived override runs against a partially-constructed object.
- **Forgetting `base(...)`** — but only an error if base has no parameterless ctor.
- **`new` instead of `override`** — silently hides instead of replacing. Always prefer `override`.
- **Public fields in a base class** — derived classes can mutate them in inconsistent ways. Use properties.
- **Diamond-ish ambiguity via interfaces with default methods** — see [§08](08-Interfaces.md).

---

## When to use inheritance

| ✓ Good case | ✗ Bad case |
|---|---|
| Genuine "is-a" relationship | Just sharing helper methods |
| Stable shape over time | Likely to change drastically |
| 2-3 levels deep | 5+ levels deep |
| Subclass adds specific behavior | Subclass mostly overrides everything |
| Modeling a known domain (Shape → Circle/Rect) | Generic "Base*" class invented for refactoring |

→ Next: [07-Polymorphism.md](07-Polymorphism.md)
