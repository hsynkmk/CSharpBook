# Nullable Reference Types (NRT)

## What it is

C# 8 (2019) introduced **Nullable Reference Types** — a compile-time system that tracks whether reference variables might be null. Annotated types (`string?`) tell the compiler "this could be null." Non-annotated types (`string`) assert "this should never be null." The compiler emits warnings when you might misuse a nullable value.

```csharp
#nullable enable

string s = "hello";        // non-null
string? maybe = null;       // explicitly nullable

s = null;                   // ⚠ warning: assigning null to non-nullable
maybe.Length;               // ⚠ warning: possible null reference
maybe?.Length;              // OK — null-conditional

if (maybe is not null) {
    int n = maybe.Length;   // OK — compiler tracked the null check
}
```

NRT is **compile-time only**. The runtime still allows null in any reference type (no runtime check is added). Nullability is metadata + warnings, not enforcement.

Despite that, NRT eliminates a huge class of bugs by surfacing nullability at the API boundary.

---

## Why it exists

`NullReferenceException` is the #1 runtime exception in .NET. It comes from "I thought this was non-null but it was."

Pre-NRT, every reference type could be null. APIs returned `string` but might give you `null`. You had to defensively null-check everywhere or just trust the docs.

NRT puts nullability into the type system:
- `string` = non-null (the API promises this).
- `string?` = nullable (the API might give you null).

The compiler tracks null state through your code and warns when you'd dereference something that might be null.

**Doesn't catch all bugs** (reflection, generics, dynamic, interop can sneak nulls past the checker), but catches most of them at compile time.

---

## Enabling NRT

Per project (recommended):

```xml
<PropertyGroup>
    <Nullable>enable</Nullable>
</PropertyGroup>
```

Per file:

```csharp
#nullable enable
// ... NRT active here ...
#nullable restore
```

