# Delegates

## What it is

A **delegate** is a type that references one or more methods with a specific signature. Once you have a delegate value, you can invoke it like a method — but it's really just a typed pointer to one (or several) targets.

```csharp
// Declare a delegate type
public delegate int Op(int a, int b);

// Bind to a method
int Add(int a, int b) => a + b;
int Mul(int a, int b) => a * b;

Op op = Add;
Console.WriteLine(op(3, 4));   // 7

op = Mul;
Console.WriteLine(op(3, 4));   // 12
```

Delegates are the foundation of:
- **Events** — multicast delegates with restricted access.
- **Lambdas and `Func<>`/`Action<>`** — generic delegate types.
- **LINQ** — operators take delegates: `Where(predicate)`, `Select(selector)`.
- **Callbacks** — async APIs that signal completion.
- **Strategy pattern** — replace virtual dispatch with delegate dispatch when convenient.

C# delegates have been around since C# 1 (2002), but they got dramatically more useful with C# 2's generics, C# 3's lambdas, and C# 5's async.

---

## Why they exist

Methods can't be passed around as values in C# without some wrapping. Delegates are that wrapping — they let you treat behavior as data:

```csharp
// Bad: tightly coupled
public void Process(int[] arr) {
    for (int i = 0; i < arr.Length; i++) {
        Console.WriteLine(arr[i] * arr[i]);   // hard-coded behavior
    }
}

// Good: behavior injected via delegate
public void Process(int[] arr, Action<int> action) {
    for (int i = 0; i < arr.Length; i++) action(arr[i]);
}

Process(new[] { 1, 2, 3 }, x => Console.WriteLine(x * x));
```

That's the whole story. Inject behavior; don't hard-code.

---

## Declaring delegate types

The pre-generics way (still in old code):

```csharp
public delegate int IntOp(int a, int b);
public delegate void StringHandler(string s);
public delegate bool Predicate<T>(T item);
```

A delegate **type** name + signature defines the contract. Methods bound to it must match the signature (allowing variance).

The modern way: use the generic delegates from the BCL:

```csharp
Func<int, int, int> intOp = (a, b) => a + b;
Action<string> stringHandler = s => Console.WriteLine(s);
Predicate<int> isPositive = n => n > 0;
```

Custom delegate types are useful when:
- The signature is reused across many places (you want a meaningful name).
- You want to attach attributes or XML docs to the type.
- You're working with legacy APIs that defined their own.

Otherwise, `Func<>` and `Action<>` are simpler.

---

## Binding methods

A delegate can wrap:

### A static method
```csharp
public static int Square(int n) => n * n;

Func<int, int> sq = Square;        // method-group conversion
int n = sq(5);                       // 25
```

### An instance method (with implicit `this`)
```csharp
public class Counter {
    private int _n;
    public int Increment() => ++_n;
}

var c = new Counter();
Func<int> increment = c.Increment;   // captures c as the receiver
Console.WriteLine(increment());      // 1
Console.WriteLine(increment());      // 2
```

The delegate holds **two** things: a reference to the receiver (`c`) and a pointer to the method. Calling the delegate calls the method on that receiver.

### A lambda
```csharp
Func<int, int> sq = x => x * x;
```

The lambda is desugared to a method (sometimes static, sometimes on a closure class — see [§04](04-Closures.md)).

### A local function
```csharp
int Square(int n) => n * n;
Func<int, int> sq = Square;
```

---

## Multicast — chaining handlers

A delegate can hold a **list** of methods. Invoking calls them all in order.

```csharp
Action a = () => Console.WriteLine("first");
a += () => Console.WriteLine("second");
a += () => Console.WriteLine("third");

a();
// first
// second
// third
```

`+=` adds; `-=` removes:

```csharp
Action one = () => Console.WriteLine("one");
Action both = one;
both += () => Console.WriteLine("two");
both();   // one, two
both -= one;
both();   // two
```

For non-void returns, the **last** method's return value wins (the others' returns are discarded). This is a frequent source of confusion — multicast delegates are really only useful for `void`-returning signatures, i.e., events and callbacks.

