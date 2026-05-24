# Pattern Matching

## What it is

Pattern matching lets you test a value's **shape** — its type, its properties, its position in a tuple, its elements in a list — and extract pieces in a single expression. It's been C#'s most-improved feature since C# 7 (2017), and it's how modern C# code does discrimination across heterogeneous data.

```csharp
string Describe(object o) => o switch {
    null => "nothing",
    int n when n < 0 => $"negative int {n}",
    int n => $"non-negative int {n}",
    string { Length: 0 } => "empty string",
    string s => $"string '{s}'",
    int[] [1, 2, ..] => "int array starting with 1, 2",
    Point(0, 0) => "origin",
    _ => $"some {o.GetType().Name}"
};
```

That single switch covers null, type tests, value ranges, property patterns, list patterns, and positional patterns — and the compiler verifies completeness.

This file is a tour. The deeper version is in [Chapter 10 §02](../10-AdvancedLanguage/02-PatternMatchingDeep.md).

---

## Why it exists

Pre-pattern-matching code did this:

```csharp
if (obj is string) {
    string s = (string)obj;
    Console.WriteLine(s.Length);
}
```

The cast and the test were separate; you might check one type and cast to another by mistake. Pattern matching collapses test + cast + bind into one:

```csharp
if (obj is string s) {
    Console.WriteLine(s.Length);
}
```

Beyond that, switch expressions + patterns turn long if/else chains into compact, readable, exhaustively-checked code. This shifts a class of code from procedural to declarative.

---

## Where you use patterns

Patterns appear in:
- **`is` expressions** — `obj is Pattern`.
- **`switch` statements** — `case Pattern: ...`.
- **`switch` expressions** — `value switch { Pattern => result, ... }`.
- **`is not` / `is and` / `is or`** — combined patterns.

A pattern always evaluates against a target value and produces:
- A boolean (does it match?).
- Optionally, bindings for parts of the value.

---

## The patterns, one by one

### 1. Constant pattern

Match a literal:

```csharp
if (n is 0) Console.WriteLine("zero");
return s switch {
    "yes" => true,
    "no" => false,
    _ => throw new()
};
```

Works for any compile-time constant: numbers, strings, `null`, enums, named consts.

### 2. Type pattern

Test the runtime type, optionally bind a variable:

```csharp
if (obj is string) { /* matches but no binding */ }
if (obj is string s) { Console.WriteLine(s.Length); }  // binds s
if (obj is Person { Age: > 18 } adult) { /* type + property + binding */ }
```

The bound variable's type is the pattern type (`string`, `Person`).

### 3. Var pattern

Always matches; binds the variable. Useful when you want a name for use in a `when` clause:

```csharp
if (computed is var x && x.IsValid) { /* x is in scope */ }

return o switch {
    var x when ExpensiveCheck(x) => "passed check",
    _ => "fail"
};
```

The bound variable's type is the **static type** of the expression — could be `object`.

### 4. Discard pattern `_`

Matches anything, no binding. Used as a fallback or "I don't care":

```csharp
return n switch {
    1 => "one",
    2 => "two",
    _ => "other"
};

if (point is (_, 0)) { /* y is 0, x is anything */ }
```

### 5. Property pattern

Match on the properties of an object:

```csharp
if (person is { Age: > 18 }) { ... }
if (person is { Age: >= 18, Country: "US" }) { ... }
if (person is { Name.Length: > 0 }) { ... }   // nested!

return user switch {
    { IsAdmin: true } => "admin",
    { Email: { Length: 0 } } => "no email",   // nested property
    { Email.Length: 0 } => "no email",         // shorter form (C# 10+)
    _ => "regular"
};
```

You can also combine type + property:

```csharp
if (obj is Person { Age: > 18 } adult) { /* adult is a Person */ }
```

This is one of the most useful patterns — it's structural matching at its purest.

### 6. Positional pattern

For types that support deconstruction (tuples, records, types with `Deconstruct`):

```csharp
public record Point(int X, int Y);

return p switch {
    (0, 0) => "origin",
    (_, 0) => "x-axis",
    (0, _) => "y-axis",
    (var x, var y) when x == y => "diagonal",
    _ => "somewhere"
};
```

For tuples, the syntax is exactly the same:

```csharp
return (x, y) switch {
    (0, 0) => "origin",
    (> 0, > 0) => "Q1",
    // ...
};
```

### 7. Relational pattern (C# 9+)

```csharp
return n switch {
    < 0 => "negative",
    0 => "zero",
    > 0 and < 10 => "small positive",
    >= 10 and <= 100 => "medium",
    > 100 => "large"
};
```

Supports `<`, `<=`, `>`, `>=`. Works on any value with comparison operators (numbers, chars, enums).

