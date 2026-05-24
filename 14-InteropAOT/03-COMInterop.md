# COM Interop

## What it is

COM (Component Object Model) is a Windows binary interface standard for components to expose objects via interfaces, with reference-counted lifetimes (`IUnknown`: `AddRef`/`Release`/`QueryInterface`). COM interop lets .NET call COM objects (Office automation, Windows Shell, DirectX, WMI) and lets COM clients call .NET objects.

```csharp
// Late-bound via dynamic (Office automation)
dynamic excel = Activator.CreateInstance(Type.GetTypeFromProgID("Excel.Application")!)!;
excel.Visible = true;
excel.Workbooks.Add();
excel.Cells[1, 1].Value = "Hello from .NET";
```

COM is **Windows-only**. Modern cross-platform .NET code rarely touches it, but it's still essential for automating Office, integrating legacy Windows components, and some shell/OS features.

---

## RCW and CCW

Two wrappers bridge the managed/COM boundary:

- **RCW (Runtime Callable Wrapper)** — a managed proxy that wraps a COM object so .NET can call it. The RCW holds the COM reference and forwards calls, translating .NET calls into `IUnknown`/vtable calls.
- **CCW (COM Callable Wrapper)** — the reverse: wraps a .NET object so COM clients can call it as a COM object.

```
.NET code → [RCW] → COM object        (calling COM from .NET)
COM client → [CCW] → .NET object       (calling .NET from COM)
```

The RCW tracks the COM object's reference count. When the RCW is collected (or explicitly released), it calls `Release` on the underlying COM object.

---

## Early binding with interop types

For type safety and IntelliSense, use a COM type library imported as an interop assembly (or `[ComImport]` declarations):

```csharp
[ComImport]
[Guid("00020400-0000-0000-C000-000000000046")]
[InterfaceType(ComInterfaceType.InterfaceIsIDispatch)]
public interface ISomeComInterface {
    void DoWork();
    int GetValue();
}

[ComImport]
[Guid("...")]
public class SomeComClass {}
```

Then:

```csharp
var obj = (ISomeComInterface)new SomeComClass();
obj.DoWork();
```

Visual Studio's "Add COM Reference" generates these interop types automatically from a registered type library.

---

## Managing COM object lifetime

COM uses reference counting. The RCW manages this, but the timing of `Release` matters — Office apps, for instance, won't quit until all references are released.

```csharp
using System.Runtime.InteropServices;

dynamic excel = ...;
try {
    // use excel
} finally {
    Marshal.ReleaseComObject(excel);    // decrement RCW ref count
    // or FinalReleaseComObject to force the count to zero
}
```

### The "Excel won't close" problem

```csharp
// ⚠ — each dotted access creates an RCW that isn't released
excel.Workbooks.Add();   // Workbooks RCW leaked
```

Every implicit COM object (like `Workbooks`) gets its own RCW. The classic fix is to hold each in a variable and release explicitly:

```csharp
var workbooks = excel.Workbooks;
var workbook = workbooks.Add();
// ... work ...
Marshal.ReleaseComObject(workbook);
Marshal.ReleaseComObject(workbooks);
Marshal.ReleaseComObject(excel);
GC.Collect();   // sometimes needed to release lingering RCWs
GC.WaitForPendingFinalizers();
```

This is tedious and error-prone — a reason to avoid Office automation server-side. Microsoft explicitly discourages Office automation on servers; prefer Open XML SDK or a library that reads/writes files directly.

---

## `dynamic` for late binding

For `IDispatch`-based COM (Office), `dynamic` makes the code far cleaner than manual `InvokeMember`:

```csharp
// With dynamic (clean)
excel.Range["A1"].Value = 42;

// Without dynamic (verbose reflection-style)
object range = excel.GetType().InvokeMember("Range", BindingFlags.GetProperty, null, excel, ["A1"]);
range.GetType().InvokeMember("Value", BindingFlags.SetProperty, null, range, [42]);
```

`dynamic` routes member access through `IDispatch.Invoke` at runtime. The trade-off: no compile-time checking, not AOT-compatible.

---

## Modern COM features (.NET 5+)

### Built-in COM hosting

.NET can expose managed classes as COM servers without `regasm`:

```csharp
[ComVisible(true)]
[Guid("...")]
[ClassInterface(ClassInterfaceType.None)]
public class MyComServer : IMyInterface { ... }
```

Generate a COM host with `<EnableComHosting>true</EnableComHosting>` in the project.

### Source-generated COM (`[GeneratedComInterface]`)

.NET 8+ introduced a source generator for COM interop — AOT-compatible, no built-in runtime marshalling:

```csharp
[GeneratedComInterface]
[Guid("...")]
public partial interface IMyComInterface {
    void DoWork();
}
```

This is the modern, trimming/AOT-friendly replacement for the runtime's built-in COM marshalling (which is **not** AOT-compatible). Use it for new COM interop on .NET 8+.

---

## COM and AOT/trimming

The runtime's classic built-in COM support (RCW/CCW via the marshaller) requires runtime code generation and reflection — **not AOT-compatible** and trim-unfriendly. For AOT scenarios:
- Use `[GeneratedComInterface]` (source-generated COM, .NET 8+).
- Avoid `dynamic`-based late binding (needs the DLR/JIT).

---

## Common bugs

### Leaked RCWs keeping COM servers alive

Covered above (Excel won't close). Release each RCW; avoid Office automation on servers.

### Calling COM from the wrong thread (apartment)

COM has threading apartments: **STA** (single-threaded) and **MTA** (multi-threaded). Many COM objects (especially UI/Office) require STA. .NET console/threadpool threads are MTA by default.

```csharp
[STAThread]   // mark the entry point
static void Main() { ... }

// Or for a worker thread:
var t = new Thread(Work);
t.SetApartmentState(ApartmentState.STA);
t.Start();
```

Calling an STA object from an MTA thread causes marshalling overhead or errors.

### Forgetting ComVisible

For .NET-to-COM (CCW), types must be `[ComVisible(true)]` (assembly default is often false). 

### Using `dynamic` in AOT

`dynamic` COM late binding won't work under NativeAOT. Use generated interfaces.

---

## When you'll encounter COM

- Automating Office (Excel, Word, Outlook) on desktop apps.
- Windows Shell integration (file dialogs, jump lists, taskbar).
- Legacy ActiveX / OLE components.
- DirectX, Media Foundation, WIC (imaging).
- WMI / system management.

When to **avoid**:
- Server-side Office processing — use Open XML SDK / file libraries.
- Cross-platform code — COM is Windows-only.
- New designs — prefer modern APIs / P/Invoke / libraries.

---

## Summary

- COM interop bridges .NET and Windows COM components via RCW (COM→.NET) and CCW (.NET→COM) wrappers.
- COM is reference-counted and Windows-only; manage RCW lifetimes with `Marshal.ReleaseComObject` (the "Excel won't close" trap).
- `dynamic` simplifies `IDispatch` late binding (Office) but isn't AOT-safe.
- Mind threading apartments (STA vs MTA) — Office/UI objects need STA.
- For .NET 8+ and AOT, use `[GeneratedComInterface]` (source-generated) instead of the built-in marshaller.
- Avoid Office automation on servers; prefer file-format libraries.

→ Next: [04-NativeAOT.md](04-NativeAOT.md)
