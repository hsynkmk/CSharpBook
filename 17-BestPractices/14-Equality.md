# Equality & Hashing

## Why this deserves its own treatment

Equality in C# is a minefield: there are *four* ways to compare ("`==`", `Equals`, `ReferenceEquals`, `IEquatable<T>.Equals`), reference vs value semantics differ, and `Equals` and `GetHashCode` must agree or your objects silently break in dictionaries and sets. Getting it wrong produces some of the most baffling bugs in C# — items that "disappear" from a `HashSet`, dictionary lookups that miss, LINQ `Distinct` that doesn't dedupe. This file is the correctness guide.

---

## The equality members

```csharp
object.Equals(a, b)           // static null-safe; calls a.Equals(b)
a.Equals(b)                   // virtual instance method (override this)
a == b                        // operator; static dispatch (override deliberately)
ReferenceEquals(a, b)         // identity — same object? never overridable
IEquatable<T>.Equals(b)       // strongly-typed, no boxing for value types
```

- **Reference types** default: `Equals`/`==` compare **identity** (same object).
- **Value types** default: `Equals` compares **field-by-field** (via reflection — slow; override for performance), and `==` is **not defined** unless you write it.
- **`string`** overrides both to compare by **value** (content).
- **`record`** synthesizes value equality for you (the modern default — see below).

---

## The cardinal rule: `Equals` and `GetHashCode` must agree

> If `a.Equals(b)` is true, then `a.GetHashCode() == b.GetHashCode()` **must** be true.

Hash-based collections (`Dictionary`, `HashSet`, LINQ `Distinct`/`GroupBy`) first bucket by hash code, then compare with `Equals` within the bucket. If two equal objects have different hash codes, they land in different buckets and are **never compared** — so the collection treats them as different.

```csharp
// ✗ — overrides Equals but NOT GetHashCode → broken in HashSet/Dictionary
public class Point {
    public int X, Y;
    public override bool Equals(object? o) => o is Point p && p.X == X && p.Y == Y;
    // no GetHashCode override → uses default identity hash → equal points hash differently
}

var set = new HashSet<Point>();
set.Add(new Point { X = 1, Y = 2 });
set.Contains(new Point { X = 1, Y = 2 });   // FALSE! equal but different hash → different bucket
```

**Always override both together, or neither.** The compiler even warns (CS0659) if you override `Equals` without `GetHashCode`.

---

## Just use `record` (the modern default)

For most value-semantics types, don't hand-write equality — use a `record` (or `record struct`). It synthesizes a **correct, consistent** `Equals`, `GetHashCode`, `==`/`!=`, and `IEquatable<T>`:

```csharp
public record Point(int X, int Y);

var a = new Point(1, 2);
var b = new Point(1, 2);
a == b;                  // true — value equality, synthesized
a.Equals(b);             // true
a.GetHashCode() == b.GetHashCode();   // true — guaranteed consistent

var set = new HashSet<Point> { a };
set.Contains(b);         // true — works correctly
```

Records handle the hard parts (consistency, `IEquatable<T>`, operators) correctly. **Reach for `record` before writing `Equals`/`GetHashCode` by hand.** See [Chapter 03 §03](../03-TypeSystem/03-Records.md).

```csharp
public readonly record struct Money(decimal Amount, string Currency);  // value type, value equality
```

---

## When you must hand-write equality

If a `record` doesn't fit (you need a class with custom equality semantics — e.g., equality on a subset of fields), do it correctly:

```csharp
public sealed class Customer : IEquatable<Customer> {
    public required int Id { get; init; }
    public required string Name { get; init; }

    // Identity is the Id only (business rule: same Id = same customer)
    public bool Equals(Customer? other) =>
        other is not null && Id == other.Id;

    public override bool Equals(object? obj) => Equals(obj as Customer);

    public override int GetHashCode() => Id.GetHashCode();   // consistent with Equals

    public static bool operator ==(Customer? a, Customer? b) =>
        a is null ? b is null : a.Equals(b);
    public static bool operator !=(Customer? a, Customer? b) => !(a == b);
}
```

Checklist for correct hand-written equality:
1. Implement `IEquatable<T>` (strongly-typed `Equals(T)`) — avoids boxing for value types and is what `Dictionary`/`HashSet` prefer.
2. Override `object.Equals(object?)` to delegate to the typed version.
3. Override `GetHashCode()` consistently with `Equals` (use the same fields).
4. Optionally overload `==`/`!=` (and document that you did).
5. Handle `null` correctly throughout.

### Building hash codes — use `HashCode.Combine`

```csharp
public override int GetHashCode() => HashCode.Combine(X, Y, Z);   // ✓ — correct, well-distributed

// For many fields or collections:
public override int GetHashCode() {
    var hash = new HashCode();
    hash.Add(Name);
    foreach (var item in Items) hash.Add(item);
    return hash.ToHashCode();
}
```

Never hand-roll `X ^ Y` or `X + Y` — poor distribution causes hash collisions and degrades `Dictionary`/`HashSet` to O(n). `HashCode.Combine` (and the `HashCode` struct) produce well-distributed hashes. See [Chapter 07 §11](../07-Collections/11-EqualityContract.md).

---

## `GetHashCode` must be stable while the object is in a hash collection

