# Classes and Objects

## What it is

A **class** is a blueprint for creating objects. An **object** (or instance) is a runtime value following that blueprint. Classes bundle **state** (fields) with **behavior** (methods), giving you reusable, composable units that model the problem domain.

```csharp
public class Person {
    public string Name;
    public int Age;

    public string Greet() => $"Hello, I'm {Name}";
}

var alice = new Person { Name = "Alice", Age = 30 };
Console.WriteLine(alice.Greet());   // "Hello, I'm Alice"
```

`Person` is the class. `alice` is one object — an *instance* of `Person`.

---

## Why it exists

OOP gives us:
- **Encapsulation** — bundle data with operations that use it; hide internals.
- **Abstraction** — name things; expose behavior, not representation.
- **Inheritance and polymorphism** — share and specialize behavior across related types.

Even in a multi-paradigm language like C#, classes remain the default building block. They map naturally to most real-world domain concepts.

---

## Declaring a class

The minimal syntax:

```csharp
public class Empty { }
```

A more realistic class:

```csharp
public class BankAccount {
    // Fields
    private decimal _balance;
    private readonly string _owner;

    // Constructor
    public BankAccount(string owner, decimal initial) {
        _owner = owner;
        _balance = initial;
    }

    // Properties
    public string Owner => _owner;
    public decimal Balance => _balance;

    // Methods
    public void Deposit(decimal amount) {
        ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);
        _balance += amount;
    }

    public bool TryWithdraw(decimal amount) {
        if (amount <= 0 || amount > _balance) return false;
        _balance -= amount;
        return true;
    }
}
```

Members in roughly conventional order: fields → constructors → properties → methods. Many teams have an `.editorconfig` that enforces order.

---

## Creating instances — `new`

```csharp
var account = new BankAccount("Alice", 1000m);
```

Three steps happen at runtime:
1. **Allocate** memory on the heap for the object.
2. **Initialize** all fields to their default values (zeros / null), then run field initializers, then run the constructor.
3. **Return** a reference to the new object.

