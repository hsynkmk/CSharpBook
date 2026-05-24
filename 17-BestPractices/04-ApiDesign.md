# API Design

## What it covers

Designing the public surface of a library or shared module — the types, methods, parameters, and exceptions other code depends on. Good API design makes the right thing easy and the wrong thing hard. Bad design haunts you forever, because **public APIs are a contract you can't break without breaking your callers.** This follows Microsoft's Framework Design Guidelines.

---

## Public surface is a contract

Once published, every public type and member is a promise. Changing it breaks consumers. So:

- Make the public surface **as small as possible**. Default to `internal`; promote to `public` deliberately.
- Use `sealed` by default for classes not designed for inheritance — you can unseal later (non-breaking) but not re-seal (breaking).
- Prefer exposing **interfaces or abstract types** for extension points you control.

```csharp
public sealed class OrderService { ... }   // sealed unless inheritance is a designed feature
internal class OrderValidator { ... }       // internal until something external needs it
```

---

## Design for the caller, not the implementer

The API should read naturally at the **call site**:

```csharp
// ✗ — flags and ambiguous parameters
parser.Parse(input, true, false, 3);

// ✓ — intent obvious at the call site
parser.Parse(input, ParseOptions.Strict);
```

Boolean parameters are a smell at the call site (`Foo(true, false)` is unreadable). Prefer enums or options objects. Name parameters so call sites self-document.

---

## Parameter ordering and conventions

Established .NET parameter conventions:

```csharp
// Required/primary inputs first, options later, CancellationToken LAST
Task<Result> ProcessAsync(
    Order order,                      // primary input
    ProcessOptions? options = null,    // optional config
    CancellationToken ct = default);   // cancellation token always last

// "this"-style: the thing being operated on first
void CopyTo(T[] destination, int index);

// out parameters last
bool TryParse(string input, out Result result);
```

- Most important/required parameters first.
- Optional parameters (with defaults) after.
- `CancellationToken` **last** (see [05-AsyncIdioms.md](05-AsyncIdioms.md)).
- `out` parameters last (Try-pattern).

---

## The Try-pattern for expected failures

```csharp
// Throwing version — for "this should succeed" cases
public int Parse(string s);                              // throws FormatException on bad input

// Try version — for "failure is expected/common"
public bool TryParse(string s, out int result);          // returns false, no exception
```

Provide a `TryX` variant when failure is a **normal, expected** outcome (user input validation). Exceptions are for **exceptional** conditions, not control flow — they're expensive and noisy when failure is routine. Many BCL types offer both (`int.Parse`/`int.TryParse`, `Dictionary` indexer/`TryGetValue`).

---

## Exception design

```csharp
// ✓ — specific, standard exception types with clear messages
throw new ArgumentNullException(nameof(order));
throw new ArgumentOutOfRangeException(nameof(amount), amount, "Amount must be positive.");
throw new InvalidOperationException("Cannot ship an order that hasn't been paid.");

// ✗ — generic, unhelpful
throw new Exception("error");
```

Rules:
- Throw the **most specific** standard exception (`ArgumentNullException`, `ArgumentOutOfRangeException`, `InvalidOperationException`, `NotSupportedException`).
- Include the parameter name (`nameof`) and a message explaining *what* and *why*.
- Define **custom exceptions** only when callers need to catch them specifically; derive from `Exception` (not `ApplicationException`), make them `[Serializable]` only if needed, and add the standard constructors.
- Don't throw `Exception`, `SystemException`, or `ApplicationException` directly.
- Document thrown exceptions in XML docs (`<exception>`).

See [Chapter 01 §08](../01-Fundamentals/08-ExceptionsBasics.md).

---

## Method overloads and defaults

```csharp
// Prefer optional parameters over a pile of overloads (where it reads well)
public void Connect(string host, int port = 443, TimeSpan? timeout = null);

// But: optional-parameter default values are baked into the CALLER at compile time.
// Changing the default in a library requires recompiling callers to take effect.
```

