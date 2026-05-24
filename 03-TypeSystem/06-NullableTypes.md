# Nullable Types

## What it is

C# has **two distinct nullable systems**, and conflating them causes confusion:

1. **`Nullable<T>`** (alias: `T?`) — for **value types**. A struct wrapping a value plus a "has value" flag. Runtime construct.
2. **Nullable Reference Types** (NRT) — for **reference types**. Compiler annotation `string?` warns about null misuse. Compile-time only.

Both let `null` flow through types that would otherwise reject it (value types) or accept it silently (reference types). The mechanics, runtime behavior, and design philosophy differ.

This file covers both. The deeper NRT story is in [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md).

---

## `Nullable<T>` — for value types

```csharp
int n = null;     // ❌ value types can't be null
int? n = null;    // ✓ — Nullable<int>, currently has no value
```

`int?` is shorthand for `System.Nullable<int>`, a struct:

```csharp
public struct Nullable<T> where T : struct {
    public T Value { get; }
    public bool HasValue { get; }
    public T GetValueOrDefault();
    public T GetValueOrDefault(T defaultValue);
    // ...
}
```

It wraps:
- The value of type T.
- A boolean flag, "has value" or not.

```csharp
int? n = 42;
Console.WriteLine(n.HasValue);     // true
Console.WriteLine(n.Value);        // 42

int? m = null;
Console.WriteLine(m.HasValue);     // false
Console.WriteLine(m.Value);        // 💥 InvalidOperationException
Console.WriteLine(m.GetValueOrDefault(0));   // 0
```

Use cases:
- Optional values where "0 / empty / default" isn't a meaningful "not set."
- Database columns that can be NULL.
- Method return when "couldn't compute" is a legitimate outcome.
- Optional parameters where defaults aren't expressible.

### Conversions

Implicit conversion from `T` to `T?`:

```csharp
int? n = 5;     // implicit from int → int?
```

Explicit cast from `T?` to `T`:

```csharp
int? n = 5;
int x = (int)n;   // explicit — throws if n is null
int y = n ?? 0;   // null-coalescing — safe
int? m = null;
int z = m.GetValueOrDefault();   // 0
```

### Operators

Most operators "lift" — they accept and return nullable. If any operand is null, the result is null:

```csharp
int? a = 5;
int? b = null;
int? sum = a + b;     // null (one operand is null)
int? sum2 = a + 3;    // 8 — both sides effectively have values
bool eq = a == b;     // false (one null, one not)
bool eq2 = a == 5;    // true
```

Subtle: `bool? == bool?` is itself `bool?` (three-valued logic). For boolean tests in `if`:

```csharp
bool? maybeReady = ...;
if (maybeReady == true) { ... }     // explicitly tests for true
if (maybeReady ?? false) { ... }    // null treated as false
```

---

## Pattern matching with `Nullable<T>`

```csharp
int? n = ...;

if (n is int value) { Console.WriteLine(value); }     // extracts value
if (n is null) { ... }
if (n is not null) { ... }
if (n is { } value) { ... }                            // "has value" pattern

string Describe(int? n) => n switch {
    null => "nothing",
    < 0 => "negative",
    0 => "zero",
    > 0 => "positive"
};
```

The compiler unwraps `Nullable<T>` automatically in pattern matching. Very clean.

---

## Nullable Reference Types (NRT)

A C# 8+ feature that tracks null-state at the **type system / compiler level** for reference types. Runtime sees regular references; the compiler sees annotations.

Enable per-project in csproj:

```xml
<Nullable>enable</Nullable>
```

Or per-file with `#nullable enable` at the top.

With NRT enabled:

```csharp
string s = "hello";   // non-null by declaration
s = null;             // ⚠ warning: assigning null

string? s2 = null;    // explicitly nullable
s2 = "hello";
s2.Length;            // ⚠ warning: dereference of possibly null

if (s2 != null) {
    s2.Length;        // OK — compiler tracked the null check
}
```

The `?` suffix marks the type as **nullable**. Without `?`, the compiler assumes non-null and warns when null might flow in.

### Modes

