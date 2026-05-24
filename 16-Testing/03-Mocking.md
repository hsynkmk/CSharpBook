# Mocking

## What it is

A **test double** is a stand-in for a real dependency, letting you test a unit in isolation. Mocking libraries (Moq, NSubstitute) generate these doubles at runtime from an interface or virtual class, so you can control what dependencies return and verify how they're called.

```csharp
// Real code depends on an interface
public class OrderService(IPaymentGateway gateway, IEmailSender email) {
    public bool Place(Order order) {
        if (!gateway.Charge(order.Total)) return false;
        email.Send(order.CustomerEmail, "Order confirmed");
        return true;
    }
}

// Test substitutes fakes for the gateway and email
```

Mocking lets you test `OrderService` without a real payment processor or SMTP server.

---

## The vocabulary (test double types)

| Type | Purpose |
|---|---|
| **Dummy** | Passed but never used (fills a parameter). |
| **Stub** | Returns canned answers to calls (provides indirect input). |
| **Mock** | A stub that **also verifies** it was called as expected (checks indirect output). |
| **Fake** | A working but simplified implementation (e.g., in-memory repository). |
| **Spy** | Records calls for later inspection. |

Common usage blurs these — people say "mock" for all of them. The meaningful distinction: a **stub** provides inputs; a **mock** asserts on interactions.

---

## Moq

The most popular mocking library.

```csharp
using Moq;

[Fact]
public void Place_PaymentSucceeds_SendsEmailAndReturnsTrue() {
    // Arrange — create mocks
    var gateway = new Mock<IPaymentGateway>();
    var email = new Mock<IEmailSender>();

    // Stub: configure return values
    gateway.Setup(g => g.Charge(It.IsAny<decimal>())).Returns(true);

    var service = new OrderService(gateway.Object, email.Object);
    var order = new Order { Total = 99m, CustomerEmail = "a@b.com" };

    // Act
    bool result = service.Place(order);

    // Assert — verify outcome
    Assert.True(result);

    // Verify interactions (mock behavior)
    email.Verify(e => e.Send("a@b.com", "Order confirmed"), Times.Once);
    gateway.Verify(g => g.Charge(99m), Times.Once);
}
```

### Moq features

```csharp
// Argument matchers
mock.Setup(x => x.Get(It.IsAny<int>())).Returns(new Item());
mock.Setup(x => x.Get(It.Is<int>(i => i > 0))).Returns(new Item());

// Return based on input
mock.Setup(x => x.Square(It.IsAny<int>())).Returns((int n) => n * n);

// Sequence of returns
mock.SetupSequence(x => x.Next()).Returns(1).Returns(2).Throws<InvalidOperationException>();

// Throwing
mock.Setup(x => x.Risky()).Throws(new TimeoutException());

// Async
mock.Setup(x => x.GetAsync(It.IsAny<int>())).ReturnsAsync(new Item());

// Out / ref
mock.Setup(x => x.TryGet(1, out item)).Returns(true);

// Verify never called
mock.Verify(x => x.Delete(It.IsAny<int>()), Times.Never);

// Strict mock — any unconfigured call throws
var strict = new Mock<IFoo>(MockBehavior.Strict);
```

---

## NSubstitute

A friendlier, less verbose syntax — many teams prefer it.

```csharp
using NSubstitute;

[Fact]
public void Place_PaymentSucceeds_SendsEmail() {
    var gateway = Substitute.For<IPaymentGateway>();
    var email = Substitute.For<IEmailSender>();

    gateway.Charge(Arg.Any<decimal>()).Returns(true);

    var service = new OrderService(gateway, email);
    service.Place(new Order { Total = 99m, CustomerEmail = "a@b.com" });

    // Verify
    email.Received(1).Send("a@b.com", "Order confirmed");
    gateway.Received().Charge(99m);
    email.DidNotReceive().Send(Arg.Any<string>(), "error");
}
```

```csharp
// Return based on args
calc.Square(Arg.Any<int>()).Returns(ci => { int n = ci.Arg<int>(); return n * n; });

// Async
repo.GetAsync(Arg.Any<int>()).Returns(new Item());

// Throwing
service.When(s => s.Risky()).Do(_ => throw new TimeoutException());

// Sequence
counter.Next().Returns(1, 2, 3);
```

