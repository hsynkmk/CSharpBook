# Encapsulation

## What it is

**Encapsulation** is the practice of bundling data with the code that operates on it, and **hiding the implementation details** behind a smaller public surface. It's the "E" in the classic OOP acronym (Encapsulation, Inheritance, Polymorphism), and arguably the most important one — the only one that pays off in **every** codebase.

```csharp
// Encapsulated
public class BankAccount {
    private decimal _balance;
    public decimal Balance => _balance;        // read-only view
    public void Deposit(decimal amt) {
        if (amt <= 0) throw new ArgumentException();
        _balance += amt;
    }
}

// Not encapsulated — fragile
public class BadAccount {
    public decimal Balance;                    // anyone can write anything
}
```

`BankAccount`'s invariants (balance never negative, deposits positive) are enforced. `BadAccount` has none — any code can corrupt it.

---

## Why it exists

Three benefits, in order of importance:

1. **Invariants**. The object guarantees its state is consistent because only its methods can modify it.
2. **Versioning**. You can change the internal representation freely without breaking callers — as long as the public surface stays the same.
3. **Reasoning**. You can read a single class to understand what mutations are possible, instead of grepping the whole codebase.

---

## Tools of encapsulation

### Access modifiers

Most basic. Make fields `private`, expose what's needed via properties or methods:

```csharp
public class Order {
    private readonly List<Item> _items = new();
    public IReadOnlyList<Item> Items => _items;
    public void AddItem(Item item) {
        ArgumentNullException.ThrowIfNull(item);
        _items.Add(item);
    }
    public bool RemoveItem(int id) {
        var idx = _items.FindIndex(i => i.Id == id);
        if (idx < 0) return false;
        _items.RemoveAt(idx);
        return true;
    }
}
```

External callers can list and add/remove items through methods; they can't bypass validation or mutate `_items` directly.

### Properties (over public fields)

Properties let you add behavior to what looks like field access:

```csharp
public int Age {
    get => _age;
    set {
        if (value < 0) throw new ArgumentOutOfRangeException();
        _age = value;
    }
}
```

The validation is part of the contract, not a separate `SetAge` method callers might forget.

### Read-only views over mutable storage

Internal `List<T>` exposed as `IReadOnlyList<T>`:

```csharp
private readonly List<string> _tags = new();
public IReadOnlyList<string> Tags => _tags;
```

Or, since C# 12, `ReadOnlyCollection<T>` / `ImmutableArray<T>` if even *read access* should be guaranteed snapshot.

### Defensive copies

When a collection enters your object, copy it so the caller's later changes don't affect you:

```csharp
public Team(IEnumerable<Player> players) {
    _players = new List<Player>(players);   // copy
}
```

When a value leaves your object, copy on the way out if the consumer might mutate it:

```csharp
public IReadOnlyList<Player> Roster => _players.ToList();   // snapshot copy
```

Most of the time the read-only interface is enough. Copy when you have a really paranoid invariant.

### Init-only and required

C# 9 `init`, C# 11 `required` — make properties settable only at construction:

```csharp
public class Config {
    public required string Host { get; init; }
    public int Port { get; init; } = 443;
}

new Config { Host = "x.com" }.Port = 8080;   // ❌ — can't mutate after construction
```

Combines well with records.

### Sealed

Prevent inheritance from being a backdoor:

```csharp
public sealed class HardenedString { ... }
```

Inheritance can break invariants — a subclass might override a virtual method to do something unexpected. Sealing prevents that.

### File-scoped types (C# 11+)

Hide helpers from other files in the same assembly:

```csharp
file class InternalHelper { ... }   // only this file can use it
```

Useful for source generators or implementation details you want to make sure nobody outside this file can touch.

---

## What to expose vs hide

A good public API:
- States **what** the object can do.
- Doesn't reveal **how** it does it.
- Forms a small, coherent set.

```csharp
// Good — public API states intent
public class CsvParser {
    public IEnumerable<Record> Parse(string text);
    public Encoding Encoding { get; init; } = Encoding.UTF8;
}

// Bad — internal details leak
public class CsvParser {
    public char Delimiter;
    public int CurrentLine;
    public StringBuilder Buffer;
    public List<Record> _accumulated;
    public bool _inQuotedField;
    public bool _seenHeader;
}
```

Each leak is a future maintenance burden — you can't change `_inQuotedField` to `_state` without breaking callers who somehow touched it.

---

## Encapsulation vs object initializers

Object initializers + auto-properties make encapsulation **looser**:

```csharp
var p = new Person { Name = "Alice", Age = 30 };

public class Person {
    public string Name { get; set; }
    public int Age { get; set; }
}
```

There's no invariant — anyone can mutate `Age` to anything. For DTOs and configuration objects, that's fine. For domain entities with business rules, lock down:

```csharp
public class Person {
    public Person(string name, int age) {
        ArgumentException.ThrowIfNullOrWhiteSpace(name);
        ArgumentOutOfRangeException.ThrowIfNegative(age);
        Name = name;
        Age = age;
    }
    public string Name { get; }
    public int Age { get; private set; }
    public void HaveBirthday() => Age++;
}
```

Compare: caller can't put a `Person` into an invalid state.

---

## Encapsulation in records

Records are typically value-equality DTOs — by default they expose all data:

```csharp
public record Person(string Name, int Age);

var p = new Person("Alice", 30) with { Age = 31 };
```

If you need invariants, validate in the body:

```csharp
public record Person {
    public string Name { get; init; }
    public int Age { get; init; }

    public Person(string name, int age) {
        if (age < 0) throw new ArgumentException();
        Name = name; Age = age;
    }
}
```

Or use positional records with validation in the body:

```csharp
public record Person(string Name, int Age) {
    public Person {
        if (Age < 0) throw new ArgumentOutOfRangeException();
    }
}
```

