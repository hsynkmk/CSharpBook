# Anonymous Methods

## What it is

An **anonymous method** is a delegate body written inline using the `delegate(...) { ... }` syntax. C# 2 (2005) introduced them; C# 3 (2007) added lambdas, and lambdas are now strongly preferred.

```csharp
// Anonymous method (C# 2 syntax — legacy)
Func<int, int> sq = delegate(int x) { return x * x; };

// Lambda equivalent (modern)
Func<int, int> sq2 = x => x * x;
```

Both compile to the same kind of thing. The lambda is shorter, supports expression bodies, and works with expression trees. Anonymous methods are essentially obsolete except for one corner case (see below).

This file is short because the topic is mostly historical. Read it once, then ignore.

---

## Why they existed (and why we moved on)

C# 1 had only **named** methods. To pass behavior, you'd declare a method:

```csharp
// C# 1
public class Filter {
    public bool IsPositive(int n) => n > 0;
}

var f = new Filter();
list.Where(f.IsPositive);
```

Ceremony for one-line logic. C# 2 added anonymous methods so you could write:

```csharp
list.FindAll(delegate(int n) { return n > 0; });
```

Inline, no class needed. But "delegate(int n) { return n > 0; }" is wordy. C# 3 lambdas reduced it to:

```csharp
list.FindAll(n => n > 0);
```

Same semantics, much cleaner. Anonymous methods became redundant.

---

## Syntax

```csharp
delegate(int a, int b) { return a + b; }
delegate { return 42; }                   // no parameters needed
delegate(string s) {
    Console.WriteLine(s);
    Console.WriteLine(s.Length);
}                                          // multi-statement
```

A `delegate` keyword followed by an optional parameter list and a statement block.

---

## One quirk lambdas don't have

Anonymous methods can have **no parameter list at all** when the delegate type allows discarding parameters:

```csharp
public event EventHandler<MouseEventArgs> Click;

button.Click += delegate { Console.WriteLine("clicked"); };
```

The handler ignores the sender and event args. The `delegate { ... }` form drops the parameter list entirely.

Lambda equivalent — you'd have to provide them:

```csharp
button.Click += (_, _) => Console.WriteLine("clicked");   // C# 9 discard parameters
button.Click += (s, e) => Console.WriteLine("clicked");
```

The modern lambda discard `_` is essentially equivalent. So even this corner case is now lambda-friendly.

This is the only practical use of anonymous methods you might encounter in modern code, and even then a discard lambda is clearer.

---

## Internals

Anonymous methods compile **the same way as lambdas**:

```csharp
Func<int, int> sq = delegate(int x) { return x * x; };
```

→ Compiler generates a static (or instance) method named `<>c__DisplayClass0_0::<M>b__0`, and a delegate is constructed pointing to it. Identical compilation strategy to lambdas. The runtime treats them indistinguishably.

The compiler does the same closure handling, same caching, same allocation patterns. The only difference is the source-level syntax.

---

## When you'll see them

- **Old code** (pre-2007 or written by developers who learned C# 2 before C# 3).
- **Code generated** by tools that emit pre-C# 3 syntax.
- The "ignore all parameters" case in event handlers.

If you're writing new code, use lambdas (or local functions when those fit). If you maintain old code, refactor anonymous methods to lambdas as opportunities arise — they read better.

---

## Migration guide

```csharp
// Anonymous method                          Lambda equivalent
delegate { return 5; }                        () => 5
delegate(int x) { return x * x; }              x => x * x
delegate(int a, int b) { return a + b; }       (a, b) => a + b
delegate(int x) {                              x => {
    var y = x * 2;                                 var y = x * 2;
    return y;                                       return y;
}                                              }
delegate { Console.WriteLine("hi"); }           () => Console.WriteLine("hi")
// (ignored parameters — only anonymous methods could do this cleanly)
button.Click += delegate { Click(); };          button.Click += (_, _) => Click();
```

Visual Studio and Rider both offer one-click refactorings.

---

## A note on capture semantics

Anonymous methods capture variables the same way lambdas do — by reference to the variable, not by value. Same patterns, same pitfalls (see [§04 Closures](04-Closures.md)).

```csharp
int n = 0;
Action a = delegate { Console.WriteLine(n); };
n = 99;
a();   // prints 99 — captured the variable, not its value
```

Identical to a lambda.

---

## Why this file exists

For completeness. You should know:
- Anonymous methods are a legacy feature.
- They compile to the same thing as lambdas.
- The "discard parameters" syntax (`delegate { ... }`) is the one tiny edge case.
- In modern code, use lambdas.

That's it. Move on.

→ Next: [08-ExpressionTrees.md](08-ExpressionTrees.md)
