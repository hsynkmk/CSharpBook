# Chapter 04 — Questions

> Drilling for everything in Chapter 04. Generics underpin almost all of modern C#.

---

## Basics

**Q1.** What's the difference between `typeof(List<>)` and `typeof(List<int>)`?
<details><summary>Answer</summary>`List<>` is the **open generic type** — type parameter not specified. `List<int>` is a **constructed (closed) generic type**. The open form is used in reflection (e.g., `MakeGenericType`); the closed form is what you actually instantiate.</details>

**Q2.** Does C# use type erasure like Java?
<details><summary>Answer</summary>No. C# generics are **reified** — type information survives at runtime. `List<int>` and `List<string>` are distinct runtime types. The JIT specializes value-type instantiations and shares code among reference-type instantiations.</details>

**Q3.** Why is `new T()` only allowed when you specify `where T : new()`?
<details><summary>Answer</summary>The compiler needs to know T has a parameterless constructor. Without the constraint, the compiler can't prove the constructor exists. The constraint promises it does, and the JIT generates the appropriate call (which became a JIT intrinsic in .NET 6+).</details>

**Q4.** What's "code sharing" in generic instantiation?
<details><summary>Answer</summary>For all reference-type Ts, the JIT generates **one** native code body for a generic method/type and dispatches via a generic context token. For each distinct value-type T, the JIT generates a separate specialized body. Reference: smaller code, slight dispatch overhead. Value: more code, but direct optimized access.</details>

---

## Methods and inference

**Q5.** Predict: does this compile?
```csharp
public T Process<T>() => default!;
var x = Process();
```
<details><summary>Answer</summary>**No.** Type inference needs T to appear in arguments. Return-only T can't be inferred. Compiler error: "The type arguments cannot be inferred from the usage."

Fix: `var x = Process<int>();`</details>

**Q6.** Can a generic method's type parameter be different from its enclosing generic class's type parameter?
<details><summary>Answer</summary>Yes — they're independent. `public class Service<T> { public TResult Process<TResult>(T input) {...} }`. T is fixed per instance; TResult per call.</details>

---

## Constraints

