# Chapter 05 — Questions

> Drilling for everything in Chapter 05.

---

## Delegates

**Q1.** What does a delegate hold internally?
<details><summary>Answer</summary>A target reference (the receiver for instance methods, null for static), a function pointer (or `MethodInfo`), and a pointer to the previous delegate in the invocation list (for multicast).</details>

**Q2.** What's the difference between `a += b` and `a = b` on a delegate?
<details><summary>Answer</summary>`a = b` replaces a's invocation list with b's. `a += b` appends b's methods to a's list. Delegates are immutable; `+=` returns a new combined delegate and reassigns it.</details>

**Q3.** Predict the output:
```csharp
Func<int> f = () => 1;
f += () => 2;
f += () => 3;
Console.WriteLine(f());
```
<details><summary>Answer</summary>**3**. Multicast delegates with non-void returns only return the LAST result. To see all returns, iterate `f.GetInvocationList()`.</details>

**Q4.** Why is `Func<int, int>` not the same as `Action<int>`?
<details><summary>Answer</summary>Different signatures: `Func<int, int>` returns int; `Action<int>` returns void. They're distinct delegate types — not assignable to each other.</details>

---

## Lambdas

**Q5.** What's a `static` lambda? When should you use it?
<details><summary>Answer</summary>A lambda marked `static` cannot capture local variables. Use it whenever you don't need captures — the compiler caches the delegate (one allocation per program instead of per call) and verifies no accidental captures.</details>

**Q6.** What does this print?
```csharp
int x = 5;
Func<int> get = () => x;
x = 99;
Console.WriteLine(get());
```
<details><summary>Answer</summary>`99`. The lambda captures the **variable** x, not its value at the time of capture. Changing x later changes what get() returns.</details>

**Q7.** Why can't you put `await Task.Delay(...)` inside an expression-bodied lambda assigned to a `Func<int>`?
<details><summary>Answer</summary>`async` lambdas must return `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, or `void` (event handlers). `Func<int>` returns int — not compatible. Use `Func<Task<int>>` instead.</details>

---

## Closures

**Q8.** The classic trap — what does this print?
```csharp
var funcs = new List<Func<int>>();
for (int i = 0; i < 3; i++) funcs.Add(() => i);
foreach (var f in funcs) Console.Write(f());
```
<details><summary>Answer</summary>`333`. All three lambdas captured the same `i` variable. After the loop, `i == 3`. Fix: copy to a fresh local inside the loop (`int copy = i; funcs.Add(() => copy);`), or use `foreach` which gives a fresh variable per iteration.</details>

**Q9.** What's the cost of a lambda that captures locals?
<details><summary>Answer</summary>The compiler synthesizes a "closure class" holding the captured variables. Each call allocates: (1) the closure object and (2) the delegate. Two heap allocations per invocation site execution. Hot paths should avoid this; static lambdas are zero-allocation.</details>

**Q10.** Can a `Span<T>` be captured by a lambda?
<details><summary>Answer</summary>No. `Span<T>` is a `ref struct` — stack-only. Lambdas capture into heap-allocated closure objects. The compiler refuses. Copy to a regular value first.</details>

---

## Events

**Q11.** Why use `event` instead of a plain public delegate field?
<details><summary>Answer</summary>`event` restricts external code to `+=` and `-=` only. With a plain delegate field, external callers could (1) reassign it (blowing away subscribers) or (2) invoke it (faking events). The `event` modifier enforces the pub-sub contract.</details>

**Q12.** Why does this leak memory?
```csharp
var pub = new LongLivedPublisher();
for (int i = 0; i < 10000; i++) {
    var sub = new ShortLivedSubscriber(pub);  // subscribes in ctor
}
```
<details><summary>Answer</summary>Each subscriber's `OnUpdated` handler is added to the publisher's delegate list. The publisher's delegate references the subscriber's `this`. Even though `sub` goes out of scope, the publisher pins each subscriber alive. Fix: implement `IDisposable` on the subscriber, unsubscribe in `Dispose`, use `using`.</details>

**Q13.** Why do you need to keep a reference to a lambda to unsubscribe?
<details><summary>Answer</summary>Each lambda literal is a NEW delegate instance. `pub.X += (s, e) => {};` then `pub.X -= (s, e) => {};` doesn't unsubscribe — the two lambdas are different delegates. To unsubscribe, store the lambda in a variable: `EventHandler h = (s,e) => {}; pub.X += h; ... pub.X -= h;`.</details>

**Q14.** What's the modern pattern to raise an event safely?
<details><summary>Answer</summary>`MyEvent?.Invoke(this, args);` — the null-conditional `?.` checks for null subscribers, and the compiler snapshots the field before invoking (preventing race conditions if a subscriber unsubscribes mid-raise).</details>

---

## Local functions

**Q15.** Why is `static int Helper(int x) => x * x;` inside a method better than `Func<int, int> helper = x => x * x;`?
<details><summary>Answer</summary>The static local function compiles to a private static method on the enclosing class — zero allocation when called by name. The lambda allocates at least the delegate; if it captures, also a closure object. Local functions are also debuggable by name and can recurse.</details>

**Q16.** Why does this pattern matter?
```csharp
public IEnumerable<int> Range(int start, int count) {
    if (count < 0) throw new ArgumentException();
    return Impl();

    IEnumerable<int> Impl() {
        for (int i = 0; i < count; i++) yield return start + i;
    }
}
```
<details><summary>Answer</summary>An iterator method's body doesn't run until the first `MoveNext`. Putting argument validation in the iterator body delays the exception until iteration — surprising. Splitting into an outer method (eager validation) and a local-function iterator (deferred) gives caller-friendly fail-fast semantics.</details>

---

## Expression trees

**Q17.** What's the difference between `Func<int, int>` and `Expression<Func<int, int>>`?
<details><summary>Answer</summary>`Func<int, int>` is a compiled delegate — invoke it to execute. `Expression<Func<int, int>>` is a data structure describing the lambda body — you can inspect it, traverse it, translate it (to SQL), or `.Compile()` it to a delegate.</details>

**Q18.** Why can't EF Core translate this?
```csharp
db.Users.Where(u => SomeMethod(u) > 0);
```
<details><summary>Answer</summary>EF receives an expression tree. The tree includes a `MethodCallExpression` to `SomeMethod`, but EF doesn't know how to translate `SomeMethod` to SQL — it's an arbitrary C# method. The provider throws (or falls back to in-memory evaluation). Only operations the provider recognizes can be translated.</details>

**Q19.** Predict: which is faster after the first call?
- `var f = expr.Compile(); for (...) f(x);`
- `var f = expr.Compile(); ... ` and `for (...) f(x);`

(Both look the same, the question is — should you call `.Compile()` inside the loop?)
<details><summary>Answer</summary>**Outside the loop, dramatically.** `.Compile()` takes hundreds of microseconds. Inside the loop, every iteration recompiles. Outside, you pay once.</details>

---

## Synthesis

**Q20.** Write a generic `Try<T>` that takes a `Func<T>` and a fallback, catches any exception, and returns either the result or the fallback.
<details><summary>Answer</summary>
```csharp
public static T Try<T>(Func<T> action, T fallback) {
    try { return action(); }
    catch { return fallback; }
}

