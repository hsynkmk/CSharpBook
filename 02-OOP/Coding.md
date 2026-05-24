# Chapter 02 — Coding Problems

> ~15 problems on classes, polymorphism, interfaces, and the OOP patterns interviewers love to trap you with. Try each before opening the solution.

---

## Problem 1 — The famous `override` vs `new` prediction

```csharp
class A {
    public virtual string Tag => "A";
    public string Read() => Tag;
}

class B : A {
    public override string Tag => "B";
}

class C : A {
    public new string Tag => "C";
}

A a = new A();
A b = new B();
A c = new C();

Console.WriteLine($"{a.Tag} {b.Tag} {c.Tag}");
Console.WriteLine($"{a.Read()} {b.Read()} {c.Read()}");
Console.WriteLine($"{((C)c).Tag}");
```

**Q: Predict the output and explain each line.**

<details><summary>Answer</summary>

```
A B A
A B A
C
```

- Line 1: `a.Tag` → A (no surprises). `b.Tag` → B because `override` participates in the vtable; dispatch by runtime type. `c.Tag` → A because `new` only hides; the static type is `A`, so the base property is selected.
- Line 2: `Read()` lives on `A` and reads `Tag` — same vtable dispatch, same results.
- Line 3: cast to `C` makes the static type `C`, so `new`-hidden `Tag` is now visible → `C`.

Mnemonic: **`override` is dynamic; `new` is static.**

</details>

---

## Problem 2 — Constructor order in a hierarchy

```csharp
class A {
    int ax = Init("A field");
    public A() => Console.WriteLine("A ctor");
    static int Init(string s) { Console.WriteLine(s); return 0; }
}

class B : A {
    int bx = Init("B field");
    public B() => Console.WriteLine("B ctor");
    static int Init(string s) { Console.WriteLine(s); return 0; }
}

new B();
```

**Q: Predict the output.**

<details><summary>Answer</summary>

```
A field
A ctor
B field
B ctor
```

Construction always runs base-to-derived. Within a class, **field initializers run before the constructor body**.

</details>

---

## Problem 3 — The virtual-from-constructor trap

```csharp
class Animal {
    public Animal() { Speak(); }
    public virtual void Speak() => Console.WriteLine("animal");
}

class Dog : Animal {
    string sound = "woof";
    public override void Speak() => Console.WriteLine(sound ?? "<null>");
}

new Dog();
```

**Q: What prints, and why is this a famous bug?**

<details><summary>Answer</summary>

Prints `<null>`. The base constructor calls the virtual `Speak`, which dispatches to `Dog.Speak`. But the derived field initializer hasn't run yet — `sound` is still its default (`null`).

The fix is "**don't call virtual methods from constructors**." If you must, treat the call as if you know nothing about derived state.

</details>

---

## Problem 4 — Polymorphism vs hiding through an interface

```csharp
interface IGreet { string Hello(); }

class English : IGreet { public string Hello() => "Hello"; }

class American : English {
    public new string Hello() => "Howdy";
}

IGreet g = new American();
English e = new American();
American a = new American();

Console.WriteLine($"{g.Hello()} {e.Hello()} {a.Hello()}");
```

**Q: Predict and explain.**

<details><summary>Answer</summary>

```
Hello Hello Howdy
```

`American.Hello` uses `new`, not `override`. `English.Hello` implicitly implements `IGreet.Hello`. When `American` hides with `new`, it doesn't replace the interface slot — `English.Hello` still answers for `IGreet`. Through the `English` reference, the static type's `Hello` is selected.

Fix to override and re-bind the interface slot: declare `English.Hello` as `virtual`, then `American.Hello` as `override`.

</details>

---

## Problem 5 — Spot the encapsulation leak

```csharp
public class ShoppingCart {
    public List<string> Items { get; } = new();
    public decimal Total { get; private set; }

    public void Add(string sku, decimal price) {
        Items.Add(sku);
        Total += price;
    }
}
```

**Q: Find the bug and propose a fix.**

<details><summary>Answer</summary>

`Items` is exposed as `List<string>`. A consumer can do:

```csharp
cart.Items.Clear();
// Total is now wrong — invariant broken
```

The encapsulation contract ("Total reflects the items in the cart") leaks through the public mutable list.

**Fix**: expose as `IReadOnlyList<string>`:

```csharp
private readonly List<string> _items = new();
public IReadOnlyList<string> Items => _items;
public decimal Total { get; private set; }

public void Add(string sku, decimal price) {
    _items.Add(sku);
    Total += price;
}
```