`Nullable` can be:
- `enable` — annotations and warnings (typical for new code).
- `disable` — no annotations, no warnings (pre-C# 8 behavior, legacy).
- `annotations` — annotations honored, no warnings (gradual migration).
- `warnings` — warnings emitted but `?` is ignored (rare).

Project-wide is the norm. Mixing files with different modes works for migration.

### What the `?` actually means

`string?` and `string` have the **same runtime type** — `System.String`. The difference is purely a compile-time annotation. There's NO `Nullable<string>` runtime wrapper.

```csharp
string? s = null;
Type t = s?.GetType() ?? typeof(string);   // System.String — same type as non-nullable string
```

This means:
- Reflection sees no difference.
- Cast `(string)null` vs `(string?)null` behaves identically at runtime.
- The compiler is your only guardrail. Override with `!` if you know better.

### The null-forgiving operator `!`

Tell the compiler "I know better; suppress the warning":

```csharp
string? maybe = GetMaybe();
int len = maybe!.Length;   // I promise it's not null
```

Use sparingly. Every `!` is a place a bug could hide. Common legitimate uses:
- After a "must be non-null" check the compiler can't see (e.g., a method named `IsNotNull`).
- Configuration values you've validated elsewhere.

For defensive coding:
```csharp
ArgumentNullException.ThrowIfNull(maybe);
// Compiler now knows maybe is non-null past this point.
int len = maybe.Length;   // no warning
```

### Nullable annotations on generics

```csharp
public T? FirstOrDefault<T>() { ... }   // ? means: nullable if T is a reference type, structurally null if value type
```

The `T?` syntax for unconstrained generics has slightly different semantics depending on whether T is a class or struct:

- For reference T: returns `null`.
- For value T: returns... well, this is more complex. In C# 9+ `T?` on unconstrained T means "default-able" — see Chapter 04 §07.

---

## Flow analysis

The compiler tracks null state across statements:

```csharp
string? s = GetMaybe();
if (s != null) {
    s.Length;   // OK — compiler knows s isn't null here
}
s.Length;       // ⚠ — back to "possibly null"

string? other = GetMaybe() ?? "default";
other.Length;   // OK — other is provably non-null
```

The compiler is conservative — if it can't prove non-null, it warns. Some helper attributes let library authors annotate their methods:

```csharp
public static bool TryGet([NotNullWhen(true)] out string? value) { ... }

// At call site:
if (TryGet(out var v)) {
    v.Length;   // OK — NotNullWhen(true) tells compiler v is non-null when TryGet returns true
}
```

Other attributes: `[NotNull]`, `[MaybeNull]`, `[NotNullIfNotNull]`, `[DisallowNull]`, `[AllowNull]`, `[DoesNotReturn]`, `[DoesNotReturnIf]`. They're how the BCL signals nuances the compiler can't infer.

[Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md) goes into all of this.

---

## Null-safety operators

| Operator | Meaning |
|---|---|
| `?.` | Null-conditional member access (skip if null, return null) |
| `?[]` | Null-conditional indexing |
| `?.=` | Null-conditional assignment (C# 14) — assigns only if non-null |
| `??` | Null-coalescing (right side if left is null) |
| `??=` | Null-coalescing assignment |
| `!` | Null-forgiving (suppress warning) |

Examples:
```csharp
user?.Profile?.Email          // null if any link in the chain is null
arr?[0]                       // null if arr is null
user?.Name = "Alice"          // C# 14: no-op if user is null
name ?? "anonymous"           // fallback if name is null
cached ??= LoadFromDb()       // assign only if cached is null
maybe!.Length                 // trust me, it's not null
```

These dramatically reduce defensive code. Used everywhere in modern C#.

---

## `Nullable<T>` and NRT interaction

`int?` (Nullable&lt;int&gt;) and `int` are **different runtime types**. `string?` and `string` are the **same**. So:

```csharp
int? x = 5;
int y = x;     // ❌ compile error — explicit conversion required
int y2 = (int)x;   // OK

string? s = "hi";
string t = s;      // ⚠ NRT warning, but compiles (same runtime type)
```

Mixing them in generics requires care. `T?` for an unconstrained T means different things depending on whether T is a class or struct:

```csharp
public T? FirstOrDefault<T>() {
    // If T is string: returns null.
    // If T is int: returns... well, complicated.
}
```

For best results, constrain:

```csharp
public T? FirstOrDefaultClass<T>() where T : class { ... }     // T? = T or null
public T? FirstOrDefaultStruct<T>() where T : struct { ... }   // T? = Nullable<T>
```

---

## Internals — `Nullable<T>` layout and behavior

A `Nullable<T>` is a struct holding:

```
[ T value      | bool hasValue ]
```

Total size: `sizeof(T) + 1 byte` (plus alignment padding). For `int?`: 8 bytes typically (4 for int, 1 for bool, 3 padding).

`HasValue == false` means `value` is irrelevant garbage — typically zero (default).

### Boxing behavior

`Nullable<T>` has **special boxing semantics**:

- Boxing a `Nullable<T>` with `HasValue = true` produces a boxed `T` (NOT a boxed `Nullable<T>`).
- Boxing a `Nullable<T>` with `HasValue = false` produces `null` (not a boxed zero).

```csharp
int? n = 5;
object o = n;          // o references a boxed int (5), not a boxed Nullable<int>
Console.WriteLine(o.GetType());   // System.Int32, NOT System.Nullable<...>

int? m = null;
object o2 = m;         // o2 is null, not a boxed Nullable<int>
Console.WriteLine(o2 is null);  // true
```

This is hard-coded into the runtime. The behavior makes `Nullable<T>` interop naturally with `object`-based APIs.

### NRT IL

`string?` and `string` compile to **the same** IL types. The `?` is metadata: `[NullableContext(2)]` / `[Nullable(2)]` attributes on the assembly, types, and members. The CLR ignores them; only the C# compiler reads them.

Decompiled C# code from another language won't see the annotations. F# and VB.NET have varying support.

### NRT compiler tricks

The compiler does **flow analysis** at the IL level — tracking whether each variable, parameter, or field is "definitely null," "definitely non-null," "maybe null," or "unknown." It does this per statement, per branch.

```csharp
string? s = GetMaybe();
if (s is null) return;
s.Length;        // compiler: "definitely non-null past the early return"
```

The cost is purely compile time. Runtime sees no extra work.

---

## Common patterns

### Optional return

```csharp
public User? FindUser(int id) {
    // ...
    return user;  // or null
}

if (FindUser(1) is { } user) {
    Console.WriteLine(user.Name);
}
```

### "Try" methods returning `T?`

Modern alternative to the classic `TryGet(out T)`:

```csharp
public T? TryFind<T>() { ... }

if (TryFind<User>() is { } u) {
    // ...
}
```

### Default value pattern

```csharp
int? maybe = ParseOrNull(input);
int value = maybe ?? -1;
```

### Lazy initialization

```csharp
private List<string>? _items;
public List<string> Items => _items ??= LoadItems();
```

### Defensive guard

```csharp
public void Process(string? input) {
    ArgumentNullException.ThrowIfNull(input);
    // From here on, compiler knows input is non-null
    Console.WriteLine(input.Length);
}
```

### Chained navigation

```csharp
var country = order?.Customer?.Address?.Country ?? "Unknown";
```

Old-school code would have nested if-statements; here it's a single readable expression.

---

## Common bugs

- **Forgetting `Nullable<T>` is a value type** — `if (n == null)` works because of lifted operators, but `default(int?)` is null, not zero.
- **Using `.Value` without checking** — throws `InvalidOperationException` on null. Prefer `GetValueOrDefault()` or `??`.
- **Mixing NRT and reflection** — reflection bypasses NRT. Don't trust `?` annotations from reflective code paths.
- **`obj is not null` vs `obj != null`** — usually equivalent, but with overloaded `==` operators they can differ. `is not null` always uses reference/structural equality.
- **Suppressing with `!` everywhere** — defeats the point of NRT. Use it only when you know something the compiler can't.
- **Returning `T?` from generic methods** — semantics depend on constraints; subtle for unconstrained T.

---

## When to use which

**Use `Nullable<T>` (`T?` on value types):**
- Optional value-type fields, return values, parameters.
- Database column representations.

**Use NRT (`T?` on reference types):**
- Always! It's a compile-time check that catches a huge class of bugs. Enable in every new project.

**Use `!`:**
- Only when you've validated null-ness in a way the compiler can't see.

---

## Performance

- `Nullable<T>` is essentially zero overhead — one extra byte, no allocation.
- NRT is purely compile-time. **Zero runtime cost.** No checks, no boxes, no anything.
- The `?.` chain operator is just an inlined null check — no method call magic.

→ Next: [07-BoxingUnboxing.md](07-BoxingUnboxing.md)