```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;
Console.WriteLine(f());   // 3 — only the last return is visible
```

To capture all returns, walk the invocation list manually:

```csharp
foreach (Func<int> g in f.GetInvocationList()) {
    Console.WriteLine(g());   // 1, 2, 3
}
```

---

## Invocation

Two ways to call:

```csharp
Func<int, int> sq = x => x * x;

int a = sq(5);             // call-syntax — typical
int b = sq.Invoke(5);      // explicit Invoke
```

These are identical. Use the call syntax in normal code; `Invoke()` is only useful when you want to be explicit (or for null-conditional invocation):

```csharp
Action? maybe = ...;
maybe?.Invoke();    // safe — no-op if null
// maybe?();        // doesn't work — null-conditional needs Invoke
```

The `maybe?.Invoke()` pattern is the standard way to raise events safely.

---

## Method-group conversion

Assigning a method (without parentheses) to a delegate type creates a delegate:

```csharp
Func<int, int> sq = Square;       // method group
Func<int, int> sq2 = (Square);    // also OK
Func<int, int> sq3 = x => Square(x);  // lambda wrapping (same thing, more verbose)
```

Method-group conversion is preferred where it works — cleaner and the compiler can optimize (caches the delegate when possible).

C# 11 added **inferred method-group conversion** in more contexts, eliminating some explicit casts.

---

## Equality

Two delegates are equal if they reference the same method on the same receiver:

```csharp
Func<int, int> a = Square;
Func<int, int> b = Square;
Console.WriteLine(a == b);   // true — same static method

class C { public int N(int x) => x; }
var c = new C();
Func<int, int> p = c.N;
Func<int, int> q = c.N;
Console.WriteLine(p == q);   // true — same instance, same method
```

For lambdas, two equal-looking lambdas usually aren't equal (each instance is its own delegate). The compiler caches some lambdas, particularly static ones (no captures), but don't rely on lambda identity for equality.

`==` on delegates calls `Delegate.Equals`, which compares the invocation lists.

---

## Internals — what a delegate actually is

A delegate is a class that derives from `System.MulticastDelegate` (which derives from `System.Delegate`):

```
System.Object
└── System.Delegate
    └── System.MulticastDelegate
        └── YourDelegateType
```

When you declare `public delegate int Op(int a, int b);`, the compiler generates a sealed class:

```csharp
public sealed class Op : MulticastDelegate {
    public Op(object target, IntPtr method) { ... }
    public int Invoke(int a, int b) { ... }
    public IAsyncResult BeginInvoke(int a, int b, AsyncCallback? cb, object? state) { ... }
    public int EndInvoke(IAsyncResult result) { ... }
}
```

Each delegate **instance** holds:
- **Target** — the receiver for instance methods, or null for static.
- **Method** — a `MethodInfo` (or `IntPtr` to native code).
- **Previous** — pointer to the previous invocation (for multicast — linked list).

Invoking the delegate calls a JIT-generated stub that:
1. Pushes the target as `this` (if any).
2. Calls the method.
3. (For multicast) walks the linked list and repeats.

```csharp
Func<int, int> sq = Square;
int n = sq(5);
```

In IL:
```il
ldnull
ldftn int32 Program::Square(int32)
newobj instance void [System.Runtime]System.Func`2<int32, int32>::.ctor(object, native int)
stloc.0

ldloc.0
ldc.i4.5
callvirt instance !1 [System.Runtime]System.Func`2<int32, int32>::Invoke(!0)
```

`ldftn` (load function token) gets the method's native pointer; `newobj` wraps it in the delegate; `callvirt` invokes through the synthesized Invoke method.

For instance methods, `ldnull` is replaced with the actual receiver:

```il
ldarg.0           // 'this'
ldftn instance int32 Counter::Increment()
newobj ...
```

### Multicast as a linked list

`MulticastDelegate` internally holds a chain via `_invocationList`. When you do `a += b`, the runtime creates a new delegate whose chain is `a`'s chain followed by `b`'s. The original `a` is unchanged (delegates are immutable).

