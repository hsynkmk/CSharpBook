# Exceptions — Basics

## What it is

An **exception** is an object that signals "something went wrong." Code throws an exception, the runtime unwinds the call stack searching for a handler, and either the handler catches it or the program crashes.

```csharp
try {
    var content = File.ReadAllText("missing.txt");
}
catch (FileNotFoundException ex) {
    Console.WriteLine($"File not found: {ex.FileName}");
}
catch (IOException ex) {
    Console.WriteLine($"I/O failure: {ex.Message}");
}
```

Exceptions are the modern alternative to error-code-returning. They:
- **Cannot be silently ignored** — the program crashes if uncaught.
- **Carry rich context** — message, stack trace, optionally inner exceptions.
- **Bypass intermediate frames** — the handler can be far up the stack.

But they're **costly**: throwing and catching an exception is hundreds of times slower than returning a value. So: use them for exceptional cases, not control flow.

---

## try / catch / finally

```csharp
try {
    // code that might throw
}
catch (SpecificException ex) {
    // handle one type
}
catch (Exception ex) {
    // catch-all
    throw;   // re-throw, preserving stack trace
}
finally {
    // always runs (after try and matching catch), whether or not an exception was thrown
}
```

### Rules

- At least one of `catch` or `finally` must be present.
- Catch clauses are tried **top to bottom**. Put **more specific exception types first**.
- `Exception` is the root — catching it catches everything (almost — see below).
- A `finally` block runs **always**, including when control leaves via `return`, `break`, `continue`, or `throw`.

### `catch` without a type

```csharp
try { ... }
catch {
    // catches everything — including non-CLR exceptions from interop
    throw;
}
```

Rare. Pre-.NET 2.0 it was needed for some interop scenarios; now `catch (Exception)` does the job in 99% of cases.

### `catch (Exception)` is usually wrong

