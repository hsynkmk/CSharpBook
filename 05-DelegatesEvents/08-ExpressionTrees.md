# Expression Trees

## What it is

An **expression tree** is a data structure representation of code — a tree where each node describes an operation (call, addition, comparison, etc.) and each leaf is a value or parameter. Where a delegate **executes** code, an expression tree **describes** code that *can* be executed, inspected, transformed, or translated.

```csharp
using System.Linq.Expressions;

// Lambda: a regular delegate, executes immediately
Func<int, int> compiledSquare = x => x * x;
Console.WriteLine(compiledSquare(5));   // 25

// Expression tree: a data structure describing the same lambda
Expression<Func<int, int>> exprSquare = x => x * x;
Console.WriteLine(exprSquare);          // x => (x * x) — prints the tree

// Compile to a delegate at runtime
var compiled = exprSquare.Compile();
Console.WriteLine(compiled(5));         // 25
```

Expression trees are how LINQ providers (EF Core, Cosmos, OData) turn C# lambdas into SQL or other backend queries. They're powerful but specialized.

---

## Why they exist

Consider:

```csharp
db.Users.Where(u => u.IsActive).ToList();
```

How does EF Core turn `u => u.IsActive` into `WHERE IsActive = 1` SQL?

If `Where` takes a `Func<User, bool>` (a compiled delegate), EF couldn't inspect the body. It would just have a black-box callback.

But `IQueryable<T>.Where` takes an `Expression<Func<User, bool>>` — an expression tree. EF receives the tree, walks it, sees `u.IsActive` is a property access, and emits the corresponding SQL.

That's the whole reason expression trees exist: **so libraries can inspect lambda bodies**.

---

## The shape of a tree

For `x => x * x`:

```
Lambda
├── Parameters: [x]
└── Body: BinaryExpression (Multiply)
    ├── Left: ParameterExpression (x)
    └── Right: ParameterExpression (x)
```

In C# code:

```csharp
Expression<Func<int, int>> e = x => x * x;
Console.WriteLine(e.Body.NodeType);            // Multiply
Console.WriteLine(((BinaryExpression)e.Body).Left);   // x
Console.WriteLine(((BinaryExpression)e.Body).Right);  // x
Console.WriteLine(e.Parameters[0].Name);       // x
```

The compiler synthesizes the tree at compile time. The library you pass it to can walk the tree at runtime.

---

## What can be in an expression tree

C# limits expression-tree lambdas to **single-expression** bodies (no statements):

```csharp
Expression<Func<int, int>> ok = x => x * x;
Expression<Func<int, int>> ok2 = x => x > 0 ? x : -x;

Expression<Func<int, int>> bad = x => {     // ❌ statement body
    int result = x * x;
    return result;
};
```

C# 4 added `Expression<Action<...>>` for void-returning ones (still expression-bodied).

Modern `Expression<T>` supports a subset of C# operations:
- Property and field access.
- Method calls.
- Binary and unary operators.
- Conditional (`?:`) expressions.
- New expressions (`new MyType(a, b)`).
- Member init / object initializers.
- Array creation and indexing.
- Casts.
- `null` checks.

Pattern matching, `await`, `yield`, statements, `using`, `try/catch` — **not** supported.

---

## Building expression trees by hand

The factory methods on `Expression`:

```csharp
using System.Linq.Expressions;

var x = Expression.Parameter(typeof(int), "x");
var square = Expression.Multiply(x, x);
var lambda = Expression.Lambda<Func<int, int>>(square, x);

Func<int, int> compiled = lambda.Compile();
Console.WriteLine(compiled(5));   // 25
```

Verbose, but powerful — you can construct trees from scratch at runtime, then compile them to delegates. Used by:

- ORM/query libraries.
- Object mappers (AutoMapper).
- Serializers.
- Anything that needs to generate code dynamically without the full Roslyn compile pipeline.

---

## Walking a tree

