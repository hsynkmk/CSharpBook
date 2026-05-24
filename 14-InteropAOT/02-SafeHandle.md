# SafeHandle

## What it is

`SafeHandle` is the correct way to wrap a **native resource handle** (file descriptor, socket, OS handle, native allocation) so it's released reliably — even on exceptions, and even if you forget to dispose. It combines `IDisposable`, critical finalization, and reference counting to prevent handle leaks and use-after-free across P/Invoke boundaries.

```csharp
using Microsoft.Win32.SafeHandles;

[LibraryImport("kernel32.dll", SetLastError = true)]
private static partial SafeFileHandle CreateFileW(string name, uint access, ...);

[LibraryImport("kernel32.dll", SetLastError = true)]
[return: MarshalAs(UnmanagedType.Bool)]
private static partial bool CloseHandle(IntPtr handle);
```

The runtime marshals the native handle into a `SafeHandle` subclass automatically, and the handle is closed deterministically on dispose or, as a backstop, on finalization.

---

## Why `IntPtr` + finalizer is wrong

The naive approach stores the handle as `IntPtr` and closes it in a finalizer:

```csharp
// ⚠ — DON'T do this
class BadWrapper : IDisposable {
    private IntPtr _handle;
    ~BadWrapper() => CloseHandle(_handle);   // runs on finalizer thread
    public void Dispose() => CloseHandle(_handle);
}
```

Problems:
1. **Handle recycling race**: between you finishing with the handle and the finalizer running, the OS may reuse the handle number for a different resource. The finalizer then closes someone else's handle.
2. **P/Invoke race**: if a native call is using the `IntPtr` while the object becomes eligible for finalization, the finalizer can close the handle mid-call → use-after-free.
3. **No critical finalization**: in some shutdown scenarios, ordinary finalizers may not run; the handle leaks.
4. **No ref counting**: nothing prevents the handle from being closed while marshalled into a P/Invoke.

`SafeHandle` solves all four.

---

## How SafeHandle solves it

`SafeHandle` is a `CriticalFinalizerObject` — its finalizer is **guaranteed to run** (even during shutdown/abort), and runs *after* ordinary finalizers, so dependent objects clean up first.

It maintains a **reference count**: when a `SafeHandle` is marshalled into a P/Invoke, the runtime increments the count for the duration of the call. The handle can't be released while a native call is using it — eliminating the use-after-free race.

```
P/Invoke call begins → SafeHandle.DangerousAddRef (count++)
... native code runs, handle guaranteed valid ...
P/Invoke call returns → SafeHandle.DangerousRelease (count--)
When count == 0 AND disposed/finalized → ReleaseHandle() called exactly once
```

---

## Writing a custom SafeHandle

Subclass `SafeHandle` (or `SafeHandleZeroOrMinusOneIsInvalid` / `SafeHandleMinusOneIsInvalid` helpers) and implement `ReleaseHandle`:

```csharp
public sealed class SafeMyHandle : SafeHandleZeroOrMinusOneIsInvalid {
    // Parameterless ctor for the runtime to marshal into
    public SafeMyHandle() : base(ownsHandle: true) {}

    // ReleaseHandle: close the native resource. Called exactly once, guaranteed.
    protected override bool ReleaseHandle() {
        return NativeClose(handle);   // 'handle' is the protected IntPtr field
    }

    [LibraryImport("mylib")]
    [return: MarshalAs(UnmanagedType.Bool)]
    private static partial bool NativeClose(IntPtr h);
}

// P/Invoke that returns the handle — runtime constructs SafeMyHandle and stores the IntPtr
[LibraryImport("mylib")]
public static partial SafeMyHandle MyOpen(string path);
```

Key points:
- `ownsHandle: true` means this instance is responsible for releasing.
- `ReleaseHandle` must not throw and should release exactly the resource. It runs on the finalizer thread if not disposed.
- The helper bases define what "invalid" means: `SafeHandleZeroOrMinusOneIsInvalid` treats `0` and `-1` as invalid (so `IsInvalid` works correctly).

---

## Built-in SafeHandle types

The BCL provides ready-made ones — use these rather than rolling your own when possible:

| Type | Wraps |
|---|---|
| `SafeFileHandle` | File handle / fd |
| `SafeWaitHandle` | Synchronization handle |
| `SafeProcessHandle` | Process handle |
| `SafeAccessTokenHandle` | Windows access token |
| `SafeMemoryMappedFileHandle` | Memory-mapped file |
| `SocketHandle` (internal) | Sockets |

```csharp
using SafeFileHandle handle = File.OpenHandle(path, FileMode.Open, FileAccess.Read);
long len = RandomAccess.GetLength(handle);
byte[] buf = new byte[len];
RandomAccess.Read(handle, buf, 0);
// handle closed deterministically on dispose
```

---

## Using SafeHandle in a class

```csharp
public sealed class NativeResource : IDisposable {
    private readonly SafeMyHandle _handle;

    public NativeResource(string path) {
        _handle = Native.MyOpen(path);
        if (_handle.IsInvalid)
            throw new Win32Exception(Marshal.GetLastPInvokeError());
    }

    public void DoWork() {
        // Pass the SafeHandle directly to P/Invoke — runtime ref-counts it
        Native.MyWork(_handle);
    }

    public void Dispose() => _handle.Dispose();   // releases the native handle
}
```

Because `SafeMyHandle` owns the lifetime, `NativeResource` doesn't need a finalizer at all — the `SafeHandle`'s critical finalizer is the backstop. This is the modern `IDisposable` pattern: **wrap each native resource in a SafeHandle, and you rarely need to write a finalizer yourself.** See [Chapter 09 §03-04](../09-MemoryPerformance/03-IDisposable.md).

---

## `DangerousGetHandle` / `DangerousAddRef`

Escape hatches for when you must get the raw `IntPtr`:

```csharp
bool added = false;
try {
    handle.DangerousAddRef(ref added);     // prevent release during use
    IntPtr raw = handle.DangerousGetHandle();
    NativeUse(raw);
} finally {
    if (added) handle.DangerousRelease();
}
```

Named "Dangerous" because you bypass the safety. Only use when an API can't take the `SafeHandle` directly, and always pair AddRef/Release in try/finally.

---

## Common bugs

### Rolling your own with IntPtr + finalizer

Covered above — recycling and P/Invoke races. Always use `SafeHandle`.

### Throwing from ReleaseHandle

`ReleaseHandle` must not throw — it runs during finalization where exceptions are catastrophic. Return `false` to signal failure (the runtime logs a `ReleaseHandleFailed` MDA in debug).

### Forgetting `ownsHandle`

If you wrap a handle you don't own (borrowed from elsewhere), pass `ownsHandle: false` so `ReleaseHandle` isn't called — otherwise you double-free.

### Using DangerousGetHandle without AddRef

The handle may be released between getting the raw pointer and using it. Use `DangerousAddRef`/`DangerousRelease`.

---

## Performance notes

- Ref counting on each P/Invoke adds a small cost (~few ns) vs raw `IntPtr` — negligible compared to the safety gained.
- `SafeHandle` marshalling is slightly more expensive than `IntPtr` marshalling. For extremely hot paths where you've proven the resource lifetime is safe, `IntPtr` with manual management is sometimes used — but only with great care.
- The critical finalizer is the backstop, not the primary path: always `Dispose` (via `using`) for deterministic, prompt release.

---

## When to use

- **Always** when wrapping a native handle/resource that needs releasing.
- Prefer built-in SafeHandle types (`SafeFileHandle`, etc.).
- Write a custom subclass for library-specific handles.

When `IntPtr` is acceptable:
- Transient pointers that never outlive a single P/Invoke call and aren't released by you.
- Values that aren't really "handles to release" (e.g., an opaque token compared by value).

---

## Summary

- `SafeHandle` wraps native handles with `IDisposable` + critical finalization + reference counting.
- It eliminates handle-recycling and P/Invoke use-after-free races that plague `IntPtr` + finalizer.
- Subclass `SafeHandleZeroOrMinusOneIsInvalid` and implement `ReleaseHandle` (must not throw).
- Use built-in types (`SafeFileHandle`, etc.) when available.
- With SafeHandle, your wrapper classes usually need no finalizer — the SafeHandle is the backstop.
- Always `Dispose` for prompt release; the finalizer is only the safety net.

→ Next: [03-COMInterop.md](03-COMInterop.md)
