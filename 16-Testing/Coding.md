# Chapter 16 — Testing — Coding Problems

---

## Problem 1: Test a method that throws

Write xUnit tests for `Account.Withdraw` covering success, insufficient funds, and negative amount.

```csharp
public class Account(decimal balance) {
    public decimal Balance { get; private set; } = balance;
    public void Withdraw(decimal amount) {
        if (amount <= 0) throw new ArgumentOutOfRangeException(nameof(amount));
        if (amount > Balance) throw new InvalidOperationException("Insufficient funds");
        Balance -= amount;
    }
}
```

<details><summary>Solution</summary>

```csharp
public class AccountTests {
    [Fact]
    public void Withdraw_ValidAmount_ReducesBalance() {
        var account = new Account(100m);
        account.Withdraw(30m);
        Assert.Equal(70m, account.Balance);
    }

    [Fact]
    public void Withdraw_MoreThanBalance_Throws() {
        var account = new Account(100m);
        var ex = Assert.Throws<InvalidOperationException>(() => account.Withdraw(200m));
        Assert.Equal("Insufficient funds", ex.Message);
        Assert.Equal(100m, account.Balance);   // balance unchanged
    }

    [Theory]
    [InlineData(0)]
    [InlineData(-10)]
    public void Withdraw_NonPositive_Throws(decimal amount) {
        var account = new Account(100m);
        Assert.Throws<ArgumentOutOfRangeException>(() => account.Withdraw(amount));
    }
}
```

Note testing that state is unchanged on failure — a common omission.

</details>

---

## Problem 2: Parameterized test with computed data

Test a `Fibonacci(n)` function for several inputs using `MemberData`.

<details><summary>Solution</summary>

```csharp
public class FibTests {
    public static IEnumerable<object[]> Cases => new[] {
        new object[] { 0, 0 },
        new object[] { 1, 1 },
        new object[] { 2, 1 },
        new object[] { 10, 55 },
    };

    [Theory]
    [MemberData(nameof(Cases))]
    public void Fibonacci_ReturnsExpected(int n, int expected) =>
        Assert.Equal(expected, Math.Fibonacci(n));

    // Strongly-typed alternative (xUnit v3):
    public static TheoryData<int, int> TypedCases => new() {
        { 0, 0 }, { 1, 1 }, { 10, 55 },
    };
    [Theory, MemberData(nameof(TypedCases))]
    public void Fibonacci_Typed(int n, int expected) =>
        Assert.Equal(expected, Math.Fibonacci(n));
}
```

`InlineData` for literals; `MemberData`/`TheoryData` when data is computed or complex.

</details>

---

## Problem 3: Test an async method with a fixture

Test a `UserRepository` backed by an in-memory SQLite database, sharing the connection across tests in the class.

<details><summary>Solution</summary>

```csharp
public class RepoFixture : IDisposable {
    public SqliteConnection Connection { get; }
    public RepoFixture() {
        Connection = new SqliteConnection("DataSource=:memory:");
        Connection.Open();   // keep open so the in-memory DB survives
        // create schema...
    }
    public void Dispose() => Connection.Dispose();
}

public class UserRepositoryTests : IClassFixture<RepoFixture>, IAsyncLifetime {
    private readonly UserRepository _repo;
    public UserRepositoryTests(RepoFixture fixture) => _repo = new UserRepository(fixture.Connection);

    public async Task InitializeAsync() => await _repo.ClearAsync();   // reset per test
    public Task DisposeAsync() => Task.CompletedTask;

    [Fact]
    public async Task AddAndGet_RoundTrips() {
        await _repo.AddAsync(new User(1, "Alice"));
        var fetched = await _repo.GetAsync(1);
        Assert.Equal("Alice", fetched!.Name);
    }
}
```

`IClassFixture` shares the expensive connection; `IAsyncLifetime.InitializeAsync` resets state per test so they stay independent. Keep the SQLite connection open or the DB vanishes.

</details>

---

## Problem 4: Mock dependencies with Moq

Test `OrderService.Place` verifies it charges the gateway and sends a confirmation email only on success.

```csharp
public class OrderService(IPaymentGateway gateway, IEmailSender email) {
    public bool Place(Order order) {
        if (!gateway.Charge(order.Total)) return false;
        email.Send(order.CustomerEmail, "Confirmed");
        return true;
    }
}
```