```csharp
// ✗ — hashing on a MUTABLE field
public class BadKey {
    public int Value { get; set; }
    public override int GetHashCode() => Value.GetHashCode();
    public override bool Equals(object? o) => o is BadKey k && k.Value == Value;
}

var dict = new Dictionary<BadKey, string>();
var key = new BadKey { Value = 1 };
dict[key] = "x";
key.Value = 2;            // ⚠ — mutated while a dictionary key! hash changed
dict.ContainsKey(key);    // FALSE — looks in the wrong bucket; the entry is "lost"
```

**Hash keys must be immutable** (at least the fields used in `GetHashCode`). Mutating a key after insertion corrupts the collection. This is a strong argument for **immutable types as keys** — records and `readonly struct`s are ideal. See [Chapter 17 §07](07-ImmutabilityPatterns.md).

---

## Custom equality without changing the type: `IEqualityComparer<T>`

When you can't (or shouldn't) bake equality into the type — e.g., you want case-insensitive string keys, or different equality in different contexts — supply an `IEqualityComparer<T>`:

```csharp
// Case-insensitive dictionary without changing string
var dict = new Dictionary<string, int>(StringComparer.OrdinalIgnoreCase);
dict["Hello"] = 1;
dict["HELLO"];   // 1 — comparer-driven equality

// Custom comparer for your type
public class CustomerByEmailComparer : IEqualityComparer<Customer> {
    public bool Equals(Customer? a, Customer? b) =>
        string.Equals(a?.Email, b?.Email, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(Customer c) =>
        c.Email.GetHashCode(StringComparison.OrdinalIgnoreCase);
}

var unique = customers.Distinct(new CustomerByEmailComparer());
var set = new HashSet<Customer>(new CustomerByEmailComparer());
```

`IEqualityComparer<T>` decouples equality from the type — pass it to `Dictionary`, `HashSet`, `Distinct`, `GroupBy`, `ToDictionary`, etc. Use `StringComparer.Ordinal`/`OrdinalIgnoreCase` for string keys (faster and culture-stable — see [Chapter 13 §08](../13-IO/08-Globalization.md)).

---

## String equality — be explicit

```csharp
// ✗ — culture-sensitive by default (Turkish-I problem, slower)
if (a == b) { ... }                                  // ordinal for ==, but for methods:
if (a.Equals(b)) { ... }                             // ordinal

// ✓ — explicit comparison for identifiers/keys
a.Equals(b, StringComparison.Ordinal);
a.Equals(b, StringComparison.OrdinalIgnoreCase);
```

For string `==` the comparison is ordinal, but methods like `string.Equals(b)` and sorting can be culture-sensitive. For identifiers, keys, and protocol strings, specify `StringComparison.Ordinal[IgnoreCase]` explicitly. See [Chapter 13 §08](../13-IO/08-Globalization.md).

---

## `record` equality gotchas

```csharp
// ✗ — record with a collection member: equality compares the LIST BY REFERENCE
public record Order(int Id, List<Item> Items);
new Order(1, [a]) == new Order(1, [a]);   // FALSE if the lists are different instances!

// ✓ — use a value-equatable member, or override equality for the collection
public record Order(int Id, ImmutableArray<Item> Items);   // still reference-ish; see below
```

Record value equality compares members with *their* `Equals`. A `List<T>`/array member compares by **reference**, not contents — so two records with equal-but-distinct lists are unequal. For content equality of collections in records, either compare manually (override `Equals`) or use a sequence-equality wrapper. This surprises people constantly.

Also: record `==` on `null` is null-safe (synthesized operators handle it), but `record struct` equality includes all fields (can't customize which without overriding).

---

## Common bugs / gotchas

### Overriding `Equals` without `GetHashCode`

The #1 equality bug — breaks `HashSet`/`Dictionary`/`Distinct`. Compiler warns (CS0659). Override both or use a `record`.

### Mutating a hash key

Changing a field used in `GetHashCode` while the object is a dictionary/set key corrupts the collection. Use immutable keys.

### Poor hash distribution

`X ^ Y` or constant hashes cause collisions → O(n) collections. Use `HashCode.Combine`.

### `ReferenceEquals` for value comparison

`ReferenceEquals(a, b)` is identity, not value — false for distinct-but-equal objects (and meaningless/always-boxes for value types). Use `==`/`Equals`.

### Records with mutable/collection members

Value equality compares collection members by reference. Two records with equal contents but different list instances are unequal. Use immutable value-equatable members or override.

### Forgetting `IEquatable<T>` for value-type keys

Without it, `Dictionary`/`HashSet` use the slow reflection-based `ValueType.Equals` and box. Implement `IEquatable<T>` (or use `record struct`).

---

## Summary

- **Use `record`/`record struct`** for value semantics — it synthesizes correct, consistent `Equals`/`GetHashCode`/`==`/`IEquatable<T>`. Reach for it before hand-writing equality.
- The **cardinal rule**: `Equals` and `GetHashCode` must agree — override both or neither (else hash collections silently break).
- Hand-written equality: implement `IEquatable<T>`, override `object.Equals` + `GetHashCode` (use `HashCode.Combine`), handle null, optionally `==`/`!=`.
- **Hash keys must be immutable** — mutating a key field corrupts the collection.
- Use **`IEqualityComparer<T>`** (e.g., `StringComparer.OrdinalIgnoreCase`) to customize equality without changing the type.
- Be explicit with **string comparison** (`StringComparison.Ordinal` for keys); beware **record equality with collection members** (compared by reference).

→ Next: [Questions.md](Questions.md)
