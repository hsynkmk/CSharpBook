# Common Anti-Patterns

## What this is

A catalog of recurring mistakes — code that "works" but causes bugs, poor performance, or unmaintainable systems. Recognizing these by name helps you spot and avoid them. Each entry: what it is, why it's bad, and the fix.

---

## `async void`

```csharp
// ✗
public async void SaveData() { await _repo.SaveAsync(); }
```

**Why bad**: Can't be awaited; exceptions escape to the synchronization context and crash the process; callers can't know when it finishes.

**Fix**: `async Task`. Only event handlers may be `async void` (with try/catch inside). See [05-AsyncIdioms.md](05-AsyncIdioms.md).

---

## Sync-over-async

```csharp
// ✗
var result = GetDataAsync().Result;
GetDataAsync().Wait();
```

**Why bad**: Deadlocks when a synchronization context is captured (UI, legacy ASP.NET); always blocks a thread, hurting scalability (thread-pool starvation).

**Fix**: `await` all the way up. If you truly must bridge, do it only at the top (console `Main`). See [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md).

---

## Swallowing exceptions

```csharp
// ✗
try { Process(); } catch { }                       // silent
try { Process(); } catch (Exception) { return; }   // silent + broad
```

**Why bad**: Hides failures; the bug surfaces later (corrupted data, mysterious nulls) far from the cause. Catching the base `Exception` can also swallow things you shouldn't (`OutOfMemoryException`).

**Fix**: Catch the **specific** exception you can handle; log and rethrow or let it propagate otherwise. Never an empty catch.

```csharp
try { Process(); }
catch (IOException ex) { _logger.LogError(ex, "Processing failed"); throw; }
```

---

## Primitive obsession

```csharp
// ✗ — everything is a string/int; no type safety, easy to swap arguments
void Transfer(string fromAccount, string toAccount, decimal amount, string currency);
Transfer(to, from, amount, "USD");   // oops — from/to swapped, compiler can't catch it
```

**Why bad**: Primitives carry no domain meaning or validation; arguments of the same type are easily transposed; validation is scattered.

**Fix**: Wrap domain concepts in types (value objects / records):

```csharp
public readonly record struct AccountId(string Value);
public readonly record struct Money(decimal Amount, string Currency);

void Transfer(AccountId from, AccountId to, Money amount);
// Transfer(to, from, ...) still compiles, but Money/AccountId can't be swapped with each other
```

---

## Anemic domain model

```csharp
// ✗ — data bag with no behavior; logic lives elsewhere
public class Order { public List<Item> Items { get; set; } public decimal Total { get; set; } }
public class OrderService {
    public void AddItem(Order o, Item i) { o.Items.Add(i); o.Total += i.Price; }  // logic outside the entity
}
```

**Why bad**: The object holds data but no behavior; invariants aren't enforced (anyone can set `Total` inconsistently); logic is scattered across "service" classes.

**Fix**: Put behavior and invariants on the entity (rich domain model):

```csharp
public class Order {
    private readonly List<Item> _items = new();
    public IReadOnlyList<Item> Items => _items;
    public decimal Total => _items.Sum(i => i.Price);   // computed; can't be inconsistent

    public void AddItem(Item item) {                     // behavior + invariant on the entity
        ArgumentNullException.ThrowIfNull(item);
        _items.Add(item);
    }
}
```

(Anemic models are sometimes acceptable for pure DTOs — the anti-pattern is when *domain* logic that belongs on the entity lives elsewhere.)

---

## God class / God method

```csharp
// ✗ — one class doing everything: data access, validation, email, reporting, logging
public class OrderManager { /* 2000 lines, 50 methods */ }
```

**Why bad**: Violates single-responsibility; impossible to test, understand, or change safely; every feature touches it.

**Fix**: Split by responsibility into focused classes (repository, validator, notifier). A class should have **one reason to change**. A method should do one thing — if you describe it with "and," split it.

---

## Magic strings and numbers

```csharp
// ✗
if (user.Role == "admin") { ... }
if (status == 3) { ... }
SetTimeout(86400);
```

**Why bad**: No compile-time checking (typos compile); meaning unclear; changes require hunting every occurrence.

**Fix**: Constants, enums, `nameof`:

```csharp
public enum Role { Admin, User, Guest }
if (user.Role == Role.Admin) { ... }

private static readonly TimeSpan DefaultTimeout = TimeSpan.FromDays(1);
SetTimeout(DefaultTimeout);

OnPropertyChanged(nameof(FullName));   // not "FullName"
```

