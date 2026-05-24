# Chapter 12 — Reflection — Q & A

---

### Q1. What is reflection?

The runtime API for inspecting and invoking code metadata — types, methods, properties, fields, attributes — without compile-time knowledge of them. Entry point is `System.Type`.

---

### Q2. Three ways to get a `Type`?

```csharp
typeof(string)                  // compile-time literal type
"x".GetType()                    // runtime type of an instance
Type.GetType("System.String")    // by assembly-qualified name string
```

---

### Q3. Difference between `typeof(T)` and `obj.GetType()`?

`typeof(T)` resolves the **declared/static** type at compile time, cheaply (token resolution). `obj.GetType()` reads the **runtime** type from the object header — the most-derived actual type.

```csharp
Animal a = new Dog();
typeof(Animal)   // Animal
a.GetType()      // Dog
```

`GetType()` on a value type boxes.

---

### Q4. What does `BindingFlags` control?

Which members `GetMethods`/`GetProperties`/etc. return: visibility (`Public`/`NonPublic`), instance vs static (`Instance`/`Static`), and whether to include inherited members (`DeclaredOnly`). Default is `Public | Instance`, so static and private members are excluded unless requested.

---

### Q5. How do you invoke a method via reflection?

```csharp
var mi = typeof(Math).GetMethod("Abs", [typeof(int)]);
int r = (int)mi!.Invoke(null, [-5])!;   // null target for static
```

`Invoke` boxes args (as `object[]`) and the return (`object?`). Slow (~150-500 ns).

---

### Q6. Open vs closed generic types?

`typeof(List<>)` is **open** (type parameter unspecified) — `IsGenericTypeDefinition`. `typeof(List<int>)` is **closed**. Construct a closed type from an open one with `MakeGenericType(typeof(int))`, and a closed method with `MakeGenericMethod`.

---

### Q7. Why is reflection slow?

Member lookup by string, boxing of value-type args/returns, `object[]` allocation, per-call security/visibility checks, and no JIT inlining. Costs ~100-500 ns/call vs < 1 ns direct.

---

### Q8. How do you make reflection fast?

1. **Cache** the `MemberInfo` (removes lookup cost).
2. **Compile to a typed delegate** via `Delegate.CreateDelegate` or `Expression.Compile`, cached — removes boxing, approaches direct-call speed.
3. For AOT/compile-time-known work, use **source generators**.

---

### Q9. What is `Activator.CreateInstance`?

Creates an instance from a `Type` object. `Activator.CreateInstance<T>()` is JIT-inlined since .NET 6 (~5 ns, needs `new()`). The non-generic `CreateInstance(Type, args)` does ctor overload resolution via reflection (~500 ns).

---

### Q10. How do you create an instance of a private-constructor type?

```csharp
Activator.CreateInstance(typeof(Singleton), nonPublic: true);
```

The `nonPublic: true` overload finds private/internal constructors.

---

### Q11. What is an attribute?

Declarative metadata attached to code elements. Defined as a class inheriting `System.Attribute`, marked with `[AttributeUsage]`, queried at runtime via reflection (`GetCustomAttribute`). Some are compiler-aware (`Obsolete`, `Conditional`, caller-info).

---

### Q12. What restricts attribute constructor arguments?

They must be **compile-time constants**: primitives, strings, `typeof(...)`, enums, or single-dimensional arrays thereof. They're encoded into assembly metadata at compile time.

---

### Q13. `IsDefined` vs `GetCustomAttribute`?

`IsDefined(type, inherit)` only checks **presence** (~50 ns, no instantiation). `GetCustomAttribute<T>()` instantiates the attribute object to read its data (~500 ns). Use `IsDefined` when you only need a yes/no.

---

### Q14. What does `[Conditional("DEBUG")]` do?

Removes **call sites** to the method when the symbol isn't defined. In Release (no `DEBUG`), `Log("x")` calls disappear entirely — including evaluation of arguments. Useful for debug-only logging.

---

### Q15. What is the `dynamic` keyword?

A static type that bypasses compile-time checking; member access resolves at runtime via the DLR. Useful for COM interop and duck typing. First call ~1-10 μs (compiles an expression tree, cached in a CallSite), then ~50-100 ns. Loses intellisense and compile-time safety; not AOT-compatible.

---

### Q16. `dynamic` vs reflection?

`dynamic` is syntactically natural (`d.Foo()`) and caches call sites for speed; reflection is explicit (`mi.Invoke`) and slower per call. `dynamic` is best for COM/duck typing; reflection for explicit metadata inspection. Both are slower than static or source-gen.

---

### Q17. What is a source generator?

A Roslyn compiler plugin that emits additional C# source at compile time. Replaces runtime reflection for performance + AOT/trimming compatibility. Use `IIncrementalGenerator` for cache-friendly, IDE-fast generation.

---

### Q18. Why prefer source generators over reflection?

Zero runtime cost (direct code), AOT-safe, trimming-safe, debuggable, no startup penalty. Reflection is slow per-call and incompatible with trimming/AOT without annotations. Examples: `[GeneratedRegex]`, STJ source-gen, `LoggerMessage`, `[LibraryImport]`.

---

### Q19. What target framework must source generators use?

`netstandard2.0` — they run inside the Roslyn compiler host process. Output must be deterministic (no `DateTime.Now`, random GUIDs).

---

### Q20. Why must you not pass `ISymbol`/`SyntaxNode` through an incremental generator pipeline?

They aren't value-equatable and hold the compilation alive, breaking Roslyn's caching (which compares stage outputs to skip unchanged work) and causing memory bloat / slow IDE. Extract the needed data into equatable records inside the `transform` step.

---

### Q21. What's the value of `nameof`?

A compile-time string of a symbol's **simple name**, refactor-safe (renames propagate or fail to compile). `nameof(Customer.Email)` → `"Email"`. Zero runtime cost.

---

### Q22. What are caller info attributes?

`[CallerMemberName]`, `[CallerFilePath]`, `[CallerLineNumber]`, `[CallerArgumentExpression]` — the compiler fills these optional parameters at the call site. Zero runtime reflection. Power INPC notifications and guard helpers like `ArgumentNullException.ThrowIfNull`.

---

### Q23. What does `[CallerArgumentExpression]` capture?

The **literal source text** of the named argument. `Require(user.Age >= 18)` can produce an error message containing `"user.Age >= 18"`. Impossible to get at runtime; only the compiler knows it.

---

### Q24. `Expression.Compile` vs `Reflection.Emit`?

Both JIT at runtime (not AOT-safe). `Expression.Compile` builds a typed tree and compiles to a delegate — readable, ~5-20 ns/call after compile. `Reflection.Emit` emits raw IL into a `DynamicMethod` — lowest overhead, hardest to write. Today `Expression` is nearly as fast and far more maintainable; Emit is rarely worth it.

---

### Q25. What reflection techniques are AOT-incompatible?

`Expression.Compile` and `Reflection.Emit` (runtime codegen — throw `PlatformNotSupportedException` under NativeAOT). Plain reflection works but needs `[DynamicallyAccessedMembers]` annotations for trimming. Source generators are the only fully AOT-safe metaprogramming option.

---

### Q26. When should you NOT use reflection?

Hot paths (millions of calls), AOT/trimmed apps without annotations, and anywhere a `Dictionary<string, Func<T>>` dispatch table or source generator is simpler. Reflection is for genuinely dynamic, infrequent work (plugin loading, one-time DI setup, test discovery).

---

→ Next: [Coding.md](Coding.md)
