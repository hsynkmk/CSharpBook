# Generic Methods

## What it is

A **generic method** has its own type parameter list, independent of any enclosing type. You can put generic methods on:
- Non-generic classes — `Util.Last<T>(...)`.
- Generic classes — adding a method with extra type parameters.
- Static classes.
- Interfaces.

```csharp
public static class Algorithms {
    public static T Max<T>(T a, T b) where T : IComparable<T> =>
        a.CompareTo(b) >= 0 ? a : b;
}

int n = Algorithms.Max(3, 5);              // T inferred as int
string s = Algorithms.Max("a", "b");        // T inferred as string
```

The method picks up its own T per call. Different calls can use different types.

---

## Why they exist

Sometimes you don't want a whole generic type — just one operation that's polymorphic over types. Generic methods give you reusable logic without forcing a containing class to be generic.

The BCL is full of them:

```csharp
Enumerable.Where<TSource>(IEnumerable<TSource> source, Func<TSource, bool> predicate);
Enumerable.Select<TSource, TResult>(IEnumerable<TSource> source, Func<TSource, TResult> selector);
Array.Find<T>(T[] array, Predicate<T> match);
JsonSerializer.Deserialize<T>(string json);
Activator.CreateInstance<T>();
```

LINQ's whole API is generic methods.

---

## Type inference

The compiler tries to infer type arguments from method arguments:

```csharp
public T First<T>(IEnumerable<T> source) => source.First();

int n = First(new[] { 1, 2, 3 });       // T inferred as int
string s = First(new[] { "a", "b" });    // T inferred as string
```

Inference works when every type parameter appears in **at least one argument** AND the compiler can deduce it from the actual arguments.

Inference fails when:
- The type parameter doesn't appear in any argument (return-only).
- The arguments are too ambiguous.

```csharp
public T Read<T>() => default!;

T t = Read();        // ❌ — no argument; compiler can't infer
T t = Read<int>();   // ✓ — explicit
```

```csharp
public TResult Map<TInput, TResult>(TInput input, Func<TInput, TResult> mapper) =>
    mapper(input);

int len = Map("hello", s => s.Length);   // both inferred — TInput from "hello", TResult from lambda's return
```

```csharp
public T Convert<T>(object value) => (T)value;

int n = Convert(42L);   // ❌ — T doesn't appear in arguments
int n = Convert<int>(42L);   // ✓
```

When in doubt, specify explicitly. The error messages are clear when inference fails.

---

## Specifying type arguments explicitly

Most of the time you don't:
```csharp
list.Where(x => x > 0);            // TSource inferred
JsonSerializer.Deserialize<User>(json);
```

Sometimes inference doesn't disambiguate:
```csharp
list.Cast<int>();   // can't infer — explicit
list.OfType<string>();
```

The `<int>` after the method name is the **type argument list**. It looks like a generic type instantiation; it works the same.

---

## Generic methods on generic types

If you have a generic class `MyService<T>` and want a method with **another** type parameter U:

```csharp
public class MyService<T> {
    public TResult Process<TResult>(T input, Func<T, TResult> mapper) =>
        mapper(input);
}

var svc = new MyService<int>();
string s = svc.Process(42, n => n.ToString());
```

T is fixed when you instantiate the class (int here). TResult is inferred per call to `Process`.

You can have an arbitrary mix:

```csharp
public class Repo<TEntity> {
    public IList<TResult> Query<TResult>(Func<TEntity, TResult> select) { ... }
}
```

`TEntity` is per-instance; `TResult` is per-call. The compiler tracks them as separate scopes.

---

## Constraints on generic methods

Same as on generic types:

```csharp
public static T Largest<T>(IEnumerable<T> source) where T : IComparable<T> {
    using var e = source.GetEnumerator();
    if (!e.MoveNext()) throw new InvalidOperationException();
    T best = e.Current;
    while (e.MoveNext()) if (e.Current.CompareTo(best) > 0) best = e.Current;
    return best;
}
```

Constraints help inference too:

```csharp
public T CreateNew<T>() where T : new() => new T();

var p = CreateNew<Person>();   // OK if Person has parameterless ctor
```

