# Control Flow

## What it is

The constructs that decide **which code runs**: `if`, `switch`, loops, and unconditional jumps. C# inherits most of this from C/C++, with modern additions: switch expressions (C# 8), pattern matching everywhere (C# 7+), and various small improvements.

---

## `if` / `else if` / `else`

```csharp
if (condition) {
    // ...
} else if (otherCondition) {
    // ...
} else {
    // ...
}
```

Braces are technically optional for single-statement bodies:

```csharp
if (n > 0) Console.WriteLine("positive");
else Console.WriteLine("non-positive");
```

**Always use braces.** They prevent the famous "goto fail" class of bugs:

```csharp
// 🚨 dangerous — looks safe, isn't
if (cond)
    DoOne();
    DoTwo();        // always runs, regardless of cond — indentation is misleading

// ✓ safe
if (cond) {
    DoOne();
    DoTwo();
}
```

C# evaluates the condition; if it's a `bool` (`true` or `false`), behavior is straightforward. There's **no implicit int-to-bool conversion** — `if (5)` is a compile error.

---

## `switch` statement

The traditional `switch`. Multiple cases jump to the same code; each case must end with `break`, `return`, `goto`, or `throw`.

```csharp
int day = 3;
switch (day) {
    case 1:
        Console.WriteLine("Monday");
        break;
    case 2:
    case 3:
        Console.WriteLine("Mid-week start");
        break;
    case 7:
        Console.WriteLine("Sunday");
        break;
    default:
        Console.WriteLine("Other");
        break;
}
```

Notable rules:
- No implicit fallthrough between cases (unlike C). Each case requires an explicit terminator.
- `goto case 7;` and `goto default;` are legal if you really want fallthrough.
- The expression and each case label must be a compatible type.
- Cases must be **constants**.

### Pattern matching in switch (C# 7+)

The switch statement was upgraded to handle patterns:

```csharp
object o = GetSomething();
switch (o) {
    case int n when n < 0:
        Console.WriteLine($"negative int {n}");
        break;
    case int n:
        Console.WriteLine($"non-negative int {n}");
        break;
    case string s:
        Console.WriteLine($"string of length {s.Length}");
        break;
    case null:
        Console.WriteLine("null");
        break;
    default:
        Console.WriteLine("something else");
        break;
}
```

`when` clauses add guards to a pattern. Cases are tried in order.

---

## `switch` expressions (C# 8+)

A **statement** does something; an **expression** produces a value. Switch expressions return values:

```csharp
string Category(int n) => n switch {
    < 0 => "negative",
    0 => "zero",
    > 0 and < 100 => "small positive",
    >= 100 => "large positive"
};
```

Syntax notes:
- The pattern comes **before** `=>`, the result comes after.
- Cases separated by commas; the whole thing ends with `;`.
- The pattern `_` (discard) is the "default" case.
- The compiler verifies **exhaustiveness** — if some input is missed, you get a warning.

### Pattern types in switch expressions

```csharp
string Describe(object o) => o switch {
    null => "null",
    int n => $"int {n}",                              // type pattern
    string { Length: 0 } => "empty string",           // property pattern
    string s => $"string '{s}'",
    Point(0, 0) => "origin",                          // positional pattern (needs Deconstruct)
    Point { X: var x, Y: var y } => $"point ({x},{y})",
    int[] { Length: > 100 } => "huge int array",
    int[] [1, 2, ..] => "int array starting 1, 2",   // list pattern (C# 11+)
    not null => "something",                          // negated
    _ => "unknown"
};
```

Patterns covered in depth in [Chapter 03 §09](../03-TypeSystem/09-PatternMatching.md).

### Switch expression vs statement

| Switch expression | Switch statement |
|---|---|
| Returns a value | Performs a side effect |
| Expression syntax (`=>`) | Statement syntax (`case x:` ... `break;`) |
| Exhaustiveness check | No exhaustiveness check |
| Cases evaluated top-down, returns first match | Cases jump-table-style, `break` required |

