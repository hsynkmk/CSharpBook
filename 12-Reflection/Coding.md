# Chapter 12 — Reflection — Coding Problems

Theme: build a JSON serializer three ways — reflection (naive), cached delegates (fast), source generator (AOT). Plus assorted reflection exercises.

---

## Problem 1: List all public properties of a type

Write a method that prints each public instance property name and its current value for any object.

<details><summary>Solution</summary>

```csharp
public static void Dump(object obj) {
    var t = obj.GetType();
    foreach (var p in t.GetProperties(BindingFlags.Public | BindingFlags.Instance)) {
        if (!p.CanRead || p.GetIndexParameters().Length > 0) continue;  // skip indexers
        Console.WriteLine($"{p.Name} = {p.GetValue(obj)}");
    }
}
```

Skip indexers (`GetIndexParameters().Length > 0`) — they require arguments.

</details>

---

## Problem 2: Invoke a method by name

Given an object, a method name, and args, call the method dynamically and return the result.

<details><summary>Solution</summary>

```csharp
public static object? Call(object target, string methodName, params object[] args) {
    var types = args.Select(a => a.GetType()).ToArray();
    var mi = target.GetType().GetMethod(methodName, types)
        ?? throw new MissingMethodException(target.GetType().Name, methodName);
    return mi.Invoke(target, args);
}

// Usage
var sb = new StringBuilder();
Call(sb, "Append", "hello");
Console.WriteLine(sb);   // hello
```

Bug to avoid: `GetMethod(name)` without types throws `AmbiguousMatchException` if overloaded. Pass the arg types.

</details>

---

## Problem 3: Generic factory with constraint

Write `Create<T>()` that returns a new T, requiring a parameterless constructor at compile time.

<details><summary>Solution</summary>

```csharp
public static T Create<T>() where T : new() => new T();   // best — compiler enforced

// If you only have a Type at runtime:
public static object CreateFromType(Type t) =>
    Activator.CreateInstance(t)
        ?? throw new InvalidOperationException($"Cannot create {t}");
```

The `new()` constraint version is preferred — no reflection, JIT-inlined. Only use `Activator` when the type isn't known at compile time.

</details>

---

## Problem 4: Naive reflection-based JSON serializer

Serialize an object's public properties to a flat JSON string using reflection. (Strings, numbers, bools only.)

<details><summary>Solution</summary>

```csharp
public static string ToJson(object obj) {
    var sb = new StringBuilder("{");
    var props = obj.GetType().GetProperties(BindingFlags.Public | BindingFlags.Instance);
    for (int i = 0; i < props.Length; i++) {
        var p = props[i];
        if (i > 0) sb.Append(',');
        sb.Append('"').Append(p.Name).Append("\":");
        var value = p.GetValue(obj);
        sb.Append(value switch {
            null      => "null",
            string s  => $"\"{s}\"",
            bool b    => b ? "true" : "false",
            _         => value.ToString()
        });
    }
    return sb.Append('}').ToString();
}

// Usage
record Person(string Name, int Age, bool Active);
Console.WriteLine(ToJson(new Person("Alice", 30, true)));
// {"Name":"Alice","Age":30,"Active":true,"EqualityContract":...}
```

Bug: records expose `EqualityContract`. Filter it: `.Where(p => p.Name != "EqualityContract")`. Cost: `GetProperties` + `GetValue` per call, both reflective and slow.

</details>

---

## Problem 5: Speed up the serializer with cached compiled getters

Make Problem 4's serializer fast by caching compiled `Func<object, object?>` getters per type.

<details><summary>Solution</summary>

```csharp
public static class FastJson {
    private static readonly ConcurrentDictionary<Type, (string Name, Func<object, object?> Get)[]> Cache = new();

    private static (string, Func<object, object?>)[] BuildAccessors(Type t) =>
        t.GetProperties(BindingFlags.Public | BindingFlags.Instance)
         .Where(p => p.CanRead && p.Name != "EqualityContract")
         .Select(p => (p.Name, BuildGetter(p)))
         .ToArray();

    private static Func<object, object?> BuildGetter(PropertyInfo p) {
        var inst = Expression.Parameter(typeof(object), "o");
        var typed = Expression.Convert(inst, p.DeclaringType!);
        var access = Expression.Property(typed, p);
        var boxed = Expression.Convert(access, typeof(object));
        return Expression.Lambda<Func<object, object?>>(boxed, inst).Compile();
    }

    public static string ToJson(object obj) {
        var accessors = Cache.GetOrAdd(obj.GetType(), BuildAccessors);
        var sb = new StringBuilder("{");
        for (int i = 0; i < accessors.Length; i++) {
            var (name, get) = accessors[i];
            if (i > 0) sb.Append(',');
            sb.Append('"').Append(name).Append("\":");
            sb.Append(Format(get(obj)));
        }
        return sb.Append('}').ToString();
    }

    private static string Format(object? v) => v switch {
        null => "null", string s => $"\"{s}\"", bool b => b ? "true" : "false", _ => v.ToString()!
    };
}
```

