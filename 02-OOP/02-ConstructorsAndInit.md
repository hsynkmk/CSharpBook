# Constructors and Initialization

## What it is

A **constructor** is a special method that runs when a new instance is created. It initializes the object's state. Constructors don't declare a return type — the compiler knows.

```csharp
public class Person {
    public string Name { get; }
    public int Age { get; }

    public Person(string name, int age) {
        Name = name;
        Age = age;
    }
}

var alice = new Person("Alice", 30);
```

---

## Why it exists

Without constructors, you'd have to remember to call an `Initialize()` method after `new`, and forgetting it would leave the object in an invalid state. Constructors **guarantee** initialization happens — you can't have a `Person` without supplying name and age.

---

## Multiple constructors

A class can have many constructors with different signatures (overloads):

```csharp
public class Person {
    public string Name { get; }
    public int Age { get; }

    public Person(string name) {
        Name = name;
        Age = 0;
    }

    public Person(string name, int age) {
        Name = name;
        Age = age;
    }
}

var anon = new Person("Anon");
var alice = new Person("Alice", 30);
```

---

## Constructor chaining — `this`

To avoid duplicating logic, one constructor can delegate to another via `this(...)`:

```csharp
public Person(string name)              : this(name, 0) { }
public Person(string name, int age) {
    Name = name;
    Age = age;
}
```

The `this(name, 0)` clause runs before the body, calling the second constructor with default age = 0. Now there's one place that actually assigns fields.

---

## Calling the base constructor — `base`

When inheriting from another class, you forward initialization with `base(...)`:

```csharp
public class Animal {
    public string Name { get; }
    public Animal(string name) { Name = name; }
}

public class Dog : Animal {
    public string Breed { get; }
    public Dog(string name, string breed) : base(name) {
        Breed = breed;
    }
}
```

`base(name)` runs the `Animal` constructor before the `Dog` constructor body.

If you don't write `base(...)`, the compiler inserts `base()` (the parameterless one). Error if there isn't one.

---

## Default constructor

If you write **no** constructor at all, the compiler generates a parameterless one for you:

```csharp
public class Empty { }
var e = new Empty();   // calls compiler-generated public Empty() { }
```

The instant you write any constructor, the default disappears:

```csharp
public class Demo {
    public Demo(int x) { }
}

var d = new Demo();    // ❌ compile error — no parameterless ctor
var d = new Demo(5);   // ✓
```

If you want both, write both:

```csharp
public Demo() { }
public Demo(int x) { /* ... */ }
```

---

## Object initializers

A succinct way to set public properties (or fields) right after `new`:

```csharp
public class Point {
    public int X { get; set; }
    public int Y { get; set; }
}

var p = new Point { X = 5, Y = 10 };
```

Equivalent to:
```csharp
var p = new Point();
p.X = 5;
p.Y = 10;
```

Object initializers run **after** the constructor. Useful when you want some fields set in the ctor and others optionally afterward.

You can combine with constructor args:

```csharp
var p = new Person("Alice") { Age = 30 };
```

### Collection initializers

For types that have an `Add(...)` method and implement `IEnumerable`:

```csharp
var nums = new List<int> { 1, 2, 3, 4 };

var dict = new Dictionary<string, int> {
    { "one", 1 },
    { "two", 2 },
    ["three"] = 3      // indexer syntax
};
```

