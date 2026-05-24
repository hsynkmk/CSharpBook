# Trimming

## What it is

Trimming (the IL Linker) **removes unused code** from your app and its dependencies at publish time, shrinking the output. It analyzes which types and members are reachable from your entry point and discards the rest.

```xml
<PropertyGroup>
  <PublishTrimmed>true</PublishTrimmed>
</PropertyGroup>
```

```bash
dotnet publish -r linux-x64 -c Release   # trimmed, smaller self-contained output
```

Trimming is a prerequisite for Native AOT (AOT always trims) and useful on its own for self-contained deployments — it can cut a ~70 MB self-contained app to ~25 MB or less.

---

## Why it's hard — the reflection problem

The trimmer uses **static analysis** to find reachable code. But **reflection breaks static analysis**: if you reflect on a type by a runtime string, the trimmer can't see that the type is used, so it removes it.

```csharp
// Trimmer sees no static reference to "MyApp.Plugin" → removes it
Type? t = Type.GetType("MyApp.Plugin");     // ⚠ returns null after trimming
var obj = Activator.CreateInstance(t!);     // NullReferenceException
```

The type was trimmed because nothing *statically* referenced it. This is the central trimming challenge: code that works in a normal build silently fails when trimmed.

---

## Trim warnings (IL2xxx)

The trimmer emits warnings when it detects patterns it can't analyze:

```
IL2026: Using 'JsonSerializer.Serialize<T>' which has 'RequiresUnreferencedCodeAttribute'.
        Members might be removed.
IL2057: Unrecognized value passed to Type.GetType. Type may not be preserved.
IL2075: 'this' argument does not satisfy 'DynamicallyAccessedMembers'.
```

**Treat warnings as errors that predict runtime failures.** A trim-warning-free app is much more likely to work trimmed/AOT.

```xml
<TreatWarningsAsErrors>true</TreatWarningsAsErrors>
<!-- or specifically -->
<WarningsAsErrors>$(WarningsAsErrors);IL2026;IL2057;IL2075</WarningsAsErrors>
```

---

## `DynamicallyAccessedMembers` — telling the trimmer what to keep

When you reflect on a type, annotate the parameter/field so the trimmer preserves the needed members:

```csharp
using System.Diagnostics.CodeAnalysis;

public object Create(
    [DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)]
    Type type)
{
    return Activator.CreateInstance(type)!;   // trimmer now keeps public ctors of any type flowing here
}
```

`DynamicallyAccessedMemberTypes` flags what to preserve:
- `PublicConstructors`, `PublicMethods`, `PublicProperties`, `PublicFields`
- `NonPublicConstructors`, etc.
- `All` (keep everything — heavy)

The annotation **flows** through your code: if you pass a `Type` to this method, the trimmer ensures whatever type reaches it keeps its public constructors.

---

## `RequiresUnreferencedCode` — propagating the warning

If your method fundamentally relies on reflection that trimming can break, mark it so callers are warned (instead of silently breaking):

```csharp
[RequiresUnreferencedCode("Uses reflection to serialize arbitrary types. Use the source-generated overload for trimming.")]
public string Serialize<T>(T value) {
    // reflection-based serialization
}
```

Callers get `IL2026` pointing them to the safe alternative. This is how the BCL marks reflection-based `JsonSerializer.Serialize<T>` (without a `JsonTypeInfo`).

Similarly, `[RequiresDynamicCode]` marks methods needing runtime codegen (AOT-incompatible).

---

## Making libraries trim-compatible

For library authors, opt into trim analysis so consumers get accurate warnings:

```xml
<PropertyGroup>
  <IsTrimmable>true</IsTrimmable>          <!-- marks assembly trimmable + enables analyzer -->
  <IsAotCompatible>true</IsAotCompatible>  <!-- also enables AOT analyzer (implies IsTrimmable) -->
  <EnableTrimAnalyzer>true</EnableTrimAnalyzer>
</PropertyGroup>
```

`IsTrimmable` declares "this assembly is safe to trim" and turns on the analyzer during the library's own build, so you catch issues before shipping. A trim-clean library is a good citizen in AOT apps.

---

## Preserving code that's reflected on

