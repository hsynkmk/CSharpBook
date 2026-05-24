# Reflection Basics

## What it is

Reflection is the runtime API for **inspecting and invoking code metadata** — types, methods, properties, fields, constructors, attributes — without knowing them at compile time.

```csharp
using System.Reflection;

Type t = typeof(string);
Console.WriteLine(t.FullName);                        // System.String
Console.WriteLine(t.IsClass);                          // True
Console.WriteLine(t.GetMethods().Length);              // ~70+ methods
foreach (var m in t.GetMethods(BindingFlags.Public | BindingFlags.Instance))
    Console.WriteLine($"  {m.Name}({m.GetParameters().Length})");
```

Reflection is how:
- Serializers (JSON, XML) discover properties on unknown types.
- ORMs (EF Core) map types to database columns.
- Test frameworks (xUnit, NUnit) discover `[Fact]`/`[Test]` methods.
- DI containers (Microsoft.Extensions.DI) inspect constructors to inject services.
- IDEs build intellisense for compiled assemblies.

It's the **dynamic** half of a "statically typed" language.

---

## Why it exists

Sometimes you don't know the type at compile time:
- A plugin loaded from a path.
- A type read from a config string.
- Generic helper code that works for any type.

Reflection lets one piece of code work polymorphically over arbitrary user-defined types. The cost: performance (slower than direct calls), and AOT/trimming friction.

---

## `Type` — the entry point

Every object has a runtime type, accessible three ways:

```csharp
Type t1 = typeof(string);              // compile-time literal type
Type t2 = "hello".GetType();            // runtime type of an instance
Type? t3 = Type.GetType("System.String, mscorlib");  // by name
```

`Type` represents a CLR type. It exposes:
- Metadata (`Name`, `FullName`, `Namespace`, `Assembly`).
- Structure (`IsClass`, `IsValueType`, `IsAbstract`, `IsSealed`, `IsGenericType`).
- Members (`GetMethods`, `GetProperties`, `GetFields`, `GetConstructors`, `GetMembers`).
- Inheritance (`BaseType`, `GetInterfaces`).
- Custom attributes (`GetCustomAttributes`).

```csharp
var t = typeof(List<int>);
Console.WriteLine(t.Name);                  // List`1
Console.WriteLine(t.IsGenericType);         // True
Console.WriteLine(t.GetGenericArguments()[0].Name);  // Int32
```

---

## `MemberInfo` hierarchy

```
MemberInfo
├── Type
├── MethodBase
│   ├── MethodInfo       — instance/static methods
│   └── ConstructorInfo  — constructors
├── PropertyInfo         — properties (get + set)
├── FieldInfo            — fields
├── EventInfo            — events
└── ParameterInfo (not strictly Member, but related)
```

Each gives you metadata + invocation primitives.

### `MethodInfo`

```csharp
var mi = typeof(Math).GetMethod("Abs", new[] { typeof(int) });
int absResult = (int)mi!.Invoke(null, [-5])!;   // 5
```

- `Invoke(target, args)` — call the method dynamically.
- For static methods, pass `null` as target.
- Args boxed into `object[]`; returns boxed `object?`.

### `PropertyInfo`

```csharp
var person = new { Name = "Alice", Age = 30 };
var pi = person.GetType().GetProperty("Name");
string name = (string)pi!.GetValue(person)!;
```

Reflection on properties uses `GetValue` / `SetValue`. Behind the scenes, calls the getter/setter methods.

### `FieldInfo`

```csharp
class Box { public int X; }
var b = new Box { X = 42 };
var fi = typeof(Box).GetField("X");
int x = (int)fi!.GetValue(b)!;   // 42
fi.SetValue(b, 99);
```

Field access is more direct than property access but uses `GetValue`/`SetValue` for reflection.

### `ConstructorInfo`

```csharp
var ci = typeof(List<int>).GetConstructor([typeof(int)]);
var list = (List<int>)ci!.Invoke([100]);   // new List<int>(100)
```

Or use `Activator.CreateInstance` (next file).

---

## `BindingFlags` — finding members

By default, `GetMethods()` returns public instance methods only. Use `BindingFlags` to control:

```csharp
var flags = BindingFlags.Public 
          | BindingFlags.NonPublic 
          | BindingFlags.Instance 
          | BindingFlags.Static
          | BindingFlags.DeclaredOnly;     // exclude inherited

var allMethods = typeof(MyClass).GetMethods(flags);
```

Common combinations:
- `Public | Instance` — usual public surface.
- `NonPublic | Instance` — private fields (test internal state, dangerous).
- `Public | Static` — static utility methods.
- `DeclaredOnly` — exclude inherited members (only this type's).

---

## Generic types and methods

### Open vs closed generics

```csharp
Type openList = typeof(List<>);             // open — type parameter unspecified
Type closedList = typeof(List<int>);         // closed — specific T
Console.WriteLine(openList.IsGenericTypeDefinition);   // True
Console.WriteLine(closedList.IsConstructedGenericType); // True
```

Close an open generic at runtime:

```csharp
Type intList = typeof(List<>).MakeGenericType(typeof(int));
var instance = Activator.CreateInstance(intList);  // new List<int>()
```

### Generic methods

```csharp
class Helper { public T Echo<T>(T x) => x; }