Modern: **collection expressions** (C# 12+):

```csharp
List<int> nums = [1, 2, 3, 4];
int[] arr = [10, 20, 30];
ReadOnlySpan<int> span = [1, 2, 3];
```

See [Chapter 10 §08](../10-AdvancedLanguage/08-CollectionExpressions.md).

---

## Target-typed `new` (C# 9+)

Skip the type on the right of `=` when context makes it clear:

```csharp
Person p = new("Alice", 30);
List<int> nums = new() { 1, 2, 3 };
Dictionary<string, int> d = new();
Person[] team = { new("Alice", 30), new("Bob", 25) };
```

Useful for trimming repetition. Not useful when readability suffers — `var x = new();` makes no sense (and doesn't compile).

---

## `init` accessors (C# 9+)

A property's setter can be marked `init` so it's settable **only during construction or object initialization**:

```csharp
public class Person {
    public string Name { get; init; } = "";
    public int Age { get; init; }
}

var alice = new Person { Name = "Alice", Age = 30 };
alice.Age = 31;   // ❌ compile error — init-only after construction
```

`init` lets external callers set the value via an object initializer (no need for a constructor parameter for every property), then locks it down. The result is **shallow immutability** without ceremony.

When to use:
- DTOs, configuration objects, command/query inputs.
- Anywhere "set once on construction, immutable thereafter."

If you want callers to be **required** to set it, mark with `required` (C# 11+):

```csharp
public class Person {
    public required string Name { get; init; }
    public int Age { get; init; }
}

var p = new Person { Name = "Alice" };   // OK
var p = new Person { };                  // ❌ — Name is required
```

[Chapter 10 §04](../10-AdvancedLanguage/04-RequiredMembers.md) has more.

---

## Field initializers

Fields can be initialized at their declaration:

```csharp
public class Settings {
    public int Retries = 3;
    public TimeSpan Timeout = TimeSpan.FromSeconds(30);
    private readonly List<string> _log = new();
}
```

Field initializers run **before** the constructor body. Useful when the value is the same for all constructors.

For `static` fields, initializers run when the type is first used (more on this in [§05](05-MethodsAdvanced.md)).

---

## Constructor execution order

For `new DerivedClass(...)`:

1. **Memory allocated** for the new object on the heap.
2. **All fields zeroed** (default values).
3. **Derived field initializers** run (`private int _x = 5;` in `DerivedClass`).
4. **Base constructor** runs.
   - That recursively does the same: base's field initializers, then base's base constructor, etc., all the way up to `Object`.
5. **Derived constructor body** runs.
6. **Object initializer** (if present at `new`) runs.

This order has consequences — see "calling virtual methods in constructors" below.

---

## Static constructors

A type can have **one** static constructor that runs once before any instance is created or any static member is accessed:

```csharp
public class Config {
    public static readonly string ConfigPath;

    static Config() {
        ConfigPath = Environment.GetEnvironmentVariable("CONFIG_PATH")
                     ?? "/etc/myapp/config.toml";
    }
}
```

Rules:
- No access modifier (always private to the runtime).
- No parameters.
- Runs at most once per AppDomain, lazily, in a thread-safe way (the CLR handles synchronization).
- Useful for one-time setup: complex `static readonly` initialization, validating config, etc.

The CLR guarantees only that the static constructor runs **before the first observable use** of the type. The exact timing depends on the `BeforeFieldInit` flag in metadata (an optimization — see "Internals" below).

---

## Primary constructors (C# 12+)

A new compact syntax where constructor parameters live for the entire type body:

```csharp
public class Person(string name, int age) {
    public string Greet() => $"Hello, I'm {name}, {age} years old";
}

var p = new Person("Alice", 30);
Console.WriteLine(p.Greet());
```

Behind the scenes the compiler generates fields and a constructor; the parameters are visible to every member.

Detailed coverage in [§12 PrimaryConstructors](12-PrimaryConstructors.md).

---

## Internals — what `new T(...)` actually does

In IL, `new Person("Alice", 30)` becomes:

```il
ldstr      "Alice"
ldc.i4.s   30
newobj     instance void Person::.ctor(string, int32)
```

`newobj`:
1. Allocates memory (via the GC).
2. Zeros all fields.
3. Calls the constructor.
4. Leaves the new reference on the IL stack.

Constructors are emitted as ordinary methods named `.ctor` (instance) or `.cctor` (static):

```il
.method public hidebysig specialname rtspecialname instance void
        .ctor(string name, int32 age) cil managed
{
  .maxstack 8
  ldarg.0        // this
  call           instance void [System.Runtime]System.Object::.ctor()
  ldarg.0        // this
  ldarg.1        // name
  stfld          string Person::'<Name>k__BackingField'
  ldarg.0
  ldarg.2        // age
  stfld          int32 Person::'<Age>k__BackingField'
  ret
}
```

Notes:
- The constructor implicitly calls `base(...)` — visible in the `call instance void System.Object::.ctor()` line.
- Properties with `{ get; init; }` get compiler-generated backing fields named like `<Name>k__BackingField` (the `<>` brackets are illegal in C# source — that's intentional, to avoid name collision).

### Static constructor and `BeforeFieldInit`

A type has the `beforefieldinit` flag by default. With it set, the CLR is **free** to delay running the static constructor until any static field is *actually accessed*. This allows aggressive optimizations (the JIT can hoist code that doesn't touch static fields without triggering `.cctor`).

If you write an explicit `static MyClass() { ... }`, the compiler does NOT emit `beforefieldinit`. The static ctor runs **eagerly** — before *any* member of the type is touched, even instance methods. This is the "predictable but slower" mode.

Performance tip: avoid writing static constructors when possible. Use field initializers (or `Lazy<T>`) instead — they let `beforefieldinit` stay, which the JIT loves.

```csharp
// beforefieldinit kept → JIT-friendly
public static readonly int X = ComputeX();

// beforefieldinit lost → eagerly initialized
static MyClass() { X = ComputeX(); }
```

### `Object..ctor`

Every constructor ultimately calls `System.Object`'s constructor. It does almost nothing (a single `ret`); the work was done by `newobj` allocating and zeroing. The chain through bases is mostly a way for derived classes to pass arguments to their base.

### Field-init order

Field initializers are reordered by the compiler so they appear **before** the body of the constructor in IL — but **after** the implicit (or explicit) `base(...)` / `this(...)` call. This means:

```csharp
class Base {
    public Base() { Init(); }       // virtual call from constructor — danger!
    protected virtual void Init() { }
}

class Derived : Base {
    public string Data = "ready";   // field init
    protected override void Init() {
        Console.WriteLine(Data);    // prints null! Field init hasn't run yet
    }
}

new Derived();
```

Order of operations for `new Derived()`:
1. Derived field init queued.
2. `base()` runs (which calls virtual `Init`).
3. **`Derived.Init` runs** because the runtime type is `Derived`.
4. **But `Data` is still null** — field initializers haven't run yet.
5. Derived field init runs (`Data = "ready"`).
6. Derived constructor body runs (empty here).

**Lesson**: never call virtual methods from constructors. They run with partially-constructed `this`.

---

## Common patterns

### Validation in constructor

```csharp
public BankAccount(string owner, decimal initial) {
    ArgumentException.ThrowIfNullOrWhiteSpace(owner);
    ArgumentOutOfRangeException.ThrowIfNegative(initial);
    _owner = owner;
    _balance = initial;
}
```

Throw early if invariants are violated. The object never exists in a bad state.

### Defensive copying

For collection parameters, copy to prevent later mutation:

```csharp
public class Team {
    private readonly List<Player> _players;
    public IReadOnlyList<Player> Players => _players;

    public Team(IEnumerable<Player> players) {
        _players = new List<Player>(players);   // copy
    }
}
```

Without the copy, the caller could later modify the list and observe the change in the object's state.

### Cloning via constructor

```csharp
public Person(Person other) {
    Name = other.Name;
    Age = other.Age;
}

var clone = new Person(original);
```

Useful for explicit shallow copy. Records' `with` expression replaces this for most cases.

### Constructor as factory

When construction requires async work or is complex, hide constructor and expose a factory:

```csharp
public class Connection {
    private Connection(string conn) { ... }

    public static async Task<Connection> ConnectAsync(string url) {
        var conn = new Connection(url);
        await conn.HandshakeAsync();
        return conn;
    }
}

var c = await Connection.ConnectAsync("...");
```

Async constructors don't exist in C# — factories are the standard workaround.

---

## Common bugs

- **Calling virtual methods from a constructor** — derived override runs with partially-initialized `this`.
- **Throwing from a constructor without cleanup** — if you've already opened a file before throwing, the object is gone but the file handle leaks. Wrap in try/catch or use `using`/SafeHandle.
- **Forgetting `this(...)`** — duplicated field assignment across multiple constructors. Chain them.
- **Long constructors** — five+ parameters is a smell. Use a builder, options object, or `required` properties.
- **Static constructor swallowing exceptions** — if `.cctor` throws, it's wrapped in `TypeInitializationException` and the type is permanently unusable in this app domain. Don't put risky work in static constructors.

→ Next: [03-Properties.md](03-Properties.md)
