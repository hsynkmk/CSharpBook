# Lambdas

## What it is

A **lambda expression** is an anonymous function written inline. The compiler turns it into a method (or delegate) you can store, pass, and invoke.

```csharp
Func<int, int> square = x => x * x;
Action<string> log = msg => Console.WriteLine(msg);
Func<int, int, int> add = (a, b) => a + b;

Console.WriteLine(square(5));   // 25
log("hello");
Console.WriteLine(add(3, 4));   // 7
```

Lambdas arrived in C# 3 (2007) alongside LINQ — they're the syntax that made LINQ ergonomic.

---

## Why they exist

You **could** declare a method, then assign it to a delegate:

```csharp
int Square(int x) => x * x;
Func<int, int> sq = Square;
nums.Select(sq);
```

But that's ceremony when the function is one line and used in one place. Lambdas let you write:

```csharp
nums.Select(x => x * x);
```

Inline. Where it's used.

The motivation is the same as any anonymous function feature: turn behavior into a value, with no naming friction.

---

## Syntax

### Expression-bodied (single expression)

```csharp
x => x * x                      // single parameter
(x, y) => x + y                 // multiple parameters
() => DateTime.Now              // no parameters
(int x) => x.ToString()         // explicit type
```

The parentheses are optional when there's exactly one parameter without a type. Otherwise they're required.

### Statement-bodied (block)

```csharp
x => {
    int result = x * x;
    return result;
}

(a, b) => {
    if (a < 0) throw new ArgumentException();
    return a + b;
}
```

Multi-statement bodies use `{ ... }`. Returns must be explicit.

### Returning nothing

```csharp
Action a = () => Console.WriteLine("hi");
Action<int> b = x => Console.WriteLine(x);
```

Lambdas matching `Action` (or other void-returning delegates) just execute statements.

### Async lambdas

```csharp
Func<int, Task<int>> doubleAsync = async x => {
    await Task.Delay(100);
    return x * 2;
};

await doubleAsync(5);   // 10
```

`async` on a lambda makes it return a `Task` / `Task<T>` / `ValueTask<T>`. Required by signature.

---

## Type inference

The compiler figures out the parameter and return types from context:

```csharp
Func<int, int> sq = x => x * x;       // x inferred as int (from Func<int, int>)
List<int> nums = new();
nums.Select(x => x * 2);                // x is int (from List<int>.Select)
```

When the context doesn't pin it down, you must specify:

```csharp
var f = x => x * x;                    // ❌ — no context to infer T from
Func<int, int> f = x => x * x;          // ✓
var f = (int x) => x * x;               // ✓ — C# 10+ explicit types in lambdas
```

C# 10 added the ability to put **types on lambda parameters** so you can use `var` for the lambda itself:

```csharp
var sq = (int x) => x * x;   // var inferred as Func<int, int>
var act = (string s) => Console.WriteLine(s);  // var → Action<string>
```

---

## Captures (closures)

A lambda can reference variables from its enclosing scope:

```csharp
int factor = 10;
Func<int, int> scale = x => x * factor;
Console.WriteLine(scale(5));   // 50

factor = 20;
Console.WriteLine(scale(5));   // 100 — sees the updated factor
```

The lambda captures `factor` by **reference to its variable** (not its current value). Changing `factor` later changes what the lambda sees.

[§04 Closures](04-Closures.md) is the full deep dive — including the famous "lambdas in a for loop" trap.

---

## `static` lambdas

To **forbid** captures (and avoid the allocation cost they imply), mark a lambda `static`:

```csharp
Func<int, int> pure = static x => x * x;        // OK — no captures
int n = 5;
Func<int, int> bad = static x => x + n;          // ❌ — can't capture
```

A static lambda:
- Cannot capture local variables.
- Can capture **other static** fields/methods.
- Can be cached by the compiler — a static lambda often allocates **once** for the whole program, not per call.

For hot-path lambdas with no captures, prefer `static` — small allocation win.

---

## Attributes on lambdas (C# 10+)

You can attach attributes to lambdas and their parameters:

```csharp
Func<int, int> validated = [Pure] (int x) => x * x;
Action<int> doNothing = [Conditional("DEBUG")] (int x) => { /* ... */ };
```

Useful when the consumer of the lambda inspects attributes.

---

## When lambdas allocate

Different cases have different costs:

### No captures + static
```csharp
Func<int, int> sq = static x => x * x;
```

The compiler generates a static method and a cached static delegate. **One** allocation per assembly, regardless of how many times this line runs.

### No captures, non-static
```csharp
Func<int, int> sq = x => x * x;
```

The compiler still generates a static method but **without** the `static` keyword on the lambda. It may or may not cache — Roslyn used to allocate per call, modern versions often cache.

### Captures from `this`
```csharp
public class C {
    public int Mult { get; init; }
    public Func<int, int> Make() => x => x * Mult;   // captures `this`
}
```

The lambda becomes an instance method on `C`. One delegate allocation per call to `Make`.

### Captures locals
```csharp
public Func<int, int> Make(int mult) => x => x * mult;
```

The compiler generates a **closure class** holding `mult`, plus a method on it. **Two** allocations per call: the closure class instance + the delegate.

The closure-class allocation is the heaviest case. For hot paths, avoid capturing locals where you can.

---

## Lambdas as expression trees

A lambda assigned to `Expression<TDelegate>` is compiled to an **expression tree**, not a delegate:

```csharp
Expression<Func<int, int>> tree = x => x * x;
// tree is a data structure representing the lambda

Func<int, int> compiled = tree.Compile();   // generate a delegate at runtime
int n = compiled(5);   // 25
```

This is the trick LINQ providers (EF Core) use — they receive expression trees, walk them, and produce SQL. [§08 Expression Trees](08-ExpressionTrees.md) has the deep version.

---

## Lambda overload resolution

When calling a method with multiple delegate parameters, the compiler picks based on the lambda's parameter and body:

```csharp
public void Process(Action<int> a) { ... }
public void Process(Func<int, bool> f) { ... }

Process(x => x > 0);   // matches Func<int, bool> — body is `x > 0` (bool)
Process(x => Console.WriteLine(x));   // matches Action<int> — body is void
```

When ambiguous, you may have to cast:

```csharp
Process((Func<int, bool>)(x => x > 0));
```

---

## Internals — how lambdas compile

The compiler chooses one of three strategies:

### Static / cacheable lambda

```csharp
Func<int, int> sq = x => x * x;
```

Compiles to a static method + cached static field:

```csharp
private static Func<int, int>? <>9__0_0;
private static int <Method>b__0_0(int x) => x * x;

// at use site:
sq = <>9__0_0 ??= new Func<int, int>(<Method>b__0_0);
```

`??=` ensures the delegate is allocated once, then reused.

### Method on `this` (captures `this`)

```csharp
public class C {
    public int Mult;
    public Func<int, int> Make() => x => x * Mult;
}
```

Compiles to a private instance method:

```csharp
public class C {
    public int Mult;
    public Func<int, int> Make() => new Func<int, int>(<Make>b__0);
    private int <Make>b__0(int x) => x * Mult;
}
```

One delegate allocation per `Make()` call; no closure class needed.

### Closure class (captures locals)

```csharp
public Func<int, int> Multiplier(int factor) =>
    x => x * factor;
```

Compiles to a generated class:

```csharp
private sealed class <>c__DisplayClass0_0 {
    public int factor;
    internal int <Multiplier>b__0(int x) => x * factor;
}

public Func<int, int> Multiplier(int factor) {
    var closure = new <>c__DisplayClass0_0 { factor = factor };
    return new Func<int, int>(closure.<Multiplier>b__0);
}
```

Two allocations: the `<>c__DisplayClass` instance + the delegate.

### Lifetimes

The closure class is GC-managed. As long as the delegate is reachable, the closure (and everything it holds) is reachable. This is why captured locals can cause memory leaks if the delegate outlives expectations — see [§04](04-Closures.md).

---

## Common patterns

### LINQ projections

```csharp
var names = users.Where(u => u.IsActive).Select(u => u.Name);
```

Two lambdas, two delegate allocations (if both capture nothing).

### Async event handlers

```csharp
button.Click += async (s, e) => {
    button.IsEnabled = false;
    await DoWorkAsync();
    button.IsEnabled = true;
};
```

The async lambda runs through the state machine machinery — multiple allocations under the hood.

### Pipeline composition

```csharp
Func<int, int> pipeline = x => x + 1;
pipeline = pipeline.AndThen(x => x * 2);
pipeline = pipeline.AndThen(x => x - 3);
Console.WriteLine(pipeline(5));   // ((5+1)*2)-3 = 9
```

Where `AndThen` is:
```csharp
public static Func<T, R> AndThen<T, M, R>(this Func<T, M> first, Func<M, R> second) =>
    x => second(first(x));
```

### `using` with cleanup actions

```csharp
public class Defer : IDisposable {
    private readonly Action _action;
    public Defer(Action a) { _action = a; }
    public void Dispose() => _action();
}

using (var _ = new Defer(() => Console.WriteLine("cleanup"))) {
    Console.WriteLine("work");
}
// work
// cleanup
```

A Go-style `defer` built with lambdas.

---

## Common bugs

- **Closure capture in `for` loop** — see [§04](04-Closures.md). All lambdas see the same loop variable.
- **Capturing `this` accidentally** in long-lived delegates → memory leak.
- **`static` lambda trying to capture** — compile error.
- **Async lambda assigned to `Action`** — `async void` semantics, exceptions vanish.
- **Method-group vs lambda performance** — method group can sometimes be cached; lambda allocates per use unless static and capture-free.

---

## Performance

| Form | Allocation per call |
|---|---|
| `static x => ...` (no capture) | One allocation total (cached) |
| `x => ...` (no capture) | One allocation total (modern compiler) OR per call (older) |
| `x => this.M(x)` (captures `this`) | One delegate per call |
| `x => x + local` (captures local) | One closure object + one delegate per call |

The captures-locals case is the heaviest. For hot loops, prefer static or capture-less forms.

The lambda's body is JIT-compiled like any method — runtime cost is the same as a regular function call.

---

## When to use lambdas

✓ One-liner predicates / projections for LINQ.
✓ Callbacks that won't outlive the method.
✓ Strategy / policy injection.
✓ Event handlers (with care about lifetime).

✗ Long, complex logic — refactor to a method.
✗ Performance-critical hot loops with allocation pressure — prefer methods or static lambdas.
✗ When you need a name for clarity or debugging — `var double = (int x) => x * 2;` is OK, but `int Double(int x) => x * 2;` is clearer.

→ Next: [03-FuncActionPredicate.md](03-FuncActionPredicate.md)