var openMethod = typeof(Helper).GetMethod("Echo");
var closedMethod = openMethod!.MakeGenericMethod(typeof(string));
var result = closedMethod.Invoke(new Helper(), ["hi"]);  // "hi"
```

`MakeGenericMethod` produces a callable closed generic.

---

## Assembly loading

```csharp
Assembly assembly = Assembly.Load("MyPlugin");
Assembly fromFile = Assembly.LoadFrom("plugin.dll");
var types = assembly.GetTypes();
```

`Assembly.Load` checks the load context; `LoadFrom` reads a specific file. Use `AssemblyLoadContext` for plugin scenarios (.NET Core+) — supports unloading.

---

## Performance characteristics

Reflection is **slow** compared to direct calls. Approximate costs:

| Operation | ~Time |
|---|---|
| Direct method call | < 1 ns |
| `MethodInfo.Invoke` | ~100-500 ns |
| `PropertyInfo.GetValue` | ~100-300 ns |
| `Activator.CreateInstance` | ~500 ns |
| Cached `Func<T>` via `Expression.Compile` | ~5-20 ns |
| Source-generated direct call | < 1 ns |

Reasons reflection is slow:
1. Hash-table lookup of member name.
2. Boxing of value-type arguments.
3. Allocation of `object[]` for args.
4. Security checks (visibility, etc.).
5. No JIT inlining.

For hot paths, **cache** `MethodInfo`/`PropertyInfo` and consider compiling to a delegate via `Expression` or `Reflection.Emit` — see [§07](07-PerformanceConcerns.md).

---

## AOT and trimming compatibility

Reflection over arbitrary types is **incompatible with trimming** by default — the trimmer doesn't know which types/members are used reflectively, so it might remove them.

Use:
- `[DynamicallyAccessedMembers(...)]` annotations to declare what you'll reflect on.
- `RequiresUnreferencedCode` / `RequiresDynamicCode` attributes to warn callers.
- **Source generators** to replace reflection at compile time (e.g., `System.Text.Json` source-gen).

For NativeAOT: `MakeGenericType` / `MakeGenericMethod` over types not pre-instantiated by the AOT compiler throw at runtime. Plan accordingly.

---

## Common patterns

### Discover all subclasses of a base type

```csharp
var derived = Assembly.GetExecutingAssembly()
    .GetTypes()
    .Where(t => typeof(BaseClass).IsAssignableFrom(t) && !t.IsAbstract);
```

Used by DI containers and test discovery.

### Copy fields from one object to another

```csharp
public static void CopyFields<T>(T src, T dst) {
    foreach (var fi in typeof(T).GetFields(BindingFlags.Public | BindingFlags.Instance))
        fi.SetValue(dst, fi.GetValue(src));
}
```

Pre-record times for shallow clones.

### Discover attributes

```csharp
var attr = typeof(MyType).GetCustomAttribute<DescriptionAttribute>();
if (attr is not null) Console.WriteLine(attr.Description);
```

Reflection's killer feature: metadata-driven behavior.

---

## Common bugs

### Forgetting to cast

```csharp
object result = mi.Invoke(null, args);
int x = result;   // ✗ — compile error
int y = (int)result!;  // ✓
```

`Invoke` returns `object?`. Cast back to expected type. Watch for `null`.

### Wrong BindingFlags

```csharp
typeof(MyClass).GetMethod("Calculate");        // null if static or non-public
typeof(MyClass).GetMethod("Calculate", BindingFlags.Static | BindingFlags.Public);
```

Default flags exclude static and non-public. Be explicit.

### Boxing value types

```csharp
fi.SetValue(b, 99);   // 99 boxed; allocation
```

For value types, reflection allocates on every call. Cache compiled delegates or use source generators.

### Reflecting on records

Records have synthetic members (`EqualityContract`, `<Clone>$`). When iterating properties, filter out compiler-generated:

```csharp
var props = typeof(MyRecord).GetProperties()
    .Where(p => p.Name != "EqualityContract");
```

---

## `Type.GetType` quirks

```csharp
Type.GetType("System.String");          // ✓ — mscorlib types found
Type.GetType("MyApp.MyClass");           // null — needs assembly name unless current assembly
Type.GetType("MyApp.MyClass, MyAssembly"); // ✓
```

Always provide the assembly-qualified name unless the type is in the current assembly or mscorlib.

---

## Internals — how reflection works

The CLR maintains rich metadata for every assembly (PE file). Reflection reads this metadata:
1. `Type` objects are lazily instantiated when requested.
2. `Type.GetMethods()` enumerates the type's method table.
3. `MethodInfo.Invoke`:
   - Verifies access (visibility).
   - Marshals args (boxes value types).
   - Calls through a thunk to the JIT-compiled method.
   - Boxes return value.

For value-type heavy code, reflection's overhead per call dominates. Source generators or `Expression.Compile` eliminate it by emitting direct IL.

---

## When to use reflection

- **Test discovery** (xUnit etc.).
- **Plugin loading** at runtime.
- **DI container setup** (one-time at startup).
- **Quick scripts** where perf doesn't matter.
- **ORM/serializer prototypes** (production: use source generators).

When **not** to use reflection:
- Hot paths (millions of calls/sec).
- AOT/trimmed apps without `DynamicallyAccessedMembers` annotations.
- Anywhere a `Dictionary<string, Func<T>>` or source generator is simpler.

---

## Summary

- Reflection inspects + invokes code metadata at runtime.
- Entry point: `Type` (via `typeof`, `GetType()`, `Type.GetType`).
- `MethodInfo`/`PropertyInfo`/`FieldInfo` for member-level operations.
- Use `BindingFlags` to control visibility/static-vs-instance.
- Costly per call (~100-500 ns) — cache and convert to delegates for hot code.
- Replaceable by source generators for ahead-of-time work.
- Watch trimming/AOT compatibility — annotate or avoid.

→ Next: [02-Activator.md](02-Activator.md)
