# Boxing and Unboxing

## What it is

**Boxing** is the runtime operation that wraps a value-type instance in a heap-allocated object, so it can be referenced by a variable of type `object` or an interface. **Unboxing** is the reverse — extracting the value back out.

```csharp
int n = 42;          // value type on the stack
object o = n;        // BOXING: heap allocation
int back = (int)o;   // UNBOXING: copy back to stack

Console.WriteLine(o);        // 42 (boxed object)
Console.WriteLine(back);     // 42 (back to int)
```

Boxing is the bridge between the two halves of C#'s type system — value types and reference types. Sometimes it's necessary. Often it's an accidental performance leak.

---

## Why it exists

C# has a unified type hierarchy: every type derives from `object`. But value types live in places (stack, registers, struct fields) where references can't reach them. Boxing reconciles those worlds: any value can be "promoted" to a heap object that fits in the reference-type world.

You need boxing when:
- A value must satisfy a non-generic interface (e.g., `IComparable`).
- A value must be stored in a non-generic collection (`ArrayList`, `Hashtable` — legacy).
- A value must be assigned to `object` or `dynamic`.

Modern generics let you avoid boxing in most cases. That's a big reason `List<T>` replaced `ArrayList`.

---

## When boxing happens

These all box:

```csharp
int n = 42;

// 1. Assigning a value type to `object`
object o = n;

// 2. Assigning a value type to a (non-generic) interface
IComparable c = n;
IConvertible ic = n;

// 3. Adding to a non-generic collection
ArrayList list = new();
list.Add(n);   // boxes

// 4. String concatenation in some cases (older overloads)
string s = "n = " + n;   // doesn't box anymore with modern overloads, but used to

// 5. Using `string.Format` with object[] args
string s2 = string.Format("n = {0}", n);   // boxes
string s3 = $"n = {n}";                     // may not box — modern interpolated string handlers avoid it

// 6. Passing a value to a method expecting `object`
Console.WriteLine((object)n);   // boxes
Console.WriteLine(n);            // doesn't box — Console.WriteLine has int overload

// 7. Calling a non-generic Enum method
Severity sev = Severity.Error;
bool eq = sev.Equals((object)Severity.Warning);   // boxes both
```

Each boxing operation:
- Allocates ~24 bytes on the heap (16-byte header + value bytes, on 64-bit).
- Copies the value into the box.
- Returns the reference.

---

## When unboxing happens

```csharp
object o = 42;
int n = (int)o;        // unbox — type must match exactly
```

Rules:
- The cast must match **exactly** — `(long)o` when `o` is a boxed `int` throws `InvalidCastException`.
- Casting to a base type is fine via `object` chain but doesn't help with the value-type identity.
- Pattern matching is a safe alternative:

```csharp
if (o is int n2) Console.WriteLine(n2);  // safe, no exception
```

Unboxing in pattern matching internally does the same `unbox.any` IL operation but with the `isinst` check first — never throws.

---

## What boxing looks like in IL

```csharp
int n = 5;
object o = n;
int back = (int)o;
```

```il
ldc.i4.5
stloc.0            // n = 5

ldloc.0
box [System.Runtime]System.Int32     // BOX
stloc.1            // o = boxed 5

ldloc.1
unbox.any [System.Runtime]System.Int32   // UNBOX
stloc.2            // back = 5
```

The `box` instruction:
1. Calls the GC to allocate a heap object sized for `Int32` + header.
2. Sets the MT pointer to the boxed type's MT.
3. Copies the value into the new object's data slot.
4. Pushes the reference.

The `unbox.any` instruction:
1. Verifies the heap object's MT matches the target type.
2. Copies the value back out.
3. Pushes it as a value type.

There's an older `unbox` opcode (no `.any`) that returns a pointer into the box — used in some optimization scenarios. Modern code mostly sees `unbox.any`.

---

## The performance cost

Each box: ~24 bytes allocation + small copy. Doesn't sound like much. Now imagine in a loop:

```csharp
// 🚨 boxing in a hot loop
ArrayList list = new();
for (int i = 0; i < 1_000_000; i++) list.Add(i);
// 1 million boxes = ~24 MB of garbage
```

