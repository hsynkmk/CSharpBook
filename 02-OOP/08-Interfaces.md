# Interfaces

## What it is

An **interface** declares a contract — a set of members a type promises to provide. Any class, struct, or record can implement multiple interfaces. Code written against an interface is decoupled from concrete implementations.

```csharp
public interface IShape {
    double Area();
    double Perimeter();
}

public class Circle : IShape {
    public double Radius { get; init; }
    public double Area() => Math.PI * Radius * Radius;
    public double Perimeter() => 2 * Math.PI * Radius;
}

IShape s = new Circle { Radius = 5 };
Console.WriteLine(s.Area());
```

Convention: interface names start with **`I`** — `IDisposable`, `IEnumerable<T>`, `IComparable`. Almost universally followed.

---

## Why they exist

Interfaces let you:
- **Decouple producers from consumers** — `IRepository` lets the controller not know whether storage is SQL or memory.
- **Mock for testing** — depend on `IClock` in production, supply a `FakeClock` in tests.
- **Mix in capabilities** — implement `IDisposable` to participate in `using`, `IComparable<T>` for sorting.
- **Get polymorphism without inheritance** — multiple interfaces, single inheritance.

---

## Declaring and implementing

```csharp
public interface ILogger {
    void Log(string message);
    void Log(string message, Exception ex);   // overload
    LogLevel Level { get; set; }              // property
    event EventHandler<string>? OnLog;        // event
    string this[int index] { get; }            // indexer
}

public class ConsoleLogger : ILogger {
    public LogLevel Level { get; set; }
    public event EventHandler<string>? OnLog;

    public void Log(string message) {
        Console.WriteLine(message);
        OnLog?.Invoke(this, message);
    }
    public void Log(string message, Exception ex) =>
        Log($"{message}: {ex.Message}");
    public string this[int index] => $"entry-{index}";
}
```

