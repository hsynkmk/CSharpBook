# Methods: Advanced

> Chapter 01 covered basic method declarations. Here we cover the OO-specific modifiers: `static`, `virtual`, `override`, `sealed`, `new`, `abstract`, `partial`, and how each interacts with the runtime.

---

## `static` methods

Belong to the type, not an instance. No `this`.

```csharp
public class MathHelpers {
    public static int Square(int n) => n * n;
}

int s = MathHelpers.Square(5);    // 25 — call on the type
```

Use when the method doesn't depend on instance state.

A class with only static members can itself be marked `static`:

```csharp
public static class StringExtensions {
    public static bool IsValidEmail(this string s) { ... }
}
```

Static classes:
- Cannot be instantiated (`new` is illegal).
- Cannot be inherited.
- All members must be static.
- Commonly hold extension methods, utility functions.

---

## `virtual` and `override` — polymorphism

By default, methods are **non-virtual** — the call site is bound at compile time based on the declared type.

To enable runtime dispatch (where the called method depends on the **actual** object type), mark with `virtual`. Subclasses then `override`:

```csharp
public class Animal {
    public virtual string Speak() => "generic sound";
}

public class Dog : Animal {
    public override string Speak() => "woof";
}

Animal a = new Dog();
Console.WriteLine(a.Speak());   // "woof" — virtual dispatch chose Dog.Speak
```

If `Animal.Speak` had been non-virtual, `a.Speak()` would have printed `"generic sound"` — bound to the declared type.

This is **polymorphism**, the topic of [§07](07-Polymorphism.md). Internals of the vtable lookup are explained there.

### Rules