<details><summary>Solution</summary>

```csharp
public class OrderServiceTests {
    [Fact]
    public void Place_PaymentSucceeds_SendsEmail() {
        var gateway = new Mock<IPaymentGateway>();
        var email = new Mock<IEmailSender>();
        gateway.Setup(g => g.Charge(100m)).Returns(true);

        var sut = new OrderService(gateway.Object, email.Object);
        bool result = sut.Place(new Order { Total = 100m, CustomerEmail = "a@b.com" });

        Assert.True(result);
        email.Verify(e => e.Send("a@b.com", "Confirmed"), Times.Once);
    }

    [Fact]
    public void Place_PaymentFails_DoesNotSendEmail() {
        var gateway = new Mock<IPaymentGateway>();
        var email = new Mock<IEmailSender>();
        gateway.Setup(g => g.Charge(It.IsAny<decimal>())).Returns(false);

        var sut = new OrderService(gateway.Object, email.Object);
        bool result = sut.Place(new Order { Total = 100m, CustomerEmail = "a@b.com" });

        Assert.False(result);
        email.Verify(e => e.Send(It.IsAny<string>(), It.IsAny<string>()), Times.Never);
    }
}
```

The second test verifies the *negative* path — no email on failure. Verifying `Times.Never` is as important as `Times.Once`.

</details>

---

## Problem 5: Same test with NSubstitute

Rewrite Problem 4's success test using NSubstitute.

<details><summary>Solution</summary>

```csharp
[Fact]
public void Place_PaymentSucceeds_SendsEmail() {
    var gateway = Substitute.For<IPaymentGateway>();
    var email = Substitute.For<IEmailSender>();
    gateway.Charge(100m).Returns(true);

    var sut = new OrderService(gateway, email);
    bool result = sut.Place(new Order { Total = 100m, CustomerEmail = "a@b.com" });

    result.Should().BeTrue();
    email.Received(1).Send("a@b.com", "Confirmed");
    gateway.Received().Charge(100m);
}
```

No `.Object`, `.Returns()` directly on the call, `.Received()` to verify. Cleaner syntax.

</details>

---

## Problem 6: Wrap a third-party SDK for testability

A class uses `StripeClient` (concrete, hard to mock) directly. Make it testable.

```csharp
public class CheckoutService {
    private readonly StripeClient _stripe = new(apiKey);
    public bool Charge(decimal amount) => _stripe.CreateCharge(amount).Succeeded;
}
```

<details><summary>Solution</summary>

```csharp
// 1. Define your own interface (a seam you own)
public interface IPaymentGateway {
    bool Charge(decimal amount);
}

// 2. Adapter wrapping the SDK
public class StripeGateway(StripeClient client) : IPaymentGateway {
    public bool Charge(decimal amount) => client.CreateCharge(amount).Succeeded;
}

// 3. Service depends on the interface
public class CheckoutService(IPaymentGateway gateway) {
    public bool Charge(decimal amount) => gateway.Charge(amount);
}

// 4. Test mocks YOUR interface, not Stripe's types
[Fact]
public void Charge_DelegatesToGateway() {
    var gateway = Substitute.For<IPaymentGateway>();
    gateway.Charge(50m).Returns(true);
    var sut = new CheckoutService(gateway);
    sut.Charge(50m).Should().BeTrue();
}
```

Don't mock types you don't own. Wrap behind an interface, mock the interface. The thin `StripeGateway` adapter is tested separately (or via integration tests).

</details>

---

## Problem 7: Integration test with WebApplicationFactory

Test that `GET /api/products/1` returns 200 and the right product, with a fake repository.

<details><summary>Solution</summary>

```csharp
// In the API's Program.cs (end of file):  public partial class Program;

public class ProductApiTests : IClassFixture<WebApplicationFactory<Program>> {
    private readonly WebApplicationFactory<Program> _factory;

    public ProductApiTests(WebApplicationFactory<Program> factory) =>
        _factory = factory.WithWebHostBuilder(b => b.ConfigureTestServices(services => {
            services.RemoveAll<IProductRepository>();
            services.AddSingleton<IProductRepository>(new FakeProductRepository(
                new Product(1, "Widget", 9.99m)));
        }));

    [Fact]
    public async Task GetProduct_Existing_Returns200() {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/products/1");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var product = await response.Content.ReadFromJsonAsync<Product>();
        product!.Name.Should().Be("Widget");
    }

    [Fact]
    public async Task GetProduct_Missing_Returns404() {
        var client = _factory.CreateClient();
        var response = await client.GetAsync("/api/products/999");
        response.StatusCode.Should().Be(HttpStatusCode.NotFound);
    }
}
```