(That `public Person { ... }` is a special "init validator" that runs after positional assignment.)

---

## Internals — what enforces access at runtime?

Access modifiers are enforced **at compile time**, not strictly at runtime. The compiler emits IL with metadata flags:

```il
.field private int32 _balance
.method public hidebysig instance void Deposit(decimal amt)
```

The CLR honors these flags via:
- The verifier (when assemblies are partially trusted).
- Reflection respects access by default — but with `BindingFlags.NonPublic`, you can access privates from outside.

So the rules are **a guideline that reflection can break**. For real security, you need higher walls (process boundaries, sandboxing). For correctness, the compiler is enough.

### What about JIT?

The JIT respects metadata. It doesn't emit private-only fast paths — but it can inline trivial accessors regardless of access level (the rules apply to whoever is calling, not whoever is being called).

### Reflection bypass

```csharp
var f = typeof(BankAccount).GetField("_balance",
    BindingFlags.NonPublic | BindingFlags.Instance);
f!.SetValue(account, -1_000_000m);   // mutates private state!
```

Don't use this for normal code. It's used in:
- Testing (sparingly).
- Serializers that need to read private state.
- Reflection-emit / source generators that bridge between domains.

It's a hole through encapsulation. Don't design your code assuming nobody will reflect through it.

---

## Common patterns

### Repository / Service hides storage

```csharp
public interface IUserRepo {
    Task<User?> GetAsync(int id);
    Task SaveAsync(User u);
}

public class SqlUserRepo : IUserRepo { /* DB code */ }
public class MemoryUserRepo : IUserRepo { /* dictionary */ }
```

Consumers know nothing about whether storage is SQL, in-memory, or somewhere else.

### Internal builders

```csharp
public class HttpRequest {
    public required Uri Url { get; init; }
    // ...
    public class Builder {
        // ...
        public HttpRequest Build() => /* validated */;
    }
    public static Builder New() => new();
}

var req = HttpRequest.New()
    .WithUrl("https://example.com")
    .Build();
```

The builder accumulates state; the final object enforces invariants.

### Aggregate root (DDD)

```csharp
public class Order {
    private readonly List<LineItem> _items = new();
    public IReadOnlyList<LineItem> Items => _items;

    public void AddItem(Product p, int qty) {
        // validates and possibly merges with existing line
        var existing = _items.FirstOrDefault(i => i.ProductId == p.Id);
        if (existing != null) existing.IncreaseQty(qty);
        else _items.Add(new LineItem(p, qty));
    }
}
```

The aggregate (Order) owns its children (LineItems). Callers manipulate through Order methods; LineItems are not accessible for direct mutation from outside.

---

## Anti-patterns

### Public getters with private setters that aren't really private

```csharp
public List<Item> Items { get; private set; } = new();
```

Callers can still **mutate** the list (Add, Remove). The "private set" only prevents reassignment. Expose `IReadOnlyList<T>` instead.

### "Anemic" objects (data without behavior)

```csharp
public class Order {
    public List<LineItem> Items { get; set; } = new();
}

public class OrderService {
    public void AddItem(Order o, Product p, int qty) { ... }
    public decimal Total(Order o) { ... }
}
```

Order is just a bag of data; all behavior lives in OrderService. That's **procedural code in OO clothing**. Push behavior into the data: `order.AddItem(...)`, `order.Total`.

This is called the "Anemic Domain Model" anti-pattern. Common in some architectural styles (DTOs in CQRS) but problematic when used everywhere.

### Public mutable fields

```csharp
public class Foo {
    public int Count;
}
```

No validation, no future flexibility, breaks data binding. Always a property.

### "Friendly" classes via `internal`

`internal` + `InternalsVisibleTo` can be a polite encapsulation breach — your test project pokes at internal state. Use for testing **judiciously**. Don't rely on internals as a normal API surface.

---

## Encapsulation across layers

In large applications:
- **Domain layer** — strong encapsulation: rich entities, internal collections, invariants enforced.
- **API / DTO layer** — loose: records and POCOs with public properties for JSON serialization.
- **Persistence layer** — sometimes loose to satisfy EF Core's reflection needs (often a tension with strict encapsulation).

A common pattern: domain entities are encapsulated, mapped to/from DTOs at boundaries. The DTOs are intentionally dumb data containers.

---

## How encapsulation helps testing

Strong encapsulation forces you to test through the **public API** rather than poking internals. This:
- Tests behavior, not implementation.
- Survives internal refactorings.
- Documents how the class is actually used.

If a test needs to call a private method, that's a smell: maybe the private method should be on its own type, or maybe what you're testing is really an output of public behavior.

---

## Common bugs

- **Mutable collection exposed as get-only property**: callers can `.Add(x)` even if they can't reassign the property.
- **`init` accessor + mutable type**: `public List<int> Items { get; init; } = new();` — caller can mutate the list, just not reassign it.
- **Reference leak via constructor**: storing a passed-in reference type means the caller can later mutate it.
- **Cloning vs reference**: `new Foo(other.SomeList)` doesn't deep-copy.
- **Public auto-property as a domain entity**: invites code to mutate from anywhere. Use methods that enforce invariants.

---

## When to relax encapsulation

Some valid scenarios:
- **DTOs**: data containers for serialization. Public properties are the whole point.
- **Settings / Config**: usually read-only after construction; `init` properties suffice.
- **Test fixtures**: bend rules in tests, not in production code.
- **Value objects** (small immutable records): all properties are public by convention. The immutability provides the safety.

The principle: **the smaller the public surface, the easier to change anything else**. Spend encapsulation effort where invariants matter.

→ Next: [11-NestedAndPartial.md](11-NestedAndPartial.md)