int n = Try(() => int.Parse(input), -1);
```
</details>

**Q21.** Explain the difference between a static lambda, an instance method-group conversion, and a closure-capturing lambda in terms of allocations.
<details><summary>Answer</summary>
- **Static lambda** (no captures): one allocation for the program (compiler-cached delegate).
- **Method-group conversion** (e.g., `Func<int,int> f = SomeMethod;`): one delegate allocation per call site execution; for static methods modern compilers often cache.
- **Capturing lambda** (`x => x + localVar`): one closure-class allocation + one delegate allocation per execution of the enclosing scope.
</details>

**Q22.** A coworker writes:
```csharp
public class Service {
    public event EventHandler? Started;
    public void Start() { Started(this, EventArgs.Empty); }
}
```
What's wrong?
<details><summary>Answer</summary>If no subscribers exist, `Started` is null → NullReferenceException. Use `Started?.Invoke(this, EventArgs.Empty);` (and the compiler will also generate the thread-safe snapshot semantics).</details>

**Q23.** Why do recursive lambdas need special handling, while recursive local functions don't?
<details><summary>Answer</summary>A lambda doesn't have a name to refer to itself. A local function is named, so it can call itself directly:
```csharp
int Fib(int n) => n <= 1 ? n : Fib(n - 1) + Fib(n - 2);   // works
```
For recursive lambdas you'd need a `Func<int, int>` variable, but you can't use it inside its own initializer:
```csharp
Func<int, int> fib = null!;
fib = n => n <= 1 ? n : fib(n - 1) + fib(n - 2);   // works, but verbose
```
</details>

**Q24.** When would you write a custom delegate type instead of using `Func<>` or `Action<>`?
<details><summary>Answer</summary>
- When the signature has a meaningful name (`TradeExecutedHandler` is more readable than `Action<object, TradeEventArgs>`).
- When you want attributes on the delegate type or its parameters.
- When IntelliSense would benefit from parameter names visible in tooling.
- For event handlers in a specific framework convention.

Otherwise, `Func`/`Action` are the modern default.
</details>

**Q25.** Open-ended: explain how an `IQueryable` provider uses expression trees to translate `db.Users.Where(u => u.IsActive && u.Age > 18).ToList()` into SQL.
<details><summary>Sample answer</summary>
1. `db.Users` is an `IQueryable<User>` whose `Expression` describes "select all from Users."
2. `.Where(u => ...)` is an extension method that receives the lambda **as an Expression&lt;Func&lt;User, bool&gt;&gt;** (because IQueryable's Where takes an Expression, unlike IEnumerable's which takes a Func).
3. It wraps the IQueryable's existing Expression in a new MethodCallExpression representing the Where, with the predicate tree attached.
4. Returns a new IQueryable whose Expression is the combined tree.
5. `.ToList()` triggers the provider to walk the expression tree, recognize the Where + the lambda body (BinaryExpression: AndAlso of two comparisons), emit `SELECT ... WHERE IsActive = 1 AND Age > 18`, execute, and materialize results.

The key insight: the lambda is never compiled to IL. It exists as a tree, inspected and translated by the provider.
</details>

---

→ [Coding.md](Coding.md)
