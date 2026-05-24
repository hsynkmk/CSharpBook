# 08 — Modern C# (pattern matching, records, NRT, C# 7→14)

## ⚡ 30-second answer

Modern C# trends toward **immutability, expressiveness, and less boilerplate**: **records** (value-equality data types), **pattern matching** (switch expressions, property/list patterns), **nullable reference types** (compile-time null-safety), **collection expressions** `[..]`, **primary constructors**, **target-typed `new`**, **`required` members**, and (C# 14) the **`field` keyword** and **extension members**. Know the *headline* features and *why* they exist: safer null handling, concise data types, exhaustive branching.

---

## Core mechanics — the high-value features

**Records** (C# 9) — concise immutable types with value equality:
```csharp
record Person(string Name, int Age);                 // value Equals/==/GetHashCode/ToString + Deconstruct
var p2 = p1 with { Age = 31 };                        // non-destructive copy
record struct Point(int X, int Y);                    // value type record (no heap alloc)
```

**Pattern matching** (C# 7→11):
```csharp
string Describe(object o) => o switch {
    null                       => "null",
    int n when n > 0           => "positive",
    string { Length: > 5 } s   => $"long string {s}",   // property pattern
    [var first, .., var last]  => $"{first}..{last}",   // list pattern (C# 11)
    Point (0, 0)               => "origin",             // positional
    _                          => "other"               // discard / default
};
```

**Nullable reference types (NRT)** (C# 8) — opt-in (`<Nullable>enable</Nullable>`):
```csharp
string  notNull;     // compiler warns if it could be null
string? maybeNull;    // explicitly nullable
maybeNull!.Length;    // null-forgiving operator (you assert non-null — use sparingly)
```
NRT is **compile-time only** (annotations + flow analysis); it doesn't change runtime types.

**Less ceremony**:
```csharp
Dictionary<string,int> d = new();                     // target-typed new (C# 9)
int[] xs = [1, 2, 3];  List<int> ys = [..xs, 4];      // collection expressions + spread (C# 12)
public class Svc(IRepo repo) { }                       // primary constructor (C# 12)
required string Name { get; init; }                    // must be set at init (C# 11)
```

**C# 14 highlights**: the **`field`** keyword (access the backing field in a property without declaring it), **extension members** (extension properties/static members), **`?.=` null-conditional assignment**, partial constructors/events.

---

## Comparison tables

| Feature | Version | Why |
|---|---|---|
| Tuples, `out var`, local funcs, pattern matching | C# 7 | conciseness, multiple returns |
| **NRT**, default interface methods, `await using` | C# 8 | null-safety, async streams |
| **records**, `init`, top-level statements, target-typed `new` | C# 9 | immutability, less boilerplate |
| file-scoped namespaces, global usings, `record struct` | C# 10 | tidiness |
| **`required`**, raw strings `"""`, list patterns, generic math | C# 11 | safety, expressiveness |
| **primary constructors**, **collection expressions** `[..]` | C# 12 | concise types/collections |
| `params` collections, `lock` object type, `\e` | C# 13 | ergonomics |
| **`field` keyword**, **extension members**, `?.=` | C# 14 | property/extension ergonomics |

| | `record class` | `record struct` | `class` |
|---|---|---|---|
| Equality | value | value | reference |
| Heap alloc | yes | no | yes |
| Immutable idiom | `init`/positional | `readonly` | mutable |

---

## 🪤 Traps & gotchas

- **NRT is compile-time only**: `string` not annotated nullable can *still* be null at runtime (deserialization, reflection, `default`). Annotations are warnings, not runtime guards — keep guard clauses ([07](07-Exceptions-Idioms.md)).
- **`!` (null-forgiving)** silences the compiler without checking — overusing it defeats NRT. Use only when you genuinely know better.
- **Records are shallow-immutable**: `record Order(List<Item> Items)` — the list is still mutable; `with` does a **shallow** copy. Use immutable collections for true immutability.
- **Record value equality can surprise** for keys/caches if a member is mutable (hash changes) — same rule as [06](06-Equality.md).
- **Switch expression non-exhaustiveness**: a missing case throws `SwitchExpressionException` at runtime; add `_` or cover all cases.
- **Primary constructor parameters are in scope for the whole class** — capturing them creates hidden fields; mutating one is allowed but subtle.
- **`with` on a class record** copies; on a `record struct` it's a value copy — semantics differ.

---

## ❓ Likely questions

**Q: What problem do records solve?**
A: Boilerplate for immutable data types — they generate value equality, `GetHashCode`, `ToString`, `Deconstruct`, and `with`-expressions, so DTOs/value objects are one line.

**Q: What is NRT and is it enforced at runtime?**
A: Nullable reference types — opt-in compile-time flow analysis that warns about possible nulls. It does **not** change runtime behavior; null can still occur.

**Q: `record` vs `class` vs `struct`?**
A: `record class` = reference type with value equality (DTOs); `record struct` = value type with value equality (small values, no alloc); plain `class` = reference equality, mutable entities.

**Q: What's a switch expression and its risk?**
A: A concise, expression-form switch returning a value with patterns. Risk: non-exhaustive matches throw at runtime — include a discard `_`.

**Q: What does `with` do?**
A: Non-destructive mutation — creates a copy of a record with some properties changed. It's a **shallow** copy.

**Q: What's new/interesting in C# 14?**
A: The `field` keyword (backing-field access without a manual field), extension members (extension properties/static members), and `?.=` null-conditional assignment.

**Q: target-typed `new` — why?**
A: Avoids repeating the type: `Dictionary<string,int> d = new();` — inferred from the declared type.

---

## 🎓 Senior Extra

- **NRT nullability flows through generics & APIs**: `[NotNullWhen(true)]`, `[MaybeNull]`, `[DisallowNull]` attributes let APIs express conditional nullability (e.g., `TryGetValue` sets the out non-null when it returns true). Library authors annotate; consumers get accurate warnings.
- **Records under the hood**: the compiler emits `EqualityContract`, `Equals`, `GetHashCode`, `PrintMembers`, `Deconstruct`, a copy constructor, and a `with`-clone — inheritance compares `EqualityContract` (runtime type), so derived records aren't equal to base ([06](06-Equality.md)).
- **Pattern matching compiles efficiently**: the compiler can lower switches over constants to jump tables and orders type checks; list/property patterns are sugar over indexers/length checks — readable *and* fast.
- **`required` + `init`** gives "mandatory immutable" without a constructor — pairs with object initializers and `SetsRequiredMembers`.
- **Source generators** (Roslyn `IIncrementalGenerator`) increasingly replace runtime reflection (System.Text.Json source-gen, LoggerMessage, regex) — AOT/trim-friendly ([21](21-Deployment-Perf-Tooling.md)); the "modern C#" direction is *compile-time* metaprogramming.
- **`Span`/`ref struct` ergonomics** improved across versions (`params ReadOnlySpan<T>`, first-class span conversions in C# 14) → low-allocation APIs are now idiomatic ([13](13-MemoryAndGC.md)).
- **Raw string literals** (`"""`) + UTF-8 literals (`"..."u8`) help with embedded JSON/SQL and byte buffers without escaping/allocation.

→ Deeper: [`../CSharpBook/10-AdvancedLanguage/`](../CSharpBook/10-AdvancedLanguage/README.md), [`../CSharpBook/11-ModernFeatures/`](../CSharpBook/11-ModernFeatures/README.md), [`../DotNet10-CSharp14.md`](../DotNet10-CSharp14.md)
