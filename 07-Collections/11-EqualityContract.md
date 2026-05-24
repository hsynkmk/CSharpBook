# The Equality Contract

## What it is

Every C# type can be compared for equality via `Equals(object?)` and hashed via `GetHashCode()` (both inherited from `Object`). When you use a type as a `Dictionary` key, in a `HashSet`, or with LINQ's `Distinct`, the runtime relies on those two methods working together correctly.

The **equality contract** is a set of invariants you must honor when overriding them. Violations don't always crash immediately — they make collections quietly misbehave: items "disappear," lookups return wrong results, `Distinct` keeps duplicates.

This file is essential reading. Most "weird Dictionary behavior" bugs trace back here.

---

## The contract (the rules)

You must satisfy ALL of these:

### 1. Reflexivity
`a.Equals(a)` is always true.

### 2. Symmetry
If `a.Equals(b)`, then `b.Equals(a)`.

### 3. Transitivity
If `a.Equals(b)` AND `b.Equals(c)`, then `a.Equals(c)`.

### 4. Consistency
Repeated calls to `a.Equals(b)` return the same result, **provided neither object's state has changed**.

### 5. Hash code agreement
**If `a.Equals(b)`, then `a.GetHashCode() == b.GetHashCode()`.**

The reverse is **NOT** required — `a.GetHashCode() == b.GetHashCode()` does NOT imply `a.Equals(b)`. Hash collisions are expected and OK.

