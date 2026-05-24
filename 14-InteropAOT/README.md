# Chapter 14 — Interop & AOT

> Talking to native code, owning unmanaged resources safely, and publishing your app as a small, fast, single-binary native executable.

**Prerequisites**: Chapters 09 (Memory), 13 (IO).

**Time to read**: ~5-6 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-PInvoke.md](01-PInvoke.md) | `[DllImport]`, the classic interop. `[LibraryImport]` (C# 11+) source-generated marshaling. Strings, structs, callbacks, pointers. |
| [02-SafeHandle.md](02-SafeHandle.md) | Wrapping a native handle with critical finalization, ref counting during P/Invoke, why `IntPtr` + finalizer is the wrong way. |
| [03-COMInterop.md](03-COMInterop.md) | Windows-specific COM interop, `[ComImport]`, RCWs and CCWs, when modern code still encounters this. |
| [04-NativeAOT.md](04-NativeAOT.md) | `PublishAot=true`, startup-time wins, size and dependency-loading limitations, dynamic-code-disabled. |
| [05-Trimming.md](05-Trimming.md) | The linker, `IsTrimmable`, `DynamicallyAccessedMembers`, trim warnings, the path to AOT-friendly code. |
| [06-PublishProfiles.md](06-PublishProfiles.md) | `dotnet publish` modes: framework-dependent, self-contained, single-file, ReadyToRun (R2R), AOT, choosing per scenario. |
| [Questions.md](Questions.md) | ~15 questions. |
| [Coding.md](Coding.md) | ~10 problems: write a `LibraryImport` wrapper, build an AOT console app, fix trim warnings. |

→ Begin: [01-PInvoke.md](01-PInvoke.md)
