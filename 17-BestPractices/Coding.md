# Chapter 17 — Best Practices — Coding Problems

Theme: take bad code, make it good. Each problem presents an anti-pattern; refactor to idiomatic C# 14.

---

## Problem 1: Fix sync-over-async

```csharp
public class DataLoader {
    public Data Load(int id) {
        return _repo.GetAsync(id).Result;   // deadlock risk, blocks a thread
    }
}
```

<details><summary>Solution</summary>

```csharp
public class DataLoader {
    public Task<Data> LoadAsync(int id) => _repo.GetAsync(id);   // async all the way; pass-through
}
```

Make the method async (and rename with `Async`). It just forwards, so return the task directly — no state machine needed. If callers were also blocking, propagate async up the chain. Never `.Result`.

</details>

---

## Problem 2: Fix exception swallowing and stack-trace loss

```csharp
public void Process(string file) {
    try {
        var data = File.ReadAllText(file);
        Save(data);
    } catch (Exception ex) {
        Console.WriteLine("error");
        throw ex;                            // resets stack trace
    }
}
```

<details><summary>Solution</summary>

```csharp
public void Process(string file) {
    try {
        var data = File.ReadAllText(file);
        Save(data);
    } catch (IOException ex) {
        _logger.LogError(ex, "Failed to read {File}", file);   // log with context
        throw;                                                  // preserve original stack
    }
}
```

Catch the **specific** exception you can reason about, log it with context, and `throw;` (not `throw ex;`) to preserve the stack. Don't swallow with an empty/generic catch. Use a real logger, not `Console`.

</details>

---

## Problem 3: Eliminate primitive obsession

```csharp
public void Transfer(string from, string to, decimal amount, string currency) {
    // from/to are both strings — easy to swap by mistake
}
Transfer(toAccount, fromAccount, 100m, "USD");   // bug: arguments transposed, compiles fine
```

<details><summary>Solution</summary>

```csharp
public readonly record struct AccountId(string Value);
public readonly record struct Money(decimal Amount, string Currency) {
    public Money {
        if (Amount < 0) throw new ArgumentOutOfRangeException(nameof(Amount));
    }
}

public void Transfer(AccountId from, AccountId to, Money amount) { ... }

Transfer(new AccountId("ACC2"), new AccountId("ACC1"), new Money(100m, "USD"));
```