Prefer **switch expression** when picking a value; **switch statement** when you need to execute statements (especially with side effects, fallthrough, or `break` semantics).

---

## Loops

### `for`

```csharp
for (int i = 0; i < 10; i++) {
    Console.WriteLine(i);
}
```

Three parts:
1. **Initializer** — runs once before the loop.
2. **Condition** — checked before each iteration.
3. **Iterator** — runs after each iteration.

Any part can be empty:

```csharp
for (;;) { /* infinite loop */ }       // same as while(true)
for (int i = 0; ; i++) { /* manual exit */ }
```

Multiple iterators:

```csharp
for (int i = 0, j = 10; i < j; i++, j--) { ... }
```

### `while`

```csharp
while (condition) {
    // ...
}
```

Tests at the top — body might run zero times.

### `do-while`

```csharp
do {
    // ...
} while (condition);
```

Tests at the bottom — body runs at least once.

### `foreach`

The most common loop. Iterates anything that implements `IEnumerable<T>` (or `IEnumerable`, or has a `GetEnumerator()` method returning a `MoveNext()`/`Current` enumerator — the "duck typing" rule).

```csharp
var names = new[] { "Alice", "Bob", "Carol" };
foreach (var name in names) {
    Console.WriteLine(name);
}
```

What `foreach` does under the hood:
```csharp
// foreach (var x in xs) { body }
// becomes (approximately):
using (var e = xs.GetEnumerator()) {
    while (e.MoveNext()) {
        var x = e.Current;
        // body
    }
}
```

(The `using` only happens if the enumerator implements `IDisposable`.)

