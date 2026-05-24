# Value vs Reference Types

## What it is

C# divides every type into one of two camps:

- **Value types** — `int`, `double`, `bool`, structs, enums, tuples. The variable **holds the value directly**.
- **Reference types** — classes, interfaces, delegates, arrays, strings. The variable holds a **reference** (a pointer) to a value living elsewhere on the heap.

This split shapes how memory is allocated, how assignment works, how parameters are passed, and how equality behaves. It's the single most important distinction in the C# type system. Get it right and many "weird" behaviors stop being weird.

```csharp
int a = 5;            // value type — `a` IS 5
int b = a;            // copy the value: now both hold 5 independently
b = 10;
Console.WriteLine(a); // 5 — a unchanged

List<int> x = new() { 1, 2 };
List<int> y = x;      // reference type — both refer to SAME list
y.Add(3);
Console.WriteLine(x.Count); // 3 — x sees the change
```

---

## Why it exists

Two camps, two trade-offs:

- **Value types** are cheap: stack or inline allocation, no heap, no GC pressure. But they get copied on assignment — costly if they're big.
- **Reference types** are flexible: shared, mutable through any holder, nullable, support inheritance. But they live on the heap and the GC has to track them.

The language gives you both so you can pick the right tool. Small data with copy semantics → struct. Identity-bearing things with shared state → class.

---

## Where things live

### Stack and heap, simplified

```
┌─────────────────────────────────┐
│            STACK                │
│  • locals                       │
│  • method args                  │
│  • return addresses             │
│  • value-type instances         │
│  • references to heap objects   │
└─────────────────────────────────┘

┌─────────────────────────────────┐
│            HEAP                 │
│  • all reference-type objects   │
│  • boxed value types            │
│  • escaped value-type locals    │
│    (closures, async state)      │
└─────────────────────────────────┘
```

When you write:

```csharp
int n = 42;             // n is on the stack: 4 bytes containing 42
string s = "hello";     // s is on the stack: 8-byte pointer to a heap object
Person p = new("Alice", 30);  // p is on the stack: pointer to a heap object
```

The values `42`, the string `"hello"`, and the `Person` object are stored differently, but `n`, `s`, and `p` are all the same kind of thing on the stack: 4–8 bytes of data.