Compare to:
```csharp
List<int> list = new();
for (int i = 0; i < 1_000_000; i++) list.Add(i);
// No boxing — int stored inline in List<int>'s backing array
```

Generics changed the game. `List<T>` doesn't box T's value type because the compiler / JIT specializes the storage layout per T.

### Cost summary

- **Box**: heap allocation + memcpy. ~10-20 ns.
- **Unbox**: type check + memcpy. ~5-10 ns.
- **Loop allocation cost**: pressures Gen0 GC. Every ~100 KB of garbage triggers a collection.

---

## Boxing-prone patterns to avoid

### Interfaces on value types

```csharp
struct Counter : IComparable<Counter> {
    public int Value;
    public int CompareTo(Counter other) => Value - other.Value;
}

// Boxes:
IComparable<Counter> ic = new Counter();
ic.CompareTo(default);   // box happens at the cast above

// Doesn't box:
Counter c = new Counter();
c.CompareTo(default);    // direct call on the struct
```

A non-generic interface call on a value-type variable boxes. The generic form, with proper constraints, doesn't.

### `Enum.HasFlag` (pre-.NET Core 2.1)

```csharp
[Flags] enum Perms { Read=1, Write=2 }
Perms p = Perms.Read;
p.HasFlag(Perms.Read);   // pre-2.1: boxed both sides
```

Modern .NET intrinsifies this. But you'll see legacy code use `(p & Perms.Read) != 0` to avoid the historical boxing.

### Calling `Equals` on a struct

```csharp
struct Coord { public int X, Y; }
new Coord().Equals(new Coord());   // boxes if not implementing IEquatable<Coord>
```

The default `Equals(object)` on `ValueType` takes an `object` argument — the right side gets boxed. Implementing `IEquatable<Coord>` lets the compiler dispatch to a non-boxing overload.

```csharp
struct Coord : IEquatable<Coord> {
    public bool Equals(Coord other) => /* ... */;
    public override bool Equals(object? o) => o is Coord c && Equals(c);
}
```

### `string.Format` and `Console.WriteLine`

```csharp
int n = 42;
string s = string.Format("n = {0}", n);   // boxes n into the args array
Console.WriteLine("n = {0}", n);          // also boxes
```

Modern interpolated strings often skip the box:

```csharp
string s = $"n = {n}";   // C# 10+ interpolated string handler — no box for primitives
Console.WriteLine($"n = {n}");
```

