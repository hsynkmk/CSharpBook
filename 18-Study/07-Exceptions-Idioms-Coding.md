# 07 — Exceptions & Idioms — Coding Questions

> Predict the output / find the bug. (Concepts: [07-Exceptions-Idioms.md](07-Exceptions-Idioms.md))

---

### Q1 — throw vs throw ex
```csharp
void Outer() { try { Inner(); } catch (Exception ex) { throw ex; } }   // vs: throw;
void Inner() => throw new InvalidOperationException("boom");
```
<details><summary>Answer</summary>

`throw ex;` **resets the stack trace** to the `Outer` catch line — you lose where it originated (`Inner`). Use **`throw;`** to rethrow preserving the original stack. Classic bug.
</details>

---

### Q2 — What's the output?
```csharp
static int M() {
    try { return 1; }
    finally { Console.WriteLine("finally"); }
}
Console.WriteLine(M());
```
<details><summary>Answer</summary>

```
finally
1
```
`finally` runs **before** the method actually returns (after the return value `1` is computed). `finally` always executes.
</details>

---

### Q3 — finally overrides return
```csharp
static int M() {
    try { return 1; }
    finally { return 2; }    // ?
}
Console.WriteLine(M());
```
<details><summary>Answer</summary>

**`2`** (with a compiler warning). A `return` in `finally` **overrides** the try's return (and would swallow exceptions too). Never `return`/`throw` from `finally`.
</details>

---

### Q4 — async void exception
```csharp
async void FireAndForget() => throw new Exception("boom");
try { FireAndForget(); }
catch (Exception) { Console.WriteLine("caught"); }
Console.WriteLine("after");
```
<details><summary>Answer</summary>

**"caught" is NOT printed** — the exception from an `async void` method can't be caught by the caller (it's posted to the sync context and typically **crashes the process**). "after" prints, then the app may crash. Use `async Task` and `await`.
</details>

---

### Q5 — Exception filter evaluation order
```csharp
try { throw new HttpRequestException("x"); }
catch (Exception) when (Log("filter") && false) { Console.WriteLine("A"); }
catch (Exception) { Console.WriteLine("B"); }
static bool Log(string s){ Console.WriteLine(s); return true; }
```
<details><summary>Answer</summary>

```
filter
B
```
The `when` **filter runs first** (prints "filter"), evaluates to `false` (so block A is skipped), then the next matching catch runs → "B". Filters run before the stack unwinds — useful for logging without catching.
</details>

---

### Q6 — Find the anti-pattern
```csharp
try { DoWork(); }
catch (Exception) { }    // ?
```
<details><summary>Answer</summary>

**Swallowing all exceptions** — hides bugs, leaves state corrupted, makes failures invisible. Catch only what you can handle (specific types) and either handle meaningfully or let it propagate. Empty `catch` is almost always wrong.
</details>

---

### Q7 — Try-pattern vs exception
```csharp
// Parsing user input that's often invalid:
int Bad(string s) { try { return int.Parse(s); } catch { return 0; } }
int Good(string s) => int.TryParse(s, out var n) ? n : 0;
```
<details><summary>Answer</summary>

`Bad` throws/catches on every invalid input — **expensive** (stack capture) and obscures intent. `Good` uses the **`Try` pattern** — no exception for the expected failure. Reserve exceptions for *exceptional* cases.
</details>

---

### Q8 — WhenAll exceptions
```csharp
var t1 = Task.FromException(new InvalidOperationException("A"));
var t2 = Task.FromException(new ArgumentException("B"));
try { await Task.WhenAll(t1, t2); }
catch (Exception ex) { Console.WriteLine(ex.Message); }
```
<details><summary>Answer</summary>

Prints **one** message (e.g., `A`) — `await Task.WhenAll(...)` surfaces only the **first** exception. To see all, inspect the task: `Task.WhenAll(...).Exception?.InnerExceptions`. ([09-Threading-and-Tasks.md](09-Threading-and-Tasks.md))
</details>

---

### Q9 — Guard clause modern form
```csharp
void Process(string input) {
    // pre-C#10 verbose version → modernize:
    if (input == null) throw new ArgumentNullException(nameof(input));
}
```
<details><summary>Answer</summary>

Modern: **`ArgumentNullException.ThrowIfNull(input);`** (and `ArgumentException.ThrowIfNullOrEmpty`, `ObjectDisposedException.ThrowIf`). Concise, fail-fast guard clauses at the method boundary.
</details>

---

### Q10 — Preserve the root cause
```csharp
try { CallDb(); }
catch (SqlException ex) { throw new DataAccessException("DB failed"); }   // ?
```
<details><summary>Answer</summary>

This **loses the original `SqlException`** (no inner exception) — you can't diagnose the root cause. **Fix:** `throw new DataAccessException("DB failed", ex);` — pass the original as the inner exception.
</details>
