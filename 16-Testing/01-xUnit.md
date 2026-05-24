# xUnit

## What it is

xUnit.net is the de-facto standard unit-testing framework for modern .NET. Tests are plain methods marked with attributes; assertions are static `Assert` calls. It emphasizes a clean object lifecycle (a fresh test-class instance per test) and parallel execution.

```csharp
using Xunit;

public class CalculatorTests {
    [Fact]
    public void Add_TwoNumbers_ReturnsSum() {
        var calc = new Calculator();
        int result = calc.Add(2, 3);
        Assert.Equal(5, result);
    }
}
```

```bash
dotnet new xunit -o MyTests
dotnet test
```

---

## `[Fact]` — a single test

A `[Fact]` is a test with no parameters — one scenario, one method.

```csharp
[Fact]
public void Withdraw_InsufficientFunds_Throws() {
    var account = new Account(balance: 100);
    var ex = Assert.Throws<InvalidOperationException>(() => account.Withdraw(200));
    Assert.Equal("Insufficient funds", ex.Message);
}
```

### Arrange-Act-Assert

The standard structure:

```csharp
[Fact]
public void Test() {
    // Arrange — set up inputs and the system under test
    var sut = new OrderService(repo);

    // Act — invoke the one behavior under test
    var result = sut.Place(order);

    // Assert — verify the outcome
    Assert.True(result.Success);
}
```

One logical assertion per test (you may need multiple `Assert` calls to express it). Test one behavior per method.

---

## `[Theory]` — parameterized tests

A `[Theory]` runs the same logic with multiple data sets. Each row is a separate test case in the runner.

```csharp
[Theory]
[InlineData(2, 3, 5)]
[InlineData(-1, 1, 0)]
[InlineData(0, 0, 0)]
public void Add_VariousInputs_ReturnsSum(int a, int b, int expected) {
    Assert.Equal(expected, new Calculator().Add(a, b));
}
```

### Data sources

```csharp
// MemberData — from a property/method returning IEnumerable<object[]>
public static IEnumerable<object[]> AddCases => new[] {
    new object[] { 2, 3, 5 },
    new object[] { 10, -4, 6 },
};

[Theory]
[MemberData(nameof(AddCases))]
public void Add_FromMember(int a, int b, int expected) =>
    Assert.Equal(expected, new Calculator().Add(a, b));

// ClassData — from a class implementing IEnumerable<object[]>
[Theory]
[ClassData(typeof(AddTestData))]
public void Add_FromClass(int a, int b, int expected) =>
    Assert.Equal(expected, new Calculator().Add(a, b));
```

