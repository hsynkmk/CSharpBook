# `required` Members (C# 11+)

## What it is

The `required` modifier marks a property or field as **mandatory** at construction. The compiler enforces that callers must set it via object initializer (or via a constructor decorated with `[SetsRequiredMembers]`).

```csharp
public class Person {
    public required string Name { get; init; }
    public required int Age { get; init; }
    public string? Bio { get; init; }
}

var p = new Person { Name = "Alice", Age = 30 };   // ✓
var bad = new Person { Name = "Alice" };           // ⚠ — Age not set
var bad2 = new Person();                            // ⚠ — Name AND Age missing
```

Added in C# 11. Replaces the awkward "constructor with N params + init-only properties" pattern for DTOs and configuration objects.

---

## Why it exists

Pre-C# 11, "this field must be set" required:
- A constructor with the field as a parameter.
- The compiler couldn't enforce that initialization happened via object initializer.

```csharp
// Pre-C# 11 — manual enforcement
public class Person {
    public Person(string name, int age) { Name = name; Age = age; }
    public string Name { get; }
    public int Age { get; }
}

new Person("Alice", 30);
// But if someone added: new Person { Name = "Alice" } via init-only — no way to enforce Age set
```

With many optional properties, the constructor parameter list grew unwieldy. `required` is the clean solution: declare the property `required` and the compiler tracks initialization at every construction site.

---

## Syntax

```csharp
public required string Name { get; init; }    // required + init
public required string Title { get; set; }     // required + set (more permissive)
public required string Address;                 // required field (rare)
```

`required` can pair with `get/set`, `get/init`, or be a plain field. The compiler checks initialization at every `new` site.

---

## Initialization sites

The compiler enforces initialization via:

### Object initializer

```csharp
new Person { Name = "Alice", Age = 30 };   // ✓
new Person { Age = 30 };                    // ⚠ — Name missing
```

### Constructor with `[SetsRequiredMembers]`

```csharp
public class Person {
    public required string Name { get; init; }
    public required int Age { get; init; }

    public Person() {}   // no [SetsRequired] — caller must use object initializer

    [SetsRequiredMembers]
    public Person(string name, int age) {   // tells compiler: this ctor sets all required members
        Name = name;
        Age = age;
    }
}

new Person("Alice", 30);             // ✓
new Person { Name = "x", Age = 0 };   // ✓
new Person();                          // ⚠
```

`[SetsRequiredMembers]` (from `System.Diagnostics.CodeAnalysis`) tells the compiler this constructor takes responsibility for setting all required members. Trust-based — the compiler doesn't verify the constructor actually sets them.

---

## In records

```csharp
public record Person {
    public required string Name { get; init; }
    public int Age { get; init; }
}

new Person { Name = "Alice" };   // OK — Age defaults to 0
new Person { Age = 30 };          // ⚠ — Name required
```

Records + required = sweet syntax for DTOs:

```csharp
public record Config {
    public required string Host { get; init; }
    public int Port { get; init; } = 443;
    public bool UseSsl { get; init; } = true;
}

new Config { Host = "example.com" };   // ✓ — defaults for others
new Config();                            // ⚠ — Host required
```

Positional records can also use required — but typically positional params are inherently required (you must pass values for them at construction).

---

## What "required" actually does

The compiler emits:
1. A `RequiredMemberAttribute` on the property/field.
2. A check at every `new` site: are all required members initialized via the object initializer?
3. An error if not.

Runtime sees the attribute but doesn't enforce it. Reflection-based instantiation (e.g., JSON deserialization) can bypass required.

For deserializers that respect required (System.Text.Json since .NET 8):

```csharp
public class Person {
    public required string Name { get; init; }
}

JsonSerializer.Deserialize<Person>("{}");   // throws — Name required
JsonSerializer.Deserialize<Person>(@"{""Name"":""Alice""}");  // OK
```

