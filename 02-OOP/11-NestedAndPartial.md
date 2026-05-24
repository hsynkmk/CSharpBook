# Nested Types and Partial Classes

## Part 1 — Nested types

A type declared inside another type. The outer type acts as a namespace + access boundary.

```csharp
public class Tree {
    public class Node {
        public int Value;
        public Node? Left;
        public Node? Right;
    }

    public Node? Root;
}

var t = new Tree();
t.Root = new Tree.Node { Value = 42 };
```

`Node` is accessed as `Tree.Node` — its qualified name includes the outer type.

### What can be nested

Inside a class, struct, or interface you can declare:
- Classes
- Structs
- Records
- Enums
- Interfaces
- Delegates

```csharp
public class Outer {
    public class InnerClass { }
    public struct InnerStruct { }
    public record InnerRecord(int X);
    public enum InnerEnum { A, B }
    public interface InnerInterface { }
    public delegate void InnerDelegate(int x);
}
```

### Access from inside

Nested types can access **private** members of the outer class:

```csharp
public class Wallet {
    private decimal _balance;

    public class Snapshot {
        public Snapshot(Wallet w) {
            Balance = w._balance;   // ✓ access to private
        }
        public decimal Balance { get; }
    }
}
```

The outer can also access private nested members. This privileged access is the main reason to nest types — close coupling that needs internal visibility.

### Access modifiers on nested types

Unlike top-level types, nested types can use **all** access modifiers:
```csharp
public class Outer {
    public class Pub { }       // visible everywhere Outer is
    internal class Int { }     // same assembly
    protected class Prot { }   // Outer + subclasses
    private class Priv { }     // only inside Outer
    private protected class PP { }
    protected internal class PI { }
}
```

A nested type's effective access is `min(outer's access, its own access)` — if `Outer` is `internal`, a `public class Pub` inside is effectively internal.

### When to nest

✓ The nested type is **only** used by the outer.
✓ It needs access to the outer's privates.
✓ Iterator/enumerator state machines (the compiler nests them automatically).
✓ The nested type is a small helper that doesn't deserve its own file.

✗ The nested type might be reused elsewhere — keep it top-level.
✗ The nested type is a major class on its own — file pollution.

### Practical examples

#### Builder nested inside the thing it builds

```csharp
public class HttpRequest {
    public Uri Url { get; }
    public string Method { get; }
    private HttpRequest(Uri u, string m) { Url = u; Method = m; }

    public class Builder {
        private Uri? _url;
        private string _method = "GET";
        public Builder WithUrl(string url) { _url = new Uri(url); return this; }
        public Builder WithMethod(string m) { _method = m; return this; }
        public HttpRequest Build() => new(_url ?? throw new(), _method);
    }
}

var req = new HttpRequest.Builder()
    .WithUrl("https://example.com")
    .WithMethod("POST")
    .Build();
```

#### State enum inside the type that uses it

```csharp
public class Connection {
    public enum State { Disconnected, Connecting, Connected, Disconnecting }
    public State Status { get; private set; }
}

Connection.State s = Connection.State.Connected;
```

Scope and namespace cleanly: no top-level `ConnectionState` enum cluttering the namespace.

#### Pimpl pattern (rare in C#)

```csharp
public class FastFile {
    private class Impl { /* lots of internal state */ }
    private readonly Impl _impl = new();

    public FastFile(string path) { /* configure _impl */ }
    public void Read() { /* delegate to _impl */ }
}
```

The implementation class is fully hidden. Used when you want a stable public interface and freedom to change internals — unusual in C#, more common in C++.

---

## Internals — nested types under the hood

A nested type is just an ordinary type with metadata flags marking its nesting and a reference to the enclosing type. At runtime it's no different from any other type — the JIT doesn't treat `Outer.Inner` specially.

The "private access to outer" rule is enforced at the **compiler level**. The compiler verifies it; the runtime trusts the compiler.

Each iterator (`yield return`) and `async` method's state machine is implemented as a compiler-generated nested type:

```csharp
public IEnumerable<int> Numbers() {
    yield return 1; yield return 2; yield return 3;
}
```

Compiles to:

```il
.class private auto ansi sealed nested
   '<Numbers>d__0' extends [System.Runtime]System.Object
   implements ... IEnumerable<int32> ...
```

The state machine class lives inside the same outer type as the enumerator method, with a mangled name beginning with `<`. We never see it in our source, but it's there.

---

## Part 2 — Partial classes

The `partial` keyword lets a single type definition span multiple files.

```csharp
// File: Person.designer.cs (auto-generated)
public partial class Person {
    public int Id { get; set; }
    public string Name { get; set; } = "";
}