### 8. Logical patterns: `and`, `or`, `not` (C# 9+)

Combine patterns:

```csharp
if (n is > 0 and < 100) { ... }          // both conditions
if (s is "yes" or "y") { ... }            // either
if (s is not null) { ... }                // negation

return c switch {
    >= 'a' and <= 'z' => "lowercase",
    >= 'A' and <= 'Z' => "uppercase",
    >= '0' and <= '9' => "digit",
    _ => "other"
};
```

Precedence: `not` binds tightest, then `and`, then `or`.

### 9. List pattern (C# 11+)

Match the elements of a list-like value (array, list, span, any type implementing the right interfaces):

```csharp
int[] arr = { 1, 2, 3 };

if (arr is [1, 2, 3]) { ... }                  // exactly these
if (arr is [1, _, 3]) { ... }                  // 1, anything, 3
if (arr is [1, .., 3]) { ... }                 // starts 1, ends 3, any number in middle
if (arr is [1, .. var rest, 3]) { ... }        // capture the middle
if (arr is [_, _, _]) { ... }                   // any 3 elements
if (arr is []) { ... }                         // empty
if (arr is [var first, ..]) { ... }            // at least one, capture first
```

The `..` (slice pattern) matches zero or more elements. At most one `..` per pattern.

Works on arrays, `List<T>`, `Span<T>`, `string` (char by char), and any type with `Length`/`Count` + `int`/`Index` indexer.

### 10. Combined and recursive patterns

Patterns nest freely:

```csharp
return shape switch {
    Circle { Radius: > 0 } c => $"circle r={c.Radius}",
    Square { Side: > 0 and < 10 } => "small square",
    Polygon { Sides: var n } when n >= 3 => $"{n}-sided polygon",
    null => "no shape",
    _ => "unknown"
};
```

You can read it: "shape is a Circle with positive Radius, bind to c."

---

## `when` clauses

A `when` adds an arbitrary boolean test alongside a pattern:

```csharp
return p switch {
    { Age: var a } when a < 0 => "invalid",
    { Age: var a } when a < 18 => "minor",
    { Age: var a } when a < 65 => "adult",
    _ => "senior"
};
```

Inside `when` you have access to any variables the pattern bound.

The compiler can't reason about `when` clauses — they're treated as "this case is conditional." Use sparingly; when possible, encode logic into the pattern itself.

---

## Switch expression vs switch statement

A **switch expression** returns a value:

```csharp
string s = n switch {
    1 => "one",
    2 => "two",
    _ => "other"
};
```

A **switch statement** runs statements per case:

```csharp
switch (n) {
    case 1:
        DoOne();
        break;
    case 2:
        DoTwo();
        break;
    default:
        DoOther();
        break;
}
```

Both support patterns. Differences:

| | Switch expression | Switch statement |
|---|---|---|
| Returns a value | ✓ | ✗ |
| `break` / `return` per case | ✗ (use `,`) | ✓ |
| Exhaustiveness checking | ✓ (warns if non-exhaustive) | ✗ |
| Multiple statements per arm | ✗ (one expression each) | ✓ |
| Fallthrough | ✗ | Only via `goto case` |

**Prefer the switch expression** when you're picking a value. Use the switch statement when you need to execute multiple statements per case.

---

## Exhaustiveness

The switch expression's checker is **conservative**:

- For enums: warns if a defined value isn't covered.
- For sealed type hierarchies: warns about uncovered cases (limited).
- For ranges: warns if not all numeric values are matched.

When in doubt, add a `_ => throw new()` arm. It documents your intent and crashes if you missed a case.

```csharp
return severity switch {
    Severity.Info => "blue",
    Severity.Warning => "yellow",
    Severity.Error => "red",
    Severity.Critical => "darkred",
    _ => throw new InvalidOperationException($"Unknown severity: {severity}")
};
```

---

## Pattern matching meets DI / domain modeling

A common modern pattern: **discriminated unions** via a sealed class/record hierarchy.

```csharp
public abstract record PaymentResult;
public sealed record PaymentSuccess(string TransactionId) : PaymentResult;
public sealed record PaymentRejected(string Reason) : PaymentResult;
public sealed record PaymentRequiresMoreInfo(string PromptUrl) : PaymentResult;

string Display(PaymentResult r) => r switch {
    PaymentSuccess { TransactionId: var id } => $"Paid! Ref: {id}",
    PaymentRejected { Reason: var why } => $"Rejected: {why}",
    PaymentRequiresMoreInfo { PromptUrl: var url } => $"Go to {url}",
    _ => throw new()    // (the only case if a new subtype is added — surfaces gaps)
};
```

This is **discriminated unions in C# clothing**. F# and Rust have first-class syntax; C# composes it from records + pattern matching. It works well, especially in CQRS, event sourcing, and result types.

