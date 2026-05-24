# Chapter 14 — Interop & AOT — Q & A

---

### Q1. What is P/Invoke?

Platform Invoke — the mechanism for calling functions exported from native libraries (`.dll`/`.so`/`.dylib`) from managed C#. Declared with `[DllImport]` (classic) or `[LibraryImport]` (modern, source-generated).

---

### Q2. `[DllImport]` vs `[LibraryImport]`?

`[DllImport]` generates marshalling stubs at runtime (IL emit) — not fully AOT-safe. `[LibraryImport]` (C# 11+, `static partial` method) generates marshalling code at **compile time** via a source generator — AOT-safe, trimming-safe, and catches errors early. Prefer `[LibraryImport]` for new code.

---

### Q3. What is a blittable type?

A type with identical managed and native memory representation, requiring no marshalling: `int`, `double`, `IntPtr`, pointers, and structs/arrays of blittable types. Blittable interop is fastest. Non-blittable types (`string`, `bool`, `char`) require conversion at the boundary.

---

### Q4. Why is `bool` a marshalling trap?

Default `bool` marshals as a 4-byte Win32 `BOOL`, but a C `bool` is 1 byte. Mismatch reads garbage. Use `[MarshalAs(UnmanagedType.I1)]` for a 1-byte C bool, or `UnmanagedType.Bool` for Win32 BOOL.

---

### Q5. Why must you keep callback delegates alive?

If you pass a managed delegate to native code that stores it, the GC may collect the delegate while native code still holds the pointer → crash on callback. Store the delegate in a field. Better: use `[UnmanagedCallersOnly]` + function pointers (`delegate* unmanaged`), which are AOT-safe.

---

### Q6. How do you capture native errors correctly?

Set `SetLastError = true` on the import and call `Marshal.GetLastPInvokeError()` (or `GetLastWin32Error()`) **immediately** after the call, before any managed code can overwrite the thread's last-error.

---

### Q7. What is SafeHandle and why use it over `IntPtr` + finalizer?

`SafeHandle` wraps a native handle with `IDisposable` + critical finalization + reference counting. It prevents (1) handle-recycling races where a finalizer closes a reused handle, and (2) use-after-free where the handle is released mid-P/Invoke. `IntPtr` + finalizer has both races.

---

### Q8. How does SafeHandle prevent the P/Invoke use-after-free race?

When a `SafeHandle` is marshalled into a P/Invoke, the runtime increments its ref count for the duration of the call. The handle can't be released (by dispose or finalization) until the count returns to zero, so it stays valid throughout the native call.

---

### Q9. What must `ReleaseHandle` not do?

Throw. It runs during (possibly critical) finalization where exceptions are catastrophic. Return `false` to signal failure instead.

---

### Q10. What are RCW and CCW in COM interop?

RCW (Runtime Callable Wrapper) wraps a COM object so .NET can call it (COM→.NET proxy). CCW (COM Callable Wrapper) wraps a .NET object so COM clients can call it (.NET→COM proxy). The RCW manages the COM object's reference count.

---

### Q11. Why won't Excel close after automation?

Each dotted COM access (e.g., `excel.Workbooks`) creates an RCW holding a COM reference. Unreleased RCWs keep the COM server alive. Fix: hold each in a variable and call `Marshal.ReleaseComObject` on each, sometimes plus `GC.Collect()`. Better: avoid Office automation on servers; use file-format libraries.

---

### Q12. What is STA vs MTA?

COM threading apartments. STA (single-threaded apartment) serializes calls — required by many UI/Office COM objects. MTA (multi-threaded) is the default for .NET threadpool threads. Calling an STA object from MTA incurs marshalling or errors. Mark entry with `[STAThread]` or set `thread.SetApartmentState(ApartmentState.STA)`.

---

### Q13. What is Native AOT and its main benefit?

Ahead-of-Time compilation to a single self-contained native binary — no JIT, no IL, no runtime install. Main benefits: very fast startup (~ms), low memory, small size, predictable latency. Ideal for CLI tools, serverless, and containers.

---

### Q14. What doesn't work under Native AOT?

Runtime code generation: `Reflection.Emit`, `DynamicMethod`, `Expression.Compile`, `dynamic`, runtime assembly loading. Reflection is limited to statically-determinable usage. Trimming is mandatory. The fix is to use source generators instead of runtime reflection.

---

### Q15. What do `IL2xxx` and `IL3xxx` warnings mean?

Trim/AOT analyzer warnings. `IL2xxx` = trimming may remove members reflection needs (`RequiresUnreferencedCode`). `IL3xxx` = code needs runtime codegen, incompatible with AOT (`RequiresDynamicCode`). Treat them as errors — they predict runtime failures.

---

### Q16. Why does trimming break reflection?

Trimming uses static reachability analysis. Reflection by runtime string (`Type.GetType(name)`, `Activator.CreateInstance`) isn't statically visible, so the trimmer removes the type/members → null or missing-member failures at runtime.

---

### Q17. How do you tell the trimmer to preserve members?

Annotate reflection entry points with `[DynamicallyAccessedMembers(...)]` (flows the requirement through type parameters), use `[DynamicDependency]`, a trimmer XML descriptor (`preserve="all"`), or `TrimmerRootAssembly`. Propagate unavoidable reflection to callers with `[RequiresUnreferencedCode]`.

---

### Q18. What's the AOT/trim-safe alternative to reflection-based serialization, regex, logging?

Source generators: STJ source-gen (`JsonSerializerContext`), `[GeneratedRegex]`, `[LoggerMessage]`, Mapperly (mapping), source-generated config binding. They emit direct code at compile time — no reflection, fully trim/AOT-safe.

---

### Q19. Framework-dependent vs self-contained deployment?

Framework-dependent ships only your DLLs; the .NET runtime must be installed (tiny, shared, needs matching runtime). Self-contained bundles the runtime + app (large, no install needed, per-RID build, version-locked).

---

### Q20. What is ReadyToRun (R2R)?

Pre-compiles IL to native at publish time for faster startup, but keeps the IL and JIT so hot paths can still be re-optimized (with Dynamic PGO). A middle ground: faster startup than pure JIT, full steady-state throughput, larger output. Good for long-running servers.

---

### Q21. AOT vs JIT — which is faster?

AOT wins **startup** (no warmup). JIT + Dynamic PGO can win **steady-state throughput** because it re-optimizes hot paths with runtime profiles, while AOT optimizes only at build time. Use AOT for short-lived/serverless; JIT (+R2R) for long-running throughput servers.

---

### Q22. What's a RID and which modes require it?

A Runtime Identifier (e.g., `win-x64`, `linux-arm64`, `linux-musl-x64`) targets a specific OS+architecture. Self-contained, single-file, ReadyToRun, and Native AOT all require a RID (`-r`). Framework-dependent can be portable.

---

### Q23. What does `IsTrimmable` / `IsAotCompatible` do for a library?

Declares the assembly safe to trim / AOT-compile and enables the trim/AOT analyzers during the library's own build, so the author catches `IL2xxx`/`IL3xxx` issues before shipping. Makes the library a good citizen in trimmed/AOT apps.

---

### Q24. When should you NOT use trimming/AOT?

Apps relying on reflection-heavy frameworks (some ORMs, plugin systems, reflection-based DI scanning, `dynamic`, `Expression.Compile`). These break under trimming/AOT. Use framework-dependent deployment for them, or migrate to source-generated equivalents.

---

### Q25. What is `[SuppressGCTransition]`?

An attribute on a P/Invoke that removes the GC mode transition (~10-30 ns) for very fast, non-blocking native calls. The native function must not block or call back into managed code. Use with care for hot, tiny native calls (e.g., simple math).

---

→ Next: [Coding.md](Coding.md)