Now mutation only goes through `Add`, and `Total` stays in sync.

</details>

---

## Problem 6 — Implement `IDisposable` correctly

Implement a class `TempFile` that creates a temporary file on construction and deletes it on `Dispose`. Make it safe to call `Dispose` multiple times.

<details><summary>Solution</summary>

```csharp
public sealed class TempFile : IDisposable {
    private string? _path;

    public TempFile() {
        _path = Path.GetTempFileName();
    }

    public string Path => _path ?? throw new ObjectDisposedException(nameof(TempFile));

    public void Dispose() {
        if (_path is null) return;       // idempotent
        try { File.Delete(_path); }
        catch { /* swallow — best effort */ }
        _path = null;
    }
}
```

Notes:
- `sealed` because we don't expect subclasses; simplifies the dispose pattern.
- Setting `_path` to null after delete makes the second call a no-op.
- No finalizer needed because `string` is managed; the file path doesn't own an OS handle directly.
- Throwing `ObjectDisposedException` on access after dispose is the convention.

A more rigorous version would suppress finalize and add a finalizer if it held unmanaged resources — covered in Chapter 09.

</details>

---

## Problem 7 — Refactor an "is-a"/"has-a" mistake

```csharp
public class Stack<T> : List<T> {
    public void Push(T item) => Add(item);
    public T Pop() {
        var v = this[Count - 1];
        RemoveAt(Count - 1);
        return v;
    }
}
```

**Q: What's wrong with this design? Refactor it.**

<details><summary>Answer</summary>

A `Stack<T>` is **not a** `List<T>`. Inheriting exposes `Insert`, `RemoveAt`, indexer access, and every other list operation that breaks LIFO semantics:

```csharp
var s = new Stack<int>();
s.Push(1);
s.Push(2);
s.Insert(0, 99);     // legal — bypasses Push
```

The right relationship is composition (has-a):

```csharp
public class Stack<T> {
    private readonly List<T> _items = new();
    public int Count => _items.Count;
    public void Push(T item) => _items.Add(item);
    public T Pop() {
        if (_items.Count == 0) throw new InvalidOperationException("empty");
        var v = _items[^1];
        _items.RemoveAt(_items.Count - 1);
        return v;
    }
    public T Peek() => _items[^1];
}
```

LSP also applies: a `Stack<T>` shouldn't satisfy code expecting a `List<T>` to support `Insert`. Composition keeps the contracts honest.

</details>

---

## Problem 8 — Interface design: capability vs entity

You're designing a system where some objects can be serialized to JSON, some can be sent over the network, and some can be cached. Each capability is independent.

**Q: Sketch the interfaces and explain why you wouldn't put all three methods in one giant `IObject` interface.**

<details><summary>Solution</summary>

```csharp
public interface IJsonSerializable {
    string ToJson();
}

public interface INetworkSendable {
    Task SendAsync(Stream destination, CancellationToken ct);
}

public interface ICacheable {
    string CacheKey { get; }
    TimeSpan? Expiry { get; }
}
```

Each interface represents one **capability**. A class can implement any subset.

```csharp
public class Receipt : IJsonSerializable, ICacheable {
    public string ToJson() => /* ... */;
    public string CacheKey => $"receipt-{Id}";
    public TimeSpan? Expiry => TimeSpan.FromHours(1);
}
```

Why not one big interface?

- **Interface segregation principle**: classes shouldn't be forced to implement methods they don't need.
- **Test doubles** are easier — mock just the capability you care about.
- **Future-proof**: adding a new capability is a new interface, not a breaking change.

</details>

---

## Problem 9 — Abstract base + interface combo

Design a shape hierarchy: shapes have an `Area`, can be drawn, and some are "regular" (all sides equal). Use both an interface and an abstract base class.

<details><summary>Solution</summary>

```csharp
public interface IDrawable {
    void DrawTo(Canvas c);
}

public abstract class Shape : IDrawable {
    public abstract double Area { get; }
    public abstract void DrawTo(Canvas c);
    public string Describe() => $"{GetType().Name} area={Area:F2}";
}

public interface IRegular {
    int Sides { get; }
    double SideLength { get; }
}

public class Square(double side) : Shape, IRegular {
    public int Sides => 4;
    public double SideLength => side;
    public override double Area => side * side;
    public override void DrawTo(Canvas c) => c.DrawSquare(side);
}

public class Circle(double r) : Shape {
    public override double Area => Math.PI * r * r;
    public override void DrawTo(Canvas c) => c.DrawCircle(r);
}
```

