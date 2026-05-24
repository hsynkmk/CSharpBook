# Properties

## What it is

A **property** looks like a field from outside but is actually a pair of methods (`get_X` and `set_X`) the compiler generates. Properties let you expose data with controlled access, validation, computation, and the freedom to evolve the implementation without breaking callers.

```csharp
public class Person {
    public string Name { get; set; }      // auto-property (compiler-generated backing field)
    public int Age { get; set; }
}

var p = new Person();
p.Name = "Alice";          // looks like field assignment...
Console.WriteLine(p.Name); // ...but goes through set_Name / get_Name under the hood
```

---

## Why it exists

C# could have followed Java's `getName()` / `setName(...)` convention. Instead, properties give you the **best of both**:
- **Source-level convenience** — `p.Name = "x"`, no parens, looks like a field.
- **Implementation flexibility** — the body can be anything.
- **Versioning safety** — start as auto-property, evolve to validated property, no caller changes.

You'll write properties more than any other member.

---

## Anatomy

```csharp
public class Demo {
    private int _count;

    public int Count {
        get { return _count; }
        set {
            if (value < 0) throw new ArgumentOutOfRangeException();
            _count = value;
        }
    }
}
```

Pieces:
- `Count` — the property name (PascalCase).
- `get` accessor — runs when someone reads the property. Returns the value.
- `set` accessor — runs when someone writes. The implicit parameter is **`value`**.

You can have just `get`, just `set`, or both. Just `set` is rare (most people prefer a method).

---

## Auto-properties

When you don't need a custom body, the compiler generates the backing field:

```csharp
public string Name { get; set; }
```

Equivalent to:
```csharp
private string <Name>k__BackingField;
public string Name {
    get { return <Name>k__BackingField; }
    set { <Name>k__BackingField = value; }
}
```

The backing field name has illegal-in-C# characters so it never collides with your code.

### Default value for auto-properties

```csharp
public string Name { get; set; } = "(unknown)";
public List<int> Items { get; set; } = new();
```

Equivalent to a field initializer on the synthesized backing field.

---

## Expression-bodied properties

For one-expression getters:

```csharp
public int Square => _value * _value;
public string Greeting => $"Hello, {Name}";
```

`=>` here is the property's get. No set. Equivalent to:

```csharp
public int Square { get { return _value * _value; } }
```

You can also have expression-bodied getters/setters separately:

```csharp
public string Name {
    get => _name;
    set => _name = value ?? throw new ArgumentNullException(nameof(value));
}
```

---

## Read-only properties

Several flavors, depending on what you want:

```csharp
// 1. Auto-property with only get — must be set in constructor or field init
public string Id { get; }

// 2. Computed — no backing store
public int FullName => $"{First} {Last}";

// 3. Explicit backing field, read-only outside
private int _count;
public int Count => _count;
```

For (1), you can set via `=` initializer or in the constructor; nothing else can change it.

---

## `init` accessors

A modern alternative to read-only: settable **only during construction or object initializer**, immutable thereafter.

```csharp
public string Name { get; init; } = "";

var p = new Person { Name = "Alice" };   // OK
p.Name = "Bob";                          // ❌ compile error
```

Replaces the "ctor parameter for every property" pattern:
```csharp
// Pre-init-only
public Person(string name) { Name = name; }
public string Name { get; }

// With init
public string Name { get; init; }
// new Person { Name = "Alice" } — done
```

[Chapter 03 §03 Records](../03-TypeSystem/03-Records.md) uses `init` accessors heavily.

---

## `required` (C# 11+)

Marks a member that **must** be set when constructing:

```csharp
public class Order {
    public required int OrderId { get; init; }
    public required string Customer { get; init; }
    public string? Notes { get; init; }
}

var o = new Order { OrderId = 1, Customer = "Alice" };   // OK
var o = new Order { OrderId = 1 };                       // ❌ Customer is required
```

Caller is forced to supply via object initializer (or via a constructor decorated with `[SetsRequiredMembers]`).

[Chapter 10 §04](../10-AdvancedLanguage/04-RequiredMembers.md) covers `required` deeply.

---

## Validation in setters