C# 14 doesn't yet have a `union` keyword (proposed for C# 15+); pattern matching is the current workaround.

---

## Internals — how patterns compile

Patterns compile down to a tree of `if` / `switch` / type tests / property reads. The compiler optimizes:

- **Constant string switches** → hash table (very fast).
- **Constant int switches** → jump table when dense.
- **Type patterns on sealed hierarchies** → MT comparisons (fast).
- **Property patterns** → calls to the property getters in order.
- **List patterns** → length check + indexed access.

For:
```csharp
return o switch {
    int n => $"int {n}",
    string s => $"string {s}",
    _ => "other"
};
```

The IL roughly compiles to:
```il
ldloc.0
isinst System.Int32
brfalse .notInt
... handle int case ...

.notInt:
ldloc.0
isinst System.String
brfalse .notString
... handle string case ...

.notString:
... default case ...
```

Each `isinst` is a fast type check. The cumulative cost is roughly the same as a hand-written if/else chain. For very tall switches with many type tests, the compiler may emit a hash-based dispatch — usually transparently.

For property patterns like `{ Age: > 18 }`:
1. Cast / type-check.
2. Call `get_Age()`.
3. Compare.

Each property access is one method call (often inlined by the JIT). So `{ Age: > 18, Name.Length: > 0 }` is one cast + two getter calls + two comparisons.

### List patterns and `Length` / `Count`

List patterns require the type to have:
- A `Length` or `Count` property (compile-time choice).
- An `int` or `System.Index` indexer.

For arrays and `string`, this is direct. For `List<T>` and `Span<T>`, the compiler emits `Count`/`Length` checks before iterating.

The slice pattern `..` uses `System.Range`:
```csharp
arr is [1, .. var rest, 3]
```
becomes roughly: `arr.Length >= 2 && arr[0] == 1 && arr[^1] == 3 && (rest = arr[1..^1])`.

---

## Common patterns in practice

### Validating input

```csharp
return input switch {
    null => Error("input cannot be null"),
    "" => Error("input cannot be empty"),
    { Length: > 100 } => Error("input too long"),
    var s when !IsValidFormat(s) => Error("bad format"),
    _ => Ok(input)
};
```

### Handling nullable structs

```csharp
int? n = ParseOrNull(input);
return n switch {
    null => "no value",
    < 0 => "negative",
    0 => "zero",
    > 0 => "positive"
};
```

### Discriminated union dispatch

(See above — sealed record hierarchies + switch.)

### Polymorphism alternative

```csharp
double Area(Shape s) => s switch {
    Circle c => Math.PI * c.Radius * c.Radius,
    Square q => q.Side * q.Side,
    Rectangle r => r.Width * r.Height,
    _ => throw new InvalidOperationException()
};
```

When you don't want to put `Area` on the type, pattern matching gives you a single function.

### Combining structural + relational patterns

```csharp
return order switch {
    { Total: > 1000, Items.Count: > 5 } big => "process VIP",
    { Status: OrderStatus.Cancelled } => "skip",
    { CreatedAt: var c } when c < DateTime.UtcNow.AddDays(-30) => "archive",
    _ => "regular processing"
};
```

---

## Common bugs

- **Order matters** — switch arms are evaluated top to bottom. Put more specific patterns first. `{ Age: > 0 }` before `{ }`.
- **Forgetting exhaustive default** — for non-enum / non-sealed input, always include `_ =>`. Even if you "know" every case.
- **`null` pattern** vs **null reference type** — `is null` always works; `is { }` matches non-null (and any properties).
- **`when` clauses with side effects** — they may run multiple times during pattern matching evaluation in some cases. Keep them pure.
- **List pattern slice with multiple `..`** — only one slice allowed per pattern.
- **Discard `_` vs variable `_` named `_`** — in patterns, `_` is always discard.

---

## Performance notes

- Pattern matching compiles to direct IL. No reflection.
- Simple patterns (const, type) are as fast as hand-written.
- Big switch expressions on strings use perfect-hash dispatch — O(1).
- List patterns on `Span<T>` and arrays are highly optimized.
- The `when` clause is a normal boolean — pay for what's inside it.

---

## When to use what

| Goal | Use |
|---|---|
| Test a value against another | `==` or constant pattern |
| Test the type | `is T` (with optional binding) |
| Type + properties | `is T { Prop: pat }` |
| Multi-way value pick | switch expression |
| Side-effectful branching | switch statement |
| Domain dispatch on a closed type set | sealed records + switch expression |
| Range comparison | relational patterns |
| Combining | `and` / `or` / `not` |
| Iterate-and-match elements | list patterns |

→ Next: [10-AnonymousTypes.md](10-AnonymousTypes.md)