// File: Person.cs (your hand-written)
public partial class Person {
    public string Greet() => $"Hello, I'm {Name}";
}
```

After compilation, both pieces merge into one class. There's no runtime artifact for partiality — it's purely a source-level convenience.

### Why partial classes exist

**1. Generated + hand-written code coexistence.** The original motivation. WinForms designers, EF Core scaffolding, ASP.NET WebForms, source generators all benefit:
- Designer generates one file.
- You write your logic in another.
- Regenerating the designer file doesn't blow away your code.

**2. Separating concerns within a single class.** Split a large class into logical sections by file. Used sparingly — usually a sign the class is too large.

### Rules

- All parts must have `partial`.
- All parts must be in the same **namespace** and **assembly**.
- Access modifiers, type kind (`class`/`struct`/`record`/`interface`), base classes/interfaces must be consistent across parts.
- Attributes from all parts are combined.

```csharp
// Part 1
partial class Foo : Base { }

// Part 2 — same baseclass required
partial class Foo : Base, IComparable { }   // ✓ — can add interfaces
partial class Foo : OtherBase { }            // ❌ — conflicting base
```

### Partial methods

A method declared in one part and (optionally) implemented in another.

Two flavors:

#### Original — "if not implemented, removed entirely"

```csharp
// Generated
partial class Customer {
    partial void OnNameChanged();   // declaration only
    public void SetName(string n) {
        _name = n;
        OnNameChanged();             // call — present whether or not implemented
    }
}

// Hand-written (optional)
partial class Customer {
    partial void OnNameChanged() {
        AuditLog.Record("name changed");
    }
}
```

Rules:
- Implicit `private`.
- Return type must be `void`.
- No `out` parameters.
- If not implemented, the **call is removed at compile time** — arguments are not evaluated. Zero overhead for unused hooks.

This is the magic source generators love. They emit code that calls hooks, and only if you opt in does any work happen.

#### Extended (C# 9+) — "must be implemented"

```csharp
partial class Service {
    public partial Task<User?> GetUser(int id);
}

partial class Service {
    public partial Task<User?> GetUser(int id) {
        return _db.Users.FindAsync(id).AsTask();
    }
}
```

When the partial method has an access modifier or non-void return type, it **must** be implemented somewhere.

Used by:
- `[GeneratedRegex]` — declare the partial method, the source generator implements it with optimized regex code.
- `[LoggerMessage]` — declare the partial method, the generator implements optimized logging.
- ASP.NET Core source-generated routing.
- `[JsonSerializable]` — partial methods on a `JsonSerializerContext` class.

### Partial properties (C# 13+)

```csharp
public partial class User {
    public partial string DisplayName { get; }
}

public partial class User {
    public partial string DisplayName => $"{FirstName} {LastName}";
}
```

Same idea, properties. Used by source generators.

### Partial constructors and events (C# 14)

```csharp
public partial class Service {
    public partial Service(IDbContext db);    // declaration
}
public partial class Service {
    private readonly IDbContext _db;
    public partial Service(IDbContext db) {   // implementation
        _db = db;
    }
}
```

Generators can declare the constructor signature; the user (or another generator) provides the body. Same for events.

---

## When to use partial

**Yes:**
- Cooperating with source generators (often required, not optional).
- Auto-generated designer code + hand-written companion code.
- A class genuinely so large it needs splitting (consider refactoring instead).

**No:**
- Hiding complexity by splitting one class across many files when it should be multiple classes.
- "Better organization" when the class is small enough to fit in one file.

---

## Internals — how partial classes are compiled

Partiality is purely a compile-time concept. The Roslyn compiler:
1. Reads all source files in the compilation.
2. For each set of `partial` declarations of the same type, merges them into a **single type definition**.
3. Emits one type to IL — no trace of partiality.

You can't reflect to discover that a class was partial in source. The metadata doesn't carry that info; once compiled, it's just a class.

Generated source files (from source generators) participate in this merge. That's how source generators contribute to your existing type without modifying your file — they emit a partial part.

---

## Common bugs

### Partial without `partial` everywhere

```csharp
// File 1
partial class Foo { void A() { } }

// File 2
class Foo { void B() { } }    // ❌ — missing partial
```

Compiler error: only one definition is allowed if you don't say partial.

### Mismatched base class

```csharp
partial class Bar : Animal { }
partial class Bar : Vehicle { }   // ❌ conflicting base
```

### Generic constraint duplication

```csharp
partial class List<T> where T : class { }
partial class List<T> where T : struct { }   // ❌ conflicting constraint
```

Constraints must match exactly (or be specified on only one part).

### Nested type access surprises

Outer class can access private members of nested. Nested can access private members of outer. But **siblings** (two nested types in the same outer) can also see each other's privates:

```csharp
public class Outer {
    private class A { public int X; }
    private class B {
        void Use(A a) { a.X = 5; }    // ✓
    }
}
```

This sometimes surprises people.

---

## Summary

### Nested types
- Useful for tightly coupled helpers, builders, state machines, enums scoped to one class.
- Can access enclosing private members.
- Compiler uses them heavily for iterator and async state machines.

### Partial types
- Split one type across multiple source files.
- Crucial for source generator integration.
- All parts merge into one type at compile time — no runtime artifact.

### Partial members
- Methods (C# 3+), properties (C# 13+), constructors/events (C# 14+).
- Optional implementation original form (void only, removed if not implemented).
- Required implementation extended form (with modifiers/non-void).

→ Next: [12-PrimaryConstructors.md](12-PrimaryConstructors.md)