Exercises the full pipeline (routing, controller, serialization) with a fake repo swapped in via `ConfigureTestServices`.

</details>

---

## Problem 8: Property-based test for a round-trip

Write an FsCheck property verifying that your JSON serializer round-trips any `Person`.

<details><summary>Solution</summary>

```csharp
public record Person(string Name, int Age, bool Active);

public class SerializationProperties {
    [Property]
    public bool Json_RoundTrips(Person original) {
        string json = JsonSerializer.Serialize(original);
        Person? restored = JsonSerializer.Deserialize<Person>(json);
        return original == restored;   // records have value equality
    }

    // Property for a known invariant
    [Property]
    public bool Reverse_Twice_IsIdentity(int[] xs) =>
        xs.Reverse().Reverse().SequenceEqual(xs);
}
```

FsCheck generates many random `Person` values (including empty strings, negative ages, `int.MaxValue`) and shrinks any failure to a minimal case. The round-trip property is the most broadly useful kind. (You may need a custom `Arbitrary` if `Name` could be null and your serializer can't handle it.)

</details>

---

## Problem 9: Generate realistic test data with Bogus

Create a faker that generates valid `Customer` objects for seeding tests, reproducibly.

<details><summary>Solution</summary>

```csharp
public static class CustomerFaker {
    public static Faker<Customer> Create() {
        Randomizer.Seed = new Random(12345);   // reproducible
        return new Faker<Customer>()
            .RuleFor(c => c.Id, f => f.IndexFaker + 1)
            .RuleFor(c => c.Name, f => f.Name.FullName())
            .RuleFor(c => c.Email, (f, c) => f.Internet.Email(c.Name))
            .RuleFor(c => c.Age, f => f.Random.Int(18, 90))
            .RuleFor(c => c.Country, f => f.Address.Country());
    }
}

[Fact]
public void Repository_HandlesManyCustomers() {
    var customers = CustomerFaker.Create().Generate(100);
    var repo = new InMemoryCustomerRepo();
    foreach (var c in customers) repo.Add(c);
    repo.Count.Should().Be(100);
    repo.GetByEmail(customers[0].Email).Should().NotBeNull();
}
```

The fixed seed makes the generated data deterministic across runs — failures are reproducible. Bogus complements FsCheck: realistic data here, adversarial edge cases there.

</details>

---

## Problem 10: Benchmark two approaches

Benchmark `List.Contains` (linear) vs `HashSet.Contains` (hash) for membership testing, reporting allocations.

<details><summary>Solution</summary>

```csharp
[MemoryDiagnoser]
public class MembershipBench {
    [Params(100, 10_000)]
    public int N;

    private List<int> _list = null!;
    private HashSet<int> _set = null!;
    private int _target;

    [GlobalSetup]
    public void Setup() {
        _list = Enumerable.Range(0, N).ToList();
        _set = new HashSet<int>(_list);
        _target = N - 1;   // worst case for the list (last element)
    }

    [Benchmark(Baseline = true)]
    public bool ListContains() => _list.Contains(_target);

    [Benchmark]
    public bool HashSetContains() => _set.Contains(_target);
}

// Program.cs
BenchmarkRunner.Run<MembershipBench>();
```

Run `dotnet run -c Release`. Expected: `ListContains` grows with N (O(n)); `HashSetContains` stays flat (O(1)), with the gap widening at N=10,000. Returning the `bool` prevents dead-code elimination; `[Params]` shows the scaling difference. This quantifies the "use the right collection" lesson from [Chapter 07](../07-Collections/README.md).

</details>

---

These problems cover the full testing toolkit: xUnit facts/theories/fixtures, async tests, Moq + NSubstitute, wrapping un-mockable SDKs, integration tests with `WebApplicationFactory`, property-based round-trips, Bogus data, and rigorous benchmarking.

→ Back to [Chapter 16 README](README.md). Next chapter: [Chapter 17 — Best Practices](../17-BestPractices/README.md).
