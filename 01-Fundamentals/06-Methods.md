# Methods

## What it is

A **method** is a named, callable unit of behavior. It takes arguments, runs some code, optionally returns a value. Methods are the building blocks above expressions and below classes.

```csharp
public int Square(int x) {
    return x * x;
}

// Call
int n = Square(5);    // 25
```

---

## Anatomy of a method

```csharp
public  static  int   Add(int a, int b)
//  │      │     │      │     │
//  │      │     │      │     └── parameters (with types)
//  │      │     │      └────── method name
//  │      │     └─────────── return type
//  │      └──────────────── modifier (static = doesn't need instance)
//  └─────────────────── access modifier
{
    return a + b;
}
```

Common modifiers:
- **Access**: `public`, `internal`, `protected`, `private`, `protected internal`, `private protected`, `file` (C# 11).
- **Other**: `static`, `virtual`, `override`, `abstract`, `sealed`, `new` (hiding), `async`, `partial`, `extern`, `unsafe`.

For details on modifiers see [Chapter 02 §04 (FieldsAndAccess)](../02-OOP/04-FieldsAndAccess.md) and [Chapter 02 §05 (MethodsAdvanced)](../02-OOP/05-MethodsAdvanced.md). This file focuses on **calling and parameter passing**.

---

## Return values

A method's return type comes before its name:

```csharp
public int Sum(int[] nums) {
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

public void Greet(string name) {
    Console.WriteLine($"Hello, {name}!");
    // No return value
}
```

`void` is "returns nothing". You can `return;` from a void method to exit early; you can't `return 5;` from one.

To return multiple values, use a **tuple**:

```csharp
public (int min, int max) MinMax(int[] arr) {
    int mn = int.MaxValue, mx = int.MinValue;
    foreach (var x in arr) {
        if (x < mn) mn = x;
        if (x > mx) mx = x;
    }
    return (mn, mx);
}

// Call and deconstruct
var (min, max) = MinMax(new[] { 5, 2, 8, 1 });
```

Tuples covered in [Chapter 03 §05](../03-TypeSystem/05-Tuples.md).

---

## Expression-bodied methods

For one-expression methods, the `=>` shorthand:

```csharp
public int Square(int x) => x * x;
public string Greeting(string name) => $"Hello, {name}";
public void Log(string msg) => Console.WriteLine(msg);   // void is fine
```

Equivalent to `{ return x * x; }`. Use when the body is genuinely one expression — don't chain ternaries to fit.

---

## Parameter passing — the four modes

In C#, parameters can be passed:
1. **By value** (the default) — caller's value is *copied* into the parameter.
2. **`ref`** — parameter is an alias for the caller's variable.
3. **`out`** — like `ref`, but caller doesn't need to initialize first; the method *must* assign before returning.
4. **`in`** — like `ref` but read-only; an alias the method can read but not modify.

### By value (default)

```csharp
void Increment(int n) { n++; }

int x = 5;
Increment(x);
Console.WriteLine(x);   // 5 — unchanged
```

For value types, the actual value is copied. For reference types, the *reference* is copied — both caller and method point to the same heap object, so mutations via either are visible to both:

```csharp
void Clear(List<int> list) { list.Clear(); }

var nums = new List<int> { 1, 2, 3 };
Clear(nums);
Console.WriteLine(nums.Count);   // 0 — same object got cleared

void Reassign(List<int> list) { list = new List<int>(); }
var nums2 = new List<int> { 1, 2, 3 };
Reassign(nums2);
Console.WriteLine(nums2.Count);   // 3 — reassign only changed the local
```

This trips up beginners. The variable holds a reference; passing the variable passes the reference; reassigning the parameter changes only the parameter; mutating *through* the parameter modifies the shared object.

### `ref`

```csharp
void Increment(ref int n) { n++; }

int x = 5;
Increment(ref x);
Console.WriteLine(x);   // 6
```

Caller and method both write `ref`. `x` must be initialized first.

Reference types can also be passed `ref` — letting the method reassign the caller's variable:

```csharp
void ReplaceWithEmpty(ref List<int> list) {
    list = new List<int>();
}

var nums = new List<int> { 1, 2, 3 };
ReplaceWithEmpty(ref nums);
Console.WriteLine(nums.Count);   // 0
```

### `out`

Like `ref` but the parameter doesn't need to be initialized before the call — the method **must** assign it before returning.

```csharp
bool TryDivide(int a, int b, out int result) {
    if (b == 0) {
        result = 0;     // must assign even in failure path
        return false;
    }
    result = a / b;
    return true;
}

if (TryDivide(10, 3, out int r))
    Console.WriteLine(r);

// Or inline declaration (C# 7+)
if (int.TryParse("42", out int parsed))
    Console.WriteLine(parsed);

// Or discard if you don't need the value
if (int.TryParse("42", out _))
    Console.WriteLine("valid");
```

The `Try*` pattern uses `out` heavily — when a value might not be produced, return a bool to signal success and an `out` to deliver the value on success.

### `in`

C# 7.2+. Read-only by-reference. Saves the cost of copying a large struct without letting the method mutate it.

```csharp
void Print(in BigStruct s) {
    Console.WriteLine(s.X);
    // s.X = 5;          // ❌ compile error — readonly
}
```

Mostly used for large structs in performance code. Rarely matters for small types.

### `ref readonly` parameters (C# 12)

```csharp
void Process(ref readonly BigStruct s) { ... }
```

Same effect as `in` but the caller **must** spell out `ref readonly` at the call site, making the by-reference intent explicit:

```csharp
Process(ref readonly mainStruct);
```

Useful in APIs where signaling by-reference passing is part of the contract.

---

## `params` arrays

Lets a method accept any number of arguments of one type:

```csharp
public int Sum(params int[] nums) {
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

Sum();              // 0
Sum(1);             // 1
Sum(1, 2, 3, 4);    // 10
Sum(new[] { 1, 2 });  // 3 — pass array directly too
```

Rules:
- Only one `params` parameter per method.
- Must be the **last** parameter.
- Must be an array (or, since C# 13, any collection).

### `params` collections (C# 13+)

C# 13 generalized `params` to any collection-like type:

```csharp
public int Sum(params IEnumerable<int> nums) { ... }
public int Sum(params Span<int> nums) { ... }
public int Sum(params ReadOnlySpan<int> nums) { ... }
public int Sum(params List<int> nums) { ... }
```

### `params ReadOnlySpan<T>` (C# 14)

The new optimization darling:

```csharp
public int Sum(params ReadOnlySpan<int> nums) {
    int total = 0;
    foreach (var n in nums) total += n;
    return total;
}

Sum(1, 2, 3);   // zero allocation — params goes through stack or inline storage
```

Before C# 14, `params int[]` allocated an array per call. With `params ReadOnlySpan<int>` the compiler can pass the args in a stack-allocated span — **zero allocation**. Used heavily in modern BCL: `Console.WriteLine` and `StringBuilder.Append` have overloads taking `params ReadOnlySpan<...>`.

---

## Optional parameters and defaults

```csharp
public void Greet(string name, string greeting = "Hello", bool loud = false) {
    var msg = $"{greeting}, {name}";
    Console.WriteLine(loud ? msg.ToUpper() : msg);
}

Greet("Alice");                        // "Hello, Alice"
Greet("Alice", "Hi");                  // "Hi, Alice"
Greet("Alice", "Hi", true);            // "HI, ALICE"
```

Defaults must be compile-time constants (or `default`, or `new()` for value types in C# 12+).

**Caution**: default values are baked into the **calling assembly**. If a library changes a default, callers compiled against the old version still pass the old default. Don't change defaults across versions.

---

## Named arguments

You can pass arguments by parameter name in any order:

```csharp
Greet(name: "Alice", loud: true, greeting: "Hi");
```

Mixed with positional:

```csharp
Greet("Alice", loud: true);   // positional name, named loud — uses default greeting
```

Rule: once you go named, you generally can't go back to positional. (C# 7.2+ relaxed this for positions that match positionally.)

Named arguments are most useful for **clarity**:

```csharp
DrawRect(x: 10, y: 20, width: 100, height: 50);   // vs DrawRect(10, 20, 100, 50)
```

And for **optional bool flags**:

```csharp
File.WriteAllText("out.txt", text, append: true);   // way better than WriteAllText("...", text, true)
```

---

## Overloading

Multiple methods with the same name, distinguished by parameter types:

```csharp
public int Add(int a, int b) => a + b;
public double Add(double a, double b) => a + b;
public string Add(string a, string b) => a + b;
```

Resolution happens at compile time based on argument types. The compiler picks the **best match**:

```csharp
Add(1, 2);          // calls Add(int, int)
Add(1.5, 2.5);      // calls Add(double, double)
Add("a", "b");      // calls Add(string, string)
Add(1, 2.5);        // calls Add(double, double) — int promotes to double
```

Overload resolution can be subtle when generics, implicit conversions, or `params` are involved. Keep overload sets simple to avoid surprising your future self.

Methods **cannot** overload on return type alone, `ref`/`out` alone, or by `params`.

---

## Local functions

Methods defined **inside** another method:

```csharp
public int Compute(int n) {
    int Helper(int x) => x * x + 1;
    int Reduce(int x) => x switch { 0 => 0, _ => Helper(x) };

    return Reduce(n);
}
```

Useful when a piece of logic is needed only inside one method and you want to give it a name without exposing it.

Benefits over private methods:
- Encapsulation — visible only within the enclosing method.
- Captures local variables (a closure) but **without allocating a closure class** when marked `static`:
  ```csharp
  public int Compute(int n) {
      static int Double(int x) => x * 2;   // no capture allowed
      return Double(n);
  }
  ```

[Chapter 05 §06](../05-DelegatesEvents/06-LocalFunctions.md) covers them in depth.

---

## Variable-length call sites

A few patterns C# developers use to build flexible APIs:

```csharp
// params + builder pattern
public class QueryBuilder {
    public QueryBuilder Where(params Expression<Func<T, bool>>[] conditions) { ... }
}

// Optional + named
public void Log(string message, LogLevel level = LogLevel.Info,
                string? category = null, Exception? exception = null) { ... }

// Overloads + extension methods for fluency
public static class StringExtensions {
    public static bool IsValidEmail(this string s) { ... }
}
"x@y.com".IsValidEmail();
```

---

## Return-by-reference (`ref` returns)

C# 7.0+. A method can return a `ref` to a variable owned by a longer-lived object:

```csharp
public ref int FindFirst(int[] arr, int target) {
    for (int i = 0; i < arr.Length; i++)
        if (arr[i] == target) return ref arr[i];
    throw new InvalidOperationException("not found");
}

int[] data = { 1, 2, 3, 4 };
ref int slot = ref FindFirst(data, 3);
slot = 99;
Console.WriteLine(data[2]);   // 99 — modified through the ref
```

Powerful but niche. Use cases: high-performance code accessing array slots, in-place struct mutation.

`ref readonly` returns give a read-only ref:

```csharp
public ref readonly Point Origin => ref _origin;
```

---

## Async methods (preview — full chapter 8)

```csharp
public async Task<string> FetchAsync(string url) {
    using var client = new HttpClient();
    return await client.GetStringAsync(url);
}
```

`async` modifier + `await` inside the body. Return type must be `Task`, `Task<T>`, `ValueTask`, `ValueTask<T>`, `void` (event handlers only), or a custom task-like type.

We'll devote all of Chapter 08 to this. For now: just recognize the shape.

---

## When parameters become problematic

### Too many parameters
```csharp
public void Configure(string host, int port, string username, string password,
    int timeout, bool useSsl, int retries, string? proxyHost, int? proxyPort) { ... }
```

Refactor to a **config object**:

```csharp
public class ConfigOptions {
    public required string Host { get; init; }
    public int Port { get; init; } = 443;
    public string Username { get; init; } = "";
    // ...
}

public void Configure(ConfigOptions opts) { ... }

Configure(new ConfigOptions {
    Host = "example.com",
    Port = 8080,
    Username = "admin",
});
```

Patterns covered in [Chapter 17 §04 (API Design)](../17-BestPractices/04-ApiDesign.md).

### Boolean parameters
```csharp
SaveFile("out.txt", true);   // what does true mean? unclear
SaveFile("out.txt", overwrite: true);   // clear
```

Use named arguments OR refactor to an enum:

```csharp
public enum SaveMode { Append, Overwrite }
SaveFile("out.txt", SaveMode.Overwrite);
```

---

## Common bugs

- **Passing a reference type and expecting copy semantics** — modifications via the parameter affect the caller's object.
- **Forgetting `ref` at the call site** — must spell `ref` or `out` at both call and declaration (compiler enforces).
- **`out` parameter not assigned** — compile error.
- **Method overload ambiguity** — when two overloads are "equally good" you get an error. Add a cast at the call site or rename one method.
- **`params` with a single array argument** — `Sum(new[] {1, 2, 3})` passes the array; `Sum(1, 2, 3)` does too. The first is **one** argument that happens to be an array; the second is three args wrapped automatically. Usually no difference, but it matters with `params object?[]` where you might wrap an array unintentionally.
- **Default values aren't inherited** when overriding. Subclass redeclares the default.

→ Next: [07-Arrays.md](07-Arrays.md)