Members of an interface:
- Methods, properties, events, indexers — yes.
- Constants (since C# 8) — yes.
- Fields — **no** (instance fields can't be in interfaces; static fields can since C# 8).

By default, every interface member is `public abstract` (you don't write the modifiers).

---

## Multiple implementation

A class can implement many interfaces:

```csharp
public class FileLogger : ILogger, IDisposable {
    private readonly StreamWriter _writer;
    public FileLogger(string path) { _writer = new StreamWriter(path, append: true); }

    public LogLevel Level { get; set; }
    public event EventHandler<string>? OnLog;
    public void Log(string message) { _writer.WriteLine(message); OnLog?.Invoke(this, message); }
    public void Log(string message, Exception ex) => Log($"{message}: {ex}");
    public string this[int i] => "...";

    public void Dispose() => _writer.Dispose();
}
```

Multiple-interface implementation is the way C# offers "mixins" — no diamond-problem ambiguity, because each interface is just a contract, not an implementation.

---

## Explicit implementation

If you have two interfaces with the same member name, or you want to hide an implementation from the regular API:

```csharp
public interface IReadable { void Read(); }
public interface IWritable { void Read(); }   // also has Read()

public class Document : IReadable, IWritable {
    void IReadable.Read() => Console.WriteLine("reading text");
    void IWritable.Read() => Console.WriteLine("reading writeable");
}

var d = new Document();
// d.Read();              // ❌ — Read is explicit, not on Document directly
((IReadable)d).Read();    // "reading text"
((IWritable)d).Read();    // "reading writeable"
```

Explicit implementation:
- Member has no access modifier (always public-via-interface).
- Not visible on the class directly — must cast.
- Useful for disambiguation, or for hiding "internal" interfaces (like `IDisposable.Dispose()` on classes where dispose is unusual).

---

## Default interface methods (DIM, C# 8+)

An interface member can have a **default implementation**:

```csharp
public interface ILogger {
    void Log(string message);

    // Default — implementers can ignore
    void LogError(string message) => Log($"ERROR: {message}");

    static int InstanceCount;          // static fields legal too
    static void Reset() { InstanceCount = 0; }   // static method
}

public class ConsoleLogger : ILogger {
    public void Log(string message) => Console.WriteLine(message);
    // LogError inherited from interface
}

ILogger l = new ConsoleLogger();
l.LogError("bad");   // calls ILogger's default
```

Important quirks:
- The default implementation runs **only when called through the interface** — `((ILogger)logger).LogError("...")`. A `ConsoleLogger.LogError(...)` call doesn't exist (the default isn't merged into the class's vtable).
- Static members in interfaces work like static members in classes.
- DIMs were added primarily to **let interface authors add members without breaking existing implementers**. Don't use them as a substitute for abstract base classes.

```csharp
// Existing in v1:
public interface ILogger {
    void Log(string msg);
}

// v2 — adding a method would break every implementer:
public interface ILogger {
    void Log(string msg);
    void LogError(string msg) => Log($"ERROR: {msg}");  // default, no break
}
```

---

## Static abstract members (C# 11+)

Interfaces can declare **static** members that implementers must provide. Key for generic math.

```csharp
public interface IShape<T> where T : IShape<T> {
    static abstract T Default();
}

public class Square : IShape<Square> {
    public static Square Default() => new Square();
}

// Generic dispatch on a static interface member
T Build<T>() where T : IShape<T> => T.Default();
Square s = Build<Square>();
```

This unlocks **generic arithmetic** — interfaces like `INumber<T>`, `IAdditionOperators<T,T,T>` let you write algorithms generic over numeric types:

```csharp
T Sum<T>(IEnumerable<T> nums) where T : INumber<T> {
    T total = T.Zero;
    foreach (var n in nums) total += n;
    return total;
}

Sum(new[] { 1, 2, 3 });           // works for int
Sum(new[] { 1.5, 2.5 });          // works for double
Sum(new[] { 1m, 2m });            // works for decimal
```

Massive deal for libraries — see [Chapter 04 §06](../04-Generics/06-GenericMath.md).

---

## Marker interfaces

Interfaces with no members, used for type discrimination:

```csharp
public interface IEntity { }     // empty contract

public class User : IEntity { ... }
public class Order : IEntity { ... }

bool IsEntity(object obj) => obj is IEntity;
```

Pre-attribute era this was common; nowadays, **attributes** (`[Entity]`) are usually a better choice — they don't pollute the type system, and reflection can find them.

---

## `IDisposable` — the most ubiquitous interface

```csharp
public class FileHandle : IDisposable {
    private FileStream _fs;
    public FileHandle(string path) { _fs = File.OpenRead(path); }
    public void Dispose() => _fs?.Dispose();
}

using (var h = new FileHandle("x.txt")) {
    // ...
}
// Dispose called automatically
```

[Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md) has the full Dispose pattern.

---

## `IEnumerable<T>` and friends

The contract underpinning `foreach`:

```csharp
public interface IEnumerable<out T> {
    IEnumerator<T> GetEnumerator();
}

public interface IEnumerator<T> : IDisposable {
    T Current { get; }
    bool MoveNext();
}
```

`foreach (var x in items)` calls `GetEnumerator()`, then loops `MoveNext` + `Current`.

You almost never implement these manually — the `yield return` syntax does it for you:

```csharp
public IEnumerable<int> Numbers() {
    yield return 1;
    yield return 2;
    yield return 3;
}
```

The compiler generates a state machine implementing `IEnumerator<int>` automatically.

---

## Variance in interfaces — `in` / `out`

A type parameter can be covariant (`out T`) or contravariant (`in T`):

```csharp
public interface IProducer<out T> {
    T Produce();
}

public interface IConsumer<in T> {
    void Consume(T item);
}

IProducer<string> sp = ...;
IProducer<object> op = sp;     // covariance: legal because IProducer only outputs T

IConsumer<object> oc = ...;
IConsumer<string> sc = oc;     // contravariance: legal — a consumer of any object accepts strings
```

`out` — outputs only. `in` — inputs only. Mutable mid-positions (`IList<T>`) must be invariant for safety.

Full coverage in [Chapter 04 §04](../04-Generics/04-Variance.md).

---

## Common interfaces you'll meet

| Interface | Purpose |
|---|---|
| `IEnumerable<T>` | Iteration via `foreach` |
| `IEnumerator<T>` | The iterator object |
| `IAsyncEnumerable<T>` | Async iteration via `await foreach` |
| `IList<T>` | Indexed, mutable list |
| `ICollection<T>` | Mutable collection |
| `IReadOnlyList<T>` | Read-only indexed |
| `IReadOnlyDictionary<K,V>` | Read-only dictionary |
| `IDictionary<K,V>` | Mutable map |
| `ISet<T>` | Mutable set |
| `IDisposable` | Has resources to release |
| `IAsyncDisposable` | Async resources |
| `IComparable<T>` | Has total order; supports `Sort` |
| `IComparer<T>` | External comparer object |
| `IEquatable<T>` | Custom equality |
| `IEqualityComparer<T>` | External equality comparer |
| `IFormattable` | Custom `ToString(format, provider)` |
| `IConvertible` | Various `ToInt32`, `ToString` etc. (rarely used directly) |
| `INotifyPropertyChanged` | MVVM binding |
| `IObservable<T>` / `IObserver<T>` | Reactive Extensions |

---

## Internals — how interface dispatch works

Each class with interface implementations has an **interface map** in its method table — a list of (interface, method-index) → method-pointer entries.

When you call `interface.Method()`:
1. `callvirt` IL instruction with the interface method as the target.
2. Runtime looks up the receiver's MT.
3. Walks the interface map to find the matching interface.
4. Reads the slot for that interface method → real implementation address.
5. Calls.

Interface dispatch is **slower** than regular virtual dispatch (one extra map lookup), but the JIT optimizes hot paths. PGO can identify the most common implementer and emit a fast type check + direct call.

### IL for an interface call

```csharp
ILogger l = new ConsoleLogger();
l.Log("hi");
```

```il
newobj    instance void ConsoleLogger::.ctor()
stloc.0
ldloc.0
ldstr     "hi"
callvirt  instance void ILogger::Log(string)
```

Note `callvirt instance void ILogger::Log(string)` — the target is the **interface** method, not `ConsoleLogger.Log`. The runtime resolves to the implementation.

### Default interface method dispatch

A default interface method has its IL body in the interface itself. When called through the interface and the implementer hasn't overridden it, the runtime calls the interface's body directly.

If the implementer overrides it, the implementer's version wins:

```csharp
public interface I { void M() => Console.WriteLine("default"); }
public class C : I { public void M() => Console.WriteLine("class"); }

I i = new C();
i.M();    // "class" — class implementation wins

((I)null!).M();   // NRE — no receiver
```

### Layout of an interface-rich object

A class implementing 5 interfaces still has just one MT pointer in each instance. The MT has a list of interface maps. Implementations are stored once; instances are not larger because they implement more interfaces.

---

## Interface vs abstract class

Often a design choice. Heuristics:

| Interface | Abstract class |
|---|---|
| Contract only | Contract + shared state and/or code |
| Can be implemented by many unrelated types | "Is-a" relationship |
| All members public | Can have non-public members |
| No state (no instance fields) | Has fields |
| Multiple implementation allowed | Single inheritance |
| Add to existing types via implementation | Requires modification |
| Source-generator friendly | Less so |

A common pattern: **interface for the contract, abstract class for shared base implementation**:

```csharp
public interface IShape {
    double Area();
}

public abstract class ShapeBase : IShape {
    public string Name { get; init; } = "";
    public abstract double Area();
    public override string ToString() => $"{Name} ({Area():F2})";
}

public class Circle : ShapeBase {
    public double Radius { get; init; }
    public override double Area() => Math.PI * Radius * Radius;
}
```

[§09](09-AbstractVsInterface.md) drills into this.

---

## Common patterns

### Dependency injection

```csharp
public class OrderService(IRepository repo, IEmailService email) {
    public async Task PlaceAsync(Order o) {
        await repo.SaveAsync(o);
        await email.SendAsync(o.UserEmail, "placed");
    }
}
```

Inject interfaces, not concretes. Tests pass fakes; production passes real impls.

### Adapter

```csharp
public interface ILogger { void Log(string s); }
public class LegacyLog { public void WriteLine(string s, int level) {} }

public class LegacyAdapter(LegacyLog inner) : ILogger {
    public void Log(string s) => inner.WriteLine(s, 0);
}
```

Wrap a non-matching API in your interface.

### Strategy

```csharp
public interface IPricingStrategy { decimal Apply(decimal s); }
public class TenPercentOff : IPricingStrategy { ... }
public class HappyHour : IPricingStrategy { ... }
```

Swap policy without conditionals.

---

## Common bugs

- **Default interface method called on null receiver** — runtime NRE.
- **Casting requires the interface or its base** — `(IDisposable)obj` when obj doesn't implement it throws.
- **Forgetting to dispose an `IDisposable` field** — implement `IDisposable` yourself and chain.
- **Boxing when implementing on structs** — `ICollection<T> ic = mySpan;` for a struct boxes it. Generic methods with `where T : IFoo` constraint avoid this.
- **Overriding interface-method-via-default in a class isn't possible** without explicit interface implementation. The class re-implements it.

---

## When to use an interface

- A capability could be implemented by unrelated types.
- You want dependency injection / testing.
- The contract is stable; implementers vary.
- You're adding mixin-like behavior to existing types.

When NOT to use an interface:
- The contract has only one realistic implementation. Just use the class.
- You're abstracting just because "interfaces are best practice." Premature abstraction hurts readability.

→ Next: [09-AbstractVsInterface.md](09-AbstractVsInterface.md)
