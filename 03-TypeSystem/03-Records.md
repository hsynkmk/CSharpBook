# Records

## What it is

A **record** is a reference type (or value type if `record struct`) with synthesized equality, `ToString`, deconstruction, and "non-destructive mutation" via `with` expressions. The compiler does the boring boilerplate; you write the data.

```csharp
public record Person(string Name, int Age);

var alice = new Person("Alice", 30);
var older = alice with { Age = 31 };
Console.WriteLine(alice == new Person("Alice", 30));  // true — value equality
Console.WriteLine(alice);                              // Person { Name = Alice, Age = 30 }
```

Records arrived in C# 9 (2020) and have become the default choice for immutable data containers.

---

## Why it exists

Before records, defining a small immutable data class meant a wall of boilerplate:

```csharp
public class Person {
    public Person(string name, int age) { Name = name; Age = age; }
    public string Name { get; }
    public int Age { get; }
    public override bool Equals(object? obj) =>
        obj is Person p && Name == p.Name && Age == p.Age;
    public override int GetHashCode() => HashCode.Combine(Name, Age);
    public override string ToString() => $"Person {{ Name = {Name}, Age = {Age} }}";
    public void Deconstruct(out string name, out int age) { name = Name; age = Age; }
    public Person With(string? name = null, int? age = null) => new(name ?? Name, age ?? Age);
}
```

Every project had dozens of these. Now:

```csharp
public record Person(string Name, int Age);
```

Identical behavior. The motivation is straightforward — immutable data containers are a common need, and the language should help.

---

## Two flavors

```csharp
public record       Person(string Name, int Age);   // reference type
public record class Person(string Name, int Age);   // explicit reference (same thing)
public record struct Point(int X, int Y);            // value type (C# 10+)
public readonly record struct Point(int X, int Y);   // immutable value type — preferred for small value types
```

`record` alone is shorthand for `record class`. Use `record struct` for small value-like things you'd otherwise write as structs.

---

## Positional vs nominal records

### Positional (the common form)

```csharp
public record Person(string Name, int Age);
```

This declares:
- An auto-property `string Name { get; init; }`.
- An auto-property `int Age { get; init; }`.
- A constructor `Person(string Name, int Age)`.
- A `Deconstruct(out string Name, out int Age)` method.
- All the equality / `ToString` / `with` synthesized members.

Properties are `init`-only by default — settable at construction, immutable thereafter.

### Nominal (longer form)

```csharp
public record Person {
    public string Name { get; init; } = "";
    public int Age { get; init; }
}
```

No constructor parameter list, no positional constructor. You instantiate via object initializer:

```csharp
var p = new Person { Name = "Alice", Age = 30 };
```

Useful when:
- You have many properties (positional gets unwieldy at 5+).
- You want `required` modifiers.
- You don't need a `Deconstruct`.

You can also mix — define a positional record and add extra properties in the body:

```csharp
public record Person(string Name, int Age) {
    public string DisplayName => $"{Name} ({Age})";
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}
```

---

## What the compiler generates

For `public record Person(string Name, int Age);` you get roughly:

```csharp
public class Person : IEquatable<Person> {
    public string Name { get; init; }
    public int Age { get; init; }

    public Person(string name, int age) { Name = name; Age = age; }

    public void Deconstruct(out string name, out int age) {
        name = Name; age = Age;
    }

    protected Person(Person original) {        // copy constructor (for `with`)
        Name = original.Name;
        Age  = original.Age;
    }
    public virtual Person <Clone>$() => new Person(this);   // method backing `with`

    protected virtual Type EqualityContract => typeof(Person);

    public override bool Equals(object? obj) => Equals(obj as Person);
    public virtual bool Equals(Person? other) =>
        !(other is null) &&
        EqualityContract == other.EqualityContract &&
        EqualityComparer<string>.Default.Equals(Name, other.Name) &&
        EqualityComparer<int>.Default.Equals(Age, other.Age);

    public override int GetHashCode() {
        var h = new HashCode();
        h.Add(EqualityContract);
        h.Add(Name);
        h.Add(Age);
        return h.ToHashCode();
    }

    public override string ToString() {
        var sb = new StringBuilder();
        sb.Append("Person { ");
        if (PrintMembers(sb)) sb.Append(' ');
        sb.Append("}");
        return sb.ToString();
    }
    protected virtual bool PrintMembers(StringBuilder sb) {
        sb.Append("Name = ").Append(Name).Append(", Age = ").Append(Age);
        return true;
    }

    public static bool operator ==(Person? a, Person? b) => Equals(a, b);
    public static bool operator !=(Person? a, Person? b) => !Equals(a, b);
}
```