Decisions:
- `IDrawable` and `IRegular` are capabilities — interfaces. `IRegular` doesn't fit every shape; it lives on the side.
- `Shape` is shared **implementation** (the `Describe` method) and shared contract (abstract `Area`) — an abstract class. It also implements `IDrawable` to centralize that capability.
- Primary constructors keep the boilerplate down without bringing record semantics.

</details>

---

## Problem 10 — Equals override pitfall

```csharp
public class Person {
    public string Name { get; }
    public Person(string name) { Name = name; }
    public override bool Equals(object? obj) =>
        obj is Person p && p.Name == Name;
}

var a = new Person("Alice");
var b = new Person("Alice");
var dict = new Dictionary<Person, int>();
dict[a] = 1;
Console.WriteLine(dict[b]);
```

**Q: What's wrong?**

<details><summary>Answer</summary>

`KeyNotFoundException`. Overriding `Equals` without overriding `GetHashCode` breaks the contract that "equal objects have equal hash codes." `Dictionary` hashes the key first, lands in the wrong bucket, and never finds the match.

Fix:

```csharp
public override int GetHashCode() => Name.GetHashCode();
```

For value-based types prefer `record`, which generates both `Equals` and `GetHashCode` from the declared members:

```csharp
public record Person(string Name);
```

Equality is covered properly in [§01-MustMaster/09-EqualsHashCode.md](../../01-MustMaster/09-EqualsHashCode.md).

</details>

---

## Problem 11 — Sealed for performance

```csharp
public class Money {
    public decimal Amount { get; }
    public string Currency { get; }
    public Money(decimal a, string c) { Amount = a; Currency = c; }
    public virtual bool IsZero => Amount == 0;
}

public class Dollars : Money {
    public Dollars(decimal a) : base(a, "USD") { }
    public override bool IsZero => Amount == 0;
}
```

**Q: Apply `sealed` where it helps. Explain the perf consequence.**

<details><summary>Answer</summary>

```csharp
public sealed class Dollars : Money {
    public Dollars(decimal a) : base(a, "USD") { }
    public override bool IsZero => Amount == 0;
}
```

Or, if `Money` itself shouldn't be extended:

```csharp
public sealed class Money {
    // remove `virtual` — no point
}
```

Sealing tells the JIT no derived type exists for that branch. Virtual calls on a `Dollars` reference can be **devirtualized** — turned into direct calls and potentially inlined. For hot leaf types in a perf-critical path, sealing is one of the cheapest wins available.

(`sealed override bool IsZero` on `Dollars.IsZero` works similarly when sealing a single method but not the whole class.)

</details>

---

## Problem 12 — Implement a builder using nested types

Design a fluent builder for an `HttpRequest` value. Use a nested `Builder` class so it has privileged access.

<details><summary>Solution</summary>

```csharp
public sealed class HttpRequest {
    public Uri Url { get; }
    public string Method { get; }
    public IReadOnlyDictionary<string, string> Headers { get; }
    public string? Body { get; }

    private HttpRequest(Builder b) {
        Url = b._url ?? throw new InvalidOperationException("Url required");
        Method = b._method;
        Headers = b._headers;
        Body = b._body;
    }

    public sealed class Builder {
        internal Uri? _url;
        internal string _method = "GET";
        internal readonly Dictionary<string, string> _headers = new();
        internal string? _body;

        public Builder Url(string u) { _url = new Uri(u); return this; }
        public Builder Method(string m) { _method = m; return this; }
        public Builder Header(string k, string v) { _headers[k] = v; return this; }
        public Builder Body(string b) { _body = b; return this; }
        public HttpRequest Build() => new(this);
    }
}

var req = new HttpRequest.Builder()
    .Url("https://example.com/api")
    .Method("POST")
    .Header("Content-Type", "application/json")
    .Body("""{"hello":"world"}""")
    .Build();
```

Why nested:
- `HttpRequest` is immutable; only the `Builder` may produce one (private constructor).
- The `Builder` lives in the same namespace as the thing it builds.
- The nested type can use `HttpRequest`'s private constructor.

</details>

---

## Problem 13 — DI service using a primary constructor

