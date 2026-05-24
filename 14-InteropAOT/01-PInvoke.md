# P/Invoke — Platform Invoke

## What it is

P/Invoke (Platform Invoke) lets managed C# code call **functions exported from native libraries** (`.dll` on Windows, `.so` on Linux, `.dylib` on macOS). It's the bridge to the OS and to C/C++ libraries.

```csharp
using System.Runtime.InteropServices;

public static partial class Native {
    [LibraryImport("kernel32.dll")]
    public static partial uint GetCurrentProcessId();
}

uint pid = Native.GetCurrentProcessId();
```

Used to call OS APIs (Win32, POSIX), hardware drivers, and existing native libraries (zlib, SQLite, OpenSSL, etc.).

---

## Two ways: `[DllImport]` vs `[LibraryImport]`

| | `[DllImport]` (classic) | `[LibraryImport]` (C# 11+, modern) |
|---|---|---|
| Marshalling | Runtime-generated IL stubs | Compile-time source-generated |
| AOT-compatible | ⚠ partial | ✓ |
| Trimming-safe | ⚠ | ✓ |
| Method | regular `static extern` | `static partial` |
| Since | .NET Framework 1.0 | .NET 7 |

**Prefer `[LibraryImport]` for new code.** It generates marshalling code at compile time — faster, AOT-safe, and the warnings catch mistakes early. `[DllImport]` still works and is required for some advanced scenarios.

---

## `[LibraryImport]` basics

```csharp
public static partial class Native {
    // Simple value types — "blittable", no marshalling needed
    [LibraryImport("kernel32.dll")]
    public static partial uint GetCurrentProcessId();

    // String marshalling must be specified
    [LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
    public static partial int MessageBoxW(IntPtr hWnd, string text, string caption, uint type);

    // UTF-8 strings (common on Linux APIs)
    [LibraryImport("libc", StringMarshalling = StringMarshalling.Utf8)]
    public static partial int puts(string s);
}
```

The method must be `partial` and the containing type `partial` — the source generator fills in the body (the marshalling stub).

---

## Blittable vs non-blittable types

**Blittable** types have the same representation in managed and native memory — no conversion needed, fastest interop:
- `byte`, `sbyte`, `short`, `ushort`, `int`, `uint`, `long`, `ulong`, `float`, `double`
- `IntPtr`, `UIntPtr`, pointers
- Structs containing only blittable fields
- 1-D arrays of blittable types (with pinning)

**Non-blittable** types require marshalling (conversion/copy):
- `string` (UTF-16 ↔ native encoding)
- `bool` (1 byte native vs 4 bytes default — specify!)
- `char` (depends on charset)
- Arrays of non-blittable types, delegates, classes

```csharp
// bool is a trap — Win32 BOOL is 4 bytes, C bool is 1 byte
[LibraryImport("lib")]
[return: MarshalAs(UnmanagedType.Bool)]   // 4-byte Win32 BOOL
public static partial bool DoThing();

[LibraryImport("lib")]
[return: MarshalAs(UnmanagedType.I1)]      // 1-byte C bool
public static partial bool DoThingC();
```

Keep interop signatures blittable where possible — fastest and AOT-friendliest.

---

## Marshalling strings

```csharp
// UTF-16 (Windows "W" APIs)
[LibraryImport("user32.dll", StringMarshalling = StringMarshalling.Utf16)]
public static partial int MessageBoxW(IntPtr h, string text, string caption, uint type);

// UTF-8 (most Linux/cross-platform C APIs)
[LibraryImport("mylib", StringMarshalling = StringMarshalling.Utf8)]
public static partial int Process(string input);

// Custom (ANSI, etc.)
[LibraryImport("mylib", StringMarshalling = StringMarshalling.Custom,
    StringMarshallingCustomType = typeof(AnsiStringMarshaller))]
public static partial void Legacy(string s);
```

Strings are copied and converted at the boundary. For returning strings from native code, you often receive an `IntPtr` and marshal manually:

```csharp
[LibraryImport("mylib")]
public static partial IntPtr GetMessage();

IntPtr ptr = GetMessage();
string? msg = Marshal.PtrToStringUtf8(ptr);   // copy native string to managed
// If the native side allocated it, you may need to free it (per the library's contract)
```

---

## Marshalling structs

```csharp
[StructLayout(LayoutKind.Sequential)]
public struct Point {
    public int X;
    public int Y;
}

[LibraryImport("mylib")]
public static partial void MovePoint(ref Point p, int dx, int dy);

var p = new Point { X = 1, Y = 2 };
MovePoint(ref p, 10, 10);   // p mutated by native code
```

`[StructLayout(LayoutKind.Sequential)]` guarantees field order matches the C struct. `LayoutKind.Explicit` with `[FieldOffset(n)]` for unions / precise control:

```csharp
[StructLayout(LayoutKind.Explicit)]
public struct Union {
    [FieldOffset(0)] public int AsInt;
    [FieldOffset(0)] public float AsFloat;   // overlaps AsInt — a C union
}
```

Use `Pack` to control alignment when matching a packed C struct:

```csharp
[StructLayout(LayoutKind.Sequential, Pack = 1)]
public struct Packed { public byte A; public int B; }   // no padding
```

---

## Pointers and `ref`/`out`/`in`

```csharp
// Pass by reference (C T*)
[LibraryImport("mylib")]
public static partial void Fill(ref int value);

// Output parameter
[LibraryImport("mylib")]
public static partial int Query(out long result);

// Read-only reference (const T*) — efficient pass of large structs
[LibraryImport("mylib")]
public static partial void Inspect(in BigStruct data);

// Raw pointer (unsafe)
[LibraryImport("mylib")]
public static unsafe partial void Process(byte* buffer, int length);
```

For buffers, prefer passing `Span<byte>`/arrays with explicit length, or use `fixed` to pin and pass a pointer.

---

## Callbacks (function pointers)

Native code calling back into managed code:

```csharp
// Delegate-based (classic)
[UnmanagedFunctionPointer(CallingConvention.Cdecl)]
public delegate int CompareCallback(int a, int b);

[LibraryImport("mylib")]
public static partial void Sort(int[] array, int count, CompareCallback cmp);
```

⚠ The delegate must be **kept alive** for the duration the native code holds it — otherwise the GC collects it and the native callback crashes. Store it in a field.

Modern approach — **unmanaged function pointers** (`delegate*`), AOT-friendly:

```csharp
[LibraryImport("mylib")]
public static unsafe partial void Sort(int[] array, int count,
    delegate* unmanaged[Cdecl]<int, int, int> cmp);

[UnmanagedCallersOnly(CallConvs = [typeof(CallConvCdecl)])]
static int Compare(int a, int b) => a - b;

// Usage
unsafe { Sort(arr, arr.Length, &Compare); }
```

`[UnmanagedCallersOnly]` methods can be called directly from native code with zero marshalling overhead — required for NativeAOT callbacks.

---

## SetLastError and error handling

```csharp
[LibraryImport("kernel32.dll", SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
public static partial bool CloseHandle(IntPtr handle);

if (!CloseHandle(h)) {
    int err = Marshal.GetLastWin32Error();   // capture immediately after the call
    throw new Win32Exception(err);
}
```

`SetLastError = true` captures the OS error code right after the call (before any managed code can clobber it). Use `Marshal.GetLastPInvokeError()` (newer, cross-platform) or `GetLastWin32Error()`.

---

## Calling conventions

```csharp
[LibraryImport("mylib")]
[UnmanagedCallConv(CallConvs = [typeof(CallConvCdecl)])]   // Cdecl, Stdcall, etc.
public static partial int Compute(int x);
```

- **Cdecl** — caller cleans the stack (most C libraries, cross-platform).
- **Stdcall** — callee cleans (Win32 APIs).

Mismatched calling convention → stack corruption → crashes. Match the library's convention.

---

## `DllImportSearchPath` and library resolution

```csharp
[LibraryImport("mylib")]
[DefaultDllImportSearchPaths(DllImportSearchPath.SafeDirectories)]
public static partial void Foo();
```

For custom resolution (e.g., loading a platform-specific library at runtime):

```csharp
NativeLibrary.SetDllImportResolver(typeof(Native).Assembly, (name, asm, path) => {
    if (name == "mylib")
        return NativeLibrary.Load(OperatingSystem.IsWindows() ? "mylib.dll" : "libmylib.so");
    return IntPtr.Zero;   // fall back to default
});
```

`NativeLibrary` (.NET Core 3+) gives programmatic control over loading native libraries — essential for cross-platform packages.

---

## Common bugs

### Delegate garbage-collected during native call

```csharp
// ⚠ — the lambda's delegate may be GC'd while native code still holds it
Sort(arr, arr.Length, (a, b) => a - b);   // crash later

// ✓ — keep a reference alive
private static readonly CompareCallback _cmp = (a, b) => a - b;
Sort(arr, arr.Length, _cmp);
```

### Wrong bool marshalling

`bool` defaults to 4-byte Win32 BOOL. For a 1-byte C `bool`, use `[MarshalAs(UnmanagedType.I1)]`. Mismatches read garbage.

### Struct layout mismatch

Forgetting `[StructLayout(LayoutKind.Sequential)]`, or padding differences (`Pack`), cause fields to read wrong offsets. Match the C struct exactly.

### Not capturing last error immediately

Any managed code between the P/Invoke and `GetLastError` can overwrite it. Capture immediately, and use `SetLastError = true`.

### Memory ownership confusion

If native code returns an allocated pointer, who frees it? Read the library's contract. Freeing with the wrong allocator (or not at all) leaks or crashes.

---

## Performance notes

- Blittable signatures are fastest — no marshalling, near-direct call (~few ns overhead).
- `[LibraryImport]` source-gen has lower overhead than `[DllImport]` runtime stubs in some cases and is AOT-safe.
- Each P/Invoke has a fixed transition cost (GC mode switch, ~10-30 ns). Batch calls; don't call native APIs in tight loops if avoidable.
- `[SuppressGCTransition]` removes the GC transition for very fast, non-blocking native calls (use with care — the function must not block or call back into managed):

```csharp
[LibraryImport("mylib")]
[SuppressGCTransition]
public static partial int FastMath(int x);
```

---

## Summary

- P/Invoke calls native exported functions; `[LibraryImport]` (source-generated, AOT-safe) is preferred over `[DllImport]`.
- Blittable types interop with zero marshalling; strings, bool, structs need care.
- Use `[StructLayout]` to match C structs; specify `StringMarshalling` and calling conventions.
- Keep callback delegates alive; prefer `[UnmanagedCallersOnly]` + function pointers for AOT.
- Capture errors with `SetLastError = true` + `GetLastPInvokeError()`.
- Use `NativeLibrary` for cross-platform loading; `[SuppressGCTransition]` for fast non-blocking calls.

→ Next: [02-SafeHandle.md](02-SafeHandle.md)
