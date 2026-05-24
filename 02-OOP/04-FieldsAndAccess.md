# Fields and Access Modifiers

## What it is

A **field** is a variable declared at the class level. It stores per-instance (or, with `static`, per-type) state.

An **access modifier** controls who can see and use a member: another class, another assembly, only subclasses, etc.

```csharp
public class Account {
    private decimal _balance;          // accessible only within Account
    public string Owner { get; init; } // accessible anywhere
}
```

---

## Why it matters

- **Encapsulation**: fields are how state lives. Access modifiers are how you protect that state from being modified inappropriately.
- **Versioning**: changes to a `private` field are invisible to consumers. Changes to a `public` API are breaking.

A general rule: **default to most restrictive access**. Open it up only when needed.

---

## Declaring a field

```csharp
public class Demo {
    private int _retries;            // initialized to 0
    private int _retries2 = 3;       // initializer
    public readonly string Name;     // assigned in ctor, read-only thereafter
    private const double Pi = 3.14;  // compile-time constant
    private static int _counter;     // per-type, not per-instance
}
```

### `readonly`

Assignable only in the constructor or at declaration:

```csharp
public class Person {
    public readonly string Name;
    public Person(string name) { Name = name; }   // ✓
    public void Rename(string n) { Name = n; }    // ❌ — readonly outside ctor
}
```

Pairs naturally with init-only properties and immutability.

### `const`

Compile-time constant, baked into the assembly:

```csharp
public const int MaxRetries = 3;
```

Restrictions:
- Must be a literal-representable type: primitive, `string`, `enum`, or `null`.
- Implicitly `static` — you don't write `static const`.
- Inlined at every call site — changing a public `const` requires recompiling consumers.

For non-compile-time constants, use `static readonly`:

```csharp
public static readonly DateTime AppStartedAt = DateTime.UtcNow;
public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
```

`static readonly` evaluates once at type initialization. Consumers see the current value at runtime — no inlining.

### `static`

Belongs to the type, not instances:

```csharp
public class Counter {
    private static int _total;
    public static int Total => _total;

    public Counter() { _total++; }
}

var a = new Counter();
var b = new Counter();
Console.WriteLine(Counter.Total);   // 2
```

There's exactly one `_total` shared by all `Counter` instances.

---

## Access modifiers — the full list

