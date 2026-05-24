# `default(T)` and the `default` Literal

## What it is

`default` produces the **default value** of a type — the all-zero / null / "nothing in particular" instance. It works for any type, including generic type parameters.

Three forms:
- **`default(T)`** — explicit, works since C# 1.
- **`default`** — target-typed literal (C# 7.1+). The compiler infers T from context.
- The implicit default of a field — applies automatically when you declare without initialization.

```csharp
int n = default;              // 0
double d = default;           // 0.0
bool b = default;             // false
char c = default;             // '\0'
string? s = default;          // null
List<int>? l = default;       // null
int? nullable = default;      // null (Nullable<int>)
Point p = default;            // struct with all fields zero
T t = default;                // depends on T

// Explicit form:
int n2 = default(int);
T t2 = default(T);
```

The default value matters because:
- **Fields** default automatically — predictable initial state.
- **Generic methods** often need to return or produce a value of T without a concrete instance.
- **Pattern matching** can check against `default`.

---

## What "default" means per type

| Type | `default` |
|---|---|
| Numeric (int, double, decimal) | `0` |
| `bool` | `false` |
| `char` | `'\0'` |
| `DateTime` | `DateTime.MinValue` (year 0001-01-01 00:00:00) |
| `TimeSpan` | `TimeSpan.Zero` (00:00:00) |
| `Guid` | `Guid.Empty` (all zeros) |
| `enum` | the member with value `0` (or the int 0 if no such member) |
| `struct` | all fields zeroed |
| `class` / interface / delegate | `null` |
| `Nullable<T>` (`T?`) | `null` (HasValue = false) |
| `string` | `null` (it's a reference type) |
| `Span<T>` | empty span (length 0, null pointer) |

For value types, default is **always reachable** — you can't prevent `default(MyStruct)` from being constructed (e.g., as a field of an uninitialized class).

---

## `default(T)` in generic methods

This is where it earns its keep:

```csharp
public T? FirstOrDefault<T>(IEnumerable<T> source) {
    foreach (var x in source) return x;
    return default;        // returns default(T) — null for ref, 0 for int, etc.
}

int n = FirstOrDefault(new int[] { });      // 0 (default int)
string? s = FirstOrDefault(new string[] { });  // null (default string)
```

`default` here is convenient — you don't have to write a separate path for "T might be a value type."

---

## The `default` literal

Since C# 7.1, `default` (without `(T)`) is **target-typed** — the compiler infers from context:

```csharp
int n = default;                  // T inferred as int → 0
List<int>? list = default;        // T inferred as List<int>? → null
Func<int, int> f = default;       // null

// Useful in pattern matching:
if (n == default) { ... }         // n == 0

// And as a fallback in generics:
public T Get<T>(string key) where T : IDefault<T> =>
    _store.TryGetValue(key, out var v) ? (T)v : default!;
```

The `!` (null-forgiving) is sometimes needed when NRT is on and T could be a reference type — `default` may be null, which NRT flags.

---

## `default` vs `null` vs `0`

For a value type, `default(int)` and `0` are equivalent. For a reference type, `default(string)` and `null` are equivalent.

Use `default` when:
- You're writing **generic** code where T might be either.
- You want clarity that the value is "whatever counts as nothing for this type."

Use `0`, `false`, `null` directly when:
- The type is concrete and well-known to the reader.
- You're making a specific choice (not just "zero out this slot").

```csharp
int counter = 0;            // ✓ — clear that this is a count starting at zero
T counter = default;        // ✓ — counter resets to T's default

string? msg = null;         // ✓ — explicit "no message"
T result = default;         // ✓ — generic "no value yet"
```

---

## Initialization: when defaults apply

### Class and struct **fields**

Auto-defaulted when an instance is created:

```csharp
public class Demo {
    public int X;             // 0
    public string? Name;      // null
    public DateTime Created;  // DateTime.MinValue
}

var d = new Demo();
Console.WriteLine(d.X);       // 0
Console.WriteLine(d.Created); // 0001-01-01 00:00:00
```

The constructor doesn't need to assign these; the runtime zeroes the memory.

### Array elements

Auto-defaulted on creation:

```csharp
int[] nums = new int[3];        // { 0, 0, 0 }
string?[] names = new string?[3];  // { null, null, null }
Point[] pts = new Point[3];      // 3 zero-Point structs
```

### Locals

**Not** auto-defaulted. You must initialize before reading:

```csharp
int x;
Console.WriteLine(x);   // ❌ compile error — use of unassigned local
```

If you want the default explicitly:

```csharp
int x = default;        // OK — 0
```

### `out` parameters

Don't need to be initialized before the call; the method must assign them. The compiler treats them as "assigned in the method's body."

```csharp
void Try(out int x) { x = 0; }
```

---

## `default` in pattern matching

You can pattern-match against `default`:

```csharp
public T? Find<T>(...) {
    // ...
    return default;
}

var result = Find<int>(...);
if (result is default(int)) { /* result is 0 */ }   // explicit form

// Or simpler with a constant:
if (result == default) { /* same */ }
```

But for clarity, comparing to a specific value (`== 0`, `== null`) is usually better than `default`.

---

## `Nullable<T>` and `default`

```csharp
int? n = default;
Console.WriteLine(n.HasValue);   // false
Console.WriteLine(n is null);    // true
Console.WriteLine(n == default); // true
```

For nullable value types, `default` is `null` (i.e., `HasValue = false`). NOT 0.

```csharp
int? a = 0;
int? b = default;
Console.WriteLine(a == b);   // false! a has value 0, b is null
```

A common confusion. Be deliberate.

---

## When `default` collides with invariants

`default(MyStruct)` is **always reachable**. If your struct has invariants (e.g., "Currency must not be null"), `default` violates them silently:

```csharp
public readonly record struct Money(decimal Amount, string Currency);

Money m = default;
Console.WriteLine(m.Currency.ToUpper());   // 💥 NullReferenceException — Currency is null
```

You can't make the type "refuse default." Options:

1. Tolerate default:
```csharp
public string Currency { get; init; } = "USD";   // doesn't help — default skips init
```
(That field initializer only runs when the constructor is called — `default` skips it.)

2. Guard members:
```csharp
public string FormattedCurrency => Currency ?? "(unset)";
```

3. Use a class instead — constructors run, defaults are null (whole object), and you can detect that.

The fundamental tension: structs are zero-overhead but their default value is forced.

---

## `default!` — the null-forgiving form

With NRT, returning `default` from a method whose T might be a reference type triggers a warning (because `default` could be null):

```csharp
public T Get<T>(string key) {
    return default;   // ⚠ CS8603: possible null return — caller expected non-null T
}
```

If you know the caller will check, suppress:

```csharp
public T Get<T>(string key) {
    return default!;   // I know better; trust me
}
```

Or constrain to avoid the issue:

```csharp
public T? Get<T>(string key) {                  // T? — caller knows it might be null
    return default;
}
public T Get<T>(string key) where T : notnull { // T is non-null — caller can rely on it
    return ...;   // must return a real value
}
```

---

## Internals — how `default` compiles

For value types, `default(T)` produces a zero-initialized location:

```csharp
int n = default;
```

IL:
```il
.locals init (int32 V_0)
ldc.i4.0
stloc.0
```

For generic T:
```csharp
T t = default;
```

IL:
```il
.locals init (!!T V_0)
ldloca.s V_0
initobj !!T
ldloc.0
stloc.1
```

`initobj` zero-fills the storage. For value types this produces all-zero fields; for reference types it produces null. Same opcode handles both — the JIT specializes per T.

For specific reference types:

```csharp
string? s = default;
```

IL:
```il
ldnull
stloc.0
```

Just pushes null.

### Cost

- Zero. `default` is a constant load (and possibly an `initobj` for a struct).
- The JIT often inlines and constant-folds.

---

## Common patterns

### "Empty" semantics

```csharp
Span<int> empty = default;      // empty span
Memory<byte> empty2 = default;  // empty memory
Range r = default;              // 0..0
Guid g = default;               // all zeros
```

Many types treat `default` as "empty" / "none" naturally. Used everywhere in `Span<T>`-based APIs.

### `TryGet` returning default

```csharp
public bool TryGet<T>(string key, out T value) {
    value = default!;
    if (_store.TryGetValue(key, out var raw)) {
        value = (T)raw;
        return true;
    }
    return false;
}
```

`out` parameters always need assignment; `default!` is the canonical "I'll set it later" placeholder.

### Resetting a struct

```csharp
Counter c = new() { Value = 5 };
c = default;   // resets to zero
```

Quick reset to the default value.

### Default for a generic field

```csharp
public class Cache<T> {
    private T? _value;
    public T? Value => _value;
    public void Clear() => _value = default;
}
```

Resets the cache regardless of whether T is a class (null) or struct (zero).

---

## Common bugs

- **Using `default(T)` for "not set" with structs** — `default(int)` is 0, a perfectly valid count. Distinguish via `T?` (Nullable) or a sentinel.
- **NRT warnings on `default`** — suppress with `default!` only when you know the caller handles null.
- **Forgetting structs always default** — your invariants can be bypassed.
- **Comparing `int?` to `0` thinking it's `null`** — `(int?)0 == 0` is true; `(int?)null == null` is true; they're different. Use `n.HasValue` for the explicit check.
- **`default` for a tuple** — `(default, default)` is `(0, null)` for `(int, string)`. Same caveats apply per element.

---

## Performance

- `default` is essentially free — a constant load.
- For value types, `initobj` is one CPU instruction (a zeroing operation).
- No allocations, no method calls, no JIT magic. Just memory zeroing.

---

## When to use `default`

✓ In generic code where you don't know T.
✓ As a fallback in pattern matching arms.
✓ For "empty" values of types that treat default as empty (`Span<T>`, `Range`, `Guid`).
✓ When you want to express "the type's default value" rather than a specific zero/null.

✗ When you mean "0" or "null" specifically — be explicit for clarity.
✗ When your type's default violates invariants — design accordingly.

→ Continue to: [Questions.md](Questions.md)
