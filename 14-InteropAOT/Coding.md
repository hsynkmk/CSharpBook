# Chapter 14 — Interop & AOT — Coding Problems

---

## Problem 1: Modern P/Invoke for getpid / GetCurrentProcessId

Write an AOT-safe wrapper that returns the current process ID, cross-platform.

<details><summary>Solution</summary>

```csharp
using System.Runtime.InteropServices;

public static partial class Native {
    [LibraryImport("kernel32.dll")]
    private static partial uint GetCurrentProcessId();   // Windows

    [LibraryImport("libc", EntryPoint = "getpid")]
    private static partial int getpid();                 // Unix

    public static int ProcessId() =>
        OperatingSystem.IsWindows() ? (int)GetCurrentProcessId() : getpid();
}
```

`[LibraryImport]` with `partial` methods is source-generated — AOT-safe. `uint`/`int` are blittable, no marshalling. (In practice `Environment.ProcessId` is the BCL way — this is for illustrating P/Invoke.)

</details>

---

## Problem 2: Marshal a string to a native UTF-8 API

Call a native `int process(const char* utf8)` with proper string marshalling.

<details><summary>Solution</summary>

```csharp
public static partial class Native {
    [LibraryImport("mylib", StringMarshalling = StringMarshalling.Utf8)]
    public static partial int Process(string input);
}

int result = Native.Process("héllo");   // marshalled to UTF-8 bytes automatically
```

`StringMarshalling.Utf8` converts the UTF-16 managed string to a null-terminated UTF-8 buffer for the call. For Windows wide APIs use `StringMarshalling.Utf16`.

</details>

---

## Problem 3: Pass and mutate a struct

Declare a C-compatible struct and call a native function that modifies it via pointer.

<details><summary>Solution</summary>

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Rect {
    public int Left, Top, Right, Bottom;
}

public static partial class Native {
    [LibraryImport("mylib")]
    public static partial void InflateRect(ref Rect r, int dx, int dy);
}

var rect = new Rect { Left = 0, Top = 0, Right = 10, Bottom = 10 };
Native.InflateRect(ref rect, 5, 5);   // native code mutates rect in place
```

`[StructLayout(LayoutKind.Sequential)]` ensures field order/layout matches the C struct. `ref` passes a pointer the native code can write through. The struct is blittable, so no copy/conversion.

</details>

---

## Problem 4: Receive a string returned by native code

A native function returns `const char*` (UTF-8). Marshal it to a managed string.

<details><summary>Solution</summary>

```csharp
public static partial class Native {
    [LibraryImport("mylib")]
    public static partial IntPtr GetVersion();   // returns char*
}

string? version = Marshal.PtrToStringUTF8(Native.GetVersion());
```

When native code owns the returned buffer, you marshal manually with `Marshal.PtrToStringUTF8` (copies to a managed string). Check the library's contract for who frees the buffer — if the library allocated it for you to free, call the appropriate free function.

</details>

---

## Problem 5: AOT-safe callback with function pointers

Pass a comparison callback to a native sort, AOT-compatible (no GC'd delegate).

<details><summary>Solution</summary>

```csharp
using System.Runtime.CompilerServices;
using System.Runtime.InteropServices;

public static partial class Native {
    [LibraryImport("mylib")]
    public static unsafe partial void Sort(int* data, int count,
        delegate* unmanaged[Cdecl]<int, int, int> compare);

    [UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
    private static int Compare(int a, int b) => a.CompareTo(b);

    public static unsafe void SortAscending(int[] data) {
        fixed (int* p = data)
            Sort(p, data.Length, &Compare);   // address of the unmanaged-callable method
    }
}
```

`[UnmanagedCallersOnly]` methods are callable directly from native code with no marshalling and no delegate object to keep alive — the AOT-safe callback pattern. The classic `delegate`-based approach risks GC collecting the delegate.

</details>

---

## Problem 6: Write a custom SafeHandle

Wrap a native handle from `OpenThing`/`CloseThing` in a SafeHandle.

<details><summary>Solution</summary>

```csharp
using System.Runtime.InteropServices;
using Microsoft.Win32.SafeHandles;

public sealed class SafeThingHandle : SafeHandleZeroOrMinusOneIsInvalid {
    public SafeThingHandle() : base(ownsHandle: true) {}

    protected override bool ReleaseHandle() => CloseThing(handle);   // never throws

