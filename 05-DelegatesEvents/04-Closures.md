# Closures

## What it is

A **closure** is a function bundled with the variables it captured from its enclosing scope. The compiler does this automatically when a lambda or local function references variables from an outer method.

```csharp
public Func<int, int> Multiplier(int factor) {
    return x => x * factor;   // captures factor
}

var times10 = Multiplier(10);
var times100 = Multiplier(100);

Console.WriteLine(times10(5));    // 50
Console.WriteLine(times100(5));   // 500
```

Each call to `Multiplier` returns a *separate* closure with its own captured `factor`.

Closures are how you do "functions with state" without writing a class. They're the engine behind most LINQ patterns, callbacks, and event handlers.

---

## Why they exist

Without closures, lambdas could only refer to global state or method parameters. That'd be a severe restriction. With closures:

```csharp
public Func<int> Counter() {
    int n = 0;
    return () => ++n;   // captures n
}

var c = Counter();
Console.WriteLine(c());   // 1
Console.WriteLine(c());   // 2
Console.WriteLine(c());   // 3
```

A counter with private state, in a single function call. Pre-closures, you'd write a class. Closures collapse the ceremony.

---

## What gets captured

Lambdas capture:
- **Local variables** of the enclosing method.
- **Parameters** of the enclosing method.
- **The `this` reference** if the lambda is inside an instance method and references an instance member.
- **`out` and `ref` locals are NOT captured directly** (forbidden — see below).

```csharp
public class Calculator {
    public int Bias = 10;

    public Func<int, int> MakeAdder(int extra) {
        int local = 5;
        return x => x + local + extra + Bias;   // captures local, extra, AND this (for Bias)
    }
}
```

Three captures: `local`, `extra`, `this` (the latter implicitly because of `Bias`).

---

## Captures variables, NOT values

The single most-important fact about closures:

```csharp
int x = 5;
Func<int> get = () => x;

x = 99;
Console.WriteLine(get());   // 99 (NOT 5)
```

The closure captures the **variable** `x` — its identity — not the value that was there when the lambda was created. Changing `x` afterward changes what `get()` returns.

This is what enables the counter pattern above (the lambda mutates `n` across calls). It's also the source of the famous trap below.

---

## The classic `for` loop trap

```csharp
var funcs = new List<Func<int>>();
for (int i = 0; i < 3; i++) {
    funcs.Add(() => i);
}

foreach (var f in funcs)
    Console.WriteLine(f());
```

Predict: 0, 1, 2?

**Actual: 3, 3, 3.**

Why? All three lambdas captured the **same variable** `i`. By the time you invoke them, the loop has finished and `i == 3`. All three see the final value.

### Fix 1 — introduce a fresh local

```csharp
for (int i = 0; i < 3; i++) {
    int copy = i;
    funcs.Add(() => copy);
}
// Output: 0, 1, 2
```

`copy` is a **new variable per iteration**. Each lambda captures its own copy.

### Fix 2 — use `foreach` (gives a fresh variable per iteration since C# 5)

```csharp
foreach (var i in Enumerable.Range(0, 3))
    funcs.Add(() => i);
// Output: 0, 1, 2
```

