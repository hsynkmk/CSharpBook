# Reflection Performance

## The cost model

Reflection trades speed for flexibility. Approximate per-operation costs:

| Operation | Time | vs direct |
|---|---|---|
| Direct method call | < 1 ns | 1× |
| `MethodInfo.Invoke` | ~150-500 ns | 200-500× |
| `PropertyInfo.GetValue` | ~100-300 ns | 200× |
| `FieldInfo.GetValue` | ~80-200 ns | 150× |
| `Activator.CreateInstance(Type)` | ~300-600 ns | 500× |
| `GetCustomAttribute<T>()` | ~300-500 ns | — |
| `Type.GetMethod("name")` | ~200-1000 ns | — |
| Cached `Func<>` (Expression.Compile) | ~5-20 ns | 5-20× |
| Source-generated direct call | < 1 ns | 1× |

Numbers vary by hardware and JIT. The key insight: reflection's overhead is **per-call**, dominated by lookup, boxing, and security checks.

---

## Why reflection is slow

1. **Member lookup** — `GetMethod("Foo")` searches the type's metadata by string. Hash + string compare.
2. **Boxing** — `Invoke(target, object[])` boxes value-type args and the return value. Heap allocation per call.
3. **Array allocation** — the `object[] args` array allocates.
4. **Security/visibility checks** — verified per invocation.
5. **No inlining** — the JIT can't inline through `Invoke`.

For a method called once at startup, none of this matters. For a method called a million times per second, it dominates.

---

## Rule 1: Cache the MemberInfo

The lookup (`GetMethod`, `GetProperty`) is often the most expensive part. Do it once.

```csharp
// ⚠ — looks up on every call
public object? GetName(object obj) =>
    obj.GetType().GetProperty("Name")!.GetValue(obj);

// ✓ — cache the PropertyInfo
private static readonly PropertyInfo NameProp =
    typeof(Customer).GetProperty("Name")!;

public object? GetName(Customer c) => NameProp.GetValue(c);
```

For polymorphic scenarios, cache per type:

```csharp
private static readonly ConcurrentDictionary<Type, PropertyInfo[]> PropCache = new();

PropertyInfo[] GetProps(Type t) =>
    PropCache.GetOrAdd(t, static x => x.GetProperties());
```

This alone removes most reflection overhead in real codebases.

---

## Rule 2: Compile to a delegate

After caching the `MemberInfo`, `Invoke`/`GetValue` still box and allocate. Convert to a typed delegate **once**, then call it directly.

### `Delegate.CreateDelegate` (cheapest, for methods)

```csharp
MethodInfo mi = typeof(Math).GetMethod(nameof(Math.Abs), [typeof(int)])!;
var abs = (Func<int, int>)Delegate.CreateDelegate(typeof(Func<int, int>), mi);

int r = abs(-5);   // ~1 ns — direct call, no boxing
```

Works when you know the delegate signature. Zero boxing.

### `Expression.Compile` (flexible)

Build an expression tree, compile to a delegate. Handles property access, constructors, conversions.

```csharp
// Compile a getter: (Customer c) => c.Name
static Func<object, object?> BuildGetter(PropertyInfo prop) {
    var instance = Expression.Parameter(typeof(object), "i");
    var cast = Expression.Convert(instance, prop.DeclaringType!);
    var access = Expression.Property(cast, prop);
    var box = Expression.Convert(access, typeof(object));
    return Expression.Lambda<Func<object, object?>>(box, instance).Compile();
}

var getName = BuildGetter(typeof(Customer).GetProperty("Name")!);
object? n = getName(customer);   // ~5-10 ns after one-time compile
```

The compile is expensive (~50 μs–1 ms) but happens once. Cache the resulting delegate.

```csharp
private static readonly ConcurrentDictionary<PropertyInfo, Func<object, object?>> GetterCache = new();
```

### Constructor factory

```csharp
static Func<object[], object> BuildFactory(ConstructorInfo ctor) {
    var args = Expression.Parameter(typeof(object[]), "args");
    var ps = ctor.GetParameters();
    var argExprs = ps.Select((p, i) =>
        Expression.Convert(
            Expression.ArrayIndex(args, Expression.Constant(i)),
            p.ParameterType));
    var body = Expression.New(ctor, argExprs);
    return Expression.Lambda<Func<object[], object>>(
        Expression.Convert(body, typeof(object)), args).Compile();
}
```

DI containers use exactly this pattern.

---

## Rule 3: Reflection.Emit (lowest-level)

`System.Reflection.Emit` lets you emit raw IL into a `DynamicMethod`. Maximum control, lowest overhead, hardest to write.

```csharp
var dm = new DynamicMethod("get_Name", typeof(object), [typeof(object)]);
var il = dm.GetILGenerator();
il.Emit(OpCodes.Ldarg_0);
il.Emit(OpCodes.Castclass, typeof(Customer));
il.Emit(OpCodes.Callvirt, typeof(Customer).GetProperty("Name")!.GetGetMethod()!);
il.Emit(OpCodes.Ret);
var getter = (Func<object, object?>)dm.CreateDelegate(typeof(Func<object, object?>));
```