```csharp
private int _age;
public int Age {
    get => _age;
    set {
        if (value < 0 || value > 150)
            throw new ArgumentOutOfRangeException(nameof(value));
        _age = value;
    }
}
```

This is the classic motivation for non-trivial properties — you can guarantee invariants.

---

## Computed (read-only) properties

When the value is derived from other state:

```csharp
public class Rectangle {
    public double Width { get; init; }
    public double Height { get; init; }
    public double Area => Width * Height;          // recomputed each access
    public double Perimeter => 2 * (Width + Height);
}
```

If the computation is expensive, you have three options:
1. **Cache lazily** — backing field, compute on first read.
2. **Recompute every call** — simple, but pay each time.
3. **Convert to a method** — signals "this isn't free" to callers.

A reasonable rule: if it's O(1) and cheap, property; if it's O(n) or has side effects, method.

---

## The `field` keyword (C# 14)

The biggest C# 14 property feature. Inside a property accessor, `field` is a contextual keyword referring to the compiler-generated backing field — letting you mix custom logic with auto-property convenience:

```csharp
// Pre-C# 14 — must declare backing field explicitly
private int _retries;
public int Retries {
    get => _retries;
    set {
        ArgumentOutOfRangeException.ThrowIfNegative(value);
        _retries = value;
    }
}

// C# 14 — use `field` instead
public int Retries {
    get;
    set {
        ArgumentOutOfRangeException.ThrowIfNegative(value);
        field = value;
    }
}
```

The compiler still creates the backing field; you just access it via `field`.

Other uses:

```csharp
// Lazy initialization
public string Cached => field ??= ExpensiveCompute();

// Default value combined with validation
public int Threshold {
    get => field;
    set => field = value > 0 ? value : throw new ArgumentException();
} = 100;
```

`field` becomes a reserved word only inside accessor bodies. Elsewhere it's an ordinary identifier. (You can still call a variable `field` outside.)

**Limitation**: `field` is only available inside the accessor bodies. You can't refer to it from another method or property.

---

## Static properties

A property on the type, not on instances:

```csharp
public class Config {
    public static string Version { get; } = "1.0";
    public static int Count { get; set; }
}

Console.WriteLine(Config.Version);
Config.Count++;
```

Common for singletons, configuration, derived constants.

---

## Indexers — `this[...]`

A property-like construct accessed via indexer syntax:

```csharp
public class Bag {
    private readonly Dictionary<string, int> _items = new();
    public int this[string key] {
        get => _items.GetValueOrDefault(key, 0);
        set => _items[key] = value;
    }
}

var bag = new Bag();
bag["apple"] = 3;
Console.WriteLine(bag["apple"]);   // 3
```

The "name" is `this` followed by a parameter list. You can have multiple indexers with different parameter types.

---

## Explicit interface property implementation

When implementing an interface property, you can do so **explicitly** so it's only callable through the interface:

```csharp
public interface IReadable { string Content { get; } }

public class File : IReadable {
    public string Path { get; init; } = "";
    string IReadable.Content {                 // explicit
        get => System.IO.File.ReadAllText(Path);
    }
}

var f = new File { Path = "x.txt" };
// f.Content;                       // ❌ — explicit, not on File directly
IReadable r = f;
Console.WriteLine(r.Content);       // ✓
```

Used to disambiguate two interfaces with the same property name, or to keep the public surface small.

---

## Internals — what a property really is

A property is **syntactic sugar for two methods** plus optional metadata.

For:
```csharp
public int Count { get; set; }
```

The compiler emits (roughly):

```il
.field private int32 '<Count>k__BackingField'

.method public hidebysig specialname instance int32 get_Count() cil managed {
  ldarg.0
  ldfld int32 Demo::'<Count>k__BackingField'
  ret
}

.method public hidebysig specialname instance void set_Count(int32 'value') cil managed {
  ldarg.0
  ldarg.1
  stfld int32 Demo::'<Count>k__BackingField'
  ret
}

.property instance int32 Count() {
  .get instance int32 Demo::get_Count()
  .set instance void Demo::set_Count(int32)
}
```

