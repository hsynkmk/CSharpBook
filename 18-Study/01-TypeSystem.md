# 01 — Type System (value vs reference, boxing, struct/class/record)

## ⚡ 30-second answer

C# types are **value types** (struct, enum, primitives) or **reference types** (class, interface, delegate, array, string). A **value type variable holds the data directly**; a **reference type variable holds a reference (pointer) to data on the heap**. Copying a value type copies the bytes; copying a reference copies the pointer (both point to the same object). **Boxing** wraps a value type in a heap object to treat it as `object`/interface — it allocates and is a common perf trap.

---

## Core mechanics

**Where data lives** (rule of thumb, not a language guarantee):
- Value type **local** → stack. Value type **field of a class** → inline inside that object on the heap.
- Reference type → the *reference* lives where the variable lives; the *object* lives on the heap.

```csharp
struct Point { public int X, Y; }
class Box   { public int X, Y; }

var p1 = new Point { X = 1 }; var p2 = p1; p2.X = 99;  // p1.X == 1  (copy of bytes)
var b1 = new Box   { X = 1 }; var b2 = b1; b2.X = 99;  // b1.X == 99 (same object)
```

**Boxing / unboxing**:
```csharp
int i = 42;
object o = i;        // BOX: allocates a heap object, copies 42 into it
int j = (int)o;      // UNBOX: copies the value back out (must be exact type)
```
- Triggered by: assigning a value type to `object`/`dynamic`/an interface, adding to non-generic collections (`ArrayList`), string-formatting a struct in some paths.
- Cost: a heap allocation + GC pressure. Avoid with **generics** (`List<int>` never boxes).

**`string`** is a reference type but **immutable** and has **value-like equality** (`==` compares contents).

---

## Comparison tables

| | Value type (struct) | Reference type (class) |
|---|---|---|
| Variable holds | the data | a reference to the data |
| Copy semantics | copies all bytes | copies the reference |
| Default value | zeroed struct (not null) | `null` |
| Allocation | inline (stack/in object) | heap |
| Equality (default) | **structural** (field-by-field) | **reference** identity |
| Can be `null`? | only as `Nullable<T>` (`int?`) | yes |
| Inheritance | no (sealed); can implement interfaces | yes |

| Type kind | Equality default | Mutability idiom | Use for |
|---|---|---|---|
| `class` | reference | mutable | entities/identity, large/poly objects |
| `struct` | value (slow reflection-based) | should be **immutable & small** (≤16 bytes) | small values (Point, Money) |
| `record class` | **value** (generated) | immutable (init) | DTOs, value objects |
| `record struct` | value | value object, no heap alloc | small immutable values |

---

## 🪤 Traps & gotchas

- **Boxing in disguise**: calling a non-overridden `ToString()`/`Equals()` on a struct, using a struct in a non-generic API, or putting a struct in an `object[]` boxes it.
- **Mutable structs are evil**: `list[0].X = 5` on a `List<MutableStruct>` mutates a *copy* (or won't compile) — surprising. Make structs immutable.
- **Default struct is not null, it's zeroed** — `default(Point)` has `X=0,Y=0`; there's no "uninitialized" state.
- **`struct` equality default is slow** — the default `ValueType.Equals` uses reflection. Implement `IEquatable<T>` (or use `record struct`) for hot paths ([06-Equality.md](06-Equality.md)).
- **Large structs are expensive to copy** — passing a 64-byte struct by value copies 64 bytes every call. Use `in`/`ref` or a class.
- **`Nullable<T>` (`int?`) is a struct** — it doesn't allocate; it's `HasValue` + `Value`. But boxing an `int?` boxes the underlying `int` (or `null`).

---

## ❓ Likely questions

**Q: Difference between value and reference types?**
A: Value types hold data directly and copy by value; reference types hold a pointer to a heap object and copy the reference. Default equality is structural vs reference.

**Q: What is boxing and why care?**
A: Wrapping a value type in a heap object to use it as `object`/interface. It allocates → GC pressure. Avoid via generics.

**Q: Is `string` a value or reference type?**
A: Reference type, but immutable with value-based `==`. So it *behaves* value-like for comparison.

**Q: Where do value types live — stack or heap?**
A: Depends: locals on the stack, but a value-type *field* of a class lives on the heap inside that object. "Value type = stack" is a simplification.

**Q: When use a struct over a class?**
A: Small (≤16 bytes), immutable, value-semantics, short-lived, allocation-sensitive (avoid heap/GC). Otherwise class.

**Q: What's `default(T)` for a struct vs class?**
A: Struct → all fields zeroed (a real instance). Class → `null`.

**Q: Does `int?` allocate?**
A: No — `Nullable<int>` is a struct (HasValue + value). Boxing it does allocate.

---

## 🎓 Senior Extra

- **`ref struct`** (e.g., `Span<T>`): stack-only, can never be boxed, heap-allocated, captured by a lambda, or used as a generic type arg / async local — the compiler enforces it so it can't outlive its stack frame ([13-MemoryAndGC.md](13-MemoryAndGC.md)).
- **`readonly struct`** guarantees immutability and lets the compiler avoid **defensive copies** when the struct is accessed through a `readonly` field or `in` parameter (otherwise each member call on a non-readonly struct via `in` copies it).
- **Pass by `in`** for large readonly structs to avoid copies (but beware defensive copies if the struct isn't `readonly`).
- **Struct layout / `[StructLayout]`**: matters for interop and cache behavior; field order affects size due to alignment/padding.
- **String interning**: literals are interned (shared); `string.Intern` exists but is rarely worth it. `==` on strings is content equality; `ReferenceEquals` checks identity.
- **Escape analysis (.NET 10)**: the JIT can stack-allocate some objects that provably don't escape — reducing heap pressure without code changes.
- Generic specialization: the JIT shares one machine-code body for all **reference-type** `T` (references are uniform) but generates a **separate** specialized body per **value-type** `T` (so `List<int>` is monomorphized and box-free).

→ Deeper: [`../CSharpBook/03-TypeSystem/`](../CSharpBook/03-TypeSystem/README.md), [`../CSharpBook/09-MemoryPerformance/`](../CSharpBook/09-MemoryPerformance/README.md)