Used by older serializers (Newtonsoft.Json, pre-source-gen STJ). Today, `Expression.Compile` is nearly as fast and far more maintainable. **Emit is rarely worth the complexity.**

Note: `Reflection.Emit` and `Expression.Compile` both **JIT at runtime** — incompatible with NativeAOT.

---

## Rule 4: Source generators (best for AOT)

For ahead-of-time work, source generators emit the direct code at compile time. Zero runtime reflection, AOT-safe, debuggable.

```csharp
// STJ source-gen — no reflection, no Expression.Compile
[JsonSerializable(typeof(Customer))]
public partial class Ctx : JsonSerializerContext {}

JsonSerializer.Serialize(c, Ctx.Default.Customer);
```

See [§05](05-SourceGenerators.md). For libraries shipping to AOT consumers, this is the only fully-compatible option.

---

## Decision tree

```
Is the work known at compile time?
├── YES → Source generator (AOT-safe, zero runtime cost)
│         or compile-time reflection (nameof/typeof) if trivial
└── NO (genuinely dynamic, e.g. plugins)
    ├── Called once / rarely?       → plain reflection (cache MemberInfo)
    ├── Called repeatedly, JIT OK?  → Expression.Compile, cached
    ├── Need absolute lowest cost?  → Delegate.CreateDelegate (methods)
    └── AOT target?                 → source generator (Expression/Emit won't work)
```

---

## Benchmark example

Reading a property 10 million times:

```csharp
// Direct:                    ~5 ms     (baseline)
// PropertyInfo.GetValue:     ~2500 ms  (500×)
// Cached PropertyInfo:       ~2400 ms  (caching lookup doesn't help GetValue's boxing)
// Expression.Compile getter: ~60 ms    (12× — boxing remains for object return)
// Typed delegate (no box):   ~6 ms     (~1× — essentially direct)
// Source-generated:          ~5 ms     (1×)
```

The lesson: caching the `MemberInfo` removes lookup cost but **not boxing**. To approach direct-call speed you must compile to a **typed** delegate (avoiding the `object` boxing on args/return) or use source generation.

Always measure with **BenchmarkDotNet** — micro-optimizing reflection without measurement is guesswork. See [Chapter 16 §07](../16-Testing/07-BenchmarkDotNet.md).

---

## Common bugs

### Caching but still boxing

```csharp
private static readonly PropertyInfo P = typeof(X).GetProperty("Val")!;
int v = (int)P.GetValue(obj)!;   // ⚠ — still boxes the int every call
```

Caching the `PropertyInfo` doesn't avoid boxing. Compile a `Func<X, int>` to eliminate it.

### Recompiling expressions per call

```csharp
// ⚠ — compiles a new delegate every call (catastrophic)
Func<object, object?> getter = BuildGetter(prop);
return getter(obj);
```

Compile once, cache the delegate. `Expression.Compile` is expensive.

### Holding compilations alive

Compiled delegates from `Expression`/`Emit` are JIT'd code — they live for the process. Caching unbounded numbers of them (e.g., per dynamic type) can leak memory. Bound the cache.

### Using reflection where a dictionary suffices

```csharp
// ⚠ — reflection to dispatch by name
var method = typeof(Handlers).GetMethod(command);
method!.Invoke(handlers, args);

// ✓ — a dictionary of delegates is simpler and faster
private static readonly Dictionary<string, Action<Args>> Map = new() {
    ["create"] = HandleCreate,
    ["delete"] = HandleDelete,
};
Map[command](args);
```

Often reflection is reached for when a simple dispatch table would do.

---

## AOT and trimming summary

| Technique | NativeAOT | Trimming |
|---|---|---|
| Plain reflection | ⚠ needs annotations | ⚠ needs `[DynamicallyAccessedMembers]` |
| `Delegate.CreateDelegate` | ⚠ if target reflected | ⚠ |
| `Expression.Compile` | ✗ (runtime codegen) | ✗ |
| `Reflection.Emit` | ✗ (runtime codegen) | ✗ |
| Source generators | ✓ | ✓ |

For AOT, source generators are the only fully-safe path. `Expression.Compile` throws `PlatformNotSupportedException` under NativeAOT.

---

## Summary

- Reflection costs ~100-500 ns/call vs < 1 ns direct — fine once, deadly in hot loops.
- **Cache** `MemberInfo` to remove lookup cost.
- **Compile to a typed delegate** (`Delegate.CreateDelegate` or `Expression.Compile`) to remove boxing — approaches direct-call speed.
- **`Reflection.Emit`** is the lowest level but rarely worth it over `Expression`.
- **Source generators** are the best option for compile-time-known work and the only AOT-safe one.
- Always benchmark; prefer a dispatch dictionary over reflection when the set is known.

→ Next: [Questions.md](Questions.md)
