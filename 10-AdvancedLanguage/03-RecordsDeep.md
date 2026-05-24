# Records — Deep Dive

> Chapter 03 §03 introduced records. This file digs into the synthesized members, `EqualityContract`, inheritance, and customization.

---

## The full synthesized member set

For `public record Person(string Name, int Age);`, the compiler generates:

| Synthesized member | Purpose |
|---|---|
| `public string Name { get; init; }` | Property |
| `public int Age { get; init; }` | Property |
| `public Person(string Name, int Age)` | Primary constructor |
| `public void Deconstruct(out string Name, out int Age)` | Deconstruction |
| `protected virtual Type EqualityContract` | Runtime type marker |
| `public override bool Equals(object? obj)` | Equality |
| `public virtual bool Equals(Person? other)` | Strongly-typed equality |
| `public override int GetHashCode()` | Hash code |
| `public static bool operator ==(Person? a, Person? b)` | Equality operator |
| `public static bool operator !=(Person? a, Person? b)` | Inequality operator |
| `public override string ToString()` | Pretty printing |
| `protected virtual bool PrintMembers(StringBuilder builder)` | Helper for ToString |
| `protected Person(Person original)` | Copy constructor |
| `public virtual Person <Clone>$()` | Method backing `with` expressions |
| `public Type EqualityContract { get; }` (alias) | Public accessor |

That's a lot. With one line of source code.

---

## Equality details

### Two `Equals` methods

```csharp
public override bool Equals(object? obj) => Equals(obj as Person);

public virtual bool Equals(Person? other) =>
    !ReferenceEquals(null, other) &&
    EqualityContract == other.EqualityContract &&
    EqualityComparer<string>.Default.Equals(Name, other.Name) &&
    EqualityComparer<int>.Default.Equals(Age, other.Age);
```

Object overload: cast to Person, defer to typed Equals.

Typed Equals:
1. Reject null.
2. Check `EqualityContract` matches (handles inheritance, see below).
3. Compare each member via `EqualityComparer<T>.Default`.

### Why `EqualityContract`

```csharp
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);

Animal a = new Animal("Rex");
Animal b = new Dog("Rex", "Lab");

a.Equals(b);   // false — different EqualityContract
```

`EqualityContract` is `typeof(Animal)` for Animal, `typeof(Dog)` for Dog. Even though Rex's name matches, they're different types — Equals returns false.

This avoids the classic equality-across-inheritance trap (where `a.Equals(b)` is true but `b.Equals(a)` is false because of asymmetric subclass checks).

The check is:
- Both Animal: contracts match.
- One Animal, one Dog: contracts differ. Not equal.
- Both Dog: contracts match, then check Name + Breed.

### GetHashCode

```csharp
public override int GetHashCode() {
    return ((EqualityComparer<Type>.Default.GetHashCode(EqualityContract) * -1521134295)
        + EqualityComparer<string>.Default.GetHashCode(Name)) * -1521134295
        + EqualityComparer<int>.Default.GetHashCode(Age);
}
```

Includes the EqualityContract hash. Then combines each member's hash using a prime multiplier (essentially FNV or similar).

Result: same EqualityContract + same members → same hash. Consistent with Equals.

---

## `with` expression mechanics

```csharp
var p = new Person("Alice", 30);
var older = p with { Age = 31 };
```

Compiles to:

```csharp
var older = p.<Clone>$();
older.Age = 31;
```

`<Clone>$` calls the copy constructor:

```csharp
protected Person(Person original) {
    Name = original.Name;
    Age = original.Age;
}
```

Then the `Age = 31` assignment uses the init-only setter (allowed during object initializer context).

The result is a new instance with most fields copied + the specified ones replaced.

### Why a virtual `<Clone>$`?

