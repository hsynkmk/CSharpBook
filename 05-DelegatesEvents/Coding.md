# Chapter 05 — Coding Problems

> 12 hands-on problems on delegates, lambdas, closures, events, local functions, expression trees.

---

## Problem 1 — Implement memoization

Write a `Memoize<T, R>` extension that wraps a `Func<T, R>` and caches results.

<details><summary>Solution</summary>

```csharp
public static Func<T, R> Memoize<T, R>(this Func<T, R> f) where T : notnull {
    var cache = new Dictionary<T, R>();
    return x => {
        if (!cache.TryGetValue(x, out var v)) cache[x] = v = f(x);
        return v;
    };
}

// Usage
Func<int, long> slow = n => { Thread.Sleep(100); return (long)n * n; };
var fast = slow.Memoize();
fast(5);  // takes 100 ms
fast(5);  // instant — cached
```

The closure captures `cache`. Each `Memoize` call creates a fresh cache.

Note: not thread-safe. For concurrent use, use `ConcurrentDictionary<T, R>` with `GetOrAdd`.

</details>

---

## Problem 2 — Predict the output

```csharp
var actions = new List<Action>();
for (int i = 0; i < 3; i++) {
    actions.Add(() => Console.Write(i));
}

foreach (var i in Enumerable.Range(10, 3)) {
    actions.Add(() => Console.Write(i));
}

foreach (var a in actions) a();
```

<details><summary>Answer</summary>

`33310 11 12`

- The first loop is `for (int i = 0; ...)` — all three lambdas captured the **same** `i`, which is 3 after the loop. They print `3 3 3`.
- The second loop is `foreach` — each iteration gives a fresh `i`. The lambdas print `10 11 12`.

</details>

---

## Problem 3 — Build a thread-safe event subscription that prevents leaks

Subscribers should be auto-unsubscribed via `IDisposable`.

<details><summary>Solution</summary>

```csharp
public class Publisher {
    public event EventHandler? Updated;
    public void RaiseUpdated() => Updated?.Invoke(this, EventArgs.Empty);
}

public sealed class Subscription : IDisposable {
    private readonly Publisher _pub;
    private readonly EventHandler _handler;

    public Subscription(Publisher pub, EventHandler handler) {
        _pub = pub;
        _handler = handler;
        _pub.Updated += _handler;
    }

    public void Dispose() {
        _pub.Updated -= _handler;
    }
}

// Use
var pub = new Publisher();
using (new Subscription(pub, (s, e) => Console.WriteLine("got it"))) {
    pub.RaiseUpdated();   // prints "got it"
}
pub.RaiseUpdated();        // nothing — subscription disposed
```

This pattern (sometimes called RAII for events) ensures no leaks.

</details>

---

## Problem 4 — Make a counter with `Func<int>`

Write a `MakeCounter()` that returns a `Func<int>` which produces 1, 2, 3, ... on each call.

<details><summary>Solution</summary>

```csharp
public static Func<int> MakeCounter() {
    int n = 0;
    return () => ++n;
}

var c1 = MakeCounter();
var c2 = MakeCounter();
Console.WriteLine(c1());   // 1
Console.WriteLine(c1());   // 2
Console.WriteLine(c2());   // 1 — independent counter
Console.WriteLine(c1());   // 3
```

Each call creates a separate closure with its own `n`.

</details>

---

## Problem 5 — Static lambda vs capturing lambda — benchmark

Write a method that takes a `Func<int, int>` and applies it 1M times. Pass two versions:
1. `static x => x * x` (no capture).
2. `x => x * factor` (captures local).

Measure allocations.

<details><summary>Sketch</summary>

```csharp
public static int Apply(Func<int, int> f, int seed) {
    int n = seed;
    for (int i = 0; i < 1_000_000; i++) n = f(n);
    return n;
}

// Test 1: static lambda
int r1 = Apply(static x => (x + 1) % 1000, 0);
// → no closure, one cached delegate, zero allocations in this call

// Test 2: capturing lambda
int factor = 7;
int r2 = Apply(x => (x + factor) % 1000, 0);
// → one closure + one delegate allocated per Apply call
```

The static-lambda version is the same speed as a regular method call; the capturing version is the same per-iteration cost but pays a single closure allocation up front.

For algorithm-with-policy patterns, the static form is strictly better when you can avoid captures.

</details>

---

## Problem 6 — Local function for argument validation in iterator

Write `IEnumerable<int> Range(int start, int count)` that throws `ArgumentOutOfRangeException` eagerly if `count < 0`, but lazily generates values.