Target-typed `new` (C# 9+) lets you skip the type if it's obvious from context:

```csharp
BankAccount account = new("Alice", 1000m);
Dictionary<string, int> counts = new();
List<int> nums = new() { 1, 2, 3 };
```

---

## Reference semantics

Classes are **reference types**. Variables hold a pointer to the heap object, not the object itself:

```csharp
var a = new BankAccount("Alice", 1000m);
var b = a;                  // b refers to the same object
b.Deposit(500);
Console.WriteLine(a.Balance);   // 1500 — same instance
Console.WriteLine(ReferenceEquals(a, b));  // true
```

Compare with structs (value types) where assignment copies — see [Chapter 03 §02](../03-TypeSystem/02-Structs.md).

### `null`

A class-typed variable can hold `null` — "no object":

```csharp
BankAccount? maybe = null;
maybe.Deposit(100);   // NullReferenceException
```

With Nullable Reference Types enabled, the compiler tracks nullable references via `?` and warns you about unchecked dereferences. See [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md).

---

## `this`

Inside an instance method, `this` refers to the current object:

```csharp
public class Counter {
    private int _n;
    public void Increment() { this._n++; }   // explicit
    public void Reset()     { _n = 0; }       // implicit
    public Counter Self()   => this;          // return ourselves
}
```

You can write `this._n` or `_n` — both work. Convention is to write `_n` (rely on the field prefix) and use `this` only when needed for disambiguation:

```csharp
public class Person {
    private readonly string name;
    public Person(string name) {
        this.name = name;        // `this.name` (field) vs `name` (parameter)
    }
}
```

Or, more commonly, name fields differently:
```csharp
public Person(string name) { _name = name; }
```

`this` is also a value — you can pass it as an argument or return it for fluent chaining:

```csharp
public Builder WithName(string n) { _name = n; return this; }
```

---

## Members — five kinds

A class can contain:
1. **Fields** — data storage.
2. **Constants** — `const` values.
3. **Properties** — `get`/`set` wrappers around storage or computation.
4. **Methods** — behavior.
5. **Events** — multicast delegate notifications.
6. **Indexers** — `obj[key]` syntax sugar.
7. **Operators** — `==`, `+`, etc. for custom types.
8. **Constructors / finalizer** — special methods for init/cleanup.
9. **Nested types** — classes/structs/enums declared inside.

We cover each in later files of this chapter and elsewhere. This file focuses on the basic shape.

---

## Access modifiers (preview)

```csharp
public class Outer {       // visible to other assemblies
    public int X;          // visible everywhere
    private int Y;         // only inside this class
    protected int Z;       // this class + subclasses
    internal int W;        // anywhere in this assembly
    protected internal int V;  // (protected) OR (internal)
    private protected int U;   // (protected) AND (internal)
    file int F;            // only inside this file (C# 11+)
}
```

Detailed in [§04 Fields and Access](04-FieldsAndAccess.md).

---

## A complete example

```csharp
public class Rectangle {
    private double _width;
    private double _height;

    public Rectangle(double width, double height) {
        ArgumentOutOfRangeException.ThrowIfNegative(width);
        ArgumentOutOfRangeException.ThrowIfNegative(height);
        _width = width;
        _height = height;
    }

    public double Width => _width;
    public double Height => _height;
    public double Area => _width * _height;
    public double Perimeter => 2 * (_width + _height);

    public Rectangle Scaled(double factor) =>
        new Rectangle(_width * factor, _height * factor);

    public override string ToString() => $"Rect({_width} × {_height})";
}

var r = new Rectangle(3, 4);
Console.WriteLine(r);              // Rect(3 × 4)
Console.WriteLine(r.Area);         // 12
Console.WriteLine(r.Scaled(2));    // Rect(6 × 8)
```

That's a complete, idiomatic small class: encapsulated state, validated construction, immutable from outside, expression-bodied computed properties, override of `ToString`.

---

## Internals — how a class becomes runtime memory

When you write `new Rectangle(3, 4)`, the runtime:

1. **Computes object size** — header + field slots, rounded up for alignment.
   - **64-bit runtime:** the header is **16 bytes** (8-byte object header / sync block + 8-byte method table pointer).
   - **32-bit runtime:** 8 bytes (4 + 4).
   - Field layout is determined by the JIT, usually packed in declaration order but reordered for alignment.

2. **Calls the GC allocator** — bumps a pointer in the current Gen0 segment (almost always — this is why allocations are typically free in throughput, expensive in collection).

3. **Initializes the method table pointer** — every object carries one. This pointer is how the runtime knows what type the object is and dispatches virtual methods.

4. **Zeros all fields** — value-type fields get their default; reference-type fields get `null`.

5. **Runs field initializers** — e.g., `private int _retries = 3;` sets `_retries = 3`.

6. **Runs the constructor body** — your `public Rectangle(...)`.

7. **Returns the reference** — basically a pointer to the start of the field area (the header is "behind" the reference).

For a `Rectangle(3, 4)` instance on 64-bit .NET 10:
```
offset 0  : sync block / object header (8 bytes)
offset 8  : method table pointer       (8 bytes)
offset 16 : _width                      (8 bytes, double)
offset 24 : _height                     (8 bytes, double)
```
Total: 32 bytes per Rectangle.

### How method calls work

Calling `r.Area` (a property — also a method internally):

```
1. Load 'this' (the reference to r) into a register
2. Dereference the method table pointer (offset 8)
3. Look up the method slot for 'get_Area' in the type's vtable
4. Call the method's native code
```

For non-virtual methods, the JIT often resolves the address at JIT time and emits a direct call — skipping the vtable lookup. Virtual methods go through the vtable.

### `new` is just shorthand

In IL, `new Rectangle(3, 4)` becomes:
```il
ldc.r8 3.0           // push 3.0
ldc.r8 4.0           // push 4.0
newobj instance void Rectangle::.ctor(float64, float64)
```

`newobj` is the IL opcode that does the allocate-and-construct.

### A peek at the memory using SOS

In `dotnet-dump`:
```
> dumpobj 0x7fff12345678
Name:        Rectangle
MethodTable: 0x7fff00ABCD00
EEClass:     0x7fff00ABCE00
Size:        32(0x20) bytes
Fields:
              MT  Field   Offset           Type      Attr    Value Name
00007fff...   4001 0x10  System.Double   instance     3.0   _width
00007fff...   4002 0x18  System.Double   instance     4.0   _height
```

Most C# developers never need this. But knowing it exists demystifies the GC, performance debugging, and discussions about object size.

---

## Common patterns

### Static factory methods

When the constructor is awkward or `new` doesn't communicate intent:

```csharp
public class Color {
    private Color(byte r, byte g, byte b) { ... }
    public static Color FromRgb(byte r, byte g, byte b) => new(r, g, b);
    public static Color FromHex(string hex) { /* parse */ ... }
    public static Color Red => new(255, 0, 0);
}

Color.FromHex("#FF0000")
Color.Red
```

The private constructor forces callers through the factory. Naming the entry point clarifies intent.

### Fluent / builder style

```csharp
var req = new HttpRequestBuilder()
    .WithMethod(HttpMethod.Post)
    .WithUrl("https://api.example.com/items")
    .WithHeader("X-Auth", "token")
    .Build();
```

Each method returns `this` (or a derived builder) so calls chain. Useful when constructing has many optional steps.

### Immutable classes

```csharp
public sealed class Point {
    public Point(double x, double y) { X = x; Y = y; }
    public double X { get; }
    public double Y { get; }
    public Point Translate(double dx, double dy) => new(X + dx, Y + dy);
}
```

State is set in the constructor and never changes. Methods that "modify" return new instances. Records (Chapter 03 §03) make this even shorter, but raw classes still work fine for immutability.

---

## Common bugs

- **Forgetting to call `new`**: `Rectangle r;` then `r.Area` is a compile error (definite assignment); but a field of a reference type defaults to `null` and dereferencing it throws.
- **Two variables, same object**: assignment doesn't copy classes. Mutations via `a` show up via `b`.
- **Using a class as a "namespace"** (all-static methods): in modern C#, prefer a `static class` to signal intent.
- **Public mutable fields**: makes encapsulation impossible. Use properties.
- **Huge classes (god classes)**: SRP violation. Refactor.

---

## When to use a class

| Use a **class** when... | Use a **struct** when... | Use a **record** when... |
|---|---|---|
| Identity matters (two instances differ even with same data) | The type is a small value-like thing | You want value-based equality |
| Inheritance is needed | No inheritance | Want `with` expressions + structural eq |
| State is shared / mutable | Immutable + small (<16 bytes) | Mostly-immutable data containers |
| You need finalizers / Dispose | Few or no resources | DTOs, command/query objects |

Structs in [Chapter 03 §02](../03-TypeSystem/02-Structs.md). Records in [Chapter 03 §03](../03-TypeSystem/03-Records.md).

→ Next: [02-ConstructorsAndInit.md](02-ConstructorsAndInit.md)