---

## Service locator

```csharp
// ✗ — dependencies hidden, pulled from a global container
public class OrderService {
    public void Process() {
        var repo = ServiceLocator.Get<IRepository>();   // hidden dependency
        var email = ServiceLocator.Get<IEmailSender>();
    }
}
```

**Why bad**: Dependencies are hidden (not in the constructor) — you can't tell what a class needs without reading its body; hard to test; couples everything to the locator.

**Fix**: Constructor injection (dependencies explicit and testable):

```csharp
public class OrderService(IRepository repo, IEmailSender email) {
    public void Process() { /* use repo, email */ }
}
```

---

## Returning null for collections

```csharp
// ✗
public List<Order>? GetOrders() => _orders.Count > 0 ? _orders : null;
```

**Why bad**: Forces null checks everywhere; one missed check → `NullReferenceException`.

**Fix**: Return an empty collection (`[]`, `Array.Empty<T>()`). See [06-CollectionIdioms.md](06-CollectionIdioms.md).

---

## Premature optimization

```csharp
// ✗ — micro-optimizing cold code, sacrificing clarity for unmeasured gains
// (hand-rolled bit twiddling in a method called once at startup)
```

**Why bad**: Adds complexity and bugs for gains that don't matter; obscures intent; you usually optimize the wrong thing without profiling.

**Fix**: Write clear code first; **profile** to find real hot spots; optimize only those (see [03-PerformanceIdioms.md](03-PerformanceIdioms.md), [Chapter 15 §08](../15-BuildTooling/08-Profiling.md)).

---

## Stringly-typed code

```csharp
// ✗ — using strings where types should be
object value = GetConfig("timeout");
int timeout = int.Parse(value.ToString());
```

**Why bad**: No type safety, runtime parse errors, no IntelliSense.

**Fix**: Strongly-typed configuration/options classes, enums, generics.

---

## Double-checked locking done wrong / reinventing lazy init

```csharp
// ✗ — subtle, easy to get wrong (memory model issues)
if (_instance == null) { lock (_lock) { if (_instance == null) _instance = new(); } }
```

**Why bad**: Easy to introduce memory-visibility bugs; reinvents what the BCL provides.

**Fix**: `Lazy<T>` (thread-safe by default):

```csharp
private static readonly Lazy<Service> _instance = new(() => new Service());
public static Service Instance => _instance.Value;
```

See [Chapter 08 §09](../08-Concurrency/09-LockingPrimitives.md).

---

## Catch-rethrow that loses the stack trace

```csharp
// ✗ — 'throw ex' resets the stack trace to here
try { ... } catch (Exception ex) { Log(ex); throw ex; }
```

**Why bad**: `throw ex;` resets the stack trace, hiding where the exception originated.

**Fix**: `throw;` (preserves the original stack), or wrap in a new exception with `innerException`:

```csharp
catch (Exception ex) { Log(ex); throw; }                              // preserve
catch (IOException ex) { throw new DataException("Load failed", ex); } // wrap with inner
```

---

## Disposing what you don't own / not disposing what you do

```csharp
// ✗ — disposes the caller's stream
void Read(Stream s) { using var r = new StreamReader(s); ... }   // closes s!

// ✗ — leaks: HttpClient created per call
void Call() { var c = new HttpClient(); ... }                     // socket exhaustion
```

**Fix**: Use `leaveOpen: true` when you don't own a resource; use `IHttpClientFactory` / a shared `HttpClient`; `using`/`await using` for resources you do own. See [Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md), [Chapter 13 §02](../13-IO/02-Streams.md).

---

## Summary table

| Anti-pattern | Fix |
|---|---|
| `async void` | `async Task` (except event handlers) |
| Sync-over-async (`.Result`/`.Wait()`) | `await` all the way |
| Swallowing exceptions | Catch specific, log + rethrow |
| Primitive obsession | Value objects / records |
| Anemic domain model | Behavior + invariants on entities |
| God class/method | Single responsibility, split |
| Magic strings/numbers | Constants, enums, `nameof` |
| Service locator | Constructor injection |
| Null collections | Empty collections (`[]`) |
| Premature optimization | Profile first, optimize hot paths |
| `throw ex;` | `throw;` (preserve stack) |
| Disposing others' resources | `leaveOpen`, `IHttpClientFactory` |
| Manual double-checked locking | `Lazy<T>` |

→ Next: [10-SOLID.md](10-SOLID.md)