When you genuinely need reflection and can't annotate (e.g., types only known at runtime), preserve them explicitly.

### Via `DynamicDependency`

```csharp
[DynamicDependency(DynamicallyAccessedMemberTypes.PublicProperties, typeof(MyModel))]
public void Configure() {
    // Trimmer keeps MyModel's public properties because of the attribute above
}
```

### Via a trimmer XML descriptor

```xml
<!-- ILLink.Descriptors.xml -->
<linker>
  <assembly fullname="MyAssembly">
    <type fullname="MyApp.Plugin" preserve="all" />
  </assembly>
</linker>
```

```xml
<ItemGroup>
  <TrimmerRootDescriptor Include="ILLink.Descriptors.xml" />
</ItemGroup>
```

This is the escape hatch for "the trimmer can't see it but I know it's used."

### Via `TrimmerRootAssembly`

```xml
<ItemGroup>
  <TrimmerRootAssembly Include="MyPlugins" />   <!-- keep this whole assembly -->
</ItemGroup>
```

---

## Trim granularity

```xml
<TrimMode>full</TrimMode>     <!-- trim everything not statically reachable (default for AOT) -->
<TrimMode>partial</TrimMode>  <!-- only trim assemblies marked IsTrimmable -->
```

- **`partial`** (conservative) — trims only assemblies that opted in. Safer; less reduction.
- **`full`** — trims everything. Maximum reduction; more likely to break reflection-dependent code.

Native AOT always uses full trimming.

---

## The modern AOT/trim-friendly stack

To build a trim-clean app, replace reflection-based components with source-generated ones:

| Reflection-based (trim-unsafe) | Source-generated (trim-safe) |
|---|---|
| `JsonSerializer.Serialize<T>(obj)` | `JsonSerializer.Serialize(obj, ctx.T)` (source-gen) |
| `new Regex(pattern)` | `[GeneratedRegex]` |
| `ILogger.LogInformation($"...")` | `[LoggerMessage]` source-gen |
| AutoMapper | Mapperly (source-gen) |
| Reflection-based config binding | `[GeneratedBindingExtensions]` / source-gen binding |
| Reflection-based DI scanning | explicit registration |

---

## Common bugs

### Silent null from trimmed Type.GetType

Covered above. Annotate or preserve the type, or avoid string-based reflection.

### Missing members at runtime

```csharp
// Trimmed: property setters removed because only getters seemed used
var prop = typeof(Model).GetProperty("Name");
prop!.SetValue(obj, "x");   // ⚠ — setter trimmed → throws or no-op
```

Annotate with `DynamicallyAccessedMemberTypes.PublicProperties` (includes accessors).

### Generic instantiation trimmed

A generic type used only via reflection may have its instantiation trimmed. Reference it statically or preserve it.

### Ignoring warnings

The single biggest mistake: publishing trimmed/AOT with `IL2xxx` warnings unaddressed, then hitting runtime failures in production. Fix warnings during development.

---

## Workflow

1. Enable analyzers early: `<IsAotCompatible>true</IsAotCompatible>` (libraries) or `<PublishAot>true</PublishAot>` (apps).
2. Build and fix all `IL2xxx`/`IL3xxx` warnings.
3. Replace reflection with source generators where flagged.
4. For unavoidable reflection, annotate (`DynamicallyAccessedMembers`) or preserve (`DynamicDependency`, descriptors).
5. Test the trimmed/AOT build thoroughly — warnings predict but don't catch everything.

---

## Summary

- Trimming removes statically-unreachable code to shrink output; required for Native AOT.
- Reflection defeats static analysis — trimmed types/members vanish, causing runtime failures.
- `IL2xxx` trim warnings predict these failures — treat them as errors and fix them.
- Annotate reflection entry points with `[DynamicallyAccessedMembers]`; propagate with `[RequiresUnreferencedCode]`.
- Preserve unavoidable reflection via `DynamicDependency`, trimmer descriptors, or root assemblies.
- Mark libraries `IsTrimmable`/`IsAotCompatible` to enable analyzers and be a good AOT citizen.
- The real fix: replace runtime reflection with source generators (STJ, regex, logging, mappers).

→ Next: [06-PublishProfiles.md](06-PublishProfiles.md)