NSubstitute uses the substitute object directly (no `.Object`), which reads more cleanly.

---

## When NOT to mock

Mocking has a cost: it couples tests to implementation details and can give false confidence.

### Don't mock types you don't own (without a wrapper)

Mocking a third-party SDK class directly couples your tests to its exact API. Instead, wrap it behind your own interface and mock that:

```csharp
// ✗ — mocking the concrete SDK type (brittle, may not be virtual)
var stripe = new Mock<StripeClient>();

// ✓ — wrap behind your interface, mock the interface
public interface IPaymentGateway { bool Charge(decimal amount); }
public class StripeGateway(StripeClient client) : IPaymentGateway { ... }
var gateway = new Mock<IPaymentGateway>();
```

### Don't mock value objects / DTOs

Mocking a `record` or simple data class is pointless — just construct a real one.

```csharp
// ✗
var order = new Mock<Order>();
// ✓
var order = new Order { Total = 99m };
```

### Don't over-verify

Verifying every interaction makes tests brittle — they break on harmless refactors. Verify only the interactions that are part of the contract (the email *was* sent), not incidental ones.

### Prefer fakes for stateful dependencies

For a repository, an in-memory fake is often clearer than configuring dozens of mock setups:

```csharp
public class InMemoryOrderRepo : IOrderRepository {
    private readonly Dictionary<int, Order> _store = new();
    public void Add(Order o) => _store[o.Id] = o;
    public Order? Get(int id) => _store.GetValueOrDefault(id);
}
```

A fake exercises real logic and is reusable across tests.

---

## Mocking requires seams

Mocking libraries can only substitute **interfaces** and **virtual/abstract members**. They generate a subclass/proxy at runtime (via Castle DynamicProxy) and override members.

- ✅ Interfaces (best — design for them via DI).
- ✅ Abstract classes, virtual methods.
- ❌ Non-virtual methods, sealed classes, static methods, extension methods — can't be overridden.

This is *why* depending on interfaces (DI) makes code testable. Static/`DateTime.Now`/`new` calls are untestable seams — inject an abstraction (`IClock`, factory) instead.

---

## Testing time, randomness, etc.

```csharp
// ✗ — untestable
public bool IsExpired() => DateTime.UtcNow > _expiry;

// ✓ — inject a clock (or use TimeProvider, .NET 8+)
public bool IsExpired(TimeProvider clock) => clock.GetUtcNow() > _expiry;

// In tests:
var fake = new FakeTimeProvider();
fake.SetUtcNow(new DateTimeOffset(2030, 1, 1, 0, 0, 0, TimeSpan.Zero));
Assert.True(thing.IsExpired(fake));
```

`TimeProvider` (`Microsoft.Extensions.TimeProvider.Testing` provides `FakeTimeProvider`) is the modern, built-in way to make time testable — no mocking needed.

---

## Common bugs / gotchas

### Mocking concrete classes with non-virtual methods

The mock silently uses the real (or default) behavior because non-virtual members can't be overridden. Design for interfaces.

### Over-specified setups

`Setup` on a method the code doesn't call (strict mock) fails; loose setups on wrong arguments return defaults silently, masking bugs. Match arguments precisely or use `It.IsAny` deliberately.

### Verifying implementation, not behavior

Tests that assert "method X called method Y in order Z" break on every refactor. Test observable behavior/outputs where possible.

### Mock returning null by default

An unconfigured mock method returns `default` (null for references, 0 for ints). A `NullReferenceException` in the test often means a missing `Setup`.

---

## Summary

- Test doubles isolate a unit: **stubs** provide inputs, **mocks** verify interactions, **fakes** are working simplifications.
- **Moq** (`.Setup`/`.Verify`, `.Object`) and **NSubstitute** (`.Returns`/`.Received`, direct object) are the two main libraries.
- Mock **interfaces**/virtual members only — this is why DI makes code testable.
- Don't mock: types you don't own (wrap them), DTOs/value objects, or over-verify incidental calls. Prefer in-memory **fakes** for stateful deps.
- Make time/randomness testable via injection (`TimeProvider`/`FakeTimeProvider`), not mocking statics.

→ Next: [04-Assertions.md](04-Assertions.md)
