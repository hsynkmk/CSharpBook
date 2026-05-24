# Anonymous Types

## What it is

An **anonymous type** is a class declared inline by listing only its members. The compiler generates an unnamed reference type with `init`-only properties. They were introduced in C# 3 (2007) to support LINQ projections, and they remain useful for ad-hoc data shaping.

```csharp
var p = new { Name = "Alice", Age = 30 };
Console.WriteLine(p.Name);   // Alice
Console.WriteLine(p.Age);    // 30
Console.WriteLine(p);        // { Name = Alice, Age = 30 }
```

You don't write a class. The compiler generates one. It has a synthesized name, value-based equality, a sensible `ToString`, and `init`-only properties.

---

## Why it exists

LINQ projections create new shapes constantly:

```csharp
var customerSummaries = customers
    .Where(c => c.IsActive)
    .Select(c => new { c.Id, c.Name, OrderCount = c.Orders.Count });
```

Without anonymous types, you'd have to declare a `CustomerSummary` class for every shape — most of which are used only once. Anonymous types let you express "I want an int, a string, and another int, with these names" without ceremony.

Modern alternatives — **records** and **tuples** — fill similar niches with more power. Anonymous types still have a place, mostly in LINQ.

---

## Syntax

```csharp
var p = new { Name = "Alice", Age = 30 };
var q = new { Name = "Bob", Age = 25 };

// Inferred member names from variables
string name = "Carol";
int age = 40;
var r = new { name, age };   // -> { name = Carol, age = 40 }

// Nested
var addr = new { Street = "1 Main", City = "Springfield" };
var person = new { Name = "Alice", Address = addr };
```

Rules:
- All members are **read-only / init-only** properties.
- Member names must be unique within the type.
- Must be assigned to `var` — there's no way to name the type explicitly.

You **cannot**:
- Add methods.
- Add explicit constructors.
- Specify modifiers (public, etc.).
- Use them as method parameters or return types unless you box them to `object` / `dynamic` or use generic methods.

---

## Equality

Anonymous types implement value-based equality and `GetHashCode` automatically:

```csharp
var a = new { Name = "Alice", Age = 30 };
var b = new { Name = "Alice", Age = 30 };
Console.WriteLine(a == b);          // false! `==` is reference equality on anonymous types
Console.WriteLine(a.Equals(b));     // true — value equality via overridden Equals
Console.WriteLine(a.GetHashCode() == b.GetHashCode());  // true
```

A quirk: `Equals` and `GetHashCode` are value-based, but `==` is NOT overloaded. The compiler doesn't synthesize `operator ==` for anonymous types. Use `.Equals` or wrap in a dictionary that uses `EqualityComparer<T>.Default` (which calls `Equals`).

---

## Two anonymous types are the "same" if...

The compiler treats two anonymous-type expressions as **the same type** if:
- Same property names, in the same order.
- Same property types.

```csharp
var a = new { X = 1, Y = 2 };
var b = new { X = 1, Y = 2 };
var c = new { Y = 2, X = 1 };   // different order = different type!

a = b;   // OK
a = c;   // ❌ compile error — different anonymous types
```

This makes anonymous types **structurally typed** by source — two equal-looking literals share a generated class, but reordering the members generates a different class.

In LINQ projections, this means consecutive `Select(... => new { ... })` calls that share the same shape share the same backing type.

---

## What the compiler generates

For `new { Name = "Alice", Age = 30 }`, the compiler emits roughly:

```csharp
internal sealed class <>f__AnonymousType0<TName, TAge> {
    public TName Name { get; }
    public TAge Age { get; }
    public <>f__AnonymousType0(TName name, TAge age) {
        Name = name; Age = age;
    }
    public override bool Equals(object? obj) {
        var o = obj as <>f__AnonymousType0<TName, TAge>;
        return o != null
            && EqualityComparer<TName>.Default.Equals(Name, o.Name)
            && EqualityComparer<TAge>.Default.Equals(Age, o.Age);
    }
    public override int GetHashCode() => HashCode.Combine(Name, Age);
    public override string ToString() => $"{{ Name = {Name}, Age = {Age} }}";
}
```

