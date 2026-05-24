# Defensive Programming

## What it is

Defensive programming means **validating assumptions and failing fast** so bugs surface at their source, with a clear message, instead of corrupting state and exploding somewhere distant. The core practice: guard clauses at method entry, the right exception types, and "fail fast" over silent error states.

---

## Guard clauses — validate at the entry

Check preconditions at the top of a method and bail immediately. This flattens nesting and locates failures at the cause:

```csharp
public Order CreateOrder(Customer customer, IReadOnlyList<LineItem> items, decimal discount) {
    ArgumentNullException.ThrowIfNull(customer);
    ArgumentNullException.ThrowIfNull(items);
    ArgumentOutOfRangeException.ThrowIfNegative(discount);
    if (items.Count == 0)
        throw new ArgumentException("Order must have at least one item.", nameof(items));

    // ... main logic at the top level, preconditions guaranteed
}
```

Guard clauses replace deep `if (x != null) { if (...) { ... } }` pyramids (see [Chapter 17 §02](02-CodingGuidelines.md)).

---

## Modern guard helpers (.NET 6+)

The BCL added concise, allocation-free guard methods that use `[CallerArgumentExpression]` to auto-fill the parameter name:

```csharp
// Null checks
ArgumentNullException.ThrowIfNull(arg);
ArgumentNullException.ThrowIfNull(arg.Property);   // throws with "arg.Property" as the name

// String checks
ArgumentException.ThrowIfNullOrEmpty(name);
ArgumentException.ThrowIfNullOrWhiteSpace(name);

// Numeric checks
ArgumentOutOfRangeException.ThrowIfNegative(count);
ArgumentOutOfRangeException.ThrowIfNegativeOrZero(count);
ArgumentOutOfRangeException.ThrowIfZero(divisor);
ArgumentOutOfRangeException.ThrowIfGreaterThan(value, max);
ArgumentOutOfRangeException.ThrowIfLessThan(value, min);
ArgumentOutOfRangeException.ThrowIfEqual(a, b);

// Disposed-object check
ObjectDisposedException.ThrowIf(_disposed, this);
```

These are clearer than hand-written checks and produce good messages automatically (the parameter name comes from `CallerArgumentExpression` — see [Chapter 10 §12](../10-AdvancedLanguage/12-CallerInfoAttributes.md)).

```csharp
// ✗ — verbose, manual name
if (customer == null) throw new ArgumentNullException(nameof(customer));

// ✓ — concise, auto-named
ArgumentNullException.ThrowIfNull(customer);
```

---

## `ObjectDisposedException` for disposed objects

```csharp
public sealed class Connection : IDisposable {
    private bool _disposed;

    public void Send(string data) {
        ObjectDisposedException.ThrowIf(_disposed, this);   // fail fast if used after dispose
        // ...
    }

    public void Dispose() => _disposed = true;
}
```

Methods on a disposable type should reject use after disposal with a clear `ObjectDisposedException` rather than producing undefined behavior. See [Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md).

---

## Fail fast vs return error

When should a method throw vs return a failure value?

| Throw an exception | Return an error result |
|---|---|
| Precondition violated (caller bug) | Failure is expected/normal |
| Invariant broken (impossible state) | User input validation |
| Unrecoverable / programming error | Control flow depends on it |
| Rare | Frequent |

```csharp
// Caller bug → throw (the contract was violated)
public void Withdraw(decimal amount) {
    ArgumentOutOfRangeException.ThrowIfNegativeOrZero(amount);   // caller passed garbage
    if (amount > _balance) throw new InvalidOperationException("Insufficient funds");
}

// Expected failure → Try-pattern or result type
public bool TryParse(string input, out int value);                    // routine failure
public Result<Order> PlaceOrder(OrderRequest req);                    // result type
```