The `.property` metadata is a hint for tools (IDE, reflection, `[SerializableAttribute]`) — but the methods are what actually exist at runtime.

### Consequences

- **Property access compiles to a method call.** `p.Count = 5;` is `p.set_Count(5);` in IL.
- **The JIT will inline trivial accessors** (e.g., auto-property getters) — most calls become a single `ldfld` / `stfld` like a plain field.
- **You can be virtual.** `public virtual int Count { get; set; }` emits `virtual` getters/setters. Subclasses can `override`.
- **Reflection sees properties separately from methods.** `typeof(Demo).GetProperty("Count")` returns a `PropertyInfo`; `GetMethod("get_Count")` returns the underlying method.
- **`specialname` and `rtspecialname`** flags tell the runtime these aren't ordinary methods — IDEs and serializers handle them accordingly.

### `[CallerMemberName]` and properties

The cleanest INotifyPropertyChanged implementation uses property setters with `[CallerMemberName]`:

```csharp
public class Vm : INotifyPropertyChanged {
    public event PropertyChangedEventHandler? PropertyChanged;

    private int _count;
    public int Count {
        get => _count;
        set {
            if (_count != value) {
                _count = value;
                OnPropertyChanged();
            }
        }
    }

    private void OnPropertyChanged([CallerMemberName] string? name = null) =>
        PropertyChanged?.Invoke(this, new PropertyChangedEventArgs(name));
}
```

`[CallerMemberName]` injects `"Count"` automatically. See [Chapter 10 §12](../10-AdvancedLanguage/12-CallerInfoAttributes.md).

---

## Properties vs fields — when to use which

| Need | Use field | Use property |
|---|---|---|
| Just storage | ✓ private | ✓ public |
| Public state | (anti-pattern) | ✓ |
| Validation on set | ✗ | ✓ |
| Computed value | ✗ | ✓ |
| Inheritable / virtual | ✗ | ✓ |
| Used with data binding | ✗ | ✓ (must) |
| Reflection-friendly | (yes) | ✓ |
| Fastest possible access | ✓ (no method call) | ✓ if inlined by JIT |

Rule: **make public state a property**, never a field. The JIT inlines trivial accessors, so the cost is zero, and you preserve the option to evolve.

---

## Common patterns

### Backing-field pattern (pre-C# 14)
```csharp
private int _retries = 3;
public int Retries {
    get => _retries;
    set => _retries = value > 0 ? value : throw new ArgumentException();
}
```

### `field` keyword (C# 14)
```csharp
public int Retries {
    get;
    set => field = value > 0 ? value : throw new ArgumentException();
} = 3;
```

Same behavior, less boilerplate.

### Lazy property
```csharp
private List<string>? _items;
public List<string> Items => _items ??= LoadItems();
```

Or with `field`:
```csharp
public List<string> Items => field ??= LoadItems();
```

### Notify on change (MVVM)
```csharp
public int Count {
    get;
    set {
        if (field == value) return;
        field = value;
        OnPropertyChanged();
    }
}
```

### Property as a contract
```csharp
public interface IUser {
    string Name { get; }       // implementers must provide getter
    bool IsActive { get; set; } // get AND set
}
```

---

## Common bugs

- **Setter that doesn't validate** — defeats the purpose of using a property over a field.
- **`get` with side effects** — `Count` shouldn't increment something. Properties should be **idempotent reads**.
- **Allocating in a getter** that gets called per frame in UI — surprise GC pressure.
- **Auto-property exposing a mutable collection** — `public List<int> Items { get; } = new();` lets callers mutate the list. Use `IReadOnlyList<int>` for the public face.
- **Recursion via the `field` keyword (C# 14)** — `set => field = field + value;` is fine. `set => Property = Property + value;` recurses infinitely. Use `field` for direct access.

---

## Performance

- Properties compile to methods. The JIT inlines trivial ones — no overhead vs a field.
- Properties with validation, virtual dispatch, or non-trivial bodies have a small call cost (still negligible most of the time).
- Don't put expensive work in property getters; callers may invoke them many times in tight loops without realizing.

→ Next: [04-FieldsAndAccess.md](04-FieldsAndAccess.md)