```csharp
Expression<Func<int, bool>> e = x => x > 0 && x < 100;

void Walk(Expression node, int depth = 0) {
    Console.WriteLine($"{new string(' ', depth * 2)}{node.NodeType}: {node}");
    switch (node) {
        case BinaryExpression b:
            Walk(b.Left, depth + 1);
            Walk(b.Right, depth + 1);
            break;
        case MethodCallExpression m:
            Walk(m.Object!, depth + 1);
            foreach (var a in m.Arguments) Walk(a, depth + 1);
            break;
        // ... etc.
    }
}

Walk(e.Body);
```

For real walking, derive from `ExpressionVisitor`:

```csharp
public class MyVisitor : ExpressionVisitor {
    protected override Expression VisitBinary(BinaryExpression node) {
        Console.WriteLine($"Binary: {node.NodeType}");
        return base.VisitBinary(node);   // recurse
    }
    protected override Expression VisitMember(MemberExpression node) {
        Console.WriteLine($"Member: {node.Member.Name}");
        return base.VisitMember(node);
    }
}

new MyVisitor().Visit(e);
```

`ExpressionVisitor` is the standard pattern. EF Core's query translator is a giant visitor walking trees and emitting SQL.

---

## A LINQ provider in miniature

The shape of how EF Core uses expression trees:

```csharp
public class MyQueryable<T> : IQueryable<T> {
    public Expression Expression { get; }
    public IQueryProvider Provider { get; }
    // ...

    public IEnumerable<T> Execute() {
        // Walk this.Expression
        // Translate to SQL
        // Execute SQL
        // Return results
    }
}
```

When you call `.Where(u => u.IsActive)` on an `IQueryable<User>`:
1. The `Where` extension method receives the expression tree.
2. It wraps the current `IQueryable.Expression` in a new `Call` expression representing the `Where`.
3. Returns a new `IQueryable` whose `Expression` is the wrapped tree.

When you call `.ToList()`:
1. The provider walks the accumulated tree.
2. Emits SQL.
3. Executes.

The lambda inside `Where` is never compiled to a delegate — it stays as a tree, translated to SQL.

---

## Compiling and executing a tree

You can compile any expression tree to a delegate:

```csharp
Expression<Func<int, int>> e = x => x * x;
Func<int, int> f = e.Compile();   // generates IL at runtime

Console.WriteLine(f(5));   // 25
```

`Compile()` emits IL using `System.Reflection.Emit` under the hood and returns a delegate. It's slow (compilation is hundreds of microseconds), so you usually cache the compiled delegate.

For really hot paths, libraries like FastExpressionCompiler are faster than the built-in compiler.

---

## Common bugs

### Lambda body restrictions

```csharp
Expression<Func<int, int>> e = x => {   // ❌ statement body
    return x * x;
};
```

Must be expression-bodied. Refactor to one expression.

### Closure over a local

```csharp
int factor = 10;
Expression<Func<int, int>> e = x => x * factor;   // OK; closure becomes part of tree
```

This works — the compiler builds a tree that includes a member access to a captured local (the captured local is hoisted to a closure class, and the tree accesses the field). When the library translates this to SQL, it might evaluate the constant up front.

### Trying to use unsupported C# features

```csharp
Expression<Func<int, int>> e = x => x switch { > 0 => x, _ => -x };   // ❌
```

Switch expressions, ranges, pattern matching, tuples in expression trees — limited or unsupported. Stick to traditional operators.

### Performance trap with Compile

Compiling an expression tree is expensive. Don't do it in a loop:

```csharp
// ⚠ — slow
foreach (var x in items) {
    Expression<Func<int, int>> e = ... ;
    var f = e.Compile();
    f(x);
}

// ✓ — compile once, reuse
var compiled = e.Compile();
foreach (var x in items) compiled(x);
```

---

## Internals — what the compiler does

For `Expression<Func<int, int>> e = x => x * x;`:

The compiler does NOT emit a method. It emits **code that constructs the tree at runtime**:

```il
.locals init (System.Linq.Expressions.ParameterExpression V_0,
              System.Linq.Expressions.Expression`1<System.Func`2<int32, int32>> V_1)

ldtoken int32                    // typeof(int)
call class Type::GetTypeFromHandle(...)
ldstr "x"
call Expression::Parameter(Type, string)
stloc.0                          // V_0 = ParameterExpression for x

ldloc.0
ldloc.0
call Expression::Multiply(Expression, Expression)   // x * x
ldc.i4.1
newarr ParameterExpression
... fill with [x] ...
call Expression::Lambda<Func<int, int>>(Expression body, ParameterExpression[] parameters)
stloc.1                          // V_1 = Expression<Func<int, int>>
```

The runtime walks this IL to construct the tree object graph each time the line executes. **This is allocation-heavy** — every expression tree literal is many allocations.

For libraries that build trees once and reuse, this is fine. For hot paths, expression trees are NOT a free abstraction.

---

## Common use cases (when to reach for expression trees)

✅ **Building a library that needs to introspect lambda code** — query providers, mappers, validators.

✅ **Generating code at runtime** — building delegates from configuration or rules.

✅ **Custom serializers / converters** — turning lambdas into different code (SQL, JSON paths, etc.).

❌ **Just to "store" a function** — use a delegate. Expression trees are heavyweight for this.

❌ **As a "free" performance boost** — building a tree, compiling it, and invoking is much slower than a direct call.

❌ **In hot paths** — see above.

---

## A real-world example

A poor-man's predicate builder:

```csharp
public static class PredicateBuilder {
    public static Expression<Func<T, bool>> True<T>() => x => true;
    public static Expression<Func<T, bool>> False<T>() => x => false;

    public static Expression<Func<T, bool>> And<T>(
        this Expression<Func<T, bool>> a,
        Expression<Func<T, bool>> b) {
        var invoked = Expression.Invoke(b, a.Parameters);
        return Expression.Lambda<Func<T, bool>>(
            Expression.AndAlso(a.Body, invoked), a.Parameters);
    }

    public static Expression<Func<T, bool>> Or<T>(
        this Expression<Func<T, bool>> a,
        Expression<Func<T, bool>> b) {
        var invoked = Expression.Invoke(b, a.Parameters);
        return Expression.Lambda<Func<T, bool>>(
            Expression.OrElse(a.Body, invoked), a.Parameters);
    }
}

// Build a query dynamically
Expression<Func<User, bool>> predicate = PredicateBuilder.True<User>();
if (filter.Active.HasValue)
    predicate = predicate.And(u => u.IsActive == filter.Active);
if (!string.IsNullOrEmpty(filter.NameContains))
    predicate = predicate.And(u => u.Name.Contains(filter.NameContains));

var results = db.Users.Where(predicate).ToList();   // EF Core translates the combined tree to SQL
```

Dynamically composed query, translated to a single SQL query. The expression-tree mechanism is what makes this possible.

---

## Performance summary

- Building an expression tree (the C# literal `Expression<...> = x => ...`): hundreds of nanoseconds per allocation.
- `Compile()`: ~hundred microseconds (uses Reflection.Emit). Cache the result.
- Calling a compiled delegate: same as any delegate (~few ns).
- Walking a tree (visitor): proportional to tree size.

For one-off use, trees are slow. For long-lived translated queries (EF Core), the upfront cost is amortized over thousands of executions.

---

## When to use what

| Need | Use |
|---|---|
| Execute behavior | Delegate (`Func<>`, `Action<>`) |
| Inspect / translate behavior | Expression tree (`Expression<Func<>>`) |
| Generate code at runtime | Expression trees (or Reflection.Emit / Source Generators) |
| Predicate composition for a query provider | Expression tree |
| Dynamic LINQ | Expression tree |
| One-off lambda | Delegate |

---

That's Chapter 05. Expression trees close out the delegates story — from "values that execute" to "data that represents code."

→ Continue to: [Questions.md](Questions.md)
