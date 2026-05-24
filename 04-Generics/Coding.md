# Chapter 04 — Coding Problems

> 12 hands-on problems. Cover basics, constraints, variance, static abstract, generic math.

---

## Problem 1 — Build your own `Box<T>` and `Pair<T,U>`

Implement two generic containers: a `Box<T>` with a single value, and a `Pair<T,U>` with two values of different types. Both should support deconstruction.

<details><summary>Solution</summary>

```csharp
public class Box<T> {
    public T Value { get; init; }
    public Box(T value) { Value = value; }
    public void Deconstruct(out T value) { value = Value; }
}

public class Pair<TFirst, TSecond> {
    public TFirst First { get; init; }
    public TSecond Second { get; init; }
    public Pair(TFirst first, TSecond second) { First = first; Second = second; }
    public void Deconstruct(out TFirst first, out TSecond second) {
        first = First; second = Second;
    }
}

// Use:
var (n) = new Box<int>(42);
var (name, age) = new Pair<string, int>("Alice", 30);
```

Modern version using records (one-liner):
```csharp
public record Box<T>(T Value);
public record Pair<TFirst, TSecond>(TFirst First, TSecond Second);
```

</details>

---

## Problem 2 — Generic `Max` with a constraint

Write a generic `Max<T>` that returns the largest of N values, using the appropriate constraint.

<details><summary>Solution</summary>

```csharp
public static T Max<T>(params T[] items) where T : IComparable<T> {
    if (items.Length == 0) throw new ArgumentException("at least one needed");
    T best = items[0];
    for (int i = 1; i < items.Length; i++) {
        if (items[i].CompareTo(best) > 0) best = items[i];
    }
    return best;
}

int big = Max(3, 7, 1, 8, 5);              // 8
string s = Max("apple", "banana", "cherry"); // "cherry"
```

With C# 14 `params ReadOnlySpan<T>` for zero allocation:

```csharp
public static T Max<T>(params ReadOnlySpan<T> items) where T : IComparable<T> {
    // same logic
}
```

</details>

---

## Problem 3 — Generic `Cache<TKey, TValue>`

Implement a thread-safe cache using `ConcurrentDictionary<TKey, TValue>`. Constrain TKey to be `notnull`. Provide `GetOrAdd` with a factory.

<details><summary>Solution</summary>

```csharp
public class Cache<TKey, TValue> where TKey : notnull {
    private readonly ConcurrentDictionary<TKey, TValue> _store = new();

    public TValue GetOrAdd(TKey key, Func<TKey, TValue> factory) =>
        _store.GetOrAdd(key, factory);

    public bool TryGet(TKey key, out TValue value) =>
        _store.TryGetValue(key, out value!);

    public void Set(TKey key, TValue value) => _store[key] = value;
    public void Clear() => _store.Clear();
    public int Count => _store.Count;
}

// Usage:
var cache = new Cache<int, User>();
var u = cache.GetOrAdd(42, id => LoadUser(id));
```

ConcurrentDictionary handles the thread safety. `notnull` prevents accidental `null` keys.

</details>

---

## Problem 4 — Spot the bug

```csharp
public class Counter<T> where T : IComparable<T> {
    private List<T> _items = new();
    public void Add(T item) {
        foreach (var i in _items) if (i.CompareTo(item) > 0) return;
        _items.Add(item);
    }
}
```

Two issues. Find them.

<details><summary>Answer</summary>

1. **Wrong semantic**: this skips Add if any existing item compares **greater than** the new one. Probably intent was "add only if not already present" — should be `i.CompareTo(item) == 0`.

2. **Boxing for value types via `IComparable<T>` overload**: actually no, `IComparable<T>.CompareTo(T)` is the strongly typed version — no box. But if you accidentally used `IComparable.CompareTo(object)` (non-generic), each comparison would box T. The generic constraint here is correct.

The real fix:
```csharp
public void AddUnique(T item) {
    foreach (var i in _items) if (i.CompareTo(item) == 0) return;   // duplicate
    _items.Add(item);
}
```

Or use a `HashSet<T>` if order doesn't matter (O(1) Add+Contains).

</details>

---

## Problem 5 — Build a covariant `IReadOnlyContainer<out T>`

Define a covariant read-only collection interface and a concrete implementation. Demonstrate variance by assigning `IReadOnlyContainer<string>` to `IReadOnlyContainer<object>`.

