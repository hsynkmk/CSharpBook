# 06 — Equality (Equals, ==, GetHashCode)

## ⚡ 30-second answer

There are two equality notions: **reference equality** (same object) and **value equality** (same contents). By default, **classes use reference equality** and **structs use value equality** (field-by-field). `==` is a **static operator** resolved at compile time; `Equals` is a **virtual method** resolved at runtime. The golden rule: **if you override `Equals`, you must override `GetHashCode`** — and equal objects must return equal hash codes, or they break as dictionary/hashset keys. **Records** generate correct value equality (and `GetHashCode`) for you.

---

## Core mechanics

**Three ways to compare**:
```csharp
a == b                    // operator (static, compile-time type); default = reference for classes
a.Equals(b)               // virtual method (runtime type); override for value equality
ReferenceEquals(a, b)     // always identity, ignores overloads
object.Equals(a, b)       // null-safe static helper
```

**The `GetHashCode` contract**:
1. If `a.Equals(b)` is true → `a.GetHashCode() == b.GetHashCode()` (**required**).
2. Equal hash codes do **not** imply equality (collisions are allowed).
3. Hash code must be **stable** while the object is used as a key (don't derive it from mutable fields).

**Correct override**:
```csharp
sealed class Money : IEquatable<Money> {
    public decimal Amount { get; }
    public string Currency { get; }
    public bool Equals(Money? o) => o is not null && Amount == o.Amount && Currency == o.Currency;
    public override bool Equals(object? o) => Equals(o as Money);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);   // don't hand-roll
}
```

**Records do this automatically**:
```csharp
record Money(decimal Amount, string Currency);   // value Equals, ==, GetHashCode, ToString — generated
```

**`IEqualityComparer<T>`** — supply custom equality to a `Dictionary`/`HashSet`/LINQ without changing the type (e.g., `StringComparer.OrdinalIgnoreCase`).

---

## Comparison tables

| | `==` operator | `Equals` method |
|---|---|---|
| Binding | **static / compile-time** | **virtual / runtime** |
| Default (class) | reference | reference |
| Default (struct) | not defined (must implement) | value (reflection-based, slow) |
| Overridable | overload per type | override the virtual method |
| Null-safe | yes (with null literal) | use `object.Equals(a,b)` |

| Type | Default equality | Generated `GetHashCode`? |
|---|---|---|
| `class` | reference | no (object identity) |
| `struct` | value (slow reflection) | yes (slow) |
| `record class` / `record struct` | **value (generated, fast)** | **yes** |

---

## 🪤 Traps & gotchas

- **Override `Equals` but not `GetHashCode`** → the object works for `==`/`Equals` but **silently fails as a dictionary/hashset key** (you can't find it). The #1 trap.
- **Mutable hash code**: deriving `GetHashCode` from a field you later mutate → the key is "lost" in the dictionary. Use immutable fields for keys.
- **`==` is not virtual**: `obj1 == obj2` where both are typed `object` uses **reference** equality even if the runtime type overloads `==` — because operators bind to the compile-time type. `Equals` would dispatch correctly.
- **Struct default `Equals` is slow**: `ValueType.Equals` boxes and uses reflection. Implement `IEquatable<T>` (or use `record struct`) for performance.
- **Inheritance + value equality** is hard to get right (symmetry: `a.Equals(b)` must equal `b.Equals(a)`). Records handle it with a type check; hand-rolled hierarchies often violate symmetry.
- **`GetHashCode` using a captured random/time** → non-deterministic, breaks lookups.
- **Floating-point equality**: `==` on `double` is exact-bit; tiny rounding differences fail. Compare with a tolerance.

---

## ❓ Likely questions

**Q: Difference between `==` and `Equals`?**
A: `==` is a static operator bound at compile time (default reference for classes); `Equals` is virtual, dispatched at runtime. Override `Equals` for value semantics; `==` only changes if you overload it.

**Q: Why must `Equals` and `GetHashCode` be overridden together?**
A: Hash-based collections bucket by `GetHashCode`, then confirm with `Equals`. If equal objects have different hash codes, they land in different buckets and can't be found — the contract requires equal ⇒ equal hash.

**Q: What's the `GetHashCode` contract?**
A: Equal objects must have equal hash codes; collisions are allowed (unequal objects may share a hash); the hash must be stable while used as a key.

**Q: How do records help with equality?**
A: They auto-generate value-based `Equals`, `==`/`!=`, `GetHashCode`, and `ToString` from the members — correct and concise.

**Q: How do you customize equality without editing the type?**
A: Pass an `IEqualityComparer<T>` (e.g., `StringComparer.OrdinalIgnoreCase`) to the dictionary/set/LINQ method.

**Q: Why is the default struct `Equals` slow?**
A: It uses reflection over fields and may box. Implement `IEquatable<T>` to provide a fast, allocation-free comparison.

**Q: Reference vs value equality?**
A: Reference = same object instance (identity). Value = same contents. Classes default to reference, structs/records to value.

---

## 🎓 Senior Extra

- **`HashCode.Combine` / `HashCode` builder**: the right way to combine fields — mixes well and is seeded per-process (mitigates hash-flooding DoS). Never `a.GetHashCode() ^ b.GetHashCode()` naively (poor distribution; `^` of equal fields cancels).
- **`IEquatable<T>`** avoids boxing for value types and is what generic collections prefer; always pair it with the `object.Equals` override and operators for consistency.
- **Records' equality is type-exact**: a `record A` and a `record B : A` with the same data are **not** equal (records compare `EqualityContract`/runtime type), which sidesteps the inheritance-symmetry problem but can surprise.
- **`record struct` vs `struct`**: record struct generates fast value equality + `GetHashCode`; a plain struct's defaults are reflection-based and slow — implement `IEquatable<T>` for hot paths.
- **Comparers vs equality**: `IComparer<T>`/`IComparable<T>` define **ordering** (sorting, sorted collections) — separate from `IEqualityComparer<T>`/`IEquatable<T>` (membership/hashing). `CompareTo == 0` should be consistent with `Equals`.
- **Cache invalidation via equality**: distributed caches and EF Core change tracking ([17](17-EFCore.md)) rely on correct key equality; a broken `GetHashCode` causes subtle "can't find the entity" bugs.
- **Frozen/perf collections** ([05](05-Collections.md)) and dictionary throughput hinge on hash quality — a clustering hash degrades O(1) to O(n).

→ Deeper: [`../CSharpBook/17-BestPractices/14-Equality.md`](../CSharpBook/17-BestPractices/README.md)
