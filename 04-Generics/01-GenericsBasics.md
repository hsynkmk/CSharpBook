# Generics Basics

## What it is

A **generic** type or method is one **parameterized over types**. Instead of writing one `IntList`, one `StringList`, one `OrderList`, you write `List<T>` once and the compiler/runtime specializes it for any T you supply.

```csharp
public class Box<T> {
    public T Value { get; set; }
    public Box(T value) { Value = value; }
}

var intBox = new Box<int>(42);
var stringBox = new Box<string>("hello");
var personBox = new Box<Person>(alice);

Console.WriteLine(intBox.Value);     // 42
Console.WriteLine(stringBox.Value);  // hello
```

`Box<int>`, `Box<string>`, and `Box<Person>` are **three distinct types**, but they share one source definition.

Generics were added in C# 2.0 (2005) — they predate basically every other modern feature. Almost every type you use day-to-day (`List<T>`, `Dictionary<K,V>`, `Task<T>`, `Func<...>`, `Span<T>`) is generic.

---

## Why they exist

Without generics, you have three bad options:

1. **Object collections** — `ArrayList`, `Hashtable`. Boxes value types, requires casting on retrieval, no compile-time type safety.

2. **Per-type duplication** — `IntList`, `StringList`, `OrderList`. Code duplication, no shared logic, drifts over time.

3. **Code generation** — error-prone, build-time complexity.

Generics solve all three:

```csharp
// Pre-generics (ArrayList) — boxing + casting + no safety
ArrayList list = new();
list.Add(5);
list.Add("hello");    // compiles, but type-wrong
int n = (int)list[0]; // unboxing + cast — could throw

// Generics
List<int> list = new();
list.Add(5);
list.Add("hello");    // ❌ compile error
int n = list[0];       // no cast, no boxing
```

Type safety + performance + reuse. The trifecta.

---

## Declaring a generic type

```csharp
public class Stack<T> {
    private T[] _items = new T[16];
    private int _count;

    public void Push(T item) {
        if (_count == _items.Length) Array.Resize(ref _items, _items.Length * 2);
        _items[_count++] = item;
    }

    public T Pop() {
        if (_count == 0) throw new InvalidOperationException();
        return _items[--_count];
    }

    public int Count => _count;
}

var s = new Stack<int>();
s.Push(1);
s.Push(2);
Console.WriteLine(s.Pop());  // 2
```

`T` is a **type parameter**. By convention:
- Single-letter `T` for the most-common case.
- Descriptive `TKey`, `TValue`, `TResult` when multiple parameters or context matters.
- Always uppercase, starting with `T`.

You can have many:
```csharp
public class Pair<TFirst, TSecond> {
    public TFirst First { get; }
    public TSecond Second { get; }
    public Pair(TFirst first, TSecond second) { First = first; Second = second; }
}

var pair = new Pair<string, int>("age", 30);
```

`Dictionary<TKey, TValue>` is the canonical two-parameter generic.

---

## Generic methods

A method can be generic without the containing type being generic:

```csharp
public static class Util {
    public static T Last<T>(T[] arr) => arr[arr.Length - 1];
}

int last = Util.Last(new[] { 1, 2, 3 });        // T inferred as int → 3
string lastStr = Util.Last(new[] { "a", "b" }); // T inferred as string → "b"
```

Type arguments can usually be **inferred** from the arguments — the compiler figures out T. You can also specify explicitly:

```csharp
int last = Util.Last<int>(new[] { 1, 2, 3 });
```

[§02](02-GenericMethods.md) covers when inference works and when it fails.

---

## What "generic" really means

Every generic type has two faces:

- **Open generic** — the unconstructed form, `List<T>` or `Dictionary<TKey, TValue>`. Has type parameters left unspecified.
- **Constructed (closed) generic** — every parameter substituted with a real type, `List<int>` or `Dictionary<string, User>`. This is what you actually instantiate.

```csharp
typeof(List<>);              // open: 1 type parameter still placeholder
typeof(List<int>);           // constructed
typeof(Dictionary<,>);       // open: 2 parameters
typeof(Dictionary<string, int>);  // constructed
```

In code, you write constructed forms (`List<int>`). Open generics appear in reflection (`typeof(List<>)`), constraints, and certain compiler scenarios.

---

## Type inference

For generic methods, the compiler can usually figure out T from the arguments:

```csharp
public T First<T>(IEnumerable<T> source) => source.First();

int n = First(new[] { 1, 2, 3 });     // T = int (inferred)
string s = First(new[] { "a", "b" }); // T = string
```