The `Expression.Compile` happens once per type (cached). Subsequent serializations skip reflection — only the compiled delegate runs. ~10-50× faster than Problem 4 in a loop.

Caveat: not AOT-safe (`Expression.Compile` needs JIT). For AOT, see Problem 6.

</details>

---

## Problem 6: The same, but as a source generator (conceptual)

Sketch a source generator that emits a `ToJson` method per `[JsonSerialize]`-marked type. Why is it superior for AOT?

<details><summary>Solution</summary>

```csharp
// User code:
[JsonSerialize]
public partial record Person(string Name, int Age, bool Active);

// Generator emits:
partial record Person {
    public string ToJson() =>
        $"{{\"Name\":{Quote(Name)},\"Age\":{Age},\"Active\":{(Active ? "true" : "false")}}}";
    private static string Quote(string s) => $"\"{s}\"";
}
```

Generator skeleton:

```csharp
[Generator]
public class JsonGen : IIncrementalGenerator {
    public void Initialize(IncrementalGeneratorInitializationContext ctx) {
        var types = ctx.SyntaxProvider.ForAttributeWithMetadataName(
            "MyLib.JsonSerializeAttribute",
            predicate: static (_, _) => true,
            transform: static (c, _) => {
                var sym = (INamedTypeSymbol)c.TargetSymbol;
                var props = sym.GetMembers().OfType<IPropertySymbol>()
                    .Where(p => p.DeclaredAccessibility == Accessibility.Public && p.Name != "EqualityContract")
                    .Select(p => (p.Name, p.Type.SpecialType))
                    .ToImmutableArray();
                return new Model(sym.ContainingNamespace.ToDisplayString(), sym.Name, props);
            });
        ctx.RegisterSourceOutput(types, static (spc, m) => spc.AddSource($"{m.Name}.json.g.cs", Emit(m)));
    }
    // Emit(...) builds the partial method string. Model is an equatable record.
}
```

**Why superior for AOT**: the `ToJson` is plain compiled C# — no reflection, no `Expression.Compile`, no JIT codegen at runtime. Works under NativeAOT and trimming. Zero startup cost, debuggable. This is exactly why `System.Text.Json` ships a source generator.

</details>

---

## Problem 7: Copy matching properties between two types

Write `Map<TSrc, TDst>(src)` that creates a `TDst` and copies properties with matching names and types.

<details><summary>Solution</summary>

```csharp
public static TDst Map<TSrc, TDst>(TSrc src) where TDst : new() {
    var dst = new TDst();
    var srcProps = typeof(TSrc).GetProperties().Where(p => p.CanRead);
    var dstProps = typeof(TDst).GetProperties()
        .Where(p => p.CanWrite)
        .ToDictionary(p => p.Name);

    foreach (var sp in srcProps) {
        if (dstProps.TryGetValue(sp.Name, out var dp) && dp.PropertyType.IsAssignableFrom(sp.PropertyType))
            dp.SetValue(dst, sp.GetValue(src));
    }
    return dst;
}
```

This is a mini-AutoMapper. Production mappers cache compiled delegates per (TSrc, TDst) pair. Note: AutoMapper-style runtime mapping is being phased out in favor of source-generated mappers (Mapperly) for AOT.

</details>

---

## Problem 8: Find all types implementing an interface

In the current assembly, find all concrete (non-abstract) types implementing `IPlugin`.

<details><summary>Solution</summary>

```csharp
public static IEnumerable<Type> FindPlugins() =>
    Assembly.GetExecutingAssembly()
        .GetTypes()
        .Where(t => typeof(IPlugin).IsAssignableFrom(t)
                 && !t.IsAbstract
                 && !t.IsInterface);

// Instantiate them
foreach (var t in FindPlugins()) {
    var plugin = (IPlugin)Activator.CreateInstance(t)!;
    plugin.Run();
}
```

`IsAssignableFrom` checks the inheritance/implementation relationship. Exclude abstract types and the interface itself. Note: `GetTypes()` can throw `ReflectionTypeLoadException` if dependencies are missing — wrap in try/catch for plugin scenarios.

