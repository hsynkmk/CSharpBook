# `dynamic` and ExpandoObject

## What `dynamic` is

`dynamic` is a static type that **bypasses compile-time type checking**. Calls on a `dynamic` are resolved at runtime via the Dynamic Language Runtime (DLR).

```csharp
dynamic d = "hello";
Console.WriteLine(d.Length);    // works — resolves at runtime
Console.WriteLine(d.Foo());     // compiles, throws RuntimeBinderException at run-time

d = 42;
Console.WriteLine(d + 1);       // resolves to int + int
```

Introduced in C# 4 (2010) primarily for **COM interop** and **dynamic languages** (Python, JavaScript via DLR). Modern C# rarely uses it.

---

## How `dynamic` works

Under the hood:
1. The compiler emits a `CallSite<T>` per dynamic expression.
2. The first call invokes the binder (`Microsoft.CSharp.RuntimeBinder.CSharpBinder`).
3. The binder uses reflection to resolve the call.
4. The binder generates an Expression Tree representing the call and compiles it.
5. The result is cached in the CallSite for subsequent calls.

The cache means **repeated dynamic calls on the same shape are fast** (~50-100 ns). First call: ~1-10 μs. Memory: ~KB per call site.

---

## When `dynamic` makes sense

Niche scenarios:

### 1. COM interop (Office, etc.)

```csharp
dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application")!);
excel.Workbooks.Add();
excel.Cells[1, 1].Value = "hello";
```

COM objects expose `IDispatch`. `dynamic` calls map to `IDispatch.Invoke`. Without `dynamic`, you'd write verbose `InvokeMember` calls.

### 2. Duck typing across non-related types

```csharp
public static int GetLength(dynamic obj) => obj.Length;

GetLength("hello");           // 5
GetLength(new[] { 1, 2, 3 }); // 3
GetLength(new List<int>{1,2}); // 2
```

Works if all candidates have a `Length` property/method. Avoids the `IHasLength` ceremony.

But: prefer generics with constraints. `dynamic` loses type safety.

### 3. Parsing JSON without strong types

```csharp
dynamic obj = JsonSerializer.Deserialize<dynamic>(json);
Console.WriteLine(obj.user.name);
```

Older JSON libraries leaned on `dynamic`. Modern `System.Text.Json` prefers `JsonNode` / `JsonElement` for the same use case — typed and discoverable.

---

## `ExpandoObject` — dynamic dictionary

```csharp
using System.Dynamic;

dynamic e = new ExpandoObject();
e.Name = "Alice";
e.Age = 30;
e.Greet = (Action)(() => Console.WriteLine("hi"));

Console.WriteLine(e.Name);   // Alice
e.Greet();                   // hi

// Also accessible as IDictionary<string, object>
var dict = (IDictionary<string, object?>)e;
dict["NewProp"] = "added dynamically";
Console.WriteLine(e.NewProp);
```

`ExpandoObject` is a class that lets you add/remove members at runtime. Implements `IDynamicMetaObjectObject` so dynamic dispatch routes through its `TryGetMember`/`TrySetMember`.

Internally, it's a hash map. Each property access is a dictionary lookup.

Used for:
- Building JSON-like data without defining types.
- Templating (Razor used this for ViewBag historically).
- Quick prototypes.

For most use cases, modern alternatives are better:
- `Dictionary<string, object?>` — explicit, no reflection.
- `JsonNode` / `JsonObject` — JSON-shaped.
- Records — when shape is known.

---

## DynamicObject — custom dynamic behavior

```csharp
public class CaseInsensitiveBag : DynamicObject {
    private readonly Dictionary<string, object?> _store = new(StringComparer.OrdinalIgnoreCase);

    public override bool TryGetMember(GetMemberBinder binder, out object? result) {
        return _store.TryGetValue(binder.Name, out result);
    }
    public override bool TrySetMember(SetMemberBinder binder, object? value) {
        _store[binder.Name] = value;
        return true;
    }
}

dynamic bag = new CaseInsensitiveBag();
bag.Name = "Alice";
Console.WriteLine(bag.NAME);   // Alice — case-insensitive
```

Subclass `DynamicObject` and override `TryGetMember` / `TrySetMember` / `TryInvokeMember`. Used for DSLs, mocking, ORM proxies.

---

## Performance characteristics

- First call: 1-10 μs (compile expression tree).
- Cached call: ~50-100 ns (lookup + invoke).
- Allocation: ~KB per call site (cached).

Compare:
- Static call: < 1 ns.
- Reflection `Invoke`: ~500 ns.

Dynamic is between reflection and static. The cache makes it competitive for repeated calls but slower than direct.

---

## Common bugs

### Loss of intellisense

```csharp
dynamic x = GetThing();
x.DoStuff();   // ⚠ no autocomplete; typos compile fine
```

Tools can't help you. Reading dynamic-heavy code is hard.

### Runtime errors instead of compile-time

```csharp
dynamic d = "hello";
d.FooBar();   // ✗ — RuntimeBinderException at runtime
```

Same code without `dynamic`: compile error. `dynamic` defers errors to runtime.

### `dynamic` in async

```csharp
async Task X() {
    dynamic d = GetTask();
    var r = await d;   // ⚠ — awaiter resolution is dynamic; subtle bugs possible
}
```

Mixing `dynamic` with `await` produces complex codegen. Avoid.

### Generic constraints don't work

```csharp
public void Process<T>(T x) where T : dynamic { ... }   // ✗ — no such constraint
```

Generics and dynamic are separate worlds. Use generics for typed polymorphism.

### AOT incompatibility

`dynamic` requires runtime codegen — incompatible with NativeAOT. The trimmer warns. Avoid `dynamic` in AOT projects.

---

## Modern alternatives

Most pre-2015 `dynamic` use cases now have better tools:

| Old `dynamic` use | Modern replacement |
|---|---|
| Untyped JSON | `JsonNode`, `JsonElement` |
| ViewBag in MVC | Strongly typed view models |
| Duck typing | Generics with `IFoo` constraint |
| Reflection-lite | `Type` + `MethodInfo` (or source gen) |
| COM interop | Still `dynamic` for Office; otherwise typed wrappers |
| Mocking | NSubstitute / Moq generate typed proxies via Castle |

If you reach for `dynamic`, ask whether one of these is cleaner.

---

## Summary

- `dynamic` defers type resolution to runtime via DLR.
- Useful for COM, ad-hoc duck typing, JSON without types.
- Cached call sites make it faster than reflection but slower than static.
- No compile-time safety; no intellisense; not AOT-friendly.
- Modern alternatives (JsonNode, generics, source gen) usually cleaner.
- `ExpandoObject` for runtime-extensible bags; prefer `Dictionary` or `JsonObject`.

→ Next: [05-SourceGenerators.md](05-SourceGenerators.md)