> **Caveat**: "on the stack" is a useful mental model, not always literally true. Value-type fields **inside** a class live on the heap (alongside the rest of the class's fields). Captured locals in lambdas can be hoisted to a heap-allocated closure class. The runtime's [escape analysis](../09-MemoryPerformance/11-EscapeAnalysis.md) (improved in .NET 10) can sometimes promote reference allocations onto the stack. The categorical statement "value types live on the stack" is a simplification — see Chapter 09 for the full story.

---

## Assignment

The defining behavior.

### Value types — copy

```csharp
int a = 5;
int b = a;  // copy the 4 bytes
b = 10;
Console.WriteLine(a); // 5
```

For structs, every field gets copied:

```csharp
struct Point { public int X, Y; }
Point p1 = new() { X = 1, Y = 2 };
Point p2 = p1;        // bitwise copy: 8 bytes total
p2.X = 99;
Console.WriteLine(p1.X); // 1
```

### Reference types — share

```csharp
class Box { public int Value; }
Box b1 = new() { Value = 5 };
Box b2 = b1;          // copy the reference (the pointer)
b2.Value = 10;
Console.WriteLine(b1.Value); // 10 — same object
```

The reference (an 8-byte pointer on 64-bit) is copied. Both `b1` and `b2` point to the **same heap object**.

---

## Parameter passing

C# is, by default, **pass-by-value** — for both kinds of types. But what's being passed differs:

### Value type passed by value

```csharp
void Increment(int n) { n++; }

int x = 5;
Increment(x);
Console.WriteLine(x); // 5 — caller's int unchanged
```

The int's bits are copied into the parameter. The method modifies its copy.

### Reference type passed by value

```csharp
void Clear(List<int> list) { list.Clear(); }

var nums = new List<int> { 1, 2, 3 };
Clear(nums);
Console.WriteLine(nums.Count); // 0 — same object got cleared
```

The **reference** is copied. Both the caller's variable and the parameter point at the **same** list. Mutating through one is visible through the other.

```csharp
void Reassign(List<int> list) { list = new List<int>(); }

var nums = new List<int> { 1, 2, 3 };
Reassign(nums);
Console.WriteLine(nums.Count); // 3 — caller unaffected
```

Reassigning the parameter only changes the local copy of the reference. The caller's pointer still points at the original list.

This is the #1 confusing thing about reference types for beginners. **Reread it**.

### `ref` to pass by reference

```csharp
void IncrementRef(ref int n) { n++; }

int x = 5;
IncrementRef(ref x);
Console.WriteLine(x); // 6 — caller's int incremented
```

`ref` makes the parameter an alias for the caller's variable, regardless of type. For value types it lets the method mutate the caller's value. For reference types it lets the method **reassign** the caller's variable to a new object.

[Chapter 01 §06](../01-Fundamentals/06-Methods.md) covers `ref`, `out`, `in` in detail.

---

## Equality

`==` behaves differently depending on the type:

```csharp
// Value types — compares value
int a = 5, b = 5;
Console.WriteLine(a == b); // true

// Reference types — compares reference, by default
class Box { public int X; }
var x = new Box { X = 5 };
var y = new Box { X = 5 };
Console.WriteLine(x == y); // false — different objects
Console.WriteLine(ReferenceEquals(x, y)); // false

// Override Equals + GetHashCode (or use records) for value equality
```

Two exceptions to the reference-equality rule for reference types:
- **`string`** — `==` compares contents.
- **`Nullable<T>`** — `==` compares contents.
- **Records** — synthesized `Equals` and `==` use value equality.
- **Custom operator overloads** — you can override `==` on any class to mean whatever you want.

Equality contract is covered deeply in [Chapter 07 §11](../07-Collections/11-EqualityContract.md).

---

## Nullability

| | Default state | Can be null? |
|---|---|---|
| Value type | zero-init (0, false, '\0') | No — `int n = null` doesn't compile |
| Reference type | `null` (NRT off) or non-null (NRT on) | Yes |
| `Nullable<T>` (T?) for structs | `null` | Yes (with `null` checks) |
| Nullable reference types (T? for classes) | `null` | Yes, compiler tracks state |

```csharp
int n;            // 0 — value type can't be null
int? n2 = null;   // Nullable<int> — yes
string s;         // null (in non-NRT context) or compile-error (in NRT)
string? s2;       // explicitly nullable reference
```

For value types, `int?` means `Nullable<int>` — a struct that wraps an int and a bool. For reference types, `string?` is just metadata for the compiler's null-state analysis; the runtime type is still `string`.

[Chapter 03 §06](06-NullableTypes.md) and [Chapter 10 §01](../10-AdvancedLanguage/01-NullableReferenceTypes.md) cover these.

---

## Inheritance

- **Reference types** support inheritance — `class Derived : Base`.
- **Value types** don't. Structs implicitly inherit from `System.ValueType` (which inherits from `Object`), but you can't have `struct B : A` for two of your structs.

Why? An inheritance chain means an object's actual type can differ from its declared type — that requires a runtime-type pointer (the method table). Reference types pay for that; value types don't, keeping them lean. Allowing struct inheritance would either inflate every struct with a method table pointer or break polymorphism — the language designers chose to disallow it.

Structs **can** implement interfaces, which gets you a different kind of polymorphism. See [Chapter 03 §02](02-Structs.md).

---

## Default values

```csharp
int n = default;            // 0
bool b = default;           // false
char c = default;           // '\0'
DateTime d = default;       // DateTime.MinValue
Point p = default;          // Point with all fields zero (struct)
string s = default;         // null (reference type)
List<int> l = default;      // null
int? x = default;           // null (Nullable<int> default)
```

Default for value types: all fields zero. Default for reference types: null. This is a critical safety property of the type system — uninitialized fields are predictable, not garbage.

---

## Internals — memory layout

Let's pin down what actually happens.

### A value type local

```csharp
void M() {
    int x = 5;
    double y = 3.14;
    DateTime d = DateTime.Now;
}
```

On the stack frame for `M`:

```
[ x: 4 bytes        ]
[ padding for align ]
[ y: 8 bytes        ]
[ d: 8 bytes        ]   ← DateTime is a struct holding one ulong
```

Total: about 24 bytes for the frame, allocated by a single subtraction from the stack pointer at method entry. Free.

### A reference-type local

```csharp
void M() {
    string s = "hello";
    Person p = new("Alice", 30);
}
```

On the stack:

```
[ s: 8 bytes — pointer to "hello" string object on heap ]
[ p: 8 bytes — pointer to Person object on heap        ]
```

On the heap:

```
"hello" string object:
[ sync block | MT ptr | length=5 | 'h' 'e' 'l' 'l' 'o' \0 ]

Person object:
[ sync block | MT ptr | name (8 bytes ref) | age (4 bytes) ]
```

Each heap object has a **16-byte header** on 64-bit .NET (8 bytes for the sync block, 8 bytes for the method-table pointer) followed by its fields.

### Where the JIT can be smarter

Modern .NET (especially .NET 10) sometimes:
- **Promotes structs into registers** instead of stack — the value never has a memory location.
- **Allocates "escaping" reference objects on the stack** if escape analysis proves they don't outlive the method.

These are JIT optimizations; the mental model "value on stack, reference on heap" remains the right starting point.

### IL difference

Loading a value-type local:

```il
ldloc.0    // push x's bits onto the eval stack
ldc.i4.5
add        // operate directly
stloc.0
```

Loading a reference-type local:

```il
ldloc.0    // push the reference (pointer) onto the eval stack
callvirt instance void System.Collections.Generic.List`1<int32>::Add(!0)
```

The reference flows through; the actual object stays on the heap.

### `string` looks like a value type — sort of

`string` is a reference type but feels value-like because:

- It's **immutable** — no one can mutate the heap object out from under you.
- `==` is overloaded for value comparison.
- String literals are **interned** — `"x" == "x"` is comparing two references that often point to the same heap object.

So `string` is the friendliest reference type to reason about. Other reference types behave more like the "share & mutate" model.

---

## Common patterns and pitfalls

### Returning a struct vs class

Returning a struct **copies** it into the caller's slot:

```csharp
public Point Move(int dx) {
    var p = new Point { X = X + dx, Y = Y };
    return p;   // copied to caller
}
```

Returning a class returns the **reference**:

```csharp
public List<int> Top10() {
    var list = new List<int>(); // ...
    return list;   // caller now shares the same list
}
```

Be careful returning internal collections — see [Chapter 02 §10 — Encapsulation](../02-OOP/10-Encapsulation.md).

### Mutating struct fields through methods doesn't always work

```csharp
struct MutablePoint { public int X, Y; }
class Container { public MutablePoint P; }

var c = new Container();
c.P.X = 5;
Console.WriteLine(c.P.X); // 5 — fine for a class field
```

But here's the trap:

```csharp
List<MutablePoint> list = new() { new() { X = 1 } };
list[0].X = 99;   // ❌ compile error — list[0] is a copy, can't be modified
```

Indexing a `List<T>` returns a copy of the struct. Same with `Dictionary[key]`. To mutate, replace the element:

```csharp
var temp = list[0];
temp.X = 99;
list[0] = temp;
```

This is the most-cited reason to **make structs immutable** when possible. `readonly struct` enforces it.

### Passing big structs

A 64-byte struct passed by value copies 64 bytes per call. If that happens in a hot loop, it adds up. Use `in` to pass by read-only reference:

```csharp
void Process(in BigStruct s) { /* read s */ }
```

The compiler treats `s` as an alias — no copy. The method can read but not write. See [Chapter 01 §06](../01-Fundamentals/06-Methods.md).

### "Should this be a class or a struct?"

The official Microsoft guideline:

Make it a **struct** when ALL of these are true:
- It logically represents a single value (like a primitive).
- Instances are small (typically ≤ 16 bytes).
- Immutable (or you control the mutation patterns).
- Won't be boxed frequently.

Otherwise: **class**.

Big or mutable structs cause more problems than they solve. A small immutable struct (`Point`, `Vector`, `Money`, `Duration`) is the sweet spot.

---

## Quick comparison cheat sheet

| | Value type (struct) | Reference type (class) |
|---|---|---|
| Stored | Stack or inline | Heap (mostly) |
| Variable holds | The value itself | A reference |
| Assignment | Copies value | Copies reference |
| Pass-by-value | Copy | Reference copy |
| `==` default | Value equality | Reference equality |
| Can be null | No (use `T?`) | Yes |
| Default value | All-zero fields | `null` |
| Inheritance | No (interfaces yes) | Yes |
| GC pressure | None (in normal cases) | Yes |
| Identity matters | No | Often |

---

## Common bugs

- **Modifying a copy and expecting the original to change** — struct fields exposed by methods (or `List<T>` indexers) return copies.
- **`==` on a custom class** — defaults to reference equality. Override or use records.
- **`null` on a reference type field** — defaults to `null` in old code or non-NRT contexts. Use NRT and initialize explicitly.
- **Reassigning a parameter inside a method** — doesn't affect the caller's variable, regardless of type.
- **Storing references to mutable internal collections** — callers can later mutate. Defensive copy or expose `IReadOnly...`.

---

## When to pick what

| You need | Pick |
|---|---|
| Identity-bearing entity | `class` |
| Inheritance / polymorphism | `class` |
| Value-like, small, immutable | `struct` (or `record struct`) |
| Need value equality, mostly immutable, no inheritance | `record` |
| Need value equality, mostly immutable, value-type semantics | `record struct` |
| Tagged data with closed set of cases | Hierarchy of classes or sealed records |
| Resource ownership (file, socket) | `class` with `IDisposable` |

→ Next: [02-Structs.md](02-Structs.md)