`foreach` also supports:
- **Deconstruction** in the loop variable (C# 7+): `foreach (var (k, v) in dict) { ... }`.
- **`ref` variable** for collections that support it (C# 7.3+): `foreach (ref var x in span) { x = 0; }` modifies in place.
- **`await foreach`** for `IAsyncEnumerable<T>` (C# 8+) — see [Chapter 08 §07](../08-Concurrency/07-AsyncStreams.md).

### Pitfall: modifying a collection while iterating

```csharp
var list = new List<int> { 1, 2, 3, 4, 5 };
foreach (var x in list) {
    if (x % 2 == 0) list.Remove(x);   // ❌ InvalidOperationException
}
```

The enumerator detects collection modification and throws. Build a separate list of items to remove, or use a `for` loop iterating backwards, or use LINQ:

```csharp
list = list.Where(x => x % 2 != 0).ToList();
```

---

## `break` and `continue`

- **`break`** — exit the **innermost** loop or switch.
- **`continue`** — skip to the next iteration of the innermost loop.

```csharp
for (int i = 0; i < 10; i++) {
    if (i == 5) break;       // exit when i == 5
    if (i % 2 == 0) continue;  // skip even numbers
    Console.WriteLine(i);       // prints 1, 3
}
```

No "labeled break" like Java. To exit nested loops, use a flag, refactor to a method (then `return`), or use `goto`:

```csharp
for (int i = 0; i < 10; i++) {
    for (int j = 0; j < 10; j++) {
        if (Found(i, j)) goto done;
    }
}
done:
Console.WriteLine("found it");
```

`goto` works but is rarely the cleanest answer. Refactoring to a method usually beats it.

---

## `return`

Exits the current method. With a value if the method returns one:

```csharp
public int Find(int[] arr, int target) {
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] == target) return i;
    return -1;
}
```

For `void` methods, `return;` is an early exit:

```csharp
public void Process(string? s) {
    if (s == null) return;
    Console.WriteLine(s.Length);
}
```

For `async` methods returning `Task`, `return;` (or just falling off the end) completes the task.

---

## `goto`

The much-maligned `goto` exists in C#. Don't use it casually, but it has legitimate uses:

```csharp
// Break out of nested loops (acceptable)
foreach (var row in matrix)
    foreach (var cell in row)
        if (cell.IsTarget) goto found;
return null;
found:
// ...

// Inside a switch — explicit fallthrough
switch (level) {
    case "verbose":
        Console.WriteLine("verbose details");
        goto case "info";
    case "info":
        Console.WriteLine("info");
        break;
}
```

For everything else, refactor.

---

## `throw` (as a control-flow construct)

Throws an exception, unwinding the stack until it's caught:

```csharp
if (n < 0) throw new ArgumentException("must be non-negative", nameof(n));
```

You can also use `throw` as an **expression** (C# 7+):

```csharp
string name = input ?? throw new ArgumentNullException(nameof(input));
return condition ? doIt() : throw new InvalidOperationException();
```

Exceptions covered in [§08](08-ExceptionsBasics.md).

---

## Conditional (ternary) operator

Already covered in [§04](04-Operators.md), but it's worth restating: it's an expression that picks one of two values based on a condition.

```csharp
int abs = n >= 0 ? n : -n;
```

Don't nest more than once or twice — switch expressions are clearer.

---

## Early return is better than deep nesting

```csharp
// 🥲 pyramid of doom
public string Process(User? user) {
    if (user != null) {
        if (user.IsActive) {
            if (user.HasPermission) {
                return DoWork(user);
            } else {
                return "no permission";
            }
        } else {
            return "inactive";
        }
    } else {
        return "no user";
    }
}

// ✓ flat with guards
public string Process(User? user) {
    if (user is null) return "no user";
    if (!user.IsActive) return "inactive";
    if (!user.HasPermission) return "no permission";
    return DoWork(user);
}
```

Guards (early returns) make the happy path the main flow and put failures up front.

---

## `if` vs switch expression vs ternary — when to use which

| Situation | Use |
|---|---|
| One condition, statements in body | `if/else` |
| Pick one of two values | ternary `?:` |
| Pick one of many values | `switch` expression |
| Match by type or pattern | `switch` expression (or `is` patterns in if) |
| Side effects per branch | `switch` statement |
| Want exhaustiveness checking | `switch` expression |

---

## `using` statements and declarations

Briefly: `using` for `IDisposable` resources. Two forms:

```csharp
// Block form — older
using (var stream = File.OpenRead("data.txt")) {
    // use stream
}
// Dispose called here

// Declaration form (C# 8+) — preferred when scope is the whole enclosing block
using var stream = File.OpenRead("data.txt");
// use stream
// Dispose called when the enclosing block exits
```

[Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md) covers this and the full Dispose pattern.

`using` is technically a control-flow construct because it inserts try/finally around the body.

---

## Performance and predictability

- The JIT inlines small methods and often optimizes simple loops aggressively.
- `for (int i = 0; i < arr.Length; i++)` is the canonical fast loop on arrays — the JIT eliminates bounds checks.
- `foreach` on arrays is also optimized — same speed as `for`.
- `foreach` on `List<T>` is a hair slower than indexed `for` due to enumerator overhead — usually not worth worrying about.
- `foreach` on `IEnumerable<T>` is the slowest — the enumerator can't be specialized.
- `switch` on small integer ranges may become a jump table; on strings it usually becomes a hash table.

---

## Common bugs

- **Forgetting `break`** in a switch statement — compiler error (C# enforces it).
- **Modifying a collection inside `foreach`** — throws `InvalidOperationException`.
- **Off-by-one** in `for` — classic. `for (i = 0; i <= arr.Length; i++)` accesses `arr[arr.Length]` which throws.
- **`while (s = ReadLine())`** — invalid; assignment isn't a bool. Use `while ((s = ReadLine()) != null)`.
- **Infinite loops** — `while (true)` is fine if you have an exit; missing exit → hang.
- **`continue` inside `using` block** — fine. But know that `Dispose` will be called when the block ends.
- **Switch with floating-point `==`** — generally a bad idea; floats rarely equal exactly. Use ranges.

→ Next: [06-Methods.md](06-Methods.md)