[§03](03-Constraints.md) is the full reference on constraint forms.

---

## Method overloading with generics

You can overload — but the rules get subtle. The compiler considers method generics when picking the best match:

```csharp
public void Process(int x) => Console.WriteLine("int");
public void Process<T>(T x) => Console.WriteLine("generic");

Process(5);          // "int" — non-generic is more specific
Process("hello");    // "generic" — only option
Process<int>(5);     // "generic" — explicit type arg forces it
```

This often works well, but ambiguous-looking calls deserve explicit type arguments for clarity.

A common idiom:

```csharp
public T Get<T>(string key) where T : notnull {
    // generic version
}

// Specific shortcut for the common case
public int GetInt(string key) => Get<int>(key);
public string GetString(string key) => Get<string>(key);
```

Specific helpers are often clearer for callers than always specifying `Get<int>("x")`.

---

## Generic extension methods

Extension methods can be generic:

```csharp
public static class Extensions {
    public static T? FirstOrDefault<T>(this IEnumerable<T> source, T? defaultValue = default) {
        foreach (var x in source) return x;
        return defaultValue;
    }
}

new[] { 1, 2, 3 }.FirstOrDefault();     // 1
new int[] {}.FirstOrDefault(99);         // 99
```

This is how all of LINQ's operators are implemented. They're static methods on `Enumerable`, with `this IEnumerable<TSource>` as the first parameter and an inferred TSource.

---

## Where they shine

### Algorithms

```csharp
public static T[] Shuffle<T>(T[] arr) {
    var rng = Random.Shared;
    var copy = (T[])arr.Clone();
    for (int i = copy.Length - 1; i > 0; i--) {
        int j = rng.Next(i + 1);
        (copy[i], copy[j]) = (copy[j], copy[i]);
    }
    return copy;
}

int[] shuffled = Shuffle(new[] { 1, 2, 3, 4, 5 });
```

### Factories

```csharp
public static T Default<T>() => default!;
public static T Construct<T>() where T : new() => new();
```

### Conversion

```csharp
public static T Cast<T>(object value) => (T)value;

int n = Cast<int>(42L);   // throws InvalidCastException — different types
int n = Cast<int>(42);
```

### Generic delegation

```csharp
public static Action<T> Then<T>(this Action<T> first, Action<T> second) =>
    arg => { first(arg); second(arg); };

Action<string> log = Console.WriteLine;
Action<string> both = log.Then(s => File.AppendAllText("log.txt", s));
```

---

## Internals — how generic methods compile

A generic method like:

```csharp
public T Max<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b) >= 0 ? a : b;
```

compiles to one IL body parameterized by T:

```il
.method public hidebysig static !!T Max<class T (class [System.Runtime]System.IComparable`1<!!T>)>
    (!!T a, !!T b) cil managed
{
    ldarga.s a
    ldarg.1
    constrained. !!T
    callvirt instance int32 class [System.Runtime]System.IComparable`1<!!T>::CompareTo(!0)
    ldc.i4.0
    bge.s .yesA
    ldarg.1
    ret
    .yesA:
    ldarg.0
    ret
}
```

Key things:
- `!!T` denotes a method-level type parameter (vs `!T` for type-level).
- `constrained. !!T` is an IL prefix instruction that tells the runtime "this virtual call's receiver is of generic type T — dispatch appropriately based on T's actual type."

### The `constrained` opcode

This is the magic. When T is a value type, `constrained.` makes the JIT call the value type's method directly — no boxing. When T is a reference type, it boxes as needed and does a normal interface call.

So `Max<int>` compiles to a direct call to `int.CompareTo(int)`. `Max<string>` boxes once at the comparison... actually no, strings are reference types — direct call again. The constrained opcode just adapts the dispatch.

### Code sharing

The JIT shares code among reference-type Ts (one body for `Max<string>`, `Max<User>`, `Max<Order>`). It generates a separate body per value-type T (`Max<int>`, `Max<double>`, `Max<MyStruct>`).

The reference-type shared body uses an implicit "generic instantiation token" that lets it look up T-specific operations (like `Equals` via `EqualityComparer<T>.Default`).

### `default(T)` in generic methods

```csharp
public T GetDefault<T>() => default;
```

In IL:

```il
.method public hidebysig static !!T GetDefault<T>() cil managed
{
    .locals init (!!T V_0)
    ldloca.s V_0
    initobj !!T
    ldloc.0
    ret
}
```

`initobj` zero-fills the storage. For value types it produces all-zero fields; for reference types it produces null. Same opcode handles both — the runtime knows what T is at JIT time.

### `new T()` performance

```csharp
public T Make<T>() where T : new() => new T();
```

Pre-.NET 6: this called `Activator.CreateInstance<T>()` under the hood — slow (reflection). .NET 6 added a JIT intrinsic so it compiles to a direct constructor call in most cases.

For maximum performance in older runtimes, use a factory delegate:

```csharp
public T Make<T>(Func<T> factory) => factory();
Make(() => new Person());
```

The JIT can inline the lambda and you get a regular constructor call.

---

## Common patterns

### Composition

```csharp
public static Func<T, T> Compose<T>(Func<T, T> f, Func<T, T> g) =>
    x => g(f(x));

Func<int, int> doubled = x => x * 2;
Func<int, int> plusOne = x => x + 1;
var both = Compose(doubled, plusOne);
Console.WriteLine(both(5));   // (5*2)+1 = 11
```

### Generic try-result

```csharp
public static bool TryConvert<T>(string s, out T result) {
    result = default!;
    try {
        result = (T)Convert.ChangeType(s, typeof(T));
        return true;
    } catch { return false; }
}

TryConvert<int>("42", out var n);   // n = 42
```

### Generic dispatch via static abstract members (C# 11+)

```csharp
public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}

Sum(new[] { 1, 2, 3 });     // 6
Sum(new[] { 1.5, 2.5 });    // 4.0
```

This is the modern "generic math" approach — see [§06](06-GenericMath.md).

### Recursive generic constraints

```csharp
public class Tree<TNode> where TNode : Tree<TNode> {
    public List<TNode> Children { get; } = new();
}

public class FileNode : Tree<FileNode> {
    public string Path { get; set; } = "";
}
```

"Curiously recurring template pattern" — T constrains itself to its own descendant. Used in chain APIs and self-referential structures.

---

## When inference picks the wrong type

```csharp
public T Pick<T>(T a, T b) => /* ... */;

object o1 = new int[0];
object o2 = "hi";
Pick(o1, o2);   // T inferred as object — likely not what you wanted
```

To bias inference toward a specific type, you sometimes have to specify or cast:

```csharp
Pick<string>(o1, o2);   // throws — o1 isn't a string
```

Or design the API to avoid ambiguity (use different signatures or specific overloads).

---

## Common bugs

- **Inference fails for return-only T** — must specify explicitly.
- **Calling `.Equals(other)` on a generic T** — calls `object.Equals` (boxing for value types). Use `EqualityComparer<T>.Default.Equals(a, b)` or constrain `T : IEquatable<T>`.
- **`null` checks on unconstrained T** — `if (t == null)` doesn't work without `where T : class` or `where T : notnull`. Use `EqualityComparer<T>.Default.Equals(t, default)` for "is default" semantics, or constrain.
- **Default(T) for reference types is null** — guard accordingly.
- **Generic interfaces vs base classes ambiguity** — when overloading. Explicit type args resolve.

---

## Performance

- Generic methods over value types are JIT-specialized — usually inlined.
- Generic methods over reference types share code — minor extra dispatch cost (negligible).
- `EqualityComparer<T>.Default` and `Comparer<T>.Default` cache the right comparer (no boxing for IEquatable/IComparable Ts).
- Lambda captures inside generic methods sometimes hoist to closure classes; impact varies.

---

## When to use a generic method vs a generic type

| Need | Use |
|---|---|
| A reusable operation across types | Generic method |
| A container of T | Generic type |
| Multiple methods sharing the same T | Generic type |
| Each method picks its own T | Generic method |
| Want type inference at call site | Generic method (inference works on methods) |

→ Next: [03-Constraints.md](03-Constraints.md)