`foreach`'s loop variable is **scoped to the body** — it's a fresh variable per iteration. (This wasn't the case pre-C# 5; the language was officially fixed in 2012 to match what most developers expected.)

### Why classic `for` is different

In `for (int i = 0; ...)`, `i` is declared **once** outside the body. Each iteration mutates the same variable. So all lambdas see the same variable, which holds the final value after the loop.

**Rule**: any lambda you create inside a `for` loop body and store somewhere should explicitly copy the loop variable to a new local.

---

## Capture and `ref` / `out`

```csharp
public void Test(ref int n) {
    Func<int> f = () => n;   // ❌ compile error
}
```

You can't capture a `ref` or `out` parameter (or a `ref` local) in a lambda. The compiler refuses because the closure object lives on the heap, but `ref` points to stack/temporary memory — the reference could outlive the storage.

For escape-analysis-friendly patterns, copy to a local first:

```csharp
public void Test(ref int n) {
    int copy = n;
    Func<int> f = () => copy;   // OK
}
```

Same for `Span<T>` (a `ref struct`) — can't be captured by a lambda.

---

## Multiple lambdas sharing one closure

```csharp
public (Action<int> Set, Func<int> Get) MakeCounter() {
    int n = 0;
    return (
        Set: v => n = v,
        Get: () => n
    );
}

var (set, get) = MakeCounter();
Console.WriteLine(get());    // 0
set(42);
Console.WriteLine(get());    // 42
```

Both lambdas reference the same `n` — they share one closure class. Mutations through one are visible through the other.

This is essentially private encapsulation: the variable `n` is only reachable through these two lambdas, no one else.

---

## Capturing `this` is expensive (long-term)

```csharp
public class BigClass {
    public byte[] Buffer = new byte[100_000_000];   // 100 MB

    public Func<int> MakeAccessor() {
        return () => Buffer.Length;   // captures `this` → captures Buffer
    }
}

var b = new BigClass();
Func<int> f = b.MakeAccessor();
b = null;                 // can the Buffer be GC'd?
```

No — `f` holds a closure that references `this` (the BigClass instance) for the `Buffer` field. Even though `b = null`, the closure keeps `b` (and its 100 MB Buffer) alive until `f` is unreferenced.

This is a classic memory leak in long-lived UIs and async code. Be deliberate about what captures.

### Avoidance

Capture only what you need:

```csharp
public Func<int> MakeAccessor() {
    int length = Buffer.Length;   // capture length, not this
    return () => length;
}
```

Now the closure captures just an int. `this` and Buffer are not pinned by it.

---

## `static` lambdas — forbid captures

C# 9 added `static` modifier on lambdas:

```csharp
Func<int, int> sq = static x => x * x;   // OK — no captures

int factor = 10;
Func<int, int> scale = static x => x * factor;   // ❌ — captured factor
```

Use `static` whenever the lambda doesn't need captures. Benefits:
- Compile-time enforcement that no captures sneak in (no accidental memory leak).
- The compiler caches the delegate — one allocation per program, not per call.

```csharp
items.Where(static x => x > 0);   // Where receives a cached delegate
```

If the cleanup story matters or you're on a hot path, prefer `static` lambdas.

---

## Local functions and captures

Local functions can also capture, but with one advantage: **`static` local functions ban captures** without forcing you to convert to a delegate:

```csharp
int factor = 10;

static int Pure(int x) => x * x;   // no captures — verifiable
int Scaled(int x) => x * factor;    // captures factor

Pure(5);     // OK
Scaled(5);   // OK
```

A `static` local function:
- No closure class allocated when used directly (no delegate conversion).
- Compiles to a regular static method.
- Compiler enforces no captures.

Use `static` local functions for **pure helper functions** within a method body — zero allocation, clean syntax. See [§06](06-LocalFunctions.md).

---

## Internals — how the compiler implements captures

Three strategies, depending on what's captured:

### No captures
Lambda compiles to a private static method:

```csharp
Func<int, int> sq = x => x * x;
```

→ One static method, one (cached) delegate. Lifetime = forever.

### Captures `this`
Lambda compiles to a private instance method on the enclosing class:

```csharp
public class C {
    public int M = 10;
    public Func<int, int> Make() => x => x * M;
}
```

→ The lambda becomes `C.<>b__0` (a private method). The delegate is constructed with `this` as its target. Allocation: just the delegate, no closure class.

### Captures locals
Lambda compiles to a method on a **closure class**:

```csharp
public Func<int, int> Make(int factor) =>
    x => x * factor;
```

The compiler synthesizes:

```csharp
public Func<int, int> Make(int factor) {
    var closure = new <>c__DisplayClass0_0 { factor = factor };
    return new Func<int, int>(closure.<Make>b__0);
}

private sealed class <>c__DisplayClass0_0 {
    public int factor;
    public int <Make>b__0(int x) => x * factor;
}
```

The closure class holds the captured variables; the method-on-the-closure is the actual lambda body. Allocations: one closure + one delegate per call.

### Multiple captures, multiple lambdas

If multiple lambdas in the same method scope capture overlapping variables, the compiler creates **one** closure class shared by all of them:

```csharp
public (Action<int> Set, Func<int> Get) MakeCounter() {
    int n = 0;
    return (v => n = v, () => n);
}
```

→ One `<>c__DisplayClass` with `int n`. Two methods on it (for the two lambdas). One closure allocation; two delegate allocations.

### Hoisting

Captured local variables are **hoisted** from the stack to the closure object's fields. The compiler rewrites every reference to them — inside the method body AND inside the lambdas — to access the closure's field instead:

```csharp
public Func<int> Increment() {
    int n = 0;
    n++;             // becomes: closure.n++
    return () => ++n; // becomes: () => ++closure.n
}
```

After the method exits, `n` (now `closure.n`) lives as long as the returned delegate references the closure.

### Why structs aren't captureable

A struct field is a value. The closure would need to either copy it (changing semantics — see "captures variables, not values") or take a `ref` to it (which can't escape). C# chooses to capture variables by reference always, so structs work the same as classes for capture purposes — the **variable** is captured.

### Lifetimes and GC

The closure object is GC-tracked. As long as any delegate references the closure (directly or transitively), it stays alive. Frequently leaks if you forget to unsubscribe from events.

---

## Common patterns

### Memoization

```csharp
public static Func<T, R> Memoize<T, R>(Func<T, R> f) where T : notnull {
    var cache = new Dictionary<T, R>();
    return x => {
        if (!cache.TryGetValue(x, out var v)) cache[x] = v = f(x);
        return v;
    };
}
```

`cache` is captured in the lambda's closure. Each call to `Memoize` creates a separate cache.

### Counter / state machine

```csharp
public Func<int> Sequence() {
    int n = 0;
    return () => n++;
}

var seq = Sequence();
Console.WriteLine(seq());   // 0
Console.WriteLine(seq());   // 1
```

### Deferred computation

```csharp
public class Lazy<T> {
    private readonly Func<T> _factory;
    private T? _value;
    private bool _has;

    public Lazy(Func<T> factory) { _factory = factory; }
    public T Value {
        get {
            if (!_has) { _value = _factory(); _has = true; }
            return _value!;
        }
    }
}

var lazy = new Lazy<int>(() => ExpensiveCompute());
```

The compute closure captures whatever it needs and defers execution until access.

---

## Common bugs

- **`for` loop capture** — see the famous trap above. Use `foreach` or copy to local.
- **Capturing `this` accidentally** — keeps the host alive longer than expected.
- **Capturing `out`/`ref`** — illegal; copy to local first.
- **Heavy captures in hot paths** — allocates closure object per call. Profile.
- **Closures retaining big objects** — capture only what you need.
- **Async lambdas + closures + cancellation** — captured cancellation tokens behave normally; ensure you await rather than fire-and-forget.

---

## Performance

- **Static lambdas** (no captures, no `this`): one delegate allocation total (cached).
- **`this` captures**: one delegate allocation per call.
- **Local captures**: one closure class + one delegate per call.

For tight loops, the closure-allocation case dominates. Three mitigations:
1. Convert to `static` lambda if possible.
2. Move the captures into method parameters (and pass them in).
3. Use a struct enumerator + non-capturing pattern (advanced).

For most code, the cost is negligible compared to actual work. Profile first.

---

## When to use closures

✓ Strategy / callback / functional patterns.
✓ Encapsulating private state without writing a class.
✓ Composing transformations on data.

✗ Hot performance paths with very small bodies — measure the allocation overhead.
✗ When the captured state is "the type" — write a small class instead.
✗ Long-lived delegates capturing large state — they pin everything.

→ Next: [05-Events.md](05-Events.md)