System.Text.Json knows the attribute and enforces during deserialization.

---

## `[SetsRequiredMembers]` rules

When you apply `[SetsRequiredMembers]` to a constructor, you're saying "I take care of all required members." The compiler:
- Allows construction without an object initializer.
- Doesn't verify your constructor actually sets them — trust-based.

You can have multiple constructors; some may set required, others may not:

```csharp
public class Person {
    public required string Name { get; init; }

    public Person() { }   // no [SetsRequired] — initializer must set Name

    [SetsRequiredMembers]
    public Person(string name) { Name = name; }   // sets Name
}
```

For derived records, if you want to inherit `[SetsRequiredMembers]` semantics:

```csharp
public record Animal {
    public required string Name { get; init; }

    [SetsRequiredMembers]
    public Animal(string name) { Name = name; }
}

public record Dog : Animal {
    [SetsRequiredMembers]
    public Dog(string name, string breed) : base(name) {
        Breed = breed;
    }
    public required string Breed { get; init; }
}

new Dog("Rex", "Lab");   // ✓
```

Both must be `[SetsRequiredMembers]`. Otherwise the derived call demands an object initializer.

---

## Required + `init` only properties

The combination is the canonical pattern:

```csharp
public class Order {
    public required int OrderId { get; init; }
    public required string Customer { get; init; }
    public DateTime CreatedAt { get; init; } = DateTime.UtcNow;
}
```

- `required`: caller MUST supply OrderId and Customer.
- `init`: settable only during construction.
- `CreatedAt` has a default — optional for callers.

Result: a compile-time validated, immutable-after-construction object. No constructor needed.

---

## Required vs nullable

```csharp
public required string Name { get; init; }      // must be set; non-null after construction
public required string? Notes { get; init; }    // must be set, but can be null
```

`required` is about "the caller must explicitly set this." It doesn't mean "non-null." Use NRT annotations + required together:

- `required string Name` — caller must set, value must be non-null.
- `required string? Notes` — caller must set, but can pass null explicitly.
- `string? Notes = null;` — optional, defaults to null.

---

## Common patterns

### Configuration objects

```csharp
public class HttpClientOptions {
    public required string BaseUrl { get; init; }
    public TimeSpan Timeout { get; init; } = TimeSpan.FromSeconds(30);
    public int MaxRetries { get; init; } = 3;
}

services.AddOptions<HttpClientOptions>()
    .Configure(opts => {
        opts.BaseUrl = config["api:baseUrl"]!;
        // Compiler enforces BaseUrl set
    });
```

Strongly-typed options with mandatory + defaulted fields.

### Builder-less DTOs

```csharp
public record CreateUserCommand {
    public required string Email { get; init; }
    public required string Password { get; init; }
    public string? Name { get; init; }
    public bool SendWelcomeEmail { get; init; } = true;
}

new CreateUserCommand { Email = "a@b.com", Password = "p" };   // ✓
new CreateUserCommand { Email = "a@b.com" };                    // ⚠ — Password missing
```

No builder pattern needed — the compiler enforces what's mandatory.

### Pre-validated state

```csharp
public class ValidatedInput {
    public required string Trimmed { get; init; }
    public required int Length { get; init; }
}

ValidatedInput Validate(string raw) {
    if (string.IsNullOrWhiteSpace(raw)) throw new ArgumentException();
    var t = raw.Trim();
    return new ValidatedInput { Trimmed = t, Length = t.Length };
}
```

Type system witnesses that validation happened. Anywhere you accept `ValidatedInput`, you know Trimmed and Length are set.

---

## Internals — what's emitted

```csharp
public class Person {
    public required string Name { get; init; }
}
```

In IL:

```il
.property instance string Name {
    .custom instance void [System.Runtime]System.Runtime.CompilerServices.RequiredMemberAttribute::.ctor()
    .get instance string Person::get_Name()
    .set instance void modreq([System.Runtime]System.Runtime.CompilerServices.IsExternalInit) Person::set_Name(string)
}

.class public auto ansi Person {
    .custom instance void [System.Runtime]System.Runtime.CompilerServices.RequiredMemberAttribute::.ctor()   // marks class as having required members
}
```

The attribute on the property + the class. Compilers reading the metadata know to enforce initialization.

For consumer enforcement:

```csharp
new Person { Name = "Alice" };
```

Compiles to:

```il
newobj instance void Person::.ctor()
... but the compiler ALSO checked that "Name" appears in the object initializer
... else error CS9035: required member 'Person.Name' must be set
```

The check is at compile time only. Runtime treats the object normally.

---

## Required and inheritance

```csharp
public class Animal {
    public required string Name { get; init; }
}

public class Dog : Animal {
    public required string Breed { get; init; }
}

new Dog { Name = "Rex", Breed = "Lab" };   // ✓
new Dog { Breed = "Lab" };                  // ⚠ — Name (inherited) required
new Dog();                                   // ⚠ — both required
```

Required propagates through inheritance. All required members (inherited + declared) must be initialized.

To bypass (rare):
```csharp
public class Dog : Animal {
    [SetsRequiredMembers]
    public Dog() {   // takes responsibility for ALL required, including inherited
        Name = "Unknown";
    }
}
```

---

## Common bugs

### Skipping required via reflection

```csharp
var p = (Person)Activator.CreateInstance(typeof(Person))!;
// p.Name is null (not initialized) — required not enforced at runtime
```

Reflection bypasses required. So does `FormatterServices.GetUninitializedObject`. If your data path goes through these, validate explicitly.

System.Text.Json since .NET 8 respects required. Most deserializers don't (yet).

### Forgetting `[SetsRequiredMembers]` on a constructor

```csharp
public class Person {
    public required string Name { get; init; }

    public Person(string name) { Name = name; }   // ⚠ — but no [SetsRequired]
}

new Person("Alice");   // ⚠ — error: must use object initializer or [SetsRequiredMembers] ctor
```

The constructor sets Name, but the compiler doesn't know that. Add `[SetsRequiredMembers]`.

### Inheriting required without `[SetsRequiredMembers]` on derived

Same logic. Derived constructor must be marked if it sets all required.

### Required + parameterless ctor + field initializer

```csharp
public class Person {
    public required string Name { get; init; } = "default";
}

new Person();   // ⚠ — still required, even with a default
```

Default values don't satisfy required. `required` means "the caller must set it" — even if there's a fallback. To make truly optional with a default, drop `required`.

---

## When to use required

✓ DTOs where some fields are mandatory.
✓ Configuration objects with required fields.
✓ Domain types where construction-time invariants matter.
✓ Replacing fat constructor parameter lists.

✗ Properties with sensible defaults — just use `init` + initializer.
✗ Internal types where you control all callers.
✗ When you'd rather encode validation in a constructor body.

---

## Migration strategy

For an existing codebase:
1. Identify constructor params that are mandatory.
2. Replace constructor with required init properties.
3. Add `[SetsRequiredMembers]` to existing constructors if you keep them.
4. Update call sites to use object initializer.

The compiler errors are loud — easy to migrate incrementally.

---

## Performance

Zero runtime cost. `required` is compile-time only.

---

## Summary

- `required` marks a property/field as mandatory at construction.
- Compiler enforces initialization via object initializer or `[SetsRequiredMembers]` constructor.
- Pairs naturally with `init` for "set once, must be set" properties.
- Records benefit hugely — clean DTOs without constructor boilerplate.
- Inheritance: required members propagate; all (inherited + declared) must be set.
- Reflection / older deserializers can bypass. System.Text.Json honors it (.NET 8+).
- Replaces "lots of constructor params + init" pattern.

→ Next: [05-FileScopedTypes.md](05-FileScopedTypes.md)
