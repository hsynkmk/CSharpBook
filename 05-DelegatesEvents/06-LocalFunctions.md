# Local Functions

## What it is

A **local function** is a method declared inside another method. It's named (unlike a lambda), can be recursive, and can capture variables from the enclosing scope — but only the enclosing method can call it.

```csharp
public int Process(int n) {
    int Helper(int x) => x * x + 1;
    int Squared(int x) => x * x;

    return Helper(n) + Squared(n);
}
```

Local functions arrived in C# 7 (2017). They're an underused tool — often a better choice than lambdas or private methods when the logic is small and tightly scoped.

---

## Why they exist

Before local functions, helpers inside a method had to be either:

- **Private methods of the class** — visible to other methods, polluting the class's surface.
- **Lambdas assigned to a local Func** — allocates a closure object, less debug-friendly, restricted to expression-typed signatures.

Local functions give you a third option: a normal-method-shaped helper that lives entirely inside its enclosing method.

```csharp
// Pre-C# 7: private method
private int Helper(int x) => x * x + 1;
public int Process(int n) => Helper(n) + Helper(n+1);

// Or: lambda
public int Process(int n) {
    Func<int, int> helper = x => x * x + 1;
    return helper(n) + helper(n+1);
}

// C# 7+: local function
public int Process(int n) {
    int Helper(int x) => x * x + 1;
    return Helper(n) + Helper(n+1);
}
```

The third form is the cleanest:
- Helper is encapsulated, not exposed.
- No delegate allocation.
- Helper can be recursive.
- Debugger gives it a real method name.

---

## Syntax

Same shape as a regular method:

```csharp
public void M() {
    int Cube(int x) => x * x * x;

    int Add(int a, int b) {
        return a + b;
    }

    bool IsEven(int x) => x % 2 == 0;

    // Recursive — refers to itself by name
    int Fib(int n) => n <= 1 ? n : Fib(n - 1) + Fib(n - 2);

    Console.WriteLine(Fib(10));
}
```

Local functions can have:
- Any return type, including `void`, `Task`, `IEnumerable<T>` (with `yield`), etc.
- Any number of parameters.
- Default values.
- Generic type parameters (with constraints).
- Attributes.

```csharp
public void M() {
    T Largest<T>(T[] arr) where T : IComparable<T> {
        T best = arr[0];
        for (int i = 1; i < arr.Length; i++)
            if (arr[i].CompareTo(best) > 0) best = arr[i];
        return best;
    }

    Largest(new[] { 1, 3, 2 });
    Largest(new[] { "a", "c", "b" });
}
```

---

## Captures

By default, local functions can capture variables from the enclosing scope, just like lambdas:

```csharp
public Func<int> Counter(int start) {
    int n = start;
    int Increment() => ++n;
    return Increment;
}
```

The local function references `n` from the enclosing method. The compiler synthesizes a closure object (just like for a lambda) to hold `n`.

### `static` local functions — forbid captures

C# 8 added `static` to local functions. A static local function **cannot capture** anything from the enclosing scope:

```csharp
public int Process(int n) {
    static int Helper(int x) => x * x + 1;   // OK: no captures
    int factor = 10;
    static int Bad(int x) => x * factor;     // ❌ — can't capture factor

    return Helper(n);
}
```

**Why this matters**: when a local function doesn't capture, the compiler can compile it as a regular private static method on the enclosing class. **No closure allocation. No delegate allocation. Same speed as a regular method call.**

Compare:
- Local function (non-static) called via name → maybe no allocation (depending on whether captures exist).
- Lambda assigned to a Func → at least one delegate allocation.
- Static local function → guaranteed zero allocation.

**Recommendation**: mark local functions `static` whenever possible. The compiler verifies you didn't accidentally capture.

```csharp
public int Compute(int n) {
    static int Helper(int x) => x * x + 1;   // ← always prefer this form when no captures needed
    return Helper(n);
}
```

---

## Local function vs lambda

