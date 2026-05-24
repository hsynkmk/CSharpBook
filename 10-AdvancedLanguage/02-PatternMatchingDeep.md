# Pattern Matching — Deep Dive

> Chapter 03 §09 introduced the patterns. This file goes deeper: combining patterns, recursive patterns, list patterns, and how the compiler optimizes switch expressions.

---

## The full pattern grammar

C# pattern grammar (informal):

```
pattern := constant_pattern
         | type_pattern
         | declaration_pattern
         | var_pattern
         | discard_pattern
         | property_pattern
         | positional_pattern
         | relational_pattern
         | list_pattern
         | slice_pattern
         | logical_pattern (and / or / not)
         | parenthesized_pattern
```

Some compose recursively — e.g., a property pattern can contain other patterns inside.

---

## Constant pattern

Match literal values:

```csharp
return s switch {
    "yes" => true,
    "no" => false,
    null => false,
    _ => throw new()
};
```

Works for any compile-time constant: literals, named constants, enum members.

---

## Type pattern + declaration

Test runtime type, optionally bind a variable:

```csharp
if (obj is string) { /* matches; no binding */ }
if (obj is string s) { /* matches; s is a non-null string */ }
```

The bound variable's type is the pattern type. For nullable annotations:

```csharp
object? maybe = ...;
if (maybe is string s) {
    // s is string (non-null) — the pattern excludes null
}
```

Type patterns DON'T match null (you have to add a separate null check or pattern).

---

## Property pattern

Match by inspecting properties:

```csharp
return person switch {
    { Age: 0 } => "newborn",
    { Age: < 18 } => "minor",
    { Country: "US", Age: >= 21 } => "US adult",
    { Name: { Length: > 0 } } => "named person",
    _ => "other"
};
```

Inside `{ }`, list property: pattern pairs. Each property is matched against its inner pattern. Multiple properties = AND.

### Nested property patterns

```csharp
{ Address: { City: "Springfield" } }   // nested
{ Address.City: "Springfield" }         // shorthand (C# 10+)
```

The dot syntax is a compact form of nested property pattern.

### Combined with type and binding

```csharp
if (obj is Person { Age: > 18 } adult) {
    // adult is a Person AND age > 18
    Console.WriteLine(adult.Name);
}
```

Type pattern + property pattern + binding all in one.

---

## Positional pattern (Deconstruct)

For types with a `Deconstruct` method (or tuples / records):

```csharp
public record Point(int X, int Y);

return p switch {
    (0, 0) => "origin",
    (_, 0) => "x-axis",
    (var x, var y) when x == y => "diagonal",
    _ => "off-axis"
};
```

Each position is a pattern. Tuples and records are deconstructable; you can add `Deconstruct` to any class.

Mix with property pattern:

```csharp
p is Point(0, var y) { Y: > 0 }
```

Position 0 = X is 0; position 1 = Y bound; property pattern says Y > 0.

---

## Relational pattern (C# 9+)

```csharp
return n switch {
    < 0 => "negative",
    0 => "zero",
    > 0 and < 100 => "small positive",
    >= 100 => "big"
};
```

`<`, `<=`, `>`, `>=` against constants. Works on numbers, chars, enums.

---

## Logical patterns: `and`, `or`, `not`

Combine patterns:

```csharp
if (n is > 0 and < 100) { ... }    // both
if (s is "yes" or "y") { ... }      // either
if (s is not null) { ... }          // negation

return c switch {
    >= 'a' and <= 'z' => "lower",
    >= 'A' and <= 'Z' => "upper",
    >= '0' and <= '9' => "digit",
    _ => "other"
};
```

Precedence (tightest first): `not`, `and`, `or`. Use parentheses to be explicit.

---

## List pattern (C# 11+)

Match elements of array-like types:

```csharp
int[] arr = { 1, 2, 3 };

if (arr is [1, 2, 3]) { /* exactly these */ }
if (arr is [1, _, 3]) { /* 1, anything, 3 */ }
if (arr is [1, ..]) { /* starts with 1 */ }
if (arr is [.., 3]) { /* ends with 3 */ }
if (arr is [1, .. var middle, 3]) { /* middle is the middle slice */ }
if (arr is [_, _, _]) { /* exactly 3 elements */ }
if (arr is []) { /* empty */ }
```

`..` is the **slice pattern** — matches zero or more elements. At most one `..` per list pattern.

Works on:
- Arrays.
- `List<T>` and `IList<T>`.
- `Span<T>` / `ReadOnlySpan<T>`.
- `string` (char-by-char).
- Any type with a `Length`/`Count` property + `int`/`Index` indexer.

