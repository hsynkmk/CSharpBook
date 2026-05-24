# `Func<>`, `Action<>`, and `Predicate<T>`

## What it is

Three generic delegate types in the BCL that cover almost every "method-as-data" need:

- **`Func<...>`** — returns a value. Up to 16 input parameters + 1 return.
- **`Action<...>`** — returns `void`. Up to 16 input parameters.
- **`Predicate<T>`** — returns `bool`, takes one `T`. Single-purpose.

```csharp
Func<int, int> square = x => x * x;
Func<int, int, int> add = (a, b) => a + b;
Func<string> getName = () => "Alice";

Action<string> log = msg => Console.WriteLine(msg);
Action<int, int> print = (a, b) => Console.WriteLine($"{a},{b}");
Action greet = () => Console.WriteLine("hi");

Predicate<int> isPositive = n => n > 0;
```

99% of the time you don't need a custom delegate type — these cover it.

---

## Why they exist

Pre-generics C# required custom delegate types per signature:

```csharp
public delegate int IntFunc1(int a);
public delegate int IntFunc2(int a, int b);
public delegate void StringHandler(string s);
// ... and so on for every shape
```

The BCL ended up with hundreds of these. With generics (C# 2), `Func<>` and `Action<>` could express any signature:

```csharp
Func<int, int> intFunc1;
Func<int, int, int> intFunc2;
Action<string> stringHandler;
```

One type per "arity" (number of parameters), used for everything. Less ceremony, more reuse.

---

## `Func<...>`

Returns a value. **The last type parameter is the return type**; earlier ones are inputs.

```csharp
Func<int> getRandom = () => Random.Shared.Next();      // () → int
Func<string, int> length = s => s.Length;              // (string) → int
Func<int, int, int> add = (a, b) => a + b;             // (int, int) → int
Func<int, int, int, int> triple = (a, b, c) => a+b+c;  // (int, int, int) → int
// ...up to Func<T1, ..., T16, TResult> (16 inputs)
```

Mnemonic: "everything before the last `>` is input; the last is output."

```csharp
Func<string, int> f;     // takes string, returns int
Func<int, string> g;     // takes int, returns string
Func<int, int> h;        // takes int, returns int — last parameter IS return
```

### Variance

`Func<in T1, ..., out TResult>` — the input parameters are contravariant, the result is covariant:

```csharp
Func<object, string> oToS = o => o.ToString();
Func<string, object> sToO = oToS;   // string is more specific input, object is broader output
```

A function that handles any object and returns a string can be used where you expected "handles strings and returns object." Variance lets this flow.

---

## `Action<...>`

Returns `void`. All type parameters are inputs.

```csharp
Action a = () => Console.WriteLine("nothing");
Action<int> b = x => Console.WriteLine(x);
Action<string, int> c = (s, n) => Console.WriteLine($"{s}: {n}");
// ...up to Action<T1, ..., T16>
```

### Variance

`Action<in T1, ...>` — all parameters contravariant.

```csharp
Action<object> printAny = o => Console.WriteLine(o);
Action<string> printString = printAny;   // works — string IS object
printString("hello");
```

---

## `Predicate<T>`

Returns `bool`, takes one `T`. A specialization of `Func<T, bool>`:

```csharp
Predicate<int> isPositive = n => n > 0;
Predicate<string> isEmpty = string.IsNullOrEmpty;
```

Used by `Array.Find`, `List<T>.RemoveAll`, etc.

Modern code often just uses `Func<T, bool>` — interchangeable. `Predicate<T>` is mostly there for backward compat with .NET 2.0-era APIs.

```csharp
list.RemoveAll(x => x.IsDone);       // Predicate<T>
list.Where(x => x.IsDone);            // Func<T, bool>
```

Both work the same way; the API just declares the parameter as a Predicate or a Func.

---

## Comparators and converters

A few other useful generic delegates:

```csharp
Comparison<T>            // (T, T) → int; used by Array.Sort, List<T>.Sort
Converter<TIn, TOut>     // (TIn) → TOut; used by Array.ConvertAll
EventHandler             // (object, EventArgs) → void; events
EventHandler<TEventArgs> // (object, TEventArgs) → void; events
```

Most could be `Func<>` / `Action<>` but they have specific roles. Use them when API expects them.

---

## A note on parameter ordering

`Func<T1, T2, ..., TResult>` puts the result last. This sometimes confuses people coming from F#/Haskell where types are written left-to-right with arrows: `T1 -> T2 -> TResult`.

```csharp
Func<int, string, double> f;     // int, string in, double out
```

To "transform an int into a string": `Func<int, string>`. Input on the left, output on the right.

---

## Pseudo-typing tips

When you read code like:

```csharp
list.Select(x => x.Name);
```

LINQ's `Select` signature is `IEnumerable<TSource> -> Func<TSource, TResult> -> IEnumerable<TResult>`. Knowing the shape of `Func<,>` (input → output) makes LINQ much more readable.

```csharp
list.Where(x => x.Active);             // Func<X, bool>  — filter
list.Select(x => x.Id);                 // Func<X, int>  — project
list.GroupBy(x => x.Category);          // Func<X, K>    — group by key
list.OrderBy(x => x.CreatedAt);          // Func<X, T>    — sort by key
list.Aggregate((acc, x) => acc + x.Total); // Func<int, X, int> — reduce
```

---

## Common patterns

### Method-group conversion

```csharp
Func<int, bool> isPositive = n => n > 0;
// Or
Func<int, bool> isPositive = IsPositiveImpl;   // existing method

bool IsPositiveImpl(int n) => n > 0;
```

### Generic helper

```csharp
public static T Try<T>(Func<T> action, T fallback) {
    try { return action(); }
    catch { return fallback; }
}

int n = Try(() => int.Parse(input), -1);
```

### Wrapped methods

```csharp
public Func<TInput, Task<TResult>> WithRetry<TInput, TResult>(
    Func<TInput, Task<TResult>> inner,
    int maxAttempts) =>
    async input => {
        Exception? last = null;
        for (int i = 0; i < maxAttempts; i++) {
            try { return await inner(input); }
            catch (Exception ex) { last = ex; await Task.Delay(100); }
        }
        throw last!;
    };
```

Wraps an async function with retry logic. The result is itself a `Func<...>`.

### Action stacks

```csharp
public class Builder {
    private readonly List<Action> _ops = new();
    public Builder Then(Action a) { _ops.Add(a); return this; }
    public void Run() { foreach (var a in _ops) a(); }
}

new Builder()
    .Then(() => Console.WriteLine("step 1"))
    .Then(() => Console.WriteLine("step 2"))
    .Then(() => Console.WriteLine("step 3"))
    .Run();
```

### Strategy injection

```csharp
public class Calculator(Func<decimal, decimal> applyDiscount) {
    public decimal Total(decimal subtotal) => subtotal - applyDiscount(subtotal);
}

var noDiscount = new Calculator(_ => 0m);
var tenPct = new Calculator(s => s * 0.10m);
```

No interface, no class hierarchy. Just inject the function.

---

## When custom delegate types win

Most of the time, `Func<>` / `Action<>` are clearer. Custom delegate types help when:

### The signature has meaning

```csharp
public delegate decimal PriceCalculator(Order order, Customer customer);
```

Reading `PriceCalculator pc` is clearer than `Func<Order, Customer, decimal> pc`.

### You want to attach attributes / events

```csharp
public delegate void TradeExecutedHandler(object sender, TradeEventArgs args);
```

Specialized to event use.

### Parameter names matter

When IntelliSense shows `Func<Order, Customer, decimal>`, you don't know which arg is which. A named delegate type can have parameter names that show up in tooling.

### Variance customization

You can declare your own delegate with specific `in` / `out` annotations if `Func`/`Action` don't fit.

---

## Internals — Func vs custom delegate

A `Func<int, int>` is just a class deriving from `MulticastDelegate`:

```il
.class public sealed Func`2<in T, out TResult> extends [System.Runtime]System.MulticastDelegate
{
    .method public hidebysig specialname rtspecialname instance void .ctor(object target, native int method) runtime managed { }
    .method public hidebysig newslot virtual instance !TResult Invoke(!T arg) runtime managed { }
    .method public hidebysig newslot virtual instance class [System.Runtime]System.IAsyncResult BeginInvoke(...) runtime managed { }
    .method public hidebysig newslot virtual instance !TResult EndInvoke(...) runtime managed { }
}
```

It's identical to a custom delegate. No magic. The BCL just ships a generic version pre-defined up to 16 parameters.

The 16-parameter limit is arbitrary — long ago, the BCL team decided 16 was enough. Going beyond requires a custom delegate.

---

## Performance

- Allocation cost: identical to custom delegates (delegate is a heap object).
- Invocation cost: identical (one indirect call).
- `Func<>` and `Action<>` are typically cached for static lambdas — same as custom delegates would be.

In other words: choosing Func over a custom delegate has **zero** runtime cost. The choice is purely about API ergonomics.

---

## Common bugs

- **Forgetting `Func`'s last parameter is the return type** — `Func<int, string>` returns string, takes int. Confusing if you skim too fast.
- **Using `Predicate<T>` when `Func<T, bool>` was expected** — they're not directly compatible without conversion in some APIs.
- **Async lambdas typed as `Action`** — `Action a = async () => await DoAsync();` is `async void` — exceptions get lost. Use `Func<Task>`.
- **Default delegate values** — `Func<int, int>` is a reference type, default is null. Invoking null throws.

---

## Cheat sheet

```csharp
// Returns nothing
Action                          // ()
Action<T>                       // (T)
Action<T1, T2>                  // (T1, T2)

// Returns a value (last type = return)
Func<TR>                        // () -> TR
Func<T1, TR>                    // (T1) -> TR
Func<T1, T2, TR>                // (T1, T2) -> TR

// Returns bool
Predicate<T>                    // (T) -> bool (legacy; use Func<T, bool> in modern code)

// Returns int (for sorting)
Comparison<T>                   // (T, T) -> int

// Conversion
Converter<TIn, TOut>            // (TIn) -> TOut (legacy; use Func)

// Events
EventHandler                    // (object, EventArgs) -> void
EventHandler<TEventArgs>        // (object, TEventArgs) -> void
```

### When to use what

| Need | Use |
|---|---|
| One-off LINQ predicate | `x => ...` (compiler infers `Func<T, bool>`) |
| Stored callback returning a value | `Func<...>` |
| Stored callback returning void | `Action<...>` |
| Event handler | `EventHandler<TEventArgs>` |
| Sort comparator | `Comparison<T>` or `IComparer<T>` |
| Highly specific named delegate | Custom delegate type |

→ Next: [04-Closures.md](04-Closures.md)