When inference can't decide, you must specify:

```csharp
public T Read<T>() => default!;

// var x = Read();      // ❌ — no argument from which to infer T
var x = Read<int>();    // ✓
```

For generic **types**, inference doesn't happen (with one notable exception): you must specify the type arguments at construction:

```csharp
var list = new List<int>();   // can't omit <int>
```

The exception is target-typed `new` (C# 9+):

```csharp
List<int> list = new();   // target type is List<int>, so compiler knows T
```

---

## Constraints

Without constraints, T is treated as `object`. You can't call any specific methods on a T value, and you can't compare with `==` (except to `null` for reference types).

To do more, **constrain** T to types meeting a requirement:

```csharp
public T Largest<T>(IEnumerable<T> source) where T : IComparable<T> {
    using var e = source.GetEnumerator();
    if (!e.MoveNext()) throw new InvalidOperationException();
    T best = e.Current;
    while (e.MoveNext()) {
        if (e.Current.CompareTo(best) > 0) best = e.Current;
    }
    return best;
}
```

`where T : IComparable<T>` says: T must implement IComparable&lt;T&gt;. Now we can call `CompareTo`.

Constraint kinds:
- `where T : class` — T is a reference type.
- `where T : struct` — T is a value type.
- `where T : new()` — T has a public parameterless constructor.
- `where T : notnull` — T is non-nullable.
- `where T : unmanaged` — T has no reference-type fields (suitable for `stackalloc`, P/Invoke).
- `where T : SomeBaseClass` — T inherits from a class.
- `where T : ISomeInterface` — T implements an interface.
- `where T : enum` — C# 7.3+, T is an enum.
- `where T : Delegate` — T is a delegate type.
- `where T : U` — T is "convertible to" U (where U is another type parameter).
- `where T : allows ref struct` — C# 13+, T can be a ref struct.

Multiple constraints separated by commas; the order is: class/struct/etc. first, base class next, interfaces, then `new()` last.

[§03](03-Constraints.md) covers constraints in depth.

---

## Where generics are everywhere

Look at .NET's surface area through a "generic" lens:

```csharp
// Collections
List<int>, Dictionary<K,V>, HashSet<T>, Queue<T>, Stack<T>, PriorityQueue<T,U>

// Async
Task<T>, ValueTask<T>, IAsyncEnumerable<T>

// Comparisons and equality
IComparable<T>, IEquatable<T>, IEqualityComparer<T>, Comparer<T>

// Conversions
IParsable<T>, ISpanParsable<T>, IFormattable

// Functional
Func<T, TResult>, Action<T>, Predicate<T>

// LINQ
IEnumerable<T>, IQueryable<T>, IGrouping<TKey, T>

// Memory
Span<T>, ReadOnlySpan<T>, Memory<T>, ArrayPool<T>

// Concurrent
ConcurrentDictionary<K,V>, ConcurrentQueue<T>, Channel<T>

// Generic math (C# 11+)
INumber<T>, IAdditionOperators<TL, TR, TR>, IEquatable<T>
```

Generics are not "an advanced feature" — they're the **default vocabulary** of modern C#.

---

## Internals — what happens at runtime

This is where generics get interesting. Different languages handle it differently:

- **Java**: type erasure — `List<T>` becomes `List<Object>` at runtime; T is gone.
- **C++**: templates — code generated per instantiation, no runtime knowledge of templates.
- **C#**: **reified generics** — T survives at runtime; the JIT can specialize based on it.

### Reification

In C#, `List<int>` and `List<string>` are **distinct types at runtime**. You can ask:

```csharp
Console.WriteLine(typeof(List<int>) == typeof(List<string>));  // false
typeof(List<int>).GenericTypeArguments;  // [System.Int32]
```

The runtime knows T. Reflection, serializers, and dependency injection rely on this.

### Code sharing

At runtime, the JIT decides whether to share code between instantiations:

- **Reference type T**: one shared body. `List<string>`, `List<User>`, `List<Order>` all run the **same** compiled methods. The body uses generic dispatch (a "type token" passed implicitly) to access type-specific operations.

- **Value type T**: per-T specialization. `List<int>`, `List<double>`, `List<MyStruct>` each get their **own** compiled methods. This is why generic code is fast for value types — no boxing, no type tokens, direct field access.

This is called **shared code for reference types, monomorphization for value types**. It's the best of both worlds:
- Memory efficiency for ref-type instantiations.
- Maximum performance for value-type instantiations.

### Method tables and `MethodTable`

Each constructed generic type has its own method table. For value-type T's, the layout is specialized — `List<int>`'s backing array is a true `int[]`, not an `object[]`. For ref-type T's, the layout uses pointers (`object[]` under the hood).

This is why this works:

```csharp
List<int> nums = new();
nums.Add(5);
// In IL: callvirt List`1<int32>::Add(!0)
// At runtime: directly stores 5 into the inline int slot. No box.
```

vs. classic:

```csharp
ArrayList nums = new();
nums.Add(5);
// In IL: callvirt ArrayList::Add(object)
// At runtime: box(5) → store reference. Plus alloc.
```

### Reflection on generics

```csharp
Type t = typeof(List<int>);
Console.WriteLine(t);                                  // System.Collections.Generic.List`1[System.Int32]
Console.WriteLine(t.IsGenericType);                    // true
Console.WriteLine(t.GetGenericTypeDefinition() == typeof(List<>));  // true
Console.WriteLine(string.Join(",", t.GetGenericArguments().Select(a => a.Name)));  // Int32
```

You can construct generic types at runtime:

```csharp
Type listType = typeof(List<>).MakeGenericType(typeof(string));
object instance = Activator.CreateInstance(listType)!;
```

Used heavily by serializers, DI containers, ORMs.

### Generic specialization in IL

```csharp
public class Box<T> { public T Value; public Box(T v) { Value = v; } }
```

In IL, T appears as `!T`:

```il
.class public auto ansi sealed Box`1<T>
{
    .field public !T Value
    .method public hidebysig specialname rtspecialname instance void .ctor(!T v) {
        ldarg.0
        ldarg.1
        stfld !0 class Box`1<!T>::Value
        ret
    }
}
```

At runtime the JIT substitutes T with the actual type and generates (or shares) machine code accordingly.

---

## When generics are the wrong tool

✗ When there's **only one** concrete T. Just write the non-generic version.
✗ When T is `object` or always one type — that's a sign your generic isn't needed.
✗ When constraints become ridiculous (`where T : IFoo, IBar, IComparable<T>, new()`) — consider an interface or abstract base.
✗ When the JIT can't specialize meaningfully (e.g., reflection-based code paths).

---

## Common patterns

### Container

```csharp
public class Cache<TKey, TValue> where TKey : notnull {
    private readonly Dictionary<TKey, TValue> _store = new();
    public TValue this[TKey key] { get => _store[key]; set => _store[key] = value; }
    public bool TryGet(TKey key, out TValue value) => _store.TryGetValue(key, out value!);
}
```

### Factory

```csharp
public class Builder<T> where T : new() {
    private readonly List<Action<T>> _steps = new();
    public Builder<T> Configure(Action<T> step) { _steps.Add(step); return this; }
    public T Build() {
        var t = new T();
        foreach (var step in _steps) step(t);
        return t;
    }
}
```

### Generic interface

```csharp
public interface IRepository<TEntity, TKey> {
    Task<TEntity?> GetAsync(TKey id);
    Task SaveAsync(TEntity entity);
    Task DeleteAsync(TKey id);
}