- **Sealed**, generic in each property's type.
- Synthesized name `<>f__AnonymousType0<...>` (illegal in C# source so it won't clash).
- One generic instantiation per distinct property-name+order combination.

Two anonymous expressions with the same property names and types share the same generic generated class — they're just different generic instantiations.

---

## Anonymous types in LINQ

The biggest real use:

```csharp
var summary = orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new {
        CustomerId = g.Key,
        OrderCount = g.Count(),
        Total = g.Sum(o => o.Total)
    })
    .OrderByDescending(x => x.Total);

foreach (var s in summary) {
    Console.WriteLine($"{s.CustomerId}: {s.OrderCount} orders, ${s.Total}");
}
```

When working with `IQueryable<T>` (EF Core), the provider often translates anonymous-type projections to `SELECT` lists in the SQL:

```sql
SELECT CustomerId, COUNT(*) AS OrderCount, SUM(Total) AS Total
FROM Orders GROUP BY CustomerId
```

EF Core sees the anonymous projection and shapes the SQL accordingly. Returning a custom class also works but requires more code.

---

## Anonymous types vs alternatives

|  | Anonymous type | Tuple | Record |
|---|---|---|---|
| Reference vs value | Reference | Value | Reference (or struct) |
| Named members | Yes | Yes (since C# 7.1) | Yes |
| Equality | Value (Equals) | Value (==) | Value (==) |
| Can be a parameter | Awkward | Yes | Yes |
| Can be a return type | Awkward | Yes | Yes |
| `with` expressions | No | No | Yes |
| Heap allocation | Yes | No (usually) | Yes |
| Compiler-known name | No | `ValueTuple<...>` | Your name |

**Default choice today**: use **tuples** for local ad-hoc grouping; **records** for things crossing API boundaries; **anonymous types** mainly for LINQ projections where you don't need to return the shape.

---

## Crossing scope

Anonymous types are hard to return or pass around because you can't write their type name:

```csharp
// 🚨 doesn't work
public { Name = "...", Age = 0 } GetPerson() { ... }
```

Tricks:
- **Return `dynamic`**: loose typing, slow, no IntelliSense.
- **Box to `object`**: works but caller can't access properties without reflection.
- **Use a generic helper that infers the type**:
  ```csharp
  public T Create<T>(T template) => template;   // type inferred from caller
  ```
  Rarely useful in practice.

If you need to pass a shape around, **use a record or named class**.

---

## Internals — IL and memory

A `new { X = 1, Y = "hi" }` allocates:

```
[ sync block | MT ptr | X | Y ]
```

with X and Y being `int` and `string` respectively. Size on 64-bit: 16 (header) + 4 (X) + 4 (padding) + 8 (Y reference) = 32 bytes.

In IL:

```il
ldc.i4.1
ldstr "hi"
newobj <>f__AnonymousType0`2<int32, string>::.ctor(!0, !1)
stloc.0
```

A typical heap allocation — anonymous types are reference types, so they incur Gen0 pressure in tight loops.

### Hot-path concerns

For LINQ in hot paths, anonymous types in `Select(x => new { ... })` allocate per element:

```csharp
var sums = items.Select(x => new { x.Id, x.Total }).Sum(s => s.Total);
// Allocates one anonymous object per item — wasted work
```

Better:

```csharp
var total = items.Sum(x => x.Total);   // no projection needed
```

Or if you need both the Id and Total, use a tuple to avoid the allocation:

```csharp
var sums = items.Select(x => (x.Id, x.Total));   // value tuple, no heap
```

---

## Quirks and gotchas

### Member name inference vs explicit

```csharp
var x = 5;
var a = new { x };          // member name is "x"
var b = new { Value = x };  // member name is "Value"
```

Inference fails when the expression isn't a simple variable or property:
```csharp
var c = new { x + 1 };       // ❌ — can't infer name from x+1
var d = new { Sum = x + 1 }; // ✓
```

### `with` expression DOES work on anonymous types (since C# 10)

```csharp
var p = new { Name = "Alice", Age = 30 };
var older = p with { Age = 31 };   // C# 10+ — works on anonymous types!
```

Useful, though records remain better for most "shape that needs `with`" scenarios.

### Anonymous types are immutable

You can't reassign properties:

```csharp
var p = new { Age = 30 };
p.Age = 31;   // ❌ — Age is init-only
```

Always use `with` (above) to create a modified copy.

### Anonymous arrays

You can create an array of anonymous-typed values, as long as all elements are the same anonymous shape:

```csharp
var people = new[] {
    new { Name = "Alice", Age = 30 },
    new { Name = "Bob", Age = 25 }
};

foreach (var p in people) Console.WriteLine(p.Name);
```

### Comparing values across LINQ provider boundaries

When you project to an anonymous type in EF Core and then materialize:

```csharp
var summary = await db.Orders
    .GroupBy(o => o.CustomerId)
    .Select(g => new { CustomerId = g.Key, Count = g.Count() })
    .ToListAsync();
```

The `summary` is a `List<>f__AnonymousType0<int, int>>`. Returning this from a controller is fine (JSON serializer reflects the properties), but you can't easily mention the type name.

---

## When to use anonymous types

✓ Inline LINQ projections — quick shapes you'll consume immediately.
✓ Local intermediate shapes you'll iterate and throw away.
✓ Debugging — quick `ToString` for arbitrary combinations.
✓ Shaping JSON responses in a controller where reflection-based serialization will work.

✗ Crossing API boundaries — use a record or class.
✗ Long-lived data — heap pressure from many short-lived objects.
✗ Anywhere you'd benefit from naming the type for documentation.

---

## Common bugs

- **`==` vs `Equals`** — `==` is reference equality. Use `.Equals(other)` for value comparison.
- **Returning anonymous types from methods** — clumsy. Use records instead.
- **Allocating one per LINQ element in hot paths** — measure before assuming it's free.
- **Inferring names from indexed access** — `new { arr[0] }` doesn't infer. Be explicit: `new { First = arr[0] }`.

---

## Performance

- Each `new { ... }` is a heap allocation.
- Generated class is fully sealed; equality and hashing are direct (no virtual indirection).
- For repeated shapes, the JIT generates one class; instantiating it is just `newobj` + field assignments.
- For LINQ on `IEnumerable<T>` (LINQ-to-Objects), anonymous-type projections allocate per item — fine in many cases but a hot-loop trap.
- For `IQueryable<T>`, anonymous-type projections often don't allocate per row — the provider rewrites them into SELECT lists.

---

## Quick comparison

```csharp
// Anonymous type — reference, no name, init-only, value Equals
var x = new { Name = "Alice", Age = 30 };

// Tuple — value, structural, names are sugar
var y = (Name: "Alice", Age: 30);

// Record — reference (or struct), named, value Equals, supports with
public record Person(string Name, int Age);
var z = new Person("Alice", 30);
```

Most of the time today, **tuples** or **records** win. Anonymous types remain a nice tool for short-lived LINQ projections.

→ Continue to: [Questions.md](Questions.md)