    [LibraryImport("mylib")]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool CloseThing(IntPtr h);
}

public static partial class Native {
    [LibraryImport("mylib", StringMarshalling = StringMarshalling.Utf8)]
    public static partial SafeThingHandle OpenThing(string name);
}

// Usage — deterministic release, exception-safe, GC backstop
using var thing = Native.OpenThing("resource");
if (thing.IsInvalid) throw new InvalidOperationException("open failed");
```

The runtime constructs the `SafeThingHandle` and stores the raw handle; ref-counts it during P/Invoke; calls `ReleaseHandle` exactly once on dispose or finalization. No manual finalizer needed.

</details>

---

## Problem 7: Build a minimal Native AOT console app

Set up a project that publishes as a native binary, and identify what to avoid.

<details><summary>Solution</summary>

```xml
<!-- app.csproj -->
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net10.0</TargetFramework>
    <PublishAot>true</PublishAot>
    <InvariantGlobalization>true</InvariantGlobalization>
  </PropertyGroup>
</Project>
```

```csharp
// Program.cs — AOT-friendly
using System.Text.Json;
using System.Text.Json.Serialization;

var data = new Config("prod", 8080);
string json = JsonSerializer.Serialize(data, AppContext.Default.Config);
Console.WriteLine(json);

record Config(string Env, int Port);

[JsonSerializable(typeof(Config))]
partial class AppContext : JsonSerializerContext {}
```

```bash
dotnet publish -r linux-x64 -c Release
```

Avoid: `JsonSerializer.Serialize(data)` (reflection-based — `IL2026`/`IL3050` warning); use the source-gen `JsonSerializerContext` overload. No `Expression.Compile`, no `dynamic`, no `Type.GetType(string)`.

</details>

---

## Problem 8: Fix a trim warning

This code produces `IL2057` and returns null when trimmed. Fix it.

```csharp
public object CreateHandler(string typeName) {
    Type? t = Type.GetType(typeName);
    return Activator.CreateInstance(t!)!;
}
```

<details><summary>Solution</summary>

The trimmer can't see which types reach `Type.GetType(string)`, so it trims them. Options:

**Option A — avoid reflection (best):** use a known mapping.
```csharp
private static readonly Dictionary<string, Func<object>> Factories = new() {
    ["create"] = () => new CreateHandler(),
    ["delete"] = () => new DeleteHandler(),
};
public object CreateHandler(string key) => Factories[key]();
```

**Option B — preserve the types explicitly:**
```csharp
[DynamicDependency(DynamicallyAccessedMemberTypes.PublicConstructors, typeof(CreateHandler))]
[DynamicDependency(DynamicallyAccessedMemberTypes.PublicConstructors, typeof(DeleteHandler))]
public object CreateHandler(string typeName) {
    Type t = Type.GetType(typeName) ?? throw new InvalidOperationException();
    return Activator.CreateInstance(t)!;
}
```

**Option C — annotate the flow** if the type comes via a `Type` parameter:
```csharp
public object Create([DynamicallyAccessedMembers(DynamicallyAccessedMemberTypes.PublicConstructors)] Type t)
    => Activator.CreateInstance(t)!;
```

Option A is the most robust — it removes reflection entirely and is fully AOT/trim-safe.

</details>

---

## Problem 9: Choose a deployment mode

For each scenario, pick the publish mode and justify.
1. A CLI tool distributed to developers.
2. A high-throughput API running 24/7 in your Kubernetes cluster (you control the image).
3. An AWS Lambda function invoked sporadically.
4. A plugin host that loads assemblies at runtime.

<details><summary>Solution</summary>

1. **CLI tool → Native AOT.** Single binary, no .NET install needed, instant startup. `dotnet publish -r <rid> -p:PublishAot=true`.

2. **24/7 high-throughput API → Framework-dependent + ReadyToRun.** You control the base image's runtime (small output). R2R gives fast startup; JIT + Dynamic PGO delivers peak steady-state throughput (AOT would forgo runtime re-optimization). `<PublishReadyToRun>true</PublishReadyToRun>`.

3. **Lambda (sporadic) → Native AOT.** Cold-start dominates cost/latency for sporadic invocation; AOT's ~ms startup wins decisively. Lambda's custom runtime supports native binaries.

4. **Plugin host → Framework-dependent (no trimming/AOT).** Runtime assembly loading and reflection over plugin types are incompatible with AOT/trimming. Keep full JIT + reflection.

</details>

---

## Problem 10: Cross-platform native library resolution

Load `mylib.dll` on Windows but `libmylib.so` on Linux from the same `[LibraryImport]`.

<details><summary>Solution</summary>

```csharp
using System.Runtime.InteropServices;

public static partial class Native {
    private const string Lib = "mylib";   // base name

    [LibraryImport(Lib)]
    public static partial int Compute(int x);

    static Native() {
        NativeLibrary.SetDllImportResolver(typeof(Native).Assembly, Resolve);
    }

    private static IntPtr Resolve(string name, Assembly asm, DllImportSearchPath? path) {
        if (name != Lib) return IntPtr.Zero;   // default handling
        string file = OperatingSystem.IsWindows() ? "mylib.dll"
                    : OperatingSystem.IsMacOS()   ? "libmylib.dylib"
                    :                                "libmylib.so";
        return NativeLibrary.Load(file, asm, path);
    }
}
```

`NativeLibrary.SetDllImportResolver` intercepts library resolution so one import name maps to the right per-OS file. Essential for cross-platform NuGet packages shipping native binaries. AOT-compatible (no reflection).

</details>

---

These problems cover modern interop (`[LibraryImport]`, function-pointer callbacks, SafeHandle) and the AOT/trimming workflow (source-gen serialization, fixing trim warnings, deployment-mode selection) — the practical skills for shipping fast, native-friendly .NET 10 binaries.

→ Back to [Chapter 14 README](README.md). Next chapter: [Chapter 15 — Build & Tooling](../15-BuildTooling/README.md).
