# Activator — Dynamic Instantiation

## What it is

`System.Activator` creates instances of types **without knowing the type at compile time**. The standard entry point for "given a `Type`, give me an instance."

```csharp
Type t = typeof(List<int>);
object? instance = Activator.CreateInstance(t);             // new List<int>()
object? withCapacity = Activator.CreateInstance(t, 100);    // new List<int>(100)

List<int>? generic = Activator.CreateInstance<List<int>>();  // typed result
```

Used by:
- DI containers (`Microsoft.Extensions.DependencyInjection`).
- Serializers (deserializing unknown types).
- Plugin systems (instantiating types loaded from disk).
- Test frameworks (instantiating test classes).

---

## API surface

```csharp
// Non-generic — returns object?
public static object? CreateInstance(Type type);
public static object? CreateInstance(Type type, params object?[]? args);
public static object? CreateInstance(Type type, bool nonPublic);
public static object? CreateInstance(Type type, BindingFlags bindingAttr, ...);

// Generic — returns T (requires parameterless ctor)
public static T CreateInstance<T>();

// Per-assembly
public static object? CreateInstance(string assemblyName, string typeName);
```

The non-generic form does **constructor overload resolution** by matching `args` against the type's constructors.

```csharp
class Point {
    public Point() {}
    public Point(int x, int y) { X = x; Y = y; }
    public int X, Y;
}

var p1 = (Point)Activator.CreateInstance(typeof(Point))!;
var p2 = (Point)Activator.CreateInstance(typeof(Point), 1, 2)!;
```

---

## Generic `CreateInstance<T>()`

```csharp
T instance = Activator.CreateInstance<T>();
```

Constraint: `T` must have a **public parameterless constructor**. The compiler enforces this if `T : new()`.

```csharp
public T Make<T>() where T : new() => Activator.CreateInstance<T>();
```

For value types, `Activator.CreateInstance<T>()` returns `default(T)`. For reference types, calls the parameterless constructor.

### Performance (.NET 6+)

The JIT special-cases `Activator.CreateInstance<T>()` to inline directly to a `new T()` call when T is closed at JIT time. Cost: ~2-5 ns. As fast as `new`.

For the **non-generic** overload with args, the JIT can't help — it goes through reflection-style lookup. ~500 ns per call.

---

## Activator vs `new`

| Aspect | `new T()` | `Activator.CreateInstance<T>()` | `Activator.CreateInstance(type)` |
|---|---|---|---|
| Type known at compile time | ✓ | ✓ | ✗ |
| Speed | < 1 ns | ~2-5 ns | ~500 ns |
| Constructor args | ✓ direct | ✗ parameterless only | ✓ via args |
| AOT-friendly | ✓ | ✓ | partial |

Use `new` when you have a compile-time type. Use `Activator` when you have a `Type` object.

---

## Common patterns

### Plugin loading

```csharp
Assembly asm = Assembly.LoadFrom("plugin.dll");
foreach (var t in asm.GetTypes()) {
    if (typeof(IPlugin).IsAssignableFrom(t) && !t.IsAbstract) {
        var plugin = (IPlugin)Activator.CreateInstance(t)!;
        plugin.Run();
    }
}
```

### DI container internals

```csharp
public object Resolve(Type type) {
    var ctor = type.GetConstructors().First();
    var paramTypes = ctor.GetParameters();
    var args = paramTypes.Select(p => Resolve(p.ParameterType)).ToArray();
    return Activator.CreateInstance(type, args)!;
}
```

Real containers cache compiled factories (see [§07](07-PerformanceConcerns.md)) but the conceptual model is this.

### Cloning via reflection

```csharp
public static T Clone<T>(T source) where T : class {
    var copy = (T)Activator.CreateInstance(typeof(T))!;
    foreach (var pi in typeof(T).GetProperties().Where(p => p.CanWrite))
        pi.SetValue(copy, pi.GetValue(source));
    return copy;
}
```

Slow but generic shallow clone.

---

## Activator with private constructors

```csharp
class Singleton {
    private Singleton() {}
}

// Default Activator.CreateInstance throws — private ctor inaccessible
var ok = Activator.CreateInstance(typeof(Singleton), nonPublic: true);   // ✓
```

The `nonPublic: true` overload finds private/internal constructors. Useful for testing internal classes.

---

## Activator with value types

```csharp
struct Point3D { public int X, Y, Z; }
var p = (Point3D)Activator.CreateInstance(typeof(Point3D))!;   // boxed default
```

For value types, `Activator` returns a **boxed** instance (because the return type is `object`). Unbox with cast. The generic form `CreateInstance<T>()` skips boxing for value types.

---

## Common bugs

### Wrong argument types

```csharp
Activator.CreateInstance(typeof(Point), "1", "2");   // ✗ — MissingMethodException
```

Activator matches args by **exact type**, not assignability. `int.Parse("1")` first.

### Forgetting to handle null

```csharp
var instance = Activator.CreateInstance(type);
instance.Method();   // ⚠ — instance may be null
```

`CreateInstance` returns `object?`. Cast and null-check.

### Sealed types with no public ctor

```csharp
Activator.CreateInstance(typeof(string));   // ⚠ MissingMethodException
```

`string` has no parameterless ctor. Each type's constructors must be checked.

### AOT trimming

```csharp
[DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)]
Type type = ...;
var instance = Activator.CreateInstance(type);
```

Without the annotation, the trimmer may remove the constructor at publish time. NativeAOT requires explicit annotation or root descriptors.

---

## Alternatives

### `Expression.Compile` factories

```csharp
// Cache once
Func<MyType> factory = Expression.Lambda<Func<MyType>>(
    Expression.New(typeof(MyType))).Compile();

// Reuse — fast
var instance = factory();
```

After the one-time compile, each call is ~5-10 ns. Beats Activator's ~500 ns by 50-100×.

### `Reflection.Emit`

Generate IL directly. Lowest overhead but complex. Used by serializers (older STJ, Newtonsoft).

### Source generators

Generate the `new T()` call at compile time. Zero runtime cost. See [§05](05-SourceGenerators.md).

---

## Performance summary

| Approach | First call | Per call |
|---|---|---|
| `new T()` | < 1 ns | < 1 ns |
| `Activator.CreateInstance<T>()` (.NET 6+) | ~5 ns | ~5 ns |
| `Activator.CreateInstance(Type)` | ~500 ns | ~500 ns |
| `Activator.CreateInstance(Type, args)` | ~1000 ns | ~1000 ns |
| Cached `Expression.Compile` factory | ~50 ms compile | ~5 ns |
| Source-generated factory | 0 (compile time) | < 1 ns |

For one-off uses, Activator is fine. For repeated calls, cache a compiled delegate.

---

## When to use Activator

- One-time setup (DI container registration).
- Plugin instantiation (rare, slow path).
- Testing (instantiate types with private ctors).
- Prototypes.

When **not** to use:
- Hot paths with frequent instantiation — use compiled factories.
- AOT/trimmed apps — prefer source generators.

---

## Summary

- `Activator.CreateInstance(Type, args?)` instantiates via reflection — slow, dynamic.
- `Activator.CreateInstance<T>()` JIT-inlined since .NET 6 — fast, requires `new()` constraint.
- For repeated instantiation, cache compiled `Expression` factories.
- Be careful with non-public constructors, value types (boxing), and AOT trimming.

→ Next: [03-Attributes.md](03-Attributes.md)