<details><summary>Solution</summary>

```csharp
public static IEnumerable<int> Range(int start, int count) {
    if (count < 0) throw new ArgumentOutOfRangeException(nameof(count));

    return Impl();

    IEnumerable<int> Impl() {
        for (int i = 0; i < count; i++) yield return start + i;
    }
}

// Test
try { Range(0, -5); }
catch (ArgumentOutOfRangeException) { Console.WriteLine("caught immediately"); }

var seq = Range(0, 3);    // no throw
foreach (var x in seq) Console.WriteLine(x);    // 0 1 2
```

Without splitting, the throw would happen on first `MoveNext` — too late and confusing.

</details>

---

## Problem 7 — Compose functions

Write a generic `Compose<A, B, C>(Func<A, B>, Func<B, C>) -> Func<A, C>` and use it to build `string → int → bool`.

<details><summary>Solution</summary>

```csharp
public static Func<A, C> Compose<A, B, C>(Func<A, B> f, Func<B, C> g) =>
    x => g(f(x));

Func<string, int> length = s => s.Length;
Func<int, bool> isLong = n => n > 5;
Func<string, bool> isLongString = Compose(length, isLong);

Console.WriteLine(isLongString("hi"));      // false
Console.WriteLine(isLongString("hello!"));   // true
```

A nice pattern in functional-style C# code. Variance lets the input/output types flow nicely.

</details>

---

## Problem 8 — Pipeline with action stack

Implement a `Pipeline<T>` builder that lets you chain transformations:

```csharp
var p = new Pipeline<int>()
    .Map(x => x + 1)
    .Map(x => x * 2)
    .Filter(x => x % 3 == 0);

foreach (var x in p.Run(new[] { 1, 2, 3, 4, 5 }))
    Console.WriteLine(x);
```

<details><summary>Solution</summary>

```csharp
public class Pipeline<T> {
    private readonly List<Func<IEnumerable<T>, IEnumerable<T>>> _steps = new();

    public Pipeline<T> Map(Func<T, T> f) {
        _steps.Add(seq => seq.Select(f));
        return this;
    }

    public Pipeline<T> Filter(Func<T, bool> pred) {
        _steps.Add(seq => seq.Where(pred));
        return this;
    }

    public IEnumerable<T> Run(IEnumerable<T> input) =>
        _steps.Aggregate(input, (seq, step) => step(seq));
}
```

Each `Map`/`Filter` adds a transformation; `Run` applies them in order. Each step is a `Func<IEnumerable<T>, IEnumerable<T>>` stored in a list. Closures over the per-step lambda capture the input.

</details>

---

## Problem 9 — Build an expression tree dynamically

Build a `predicate: User => User.Age > 18 && User.IsActive` at runtime from configuration:

```csharp
var filters = new List<(string Field, string Op, object Value)> {
    ("Age", ">", 18),
    ("IsActive", "==", true)
};
```

<details><summary>Solution</summary>

```csharp
public static Expression<Func<T, bool>> BuildPredicate<T>(
    List<(string Field, string Op, object Value)> filters)
{
    var param = Expression.Parameter(typeof(T), "x");
    Expression? body = null;

    foreach (var (field, op, value) in filters) {
        var prop = Expression.Property(param, field);
        var constant = Expression.Constant(value, prop.Type);
        Expression compare = op switch {
            ">" => Expression.GreaterThan(prop, constant),
            "<" => Expression.LessThan(prop, constant),
            "==" => Expression.Equal(prop, constant),
            "!=" => Expression.NotEqual(prop, constant),
            _ => throw new NotSupportedException()
        };
        body = body is null ? compare : Expression.AndAlso(body, compare);
    }

    body ??= Expression.Constant(true);
    return Expression.Lambda<Func<T, bool>>(body, param);
}

// Use
var pred = BuildPredicate<User>(filters);
Console.WriteLine(pred);        // x => ((x.Age > 18) AndAlso (x.IsActive == True))

var compiled = pred.Compile();
compiled(new User { Age = 25, IsActive = true });   // true
```

This is the core technique behind dynamic LINQ and query builders. The tree can be passed to `IQueryable.Where(...)` and translated to SQL by EF Core.

</details>

---

## Problem 10 — Custom event with thread-safe raise

Implement an `OrderManager` with `OrderPlaced` event. Raise it in a way that safely snapshots subscribers, and demonstrate adding/removing handlers.

<details><summary>Solution</summary>