Because records can inherit. A `Dog with { Breed = "Lab" }` calls Dog's `<Clone>$`, which calls Dog's copy constructor (which chains to Animal's copy constructor). The virtual dispatch ensures the right type is created.

```csharp
public record Animal(string Name) {
    public virtual Animal <Clone>$() => new Animal(this);
}

public record Dog(string Name, string Breed) : Animal(Name) {
    public override Dog <Clone>$() => new Dog(this);   // covariant return
}

Animal a = new Dog("Rex", "Lab");
var copy = a with { Name = "Buddy" };
// copy is actually a Dog, not just an Animal.
```

Covariant return on `<Clone>$` ensures `with` preserves the runtime type.

---

## `PrintMembers` and `ToString`

```csharp
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
```

`ToString` calls `PrintMembers` to dump each property.

For inheritance, derived `PrintMembers` overrides and appends its own properties:

```csharp
public record Dog : Animal {
    protected override bool PrintMembers(StringBuilder sb) {
        if (base.PrintMembers(sb)) sb.Append(", ");
        sb.Append("Breed = ").Append(Breed);
        return true;
    }
}

new Dog("Rex", "Lab").ToString();
// "Dog { Name = Rex, Breed = Lab }"
```

The `if (base.PrintMembers(sb))` chains through the hierarchy. Each level appends its members.

You can override `PrintMembers` manually for custom formatting (e.g., omitting password fields).

---

## Customizing record equality

Suppose you want a record to compare values approximately:

```csharp
public sealed record Coord(double X, double Y) {
    public virtual bool Equals(Coord? other) =>
        other is not null &&
        EqualityContract == other.EqualityContract &&
        Math.Abs(X - other.X) < 1e-9 &&
        Math.Abs(Y - other.Y) < 1e-9;

    public override int GetHashCode() => HashCode.Combine(
        EqualityContract,
        Math.Round(X, 9),
        Math.Round(Y, 9));
}
```

Override `Equals(Coord? other)` and `GetHashCode`. The compiler still synthesizes everything else (including the public `bool Equals(object?)`), and it calls your typed Equals.

**Keep `EqualityContract` check**: ensures inheritance-aware equality.

---

## Excluding members from equality

Records auto-include positional parameters in equality. To exclude a member, you have to take more control.

If the property is in the **body** (not positional), it's NOT included in synthesized equality:

```csharp
public record Person(string Name) {
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
    // CreatedAt is NOT part of the synthesized Equals/GetHashCode
}
```

For positional parameters, you can't easily exclude. Workaround: move to body:

```csharp
public record Person {
    public string Name { get; init; } = "";
    public int Age { get; init; }   // body — included in equality
    public int CacheHits { get; init; }   // body — also included by default
}
```

Hmm, body properties ARE included in synthesized equality (since C# 10's design). The exclusion rule is more subtle — see specs. For tight control, override Equals + GetHashCode manually.

---

## Records with reference-type members

```csharp
public record Cart(int UserId, List<string> Items);
```

`List<string>` doesn't override Equals → reference equality.

```csharp
var a = new Cart(1, new() { "x" });
var b = new Cart(1, new() { "x" });
a == b;   // false — different list instances
```

Synthesized equality uses `EqualityComparer<T>.Default`. For List<T>, that's reference equality.

Fixes:
- Use `ImmutableArray<string>` (has structural equality).
- Override Equals manually with `SequenceEqual`.
- Wrap in a custom value-equal type.

```csharp
public record Cart(int UserId, ImmutableArray<string> Items);   // structural equality
```

---

## Records with derived classes

```csharp
public record Animal(string Name);
public record Dog(string Name, string Breed) : Animal(Name);
```

The `: Animal(Name)` calls the base record's primary constructor.

Both `Animal` and `Dog` are valid records — each has its own synthesized members.

Equality across hierarchy is **type-aware**:
```csharp
Animal a = new Dog("Rex", "Lab");
Animal b = new Dog("Rex", "Lab");
a == b;     // true — same runtime type, same members

Animal a2 = new Animal("Rex");
Animal b2 = new Dog("Rex", "Lab");
a2 == b2;   // false — different runtime types
```

The `EqualityContract` check catches the type difference. No surprising "Animal equals Dog" bugs.

---

## Inheritance restrictions

Records can inherit from records (or object). Not from non-record classes:

```csharp
public class Base { }
public record Sub : Base { }   // ❌ — error: record can only inherit from record or object
```

Record class hierarchy is closed to record class. Non-record classes can implement interfaces a record provides; otherwise they're separate hierarchies.

---

## Sealing records

```csharp
public sealed record Person(string Name, int Age);
```

`sealed` prevents further inheritance. Synthesized equality avoids the virtual dispatch (no need to check EqualityContract subtypes). Slightly faster.

For final, leaf records used in collections heavily: seal them. Otherwise leave open.

---

## Record validation

C# 10 added the "validating primary constructor body" for records:

```csharp
public record Person(string Name, int Age) {
    public Person {                              // notice: no parens
        if (Age < 0) throw new ArgumentOutOfRangeException(nameof(Age));
        ArgumentException.ThrowIfNullOrWhiteSpace(Name);
    }
}
```

The `public Person { ... }` block runs **after** positional assignment, validating the values. If you throw, the object isn't constructed (caller gets the exception).

This is the cleanest validation pattern for records.

You can also normalize:

```csharp
public record Email(string Address) {
    public Email {
        Address = Address.Trim().ToLowerInvariant();
    }
}
```

The positional parameter is assignable INSIDE the validator (with care — it's `init` semantics).

---

## Record struct

```csharp
public record struct Point(int X, int Y);
public readonly record struct Coord(double X, double Y);   // immutable
```

Record-struct = value type with synthesized equality + ToString + Deconstruct.

Differences from `record class`:
- No inheritance (structs are sealed).
- Cheaper allocation (stack or inline).
- No EqualityContract (no inheritance to worry about).
- Equality just compares fields.

For small value-equal types, prefer `readonly record struct` over `class record`. Faster, less GC pressure.

---

## When to use record vs class

| Use record when | Use class when |
|---|---|
| Type represents data (DTO, event, message) | Type represents an entity with identity |
| Value equality makes sense | Reference equality is correct |
| Mostly immutable | Mutable, complex behavior |
| Want `with`-style updates | Don't need non-destructive mutation |
| Pattern-match heavily | Pattern matching unimportant |

Domain entities (Customer, Order) usually want reference identity → class.
DTOs, commands, events → record.

---

## Internals — IL details

The synthesized members compile to ordinary IL. Reflection sees a record like any other class, just with the extra members (`PrintMembers`, `EqualityContract`).

The `EqualityContract` property is `protected virtual` so derived records can override. The default returns `typeof(MyType)`.

For sealed records, the JIT often devirtualizes the Equals chain → near-array-fast comparisons.

`<Clone>$` has the `[CompilerGenerated]` attribute. Reflection can find it but the name itself (with `<>$`) is illegal in C# source.

---

## Performance notes

- Record equality is competitive with hand-written.
- HashCode.Combine + structural per-property compare.
- For records with many properties, equality is O(N) in property count.
- For records used as Dictionary keys, equality + hash combine well.
- `with` allocates one copy. For deeply nested records, multiple allocations possible.
- Sealed records perf-better in tight equality loops.

---

## Common patterns

### Versioned commands (CQRS)

```csharp
public abstract record Command;
public sealed record CreateUser(string Email, string Password) : Command;
public sealed record UpdateProfile(int UserId, string DisplayName) : Command;

string Describe(Command c) => c switch {
    CreateUser cu => $"Create {cu.Email}",
    UpdateProfile up => $"Update {up.UserId}",
    _ => throw new()
};
```

Records + pattern matching = clean dispatch.

### Result types

```csharp
public abstract record Result<T>;
public sealed record Ok<T>(T Value) : Result<T>;
public sealed record Err<T>(string Reason) : Result<T>;

string Handle<T>(Result<T> r) => r switch {
    Ok<T>(var v) => $"got {v}",
    Err<T>(var e) => $"error: {e}",
    _ => throw new()
};
```

Functional-style error handling. Awkward generic constraints sometimes, but clear semantics.

### Domain events

```csharp
public abstract record DomainEvent(DateTime OccurredAt);
public sealed record OrderPlaced(int OrderId, DateTime OccurredAt) : DomainEvent(OccurredAt);
public sealed record OrderCancelled(int OrderId, string Reason, DateTime OccurredAt) : DomainEvent(OccurredAt);
```

Records for events; collect, replay, dispatch via switch.

---

## Common bugs

### Reference-type members defeat value equality

Already covered: `List<T>` and other mutable references compare by reference. Use immutable collections or override.

### Modifying init-only properties

```csharp
var p = new Person("Alice", 30);
p.Age = 31;   // ❌ init-only — can't write outside ctor / object initializer
```

Use `with` to create a copy with the changed value:
```csharp
var p2 = p with { Age = 31 };
```

### Capturing `this` in record methods

```csharp
public record Person(string Name) {
    public Func<string> Greet => () => $"Hello, {Name}";   // captures this
}
```

Same as class. The lambda captures the record instance. If the record is long-lived, no problem. For ephemeral records, the lambda keeps it alive.

---

## Summary

- Records synthesize equality, `with`, deconstruction, ToString — saving boilerplate.
- `EqualityContract` enables inheritance-aware equality.
- `<Clone>$` + copy constructor power `with` expressions.
- Reference-type members compare by reference; use immutables for true value equality.
- `record struct` is even faster — prefer for small value-like data.
- Use for DTOs, events, commands; use class for entities with identity.

→ Next: [04-RequiredMembers.md](04-RequiredMembers.md)