### 6. Hash code stability
`a.GetHashCode()` returns the **same value** every time on the same object (assuming state doesn't change). If you mutate state that affects hash, you must not have added the object to a hashed collection — or you'll lose it.

### 7. Null
`a.Equals(null)` is false (for non-null `a`).

---

## What goes wrong if you break it

### Break "Equals implies same hash"

```csharp
public class Bad {
    public int X;
    public override bool Equals(object? o) => o is Bad b && b.X == X;
    public override int GetHashCode() => DateTime.UtcNow.Millisecond;   // ⚠ changes!
}

var set = new HashSet<Bad>();
var a = new Bad { X = 1 };
set.Add(a);
set.Contains(a);   // probably false — different hash now, looks in wrong bucket
```

The runtime computes `a.GetHashCode()` once (when Adding) to pick a bucket. On `Contains`, it computes again — different value, looks elsewhere — can't find it.

### Break "consistency" via mutation

```csharp
public class MutableKey {
    public int X;
    public override int GetHashCode() => X.GetHashCode();
    public override bool Equals(object? o) => o is MutableKey k && k.X == X;
}

var dict = new Dictionary<MutableKey, string>();
var k = new MutableKey { X = 5 };
dict[k] = "five";

k.X = 99;                  // hash now reflects 99
dict.ContainsKey(k);        // false! Bucket lookup uses hash(99), entry is in hash(5) bucket
```

The entry still exists in the dictionary, but it's in the bucket for hash(5). The new lookup goes to bucket for hash(99). Can't find it.

**Cardinal rule for keys**: don't mutate state that affects equality after adding to a hashed collection. Easiest fix: make keys immutable.

### Break symmetry via inheritance

```csharp
public class A {
    public override bool Equals(object? o) => o is A;   // any A is equal to any A
}
public class B : A {
    public override bool Equals(object? o) => o is B && /* deeper check */;
}

A a = new A();
B b = new B();

a.Equals(b);   // true — b is an A
b.Equals(a);   // false — a is not a B
```

Symmetry broken. Hashed collections get confused.

**Fix**: don't allow A's Equals to accept any subtype. Use `GetType() == other.GetType()` or seal the class.

---

## The right way to override

For a **class** with value semantics:

```csharp
public sealed class Point : IEquatable<Point> {
    public Point(int x, int y) { X = x; Y = y; }
    public int X { get; }
    public int Y { get; }

    public bool Equals(Point? other) =>
        other is not null && X == other.X && Y == other.Y;

    public override bool Equals(object? obj) => Equals(obj as Point);

    public override int GetHashCode() => HashCode.Combine(X, Y);

    public static bool operator ==(Point? a, Point? b) =>
        a is null ? b is null : a.Equals(b);

    public static bool operator !=(Point? a, Point? b) => !(a == b);
}
```

Six things:
1. **Implement `IEquatable<T>`** — generic, no boxing. The version with the strongly-typed argument.
2. **Override `Equals(object?)`** — delegates to the typed version.
3. **Override `GetHashCode`** — use `HashCode.Combine` for good distribution.
4. **Implement `==` and `!=`** — for natural call-site syntax.
5. **`sealed`** — to avoid inheritance breaking symmetry.
6. **Make fields immutable** — readonly or init-only.

For a struct:

```csharp
public readonly struct Coord : IEquatable<Coord> {
    public Coord(int x, int y) { X = x; Y = y; }
    public int X { get; }
    public int Y { get; }

    public bool Equals(Coord other) => X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => obj is Coord c && Equals(c);
    public override int GetHashCode() => HashCode.Combine(X, Y);

    public static bool operator ==(Coord a, Coord b) => a.Equals(b);
    public static bool operator !=(Coord a, Coord b) => !a.Equals(b);
}
```

For structs, `IEquatable<T>` is especially important — without it, `Equals(object?)` boxes the argument.

For **records** — all of this is generated automatically:

```csharp
public sealed record Point(int X, int Y);

// You get: IEquatable<Point>, Equals(object?), GetHashCode, ==, !=, value equality.
```

For data containers with value semantics, **use records** — they correctly implement the contract for free.

---

## `HashCode.Combine`

Microsoft's recommended way to combine field hashes:

```csharp
public override int GetHashCode() => HashCode.Combine(X, Y, Name);
```

Up to 8 arguments. For more, use the `HashCode` builder:

```csharp
public override int GetHashCode() {
    var hash = new HashCode();
    hash.Add(X);
    hash.Add(Y);
    hash.Add(Name);
    hash.Add(Items.Count);
    return hash.ToHashCode();
}
```

`HashCode.Combine` uses **xxHash** internally with a per-process random seed, which:
- Distributes well.
- Mitigates hash-flood DoS attacks (attackers can't precompute collisions).
- Is fast.

**Don't** roll your own:

```csharp
public override int GetHashCode() => X ^ Y ^ Z;   // ⚠ poor distribution

public override int GetHashCode() {
    unchecked {
        int hash = 17;
        hash = hash * 23 + X;
        hash = hash * 23 + Y;
        return hash;
    }
}   // OK but inferior to HashCode.Combine
```

For most code, `HashCode.Combine` is the right tool.

---

## `IEqualityComparer<T>` — when you can't change the type

Sometimes you can't (or don't want to) modify the type. Pass a custom comparer:

```csharp
public class CaseInsensitive : IEqualityComparer<string> {
    public bool Equals(string? a, string? b) =>
        string.Equals(a, b, StringComparison.OrdinalIgnoreCase);
    public int GetHashCode(string s) =>
        StringComparer.OrdinalIgnoreCase.GetHashCode(s);
}

var dict = new Dictionary<string, int>(new CaseInsensitive());
dict["Hello"] = 1;
dict["HELLO"];   // 1
```

In practice, use `StringComparer.OrdinalIgnoreCase` directly — same thing, no custom class needed.

For dictionary keys that need composite equality:

```csharp
public class PointComparer : IEqualityComparer<Point> {
    public bool Equals(Point? a, Point? b) =>
        a is not null && b is not null && a.X == b.X && a.Y == b.Y;
    public int GetHashCode(Point p) => HashCode.Combine(p.X, p.Y);
}

var dict = new Dictionary<Point, string>(new PointComparer());
```

---

## Records and equality

Records auto-generate value equality:

```csharp
public record Person(string Name, int Age);

var a = new Person("Alice", 30);
var b = new Person("Alice", 30);

a == b;                  // true
a.Equals(b);             // true
a.GetHashCode() == b.GetHashCode();   // true
```

What's synthesized:
- `IEquatable<T>.Equals(T?)`.
- Override of `Equals(object?)`.
- Override of `GetHashCode` — uses `HashCode.Combine` over all properties.
- `==` and `!=` operators.
- A protected `EqualityContract` property — used to ensure `Person` and `Employee : Person` aren't equal even if their data matches.

For **structural** types, records are the no-effort path.

### Records with reference-type properties

```csharp
public record Cart(int UserId, List<string> Items);

var a = new Cart(1, new List<string> { "x" });
var b = new Cart(1, new List<string> { "x" });

a == b;   // false! Lists compare by reference, not contents
```

The synthesized equality uses `EqualityComparer<T>.Default` per property — for `List<T>`, that's reference equality. Records aren't magic for nested mutable collections.

**Fix**:
- Use immutable collections (`ImmutableArray<T>`, `ImmutableList<T>`) — they don't have value equality either by default, but at least they're stable.
- Manually override `Equals` for the record to use `SequenceEqual` on the list.

For deep value equality on complex types, you sometimes need to compose carefully.

---

## Common bugs

### Override Equals, forget GetHashCode

```csharp
public class Bad {
    public int X;
    public override bool Equals(object? o) => o is Bad b && b.X == X;
    // Forgot GetHashCode!
}
```

Default `GetHashCode` returns something based on identity — equal objects have different hash codes → dictionary fails.

Compiler warning **CS0659** fires for this exact case. Heed it.

### Hash code uses too few fields

```csharp
public class User {
    public string FirstName, LastName;
    public override bool Equals(object? o) => o is User u && u.FirstName == FirstName && u.LastName == LastName;
    public override int GetHashCode() => FirstName.GetHashCode();   // ignores LastName
}
```

Valid (hash isn't unique), but performance suffers — all (FirstName, *) hash to the same bucket. Use `HashCode.Combine(FirstName, LastName)`.

### Override Equals on mutable class without immutability discipline

```csharp
class Person { public string Name; public override bool Equals(object? o) => ...; public override int GetHashCode() => Name.GetHashCode(); }

var p = new Person { Name = "Alice" };
var set = new HashSet<Person>();
set.Add(p);
p.Name = "Bob";
set.Contains(p);   // false — bucket lookup uses new hash
```

If a type is going to be a Dictionary key or HashSet element, the fields used in equality must not change. Period.

### Asymmetric Equals across types

```csharp
public class Money {
    public int Amount;
    public override bool Equals(object? o) => o switch {
        Money m => Amount == m.Amount,
        int i => Amount == i,           // ⚠ — Money equals int?
        _ => false
    };
    public override int GetHashCode() => Amount;
}

new Money { Amount = 5 }.Equals(5);   // true
5.Equals(new Money { Amount = 5 });    // false — int doesn't know about Money
```

Symmetry broken. Don't cross types in Equals.

---

## When to override

- The type represents a **value** (Point, Money, Date, Coord).
- You'll use it as a Dictionary key or HashSet element.
- You want LINQ's Distinct / GroupBy / Join to dedupe correctly.

When NOT to override:
- The type represents an **entity with identity** (Customer, Order — two Customers with the same name aren't the same customer).
- The default reference equality is what you want.

For value-like classes, **prefer records** — auto-correct equality contract, less code.

---

## Performance considerations

- `GetHashCode` is called **per access** in hashed collections. Make it cheap. Avoid string concatenation, allocations, etc.
- `Equals` runs after a hash match. Compare the fastest-rejecting fields first (small ints, then strings).
- `IEquatable<T>` matters more for structs — avoids boxing.
- `EqualityComparer<T>.Default` chooses the best implementation: `IEquatable<T>` if implemented, else `Object.Equals` (which boxes for structs).

---

## Cheat sheet

```csharp
// VALUE-LIKE TYPE (Point, Money, etc.)
public sealed record Point(int X, int Y);   // ← prefer this

// OR if you can't use a record:
public sealed class Point : IEquatable<Point> {
    public int X { get; }
    public int Y { get; }
    public Point(int x, int y) { X = x; Y = y; }
    public bool Equals(Point? other) => other is not null && X == other.X && Y == other.Y;
    public override bool Equals(object? obj) => Equals(obj as Point);
    public override int GetHashCode() => HashCode.Combine(X, Y);
    public static bool operator ==(Point? a, Point? b) => a is null ? b is null : a.Equals(b);
    public static bool operator !=(Point? a, Point? b) => !(a == b);
}

// STRUCT WITH VALUE SEMANTICS
public readonly record struct Coord(int X, int Y);

// ENTITY (identity-based)
// Don't override Equals/GetHashCode. Default reference equality is correct.
```

---

## One last test

You override `Equals` and `GetHashCode`. You add an object to a HashSet. You mutate a field used in equality. What happens?

- The object is still in the HashSet (the underlying bucket still has it).
- `set.Contains(obj)` may return false (looking in the wrong bucket).
- `foreach (var x in set)` will still yield it.
- `set.Remove(obj)` may fail to remove it (same wrong-bucket issue).

The object is **orphaned**: present but unreachable through normal Contains/Remove. The fix is "don't do that" — keep equality-relevant fields immutable.

→ Next: [12-IEnumerableHierarchy.md](12-IEnumerableHierarchy.md)