**Q7.** What does `where T : notnull` mean?
<details><summary>Answer</summary>T is non-nullable — either a value type (which can't be null) or a non-nullable reference type (`string`, not `string?`). Used in dictionary key types like `Dictionary<TKey, TValue> where TKey : notnull`.</details>

**Q8.** Why doesn't this compile?
```csharp
public class Box<T> where T : new(), IDisposable { ... }
```
<details><summary>Answer</summary>Constraint order: `class`/`struct` first, base classes, interfaces, then `new()` LAST. Should be `where T : IDisposable, new()`.</details>

**Q9.** What does `where T : unmanaged` enable?
<details><summary>Answer</summary>T must be a value type whose fields (recursively) are all value types — no managed references. Enables `stackalloc T[size]`, unsafe pointer operations, low-level interop. Examples: primitives, enums, structs of primitives.</details>

**Q10.** When would you use `where T : U` (one type param constrains another)?
<details><summary>Answer</summary>When one parameter must be assignable to another. Example: `class Cache<TKey, TValue> { public void Add<TActualValue>(TKey k, TActualValue v) where TActualValue : TValue { ... } }`. Lets you add a derived value with compile-time safety.</details>

---

## Variance

**Q11.** What does `out T` mean on an interface parameter?
<details><summary>Answer</summary>Covariant. T can only be **output** (return values, get-only properties). `IEnumerable<out T>` lets `IEnumerable<string>` flow as `IEnumerable<object>`.</details>

**Q12.** What does `in T` mean?
<details><summary>Answer</summary>Contravariant. T can only be **input** (method parameters, set accessors). `IComparer<in T>` lets `IComparer<object>` flow as `IComparer<string>`.</details>

**Q13.** Why isn't `List<T>` covariant or contravariant?
<details><summary>Answer</summary>It uses T as both input (`Add(T)`, `this[int] = T`) and output (`this[int]`, enumeration). Variance only works when usage is one-directional. Mutable collections are always invariant.</details>

**Q14.** Predict — does this compile?
```csharp
public class MyList<out T> { public T this[int i] { get; set; } }
```
<details><summary>Answer</summary>No. (a) Classes can't have variance — only interfaces and delegates can. (b) The indexer has a setter, making T input. Variance would be unsafe even on an interface.</details>

**Q15.** What's the difference between covariance on `IEnumerable<int>` → `IEnumerable<object>` for a value type vs reference type?
<details><summary>Answer</summary>For reference types (e.g., `IEnumerable<string>` → `IEnumerable<object>`), no conversion happens — it's pure metadata. For value-type-to-reference-type (`IEnumerable<int>` → `IEnumerable<object>`), each enumeration **boxes** the value. Same syntax, different cost.</details>

---

## Static abstract members

**Q16.** What's a static abstract member?
<details><summary>Answer</summary>Since C# 11, an interface can declare static methods, properties, operators, and conversion operators that implementers MUST provide. Enables generic algorithms that call static members through a type parameter: `T.Method()`, `T.Zero`, `T + T`.</details>

**Q17.** What's the "curiously recurring template pattern" in C# generics?
<details><summary>Answer</summary>`where T : IFoo<T>` — T constrains itself. Used so static abstract members on IFoo can return or accept T directly. Example: `where T : INumber<T>` lets you write `T.Zero` and `T + T`.</details>

**Q18.** What's `INumber<T>` and why does it exist?
<details><summary>Answer</summary>A combo interface in `System.Numerics` requiring all the arithmetic operators (`+`, `-`, `*`, `/`), comparison, identity values (`Zero`, `One`), and conversions. Lets you write generic math algorithms (`Sum<T>`, `Mean<T>`) that work for any numeric type. Built into int, double, decimal, BigInteger, etc., in .NET 7+.</details>

---

## `default`

**Q19.** What's the difference between `default(int?)` and `default(int)`?
<details><summary>Answer</summary>`default(int?)` is `null` (Nullable&lt;int&gt; with HasValue = false). `default(int)` is `0`. Easy to confuse — they're different types entirely.</details>

**Q20.** Predict:
```csharp
int? a = 0;
int? b = default;
Console.WriteLine(a == b);
```
<details><summary>Answer</summary>**False.** `a` has value 0. `b` is null. They're not equal.</details>

**Q21.** What does `T t = default;` do for unconstrained T?
<details><summary>Answer</summary>The runtime zeros the storage. For reference types T, t becomes null. For value types T, t becomes the all-zero instance. For Nullable&lt;T&gt;, t becomes null (HasValue = false). The IL is `initobj T` — one instruction.</details>

---

## Performance and internals

**Q22.** How many times does the JIT compile the body of `List<T>.Add` if your code uses `List<int>`, `List<string>`, `List<User>`, `List<Order>`?
<details><summary>Answer</summary>**Twice.** Once for `List<int>` (value-type specialization), once shared across `List<string>`, `List<User>`, `List<Order>` (reference-type shared body). The JIT distinguishes value-type instantiations (each gets its own body) from reference-type instantiations (sharing).</details>

**Q23.** Why does `EqualityComparer<T>.Default` avoid boxing for value types implementing `IEquatable<T>`?
<details><summary>Answer</summary>`EqualityComparer<T>.Default` picks `GenericEqualityComparer<T>` for types implementing `IEquatable<T>`. Its `Equals(T a, T b)` calls `a.Equals(b)` directly via the constrained-call IL prefix — no boxing. Without IEquatable, it falls back to `ObjectEqualityComparer<T>` which uses `Object.Equals(object)` and boxes.</details>

**Q24.** What does this IL prefix mean?
```il
constrained. !!T
callvirt instance bool IEquatable`1<!!T>::Equals(!0)
```
<details><summary>Answer</summary>The `constrained.` prefix tells the JIT: "the receiver is of generic type T; if T is a value type and overrides this interface method directly, call without boxing; otherwise box-then-callvirt." This is the runtime mechanism behind allocation-free generic dispatch over value types.</details>

**Q25.** Why is `where T : struct` good for performance?
<details><summary>Answer</summary>The JIT can specialize per T, eliminate boxing for interface dispatch, devirtualize calls (structs are implicitly sealed), and lay out fields inline. Reference-type Ts share code; struct Ts each get optimized code.</details>

---

## Generic math (C# 11+)

**Q26.** Write a generic `Sum` that works for any numeric type.
<details><summary>Answer</summary>
```csharp
using System.Numerics;
public static T Sum<T>(IEnumerable<T> source) where T : INumber<T> {
    T total = T.Zero;
    foreach (var x in source) total += x;
    return total;
}
```
Works for `int`, `long`, `double`, `decimal`, `BigInteger`, and any custom type implementing `INumber<T>`.</details>

**Q27.** What's the difference between `T.CreateChecked`, `T.CreateSaturating`, `T.CreateTruncating`?
<details><summary>Answer</summary>
- `CreateChecked` — throws on overflow.
- `CreateSaturating` — clamps to MinValue/MaxValue.
- `CreateTruncating` — wraps / truncates.

E.g., `byte.CreateChecked(300)` throws; `byte.CreateSaturating(300)` is 255; `byte.CreateTruncating(300)` is 44.</details>

---

## Synthesis

**Q28.** A coworker writes:
```csharp
public class Cache<T> where T : new() {
    private Dictionary<string, T> _items = new();
    public T Get(string key) {
        if (!_items.TryGetValue(key, out var v)) v = _items[key] = new T();
        return v;
    }
}
```
What's good, what's worrying?
<details><summary>Answer</summary>
**Good**: clean generic factory pattern. `new()` constraint allows lazy creation.

**Worrying**:
- Not thread-safe. Two threads might both create instances. Use `ConcurrentDictionary<string, T>` with `GetOrAdd`.
- `new T()` was reflection-based pre-.NET 6 — slow in tight loops. Modern .NET intrinsifies it but if you're targeting older runtimes, consider a factory delegate.
- No expiration / size limit. For a long-running app, this grows forever.
</details>

**Q29.** When would you make a method generic vs accepting `object`?
<details><summary>Answer</summary>
**Generic** when:
- You want type safety at the call site (no casts).
- You want to avoid boxing for value types.
- Operations on T need to be type-aware (constraints unlock methods).

**Object** when:
- You genuinely treat any input the same way (e.g., logging).
- The type isn't known until runtime (config parsing).
- The arg variety is small and discrete (a switch on `is`).

Modern style strongly favors generics — even for "any type", `T : notnull` is usually cleaner than `object`.
</details>

**Q30.** Explain why `IEnumerable<T>` was made covariant in .NET 4.0.
<details><summary>Answer</summary>Pre-.NET 4, `IEnumerable<string>` couldn't flow to `IEnumerable<object>`. That meant any LINQ chain that needed to "widen" types required explicit `.Cast<object>()`. Making T covariant (`out T`) eliminated thousands of casts and made polymorphic iteration natural — at no runtime cost.</details>

---

→ [Coding.md](Coding.md)