Refactor this verbose service into one using a primary constructor (C# 12).

```csharp
public class OrderService {
    private readonly IDbContext _db;
    private readonly ILogger<OrderService> _log;
    private readonly IPriceCalculator _calc;

    public OrderService(IDbContext db, ILogger<OrderService> log, IPriceCalculator calc) {
        _db = db;
        _log = log;
        _calc = calc;
    }

    public Order Place(Cart c) {
        _log.LogInformation("placing {N}", c.Items.Count);
        var o = new Order(c, _calc.Total(c));
        _db.Orders.Add(o);
        _db.SaveChanges();
        return o;
    }
}
```

<details><summary>Solution</summary>

```csharp
public class OrderService(
    IDbContext db,
    ILogger<OrderService> log,
    IPriceCalculator calc) {

    public Order Place(Cart c) {
        log.LogInformation("placing {N}", c.Items.Count);
        var o = new Order(c, calc.Total(c));
        db.Orders.Add(o);
        db.SaveChanges();
        return o;
    }
}
```

The compiler generates the private capture fields automatically. Note we lose the leading underscore convention — a stylistic price.

If you want `ArgumentNullException.ThrowIfNull` validation, you can hoist it into a field initializer:

```csharp
public class OrderService(
    IDbContext db,
    ILogger<OrderService> log,
    IPriceCalculator calc) {

    private readonly IDbContext _db = db ?? throw new ArgumentNullException(nameof(db));
    // ... etc
}
```

But at that point the savings disappear; a regular constructor is just as clear.

</details>

---

## Problem 14 — Default interface method override

```csharp
interface ILogger {
    void Log(string msg);
    void Warn(string msg) => Log("WARN: " + msg);
}

class ConsoleLogger : ILogger {
    public void Log(string msg) => Console.WriteLine(msg);
}

class StrictLogger : ConsoleLogger, ILogger {
    void ILogger.Warn(string msg) => Log("STRICT-WARN: " + msg);
}

ILogger a = new ConsoleLogger();
ILogger b = new StrictLogger();
a.Warn("oops");
b.Warn("oops");
```

**Q: Predict the output.**

<details><summary>Answer</summary>

```
WARN: oops
STRICT-WARN: oops
```

`ConsoleLogger` doesn't override `Warn`, so it inherits the default — calling it produces `"WARN: oops"`. `StrictLogger` explicitly re-implements `ILogger.Warn`. Interface dispatch picks the most-derived implementation in the interface map — that's `StrictLogger`'s.

If you removed `, ILogger` from `StrictLogger` (left only the `ConsoleLogger` base), the explicit interface implementation wouldn't be legal — explicit implementation requires re-declaring the interface on the class.

</details>

---

## Problem 15 — Code smell hunt

Find every OOP smell in this snippet. List them; don't refactor.

```csharp
public class Manager {
    public List<Employee> Employees;
    public Database Db = new();
    public Logger Log;

    public void HireFireRaisePromote(Employee e, string action) {
        if (Log == null) Log = new Logger();
        Log.Write(action);
        if (action == "hire")   { Employees.Add(e); Db.Insert(e); }
        if (action == "fire")   { Employees.Remove(e); Db.Delete(e); }
        if (action == "raise")  { e.Salary *= 1.1m; Db.Update(e); }
        if (action == "promo")  { e.Title = "Senior " + e.Title; Db.Update(e); }
    }
}
```

<details><summary>Answer</summary>

A non-exhaustive list:

1. **Public mutable fields** (`Employees`, `Db`, `Log`) instead of properties — encapsulation leak.
2. **Field of type `List<>` exposed publicly** — callers can scramble state.
3. **No dependency injection** — `Db = new()` and lazy `new Logger()` hard-code dependencies, untestable.
4. **God method** — `HireFireRaisePromote` violates SRP. One method, four responsibilities, picked by a string.
5. **Stringly-typed dispatch** — `action` should be an enum or separate methods.
6. **Missing else / fallthrough risk** — multiple `if` statements; if `action == "raise"` and `action == "promo"` both matched, both would run. Use `switch` or `else if`.
7. **No validation** — what if `e` is null? What if `action` is "delete-all"?
8. **No transaction boundary** — adding to local list and DB separately can desync on error.
9. **No constructor** — `Employees` and `Log` start null; first use will NRE.
10. **Salary mutation via public setter** — invariants (positive, currency) not enforced.
11. **Title mutation by string concat** — promoting twice gives `"Senior Senior Engineer"`.
12. **No interface boundary** — `Database` and `Logger` are concrete; can't swap for tests.

Refactoring would split this into a `Manager` that depends on `IEmployeeRepository`, `ILogger`, and explicit `Hire`, `Fire`, `Raise(decimal pct)`, `Promote` methods, each enforcing its invariants.

</details>

---

→ Back to: [README.md](README.md)