`InlineData` for simple literals; `MemberData`/`ClassData` for computed or complex data (objects that aren't compile-time constants). xUnit v3 also adds strongly-typed `TheoryData<T1,T2,...>`:

```csharp
public static TheoryData<int, int, int> Cases => new() {
    { 2, 3, 5 }, { 0, 0, 0 },
};
[Theory, MemberData(nameof(Cases))]
public void Add_Typed(int a, int b, int expected) => ...;
```

---

## Test lifecycle — constructor and Dispose

xUnit creates a **new instance of the test class for every test method** — tests are isolated by default, no shared mutable state between them.

```csharp
public class DatabaseTests : IDisposable {
    private readonly SqliteConnection _conn;

    public DatabaseTests() {                 // runs before EACH test (Arrange shared setup)
        _conn = new SqliteConnection("DataSource=:memory:");
        _conn.Open();
    }

    public void Dispose() {                    // runs after EACH test (cleanup)
        _conn.Dispose();
    }

    [Fact]
    public void Test1() { /* fresh _conn */ }
    [Fact]
    public void Test2() { /* a different fresh _conn */ }
}
```

Constructor = setup, `Dispose` = teardown. xUnit deliberately has **no `[SetUp]`/`[TearDown]` attributes** — it uses the language's own constructor/dispose. This is a key philosophical difference from NUnit/MSTest.

### Async lifecycle

For async setup/teardown, implement `IAsyncLifetime`:

```csharp
public class ApiTests : IAsyncLifetime {
    private HttpClient _client = null!;

    public async Task InitializeAsync() {         // async setup before each test
        _client = await CreateAuthenticatedClientAsync();
    }
    public async Task DisposeAsync() {            // async teardown
        _client.Dispose();
        await Task.CompletedTask;
    }
}
```

---

## Sharing context — fixtures

Recreating an expensive resource (database, web host) per test is slow. Fixtures share one instance across tests.

### `IClassFixture<T>` — shared within one test class

```csharp
public class DbFixture : IDisposable {
    public DbConnection Connection { get; }
    public DbFixture() { Connection = CreateAndSeed(); }   // once for the whole class
    public void Dispose() => Connection.Dispose();
}

public class OrderTests : IClassFixture<DbFixture> {
    private readonly DbFixture _fixture;
    public OrderTests(DbFixture fixture) => _fixture = fixture;   // injected

    [Fact] public void Test1() { /* uses _fixture.Connection */ }
}
```

The fixture is constructed once, shared across all tests in the class.

### `ICollectionFixture<T>` — shared across multiple classes

```csharp
[CollectionDefinition("Database collection")]
public class DatabaseCollection : ICollectionFixture<DbFixture> {}

[Collection("Database collection")]
public class OrderTests { /* shares DbFixture with other classes in the collection */ }

[Collection("Database collection")]
public class CustomerTests { /* same shared fixture */ }
```

Collection fixtures share one expensive resource across many test classes — and serialize those classes (no parallelism within the collection).

---

## Assertions

xUnit's built-in assertion vocabulary:

```csharp
Assert.Equal(expected, actual);
Assert.Equal(expected, actual, precision: 3);     // floating point
Assert.NotEqual(a, b);
Assert.True(condition);
Assert.False(condition);
Assert.Null(obj);
Assert.NotNull(obj);
Assert.Same(expected, actual);                     // reference equality
Assert.Contains("substr", actualString);
Assert.Contains(item, collection);
Assert.Empty(collection);
Assert.Single(collection);                         // exactly one element
Assert.Equal(new[]{1,2,3}, list);                  // sequence equality
Assert.IsType<Dog>(animal);                        // exact type
Assert.IsAssignableFrom<Animal>(dog);              // type or subtype
Assert.InRange(value, low, high);

// Exceptions
var ex = Assert.Throws<ArgumentNullException>(() => Foo(null));
await Assert.ThrowsAsync<TimeoutException>(() => BarAsync());

// Multiple — all evaluated, all failures reported
Assert.Multiple(
    () => Assert.Equal(5, result.Count),
    () => Assert.True(result.IsValid));
```

Many teams layer FluentAssertions/Shouldly on top for readability — see [04-Assertions.md](04-Assertions.md).

---

## Testing async code

```csharp
[Fact]
public async Task FetchAsync_ReturnsData() {
    var service = new DataService();
    var result = await service.FetchAsync(id: 1);   // await it — the test method is async Task
    Assert.NotNull(result);
}
```

Make the test `async Task` and `await`. **Never** use `async void` for tests (the runner can't observe completion or exceptions) and **never** block with `.Result`/`.Wait()` (can deadlock and hides exceptions). See [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md).

---

## Parallelism

xUnit runs test **collections** in parallel by default (different classes = different collections unless grouped). Tests **within** a class run sequentially.

```csharp
// Disable parallelism for an assembly
[assembly: CollectionBehavior(DisableTestParallelization = true)]

// Or limit threads
[assembly: CollectionBehavior(MaxParallelThreads = 4)]
```

Tests sharing mutable global state (static fields, a real database, environment variables) must be grouped into a collection to serialize them — otherwise they race. Prefer isolated, parallel-safe tests.

---

## Skipping and traits

```csharp
[Fact(Skip = "Flaky — tracked in #1234")]
public void NotReady() { }

[Fact]
[Trait("Category", "Integration")]
public void DbTest() { }
```

```bash
dotnet test --filter "Category=Integration"
dotnet test --filter "FullyQualifiedName~OrderTests"
```

Traits categorize tests for filtering in CI (run unit tests on every push, integration tests nightly).

---

## Output

```csharp
public class MyTests {
    private readonly ITestOutputHelper _output;
    public MyTests(ITestOutputHelper output) => _output = output;

    [Fact]
    public void Test() {
        _output.WriteLine("diagnostic info");   // shows in test output, not Console
    }
}
```

Use `ITestOutputHelper` (injected) for test diagnostics — `Console.WriteLine` doesn't reliably surface in parallel test runs.

---

## Common bugs / gotchas

### Shared mutable state across parallel tests

Static fields or a shared external resource mutated by parallel tests cause flaky failures. Isolate state or group into a serialized collection.

### `async void` tests

The runner can't await them — failures are missed and the process may crash. Always `async Task`.

### Assuming test order

xUnit doesn't guarantee execution order (and randomizes/parallelizes). Each test must be independent — no "test B depends on test A having run."

### Over-using fixtures for cheap setup

Fixtures add complexity. Only share genuinely expensive resources; for cheap setup, the constructor (per-test) is simpler and safer.

### `Assert.Equal` on floating point

`Assert.Equal(0.3, 0.1 + 0.2)` fails (binary float). Use the `precision` overload or a tolerance.

---

## Summary

- xUnit: `[Fact]` (one case), `[Theory]` + `InlineData`/`MemberData`/`ClassData`/`TheoryData` (parameterized).
- Lifecycle uses the **constructor** (setup) and `Dispose`/`IAsyncLifetime` (teardown) — fresh instance per test, no `[SetUp]`.
- Share expensive resources with `IClassFixture<T>` (per class) or `ICollectionFixture<T>` (across classes).
- Test async with `async Task` + `await`; never `async void` or `.Result`.
- Collections run in parallel; isolate or serialize shared state.
- Use traits for filtering, `ITestOutputHelper` for diagnostics; mind float comparisons and test independence.

→ Next: [02-NUnitMSTest.md](02-NUnitMSTest.md)
