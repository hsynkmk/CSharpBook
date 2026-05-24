# File-Scoped Types (`file` modifier, C# 11+)

## What it is

The `file` modifier limits a type's visibility to its **single source file**. Other files in the same assembly can't see it — even if they're in the same namespace. Useful for source generators emitting helpers, or for isolating implementation details that should never leak.

```csharp
// File: Service.cs
file class Internals {
    public static int Compute(int x) => x * 2;
}

public class Service {
    public int Use(int x) => Internals.Compute(x);
}

// File: Other.cs
public class Other {
    public int Y() => Internals.Compute(5);   // ❌ — Internals not visible here
}
```

Added in C# 11 (2022). Mostly used by **source generators** that emit per-file helpers without polluting the assembly's namespace.

---

## Why it exists

Source generators often need to emit helper types that:
- Should be private to a specific file (to avoid name collisions with user code).
- Aren't meant to be part of the public or internal API.

Pre-`file`, generators emitted types with mangled names (e.g., `__Generated_Helpers`) hoping nothing else used them. Fragile.

`file` makes this clean: the type genuinely cannot be referenced from outside its file. Source generators emit safe per-file types without coordination.

For hand-written code, `file` is occasionally useful for "this helper is for this file only, full stop." But most app code uses `internal` and lives with the slightly broader scope.

---

## Syntax

```csharp
file class Helper { ... }
file struct Data { ... }
file interface IPrivate { ... }
file enum Color { ... }
file delegate void Callback(int x);
file record Note(string Text);
```

Works on any type. Members of a `file` type follow normal access rules — they can be public, private, etc., but their visibility is bounded by the file modifier.

---

## Behavior

- Visible only within the source file.
- Cannot be referenced from another file, even in the same namespace or assembly.
- Cannot be a member of a non-`file` type (because that type might be referenced from elsewhere; can't expose `file` content).
- Cannot be a generic parameter constraint of a non-file type (same reason).
- IS allowed: extending another `file` type via inheritance; implementing interfaces; using as a private field of a public class — but only if the field's type itself is also constrained appropriately.

```csharp
file class Internal { }
public class Service {
    private Internal? _internal;   // ⚠ if anyone outside this file inspects via reflection
}
```

Hmm, actually let's verify: a private field of type `file class Internal` is allowed (the field is internal to the public class), but exposing `Internal` in a public signature is forbidden:

```csharp
public Internal GetInternal() => new();   // ❌ — Internal isn't visible to consumers
```

The compiler enforces consistency.

---

## Common pattern: source generator helpers

A source generator that adds a `[Serialize]` attribute might emit per-file helper code:

```csharp
// User's file: Person.cs
[Serialize]
public partial class Person {
    public string Name { get; set; } = "";
}

// Generated file: Person.Serialize.g.cs
file static class PersonSerializer {
    public static string Serialize(Person p) => /* ... */;
    public static Person Deserialize(string json) => /* ... */;
}

partial class Person {
    public string ToJson() => PersonSerializer.Serialize(this);
    public static Person FromJson(string json) => PersonSerializer.Deserialize(json);
}
```

`PersonSerializer` is `file` — it can't conflict with another generated `PersonSerializer` from a different file or a user's class.

System.Text.Json's source generator, ASP.NET Core's [LoggerMessage] generator, and other modern source generators use this pattern extensively.

---

## File types and partial classes

`file` works with partial:

```csharp
// Person.g.cs
file partial class Helper {
    public static int X => 5;
}

// Person.cs
file partial class Helper {
    public static int Y => 10;
}
```

But both parts must be in the SAME file (otherwise different "files" → different scope). For multi-file partial classes, use `internal` instead.

---

## Internals — how the compiler emits `file` types

The compiler emits the type with a special name:

```il
.class private auto ansi nested sealed beforefieldinit
    '<Helpers>FB$0_Internal' { ... }
```

The name includes a hash of the file path. This ensures uniqueness across files but is illegal in user-written C# (the `<>` characters).

The metadata is `internal` to the assembly (technically), but the compiler enforces file-scope at the source level. Reflection can still discover and access these types (they're `internal` runtime-wise).

For Native AOT, file types compile to whatever the rest of the type system needs — usually fine.

---

## When to use `file`

✓ **Source generators emitting per-file helpers** — primary use case.
✓ **Isolating implementation details** that should never leak.
✓ **Avoiding name collisions** in shared namespaces.

✗ Regular application code where `internal` works.
✗ Types you want testable from outside the file.
✗ Types reused across multiple source files.

For hand-written code, you'll rarely reach for `file`. It's mostly a tool for generators.

---

## Reflection considerations

```csharp
file class MyHelper { }

Type t = typeof(MyHelper);   // Works from inside the file
Type t2 = Type.GetType("...");   // Can find from outside, with the mangled name
```

Reflection bypasses file scoping. The type is `internal` at the metadata level. If you really need to hide from reflection too, that's a different problem (and not really solvable without obfuscation).

---

## Comparison with `internal`

| | `internal` | `file` |
|---|---|---|
| Visible in | Same assembly | Same file |
| Used by | Application code, libraries | Source generators (mostly) |
| Cross-file collision risk | None (full assembly visibility) | None (each file scoped) |
| Reflection | Findable | Findable (with mangled name) |
| Use in API surface | Yes (internally) | Limited — can't expose |

For 95% of code, `internal` is enough. `file` is for the specific case of "this MUST NOT leak."

---

## Common bugs

### Trying to share a file type across files

```csharp
// File A
file class Shared { public int X; }

// File B
var s = new Shared();   // ❌ — Shared not visible here
```

If you need cross-file sharing, use `internal`.

### Exposing a file type in a public API

```csharp
file class Internal { }
public class Service {
    public Internal Get() => new();   // ❌ — can't expose file-scoped type
}
```

The compiler refuses — consumers couldn't use it.

### Forgetting that file types can't be partial across files

```csharp
// File A
file partial class Helper { /* ... */ }

// File B
file partial class Helper { /* ... */ }   // ❌ — different files; different scopes; conflicts
```

For multi-file partial, use `internal` or `public`.

---

## Summary

- `file` limits a type's visibility to its single source file.
- Primary use: source generators emitting per-file helpers without name collisions.
- Hand-written code rarely needs it; `internal` usually suffices.
- Reflection bypasses (file types are `internal` in metadata).
- Can't expose in any wider-scoped public/internal API.

→ Next: [06-TopLevelStatements.md](06-TopLevelStatements.md)