If you catch the base `Exception` class and don't re-throw, you've swallowed every bug — `NullReferenceException`, `OutOfMemoryException`, even `StackOverflowException` (well, that one's special). You almost never want this.

```csharp
// 🚨 anti-pattern
try {
    Process();
}
catch (Exception) {
    // suppressed — what happened?
}
```

Instead:
- Catch **specific** exceptions you know how to handle.
- Let unexpected ones bubble up and crash the process (or be caught at a top-level handler that logs).

---

## Throwing

```csharp
throw new ArgumentException("Invalid input", nameof(input));
throw new InvalidOperationException("State machine in bad state");
throw new NotImplementedException();
throw new NotSupportedException("This operation requires elevated privileges");
```

You can also `throw` an already-existing exception (re-raising):

```csharp
try { ... }
catch (Exception ex) {
    Log(ex);
    throw;       // re-throws preserving original stack trace
    throw ex;    // ❌ same exception but RESETS the stack trace — almost always wrong
}
```

**Use bare `throw;` to re-raise.** Using `throw ex;` rewrites the StackTrace property and you lose where the exception was originally raised — a debugging nightmare.

### `throw` expressions (C# 7+)

`throw` can appear in expression contexts:

```csharp
public string Name {
    get => _name ?? throw new InvalidOperationException("Name not set");
    set => _name = value ?? throw new ArgumentNullException(nameof(value));
}

return condition ? doIt() : throw new ArgumentException("bad");
```

---

## The `Exception` class hierarchy

```
Exception (System)
├── SystemException
│   ├── ArgumentException
│   │   ├── ArgumentNullException
│   │   ├── ArgumentOutOfRangeException
│   ├── InvalidOperationException
│   │   ├── ObjectDisposedException
│   ├── NotImplementedException
│   ├── NotSupportedException
│   ├── ArithmeticException
│   │   ├── DivideByZeroException
│   │   ├── OverflowException
│   ├── IOException
│   │   ├── FileNotFoundException
│   │   ├── DirectoryNotFoundException
│   │   ├── PathTooLongException
│   ├── NullReferenceException
│   ├── IndexOutOfRangeException
│   ├── TypeLoadException
│   ├── FormatException
│   ├── OutOfMemoryException
│   ├── StackOverflowException     ← can't be caught reliably
│   ├── ThreadAbortException        ← deprecated since .NET Framework
├── ApplicationException             ← deprecated convention; just inherit Exception
├── AggregateException               ← used by tasks, parallel
├── TaskCanceledException            ← thrown on cancellation
├── OperationCanceledException
```

Some you'll throw or see often:
- **`ArgumentException`** — bad argument value.
- **`ArgumentNullException`** — argument was unexpectedly null.
- **`ArgumentOutOfRangeException`** — argument outside expected range.
- **`InvalidOperationException`** — object state doesn't permit the operation (e.g., calling `MoveNext` after iterating done).
- **`ObjectDisposedException`** — caller used a disposed object.
- **`NotSupportedException`** — this method isn't available for this type.
- **`NotImplementedException`** — stub for "I'll write this later."
- **`FormatException`** — string couldn't be parsed.
- **`IOException`** — file system / network / device error.
- **`OperationCanceledException`** — token-based cancellation fired.

---

## Exception filters — `when` (C# 6+)

A `when` clause lets you decide whether to catch based on a condition:

```csharp
try {
    DoNetwork();
}
catch (HttpRequestException ex) when (ex.StatusCode == HttpStatusCode.NotFound) {
    return null;
}
catch (HttpRequestException ex) when (ex.StatusCode >= HttpStatusCode.InternalServerError) {
    return Retry();
}
catch (HttpRequestException) {
    throw;
}
```

Two advantages over `if` inside the catch body:
1. **The stack is not unwound** if the filter returns false — better debugging (the original throw point stays in scope).
2. The exception **isn't logged** as "caught and rethrown" if you don't enter the body.

Use filters for "catch only certain HTTP status codes," "retry only if it's a network blip," etc.

---

## `using` and `finally` — automatic cleanup

`try`/`finally` is fine for cleanup, but for objects implementing `IDisposable`, the `using` statement is cleaner:

```csharp
// Equivalent forms
using (var stream = File.OpenRead("f.txt")) {
    // use stream
}
// Stream.Dispose() called automatically at end of block

// Compiles to:
{
    var stream = File.OpenRead("f.txt");
    try {
        // use stream
    }
    finally {
        if (stream != null) stream.Dispose();
    }
}
```

Modern declaration form (C# 8+):

```csharp
using var stream = File.OpenRead("f.txt");
// Dispose called at end of enclosing block
```

[Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md) covers IDisposable in depth.

---

## Custom exceptions

```csharp
public class DomainException : Exception {
    public DomainException() {}
    public DomainException(string message) : base(message) {}
    public DomainException(string message, Exception inner) : base(message, inner) {}
}

public class UserNotFoundException : DomainException {
    public int UserId { get; }
    public UserNotFoundException(int userId)
        : base($"User {userId} not found") {
        UserId = userId;
    }
}
```

When designing your own:
- **Inherit from `Exception`** (not `ApplicationException` — Microsoft's official recommendation since 2005 is to ignore that class).
- Provide the three standard constructors (parameterless, message, message + inner).
- Add domain-specific properties when context is useful.
- Class name ends with `Exception` by convention.
- Make them `[Serializable]` only if you need cross-process / remoting; .NET Core no longer requires it.

---

## Inner exceptions and exception wrapping

When you catch an exception and want to rethrow with more context, wrap it:

```csharp
try {
    LoadConfig("settings.json");
}
catch (IOException ex) {
    throw new InvalidOperationException("Failed to load configuration", ex);
}
```

The `inner` exception is preserved — callers can walk the chain via `.InnerException`:

```csharp
try { ... }
catch (Exception ex) {
    var root = ex;
    while (root.InnerException != null) root = root.InnerException;
    Console.WriteLine($"Root cause: {root.Message}");
}
```

For aggregate exceptions (`Task.WhenAll`), use `AggregateException.Flatten()` and iterate `InnerExceptions`.

---

## When to throw vs return false

Two schools of thought:

**Throw** when:
- The error is **unexpected** — probably a bug or environment failure.
- Recovery requires action far up the stack.
- The method has no natural "failure value."

**Return false / null / Result type** when:
- Failure is **expected** in normal flow.
- The caller will frequently check.
- The error doesn't need a stack trace.

`int.TryParse` returns false because parsing user input often fails; `int.Parse` throws because if you're sure it's an int, failure is a bug.

For complex flows, **Result types** (a `(bool ok, T value, string error)` style record) work well — see [Chapter 17 §08 (DefensiveProgramming)](../17-BestPractices/08-DefensiveProgramming.md).

---

## The new throw-helper APIs (.NET 6+)

Modern BCL provides static helpers that throw conditionally:

```csharp
ArgumentNullException.ThrowIfNull(arg);                  // .NET 6+
ArgumentNullException.ThrowIfNull(arg, nameof(arg));     // explicit name (rarely needed thanks to CallerArgumentExpression)
ArgumentException.ThrowIfNullOrEmpty(str);               // .NET 7+
ArgumentException.ThrowIfNullOrWhiteSpace(str);          // .NET 8+
ArgumentOutOfRangeException.ThrowIfNegative(n);          // .NET 8+
ArgumentOutOfRangeException.ThrowIfGreaterThan(n, 100);  // .NET 8+
ArgumentOutOfRangeException.ThrowIfLessThan(n, 0);
ArgumentOutOfRangeException.ThrowIfZero(n);
ObjectDisposedException.ThrowIf(_disposed, this);        // .NET 7+
```

These replace boilerplate like:
```csharp
if (arg == null) throw new ArgumentNullException(nameof(arg));
if (string.IsNullOrEmpty(str)) throw new ArgumentException("...");
```

Adopt them. The argument name is captured automatically via `[CallerArgumentExpression]`.

---

## Stack traces

When an exception is thrown, `ex.StackTrace` captures where:

```
   at MyApp.UserService.Load(Int32 id) in /src/UserService.cs:line 42
   at MyApp.Program.Main(String[] args) in /src/Program.cs:line 12
```

PDB files (debug symbols) must be deployed alongside DLLs for line numbers to appear. They usually are by default in Release builds since .NET Core 3.0.

For richer traces, `Exception.Demystify()` (from the `Ben.Demystifier` package) cleans up async stack traces:

```csharp
catch (Exception ex) {
    Console.WriteLine(ex.Demystify());
}
```

Useful but a non-stdlib dependency. Built-in `.ToString()` is usually good enough.

---

## Performance of exceptions

Throwing an exception is **expensive** — generally 1-10 microseconds vs nanoseconds for a regular function return. The cost comes from:
1. Allocating the exception object.
2. Capturing the stack trace.
3. Searching for a handler (unwinding).

**Therefore**: don't use exceptions for normal control flow. Specifically:
- Don't throw in loops to break out early.
- Don't throw to signal an empty collection.
- Use `TryParse` / `TryGetValue` patterns instead of `try { Parse... } catch`.

But: **don't worry about exception cost in genuinely exceptional paths** — a 1ms cost per "file not found" is fine.

---

## Common bugs

- **`throw ex;` instead of `throw;`**: rewrites the stack. Always use `throw;` to rethrow.
- **Catching `Exception` and swallowing**: hides bugs. Catch specific types or re-throw.
- **`catch (Exception ex)` that logs `ex.Message` only**: loses the stack trace. Log `ex.ToString()`.
- **Disposing in `catch` instead of `finally`**: skips cleanup if no exception thrown. Use `using`.
- **Throwing in finalizers**: never. The process crashes (in .NET Framework) or the exception is swallowed (in modern .NET).
- **Long error-handling pyramids**: refactor with `using` + early returns + ThrowIf helpers.

---

## Beyond basics

This file covered the mechanics. For more:

- **`IDisposable` and resource cleanup** — [Chapter 09 §03](../09-MemoryPerformance/03-IDisposable.md).
- **Async exception handling** — [Chapter 08 §02](../08-Concurrency/02-AsyncAwaitFundamentals.md) and [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md).
- **Exception design guidelines** — [Chapter 17 §04 (API Design)](../17-BestPractices/04-ApiDesign.md) and [Chapter 17 §08 (Defensive Programming)](../17-BestPractices/08-DefensiveProgramming.md).

→ Next: [09-CommentsAndXmlDocs.md](09-CommentsAndXmlDocs.md)
