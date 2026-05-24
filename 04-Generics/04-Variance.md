# Variance: `in`, `out`, and Why `List<string>` Isn't a `List<object>`

## What it is

**Variance** describes how generic type parameters relate when types are substituted. If `Cat` derives from `Animal`, does `IEnumerable<Cat>` derive from `IEnumerable<Animal>`? It depends on how the parameter is *used* inside the generic type.

C# supports three kinds:
- **Covariance** (`out T`) — a generic type that **produces** T can substitute up the inheritance chain.
- **Contravariance** (`in T`) — a generic type that **consumes** T can substitute *down*.
- **Invariance** — no substitution allowed; `Foo<Cat>` and `Foo<Animal>` are unrelated.

```csharp
IEnumerable<string> strings = new List<string> { "a", "b" };
IEnumerable<object> objects = strings;          // covariance — legal!

IComparer<object> objComparer = ...;
IComparer<string> strComparer = objComparer;    // contravariance — legal!

List<string> strList = new();
List<object> objList = strList;                 // ❌ invariance — illegal
```

Variance is C#'s answer to the classic question: "If T is special, is `Container<T>` special?" The answer is "it depends on what the container does with T."

---

## Why it exists

Without variance, generic code is annoyingly rigid. Consider:

```csharp
void Print(IEnumerable<object> items) {
    foreach (var x in items) Console.WriteLine(x);
}

List<string> strs = new() { "a", "b" };
Print(strs);   // would be illegal without variance!
```

`IEnumerable<string>` is *clearly* substitutable for `IEnumerable<object>` — every string IS an object, and we're only reading. Variance lets the type system express that.

The same reasoning applies in reverse for consumers — a comparer of `object` is a comparer of `string` (it can compare anything, including strings).

---

## Covariance — `out T`

A type parameter marked `out` can flow **only out** — return values, properties (get), `yield return`. Such a parameter is **covariant**: `IFoo<Derived>` is a subtype of `IFoo<Base>`.

```csharp
public interface IProducer<out T> {
    T Produce();
}

public class IntProducer : IProducer<int> {
    public int Produce() => 42;
}

IProducer<int> ip = new IntProducer();
IProducer<object> op = ip;     // covariance: IProducer<int> → IProducer<object>
object x = op.Produce();        // returns 42 (boxed) — works fine
```

This works because:
- `Produce` only **outputs** T.
- An "int producer" *is* an "object producer" (the int it produces is an object).

If T appeared as a method **input** anywhere, covariance would be unsafe:

```csharp
public interface IProducer<out T> {
    T Produce();
    void Take(T t);     // ❌ compile error: 'out T' can't be used as input
}
```

The compiler enforces it.

### Built-in covariant types

```csharp
IEnumerable<out T>
IEnumerator<out T>
IGrouping<out TKey, out TElement>
IReadOnlyList<out T>
IReadOnlyCollection<out T>
IObservable<out T>
Func<..., out TResult>     // return type
delegate TResult Func<out TResult>();
```

Any read-only collection of T is naturally covariant — you can only get T's out.

`Func<out TResult>` lets a `Func<string>` flow as a `Func<object>`.

---

## Contravariance — `in T`

A type parameter marked `in` can flow **only in** — method parameters, `set` accessors. Such a parameter is **contravariant**: `IFoo<Base>` is a subtype of `IFoo<Derived>`.

```csharp
public interface IConsumer<in T> {
    void Consume(T t);
}

public class AnyObjectConsumer : IConsumer<object> {
    public void Consume(object t) => Console.WriteLine(t);
}

IConsumer<object> oc = new AnyObjectConsumer();
IConsumer<string> sc = oc;    // contravariance — legal!
sc.Consume("hello");           // calls AnyObjectConsumer.Consume("hello") — works because string IS object
```

Why this is safe:
- `Consume` only **inputs** T.
- A "consumer of any object" *can* consume a string (since string IS an object).

If T appeared as a return value, contravariance would be unsafe:

```csharp
public interface IConsumer<in T> {
    void Consume(T t);
    T Make();              // ❌ — 'in T' can't be used as output
}
```

### Built-in contravariant types

```csharp
IComparer<in T>
IEqualityComparer<in T>
Comparison<in T>          // delegate
Action<in T>              // delegate
Predicate<in T>           // delegate
Func<in T, ..., TResult>  // input parameters
```

`Action<object>` flows as `Action<string>`. A handler that accepts any object certainly handles strings.

---

## Invariance — the default

If a type parameter is **both** input and output, variance can't help — neither direction is safe. Such parameters must be **invariant**:

```csharp
public interface IList<T> {        // notice: no `in` or `out`
    T this[int i] { get; set; }    // both!
}

IList<string> sl = new List<string>();
IList<object> ol = sl;             // ❌ — would let you do ol[0] = 42; corrupting the underlying string list.
```

`List<T>`, `Dictionary<K,V>`, `IList<T>`, `IDictionary<K,V>` — all invariant. T appears as both input (Add, this[]=) and output (this[]=).

C# enforces invariance unless you opt into `in` or `out`.

### Why mutable collections must be invariant

```csharp
// If List<T> were covariant — UNSAFE:
List<string> strs = new() { "a", "b" };
List<object> objs = strs;        // hypothetical co-variance
objs.Add(42);                     // would compile — but the underlying list is List<string>!
string s = strs[2];               // would throw or return garbage
```

This is the same reason Java's array covariance was a mistake (it requires runtime checks and `ArrayStoreException`). C# arrays are also (regrettably) covariant for historical reasons — but generic collections aren't.

```csharp
string[] strs = { "a", "b" };
object[] objs = strs;     // C# allows this (array covariance — historical mistake)
objs[0] = 42;             // ⚠ throws ArrayTypeMismatchException at runtime
```

Generics fixed this by making invariance the default.

---

## A worked example: zoo handlers

```csharp
public abstract class Animal { public abstract string Name { get; } }
public class Dog : Animal { public override string Name => "dog"; }
public class Cat : Animal { public override string Name => "cat"; }

// Producer (covariant)
public interface IBreeder<out TAnimal> {
    TAnimal Produce();
}
public class DogBreeder : IBreeder<Dog> {
    public Dog Produce() => new Dog();
}

IBreeder<Dog> db = new DogBreeder();
IBreeder<Animal> ab = db;        // covariance — legal
Animal a = ab.Produce();          // returns a Dog seen as Animal

// Consumer (contravariant)
public interface IFeeder<in TAnimal> {
    void Feed(TAnimal a);
}
public class AnyAnimalFeeder : IFeeder<Animal> {
    public void Feed(Animal a) => Console.WriteLine($"feeding {a.Name}");
}

IFeeder<Animal> af = new AnyAnimalFeeder();
IFeeder<Dog> df = af;            // contravariance — legal
df.Feed(new Dog());               // works — generic feeder feeds a dog

// Invariant (both produces and consumes)
public interface IRing<T> {
    T Current { get; }
    void Set(T item);
}
// Can't make T covariant or contravariant — both directions used.
```

---

## Variance with delegates

Delegates have implicit covariance (return) and contravariance (parameters):

```csharp
delegate object DerivedReturn();
delegate string DerivedString();

DerivedString returnsString = () => "hi";
DerivedReturn returnsObject = returnsString;   // covariant: string return → object return

delegate void ActionString(string s);
delegate void ActionObject(object o);

ActionObject takesObject = o => Console.WriteLine(o);
ActionString takesString = takesObject;        // contravariant: object param → string param
takesString("hello");                          // OK — string IS object
```

For `Func<in T1, out TR>` and `Action<in T>`, the compiler's variance annotations make this work for arbitrary types.

---

## Variance with arrays (the historical mistake)

```csharp
string[] strs = { "a", "b" };
object[] objs = strs;     // legal — arrays are covariant
objs[0] = 42;              // ⚠ ArrayTypeMismatchException at runtime
```

Java did this in 1995 to make `sort(Object[])` work for any array. It came at the cost of a runtime type check on **every** assignment to an array element. C# inherited the mistake.

Modern alternatives:
- `IReadOnlyList<T>` if you only read.
- `List<T>` if you mutate (and it's invariant, so the bug class doesn't exist).
- `Span<T>` / `ReadOnlySpan<T>` for views.

If you have an `object[]` that's actually a `string[]`, the runtime type check happens on every write. Negligible for most code, surprising in benchmarks.

---

## When does variance kick in?

Variance is **only** legal on:
- **Interface** generic parameters.
- **Delegate** generic parameters.

NOT on:
- Class generic parameters.
- Struct generic parameters.
- Method generic parameters.

```csharp
public class Box<out T> { }   // ❌ classes can't have variance
public interface IBox<out T> { ... }   // ✓
public delegate T Producer<out T>();   // ✓
```

This is because variance requires the type to be "shape only" — a class might add operations that violate the constraint, so the compiler refuses.

For variance on a class-like API, define an interface alongside and have the class implement it. Callers depend on the interface.