</details>

---

## Problem 9: Read a custom attribute's data

Define a `[DisplayName("...")]` attribute, apply it to enum members, and write a method to read the display name for a given enum value.

<details><summary>Solution</summary>

```csharp
[AttributeUsage(AttributeTargets.Field)]
public class DisplayNameAttribute(string name) : Attribute {
    public string Name { get; } = name;
}

public enum Status {
    [DisplayName("In Progress")] Active,
    [DisplayName("Completed")]   Done,
    Cancelled  // no attribute
}

public static string GetDisplayName(Enum value) {
    var field = value.GetType().GetField(value.ToString())!;
    var attr = field.GetCustomAttribute<DisplayNameAttribute>();
    return attr?.Name ?? value.ToString();
}

// Usage
Console.WriteLine(GetDisplayName(Status.Active));     // In Progress
Console.WriteLine(GetDisplayName(Status.Cancelled));  // Cancelled
```

The pattern behind enum→display-string mapping in UI frameworks. Cache the field lookups for hot paths.

</details>

---

## Problem 10: Generic method invocation

Given a generic method `Repository.GetAll<T>()`, invoke it with a runtime `Type`.

<details><summary>Solution</summary>

```csharp
class Repository {
    public List<T> GetAll<T>() => new();
}

public static object InvokeGetAll(Repository repo, Type entityType) {
    var openMethod = typeof(Repository).GetMethod(nameof(Repository.GetAll))!;
    var closedMethod = openMethod.MakeGenericMethod(entityType);
    return closedMethod.Invoke(repo, null)!;
}

// Usage
var result = InvokeGetAll(new Repository(), typeof(Customer));  // List<Customer>
```

`MakeGenericMethod` closes the open generic. Caveat: under NativeAOT, `MakeGenericMethod` over types not seen at compile time throws — needs source-gen or pre-instantiation.

</details>

---

## Problem 11: Build a compiled constructor factory

Cache and compile a `Func<object>` factory for a parameterless constructor. Compare its speed to `Activator.CreateInstance`.

<details><summary>Solution</summary>

```csharp
public static class Factory {
    private static readonly ConcurrentDictionary<Type, Func<object>> Cache = new();

    public static object Create(Type t) =>
        Cache.GetOrAdd(t, static type => {
            var ctor = type.GetConstructor(Type.EmptyTypes)
                ?? throw new InvalidOperationException($"{type} has no parameterless ctor");
            return Expression.Lambda<Func<object>>(Expression.New(ctor)).Compile();
        })();
}
```

Benchmark (creating 10M instances):
- `Activator.CreateInstance(type)`: ~5000 ms
- Cached compiled factory: ~50 ms (~100× faster)

The first call to `Create(t)` pays the compile cost (~50 μs); all subsequent calls reuse the cached delegate. This is the core trick in fast DI containers.

</details>

---

## Problem 12: Refactor reflection-based dispatch to a delegate table

This command handler uses reflection per call. Make it fast and AOT-safe.

```csharp
public void Handle(string command, string[] args) {
    var mi = GetType().GetMethod("Handle" + command);
    mi!.Invoke(this, [args]);
}
```

<details><summary>Solution</summary>

```csharp
public class Dispatcher {
    private readonly Dictionary<string, Action<string[]>> _handlers;

    public Dispatcher() {
        _handlers = new(StringComparer.OrdinalIgnoreCase) {
            ["create"] = HandleCreate,
            ["delete"] = HandleDelete,
            ["update"] = HandleUpdate,
        };
    }

    public void Handle(string command, string[] args) {
        if (_handlers.TryGetValue(command, out var handler))
            handler(args);
        else
            throw new InvalidOperationException($"Unknown command: {command}");
    }

    private void HandleCreate(string[] args) { /* ... */ }
    private void HandleDelete(string[] args) { /* ... */ }
    private void HandleUpdate(string[] args) { /* ... */ }
}
```

Direct delegate dispatch (~1 ns) vs reflection per call (~500 ns). AOT-safe (no reflection). Clearer too — the dispatch table is explicit. This is the lesson: **reflection is often reached for when a dictionary of delegates is simpler and faster.**

</details>

---

These problems trace the arc from naive reflection → cached delegates → source generation — the same evolution the .NET BCL itself went through (Newtonsoft → STJ reflection → STJ source-gen).

→ Back to [Chapter 12 README](README.md). Next chapter: [Chapter 13 — I/O & Serialization](../13-IO/README.md).
