# Integration Testing

## What it is

Unit tests verify a single class in isolation (with mocked dependencies). **Integration tests** verify that components work together — the real DI container, real middleware pipeline, real (or realistic) database — exercising the actual request path end to end.

```
Unit test:        OrderService (mocked repo, mocked gateway)
Integration test: HTTP request → routing → middleware → controller → real service → real DB → response
```

Integration tests catch wiring bugs that unit tests can't: misconfigured DI, broken serialization, wrong routes, EF Core query mistakes, middleware order issues.

---

## The test pyramid

```
        /\          Few, slow, high-confidence
       /E2E\         (full system, browser)
      /------\
     / Integ. \      Some — components together
    /----------\
   /   Unit     \    Many, fast, isolated
  /--------------\
```

Many fast unit tests at the base, fewer integration tests in the middle, a handful of end-to-end tests at the top. Integration tests are slower than unit tests but give confidence that the pieces connect.

---

## `WebApplicationFactory<TEntryPoint>` — ASP.NET Core in-memory

The standard tool for testing ASP.NET Core apps. It boots your real app in memory (no network, no Kestrel port) and gives you an `HttpClient` wired to it.

```csharp
// The app's Program must expose its entry point. With top-level statements,
// add at the end of Program.cs:  public partial class Program;

public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>> {
    private readonly WebApplicationFactory<Program> _factory;
    public OrdersApiTests(WebApplicationFactory<Program> factory) => _factory = factory;

    [Fact]
    public async Task GetOrder_ExistingId_Returns200() {
        HttpClient client = _factory.CreateClient();

        HttpResponseMessage response = await client.GetAsync("/api/orders/1");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var order = await response.Content.ReadFromJsonAsync<OrderDto>();
        order!.Id.Should().Be(1);
    }
}
```

`CreateClient()` returns an `HttpClient` that calls your app's pipeline directly through an in-memory `TestServer` — fast, no sockets, full middleware.

---

## Replacing services for tests

Override real dependencies (e.g., swap a real DB for in-memory, stub an external API) via `WithWebHostBuilder` + `ConfigureTestServices`:

```csharp
public class OrdersApiTests : IClassFixture<WebApplicationFactory<Program>> {
    private readonly WebApplicationFactory<Program> _factory;

    public OrdersApiTests(WebApplicationFactory<Program> factory) =>
        _factory = factory.WithWebHostBuilder(builder => {
            builder.ConfigureTestServices(services => {
                // Remove the real registration and add a test double
                services.RemoveAll<IPaymentGateway>();
                services.AddSingleton<IPaymentGateway, FakePaymentGateway>();

                // Swap the DB for in-memory or a test container connection
                services.RemoveAll<DbContextOptions<AppDbContext>>();
                services.AddDbContext<AppDbContext>(o => o.UseSqlite("DataSource=:memory:"));
            });
        });

    [Fact]
    public async Task Checkout_UsesFakeGateway() { /* ... */ }
}
```

`ConfigureTestServices` runs **after** the app's normal registration, so your overrides win. This lets you keep the real pipeline while controlling the dangerous/slow bits (payments, email, external HTTP).

---

## Testing the database

Three strategies, increasing fidelity:

### 1. EF Core in-memory provider (lowest fidelity)

```csharp
services.AddDbContext<AppDbContext>(o => o.UseInMemoryDatabase("test"));
```

Fast but **not a real database** — no SQL, no constraints, no transactions, different query translation. Catches basic logic but misses real DB behavior. Use sparingly.

### 2. SQLite in-memory (medium fidelity)

```csharp
var conn = new SqliteConnection("DataSource=:memory:");
conn.Open();   // keep open — closing drops the in-memory DB
services.AddDbContext<AppDbContext>(o => o.UseSqlite(conn));
```

A real SQL engine (constraints, transactions, real query translation) but not your production engine (Postgres/SQL Server). Good middle ground.

### 3. Testcontainers (highest fidelity)

Spin up the **real database** in a Docker container per test run:

```csharp
using Testcontainers.PostgreSql;

public class DbFixture : IAsyncLifetime {
    private readonly PostgreSqlContainer _db = new PostgreSqlBuilder()
        .WithImage("postgres:16")
        .Build();

    public string ConnectionString => _db.GetConnectionString();

    public async Task InitializeAsync() {
        await _db.StartAsync();                    // start the container
        // run migrations against it
    }
    public async Task DisposeAsync() => await _db.DisposeAsync();   // tear down
}
```

Testcontainers gives production-equivalent behavior (real Postgres/SQL Server/Redis/etc.) at the cost of needing Docker and slower startup. The gold standard for integration tests that must trust DB behavior.

---

## Seeding and cleanup

Each integration test should start from a known state. Options:

- **Respawn** (library) — resets the DB to empty between tests quickly.
- **Transaction rollback** — wrap each test in a transaction and roll back (fast, but doesn't test commit behavior).
- **Recreate per test** — slow but fully isolated.
- **Unique data per test** — each test uses distinct keys/tenants so they don't collide (enables parallelism).

```csharp
public async Task InitializeAsync() {
    await using var scope = _factory.Services.CreateAsyncScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.EnsureCreatedAsync();
    db.Orders.Add(new Order { Id = 1, Total = 99m });
    await db.SaveChangesAsync();
}
```

---

## Authentication in integration tests

Bypass real auth with a test authentication handler:

```csharp
builder.ConfigureTestServices(services => {
    services.AddAuthentication("Test")
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>("Test", _ => {});
});

// TestAuthHandler returns a fixed authenticated ClaimsPrincipal
```

Then `CreateClient()` requests are authenticated without real tokens — you test authorization logic without an identity provider.

---

## Testing the full HTTP contract

```csharp
[Fact]
public async Task CreateOrder_ValidatesAndPersists() {
    var client = _factory.CreateClient();

    var response = await client.PostAsJsonAsync("/api/orders",
        new CreateOrderRequest { CustomerId = 1, Total = 50m });

    response.StatusCode.Should().Be(HttpStatusCode.Created);
    response.Headers.Location.Should().NotBeNull();

    var created = await response.Content.ReadFromJsonAsync<OrderDto>();
    created!.Total.Should().Be(50m);

    // Verify persistence via a follow-up request or direct DB query
    var fetched = await client.GetAsync(response.Headers.Location);
    fetched.StatusCode.Should().Be(HttpStatusCode.OK);
}
```

This exercises routing, model binding, validation, serialization, the controller/handler, the service, and the database — the whole vertical slice.

---

## Common bugs / gotchas

### Forgetting `public partial class Program;`

With top-level statements, `WebApplicationFactory<Program>` needs `Program` to be accessible. Add `public partial class Program;` at the end of `Program.cs` (or use `[assembly: InternalsVisibleTo]`).

### SQLite in-memory DB disappearing

A `:memory:` SQLite DB exists only while a connection is open. Closing the connection drops the data. Keep one connection open for the test's lifetime.

### Tests interfering via shared DB state

Parallel integration tests hitting the same database race. Use per-test isolation (Respawn, unique data, or serialize via a collection fixture). See [Chapter 16 §01](01-xUnit.md).

### Slow tests from per-test container startup

Starting a Testcontainer per test is very slow. Start it once per collection (`ICollectionFixture` + `IAsyncLifetime`) and reset data between tests instead.

### Over-relying on in-memory EF provider

It diverges from real SQL (no constraint enforcement, different query translation). A test passing on in-memory may fail on Postgres. Prefer SQLite or Testcontainers for query-sensitive tests.

---

## Summary

- Integration tests verify components working together (real DI, pipeline, DB) — catching wiring bugs unit tests miss.
- Follow the pyramid: many unit, fewer integration, few E2E.
- **`WebApplicationFactory<Program>`** boots ASP.NET Core in memory with a wired `HttpClient`; `ConfigureTestServices` swaps real deps for test doubles.
- DB fidelity ladder: EF in-memory (low) → SQLite in-memory (medium) → **Testcontainers** (real DB, high).
- Isolate per-test state (Respawn, transactions, unique data); start containers once per collection.
- Add `public partial class Program;`, keep SQLite connections open, and avoid over-trusting the in-memory provider.

→ Next: [06-PropertyBased.md](06-PropertyBased.md)