| Modifier | Visible from |
|---|---|
| `public` | Anywhere (any assembly) |
| `private` | Only inside the declaring type |
| `protected` | Declaring type + derived classes (any assembly) |
| `internal` | Anywhere in the same assembly |
| `protected internal` | Same assembly **OR** any derived class anywhere |
| `private protected` | Same assembly **AND** derived class |
| `file` | Only within the same source file (C# 11+) |

Default (no modifier):
- Class members: `private`.
- Top-level types: `internal`.

```csharp
class TopLevel { }              // internal
internal class Same { }         // explicit, same effect
public class Visible { }        // exposed to consumers

public class Demo {
    int x;                       // private (default)
    private int y;               // private (explicit)
    public int Z;                // public
}
```

### `protected` vs `internal` — the most useful distinction

```csharp
// ProjectA/Library.cs
public class Base {
    protected int Inherited;     // visible to subclasses anywhere
    internal int SameAssembly;   // visible only inside ProjectA
}

// ProjectA/Derived.cs
public class Derived : Base {
    void M() {
        Inherited = 5;    // ✓
        SameAssembly = 5; // ✓ — same assembly
    }
}

// ProjectB/OtherDerived.cs (references ProjectA)
public class Other : Base {
    void M() {
        Inherited = 5;    // ✓ — protected, derived class
        SameAssembly = 5; // ❌ — internal, different assembly
    }
}
```

### `protected internal` — OR

Either condition is enough:

```csharp
public class Base {
    protected internal int X;
}

// Same assembly, not derived → ✓
// Different assembly, derived  → ✓
// Different assembly, not derived → ❌
```

### `private protected` — AND

Both conditions required:

```csharp
public class Base {
    private protected int X;
}

// Same assembly, derived → ✓
// Same assembly, not derived → ❌
// Different assembly, anything → ❌
```

Useful for framework-internal extension points.

### `file` (C# 11+)

A class (or other type) can be `file`-scoped — visible only inside the same source file:

```csharp
// File: Helpers.cs
file class Utilities { ... }   // not visible from any other file
public class Service {
    // can use Utilities
}
```

Primary use case: **source generators** can emit `file`-scoped helper types without polluting the namespace.

You can also apply `file` to delegates, enums, records, interfaces.

---

## Field initializers

```csharp
public class Settings {
    public int Retries = 3;
    public List<string> Log = new();
    public string Source = Environment.MachineName;
}
```

Initializers run **before** the constructor body. Useful when the value is the same in all constructors. For values that depend on constructor parameters, initialize in the body.

For `static` fields, initializers run at type initialization time (the static constructor, implicit or explicit).

---

## When to expose state — fields vs properties

**Never** make a field `public`:

```csharp
// 🚨 anti-pattern
public class Person {
    public string Name;
}
```

Why not:
- No validation possible.
- Can't make virtual / overridable.
- Data binding (WPF, Blazor, MAUI) only works with properties.
- Future change to a property is a **binary breaking change** for callers.

Always use a property:

```csharp
public class Person {
    public string Name { get; init; } = "";
}
```

The JIT inlines trivial property accessors, so there's no perf cost.

---

## The `readonly` keyword on instances vs types

Three places `readonly` can appear, with different meanings:

1. **On a field** — assignable only in declaration or constructor.
   ```csharp
   private readonly int _x;
   ```

2. **On a struct** — every field must be readonly; instance methods can't mutate fields. See [Chapter 03 §02](../03-TypeSystem/02-Structs.md).
   ```csharp
   public readonly struct Point { public int X { get; } }
   ```

3. **On a struct instance method** — promises the method doesn't mutate `this`.
   ```csharp
   public struct Counter {
       public int Value;
       public readonly int Get() => Value;   // can read but not write
   }
   ```

---

## Internals — how fields live in memory

For an instance class:
```csharp
public class Foo {
    public int A;
    public byte B;
    public int C;
}
```

On a 64-bit runtime, layout is (typically):

```
offset 0  : sync block (8 bytes)
offset 8  : method table pointer (8 bytes)
offset 16 : A   (4 bytes)
offset 20 : B   (1 byte) + 3 bytes padding for alignment
offset 24 : C   (4 bytes) + 4 bytes padding (total size aligned to 8)
```

Size: 32 bytes. The runtime is free to reorder fields for better packing (modern .NET often does), unless you specify `[StructLayout(LayoutKind.Sequential)]` or `[FieldOffset]` (for interop).

For statics, the runtime allocates a separate per-type storage area (one per loaded type, lifetime = AppDomain). Static fields are addressed by offset into this area.

### IL for field access

```csharp
public class Demo {
    public int X;
    public void Set() { X = 5; }
}
```

```il
.method public hidebysig instance void Set() cil managed {
  ldarg.0          // 'this'
  ldc.i4.5         // push 5
  stfld int32 Demo::X
  ret
}
```

`stfld` (store field) takes `this` and the value, writes the value to the field. Reading is `ldfld`. For static fields, `ldsfld` / `stsfld`.

### How `const` is different from `static readonly`

```csharp
public class Config {
    public const int Version = 1;
    public static readonly int Build = 100;
}
```

In IL:
- `Version` becomes a `literal` field in metadata. Its value is inlined into every consumer's IL.
- `Build` becomes a regular `static` field. Consumers access it via `ldsfld`.

This is why changing a `public const` requires consumers to recompile — they have the old value embedded in their IL. Changing `static readonly` lets new consumers immediately pick up the new value.

### Memory savings — pack tightly

If you have many instances of a small class, field layout can matter:

```csharp
public class Bad {
    public byte A;
    public long B;
    public byte C;
}
// Layout (after padding): A(1) + 7pad + B(8) + C(1) + 7pad = 24 bytes for fields
// + 16 bytes object header = 40 bytes
```

```csharp
public class Good {
    public byte A;
    public byte C;
    public long B;
}
// Layout: A(1) + C(1) + 6pad + B(8) = 16 bytes for fields
// + 16 bytes header = 32 bytes
```

Reorder smaller fields next to each other. The CLR's auto-layout often does this for you, but for `StructLayout.Sequential` types (common in interop) field order matters.

---

## Internal access — friend assemblies

By default, `internal` means "this assembly only." To extend access to specific other assemblies (e.g., a test project), use `InternalsVisibleTo`:

```csharp
// AssemblyInfo.cs (or csproj)
[assembly: InternalsVisibleTo("MyApp.Tests")]
```

Now `MyApp.Tests` can access `internal` members of `MyApp` as if they were `public`. Used universally for unit testing — your tests don't need to bend over backwards to test internal logic.

In csproj:
```xml
<ItemGroup>
  <InternalsVisibleTo Include="MyApp.Tests" />
</ItemGroup>
```

---

## Common patterns

### The "encapsulation pair"

```csharp
private readonly List<Item> _items = new();
public IReadOnlyList<Item> Items => _items;
```

Mutable storage internally, read-only view externally. Callers can iterate but not Add/Remove.

### Public read, private write

```csharp
public int Count { get; private set; }
```

Or with init-only setters:

```csharp
public int Count { get; init; }
```

### Constants of complex types

```csharp
public static readonly Color Red = new(255, 0, 0);
public static readonly Color Green = new(0, 255, 0);
public static readonly Color Blue = new(0, 0, 255);
```

Used when you can't use `const` (custom type) but want named values.

### Defensive readonly copy

```csharp
private readonly int[] _values;
public IReadOnlyList<int> Values => _values;

public Demo(IEnumerable<int> values) {
    _values = values.ToArray();   // copy — caller can't mutate later
}
```

---

## Common bugs

- **Public mutable field** — anyone can write anything. Use a property with controlled access.
- **`readonly` field exposed as `public`** — readonly only prevents reassignment of the reference, not mutation of the object's interior. `public readonly List<int> Items;` still lets callers `.Add(1)`.
- **`const` value that "should be configurable later"** — changing it requires rebuilding everything that referenced it. Default to `static readonly`.
- **Static field as a cache without thread safety** — multi-threaded reads are usually fine, but writes need `lock` / `Interlocked` / `ConcurrentDictionary`.
- **Field initializer order surprises** — initializers run top-to-bottom **of the class**, not by call order. Don't rely on initializer A reading from initializer B.

---

## Performance

- Field access is the fastest data operation on the CLR.
- Auto-property access is **the same speed** in optimized code (JIT inlines the trivial accessor).
- Static field access is similar to instance — both are direct memory reads.
- Volatile reads (`volatile` keyword or `Volatile.Read`) prevent reordering but are slightly slower on weak-memory CPUs (ARM). Use only when needed.

---

## When to use which

| Need | Use |
|---|---|
| Private mutable storage | `private` field |
| Public read-only computed value | `public T Prop => ...` |
| Public read-write | `public T Prop { get; set; }` |
| Public set-once-only | `public T Prop { get; init; }` |
| Compile-time constant | `public const T NAME = ...;` (primitive/string/enum) |
| Runtime constant | `public static readonly T NAME = ...;` |
| Shared per-type counter / config | `private static T _name;` |
| Subclass-only extension point | `protected T` |
| Test access to internals | `[assembly: InternalsVisibleTo("...")]` |

→ Next: [05-MethodsAdvanced.md](05-MethodsAdvanced.md)