Phew. That's why we don't write it by hand anymore.

---

## `with` expressions

The signature feature of records:

```csharp
var alice = new Person("Alice", 30);
var older = alice with { Age = 31 };
```

`with` returns a **new** record, copying everything from the original and replacing the listed properties. It does NOT mutate. The original is untouched.

How it works:
1. Compiler calls the synthesized **copy constructor** (`protected Person(Person original)`).
2. Then applies the property assignments in the `{ ... }` block.

This means `with` only works on **init-able properties**. Read-only properties without `init` setters can't be replaced.

You can `with`-mutate as many properties as you want:

```csharp
var changed = alice with { Name = "Alice Smith", Age = 31 };
```

Or none:

```csharp
var cloned = alice with { };   // equivalent to a clone
```

---

## Equality

Records use **value equality** out of the box. Two records are equal if:
1. They have the same runtime type (the `EqualityContract`).
2. All instance properties (and fields) compare equal via `EqualityComparer<T>.Default`.

```csharp
new Person("Alice", 30) == new Person("Alice", 30);   // true
```

Reference equality is still available via `ReferenceEquals`:

```csharp
ReferenceEquals(new Person("Alice", 30), new Person("Alice", 30));   // false
```

### The reference-property trap

```csharp
public record Cart(int UserId, List<string> Items);

var a = new Cart(1, new() { "milk", "eggs" });
var b = new Cart(1, new() { "milk", "eggs" });
Console.WriteLine(a == b);   // false! List<T> equality is reference
```

`EqualityComparer<List<string>>.Default` compares **references** because `List<T>` doesn't override `Equals`. The records aren't equal even though they "look the same."

Fixes:
- Use `ImmutableArray<string>` or `ImmutableList<string>` — they have value equality.
- Override `Equals` manually for the record.
- Avoid mutable reference properties on records.

```csharp
public record Cart(int UserId, ImmutableArray<string> Items);

var a = new Cart(1, ImmutableArray.Create("milk", "eggs"));
var b = new Cart(1, ImmutableArray.Create("milk", "eggs"));
Console.WriteLine(a == b);   // true
```

---

## Inheritance among records

Records can inherit from other records (NOT from non-record classes, except `object`):

```csharp
public record Person(string Name);
public record Employee(string Name, decimal Salary) : Person(Name);
```

The synthesized equality respects the **runtime type** (`EqualityContract`), so:

```csharp
Person p = new Person("Alice");
Person e = new Employee("Alice", 50000);
Console.WriteLine(p == e);   // false — different EqualityContract
```

This handles a classic equality footgun across inheritance hierarchies.