<details><summary>Solution</summary>

```csharp
public interface IReadOnlyContainer<out T> {
    int Count { get; }
    T this[int index] { get; }
}

public class ReadOnlyList<T> : IReadOnlyContainer<T> {
    private readonly List<T> _items;
    public ReadOnlyList(IEnumerable<T> items) { _items = new(items); }
    public int Count => _items.Count;
    public T this[int index] => _items[index];
}

// Demo:
IReadOnlyContainer<string> strs = new ReadOnlyList<string>(new[] { "a", "b" });
IReadOnlyContainer<object> objs = strs;   // covariance: ✓
Console.WriteLine(objs[0]);                // "a" (as object)
```

If you added a setter `T this[int] { get; set; }`, the compiler would refuse `out T` — variance requires output-only.

</details>

---

## Problem 6 — Generic `Median` using INumber<T>

Write a `Median<T>` that works for any numeric type. Sort the values and return the middle (or average of two middles).

<details><summary>Solution</summary>

```csharp
using System.Numerics;

public static T Median<T>(IEnumerable<T> source) where T : INumber<T> {
    var arr = source.OrderBy(x => x).ToArray();
    if (arr.Length == 0) throw new InvalidOperationException();
    int mid = arr.Length / 2;
    if (arr.Length % 2 == 1) return arr[mid];
    T two = T.One + T.One;
    return (arr[mid - 1] + arr[mid]) / two;
}

int m1 = Median(new[] { 1, 2, 3, 4, 5 });          // 3
double m2 = Median(new[] { 1.0, 2.0, 3.0, 4.0 });   // 2.5
decimal m3 = Median(new[] { 10m, 20m, 30m });        // 20
```

`T.One + T.One` gets us 2 in the generic numeric. Could also use `T.CreateChecked(2)`.

</details>

---

## Problem 7 — Predict (variance pitfall)

```csharp
public class Animal { public virtual string Name => "?"; }
public class Dog : Animal { public override string Name => "dog"; }

void Print(IEnumerable<Animal> animals) {
    foreach (var a in animals) Console.WriteLine(a.Name);
}

List<Dog> dogs = new() { new Dog(), new Dog() };
Print(dogs);
```

Does this compile? Does it work?

<details><summary>Answer</summary>

**Yes** to both.

`List<Dog>` implements `IEnumerable<Dog>`. Since `IEnumerable<out T>` is covariant, `IEnumerable<Dog>` is substitutable for `IEnumerable<Animal>`. Print accepts the latter; the call compiles and works.

If `Print` accepted `List<Animal>`, the call would FAIL — `List<T>` is invariant.

A practical takeaway: **accept the most-derived interface that suffices** in method parameters. `IEnumerable<T>` works for read-only iteration and gives callers maximum flexibility.

</details>

---

## Problem 8 — Implement Stack&lt;T&gt; from scratch

Build a `Stack<T>` with `Push`, `Pop`, `Peek`, `Count`, and amortized O(1) Push.

<details><summary>Solution</summary>

```csharp
public class Stack<T> {
    private T[] _items = new T[4];
    private int _count;

    public int Count => _count;

    public void Push(T item) {
        if (_count == _items.Length) {
            Array.Resize(ref _items, _items.Length * 2);
        }
        _items[_count++] = item;
    }

    public T Pop() {
        if (_count == 0) throw new InvalidOperationException("empty");
        var item = _items[--_count];
        _items[_count] = default!;   // optional: clear reference to allow GC
        return item;
    }

    public T Peek() {
        if (_count == 0) throw new InvalidOperationException("empty");
        return _items[_count - 1];
    }

    public IEnumerator<T> GetEnumerator() {
        for (int i = _count - 1; i >= 0; i--) yield return _items[i];
    }
}
```

The `default!` clears the array slot so the GC can reclaim the popped object — only matters when T is a reference type. For value-type T, this is a no-op.

</details>

---

## Problem 9 — Variance-aware function composition

Write a `Compose` that combines two functions, working across type hierarchies via variance.

<details><summary>Solution</summary>

```csharp
public static Func<TIn, TOut> Compose<TIn, TMid, TOut>(
    Func<TIn, TMid> first,
    Func<TMid, TOut> second) => x => second(first(x));

Func<string, int> length = s => s.Length;
Func<int, double> reciprocal = n => 1.0 / n;
Func<string, double> composed = Compose(length, reciprocal);
Console.WriteLine(composed("hello"));  // 0.2
```