**Exceptions are for exceptional conditions**, not routine control flow (they're expensive and noisy). For expected failures (validation, parsing, "not found"), prefer the Try-pattern or a result type. See [Chapter 17 §04](04-ApiDesign.md).

---

## Result types for expected failures

```csharp
public readonly record struct Result<T>(bool Success, T? Value, string? Error) {
    public static Result<T> Ok(T value) => new(true, value, null);
    public static Result<T> Fail(string error) => new(false, default, error);
}

public Result<Order> PlaceOrder(OrderRequest req) {
    if (req.Items.Count == 0) return Result<Order>.Fail("No items");
    if (!_inventory.HasStock(req)) return Result<Order>.Fail("Out of stock");
    return Result<Order>.Ok(BuildOrder(req));
}

// Caller handles both paths explicitly — no try/catch for routine failure
var result = service.PlaceOrder(req);
if (!result.Success) return BadRequest(result.Error);
```

Result types make expected failures part of the type signature, forcing callers to handle them — clearer and faster than exceptions for routine outcomes. (Libraries like `OneOf`, `ErrorOr`, `FluentResults`, or `LanguageExt` provide richer versions.)

---

## Validate at boundaries, trust internally

```csharp
// Public API boundary — validate untrusted input
public Order CreateOrder(OrderRequest request) {
    ArgumentNullException.ThrowIfNull(request);
    Validate(request);   // thorough validation of external input
    return CreateOrderInternal(request);
}

// Internal method — preconditions already guaranteed; lighter checks (Debug.Assert)
private Order CreateOrderInternal(OrderRequest request) {
    Debug.Assert(request is not null);   // documents the invariant; removed in Release
    // ... trust the input
}
```

Validate **untrusted input at boundaries** (public APIs, deserialization, user input). Internally, where preconditions are already established, use `Debug.Assert` to document invariants without runtime cost in Release. Don't re-validate the same thing at every layer.

---

## Nullable reference types as a defensive tool

```csharp
#nullable enable

public Order? Find(int id);             // explicitly may return null — caller must check
public Order Get(int id);               // never returns null — throws if not found

public void Process(Order order) {       // 'order' is non-nullable — NRT flags null callers at compile time
    // no null check needed if NRT is honored throughout
}
```

Enable NRT (`<Nullable>enable</Nullable>`) so the compiler tracks nullability and flags potential null dereferences at compile time — defensive programming enforced by the type system. It reduces (but doesn't eliminate) the need for runtime null checks at trusted boundaries. See [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md).

---

## Common bugs / gotchas

### Swallowing exceptions

```csharp
try { Process(); }
catch { }   // ✗ — silently hides failures; the bug vanishes until it corrupts something later
```

Never swallow exceptions silently. Catch specific types you can handle; log and rethrow (or let it propagate) otherwise. See [09-CommonAntiPatterns.md](09-CommonAntiPatterns.md).

### Validating too late

Checking arguments deep in the call stack means the failure points far from the cause. Validate at the entry/boundary.

### Using exceptions for control flow

```csharp
try { value = dict[key]; } catch (KeyNotFoundException) { value = default; }   // ✗ — slow, noisy
value = dict.GetValueOrDefault(key);                                            // ✓
```

Exceptions are expensive; don't use them for expected, routine cases. Use Try-patterns.

### Over-defensive code

Re-null-checking the same parameter at every layer adds noise. Validate at the boundary, then trust. Trust your own invariants (with `Debug.Assert` to document them).

### Catching `Exception` broadly in libraries

Catching the base `Exception` hides bugs (and can catch things you shouldn't, like `OutOfMemoryException`). Catch the specific types you can actually handle.

---

## Summary

- **Guard clauses** at method entry validate preconditions and fail fast with clear messages; they flatten nesting.
- Use modern helpers: `ArgumentNullException.ThrowIfNull`, `ArgumentException.ThrowIfNullOrWhiteSpace`, `ArgumentOutOfRangeException.ThrowIfNegative`, `ObjectDisposedException.ThrowIf`.
- **Throw** for caller bugs/broken invariants (rare); **return a result/Try-pattern** for expected failures (routine) — don't use exceptions for control flow.
- Validate **untrusted input at boundaries**; trust internally (`Debug.Assert` documents invariants without Release cost).
- Enable **NRT** to push null-safety into the compiler.
- Never swallow exceptions; catch only specific, handleable types.

→ Next: [09-CommonAntiPatterns.md](09-CommonAntiPatterns.md)