### Slice pattern combined with other patterns

```csharp
arr is [_, .. [< 0, _, ..], _]   // 2nd element is negative, in a slice
arr is [var first, .., var last]  // first and last bound; middle ignored
```

You can nest list patterns recursively. Powerful but can get unreadable — keep them simple.

---

## Var pattern + discard

```csharp
return obj switch {
    var x when ExpensiveCheck(x) => "passed",
    _ => "failed"
};
```

`var x` always matches; binds the value as `x`. Useful when you want to introduce a name to use in the `when` clause.

`_` always matches; binds nothing. The catch-all in switch.

---

## Pattern matching in `is` vs `switch`

```csharp
// In is-expression — returns bool, binds variable
if (obj is Person { Age: > 18 } adult) { ... }

// In switch expression — returns a value
return obj switch {
    Person { Age: > 18 } adult => adult.Name,
    _ => "unknown"
};

// In switch statement — runs statements
switch (obj) {
    case Person { Age: > 18 } adult:
        Console.WriteLine(adult.Name);
        break;
}
```

Same pattern grammar in all three contexts.

---

## `when` clause — guards

When a pattern's structure doesn't capture everything you need:

```csharp
return p switch {
    Person { Age: var a } when a < 0 => "invalid",
    Person { Age: < 18 } => "minor",
    _ => "other"
};
```

`when` is an arbitrary boolean expression. Runs after the pattern matches. If false, the arm doesn't match — the switch tries the next one.

Use sparingly. When possible, encode in the pattern itself for the compiler's exhaustiveness analysis.

---

## Exhaustiveness analysis

For switch expressions, the compiler tries to verify all cases are covered:

```csharp
public enum Severity { Info, Warning, Error, Critical }

ConsoleColor Color(Severity s) => s switch {
    Severity.Info => ConsoleColor.White,
    Severity.Warning => ConsoleColor.Yellow,
    Severity.Error => ConsoleColor.Red,
    // ⚠ — CS8509: not all cases covered
};
```

Adding the missing case (or `_ =>`) suppresses the warning.

For non-enum types: usually warns "default case missing." Best practice: always include `_ => throw ...` to handle unexpected values.

For sealed type hierarchies (discriminated unions), the compiler has improving but not yet exhaustive coverage analysis. Always include the catch-all.

---

## Discriminated union pattern

```csharp
public abstract record Shape;
public sealed record Circle(double Radius) : Shape;
public sealed record Square(double Side) : Shape;
public sealed record Rectangle(double W, double H) : Shape;

double Area(Shape s) => s switch {
    Circle c => Math.PI * c.Radius * c.Radius,
    Square q => q.Side * q.Side,
    Rectangle r => r.W * r.H,
    _ => throw new InvalidOperationException("unknown shape")
};
```

C#'s pragmatic discriminated union: sealed record hierarchy + switch expression with type patterns.

Adding a new Shape subclass: the existing switch doesn't break (the `_ => throw` catches it), but you should add a new arm. There's no compile-time enforcement (yet) — that's the main weakness vs F# or Rust.

---

## Cool patterns

### Match on tuple

```csharp
return (x, y) switch {
    (0, 0) => "origin",
    (> 0, > 0) => "Q1",
    (< 0, > 0) => "Q2",
    (< 0, < 0) => "Q3",
    (> 0, < 0) => "Q4",
    (0, _) => "y-axis",
    (_, 0) => "x-axis",
};
```

Tuple in a switch's value — patterns on each position.

### Empty / single / multiple list

```csharp
return items switch {
    [] => "empty",
    [var only] => $"one: {only}",
    [var first, _] => $"two starting {first}",
    [var first, .., var last] => $"many: {first}..{last}"
};
```

List patterns let you discriminate by length.

### Nested record with property pattern

```csharp
public record Order(int Id, Customer Customer, decimal Total);

return order switch {
    { Customer: { Country: "US" }, Total: > 1000m } => "ship priority US",
    { Customer.Country: "US", Total: > 1000m } => "shorter form, same thing",
    { Total: < 10m } => "skip",
    _ => "regular"
};
```

Deep destructuring through nested properties.

### Sequence start/end check

```csharp
return chars switch {
    ['<', .., '>'] => "HTML-like",
    ['{', .., '}'] => "JSON-like",
    ['"', .., '"'] => "string-like",
    _ => "other"
};
```

Quick parsing of recognizable patterns.

---

## Internals — what the compiler generates

For a switch expression:

```csharp
return shape switch {
    Circle c => c.Radius,
    Square q => q.Side,
    _ => 0
};
```

The compiler emits a series of type tests:

```il
isinst Circle
brfalse .notCircle
... use the casted Circle ...

.notCircle:
isinst Square
brfalse .notSquare
... use the casted Square ...

.notSquare:
... default ...
```

For **constant string switches**, the compiler can use a perfect-hash table for O(1) dispatch.

For **dense integer switches**, a jump table.

For **type switches**, sequential isinst checks (or for sealed hierarchies, sometimes optimized).

For **property patterns**, getter calls in sequence. Short-circuits at first mismatch.

The generated code is competitive with hand-written if/else.

---

## Common patterns in real code

### Pretty type printer

```csharp
string Describe(object? o) => o switch {
    null => "null",
    int i => $"int {i}",
    string { Length: 0 } => "empty string",
    string s => $"string '{s}'",
    int[] [] => "empty array",
    int[] [var x] => $"single int [{x}]",
    int[] [.., var last] => $"int array ending {last}",
    Exception ex => $"exception: {ex.Message}",
    var x => $"some {x.GetType().Name}"
};
```

### State machine transition

```csharp
public OrderStatus Next(OrderStatus current, OrderEvent e) =>
    (current, e) switch {
        (OrderStatus.Created, OrderEvent.Pay) => OrderStatus.Paid,
        (OrderStatus.Paid, OrderEvent.Ship) => OrderStatus.Shipped,
        (OrderStatus.Shipped, OrderEvent.Deliver) => OrderStatus.Delivered,
        (_, OrderEvent.Cancel) when current != OrderStatus.Delivered => OrderStatus.Cancelled,
        _ => throw new InvalidOperationException("illegal transition")
    };
```

Tuple of (current, event), match the legal transitions.

### Parsing input

```csharp
public Result Parse(string input) => input switch {
    null or "" => Result.Empty,
    ['#', .. var hex] when IsHex(hex) => Result.Color(hex),
    ['/', .. var path] => Result.Path(path),
    var s when int.TryParse(s, out var n) => Result.Number(n),
    _ => Result.Invalid
};
```

Combine list patterns + `when` + property access.

---

## Common bugs

### Wrong order of arms

```csharp
return n switch {
    > 0 => "positive",
    > 10 => "big positive",   // unreachable — > 0 always matches first
    _ => "non-positive"
};
```

Top to bottom, first match wins. Put more specific patterns first.

### Missing null in type pattern

```csharp
public void M(object? obj) {
    var len = obj switch {
        string s => s.Length,
        // ⚠ missing null case if obj could be null
        _ => 0
    };
}
```

Type patterns don't match null. Add `null => 0` if obj could be null.

### `Single` vs `var`

```csharp
return n switch {
    var x => $"got {x}",   // matches anything — equivalent to `_`
    1 => "one"             // unreachable
};
```

`var x` matches everything. Subsequent arms are unreachable.

### `when` with side effects

```csharp
return n switch {
    var x when LogAndCheck(x) => "matches",
    _ => "nope"
};
```

If the side-effecting predicate runs multiple times (e.g., across pattern fallthrough), surprises happen. Keep `when` pure.

### Misunderstanding `or` precedence

```csharp
return n switch {
    < 0 or 0 and > -10 => "...",  // ⚠ — and binds tighter than or
    // parses as: (< 0) or (0 and (> -10))
    _ => "..."
};
```

Always parenthesize for clarity: `(< 0 or 0) and (> -10)`.

---

## Performance

- Constant patterns on strings: O(1) via perfect hash for ≥ 7-ish entries; linear else.
- Constant patterns on ints: jump table for dense; binary search for sparse.
- Type patterns: O(N) sequential `isinst` checks. For sealed hierarchies, may be optimized.
- Property patterns: one getter call per property + comparison.
- List patterns: length check + per-element comparison.

Modern .NET makes pattern matching as fast as hand-written code. Don't avoid patterns for performance reasons — use them for clarity.

---

## When to prefer patterns

✓ Multi-branch dispatch on type or shape.
✓ Discriminated unions via sealed records.
✓ State machine transitions.
✓ Parsing / classification.

✗ Single-branch checks — `if (x is Person p)` is fine, but switch is overkill for 1-2 cases.
✗ When clarity suffers — overly clever patterns are hard to read.

---

## Summary

- Pattern matching is C#'s structural-dispatch system.
- Patterns: constant, type, declaration, var, discard, property, positional, relational, logical, list, slice.
- Combine in switch expressions for compact multi-case logic.
- Compiler verifies exhaustiveness when it can.
- Encode discriminated unions as sealed record hierarchies + switch.
- Performance is competitive with hand-written code.

→ Next: [03-RecordsDeep.md](03-RecordsDeep.md)