`Func<in T, out TResult>` is already variant. The compiler handles the rest.

</details>

---

## Problem 10 — Static abstract — `IDefault<T>`

Define an interface with a static abstract Default property. Implement on two types. Write a generic `Build<T>` method that uses it.

<details><summary>Solution</summary>

```csharp
public interface IDefault<T> where T : IDefault<T> {
    static abstract T Default { get; }
}

public class Settings : IDefault<Settings> {
    public int Retries { get; init; }
    public TimeSpan Timeout { get; init; }
    public static Settings Default => new() { Retries = 3, Timeout = TimeSpan.FromSeconds(30) };
}

public class Config : IDefault<Config> {
    public string Host { get; init; } = "localhost";
    public int Port { get; init; }
    public static Config Default => new() { Port = 8080 };
}

public static T Build<T>() where T : IDefault<T> => T.Default;

var s = Build<Settings>();   // Settings { Retries=3, Timeout=00:00:30 }
var c = Build<Config>();      // Config { Host=localhost, Port=8080 }
```

This is more flexible than `where T : new()` because `new()` only invokes the parameterless constructor; `IDefault<T>` lets each type define how to construct its own default with whatever values it wants.

</details>

---

## Problem 11 — A discriminated union with generics

Implement a `Result<T, TError>` type — either a Success with a value or an Error with a reason. Use sealed records.

<details><summary>Solution</summary>

```csharp
public abstract record Result<T, TError>;
public sealed record Success<T, TError>(T Value) : Result<T, TError>;
public sealed record Error<T, TError>(TError Reason) : Result<T, TError>;

// Helpers
public static class Result {
    public static Result<T, TError> Ok<T, TError>(T value) =>
        new Success<T, TError>(value);
    public static Result<T, TError> Fail<T, TError>(TError reason) =>
        new Error<T, TError>(reason);
}

// Use with pattern matching
Result<int, string> Divide(int a, int b) =>
    b == 0
        ? Result.Fail<int, string>("division by zero")
        : Result.Ok<int, string>(a / b);

string Describe(Result<int, string> r) => r switch {
    Success<int, string>(var v) => $"OK: {v}",
    Error<int, string>(var why) => $"Error: {why}",
    _ => throw new()
};

Console.WriteLine(Describe(Divide(10, 2)));  // OK: 5
Console.WriteLine(Describe(Divide(10, 0)));  // Error: division by zero
```

A common alternative to exceptions for "expected failure" flows.

</details>

---

## Problem 12 — Performance benchmark — boxing vs generic constraints

Write two versions of a "count equal items" method:
1. Using `object.Equals` (boxes value types).
2. Using `IEquatable<T>` constraint (no boxing).

How much faster is the second on a `List<int>` of 1M elements?

<details><summary>Sketch</summary>

```csharp
// Boxing version
public static int CountEqualSlow(IEnumerable<object> items, object target) {
    int count = 0;
    foreach (var x in items) if (x.Equals(target)) count++;
    return count;
}

// Generic version
public static int CountEqualFast<T>(IEnumerable<T> items, T target) where T : IEquatable<T> {
    int count = 0;
    foreach (var x in items) if (x.Equals(target)) count++;
    return count;
}

var data = Enumerable.Range(0, 1_000_000).ToList();
// CountEqualSlow boxes each int → ~10M+ allocations
// CountEqualFast: no boxing — direct int.Equals(int) call
```

On a typical machine, with `BenchmarkDotNet`:
- Slow version: ~50 ms, ~50 MB allocated.
- Fast version: ~5 ms, 0 bytes allocated.

The factor varies by hardware, but **~10× speedup + zero allocation** is the typical win for proper generic constraints. Multiplied across many hot paths in a large codebase, this is huge.

</details>

---

That's Chapter 04. You should now understand:
- Generics as the default vocabulary of modern C#.
- Type parameters, constraints, inference.
- Variance (`in`, `out`, invariant) and when each applies.
- Static abstract members enabling generic math.
- The performance story: monomorphization for value types, code sharing for reference types.

→ [Chapter 05 — Delegates & Events](../05-DelegatesEvents/README.md)