`record struct` does **not** support inheritance (they're sealed structurally, like all structs).

---

## Records with bodies (additional members)

```csharp
public record Order(int Id, decimal Subtotal) {
    public decimal Tax { get; init; }
    public decimal Total => Subtotal + Tax;

    public Order WithDiscount(decimal pct) =>
        this with { Tax = Tax * (1 - pct) };

    public void Validate() {
        if (Subtotal < 0) throw new InvalidOperationException();
    }
}
```

You can add properties (with or without `init`), methods, indexers, static members. The synthesized `Equals` and `GetHashCode` only consider properties **declared in the positional parameter list** plus **explicitly declared properties** in the body.

To control which members participate in equality, see [Chapter 10 §03 — RecordsDeep](../10-AdvancedLanguage/03-RecordsDeep.md).

---

## Records and validation

Positional records run their primary constructor's argument assignments without a body. To validate, use the new "primary ctor body":

```csharp
public record Person(string Name, int Age) {
    public Person {                            // validator
        ArgumentException.ThrowIfNullOrWhiteSpace(Name);
        ArgumentOutOfRangeException.ThrowIfNegative(Age);
    }
}
```

That `public Person { ... }` syntax is **only valid in records**. It runs after the positional assignments.

For nominal records, use a regular constructor:

```csharp
public record Person {
    public string Name { get; init; } = "";
    public int Age { get; init; }
    public Person(string name, int age) {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentOutOfRangeException.ThrowIfNegative(age);
        Name = name; Age = age;
    }
}
```

---

## Deconstruction

Positional records auto-generate `Deconstruct`:

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30);
var (name, age) = p;   // calls p.Deconstruct(out var name, out var age)
```

Useful in switch expressions:

```csharp
string Category(Person p) => p switch {
    (_, < 18) => "minor",
    (_, < 65) => "adult",
    _         => "senior"
};
```

For nominal records, write your own `Deconstruct` method if you want this.

---

## Sealed and override considerations

Records' synthesized `Equals` and `ToString` are `virtual`. Subclassed records override them. If you want to lock things down, mark the record `sealed`:

```csharp
public sealed record Person(string Name, int Age);
```

For records that won't be inherited, `sealed` lets the JIT devirtualize equality checks. Worth doing for hot-path records.

---

## Records vs classes — decision matrix

| | Use a `record` | Use a `class` |
|---|---|---|
| Equality semantics | Value | Reference |
| Mostly immutable | ✓ | Maybe |
| Has identity beyond data | ✗ | ✓ |
| Inheritance hierarchy with shared behavior | Possible but unusual | ✓ |
| Mutable state and operations | ✗ | ✓ |
| DTO, command, query, event | ✓ | ✗ |
| Domain entity | ✗ | ✓ |
| Owns resources (file, socket) | ✗ | ✓ |
| `with` non-destructive mutation | ✓ | ✗ (manual) |

Default: use a `record` for **data containers**, a `class` for **stateful objects with identity**.

---

## Internals — performance and IL

### How `with` works

```csharp
var p2 = p with { Age = 31 };
```

IL:
```
ldarg.0 (p)
callvirt Person::<Clone>$()    // creates a copy via the copy ctor
dup
ldc.i4.s 31
callvirt Person::set_Age(int32)
stloc.0
```

The clone, then set the changed property, then assign. So `with` is O(N) in number of properties (the copy ctor copies each), plus one set per `with` clause.

### Equality

The synthesized `Equals` does field-by-field comparison via `EqualityComparer<T>.Default`. For simple property types like primitives, that's just a value compare — fast. For reference type properties, it depends on whether they override `Equals`.

### `ToString`

The synthesized `ToString` builds a string each call — there's no caching. For a hot logging path, consider overriding `ToString` to use a `StringBuilder` directly, or precompute and cache.

### Heap layout

A record class is a regular reference type — heap allocation, MT pointer, fields packed normally. The synthesized members (`<Clone>$`, copy ctor, `EqualityContract` property, `PrintMembers`) add a few KB of generated code per record type, but nothing per instance.

A `record struct` is laid out exactly like a regular struct.

---

## Common patterns

### Domain events

```csharp
public abstract record DomainEvent(DateTime OccurredAt);
public sealed record OrderPlaced(int OrderId, DateTime OccurredAt) : DomainEvent(OccurredAt);
public sealed record OrderShipped(int OrderId, string TrackingNumber, DateTime OccurredAt) : DomainEvent(OccurredAt);
```

### Commands / queries (CQRS)

```csharp
public record CreateUserCommand(string Email, string Password);
public record GetUserByIdQuery(int Id);
```

### Result types

```csharp
public abstract record Result<T>;
public sealed record Ok<T>(T Value) : Result<T>;
public sealed record Err<T>(string Reason) : Result<T>;
```

Pair with pattern matching:

```csharp
string Display<T>(Result<T> r) => r switch {
    Ok<T>(var v)  => $"Got {v}",
    Err<T>(var e) => $"Error: {e}",
    _ => throw new()
};
```

### Configuration objects

```csharp
public record Config {
    public required string ConnectionString { get; init; }
    public int Timeout { get; init; } = 30;
    public bool UseSsl { get; init; } = true;
}

services.AddSingleton(new Config {
    ConnectionString = "...",
    Timeout = 60,
});
```

---

## Common bugs

- **Reference-property equality** — `List<T>` and other reference types compare by reference. Use immutables or override.
- **Mutable reference properties** — `with` doesn't deep-copy. If you do `r with { Items = r.Items }` and then mutate `Items`, both records see the change.
- **Inheritance breaking equality** — comparing `Person` to `Employee` via `Person` reference is unequal (different `EqualityContract`) — but the same code without records might accidentally compare equal.
- **Records that should be classes** — a "user with a database identity" usually has reference identity, not value identity. Don't make it a record.
- **Records with hidden state** — adding mutable fields breaks the immutability promise without breaking compilation.

---

## Performance notes

- Synthesized equality is competitive with hand-written. Sometimes faster (HashCode helper is optimized).
- `with` allocates a new object — be mindful in hot loops.
- `record struct` avoids heap allocation. Always prefer `readonly record struct` for small value types.
- Sealing records lets the JIT devirtualize the equality contract check.

→ Next: [04-Enums.md](04-Enums.md)