```csharp
public class OrderPlacedEventArgs : EventArgs {
    public int OrderId { get; }
    public DateTime PlacedAt { get; }
    public OrderPlacedEventArgs(int id, DateTime when) {
        OrderId = id; PlacedAt = when;
    }
}

public class OrderManager {
    public event EventHandler<OrderPlacedEventArgs>? OrderPlaced;

    public void Place(int orderId) {
        // (do business logic)
        OnOrderPlaced(new OrderPlacedEventArgs(orderId, DateTime.UtcNow));
    }

    protected virtual void OnOrderPlaced(OrderPlacedEventArgs args) {
        // Compiler snapshots OrderPlaced internally and null-checks safely
        OrderPlaced?.Invoke(this, args);
    }
}

// Use
var mgr = new OrderManager();
EventHandler<OrderPlacedEventArgs> log = (s, e) =>
    Console.WriteLine($"Order {e.OrderId} placed at {e.PlacedAt:O}");

mgr.OrderPlaced += log;
mgr.Place(42);     // prints
mgr.OrderPlaced -= log;
mgr.Place(43);     // no handler, no print
```

The compiler-generated `add`/`remove` accessors use `Interlocked.CompareExchange` — thread-safe by default for field-like events.

</details>

---

## Problem 11 — Convert closure-heavy code to static + parameter

Refactor this to remove the closure allocation:

```csharp
public IEnumerable<int> Square(IEnumerable<int> nums, int factor) {
    return nums.Select(x => x * factor);   // captures `factor`
}
```

<details><summary>Solution</summary>

Several options.

**Option 1 — local function with explicit parameter** (zero closure):
```csharp
public IEnumerable<int> Square(IEnumerable<int> nums, int factor) {
    return nums.Select(x => x * factor);
}
```

Wait, this still captures. To avoid:

```csharp
public IEnumerable<int> Square(IEnumerable<int> nums, int factor) {
    foreach (var n in nums) yield return n * factor;
}
```

A manual iterator avoids the lambda + closure entirely.

**Option 2 — tupled state via `Select(source, selector, state)`** (LINQ doesn't have this directly; you'd need a custom extension):

```csharp
public static IEnumerable<TR> Select<T, TState, TR>(
    this IEnumerable<T> source, TState state, Func<T, TState, TR> selector)
{
    foreach (var item in source) yield return selector(item, state);
}

// Use — static lambda, state explicit, zero closure:
return nums.Select(factor, static (x, f) => x * f);
```

In hot paths, the manual iterator is the most common fix. Closure allocation per call is fine for most code but matters in inner loops.

</details>

---

## Problem 12 — Implement IObservable-like pub/sub by hand

Build a `Channel<T>` (toy version) that has Subscribe and Publish. Subscribe returns an IDisposable for clean unsubscribe.

<details><summary>Solution</summary>

```csharp
public class Channel<T> {
    private readonly List<Action<T>> _subscribers = new();
    private readonly object _lock = new();

    public IDisposable Subscribe(Action<T> handler) {
        lock (_lock) _subscribers.Add(handler);
        return new Subscription(this, handler);
    }

    public void Publish(T value) {
        Action<T>[] snapshot;
        lock (_lock) snapshot = _subscribers.ToArray();
        foreach (var s in snapshot) {
            try { s(value); } catch { /* swallow per-subscriber errors */ }
        }
    }

    private sealed class Subscription : IDisposable {
        private readonly Channel<T> _channel;
        private readonly Action<T> _handler;
        public Subscription(Channel<T> ch, Action<T> h) { _channel = ch; _handler = h; }
        public void Dispose() {
            lock (_channel._lock) _channel._subscribers.Remove(_handler);
        }
    }
}

// Use
var ch = new Channel<string>();
using (ch.Subscribe(msg => Console.WriteLine($"got: {msg}"))) {
    ch.Publish("hello");
    ch.Publish("world");
}   // subscription disposed at end of using
ch.Publish("nobody listening");   // no output
```

A practical event-bus pattern. Real implementations would use weak references, more sophisticated concurrency, or `IObservable<T>` from System.Reactive.

</details>

---

That's Chapter 05. You should now understand:
- Delegates as typed method pointers, multicasting, equality.
- Lambdas, captures, closures, and the for-loop trap.
- `Func<>` / `Action<>` / `Predicate<T>` as the universal vocabulary.
- Events as access-controlled delegates with pub-sub semantics.
- Local functions, `static` local functions, and their performance benefits.
- Anonymous methods (legacy).
- Expression trees as code-as-data, enabling LINQ providers.

→ [Chapter 06 — LINQ](../06-LINQ/README.md)
