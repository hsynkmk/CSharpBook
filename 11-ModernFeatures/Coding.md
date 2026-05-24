# Chapter 11 — Modern Features — Coding Problems

Refactor and modernize. Each problem starts with legacy code; rewrite for C# 14 / .NET 10.

---

## Problem 1: Modernize a DTO

Convert this C# 7 class to an idiomatic C# 14 record:

```csharp
public class Person {
    public string FirstName { get; set; }
    public string LastName { get; set; }
    public int Age { get; set; }

    public Person(string firstName, string lastName, int age) {
        FirstName = firstName;
        LastName = lastName;
        Age = age;
    }

    public override bool Equals(object obj) {
        if (!(obj is Person other)) return false;
        return FirstName == other.FirstName && LastName == other.LastName && Age == other.Age;
    }
    public override int GetHashCode() => HashCode.Combine(FirstName, LastName, Age);
    public override string ToString() => $"Person {{ FirstName = {FirstName}, LastName = {LastName}, Age = {Age} }}";
}
```

<details><summary>Solution</summary>

```csharp
public record Person(string FirstName, string LastName, int Age);
```

That's it. Records auto-generate Equals, GetHashCode, ToString, Deconstruct, ==, !=, copy constructor.

Add validation if needed:

```csharp
public record Person(string FirstName, string LastName, int Age) {
    public string FullName => $"{FirstName} {LastName}";
    public Person {
        if (Age < 0) throw new ArgumentOutOfRangeException(nameof(Age));
    }
}
```

</details>

---

## Problem 2: Replace manual backing field

Modernize this using C# 14's `field` keyword:

```csharp
public class Settings {
    private string _name = "";
    public string Name {
        get => _name;
        set => _name = (value ?? "").Trim();
    }
}
```

<details><summary>Solution</summary>

```csharp
public class Settings {
    public string Name {
        get;
        set => field = (value ?? "").Trim();
    }
}
```

Initial value can be assigned via property initializer if needed:

```csharp
public string Name { get; set => field = (value ?? "").Trim(); } = "";
```

</details>

---

## Problem 3: Convert a method to use collection expressions and `params ReadOnlySpan`

Refactor:

```csharp
public static int Sum(params int[] values) {
    int total = 0;
    foreach (var v in values) total += v;
    return total;
}
```

Make it zero-allocation when called inline.

<details><summary>Solution</summary>

```csharp
public static int Sum(params ReadOnlySpan<int> values) {
    int total = 0;
    foreach (var v in values) total += v;
    return total;
}

// Callers
Sum(1, 2, 3);                   // stack-allocated span
Sum([1, 2, 3]);                 // collection expression
int[] arr = [4, 5, 6];
Sum(arr);                       // array converts to ReadOnlySpan
```

The `Sum(1, 2, 3)` form is now alloc-free.

</details>

---

## Problem 4: Null-conditional assignment

Refactor:

```csharp
if (logger is not null) {
    logger.LastMessage = msg;
    logger.MessageCount = logger.MessageCount + 1;
}

if (config?.Cache is not null) {
    config.Cache.LastAccess = DateTime.UtcNow;
}
```

<details><summary>Solution</summary>

```csharp
logger?.LastMessage = msg;
logger?.MessageCount += 1;

config?.Cache?.LastAccess = DateTime.UtcNow;
```

Three lines becomes three expressions. Watch: if `logger` is null, neither assignment runs.

</details>

---

## Problem 5: Modernize a switch statement