Value objects give type safety (can't pass `Money` where `AccountId` is expected) and centralize validation. `from`/`to` are still both `AccountId` (transposition possible) but currency/amount can no longer be mixed with account ids, and validation lives in `Money`.

</details>

---

## Problem 4: Refactor an anemic model

```csharp
public class ShoppingCart {
    public List<CartItem> Items { get; set; } = new();
    public decimal Total { get; set; }
}
public class CartService {
    public void AddItem(ShoppingCart cart, CartItem item) {
        cart.Items.Add(item);
        cart.Total += item.Price;   // can drift out of sync; anyone can set Total directly
    }
}
```

<details><summary>Solution</summary>

```csharp
public class ShoppingCart {
    private readonly List<CartItem> _items = new();
    public IReadOnlyList<CartItem> Items => _items;       // read-only view
    public decimal Total => _items.Sum(i => i.Price);     // computed — always consistent

    public void AddItem(CartItem item) {                   // behavior + invariant on the entity
        ArgumentNullException.ThrowIfNull(item);
        if (_items.Count >= 100) throw new InvalidOperationException("Cart full");
        _items.Add(item);
    }
    public void RemoveItem(CartItem item) => _items.Remove(item);
}
```

Behavior and invariants move onto the entity. `Total` is computed (can't drift). `Items` is a read-only view (callers can't bypass `AddItem`). The separate service is no longer needed for this logic.

</details>

---

## Problem 5: Replace service locator with DI

```csharp
public class OrderProcessor {
    public void Process(Order order) {
        var repo = ServiceLocator.Get<IOrderRepository>();
        var email = ServiceLocator.Get<IEmailSender>();
        repo.Save(order);
        email.Send(order.Email, "Confirmed");
    }
}
```

<details><summary>Solution</summary>

```csharp
public class OrderProcessor(IOrderRepository repo, IEmailSender email) {
    public void Process(Order order) {
        ArgumentNullException.ThrowIfNull(order);
        repo.Save(order);
        email.Send(order.Email, "Confirmed");
    }
}

// Registration
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<IEmailSender, SmtpEmailSender>();
services.AddScoped<OrderProcessor>();
```

Constructor injection (C# 12 primary constructor) makes dependencies explicit and testable — you can see what `OrderProcessor` needs from its signature, and mock both in tests. The hidden global locator is gone.

</details>

---

## Problem 6: Fix collection API design

```csharp
public class Team {
    public List<Player> Players = new();          // public mutable field exposing internals
    public List<Player> GetActivePlayers() {
        var active = Players.Where(p => p.IsActive).ToList();
        return active.Count > 0 ? active : null;   // returns null
    }
}
```

<details><summary>Solution</summary>

```csharp
public class Team {
    private readonly List<Player> _players = new();
    public IReadOnlyList<Player> Players => _players;     // read-only view, not the raw list

    public void AddPlayer(Player p) {                      // controlled mutation
        ArgumentNullException.ThrowIfNull(p);
        _players.Add(p);
    }

    public IReadOnlyList<Player> GetActivePlayers() =>
        _players.Where(p => p.IsActive).ToArray();         // never null — empty if none
}
```

Fixes: encapsulate the field, expose `IReadOnlyList<T>`, add controlled mutation, and return empty (`ToArray()`) instead of null. Now callers can't `Team.Players.Clear()` or hit a `NullReferenceException`.

</details>

---

## Problem 7: Replace magic strings/numbers

```csharp
public decimal CalculateShipping(string method, decimal weight) {
    if (method == "express") return weight * 5;
    if (method == "standard") return weight * 2;
    if (method == "free") return 0;
    throw new Exception("unknown");
}
```

<details><summary>Solution</summary>

```csharp
public enum ShippingMethod { Standard, Express, Free }

private const decimal ExpressRate = 5m;
private const decimal StandardRate = 2m;

public decimal CalculateShipping(ShippingMethod method, decimal weight) {
    ArgumentOutOfRangeException.ThrowIfNegative(weight);
    return method switch {
        ShippingMethod.Express  => weight * ExpressRate,
        ShippingMethod.Standard => weight * StandardRate,
        ShippingMethod.Free     => 0m,
        _ => throw new ArgumentOutOfRangeException(nameof(method)),
    };
}
```

Enum replaces magic strings (compile-checked, IntelliSense); named constants replace magic numbers; switch expression is exhaustive; specific exception. Typos like `"expres"` are now impossible.

</details>

---

## Problem 8: Add guard clauses with modern helpers

```csharp
public Order Create(Customer customer, string description, int quantity, decimal price) {
    if (customer == null) throw new ArgumentNullException("customer");
    if (description == null || description.Trim() == "")
        throw new ArgumentException("description required");
    if (quantity <= 0) throw new ArgumentException("quantity must be positive");
    // ... nested logic
}
```

<details><summary>Solution</summary>

```csharp
public Order Create(Customer customer, string description, int quantity, decimal price) {
    ArgumentNullException.ThrowIfNull(customer);
    ArgumentException.ThrowIfNullOrWhiteSpace(description);
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(quantity);
    ArgumentOutOfRangeException.ThrowIfNegative(price);

    // main logic at the top level — preconditions guaranteed
    return new Order(customer, description, quantity, price);
}
```

The modern helpers are concise, auto-fill the parameter name (via `CallerArgumentExpression`), and produce clear messages. Guard clauses keep the body flat.

</details>

---

## Problem 9: Make a type immutable

```csharp
public class Reservation {
    public int Id { get; set; }
    public DateTime Date { get; set; }
    public List<string> Guests { get; set; } = new();
}
// Anyone can mutate any field after creation, including the Guests list.
```

<details><summary>Solution</summary>

```csharp
public record Reservation {
    public required int Id { get; init; }
    public required DateTime Date { get; init; }
    public ImmutableArray<string> Guests { get; init; } = [];   // truly immutable member
}

var r = new Reservation { Id = 1, Date = date, Guests = ["Alice", "Bob"] };
var updated = r with { Date = newDate };   // non-destructive update; r unchanged
```

`init`-only + `required` makes fields immutable-after-construction and mandatory. Crucially, `Guests` is `ImmutableArray<string>` — using `List<string>` would be **shallow immutability** (callers could mutate the list). `with` enables non-destructive updates.

</details>

---

## Problem 10: The grand refactor

Modernize this 2015-era code to idiomatic C# 14, applying every relevant best practice.

```csharp
public class UserManager {
    public static UserManager Instance = new UserManager();
    private List<User> users = new List<User>();

    public List<User> GetUsers(string status) {
        List<User> result = new List<User>();
        for (int i = 0; i < users.Count; i++) {
            if (users[i].Status == status) {
                result.Add(users[i]);
            }
        }
        if (result.Count == 0) return null;
        return result;
    }

    public void AddUser(User u) {
        try {
            users.Add(u);
            EmailService.GetInstance().Send(u.Email, "Welcome");
        } catch (Exception ex) {
            throw ex;
        }
    }
}
```

<details><summary>Solution</summary>

```csharp
public enum UserStatus { Active, Suspended, Pending }

public record User {
    public required int Id { get; init; }
    public required string Email { get; init; }
    public UserStatus Status { get; init; } = UserStatus.Pending;
}

public class UserManager(IEmailSender email, ILogger<UserManager> logger) {
    private readonly List<User> _users = [];

    public IReadOnlyList<User> Users => _users;

    public IReadOnlyList<User> GetUsers(UserStatus status) =>
        _users.Where(u => u.Status == status).ToArray();   // LINQ, empty not null, enum not string

    public void AddUser(User user) {
        ArgumentNullException.ThrowIfNull(user);
        try {
            _users.Add(user);
            email.Send(user.Email, "Welcome");
        } catch (EmailException ex) {
            logger.LogError(ex, "Failed to send welcome email to {Email}", user.Email);
            throw;   // preserve stack; caller decides
        }
    }
}

// Registration (no singleton field, no service locator)
services.AddSingleton<UserManager>();
services.AddSingleton<IEmailSender, SmtpEmailSender>();
```

Changes applied:
- **Singleton field + service locator** → DI (constructor injection).
- **Magic string status** → `UserStatus` enum.
- **Mutable public state** (`Instance`, `users`) → encapsulated `_users`, read-only `Users` view.
- **Manual loop** → LINQ `Where`.
- **Returns null** → returns empty (`ToArray()`).
- **`throw ex;`** → `throw;` (preserve stack) + specific catch + structured logging.
- **Mutable `User`** → immutable `record` with `required`/`init`.
- **Field naming** → `_users`; collection expression `[]`.
- **Guard clause** → `ArgumentNullException.ThrowIfNull`.

</details>

---

This is the culmination: recognizing anti-patterns and reflexively refactoring to idiomatic, safe, modern C#. Once these refactors feel automatic, you've internalized the craft — beginner to expert, complete.

---

## Problem 11: Apply Open/Closed — kill the growing switch

This method must be edited for every new discount type. Refactor so new types require no edits to existing code.

```csharp
public decimal ApplyDiscount(Order o, string discountType) {
    switch (discountType) {
        case "percentage": return o.Total * 0.9m;
        case "fixed": return o.Total - 10m;
        case "none": return o.Total;
        default: throw new ArgumentException("unknown");
    }
}
```

<details><summary>Solution</summary>

```csharp
public interface IDiscount { decimal Apply(decimal total); }

public sealed class PercentageDiscount(decimal percent) : IDiscount {
    public decimal Apply(decimal total) => total * (1 - percent);
}
public sealed class FixedDiscount(decimal amount) : IDiscount {
    public decimal Apply(decimal total) => Math.Max(0, total - amount);
}
public sealed class NoDiscount : IDiscount {
    public decimal Apply(decimal total) => total;
}

public decimal ApplyDiscount(Order o, IDiscount discount) => discount.Apply(o.Total);
// New discount type = new class implementing IDiscount. ApplyDiscount never changes.
```

Open for extension (add a class), closed for modification (existing code untouched). Note: if the set of discounts is genuinely small and stable, a `switch` expression is also acceptable — apply OCP where variation is real.

</details>

---

## Problem 12: Decorator for cross-cutting concerns

Add caching and logging to an `IProductRepository` without modifying the SQL implementation.

<details><summary>Solution</summary>

```csharp
public interface IProductRepository { Product? Get(int id); }

public sealed class SqlProductRepository : IProductRepository {
    public Product? Get(int id) => /* hit the database */;
}

public sealed class CachingProductRepository(IProductRepository inner, IMemoryCache cache)
    : IProductRepository {
    public Product? Get(int id) => cache.GetOrCreate(id, _ => inner.Get(id));
}

public sealed class LoggingProductRepository(IProductRepository inner, ILogger<IProductRepository> log)
    : IProductRepository {
    public Product? Get(int id) {
        log.LogDebug("Fetching product {Id}", id);
        return inner.Get(id);
    }
}

// Compose: log every call, cache misses hit SQL
IProductRepository repo =
    new LoggingProductRepository(
        new CachingProductRepository(
            new SqlProductRepository(), cache), logger);
```

Each decorator adds one concern (SRP) without touching the others. Order matters: this logs every call and caches around SQL. Swap the order to log only cache misses.

</details>

---

## Problem 13: Replace exception-as-control-flow with a result type

This login uses exceptions for the (very common) failure case. Refactor.

```csharp
public User Login(string username, string password) {
    var user = _repo.Find(username);
    if (user == null) throw new Exception("User not found");
    if (!user.CheckPassword(password)) throw new Exception("Wrong password");
    return user;
}
// Caller:
try { var u = Login(name, pwd); ... } catch (Exception ex) { ShowError(ex.Message); }
```

<details><summary>Solution</summary>

```csharp
public readonly record struct Result<T>(bool Success, T? Value, string? Error) {
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}

public Result<User> Login(string username, string password) {
    var user = _repo.Find(username);
    if (user is null) return Result<User>.Fail("Invalid credentials");
    if (!user.CheckPassword(password)) return Result<User>.Fail("Invalid credentials");
    return Result<User>.Ok(user);
}

// Caller — no try/catch for an expected outcome
var result = Login(name, pwd);
if (!result.Success) { ShowError(result.Error!); return; }
UseUser(result.Value!);
```

Login failure is **expected**, not exceptional — a result type makes it part of the signature, is far cheaper than throwing, and forces the caller to handle it. (Note: returning a generic "Invalid credentials" avoids leaking whether the username exists — a security best practice.)

</details>

---

## Problem 14: Fix broken equality

This type is used as a dictionary key but lookups fail. Find and fix the bug.

```csharp
public class Coordinate {
    public int Lat { get; set; }
    public int Lng { get; set; }
    public override bool Equals(object? o) => o is Coordinate c && c.Lat == Lat && c.Lng == Lng;
}

var map = new Dictionary<Coordinate, string>();
map[new Coordinate { Lat = 1, Lng = 2 }] = "home";
var found = map[new Coordinate { Lat = 1, Lng = 2 }];   // KeyNotFoundException!
```

<details><summary>Solution</summary>

Two bugs: (1) `GetHashCode` isn't overridden (so equal coordinates hash differently → different buckets → lookup misses); (2) mutable properties on a hash key are dangerous. Fix both — easiest is a `record struct`:

```csharp
public readonly record struct Coordinate(int Lat, int Lng);
// Synthesizes consistent Equals + GetHashCode + IEquatable; immutable.

var map = new Dictionary<Coordinate, string> {
    [new Coordinate(1, 2)] = "home"
};
var found = map[new Coordinate(1, 2)];   // "home" ✓
```

If a class is required, hand-write it correctly:

```csharp
public sealed class Coordinate(int lat, int lng) : IEquatable<Coordinate> {
    public int Lat { get; } = lat;   // immutable
    public int Lng { get; } = lng;
    public bool Equals(Coordinate? o) => o is not null && o.Lat == Lat && o.Lng == Lng;
    public override bool Equals(object? o) => Equals(o as Coordinate);
    public override int GetHashCode() => HashCode.Combine(Lat, Lng);   // consistent with Equals
}
```

The original overrode `Equals` without `GetHashCode` (the compiler warns CS0659) and used mutable properties — the two classic equality bugs.

</details>

---

## Problem 15: Refactor a service-locator class to constructor injection

```csharp
public class InvoiceService {
    public void Generate(int orderId) {
        var orders = ServiceLocator.Resolve<IOrderRepository>();
        var pdf = ServiceLocator.Resolve<IPdfGenerator>();
        var order = orders.Get(orderId);
        pdf.Create(order);
    }
}
```

<details><summary>Solution</summary>

```csharp
public class InvoiceService(IOrderRepository orders, IPdfGenerator pdf) {
    public void Generate(int orderId) {
        var order = orders.Get(orderId)
            ?? throw new InvalidOperationException($"Order {orderId} not found");
        pdf.Create(order);
    }
}

// Composition root
services.AddScoped<IOrderRepository, SqlOrderRepository>();
services.AddScoped<IPdfGenerator, PdfGenerator>();
services.AddScoped<InvoiceService>();
```

Dependencies are now explicit in the constructor (you can see what `InvoiceService` needs from its signature), the hidden global locator is gone, and it's trivially testable — inject mocks for `IOrderRepository`/`IPdfGenerator`. This is DIP + DI applied. See [12-DependencyInjection.md](12-DependencyInjection.md).

</details>

---

These problems exercise the design layer: applying OCP, composing with decorators, choosing results over exceptions, getting equality right, and inverting dependencies. Together with problems 1–10, you can now recognize *and reflexively fix* the full spectrum of C# design and idiom issues — the mark of an expert.

→ Back to [Chapter 17 README](README.md) · [Book Home](../README.md)
