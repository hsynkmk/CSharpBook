# Variables

## What it is

A **variable** is a named storage location holding a value of a specific type. In C# you declare a variable, then assign values to it.

```csharp
int age;        // declaration — reserves storage, no value yet
age = 30;       // assignment

int year = 2026;   // declaration + initialization (in one line, the usual form)
```

After this, you can refer to `age` and `year` by name.

---

## Why it exists

This is the most basic building block of any programming language. Variables let you name things, change them, and refer to them later instead of repeating values. Without variables you'd be writing constants over and over.

---

## Syntax

```csharp
<type> <name> = <expression>;
```

Examples:

```csharp
int       count    = 0;
double    price    = 19.99;
string    name     = "Alice";
bool      isActive = true;
char      grade    = 'A';
DateTime  now      = DateTime.Now;
```

Multiple variables of the same type can be declared in one statement (rarely used):

```csharp
int x = 1, y = 2, z = 3;
```

You can also declare without immediately initializing — but using an uninitialized local is a **compile error**:

```csharp
int n;
Console.WriteLine(n);   // ❌ CS0165: Use of unassigned local variable 'n'
```

The compiler enforces **definite assignment** for locals — every code path must initialize the variable before reading it. Fields of classes/structs don't need this — they get a default value automatically.

---

## `var` — implicit typing

When the type is obvious from the right-hand side, you can use `var`:

```csharp
var count = 0;              // inferred as int
var name = "Alice";         // string
var items = new List<int>(); // List<int>
var when = DateTime.Now;     // DateTime
```

`var` is **not** a dynamic type. It's a compile-time inference of the actual type. After compilation, `var count = 0;` is identical to `int count = 0;`.

### When to use `var`

The C# coding conventions and most teams say: **use `var` when the type is obvious from the right-hand side; spell it out when it isn't.**

```csharp
var users = new List<User>();          // ✓ obvious
var first = users.First();             // ✓ obvious — it's a User

var result = ProcessAsync();           // ? — what does that return?
Task<UserResult> result = ProcessAsync();  // ✓ clearer
```

Some teams disagree and use `var` everywhere or never. Pick a style, be consistent. JetBrains Rider and modern Visual Studio will tell you the inferred type on hover.

### Where you CAN'T use `var`

- For declaring **fields** (members of a class): no, fields need an explicit type.
- For declaring **method parameters**: no.
- When there's **no initializer**: no — there's nothing to infer from.
- For **null literals** without a context: `var x = null;` is a compile error because `null` has no type. Use `string? x = null;` instead.

```csharp
public class Foo {
    var x = 0;          // ❌ — can't use var for a field
    int y = 0;          // ✓
}
```

---

## `const` — compile-time constants

A `const` is a value baked into the assembly at compile time:

```csharp
const int MaxRetries = 3;
const string DefaultGreeting = "Hello";
const double Pi = 3.14159265358979;
```

Rules:
- The value must be a compile-time constant (literal or simple constant expression).
- Constants are implicitly `static` — they live on the type, not on instances.
- Can be `int`, `string`, `bool`, `char`, `double`, etc. — **only types that can have literal values**.
- Cannot be a custom class.
- **The value is inlined** at every use site — changing a public `const` in a library forces every consumer to recompile.

For values that shouldn't change but aren't compile-time constant, use `static readonly`:

```csharp
public static readonly DateTime AppStartedAt = DateTime.UtcNow;
public static readonly TimeSpan DefaultTimeout = TimeSpan.FromSeconds(30);
```

`static readonly` is set once (in the static constructor or at field init), then immutable. Unlike `const`, the value lives in the assembly's data, so callers don't need recompiling when it changes.

[Chapter 02 §04](../02-OOP/04-FieldsAndAccess.md) covers fields, readonly, and `const` vs `static readonly` in depth.

---

## Scope

A variable's **scope** is the region of code where its name is visible.

### Block scope

Every `{ }` defines a scope. Variables declared inside a block are visible only within it.

```csharp
{
    int temp = 10;
    Console.WriteLine(temp);   // ✓
}
Console.WriteLine(temp);       // ❌ — out of scope
```

### Method scope

A variable declared inside a method is local to that method:

```csharp
void DoWork() {
    int x = 5;
    // x is visible here
}
// x not visible here
```

### Loop scope

The variable declared in a `for`, `while`, or `foreach` is scoped to the loop:

```csharp
for (int i = 0; i < 10; i++) {
    // i is visible
}
// i NOT visible here

foreach (var item in items) {
    // item is visible
}
// item NOT visible here
```

### `if` scope

Variables declared in an `if` body are scoped to that body:

```csharp
if (cond) {
    int n = 5;
    // n visible here
}
// n NOT visible here
```

### Pattern-introduced variables

`is` patterns and `out var` introduce variables into the enclosing scope:

```csharp
if (obj is string s) {
    Console.WriteLine(s.Length);   // s visible here
}
// s technically still in scope in newer C# versions in some forms,
// but only definitely assigned where the pattern matched

if (int.TryParse(input, out int n)) {
    Console.WriteLine(n);    // n visible
}
// n is also visible here but not definitely assigned
```

---