- Only `virtual`, `abstract`, or `override` methods can be overridden.
- The override must have the **exact same signature** (modulo covariant return types since C# 9).
- The override's access modifier must match (with one exception: `protected` → `protected` etc.).
- You cannot override a `private` method (it's invisible to derived classes).

### Covariant return types (C# 9+)

An override can return a **more derived** type than the base:

```csharp
public class Animal {
    public virtual Animal Clone() => new Animal();
}

public class Dog : Animal {
    public override Dog Clone() => new Dog();   // Dog is more specific than Animal
}
```

Callers via `Dog` see a `Dog`; callers via `Animal` see an `Animal`. No `(Dog)` cast required.

---

## `abstract` methods

Declared without a body. The class must also be `abstract`. Subclasses **must** override to provide an implementation:

```csharp
public abstract class Shape {
    public abstract double Area();      // no body
}

public class Circle : Shape {
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
}

// new Shape();    // ❌ — can't instantiate abstract class
new Circle();      // ✓
```

Mixing concrete and abstract members is fine:

```csharp
public abstract class Shape {
    public string Name { get; init; } = "";
    public abstract double Area();              // must override
    public virtual string Describe() =>          // optional override
        $"{Name} has area {Area()}";
}
```

Abstract methods are implicitly `virtual` — they go through the vtable.

---

## `sealed`

Two uses:

### Sealed class — can't be inherited

```csharp
public sealed class String { ... }   // BCL: System.String

class MyString : String { }          // ❌ compile error
```

Sealing prevents extension. Benefits:
- The JIT can **devirtualize** all calls on the type (no vtable lookup).
- Prevents callers from accidentally depending on internals via inheritance.
- Signals "this is finished — don't extend it."

Default to sealed for leaf classes. The .NET design guidelines recommend "design for inheritance, or seal."

### Sealed override — overridden but no further override

```csharp
public class A {
    public virtual void Do() { }
}

public class B : A {
    public sealed override void Do() { ... }   // can't be overridden in subclasses of B
}

public class C : B {
    public override void Do() { ... }          // ❌
}
```

Useful when an intermediate class wants to lock down a method while still allowing the rest of the class to be extended.

---

## `new` — method hiding (not polymorphism)

`new` declares a method that **hides** an inherited method, rather than overriding it. The dispatch is by declared type, not runtime type:

```csharp
public class Animal {
    public virtual string Speak() => "generic";
}

public class Dog : Animal {
    public new string Speak() => "woof";   // hides, doesn't override
}

Animal a = new Dog();
Console.WriteLine(a.Speak());   // "generic" — DECLARED type Animal, hides ignored
Dog d = new Dog();
Console.WriteLine(d.Speak());   // "woof" — declared type Dog
```

**Almost always a bug or a code smell.** Common scenarios:
- You're maintaining backward compatibility and changing semantics (rare).
- You inherited from a third-party class and need a same-named method with different semantics (workaround).

If you want polymorphism, use `override`. If you want a different method, give it a different name.

---

## `partial` methods

Splits a method across multiple files (paired with `partial class`):

```csharp
// File 1: Generated.cs
partial class Customer {
    partial void OnNameChanged();    // declaration only

    public void SetName(string n) {
        _name = n;
        OnNameChanged();             // call — may or may not be implemented
    }
}

// File 2: Customer.cs (your hand-written)
partial class Customer {
    partial void OnNameChanged() {
        Console.WriteLine("Name was changed");
    }
}
```

If no implementation is provided, the compiler **removes the call entirely**. Used heavily by source generators — generated code can call hooks that users opt into.

### Modern partial methods (C# 9+)

Partial methods can also be **public** with an explicit `partial` modifier that requires implementation:

```csharp
public partial class Service {
    public partial Task<User?> GetUser(int id);
}

public partial class Service {
    public partial async Task<User?> GetUser(int id) {
        return await _db.Users.FindAsync(id);
    }
}
```

Used by source generators like `[GeneratedRegex]`, `[LoggerMessage]`, ASP.NET Minimal APIs `[GeneratedRoute]`, JSON source generators, etc.

### Partial properties, constructors, events

C# 13 added partial properties; C# 14 added partial constructors and events. Same idea: declaration in one file, implementation in another. Used heavily by source generators that emit getters/setters or constructor logic.

---

## `extern` methods

For platform invocation (P/Invoke) or other external implementations:

```csharp
[DllImport("kernel32.dll")]
public static extern bool Beep(uint freq, uint duration);

// Or C# 11+ source-generated form:
[LibraryImport("kernel32.dll")]
public static partial bool Beep(uint freq, uint duration);
```

[Chapter 14 §01](../14-InteropAOT/01-PInvoke.md) covers P/Invoke.

---

## `async` modifier

Tells the compiler to rewrite the method into a state machine that can `await` other tasks:

```csharp
public async Task<string> FetchAsync(string url) {
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
}
```

Return types: `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `void` (event handlers only), or a custom task-like type.

[Chapter 08 §02](../08-Concurrency/02-AsyncAwaitFundamentals.md) covers async deeply.

---

## Extension methods

Static methods that look like instance methods on another type:

```csharp
public static class StringExt {
    public static bool IsValidEmail(this string s) =>
        s.Contains('@') && s.Contains('.');
}

"x@y.com".IsValidEmail();   // syntactic sugar for StringExt.IsValidEmail("x@y.com")
```

Three rules:
1. Inside a `static` class.
2. The first parameter has `this` before it.
3. The class is `using`d (or globally imported) where the call appears.

Extension methods are how LINQ works (`Where`, `Select`, etc., are extension methods on `IEnumerable<T>`).

C# 14 introduces **extension members** — properties, statics, operators — see [Chapter 11 §07](../11-ModernFeatures/07-CSharp14.md).

---

## Overload resolution recap

When multiple methods match a call site:
1. Filter to **applicable** overloads (parameters compatible).
2. Pick the **best match** — most specific argument types, fewest implicit conversions.
3. If tied, ambiguous → compile error.

Subtleties:
- `params` overloads are considered last.
- Generic methods can win if the constraint is specific enough.
- Extension methods only kick in after instance methods on the type.

When overloads get confusing, give the methods different names instead. Compilers can't read your mind.

---

## Internals — how virtual dispatch works (the vtable)

### Non-virtual method

In IL, a non-virtual call is `call`:

```il
call instance void Demo::DoSomething()
```

The JIT resolves the address directly. No runtime lookup.

### Virtual method

A virtual call is `callvirt`:

```il
callvirt instance string Animal::Speak()
```

`callvirt`:
1. Loads the method table pointer from the object's header (offset 8 on 64-bit).
2. Indexes into the **vtable** — an array of method pointers, one per virtual slot.
3. Calls the resolved address.

Each class's vtable holds entries for all its virtual methods, inherited or its own. Subclasses override entries by replacing the pointer.

For:
```csharp
class A { public virtual void M() { } public virtual void N() { } }
class B : A { public override void M() { } }
```

```
A's vtable:                B's vtable:
[0] -> A::M                 [0] -> B::M     (overridden)
[1] -> A::N                 [1] -> A::N     (inherited)
```

When you call `((A)b).M()`, the JIT emits a `callvirt` that uses B's vtable (because `b` is actually a B) and dispatches to `B::M`.

### `callvirt` on non-virtual methods

Confusingly, the C# compiler emits `callvirt` even for non-virtual instance methods. Why? Because `callvirt` also implicitly checks for null — `call` doesn't. The JIT optimizes the dispatch to a direct call when it can, but the null check is preserved.

### Devirtualization

The JIT can sometimes prove a virtual call's target at compile time and replace `callvirt` with a direct `call`:
- The receiver is a `sealed` type or has a `sealed` override.
- The receiver is a value type (structs are sealed implicitly).
- Profile-guided optimization (PGO) sees one type dominate in practice.

This is why `sealed` matters for performance — it gives the JIT proof that no further override exists.

### Abstract methods

Abstract methods occupy a vtable slot but the slot contains a "throw if reached" stub. Subclasses must override and fill it in. Cannot be instantiated without overriding, enforced by the compiler.

### Static method internals

Static methods have no `this`, no vtable involvement. They emit a plain `call` (or `calli` for delegates). The JIT is free to inline them aggressively.

### `partial` method internals

If a partial method has no implementation, the compiler **removes calls to it entirely** — including evaluating the arguments. This is the magic source generators rely on: a generated class can call partial hooks unconditionally, and users pay nothing for hooks they don't implement.

---

## Method dispatch decision tree

```
Method call
│
├── Is it static? ──► direct call, no this
│
├── Is it on a sealed type or value type? ──► direct call (devirtualized)
│
├── Is it on a generic constrained to a value type? ──► direct call
│
├── Is it virtual / abstract? ──► callvirt via vtable
│
└── Is it non-virtual on a class? ──► callvirt (for null check) but JIT may inline
```

---

## Common bugs

- **Forgetting `override`** — adding a method with the same name as a base virtual method without `override` gives a warning (CS0114) and the new method **hides** the base. Use `override` or `new` explicitly.
- **Calling virtual methods in constructors** — derived override runs before derived ctor body. Don't do it.
- **Hiding `Equals` / `GetHashCode`** without `override` — wrong, you almost certainly meant to override.
- **Using `new` to "fix" a base class** — code smell. The hidden method only runs through a derived-type reference; consumers via the base see the original. Find a different solution.
- **Sealing too eagerly in a library** — your consumers can't extend. Default to sealed for application code; design carefully for libraries.

---

## Performance summary

| Call shape | Cost |
|---|---|
| Static method | One direct call. JIT inlines small ones. |
| Non-virtual instance method | One direct call + null check (may be elided). JIT inlines. |
| Virtual method on sealed receiver | Devirtualized → direct call. |
| Virtual method on non-sealed | One vtable lookup + indirect call. |
| Generic method on value type | Specialized per T, direct calls, often inlined. |
| Generic method on reference type | Single shared body, virtual-call-shaped. |

For hot code, marking classes `sealed` and avoiding interface dispatch when possible helps the JIT optimize. For everyday code, none of this matters.

→ Next: [06-Inheritance.md](06-Inheritance.md)