The compiler can use the `DefaultInterpolatedStringHandler` (or `ILogger`'s custom handler in logging libraries), which calls `AppendFormatted(int)` directly — no boxing.

### Storing value types as `object`

```csharp
Dictionary<string, object> bag = new();
bag["count"] = 5;       // boxes 5 to object
int count = (int)bag["count"];   // unbox
```

For "config-like" stores, this is sometimes unavoidable. But on hot paths or with many entries, consider `Dictionary<string, int>` or a structured type.

---

## Special case: `Nullable<T>` boxing

Boxing a `Nullable<T>` has weird, helpful semantics:

```csharp
int? n = 5;
object o = n;                 // boxes the underlying int (NOT Nullable<int>)
Console.WriteLine(o.GetType()); // System.Int32

int? m = null;
object o2 = m;                // boxes to null (no allocation!)
Console.WriteLine(o2 is null);  // true
```

The runtime special-cases `Nullable<T>` boxing:
- `HasValue == true` → boxes the T (not the wrapper).
- `HasValue == false` → produces null (no allocation).

This makes `Nullable<T>` interoperate naturally with `object`-based APIs.

---

## Generic constraints avoid boxing

The most reliable way to dodge boxing in generic code is **constrained generic dispatch**:

```csharp
public static int Compare<T>(T a, T b) where T : IComparable<T> =>
    a.CompareTo(b);   // direct call via constraint, no box

Compare(3, 5);        // works for int, no boxing
Compare("a", "b");    // works for string
```

Without the constraint:

```csharp
public static int CompareNoConstraint(IComparable a, IComparable b) =>
    a.CompareTo(b);   // a and b are boxed when called with value types

CompareNoConstraint(3, 5);   // boxes both 3 and 5
```

For LINQ-style operations on lots of value-typed data, constraints + generic interfaces (`IEquatable<T>`, `IComparable<T>`) cut allocations dramatically.

---

## Internals — heap layout of a boxed value

A boxed value type instance on 64-bit:

```
offset 0:  sync block (8 bytes)
offset 8:  MT pointer (8 bytes) → points to MT of the boxed type
offset 16: value bytes
```

For a boxed `int`, total = 24 bytes (16 header + 4 int + 4 padding). For a boxed `decimal`, 32 bytes (16 + 16). For a boxed large struct, larger.

The MT pointer encodes "this is an Int32" — that's how `unbox.any` and `is int` checks work. They compare the MT pointer to the expected type's MT.

### `object.ReferenceEquals` and boxes

```csharp
int n = 5;
object a = n;
object b = n;
Console.WriteLine(ReferenceEquals(a, b));   // false — two separate boxes
```

Each box allocates a **new** object. Even boxing the same value twice gives different references.

Exception: small `int` boxes might be cached in some runtimes (Java does this; .NET historically doesn't). Don't rely on identity for boxes.

### Boxed enums

Enums are structs. Boxing an enum is the same cost as boxing an int. `(object)Severity.Error` allocates a boxed object whose MT identifies it as `Severity`.

```csharp
Severity s = Severity.Error;
object o = s;
o.Equals(Severity.Warning);   // boxes Severity.Warning too — comparison boxes both sides
```

A surprise source of allocation when comparing enums via `object`.

---

## Common patterns and best practices

### Use generic collections

```csharp
// 🚨 boxing
ArrayList list = new();   // legacy non-generic
list.Add(5);

// ✓ no boxing
List<int> list2 = new();
list2.Add(5);
```

`ArrayList`, `Hashtable`, `Queue` (non-generic), `Stack` (non-generic) all box value types. They exist for backward compat. Use `List<T>`, `Dictionary<K,V>`, `Queue<T>`, `Stack<T>`.

### Implement `IEquatable<T>` on structs

```csharp
public struct Money : IEquatable<Money> {
    public decimal Amount;
    public string Currency;

    public bool Equals(Money other) =>
        Amount == other.Amount && Currency == other.Currency;

    public override bool Equals(object? o) => o is Money m && Equals(m);
    public override int GetHashCode() => HashCode.Combine(Amount, Currency);
}
```

`HashSet<Money>` and `Dictionary<Money, T>` won't box on lookups.

### Avoid params object[] for primitives

```csharp
public void Log(params object[] args) { ... }   // boxes each int/double argument
public void Log<T>(params T[] args) { ... }    // doesn't box if T is value type
```

In .NET 8+ and especially C# 14 with `params ReadOnlySpan<T>`, the BCL has overloads that skip boxing.

### Watch for boxes in profiler

In dotMemory or PerfView, look for "boxing" allocations:
- `System.Int32` objects.
- `System.Double` objects.
- Custom value type allocations on the heap.

These are often unintentional. Trace back to find the conversion site.

---

## Common bugs

- **Boxing in tight loops** — millions of small allocations swamp Gen0.
- **`Equals(object?)` boxing the argument** — implement `IEquatable<T>` and the matching `Equals(object?)` override.
- **Storing value types in `Dictionary<TKey, object>`** — every access boxes/unboxes.
- **Using non-generic `IComparable`** in algorithms — switch to `IComparable<T>` with a generic constraint.
- **Casting an enum to `Enum` or `object`** to call methods like `HasFlag` (legacy concern).

---

## Performance summary

- A box: ~10-20 ns + heap allocation.
- An unbox: ~5-10 ns + type check.
- 1 million boxes ≈ 24 MB of garbage, ~10 ms in GC pressure.
- Generic constraints + `IEquatable<T>`/`IComparable<T>` eliminate most accidental boxing.

The general rule: **for hot paths, use generic APIs; let the JIT specialize per T.**

→ Next: [08-TypeConversions.md](08-TypeConversions.md)