Convert to switch expression + patterns (C# 8+):

```csharp
public string Describe(object obj) {
    if (obj is null) return "null";
    if (obj is int i) {
        if (i == 0) return "zero";
        if (i > 0) return "positive";
        return "negative";
    }
    if (obj is string s) return $"string of length {s.Length}";
    if (obj is int[] arr) {
        if (arr.Length == 0) return "empty array";
        if (arr.Length == 1) return $"single [{arr[0]}]";
        return $"array of {arr.Length}";
    }
    return "unknown";
}
```

<details><summary>Solution</summary>

```csharp
public string Describe(object obj) => obj switch {
    null                 => "null",
    int and 0            => "zero",
    int n and > 0        => "positive",
    int                  => "negative",
    string s             => $"string of length {s.Length}",
    int[] []             => "empty array",
    int[] [var single]   => $"single [{single}]",
    int[] arr            => $"array of {arr.Length}",
    _                    => "unknown",
};
```

Uses constant patterns, type patterns, relational patterns, list patterns. Reads top-to-bottom as a decision table.

</details>

---

## Problem 6: Convert a verbose object initializer

Modernize:

```csharp
public class Order {
    public int Id;
    public string Customer = "";
    public DateTime Date;
    public List<string> Items = new List<string>();
}

var o = new Order();
o.Id = 1;
o.Customer = "Alice";
o.Date = DateTime.UtcNow;
o.Items.Add("widget");
o.Items.Add("gadget");
```

<details><summary>Solution</summary>

```csharp
public record Order(int Id, string Customer, DateTime Date) {
    public List<string> Items { get; init; } = [];
}

var o = new Order(1, "Alice", DateTime.UtcNow) {
    Items = ["widget", "gadget"],
};
```

Or with required:

```csharp
public class Order {
    public required int Id { get; init; }
    public required string Customer { get; init; }
    public required DateTime Date { get; init; }
    public List<string> Items { get; init; } = [];
}

var o = new Order {
    Id = 1, Customer = "Alice", Date = DateTime.UtcNow,
    Items = ["widget", "gadget"],
};
```

</details>

---

## Problem 7: Use `Lock` instead of `object`

Modernize:

```csharp
public class Counter {
    private readonly object _gate = new();
    private int _value;
    public void Increment() {
        lock (_gate) _value++;
    }
}
```

<details><summary>Solution</summary>

```csharp
public class Counter {
    private readonly Lock _gate = new();   // System.Threading.Lock
    private int _value;
    public void Increment() {
        lock (_gate) _value++;
    }
}
```

The compiler now emits calls to `Lock.Enter()` / `Lock.Exit()` (faster than `Monitor.Enter/Exit`).

</details>

---

## Problem 8: Extension property (C# 14)

Add an `IsValidEmail` extension property to `string`:

<details><summary>Solution</summary>

```csharp
public static partial class StringExtensions {
    extension(string s) {
        public bool IsValidEmail => s.Contains('@') && s.IndexOf('@') < s.LastIndexOf('.');
    }
}

// Usage
if (email.IsValidEmail) { ... }
```

Reads as a real property. Pre-C# 14 required `.IsValidEmail()` method call.

</details>

---

## Problem 9: File-based app

Write a single-file `.cs` script that reads command-line args and prints a greeting in JSON. Use `dotnet run hello.cs`.

<details><summary>Solution</summary>

```csharp
// hello.cs
#:package System.Text.Json

using System.Text.Json;

var name = args.Length > 0 ? args[0] : "World";
var payload = new { greeting = $"Hello, {name}!", timestamp = DateTime.UtcNow };
Console.WriteLine(JsonSerializer.Serialize(payload));
```

Run:
```bash
dotnet run hello.cs -- Alice
```

</details>

---

## Problem 10: Migrate from string concatenation to UTF-8 literal

Hot path that sends a constant HTTP header. Avoid allocation:

```csharp
async Task SendAsync(Stream s) {
    string header = "Content-Type: application/json\r\n";
    byte[] bytes = Encoding.UTF8.GetBytes(header);
    await s.WriteAsync(bytes);
}
```

<details><summary>Solution</summary>

```csharp
private static ReadOnlySpan<byte> Header => "Content-Type: application/json\r\n"u8;

async Task SendAsync(Stream s) {
    await s.WriteAsync(Header.ToArray());   // or use Memory<byte>
}
```

The `u8` literal is compiled into assembly metadata as UTF-8 bytes. No runtime encoding work. For full alloc-free path, use `Memory<byte>` field:

```csharp
private static readonly ReadOnlyMemory<byte> HeaderMem = "Content-Type: application/json\r\n"u8.ToArray();

await s.WriteAsync(HeaderMem);
```

</details>

---

## Problem 11: Modernize a class with target-typed new and collection expressions

```csharp
private Dictionary<string, List<int>> _data = new Dictionary<string, List<int>>();

public void AddDefault() {
    _data["x"] = new List<int> { 1, 2, 3 };
    _data["y"] = new List<int>();
}
```

<details><summary>Solution</summary>

```csharp
private Dictionary<string, List<int>> _data = new();   // target-typed new

public void AddDefault() {
    _data["x"] = [1, 2, 3];
    _data["y"] = [];
}
```

Or with collection expression for the dict too (C# 13+):

```csharp
private Dictionary<string, List<int>> _data = new() {
    ["x"] = [1, 2, 3],
    ["y"] = [],
};
```

</details>

---

## Problem 12: Convert an immutable update pattern

Pre-record code:

```csharp
public class Address {
    public string Street { get; }
    public string City { get; }
    public Address(string street, string city) { Street = street; City = city; }
    public Address WithCity(string newCity) => new(Street, newCity);
}
```

<details><summary>Solution</summary>

```csharp
public record Address(string Street, string City);

// Usage
var a = new Address("123 Main", "Boston");
var b = a with { City = "NYC" };   // immutable update via with-expression
```

Record's `with` expression replaces all the `WithX` boilerplate. Auto-generates copy ctor and clone.

</details>

---

## Problem 13: Use raw string + interpolation for a JSON template

Build a JSON request body cleanly:

<details><summary>Solution</summary>

```csharp
string MakeRequest(string user, int age) => $$"""
    {
      "user": "{{user}}",
      "age": {{age}},
      "active": true,
      "roles": ["admin", "user"]
    }
    """;
```

`$$"""..."""` uses `{{` for interpolation (so single `{` doesn't conflict with JSON). Raw strings preserve formatting without escaping quotes.

</details>

---

## Problem 14: Generic math operator

Implement a generic `Sum<T>` using C# 11 generic math:

<details><summary>Solution</summary>

```csharp
using System.Numerics;

public static T Sum<T>(ReadOnlySpan<T> values) where T : INumber<T> {
    T total = T.Zero;
    foreach (var v in values) total += v;
    return total;
}

// Works for any numeric type
int i = Sum<int>([1, 2, 3]);          // 6
double d = Sum<double>([1.5, 2.5]);   // 4.0
decimal m = Sum<decimal>([1m, 2m, 3m]); // 6m
```

`INumber<T>` (and friends like `IAdditionOperators<T,T,T>`) let you write polymorphic numeric code without `dynamic` or massive overload sets.

</details>

---

## Problem 15: Combine everything — modern API

Rewrite this 2017-era code in idiomatic C# 14:

```csharp
public class UserService {
    private readonly object _lock = new object();
    private readonly Dictionary<int, User> _users = new Dictionary<int, User>();
    private readonly ILogger _logger;

    public UserService(ILogger logger) {
        this._logger = logger;
    }

    public User Get(int id) {
        lock (_lock) {
            if (_users.ContainsKey(id)) return _users[id];
            return null;
        }
    }

    public void Add(User u) {
        if (u == null) throw new ArgumentNullException("u");
        lock (_lock) {
            if (!_users.ContainsKey(u.Id)) {
                _users.Add(u.Id, u);
                _logger.Log("added user " + u.Id);
            }
        }
    }
}

public class User {
    public int Id { get; set; }
    public string Name { get; set; }
}
```

<details><summary>Solution</summary>

```csharp
public record User(int Id, string Name);

public class UserService(ILogger logger) {
    private readonly Lock _lock = new();
    private readonly Dictionary<int, User> _users = [];

    public User? Get(int id) {
        lock (_lock) {
            return _users.TryGetValue(id, out var u) ? u : null;
        }
    }

    public void Add(User u) {
        ArgumentNullException.ThrowIfNull(u);
        lock (_lock) {
            if (_users.TryAdd(u.Id, u)) {
                logger.Log($"added user {u.Id}");
            }
        }
    }
}
```

What changed:
- `User` → record (value-equal, immutable, less code).
- Primary constructor on `UserService` — no field declaration for `logger`.
- `Lock` type instead of `object`.
- Collection expressions: `new Dictionary<int, User>()` → `[]`.
- Nullable annotation: `User?` return.
- `ArgumentNullException.ThrowIfNull` — modern guard.
- `TryGetValue` / `TryAdd` — avoid double lookups.
- String interpolation — `$"..."`.

</details>

---

These problems compress two decades of C# evolution. Once these refactors feel natural, you've internalized the modern idiom.

→ Back to [Chapter 11 README](README.md). Next chapter: [Chapter 12 — Reflection](../12-Reflection/README.md).
