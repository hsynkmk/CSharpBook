# Generic Constraints

## What it is

A **constraint** restricts what types can be supplied as a generic type argument. Without constraints, the compiler treats T as `object` — you can do almost nothing with a value of type T besides hold it, pass it, and compare to null (sometimes).

Constraints unlock operations:

```csharp
// Unconstrained — can't call CompareTo
public T Max<T>(T a, T b) {
    return a.CompareTo(b) >= 0 ? a : b;   // ❌ compile error
}

// Constrained — can
public T Max<T>(T a, T b) where T : IComparable<T> {
    return a.CompareTo(b) >= 0 ? a : b;   // ✓
}
```

Each constraint says "T must be (at least) this kind of thing." The compiler verifies the constraint at every call site.

---

## All the constraint forms

| Constraint | Means |
|---|---|
| `where T : struct` | T is a non-nullable value type |
| `where T : class` | T is a reference type (or nullable reference for NRT context) |
| `where T : class?` | T is a reference type (NRT-friendly form for "could be nullable") |
| `where T : notnull` | T is non-null (could be value or non-nullable reference) |
| `where T : default` | T can be `default` (mostly for generic method overload disambiguation) |
| `where T : new()` | T has a public parameterless constructor |
| `where T : unmanaged` | T is an unmanaged type (no reference-type fields anywhere) |
| `where T : enum` | T is an enum type |
| `where T : Delegate` | T is a delegate type |
| `where T : System.Enum` | Same as `: enum` |
| `where T : BaseClass` | T inherits from (or equals) BaseClass |
| `where T : IInterface` | T implements IInterface |
| `where T : U` | T is convertible to U (where U is another type parameter) |
| `where T : allows ref struct` | (C# 13+) T can be a ref struct |

Multiple constraints on the same T are separated by commas:

```csharp
where T : class, IDisposable, new()
```

Order matters: `class`/`struct` first, then base classes, then interfaces, then `new()` last.

---

## `class` and `struct`

```csharp
public class RefCache<T> where T : class { }   // T must be a reference type
public class ValueBox<T> where T : struct { }  // T must be a value type
```

What they enable:
- `class`: T can be null. You can write `T? x = null`.
- `struct`: T cannot be null. `default(T)` is the all-zeros value. T? means `Nullable<T>`.

```csharp
public T? FindOrDefault<T>() where T : class { ... }    // T? = T or null
public T? FindOrDefault<T>() where T : struct { ... }   // T? = Nullable<T>
```

Without `class` or `struct`, generic `T?` has unclear semantics — the compiler asks for clarity.

---

## `new()`

```csharp
public T MakeOne<T>() where T : new() => new T();
```

T must have a public parameterless constructor. Lets you do `new T()`.

Restrictions:
- Doesn't accept constructors with parameters.
- Records with positional parameters don't satisfy `new()` unless you also declare a parameterless ctor.

```csharp
public record Person(string Name);
MakeOne<Person>();   // ❌ — no parameterless ctor

public record Person2 { public string Name { get; init; } = ""; }
MakeOne<Person2>();   // ✓
```

For more flexible "build a T" patterns, use a factory delegate instead:

```csharp
public T Make<T>(Func<T> factory) => factory();
Make(() => new Person("Alice"));
```

The delegate approach is also faster pre-.NET 6 (avoided `Activator.CreateInstance`).

---

## `notnull`

```csharp
public class Cache<TKey, TValue> where TKey : notnull { ... }
```

`TKey` must be **non-nullable** — either a value type (which can't be null) or a non-nullable reference type (`string`, not `string?`).

`Dictionary<TKey, TValue>` has this constraint. You can't use `string?` as a dictionary key — `Dictionary<string?, int>` is a compile error.

Useful for collections and APIs where null keys would be a bug.

---

## `unmanaged`

```csharp
public unsafe T Read<T>(byte* ptr) where T : unmanaged {
    return *(T*)ptr;
}
```

T must be a value type whose fields (recursively) are also value types — no references anywhere. This means T's bytes can be safely treated as a flat buffer, copied with memcpy, used in P/Invoke, written to native memory.

Allowed unmanaged types:
- All primitives (`int`, `double`, `byte`, ...).
- Pointers.
- Enums.
- Structs whose fields are all unmanaged (recursively).

Useful for:
- Low-level pointer code (`unsafe`).
- `Span<T>` + interop.
- `stackalloc T[size]` — only works for unmanaged T.
- Marshaling.

---

## `enum` and `Delegate`

```csharp
public string Name<T>(T value) where T : Enum =>
    Enum.GetName(typeof(T), value)!;

public T Combine<T>(T a, T b) where T : Delegate {
    return (T)Delegate.Combine(a, b)!;
}
```

C# 7.3+. Useful for utility methods over all enums or delegates without per-enum code.

---

## Base class / interface constraints

```csharp
public abstract class Animal { public abstract void Speak(); }

public class Zoo<T> where T : Animal {
    public void MakeSpeak(T animal) => animal.Speak();
}

public class Zoo2<T> where T : IComparable<T> { ... }
```

The constraint lets you call any member declared on the base class or interface.

Multiple interfaces allowed:

```csharp
public class Service<T> where T : class, IDisposable, ICloneable { ... }
```

---

## Type parameter as constraint (`where T : U`)

```csharp
public class Cache<TKey, TValue> {
    public void Add<TActualValue>(TKey key, TActualValue value)
        where TActualValue : TValue { ... }
}
```

Useful when one type parameter is constrained to be assignable to another. Niche but powerful for collection / repository scenarios.

---

## `allows ref struct` (C# 13+)

Previously, `Span<T>` and other ref structs couldn't be passed as generic arguments to most APIs. C# 13 added the ability to mark a generic parameter as **allowing** ref struct types:

```csharp
public T Process<T>(T input) where T : allows ref struct {
    // T can be a ref struct (like Span<byte>), or any other type
}

Process<Span<byte>>(stackalloc byte[100]);
```

Without `allows ref struct`, passing a `Span<T>` as a generic argument is forbidden because the compiler can't prove the span won't escape.

This enables generic algorithms that work over both regular types AND ref structs.

---

## Combining constraints

```csharp
public class Builder<T>
    where T : class, IValidatable, IEquatable<T>, new()
{
    // T is a reference type, implements IValidatable and IEquatable<T>, has a parameterless ctor.
}
```

Order requirement:
1. `class` or `struct` (at most one).
2. Base class (at most one).
3. Interfaces (any number).
4. `new()` (at most one, must be last).

Some are mutually exclusive:
- `struct` and `class` — pick one.
- `struct` and `new()` — `new()` is redundant (every struct has implicit parameterless ctor).

---

## Inheritance of constraints

Inheriting a generic class **passes through** the constraints:

```csharp
public class Base<T> where T : IComparable<T> { ... }

public class Derived<T> : Base<T> { }
// implicit: where T : IComparable<T> (compiler enforces)
```

You can add more constraints in the derived class (`where T : IComparable<T>, new()`), but you can't loosen them.

Generic methods overriding base methods must repeat constraints implicitly — they're inherited from the base declaration.

---

## How constraints help inference

Inference uses constraints as hints:

```csharp
public T Read<T>(string key) where T : IParsable<T> { ... }

int n = Read<int>("42");   // explicit
// var n = Read("42");      // ❌ — return-only T can't infer
```

Inference still needs T to appear in arguments to deduce it. Constraints alone don't enable inference.

---

## What constraints unlock — practical examples

### Method calls

```csharp
where T : IComparable<T>  // allows a.CompareTo(b)
where T : IEquatable<T>   // allows a.Equals(b) without boxing
where T : IFormattable    // allows a.ToString(format, culture)
where T : IDisposable     // allows using t / t.Dispose()
```

### Constructor

```csharp
where T : new()           // allows new T()
```

### Conversions

```csharp
where T : Animal          // allows T → Animal upcast
where T : class           // allows T → object (already implicit) AND `T x = null;`
where T : struct          // allows default(T), Nullable<T> (T?)
```

### Stack ops

```csharp
where T : unmanaged       // allows stackalloc T[size], unsafe pointer ops
```

---

## Internals — how constraints affect IL and the JIT

### Constraint metadata

Each generic type parameter has a constraint set encoded in metadata:

```il
.class public auto ansi sealed Box`1<class T (class [System.Runtime]System.IComparable`1<!T>)>
```

The runtime verifies at type construction (when `Box<MyType>` is created) that `MyType` satisfies the constraints. Mismatch → `TypeLoadException`.

### Constrained call

When you call an interface method on a generic T constrained to that interface, the compiler emits `constrained.` prefix in IL:

```csharp
public bool AreEqual<T>(T a, T b) where T : IEquatable<T> =>
    a.Equals(b);
```

```il
ldarga.s a
ldarg.1
constrained. !!T
callvirt instance bool class [System.Runtime]System.IEquatable`1<!!T>::Equals(!0)
```

`constrained.` tells the JIT:
- If T is a value type and implements the method directly, call it without boxing.
- If T is a value type and the method is inherited via boxing (`object.Equals`), box first.
- If T is a reference type, just `callvirt` normally.

This is **the magic** that makes generic code over structs allocation-free.

### Specialization

The JIT generates one machine-code body per **value-type T**. So `AreEqual<int>` and `AreEqual<double>` each get their own native code; the IL is reused. For reference-type T, one body is shared.

### Constraint-driven inlining

Constraints help the JIT make stronger assumptions:

```csharp
where T : struct, IEquatable<T>
```

→ T is sealed (structs are implicitly sealed). The JIT can devirtualize `T.Equals(T)` → direct call → inlinable.

For `where T : class`, virtual dispatch is harder to devirtualize unless PGO observes a stable type.

---

## Common patterns

### Constrained equality

```csharp
public bool AreEqual<T>(T a, T b) where T : IEquatable<T> =>
    a.Equals(b);
```

No boxing, direct call.

### Constrained comparison

```csharp
public T Min<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) <= 0 ? a : b;
```

### Constrained creation

```csharp
public TBag NewBag<TBag, TItem>() where TBag : ICollection<TItem>, new() =>
    new TBag();

var list = NewBag<List<int>, int>();
```

### Optional-only via Nullable

```csharp
public TValue? TryGet<TValue>() where TValue : struct {
    return _store is { } v ? (TValue?)v : null;
}
```

Notice the constraint clarifies what `TValue?` means.

### Modern generic math

```csharp
public T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total = total + x;
    return total;
}
```

`INumber<T>` is a constraint that gives you arithmetic operators. [§06](06-GenericMath.md).

---

## When you might NOT want a constraint

For very flexible code (e.g., a `Cache<T>` that just stores), you may genuinely not need constraints. Then T is treated as `object` and the only operations are storage, copy, equality via `EqualityComparer<T>.Default`.

`Func<T>`, `Action<T>`, and `IEnumerable<T>` deliberately have no constraints — they wrap T without operating on it.

Rule of thumb: **don't constrain unless you need to**. Over-constraining limits who can use your API.

---

## Common bugs

- **Constraint order mistakes** — `where T : new(), IDisposable` doesn't compile; should be `where T : IDisposable, new()`.
- **Conflating `class` and `notnull`** — `class` means "reference type, may be nullable in NRT context" (or you must write `class?` explicitly with NRT). `notnull` means "guaranteed not null."
- **Trying to call `==` on unconstrained T** — doesn't compile. Use `EqualityComparer<T>.Default.Equals` or constrain to `IEquatable<T>`.
- **`new T(...)` with arguments** — `new()` doesn't allow parameters. Use a factory.
- **Constraints don't aid return-only inference** — must specify T explicitly.
- **Forgetting `: U` chain constraint** when the API needs the relationship.

---

## Performance

- Constraints enable the JIT to **devirtualize** more aggressively.
- `where T : struct, ISomething<T>` is the most-specializable. Best performance for hot paths.
- `where T : class` lets the JIT eliminate value-type code paths.
- `where T : new()` is fast in .NET 6+ (was slow before due to Activator.CreateInstance).

→ Next: [04-Variance.md](04-Variance.md)