## Naming conventions

C# uses **PascalCase** for public things and **camelCase** for locals/private fields:

| Kind | Convention | Example |
|---|---|---|
| Local variable | camelCase | `int itemCount` |
| Method parameter | camelCase | `void Set(int newValue)` |
| Private field | camelCase with `_` prefix | `private int _retries` |
| Public field | PascalCase (avoid public fields) | `public int Count` |
| Property | PascalCase | `public int Count { get; }` |
| Method | PascalCase | `void DoSomething()` |
| Class / struct / record / enum | PascalCase | `class HttpClient` |
| Interface | PascalCase, `I` prefix | `interface IDisposable` |
| Constant | PascalCase (Microsoft style) | `const int MaxRetries = 3` |
| Type parameter | `T` prefix | `class List<TElement>` |

Underscores in names are unusual outside private fields. ALL_CAPS for constants is **not** C# convention — that's a C/Java carryover.

[Chapter 17 §01](../17-BestPractices/01-NamingConventions.md) has the full set of conventions with rationale.

---

## Default values

If you declare a **field** without initializing it, it gets a default value:

| Type | Default |
|---|---|
| `int`, `long`, `short`, `byte`, `uint`, etc. | `0` |
| `float`, `double`, `decimal` | `0.0` |
| `bool` | `false` |
| `char` | `'\0'` (null char) |
| `enum` | `0` cast to the enum |
| Any value type (struct) | all fields zeroed |
| Any reference type (class, string, interface, delegate) | `null` |

For **locals** the compiler forces you to initialize before use, so defaults don't matter — but you can ask for one explicitly:

```csharp
int x = default;            // 0
string? s = default;        // null
DateTime when = default;    // 0001-01-01 00:00:00
```

`default` is a keyword that produces the default value of the target type, inferred from context. `default(int)` and `default(string)` are explicit forms.

---

## Reassignment vs immutability

C# variables are **mutable** by default — you can change them as often as you like:

```csharp
int n = 5;
n = 10;
n = n + 1;
```

To make a local unchangeable after init, there's no keyword like `let` (Swift) or `val` (Kotlin) — but you can use `const` (compile-time) or design your types to be immutable. We'll get to immutability patterns in [Chapter 17 §07](../17-BestPractices/07-ImmutabilityPatterns.md).

For records (C# 9+), positional properties are init-only by default — see [Chapter 03 §03](../03-TypeSystem/03-Records.md).

---

## A subtle thing — boxed locals in closures

When a lambda captures a local variable, the compiler **hoists** that variable into a generated class. This makes captures behave a bit differently from raw values:

```csharp
int x = 5;
Action a = () => Console.WriteLine(x);  // x is captured

x = 10;
a();    // prints 10 — the closure sees the latest value
```

If you have a loop and want each lambda to capture its iteration's value, declare a **new local** inside the loop:

```csharp
var funcs = new List<Func<int>>();
for (int i = 0; i < 3; i++) {
    int copy = i;   // new variable per iteration
    funcs.Add(() => copy);
}
funcs.ForEach(f => Console.WriteLine(f()));   // 0, 1, 2

// vs without the copy:
for (int i = 0; i < 3; i++)
    funcs.Add(() => i);
funcs.ForEach(f => Console.WriteLine(f()));   // 3, 3, 3 (all see final i)
```

`foreach` (since C# 5) already gives each iteration a fresh variable — only classic `for` loops have this issue. We'll explore this thoroughly in [Chapter 05 §04 — Closures](../05-DelegatesEvents/04-Closures.md).

---

## Compile-time errors you'll hit

- **CS0165** — Use of unassigned local variable.
- **CS0103** — The name 'x' does not exist in the current context.
- **CS0136** — A local already exists with that name (shadowing).
- **CS0029** — Cannot implicitly convert type 'X' to 'Y'.
- **CS0815** — Cannot assign void to an implicitly-typed variable.
- **CS0822** — `const` must be a value type, string, or enum.

Don't memorize the numbers — but recognize that errors with `CS####` come from the C# compiler itself (the language) rather than from analyzers (rules a tool layers on top).

---

## Performance note

A local variable lives on the **stack** in most cases. Stack allocation is essentially free — it's a pointer bump on entry to a method.

Exceptions:
- **Captured locals** (in closures): get hoisted to a heap-allocated class — they cost an allocation.
- **iterator/async locals**: when a method has `yield return` or `await`, the compiler hoists locals into a state-machine struct that may end up on the heap (boxing to interface) or — in .NET 10 — may stay on the stack via escape analysis.
- **Reference types**: the variable itself is a pointer on the stack; the object it points to is on the heap.

See [Chapter 09 §01](../09-MemoryPerformance/01-StackVsHeap.md) for the deep version.

---

## When to use what

| Goal | Use |
|---|---|
| Local value you'll change | `int n = 0; ...` |
| Local where type is obvious | `var users = new List<User>();` |
| Compile-time constant | `const int MaxRetries = 3;` |
| Runtime "set once" value | `static readonly` field |
| Field that's part of a class | declare as a field, not a local |

→ Next: [02-PrimitiveTypes.md](02-PrimitiveTypes.md)
