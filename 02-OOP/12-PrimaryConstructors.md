# Primary Constructors (C# 12+)

A primary constructor moves a constructor's parameters into the type header, where they stay in scope for the entire type body. The compiler generates the actual constructor for you, and any parameter you actually use becomes a private capture field — no manual plumbing.

```csharp
public class Person(string name, int age) {
    public string Greet() => $"Hi, I'm {name}, {age} years old";
}

var p = new Person("Alice", 30);
Console.WriteLine(p.Greet());     // Hi, I'm Alice, 30 years old
```

`name` and `age` are not properties, not auto-fields you can see by name — they're constructor parameters that the body happens to be able to read.

---

## Where you can use it

Primary constructors work on:

- `class` (C# 12)
- `struct` and `record struct` (C# 12)
- `record` and `record class` (C# 9 — records had primary constructors first; C# 12 extended the feature to plain classes and structs)
- `interface`: **no** — interfaces don't have constructors.

You can have **one** primary constructor per type. Other constructors are allowed, but they must chain into the primary one (see below).

---

## What the compiler generates

Two things, on demand:

1. **A constructor** with the declared parameters. If the type would otherwise be parameterless, the implicit default constructor disappears — `new Person()` no longer compiles.
2. **A hidden capture field** for each parameter you reference from an instance member outside the constructor body.

```csharp
public class Counter(int start) {
    public int Read() => start;          // captures `start`
    public void Inc()  => start++;       // captures `start` (and mutates it!)
}
```

Roughly equivalent hand-written code:

```csharp
public class Counter {
    private int start;                   // capture field (compiler-named)
    public Counter(int start) { this.start = start; }
    public int Read() => start;
    public void Inc()  => start++;
}
```

If a parameter is only used **inside** field initializers or the constructor's own setup — never from any other instance member — **no capture field is emitted**. The compiler treats it like a normal constructor argument:

```csharp
public class Logger(string path) {
    private readonly StreamWriter _writer = new(path);   // `path` used only here
    // No capture field for `path`; it's gone after the ctor runs.
}
```

This is the headline efficiency story: pay for capture only when you use it.

---

## Parameters are mutable

Primary-constructor parameters behave like local variables/fields, not like `readonly`. You can reassign them and the change is observed by every later read:

```csharp
public class Cache(int capacity) {
    public void Resize(int n) => capacity = n;     // mutates the capture field
    public int Capacity => capacity;
}
```

Some teams find this surprising — it looks like a parameter but acts like a field. Two ways to defend against accidental mutation:

```csharp
public class Cache(int capacity) {
    private readonly int _capacity = capacity;     // freeze on construction
    public int Capacity => _capacity;
}
```

or, with `field` (C# 14):

```csharp
public class Cache(int capacity) {
    public int Capacity { get; } = capacity;       // immutable from outside
}
```

The compiler emits a warning (CS9124) when you capture a parameter that's also assigned to a field with the same name — see "Double storage" below.

---

## Exposing parameters as properties

Primary-constructor parameters are **not** properties. Outside code can't write `person.name`. To expose them, declare a property and initialize it from the parameter:

```csharp
public class Person(string name, int age) {
    public string Name { get; } = name;
    public int Age    { get; init; } = age;
}
```

Compare records (C# 9), which auto-generate public `init`-only properties for every primary-constructor parameter:

```csharp
public record PersonRecord(string Name, int Age);
// implicit: public string Name { get; init; }
//          public int    Age  { get; init; }
//          + value equality, ToString, Deconstruct, with-expressions
```

This is the key difference: a record's primary constructor is a shortcut for a **whole bag of synthesized members**. A class's primary constructor is just a shortcut for the **constructor itself**.

---

## Multiple constructors — chaining

If you want secondary constructors, they must delegate to the primary one with `: this(...)`. There's no way around it.

```csharp
public class Connection(string host, int port) {
    // Convenience overload
    public Connection(string host) : this(host, 5432) { }

    // ❌ Compile error — must chain to primary
    // public Connection() { }
}
```

The reasoning: the primary constructor owns initialization of the capture fields. Any other entry point has to go through it, or the capture state is undefined.

This also means **field initializers always run on entry to the primary constructor**, not on entry to any secondary one. The order is:

1. Caller invokes a secondary constructor.
2. Secondary chains to the primary via `: this(...)`.
3. Primary constructor body begins: base constructor, then field initializers (which can use the parameters), then any explicit body code (rare — primary constructors are usually bodyless).
4. Control returns to the secondary's body (almost always empty).

---

## Calling base constructors

A primary constructor can pass arguments to its base:

```csharp
public class Animal(string name) {
    public string Name => name;
}

public class Dog(string name, string breed) : Animal(name) {
    public string Breed => breed;
}
```

The `Animal(name)` part is the base-constructor call — it runs before the derived primary's capture fields are assigned. This is the cleanest replacement for the old `: base(name)` pattern.

You cannot mix syntaxes: if the derived class has a primary constructor, the base call goes in the header, not on each individual constructor.

---

## Structs

Primary constructors work for `struct` and `record struct`. The same capture-only-if-used rule applies:

```csharp
public struct Point(double x, double y) {
    public double DistanceFromOrigin => Math.Sqrt(x * x + y * y);
}
```

One subtlety: every struct still has an **implicit parameterless constructor** that zero-initializes all fields. That implicit constructor does **not** run your primary constructor — so `default(Point)` gives `x = 0, y = 0`, never the values from your primary parameters:

```csharp
Point p1 = new Point(3, 4);   // primary ctor runs → x=3, y=4
Point p2 = default;           // primary ctor skipped → x=0, y=0
Point[] arr = new Point[10];  // every element is default — primary not run
```

This is the same gotcha that has always applied to structs: their parameterless construction skips your initialization code. Primary constructors don't fix it.

---

## When primary constructors are a good fit

✓ Small immutable-ish classes where every method just reads the constructor parameters (typical DI service: `class OrderService(IDbContext db, ILogger log)`).
✓ Replacing a class whose constructor's only job is to assign parameters to private fields.
✓ Cases where you'd otherwise reach for a record but you don't want value equality or `with`-expressions.

```csharp
// Idiomatic DI service in C# 12+
public class OrderService(IDbContext db, ILogger<OrderService> log) {
    public Order Place(Cart c) {
        log.LogInformation("placing {Count} items", c.Items.Count);
        var o = new Order(c);
        db.Orders.Add(o);
        db.SaveChanges();
        return o;
    }
}
```

Less ceremony, same behavior. The captured `db` and `log` are kept in private fields.

---

## When to avoid them

✗ The class has many constructors with significantly different responsibilities — chaining everything through one primary is awkward.
✗ You need a parameter to be `readonly` *and* visible everywhere — declare an explicit field, since primary parameters are mutable.
✗ You'd want most parameters exposed as properties — use a record instead.
✗ The captured value needs validation. Primary constructors don't have a natural place for `ArgumentNullException.ThrowIfNull(...)`. You can hoist it into a field initializer (`private readonly IDbContext _db = db ?? throw new ArgumentNullException(nameof(db));`), but at that point a regular constructor is clearer.

---

## Common pitfalls

### Double storage

```csharp
public class Box(int size) {
    private int size = size;   // ⚠ CS9124 — capture conflict
}
```

The compiler warns: it now has both a *capture field* for the parameter (because `Box` references `size` from somewhere) **and** the field you declared. Two slots for the same value. Fix by renaming or by using the parameter directly:

```csharp
public class Box(int size) {
    private readonly int _size = size;   // ✓ different name
}
```

### Mutability surprises

```csharp
public class Window(int width) {
    public void Resize(int w) => width = w;     // mutates capture
    public int Width => width;
}
```

Looks like a parameter; behaves like a field. If you want to forbid mutation, mirror the value into a `readonly` field as shown above.

### Equality

Primary constructors on classes (not records) give you **nothing** for equality. Two `Person("Alice", 30)` instances are not equal — `Equals` falls through to reference equality. If you want value equality, use a `record` or override `Equals`/`GetHashCode` manually.

### Capturing inside lambdas

```csharp
public class Service(IDbContext db) {
    public Func<int, Task<Order?>> Find => id => db.Orders.FindAsync(id).AsTask();
}
```

The lambda captures `db` through the same hidden field. Lifetime is fine — the closure holds `this` indirectly, which keeps the capture field alive. No special pitfalls here, but worth knowing the chain.

### Reassigning during construction

```csharp
public class Demo(int x) {
    private readonly int _square = x * x;
    public void Cheat() => x = 0;       // legal; `_square` is unaffected
}
```

`x` is mutable; `_square` was computed once during construction. After `Cheat()`, `_square` still holds the original square. This is by design — field initializers run once — but it can be surprising if you expected `_square` to track `x`.

---

## Primary constructors vs records — quick comparison

| Aspect | `class C(int x)` | `record R(int X)` |
|---|---|---|
| Generates a constructor | yes | yes |
| Generates properties | no | yes — public `init` |
| Value equality | no | yes |
| `ToString()` shows members | no | yes |
| Deconstruction | no | yes |
| `with` expression support | no | yes |
| Parameters mutable in body | yes (capture field) | no (properties are `init`-only) |
| Use for | DI services, small wrappers | data carriers, DTOs |

Rule of thumb: if the type's job is to *carry data*, use a record. If it's to *do behavior* and just happens to need a few injected dependencies, use a primary-constructor class.

---

## Internals — what the IL looks like

Source:

```csharp
public class Greeter(string name) {
    public string Hello() => $"Hello, {name}";
}
```

The compiler emits roughly:

```il
.class public auto ansi beforefieldinit Greeter extends [System.Runtime]System.Object
{
  .field private initonly string '<name>P'           // capture field, mangled name

  .method public hidebysig specialname rtspecialname
          instance void .ctor(string name) cil managed
  {
    ldarg.0
    ldarg.1
    stfld    string Greeter::'<name>P'              // assign capture
    ldarg.0
    call     instance void [System.Runtime]System.Object::.ctor()
    ret
  }

  .method public hidebysig instance string Hello() cil managed
  {
    ldstr     "Hello, "
    ldarg.0
    ldfld     string Greeter::'<name>P'             // read capture
    call      string [System.Runtime]System.String::Concat(string, string)
    ret
  }
}
```

A few things to notice:

- The capture field is named with angle brackets (`<name>P`) — illegal in C# source, intentional, prevents name collisions with user code.
- The capture is `initonly` (i.e., `readonly` at the IL level) **only when the parameter is never assigned** anywhere in the type body. Mutate the parameter once and the field loses `initonly`.
- The base `System.Object::.ctor()` call happens **after** the capture assignment. This differs from a hand-written constructor where you'd usually see the base call first. The C# language spec orders capture-field assignment ahead of the base call so the base constructor can — in theory — see the captured values via virtual dispatch. In practice few people rely on it.
- If no member references `name`, the field is omitted entirely and the constructor just calls the base.

### Capture decisions are made per parameter, per type

```csharp
public class WithBoth(int used, int unused) {
    public int Read() => used;
}
```

`used` becomes a capture field; `unused` does not. The constructor signature still takes both — callers must pass both — but the type pays storage cost only for what it actually keeps.

### Records add more synthesis

For `record R(int X)`, the compiler additionally generates:

- `public int X { get; init; } = X;` (auto-property initialized from the parameter)
- `public override bool Equals(object?)`, structural
- `public bool Equals(R?)`, structural
- `public override int GetHashCode()`
- `public override string ToString()`
- `protected virtual bool PrintMembers(StringBuilder)`
- `public void Deconstruct(out int X)`
- `public static bool operator ==(R?, R?)` / `!=`
- A protected copy constructor used by `with`-expressions

Hence the dramatic difference in compiled size between a class and a record with the same primary-constructor declaration. The primary constructor *syntax* is the same; the *synthesis* attached to it is not.

---

## Summary

- Primary constructors put parameters in the type header, in scope for the whole body.
- The compiler generates the actual constructor, plus a private capture field for each parameter you actually use — zero overhead for unused parameters.
- Parameters are **mutable** capture fields, not auto-properties; they're not exposed to outside code unless you wire a property.
- A type with a primary constructor can have secondary constructors, but every one of them must chain via `: this(...)`.
- Primary constructors propagate to the base via the header: `class Dog(string name) : Animal(name)`.
- For data-carrier types you almost always want a record instead — records build a long list of synthesized members on top of the same syntax.
- Watch out for double-storage (CS9124), mutation-by-accident, and struct `default` skipping the primary constructor.

→ Back to: [README.md](README.md)
