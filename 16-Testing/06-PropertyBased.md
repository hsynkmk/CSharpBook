# Property-Based Testing

## What it is

Example-based tests check specific inputs you thought of (`Add(2, 3) == 5`). **Property-based testing** checks that a *property* holds for **hundreds of randomly generated inputs**, automatically exploring edge cases you didn't think of — and when it finds a failure, it **shrinks** the input to the smallest counterexample.

```csharp
// Example-based — checks the cases you imagined
Assert.Equal(5, Add(2, 3));

// Property-based — checks a law for many random inputs
// Property: addition is commutative for all integers
Prop.ForAll((int a, int b) => Add(a, b) == Add(b, a));
```

The shift in mindset: from "does this specific case work?" to "what *invariants* must always hold?"

---

## Why it finds bugs example tests miss

You write tests for the cases you anticipate. Bugs hide in the cases you *didn't* — empty input, `int.MaxValue`, negative zero, Unicode edge cases, duplicate elements, off-by-one boundaries. A property-based runner generates thousands of these automatically.

```csharp
// Property: reversing a list twice yields the original
// The runner tries: [], [0], [int.MinValue], [1,1,1], huge lists, ...
Prop.ForAll((List<int> xs) => Reverse(Reverse(xs)).SequenceEqual(xs));
```

If `Reverse` has a bug on single-element or empty lists, the runner finds it — and shrinks the failing input to, say, `[]` or `[0]`, the minimal reproducer.

---

## FsCheck (the .NET library)

FsCheck (usable from C#) generates inputs and checks properties.

```csharp
using FsCheck;
using FsCheck.Fluent;
using FsCheck.Xunit;

public class MathProperties {
    [Property]
    public bool Addition_IsCommutative(int a, int b) =>
        Calculator.Add(a, b) == Calculator.Add(b, a);

    [Property]
    public bool Reverse_Twice_IsIdentity(int[] xs) =>
        xs.Reverse().Reverse().SequenceEqual(xs);

    [Property]
    public Property Sort_IsIdempotent(int[] xs) {
        var once = xs.OrderBy(x => x).ToArray();
        var twice = once.OrderBy(x => x).ToArray();
        return (once.SequenceEqual(twice)).ToProperty();
    }
}
```

`[Property]` (from `FsCheck.Xunit`) runs the method ~100 times with generated arguments. Return `bool` (or `Property`) — `true`/holds = pass.

### Conditional properties

```csharp
[Property]
public Property Division_Inverts_Multiplication(int a, int b) =>
    (b != 0).Implies(() => (a * b) / b == a);   // only test when b != 0
```

`Implies` filters out inputs where the property doesn't apply.

### Custom generators

When the default random data isn't right (e.g., only positive numbers, valid emails):

```csharp
public static Arbitrary<int> PositiveInts() =>
    Arb.From<int>().Filter(x => x > 0);

[Property(Arbitrary = [typeof(MyGenerators)])]
public bool Sqrt_OfSquare(int n) => /* ... uses positive ints ... */;
```

---

## Shrinking — the killer feature

When a property fails, naive random testing reports a huge ugly input. FsCheck **shrinks** it: repeatedly simplifies the failing input while it still fails, reporting the minimal counterexample.

```
Falsifiable, after 37 tests (5 shrinks):
Original: [4, -2387, 0, 99, -1, 1000000]
Shrunk:   [0, -1]            ← minimal input that still breaks the property
```

A 6-element failure shrunk to `[0, -1]` immediately points you at the bug (something about a zero followed by a negative). Shrinking turns "it failed on random garbage" into "it failed on this tiny obvious case."

---

## Good properties to look for

Common property patterns ("the property zoo"):

| Pattern | Example |
|---|---|
| **Inverse / round-trip** | `Deserialize(Serialize(x)) == x` |
| **Idempotence** | `Sort(Sort(x)) == Sort(x)` |
| **Commutativity** | `Add(a,b) == Add(b,a)` |
| **Associativity** | `(a+b)+c == a+(b+c)` |
| **Invariants** | `Sort(x).Length == x.Length` and is ordered |
| **Oracle** (compare to known-good) | `MyFastSort(x) == x.OrderBy(i=>i)` |
| **Metamorphic** | `Sum(x ++ y) == Sum(x) + Sum(y)` |

Round-trip (serialize/deserialize, encode/decode, parse/format) is the most broadly useful — if `Parse(Format(x)) == x` for all `x`, your parser/formatter pair is solid.

---

## Bogus — realistic fake data

Bogus generates **realistic** test data (names, emails, addresses) — useful for seeding tests and for property/fuzz inputs that look like real domain objects.

```csharp
using Bogus;

var faker = new Faker<Customer>()
    .RuleFor(c => c.Id, f => f.IndexFaker)
    .RuleFor(c => c.Name, f => f.Name.FullName())
    .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.Name))
    .RuleFor(c => c.BirthDate, f => f.Date.Past(80))
    .RuleFor(c => c.Address, f => f.Address.FullAddress());

Customer one = faker.Generate();
List<Customer> many = faker.Generate(100);
```

Use a fixed seed (`Randomizer.Seed = new Random(12345);`) for reproducible test data. Bogus is for *plausible* data; FsCheck is for *adversarial* edge-case data — they complement each other.

---

## When property-based testing shines

- **Parsers/serializers** — round-trip properties find edge cases instantly.
- **Data structures / algorithms** — sort, search, collections (invariants).
- **Math/encoding** — commutativity, associativity, inverses.
- **State machines** — model-based testing (FsCheck supports command sequences).
- **Anywhere with clear invariants** — properties express the *spec* directly.

When example-based is better:
- Specific business rules with few discrete cases ("VIP customers get 20% off").
- Tests documenting a particular scenario/bug fix.
- When you can't articulate a general property.

Most codebases use both: example tests for specific scenarios, property tests for invariant-heavy logic.

---

## Common bugs / gotchas

### Property that's trivially true

```csharp
[Property]
public bool Always(int x) => Add(x, 0) == Add(x, 0);   // tests nothing — both sides identical
```

Make sure the property actually constrains behavior (compare to an independent oracle or a real invariant).

### Generator produces invalid inputs

If your function has preconditions (non-null, positive), unconstrained generators feed invalid data and the property fails for the wrong reason. Use `Implies` or custom generators to stay in the valid domain.

### Flaky from too few iterations / unbounded data

Default ~100 runs may miss rare cases; huge generated collections may be slow. Tune iteration count and size for the property.

### Non-deterministic property

A property depending on `DateTime.Now`/external state fails randomly. Properties must be pure functions of their generated inputs.

---

## Summary

- Property-based testing checks **invariants over hundreds of generated inputs**, finding edge cases you wouldn't enumerate.
- **FsCheck** (`[Property]`) generates inputs, checks a `bool`/`Property`, and **shrinks** failures to a minimal counterexample — its most valuable feature.
- Look for properties: round-trip/inverse, idempotence, commutativity, invariants, oracle comparison.
- **Bogus** generates realistic fake data (use a fixed seed for reproducibility); it complements FsCheck's adversarial data.
- Use property tests for parsers, algorithms, and invariant-rich code; example tests for specific business scenarios. Most projects use both.

→ Next: [07-BenchmarkDotNet.md](07-BenchmarkDotNet.md)