---

## How variance interacts with constraints

You can constrain a variant parameter:

```csharp
public interface IConverter<in TIn, out TOut> where TIn : ISomething { ... }
```

Constraints are independent of variance — they restrict what types can be substituted; variance describes how the resulting types relate.

---

## Internals — how the JIT handles variance

Variance is purely a **type system** concept at compile time. At runtime, the CLR doesn't know that `IEnumerable<string>` is "covariantly compatible" with `IEnumerable<object>`. So how does the assignment work?

The CLR treats reference-type generic substitution as compatible based on the variance annotations stored in metadata:

```il
.interface public abstract auto ansi IEnumerable`1<+ T>   // + = covariant
.interface public abstract auto ansi IComparer`1<- T>     // - = contravariant
```

When you do:

```csharp
IEnumerable<object> objs = strings;   // strings is IEnumerable<string>
```

The CLR sees the assignment, looks at `IEnumerable`'s metadata, sees the `+ T` variance annotation, and accepts the assignment. The runtime types don't change — `strings` is still a `List<string>` — but the *interface view* is `IEnumerable<object>`.

When you call `objs.GetEnumerator().Current`, the runtime returns a string, which gets implicitly boxed/cast/handled as an object.

### Value type variance limitations

Variance does **not** apply to value types:

```csharp
IEnumerable<int> ints = new[] { 1, 2 };
IEnumerable<object> objs = ints;    // ⚠ this DOES compile and work, but...
foreach (var o in objs) { ... }      // each iteration boxes the int
```

Wait — the assignment compiles? Yes, with **boxing** under the hood. `IEnumerable<int>` is covariant for reference types only; for value-type-to-reference-type conversions, the runtime boxes during iteration.

In practice this works seamlessly but has a hidden allocation cost per element.

To avoid the box, keep types compatible at the value-type level (use generic constraints + `EqualityComparer<T>` etc., or operate on `Span<int>`).

---

## Common patterns

### Covariance for read-only views

```csharp
public IReadOnlyList<Animal> GetAnimals() => _dogs;   // _dogs is List<Dog>
// Legal because IReadOnlyList<out T> is covariant — Dog → Animal substitution OK
```

### Contravariance for callbacks

```csharp
public void ForEach<T>(IEnumerable<T> source, Action<T> action) { ... }

Action<Animal> printAny = a => Console.WriteLine(a.Name);
ForEach(new List<Dog> { new(), new() }, printAny);
// Action<Animal> flows as Action<Dog> — contravariance
```

### Mixed (in delegates)

```csharp
public Func<TIn, TOut> Compose<TIn, TMid, TOut>(Func<TIn, TMid> first, Func<TMid, TOut> second) =>
    x => second(first(x));
```

Type inference + variance lets composition flow naturally without casts.

---

## Common bugs

- **Forgetting that variance is interface/delegate only** — classes can't be variant.
- **Trying to make `IList<T>` covariant** — `IList<out T>` is illegal because `Add(T)` is contravariant. The runtime type system would have to enforce both directions.
- **Hidden boxing in value-to-reference variance** — `IEnumerable<int> → IEnumerable<object>` boxes per element.
- **Array covariance trap** — `string[] → object[]` assignment compiles but writes throw. Use generic collections.
- **Variance with constraints** — sometimes the constraint and variance combine in surprising ways. Test.

---

## Performance

- Variance has **zero runtime cost** for reference-type generic interfaces (it's metadata only).
- Value-type → reference-type variance boxes per element (lazy on iteration).
- Array variance forces a runtime check per write — small cost, but real.

---

## When to add variance

You're designing an interface or delegate. Ask:
- Is T **only** an output? Add `out T`.
- Is T **only** an input? Add `in T`.
- Is T both? Leave invariant.

It costs you nothing to add the right modifier, and your callers gain flexibility.

**Bad omission**: `IEnumerable<T>` was invariant in .NET 2.0. .NET 4.0 added the `out T`, suddenly thousands of patterns "just worked."

---

## Quick reference

| Modifier | Means | Examples |
|---|---|---|
| `out T` | Covariant (output-only) | `IEnumerable<T>`, `IReadOnlyList<T>`, `Func<..., out TResult>` |
| `in T` | Contravariant (input-only) | `IComparer<T>`, `Action<T>`, `IEqualityComparer<T>` |
| (none) | Invariant (both directions) | `List<T>`, `IList<T>`, `Dictionary<K,V>` |

→ Next: [05-StaticAbstractMembers.md](05-StaticAbstractMembers.md)