A subtle versioning trap: **default parameter values are compiled into the caller**. If you ship a library and later change a default, existing compiled callers keep the old default. For library public APIs where the default might change, overloads can be safer than optional parameters.

---

## Return types — be generous in what you return

```csharp
// ✓ — return the most useful concrete-enough type
public IReadOnlyList<Order> GetOrders();    // caller can index + count, can't mutate

// ✗ — returning a mutable List exposes internal state for mutation
public List<Order> GetOrders();             // caller can Add/Remove your internal list!
```

Return interfaces that give callers what they need without over-committing or exposing mutability. `IReadOnlyList<T>`/`IReadOnlyCollection<T>` for results; never hand out your internal mutable collection. See [06-CollectionIdioms.md](06-CollectionIdioms.md).

---

## Accept the least specific type you need

```csharp
// ✓ — accept the broadest type that works (callers can pass anything enumerable)
public int Sum(IEnumerable<int> numbers);

// ✗ — forces callers to have a List
public int Sum(List<int> numbers);
```

"Be liberal in what you accept, conservative in what you return." Accept `IEnumerable<T>` (or `ReadOnlySpan<T>` for perf) for inputs you only enumerate; return concrete-enough read-only types.

---

## Versioning and breaking changes

A change is **breaking** if it can break a consumer at compile time or runtime:

| Breaking | Non-breaking |
|---|---|
| Remove/rename public member | Add a new member |
| Change a method signature | Add an overload |
| Add a member to an interface* | Add a default interface method (mostly) |
| Change return type | Add an optional parameter (caller-recompile caveat) |
| Make a type/member less accessible | Make it more accessible |
| Throw a new exception type | — |
| Re-seal a class | Unseal a class |

\*Adding to an interface breaks implementers — unless you provide a **default interface method** (C# 8+), which lets you extend an interface without breaking existing implementers.

Follow **SemVer** (see [Chapter 15 §03](../15-BuildTooling/03-NuGet.md)): breaking changes → major version bump. Use `[Obsolete]` to deprecate before removing:

```csharp
[Obsolete("Use ProcessAsync instead. Will be removed in v3.0.")]
public void Process() => ProcessAsync().GetAwaiter().GetResult();
```

---

## Immutability and thread-safety in APIs

- Prefer immutable types (`record`, `init`-only) for data crossing API boundaries — callers can't corrupt your state, and they're thread-safe to share.
- Document thread-safety: "instances are thread-safe" / "not thread-safe; use one per request."
- Static members should be thread-safe; instance members may not be (state convention).

---

## Common bugs / gotchas

### Exposing mutable internal state

Returning your internal `List<T>`/array lets callers mutate it. Return read-only views or copies.

### Boolean parameters

`Create(true, false, true)` is unreadable. Use enums/options objects.

### Throwing generic exceptions

`throw new Exception("...")` forces callers to catch everything. Throw specific types.

### Changing optional-parameter defaults in a shipped library

Existing callers keep the old default (baked in at their compile time). Use overloads when the default may evolve.

### Over-exposing the surface

Everything `public` becomes a maintenance burden and a breaking-change risk. Default to `internal`.

---

## Summary

- The public surface is a **contract** — keep it small, default to `internal`/`sealed`, promote deliberately.
- Design for the **call site**: avoid boolean params (use enums/options), order params required→optional→`CancellationToken` last, `out` last.
- Provide **Try-patterns** for expected failures; reserve exceptions for exceptional cases. Throw specific exceptions with `nameof` + clear messages.
- Accept the **broadest** input type (`IEnumerable<T>`), return **read-only** concrete-enough types; never expose mutable internals.
- Know what's breaking (rename/remove/signature) vs not (add member/overload); follow SemVer; deprecate with `[Obsolete]`; extend interfaces via default methods.

→ Next: [05-AsyncIdioms.md](05-AsyncIdioms.md)