| Feature | Local function | Lambda |
|---|---|---|
| Named (debugger-friendly) | ✓ | ✗ (compiler-generated name) |
| Recursive | ✓ (refers to itself by name) | ✗ (no name to recurse to) |
| Multiple return paths | ✓ (any kind of body) | ✓ |
| Forbid captures | `static` modifier | `static` modifier (C# 9+) |
| Default values, attributes, generics | ✓ | (lambdas can have attributes since C# 10) |
| Allocation when called directly | None if `static`, else closure if captures | Always at least the delegate; closure if captures |
| Allocation when passed as delegate | One delegate (potentially +closure) | Same |
| `yield return` | ✓ | ✗ (lambdas can't be iterators) |
| Can use `await` | ✓ (with `async` modifier) | ✓ (with `async`) |

The big advantages of local functions over lambdas:
1. **Iterators**: lambdas can't `yield return`; local functions can.
2. **Recursion**: lambdas can't refer to themselves; local functions can.
3. **Zero allocation when static**: lambdas always allocate at least a delegate when used as one.
4. **Better debugger experience**: real names in stack traces.

The advantage of lambdas: brevity for short, anonymous bits of behavior.

---

## Iterators inside local functions

A pattern that needs local functions specifically — combining iterator methods (`yield return`) with eager argument checks:

```csharp
public IEnumerable<int> Range(int start, int count) {
    if (count < 0) throw new ArgumentOutOfRangeException(nameof(count));

    return RangeIterator();

    IEnumerable<int> RangeIterator() {
        for (int i = 0; i < count; i++) yield return start + i;
    }
}
```

Why the split? In an iterator method, the body doesn't run until you start enumerating. So if you put the validation inside the iterator body, callers don't see the exception until they iterate — which is surprising:

```csharp
public IEnumerable<int> Range(int start, int count) {
    if (count < 0) throw new ArgumentOutOfRangeException();   // ⚠ won't throw until iteration starts
    for (int i = 0; i < count; i++) yield return start + i;
}

var seq = Range(0, -1);   // does NOT throw here
foreach (var x in seq) { } // throws here — surprising!
```

The local-function pattern splits validation (eager) from iteration (deferred):

```csharp
public IEnumerable<int> Range(int start, int count) {
    if (count < 0) throw new ArgumentOutOfRangeException();   // throws immediately

    return Impl();

    IEnumerable<int> Impl() {
        for (int i = 0; i < count; i++) yield return start + i;
    }
}

Range(0, -1);   // throws now
```

Used in the BCL's LINQ implementations and many libraries.

---

## Async local functions

Same idea, but with `async`:

```csharp
public Task<int> ComputeAsync(int n) {
    if (n < 0) throw new ArgumentException();   // eager validation

    return Impl();

    async Task<int> Impl() {
        await Task.Delay(100);
        return n * n;
    }
}
```

Without the split, the validation would happen only when the task starts (a `Task<int>` from `async Task<int>` returns lazily).

---

## Placement

Local functions are typically placed at the bottom of the enclosing method, after the main logic. This keeps the main flow front and center:

```csharp
public int Compute(int[] arr) {
    int sum = 0;
    foreach (var x in arr) sum += Square(x);
    return sum;

    static int Square(int n) => n * n;
}
```

C# allows them anywhere a statement is legal, but the bottom is conventional.

You can also nest local functions:

```csharp
public void M() {
    int Outer(int x) {
        int Inner(int y) => y * 2;
        return Inner(x) + 1;
    }
    Console.WriteLine(Outer(5));
}
```

Inner can only be called from inside Outer. Less common but legal.

---

## Internals — how they compile

### Non-capturing local function

```csharp
public int M(int n) {
    static int Sq(int x) => x * x;
    return Sq(n);
}
```

Compiles to a private static method:

```csharp
public int M(int n) => <M>g__Sq|0_0(n);
private static int <M>g__Sq|0_0(int x) => x * x;
```

The call is a direct method call — same speed as any private static method.

### Capturing local function

```csharp
public int M(int n) {
    int factor = 10;
    int Multiply(int x) => x * factor;
    return Multiply(n);
}
```

Compiles to a method on a closure struct (often a struct, since the function doesn't escape):

```csharp
public int M(int n) {
    var closure = new <>c__DisplayClass0_0 { factor = 10 };
    return <M>g__Multiply|0_0(n, ref closure);
}

private struct <>c__DisplayClass0_0 { public int factor; }
private static int <M>g__Multiply|0_0(int x, ref <>c__DisplayClass0_0 c) => x * c.factor;
```

Notice: when a local function is called **directly by name** (not converted to a delegate), the compiler can use a **closure struct** passed by `ref` — no heap allocation. This is a significant optimization.

If you convert to a delegate (`Func<int, int> f = Multiply;`), the compiler needs a heap closure (delegates can't capture `ref struct`s). Then the local function is no cheaper than a lambda.

---

## Common patterns

### Iterator + validation

(See above.)

### State machine

```csharp
public Func<int> MakeSequence() {
    int n = 0;
    int Next() => ++n;
    return Next;   // delegate conversion — heap closure allocated
}
```

If you don't need to return it as a delegate, the local function is cheaper.

### Recursion

```csharp
public int Factorial(int n) {
    if (n < 0) throw new ArgumentException();
    return Compute(n);

    static int Compute(int x) => x <= 1 ? 1 : x * Compute(x - 1);
}
```

Recursive helper is encapsulated and validated.

### Helper extraction

```csharp
public string FormatReport(IEnumerable<Order> orders) {
    var sb = new StringBuilder();

    foreach (var order in orders) {
        AppendLine($"Order {order.Id}: {Money(order.Total)}");
    }
    return sb.ToString();

    void AppendLine(string s) => sb.AppendLine(s);
    static string Money(decimal m) => m.ToString("C");
}
```

Two helpers — one captures (`AppendLine` references `sb`), one doesn't (`Money` is static).

---

## When to use a local function

✓ Helper logic used only inside one method.
✓ Iterators/async methods that need eager argument validation.
✓ Recursive helpers.
✓ Where a lambda would otherwise allocate a closure unnecessarily.

✗ When the helper is general enough to be shared — make it a private method.
✗ When the helper would be lifetime-tied to an object — instance method.
✗ For one-line lambda-like uses — a lambda might be cleaner.

---

## Common bugs

- **Forgetting `static` and unintentionally capturing** — leads to closure allocation.
- **Calling a non-static local function but converting to a delegate** later — closure now needs to be on the heap.
- **Recursive lambda confusion** — must use `Func<...>` and self-reference can be awkward. Local functions handle it naturally.
- **`yield return` in main body of an async method** — illegal; use a local function for the iterator part.

---

## Performance

- Static local function called by name: **zero overhead vs a regular method call**.
- Capturing local function called by name: stack-allocated closure struct (often), passed by ref — much cheaper than lambda's heap allocation.
- Local function converted to delegate: same cost as a lambda (delegate + closure).

**Rule of thumb**: if you have a small helper you'd otherwise write as a lambda, and you'll call it by name (not pass as a delegate), a `static` local function is strictly better.

→ Next: [07-AnonymousMethods.md](07-AnonymousMethods.md)
