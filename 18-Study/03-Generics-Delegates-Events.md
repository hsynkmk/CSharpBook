# 03 — Generics, Delegates, Events

## ⚡ 30-second answer

**Generics** give type-safe, reusable code without boxing — `List<int>` stores ints directly, and the JIT specializes value-type generics. **Delegates** are type-safe function pointers (`Func`/`Action`/`Predicate`); **lambdas** are inline delegates that can **capture** variables (closures). **Events** are a publisher/subscriber wrapper over a delegate that only lets outsiders `+=`/`-=` (not invoke or overwrite). The big traps: **closure capture by reference** (especially in loops) and **event handler leaks** (subscriber kept alive by publisher).

---

## Core mechanics

**Generics + constraints**:
```csharp
T Max<T>(T a, T b) where T : IComparable<T> => a.CompareTo(b) >= 0 ? a : b;
```
Constraints: `where T : class` / `struct` / `new()` / `notnull` / `unmanaged` / `BaseType` / `IInterface` / `T : U`. `where T : class` allows null; `struct` forbids it.

**Variance** (only on interfaces/delegates, reference types):
- `out T` **covariant** — `IEnumerable<Derived>` is an `IEnumerable<Base>` (T only comes *out*).
- `in T` **contravariant** — `Action<Base>` is an `Action<Derived>` (T only goes *in*).

```csharp
IEnumerable<string> s = ...;
IEnumerable<object> o = s;   // covariance (out)
Action<object> ao = x => {};
Action<string> a = ao;       // contravariance (in)
```

**Delegates**:
```csharp
Func<int,int,int> add = (a,b) => a + b;     // takes args, returns
Action<string> log = Console.WriteLine;     // takes args, returns void
Predicate<int> isEven = n => n % 2 == 0;    // returns bool
```
Multicast: `+=` chains; invoking runs all; the return value is the *last* one.

**Closures**: a lambda captures *variables* (not values) — the compiler hoists them into a heap object so they outlive the method.

```csharp
int factor = 10;
Func<int,int> f = x => x * factor;   // captures 'factor' by reference
factor = 20;
f(5);                                 // 100 — sees the updated value
```

**Events**:
```csharp
public event EventHandler<OrderArgs>? OrderPlaced;     // delegate field + add/remove accessors
OrderPlaced?.Invoke(this, args);                        // only the declaring type can raise
// Subscribers: obj.OrderPlaced += Handler;  obj.OrderPlaced -= Handler;
```

---

## Comparison tables

| | Delegate | Event |
|---|---|---|
| Who can invoke | anyone holding it | only the declaring type |
| Who can assign (`=`) | anyone | only the declaring type |
| Outside access | full | `+=` / `-=` only |
| Use | callbacks, strategy, LINQ | pub/sub notifications |

| Variance | Keyword | Direction | Example |
|---|---|---|---|
| Covariant | `out` | output only | `IEnumerable<out T>`, `Func<out R>` |
| Contravariant | `in` | input only | `Action<in T>`, `IComparer<in T>` |
| Invariant | (none) | both | `List<T>`, `IList<T>` |

---

## 🪤 Traps & gotchas

- **Loop variable capture** — closures capture the *variable*. In `foreach` (C# 5+) each iteration has a fresh variable, so it's fine. But a **`for` loop** shares one `i`:
  ```csharp
  var fns = new List<Func<int>>();
  for (int i = 0; i < 3; i++) fns.Add(() => i);   // all return 3!
  // fix: capture a local copy:  int copy = i; fns.Add(() => copy);
  ```
- **Event handler leaks**: a publisher holds references to subscribers. If a short-lived subscriber `+=` to a long-lived publisher and never `-=`, the subscriber can't be GC'd → memory leak ([13-MemoryAndGC.md](13-MemoryAndGC.md)). Always unsubscribe (or use weak events).
- **Raising a null event** — use `Event?.Invoke(...)`; capturing to a local avoids a race where the last subscriber unsubscribes between the null-check and invoke.
- **Multicast return value** — only the last delegate's return is observed; the rest are lost. Don't rely on returns from multicast.
- **Generic constraints & boxing** — `where T : IComparable` (non-generic) can box value types; prefer `IComparable<T>`.
- **Variance only for reference types** — `IEnumerable<int>` is **not** an `IEnumerable<object>` (would require boxing).

---

## ❓ Likely questions

**Q: Why generics over `object`?**
A: Type safety (compile-time), no boxing for value types, and JIT specialization for performance.

**Q: Covariance vs contravariance?**
A: Covariant (`out`) lets you use a more-derived `T` where a base is expected (output positions). Contravariant (`in`) lets you use a more-base `T` (input positions). Reference types only.

**Q: Func vs Action vs Predicate?**
A: `Func<...,TResult>` returns a value; `Action<...>` returns void; `Predicate<T>` is `Func<T,bool>`.

**Q: What is a closure and what does it capture?**
A: A lambda + the captured variables (by reference, hoisted to a heap object). It captures *variables*, not snapshots — so later mutations are visible.

**Q: Delegate vs event?**
A: An event is a restricted delegate: outsiders can only subscribe/unsubscribe, not invoke or overwrite. Use events for pub/sub.

**Q: How do event handler memory leaks happen?**
A: The publisher's delegate references the subscriber. If the subscriber never unsubscribes and the publisher outlives it, the subscriber stays rooted → leak. Unsubscribe in Dispose / use weak events.

**Q: Classic loop-capture bug?**
A: A `for` loop's `() => i` all capture the same `i` and return its final value. Copy to a per-iteration local. (`foreach` is fine since C# 5.)

---

## 🎓 Senior Extra

- **JIT generic instantiation**: reference-type args share one code body (references are uniform pointers); each value-type arg gets a **specialized** body (monomorphization) → no boxing, but more JIT'd code. This is why `List<int>` is fast and box-free.
- **Delegate internals**: a delegate is an object with a `Target` (the captured `this`/closure, null for static) and a method pointer; multicast delegates hold an invocation list. Allocating delegates/closures in hot paths causes GC pressure — cache static lambdas, or use method groups; the compiler caches lambdas that capture nothing.
- **`static` lambdas** (C# 9) prevent accidental captures (compile error if you capture) → guarantees no closure allocation.
- **Weak events / `WeakEventManager`** (WPF) or manual `WeakReference` subscriptions solve handler leaks for long-lived publishers.
- **`unmanaged` constraint** + `Span<T>` enable low-level generic code; `where T : struct, Enum` enables generic enum helpers.
- **Generic math** (C# 11): `static abstract` interface members (`INumber<T>`, `IAdditionOperators<T,T,T>`) let you write `T Sum<T>(...) where T : INumber<T>` — arithmetic over any numeric type.
- **Expression trees** (`Expression<Func<...>>`): lambdas compiled to *data* (an AST) instead of IL — the basis of `IQueryable` / EF Core translating LINQ to SQL ([04-LINQ.md](04-LINQ.md)).

→ Deeper: [`../CSharpBook/04-Generics/`](../CSharpBook/04-Generics/README.md), [`../CSharpBook/05-DelegatesEvents/`](../CSharpBook/05-DelegatesEvents/README.md)