Modes:
- `enable` — annotations honored, warnings emitted.
- `disable` — neither (pre-C# 8 behavior).
- `annotations` — annotations honored, no warnings emitted (for libraries: annotate your API, but don't warn your own code).
- `warnings` — warnings emitted but no annotations.

**For new code, always `enable`**. For migrating large codebases, sometimes `annotations` first while you clean up internal warnings.

---

## The annotations

```csharp
string s = "hello";        // non-null — type "string"
string? maybe = null;       // nullable — type "string?"
```

`?` on a reference type marks it as nullable. Internally, this is just metadata — both `string` and `string?` are the same runtime type (`System.String`).

For value types, `int?` is `Nullable<int>` — a different runtime type (a struct wrapping int + bool). NRT for reference types is purely compile-time.

```csharp
string a = null;     // ⚠ CS8600: converting null to non-nullable
string? b = null;    // OK
```

---

## Flow analysis

The compiler tracks each variable's null state through control flow:

```csharp
string? s = GetMaybe();
// state: "maybe null"

if (s != null) {
    int n = s.Length;   // state in this branch: "not null" — OK
}
// state after the if: still "maybe null" — outside the check

s.Length;               // ⚠ — might be null
```

Within an `if (s != null)` block, the compiler knows `s` is non-null. Outside, it goes back to "maybe."

Equivalent patterns it tracks:
- `if (s is not null)`
- `if (string.IsNullOrEmpty(s)) return; ... s.Length` (knows non-null after the early return)
- `s ?? throw new...` then s is non-null
- Pattern matching: `if (s is string str) { /* str is non-null */ }`

Some patterns are too complex for the analyzer — see "Helping the flow analysis" below.

---

## The null-forgiving operator (`!`)

When you know more than the compiler:

```csharp
string? maybe = GetMaybe();
int n = maybe!.Length;   // I promise it's not null — suppress the warning
```

`!` tells the compiler "trust me." If you're wrong, `NullReferenceException` at runtime.

Use sparingly. Every `!` is a place a bug could hide. Common legitimate uses:
- After validation the compiler can't see (`ValidateThis(input)` then use `input`).
- Initialization fields that the compiler considers null but you've initialized in a constructor it didn't analyze.
- Lazy initialization patterns.

```csharp
private string _name = null!;   // assigned later; suppress warning
```

This pattern (initialize to null!, set later) is common but fragile. Prefer `required` (C# 11+).

---

## `[NotNullWhen]` and friends

When NRT can't infer from method calls, attributes help:

```csharp
public static bool TryGet([NotNullWhen(true)] out string? value) {
    if (cond) { value = "x"; return true; }
    value = null;
    return false;
}

// At call site:
if (TryGet(out var s)) {
    s.Length;   // OK — NotNullWhen(true) tells compiler: when this returns true, s is non-null
}
```

`[NotNullWhen(true)]` on an `out` parameter: "when the method returns true, this out is not null."
`[NotNullWhen(false)]` for the opposite.

Other helpful attributes:
- `[NotNull]` on a return value: "always non-null."
- `[MaybeNull]` on a return value: "might be null even if T is non-null."
- `[NotNullIfNotNull("paramName")]`: "result is non-null if paramName is non-null."
- `[DisallowNull]`: "callers must pass non-null even though type is `T?`."
- `[AllowNull]`: "callers may pass null even though type is `T`."
- `[DoesNotReturn]`: "this method never returns (throws or exits)."
- `[DoesNotReturnIf(false)]`: "doesn't return if argument is false" — for guard methods.
- `[MemberNotNull("field")]`: "after this method returns, the named field is non-null."

These help the flow analysis understand library APIs.

Examples in BCL:

```csharp
public static void ThrowIfNull(
    [NotNull] object? argument,
    [CallerArgumentExpression("argument")] string? paramName = null) { ... }

// After:
ArgumentNullException.ThrowIfNull(arg);
arg.Method();   // compiler knows arg is non-null past the ThrowIfNull call
```

`[NotNull]` on the argument means: "if this method returns normally, the argument was not null."

---

## NRT in generics

```csharp
public T? FindOrDefault<T>() {
    // ...
}
```

What does `T?` mean when `T` is unconstrained?
- If T is `string`: `T?` = `string?` (nullable reference).
- If T is `int`: `T?` = `int?` (Nullable<int>).
- If T is `string?`: `T?` = `string?` (already nullable).

For clarity, constrain:

```csharp
public T? FindReference<T>() where T : class { ... }   // T? = T or null
public T? FindValue<T>() where T : struct { ... }       // T? = Nullable<T>
```

Or use `[MaybeNull]` for "the return might be default(T) which could be null":

```csharp
[return: MaybeNull]
public T FindOrDefault<T>() { ... }
```

The `where T : notnull` constraint forbids nullable Ts:

```csharp
public class Cache<TKey, TValue> where TKey : notnull { ... }
// Dictionary<TKey, TValue> uses this — keys can't be null.
```

---

## Migrating to NRT

For an existing codebase:

1. Set `<Nullable>annotations</Nullable>` — emits no warnings but enables `?` annotation syntax.
2. Annotate your **public APIs** first — mark return types, parameters, fields with `?` where they're nullable.
3. Switch to `<Nullable>enable</Nullable>` — now warnings emit.
4. Fix warnings file by file. Suppress with `!` only when justified.
5. Eventually: no warnings, no `!`s.

For new code: start with `enable`. Don't rack up technical debt.

---

## Common patterns

### Defensive guard with `[NotNull]`

```csharp
public void Process(string input) {
    ArgumentNullException.ThrowIfNull(input);
    // compiler knows input is non-null past this point
    Console.WriteLine(input.Length);
}
```

### Lazy init with `null!`

```csharp
private string _name = null!;   // will be set later

public void Initialize(string name) {
    _name = name;
}
```

Risky — if you read `_name` before `Initialize`, NRE at runtime. Compiler doesn't warn.

Better with `required`:

```csharp
public required string Name { get; init; }
```

Forces callers to set Name. Compile-time enforced.

### Pattern matching

```csharp
public void M(object? obj) {
    if (obj is string s) {
        Console.WriteLine(s.Length);   // s is non-null
    }
}
```

The `is` pattern introduces non-null variable.

### Null-coalescing with throw

```csharp
public string Get(string? maybe) {
    string s = maybe ?? throw new ArgumentNullException(nameof(maybe));
    return s.ToUpper();
}
```

Convert nullable to non-null, throwing if it was actually null.

---

## Internals — what the compiler emits

Annotations are metadata, not runtime checks. The compiler emits:

```il
.field private string Name
.custom instance void System.Runtime.CompilerServices.NullableAttribute::.ctor(uint8) = (01 00 02 ...) // 2 = nullable
```

For each member, an attribute records its nullability. Tools and other compilers read this when consuming the assembly.

At runtime: no check. `string` and `string?` are the same type. If null sneaks through (reflection, dynamic), no warning fires.

### Cross-assembly NRT

If you reference an assembly compiled with NRT off, all its reference types are "oblivious" (neither annotated nor warning-emitting from your perspective). You get warnings only against assemblies that compiled with NRT enabled.

Major BCL: nullable-annotated since .NET 6+. Older NuGet packages may not be. They're a porous boundary.

---

## Common bugs and gotchas

### Field initializer issues

```csharp
public class Demo {
    private string _name;   // ⚠ — never assigned

    public Demo(string name) { _name = name; }
}
```

Compiler warns: `_name` may be null after construction (if construction itself doesn't assign).

Fixes:
- Initialize at declaration: `private string _name = "";`.
- Always assign in constructor.
- Mark `required` (C# 11+).
- Suppress with `_name = null!;` if you'll set it elsewhere.

### NRT with collections

```csharp
List<string?> list = new() { "a", null, "b" };   // explicitly nullable elements
foreach (var s in list) {
    s.Length;   // ⚠ — s is string?
    if (s != null) s.Length;   // OK
}
```

Annotate the type parameter to mark elements nullable.

### Generics + unconstrained T

```csharp
public T? GetOrDefault<T>() => default;
```

For unconstrained T, default might be null (for reference T) or `default(T)` (for value T, often a zero, not null). The `T?` notation handles both.

### Dictionary indexing

```csharp
Dictionary<string, string> d = new();
string s = d["missing"];   // throws KeyNotFoundException, not null
```

NRT doesn't help here — the indexer is declared as returning `TValue` (non-null). The runtime exception is different. Use `TryGetValue` or `GetValueOrDefault`.

### Interop with non-NRT code

Old libraries return non-annotated types. The compiler treats them as "oblivious" — no warnings, but also no help.

Annotate at your wrapper boundary if needed.

### Reflection bypass

```csharp
typeof(C).GetField("_field")!.SetValue(obj, null);
```

NRT doesn't apply. Reflection can stuff nulls into non-nullable fields. Then runtime code dereferences → NRE.

For 99% of code, this isn't a real concern. For frameworks doing reflection-based deserialization, validators handle null-checking.

---

## NRT and inheritance

```csharp
public abstract class Base {
    public abstract string GetName();
}

public class Derived : Base {
    public override string? GetName() => null;   // ⚠ — return type changed
}
```

You can't relax nullability in an override. Derived's return type must be `string` (or stricter).

For covariance: override may **strengthen** (non-null returning instead of nullable). But not weaken.

Parameter contravariance is similar but for inputs.

---

## NRT in records

```csharp
public record Person(string Name, int Age);   // Name is non-null
public record Person(string? Name, int Age);   // Name nullable

var p = new Person("Alice", 30);   // OK
var q = new Person(null, 30);       // ⚠ — null to non-nullable
```

Records honor NRT like any class.

---

## When NRT bites in subtle ways

### Boxing-style escape

```csharp
public T? Get<T>() where T : class {
    return null;
}

string? s = Get<string>();   // ✓
string nonNull = s;          // ⚠ — flow tracks s as nullable
```

### Override with `notnull` constraint

```csharp
public class Base<T> where T : class? {
    public virtual T M() => default!;
}

public class Derived : Base<string> {
    // T = string (non-null) here
    public override string M() => "hi";
}
```

Constraints propagate to derived types, sometimes in surprising ways. Read warnings carefully.

---

## Performance

NRT is zero runtime cost. The compiler emits the same IL whether you have `string` or `string?`. Metadata is read at compile time only.

The savings come from **catching bugs at compile time** that would have been runtime exceptions. Net cost: zero. Net benefit: large.

---

## When to suppress with `!`

- Compiler can't see through a complex check.
- Initialization that the analyzer doesn't track.
- Defensive `!` for "I checked elsewhere."

When NOT to suppress:
- "I'm pretty sure" without verification.
- Anywhere you can refactor to prove non-null at compile time.
- Defensive coding habit — every `!` is a future NRE waiting to happen.

---

## Summary

- NRT marks references as nullable (`string?`) or non-null (`string`).
- Compile-time only — no runtime check or type change.
- Catches a huge class of NullReferenceException bugs.
- Flow analysis tracks null state through if / pattern / early-return.
- Attributes (`NotNullWhen`, `MaybeNull`, etc.) help library authors annotate behavior.
- `!` suppresses warnings — use sparingly.
- Enable globally for new code via `<Nullable>enable</Nullable>`.

→ Next: [02-PatternMatchingDeep.md](02-PatternMatchingDeep.md)