This is why `a += b` only works if you re-assign the result. It returns a new delegate; it doesn't mutate `a`.

### Allocation cost

Every method-to-delegate conversion **allocates** a new delegate instance — small but real:
- ~24-32 bytes on 64-bit (header + Target + Method + previous link).

Static methods often get cached delegates (the compiler emits a static field to hold the singleton), avoiding repeat allocations. For instance methods, each conversion typically allocates.

---

## Common patterns

### Strategy pattern

```csharp
public class OrderProcessor {
    private readonly Func<Order, decimal> _calculateTax;

    public OrderProcessor(Func<Order, decimal> calculateTax) {
        _calculateTax = calculateTax;
    }

    public decimal Total(Order o) => o.Subtotal + _calculateTax(o);
}

var simple = new OrderProcessor(o => o.Subtotal * 0.08m);
var ny = new OrderProcessor(o => CalculateNYTax(o));
```

The tax strategy is injected as a delegate. No need for an interface and a class hierarchy.

### Event-like callbacks

```csharp
public class Worker {
    public Action<string>? OnProgress;
    public Action<Exception>? OnError;

    public void Run() {
        OnProgress?.Invoke("started");
        try {
            // ...
        } catch (Exception ex) {
            OnError?.Invoke(ex);
        }
    }
}

var w = new Worker();
w.OnProgress = msg => Console.WriteLine($"[progress] {msg}");
w.OnError = ex => Console.WriteLine($"[error] {ex.Message}");
w.Run();
```

Simple than full events but the same idea.

### Function pipelines

```csharp
public static Func<T, T> Pipe<T>(params Func<T, T>[] steps) =>
    x => steps.Aggregate(x, (acc, step) => step(acc));

var pipeline = Pipe<int>(
    x => x + 1,
    x => x * 2,
    x => x - 3
);
Console.WriteLine(pipeline(5));   // ((5+1)*2)-3 = 9
```

### Memoization

```csharp
public static Func<T, R> Memoize<T, R>(Func<T, R> f) where T : notnull {
    var cache = new Dictionary<T, R>();
    return x => {
        if (!cache.TryGetValue(x, out var v)) cache[x] = v = f(x);
        return v;
    };
}

Func<int, long> slow = n => { Thread.Sleep(100); return n * n; };
var fast = Memoize(slow);
fast(5);  // slow first time
fast(5);  // instant second time
```

The cache is captured in the lambda's closure.

---

## Common bugs

- **Reassigning vs adding** — `a = b` replaces; `a += b` adds. Get them mixed up and you lose all previous handlers.
- **Raising a null event** — `OnProgress(...)` throws if `OnProgress` is null. Use `OnProgress?.Invoke(...)`.
- **Forgetting to unsubscribe** — events can pin objects in memory (subscriber lives as long as publisher's invocation list holds the delegate). See [§05](05-Events.md).
- **Delegate equality with lambdas** — each `() => x` is a new instance unless captured/cached. Don't rely on lambda identity.
- **Multicast with non-void return** — only the last return is observed. Use `GetInvocationList()` if you need all.

---

## Performance

- Delegate creation: ~24-32 bytes per instance.
- Invocation: one indirect call (~few nanoseconds).
- The JIT optimizes "called many times" delegates by potentially inlining the target — sometimes.
- For tight loops in hot code, a virtual method on an interface can be slightly faster (devirtualization opportunities). For most code, delegate-vs-virtual is a wash.

---

## When to use delegates

✓ Strategy / callback patterns — inject behavior.
✓ LINQ-style fluent APIs.
✓ Events (use `event` keyword for the encapsulation).
✓ Memoization / function composition / functional patterns.

✗ When an interface with one method would be just as readable AND multiple methods might be needed later.
✗ When the behavior has state — a small class with a method might be clearer.
✗ For "many handlers" — events with explicit subscribe/unsubscribe semantics.

→ Next: [02-Lambdas.md](02-Lambdas.md)
