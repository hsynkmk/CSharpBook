# Assertions

## What they are

An assertion checks that a value meets an expectation; if not, the test fails with a message. The **quality of that failure message** determines how fast you diagnose a failing test. Fluent assertion libraries (FluentAssertions, Shouldly) produce far more readable assertions and better failure output than raw `Assert.Equal`.

```csharp
// Built-in xUnit
Assert.Equal(5, result);
// Failure: "Assert.Equal() Failure: Expected: 5, Actual: 3"

// FluentAssertions
result.Should().Be(5);
// Failure: "Expected result to be 5, but found 3."
```

The fluent version reads as a sentence and (for complex objects/collections) shows *exactly what differs*.

---

## Why failure messages matter

Compare diagnosing a collection mismatch:

```csharp
// Built-in
Assert.Equal(expected, actual);
// Failure: "Assert.Equal() Failure" + a hard-to-read diff

// FluentAssertions
actual.Should().BeEquivalentTo(expected);
// Failure: "Expected item[2].Name to be \"Bob\", but found \"Bobby\"."
```

A precise message ("item[2].Name differs") saves minutes per failure. Over a project's life, that's enormous. Good assertions are an investment in debugging speed.

---

## FluentAssertions

The most popular fluent assertion library. `.Should()` extends every type.

```csharp
using FluentAssertions;

// Scalars
result.Should().Be(5);
name.Should().NotBeNullOrEmpty();
value.Should().BeGreaterThan(0).And.BeLessThan(100);
text.Should().StartWith("Hello").And.Contain("world");
flag.Should().BeTrue();
obj.Should().BeNull();

// Numbers with tolerance
pi.Should().BeApproximately(3.14, precision: 0.01);

// Types
animal.Should().BeOfType<Dog>();
dog.Should().BeAssignableTo<Animal>();

// Collections
list.Should().HaveCount(3);
list.Should().Contain(42);
list.Should().BeInAscendingOrder();
list.Should().OnlyContain(x => x > 0);
list.Should().BeEquivalentTo(new[] { 3, 1, 2 });   // order-independent by default
list.Should().Equal(1, 2, 3);                        // order matters
list.Should().ContainSingle(x => x.IsActive);

// Object graphs — deep structural comparison
actual.Should().BeEquivalentTo(expected);            // compares all properties recursively
actual.Should().BeEquivalentTo(expected, opts =>
    opts.Excluding(x => x.Id).Excluding(x => x.Timestamp));   // ignore volatile fields

// Exceptions
Action act = () => service.Process(null!);
act.Should().Throw<ArgumentNullException>()
   .WithMessage("*input*")
   .And.ParamName.Should().Be("input");

// Async exceptions
Func<Task> act = async () => await service.FetchAsync(-1);
await act.Should().ThrowAsync<ArgumentOutOfRangeException>();

// No exception
act.Should().NotThrow();

// Execution time
act.ExecutionTime().Should().BeLessThan(100.Milliseconds());
```

### `BeEquivalentTo` — the killer feature

Deep, property-by-property comparison of object graphs without writing per-field asserts:

```csharp
var actual = service.GetCustomer(1);
actual.Should().BeEquivalentTo(new Customer {
    Id = 1, Name = "Alice", Address = new() { City = "Boston" }
});
```

It recurses through nested objects and collections, reports the exact path that differs, and ignores reference identity (structural equality). Configure to exclude volatile fields, match by member name, handle ordering, etc.

> Licensing note: FluentAssertions v8+ changed to a commercial license for some uses. Many teams pin v7 (free) or use Shouldly/AwesomeAssertions (a free fork). Check current licensing for your context.

---

## Shouldly

A lighter fluent library; method-call style. Its standout feature: failure messages that **echo the asserted expression**.

```csharp
using Shouldly;

result.ShouldBe(5);
name.ShouldNotBeNullOrEmpty();
value.ShouldBeGreaterThan(0);
list.ShouldContain(42);
list.ShouldBe(new[] { 1, 2, 3 });
animal.ShouldBeOfType<Dog>();

Should.Throw<ArgumentNullException>(() => service.Process(null!));
await Should.ThrowAsync<TimeoutException>(async () => await SlowAsync());

// Source-aware messages:
// result.ShouldBe(5);  →  "result should be 5 but was 3"
```

Shouldly reads the source to include the variable name in the message — `"result should be 5 but was 3"` — which built-in asserts can't do.

---

## Comparison

| | Built-in (xUnit/NUnit/MSTest) | FluentAssertions | Shouldly |
|---|---|---|---|
| Readability | terse | sentence-like | method + source echo |
| Object graph compare | manual | `BeEquivalentTo` (excellent) | basic |
| Failure messages | basic | very detailed | source-aware |
| Chaining (`.And.`) | no | yes | no |
| Extra dependency | no | yes | yes |
| License | free | v8+ commercial-ish | free |

Built-in assertions are perfectly fine for simple checks. Fluent libraries pay off for complex objects/collections and team-wide readability.

---

## Writing good assertions

### One logical assertion per test

```csharp
// ✗ — testing two behaviors; unclear what failed
[Fact]
public void Process() {
    result.Success.Should().BeTrue();
    result.Items.Should().HaveCount(3);
    audit.Verify(...);   // different concern
}

// ✓ — focused; the test name says what's verified
[Fact]
public void Process_ValidInput_Succeeds() => result.Success.Should().BeTrue();
```

Multiple `Should()` calls are fine if they together express *one* logical expectation (a result and its shape).

### Assert on behavior, not implementation

Prefer `result.Should().Be(expected)` over verifying internal method calls — outputs survive refactoring; interactions don't.

### Use specific assertions

```csharp
// Weak — message just says "false"
Assert.True(list.Count == 3);

// Strong — message says "expected count 3 but found 2"
list.Should().HaveCount(3);
```

Specific assertions produce specific failure messages. `Should().HaveCount(3)` beats `Should().BeTrue()` on `Count == 3`.

---

## Common bugs / gotchas

### `BeEquivalentTo` vs `Be`

`Be` (or `Equal`) uses the type's equality; `BeEquivalentTo` does structural property comparison. For records (value equality) they may agree; for classes (reference equality), `Be` fails unless it's the same instance. Use `BeEquivalentTo` for "same data."

### Comparing volatile fields

`BeEquivalentTo` on objects with timestamps/Guids fails on those fields. Exclude them: `opts => opts.Excluding(x => x.CreatedAt)`.

### Float equality

`pi.Should().Be(3.14)` fails on binary float. Use `BeApproximately(3.14, 0.001)`.

### Assertion inside a loop without context

```csharp
foreach (var x in items) x.Should().BePositive();   // which item failed? unclear
items.Should().OnlyContain(x => x > 0);              // clearer, single assertion
```

---

## Summary

- The value of an assertion is its **failure message** — fluent libraries produce far clearer ones.
- **FluentAssertions** (`.Should().Be(...)`, chaining, `BeEquivalentTo` for deep object/collection comparison) is the most powerful; mind the v8+ licensing.
- **Shouldly** (`.ShouldBe(...)`) is lighter and echoes the source expression in messages.
- Built-in asserts are fine for simple checks; fluent libraries shine for complex graphs and readability.
- Write one logical assertion per test, assert on behavior not implementation, and use specific assertions for specific failure messages.

→ Next: [05-IntegrationTesting.md](05-IntegrationTesting.md)