public class UserRepo : IRepository<User, int> { ... }
public class OrderRepo : IRepository<Order, Guid> { ... }
```

### Generic delegate

```csharp
public delegate TResult Mapper<in T, out TResult>(T input);

Mapper<int, string> toString = n => n.ToString();
Mapper<string, int> length = s => s.Length;
```

---

## Common bugs

- **Forgetting constraints**, then trying to compare or call methods on T. The compiler enforces what you didn't constrain.
- **Inferring the wrong T** with ambiguous overloads. Specify explicitly.
- **`default(T)`** — for reference types it's null; for `int?` it's null; for `int` it's 0. Behavior differs depending on T. Use `default!` with NRT when you know T is reference and you want a placeholder.
- **Boxing in generic code with non-generic interfaces** — see [§07 Boxing](../03-TypeSystem/07-BoxingUnboxing.md).
- **`new T()` performance** — pre-.NET 6 was slow (reflection). Modern JIT specializes it well. Still — factory delegates are an alternative when speed matters.

---

## Performance

- Generics over **value types** = near-zero overhead. The JIT specializes, eliminating boxing and virtual dispatch.
- Generics over **reference types** = small overhead from shared-code dispatch (a few cycles per call). Negligible in normal code.
- Constructing types at runtime via reflection (`MakeGenericType`) = slow. Cache the constructed types.
- `where T : struct` lets the JIT do best-case devirtualization on interface calls.

→ Next: [02-GenericMethods.md](02-GenericMethods.md)
