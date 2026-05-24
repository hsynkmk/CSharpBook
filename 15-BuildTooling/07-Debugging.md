# Debugging

## What it is

Debugging is inspecting and controlling a running program to understand its behavior — pausing execution, examining state, stepping through code, and diagnosing crashes/hangs. .NET offers IDE debuggers (Visual Studio, VS Code, Rider) and command-line diagnostic tools (`dotnet-dump`, `dotnet-trace`, `dotnet-gcdump`) for production.

---

## Breakpoints

```csharp
int Total(int[] xs) {
    int sum = 0;
    foreach (var x in xs) {   // ← set a breakpoint here (F9 in VS)
        sum += x;
    }
    return sum;
}
```

When hit, execution pauses and you can inspect locals, the call stack, and watch expressions.

### Conditional breakpoints

Break only when a condition holds — invaluable for loops:

```
Condition:  x > 100 && index == 5
Hit count:  break when hit count == 10
Filter:     break only on a specific thread
```

Set via right-click → Conditions. Avoids manually stepping through thousands of iterations.

### Tracepoints (logpoints)

A breakpoint that **logs a message instead of stopping** — like inserting `Console.WriteLine` without editing code:

```
Message: "Processing {x} at index {index}, sum={sum}"
```

VS Code calls these "logpoints"; VS calls them tracepoints ("Actions" → "Log a message"). Great for high-frequency code where stopping is impractical.

### Function and data breakpoints

- **Function breakpoint** — break when a named method is entered (without opening the file).
- **Data breakpoint** — break when a specific object's field/property changes value (powerful for "who is mutating this?").

---

## Stepping

| Action | Key (VS) | Behavior |
|---|---|---|
| Step Over | F10 | Execute the line; don't enter called methods |
| Step Into | F11 | Enter the called method |
| Step Out | Shift+F11 | Run to the end of the current method, return to caller |
| Continue | F5 | Resume until the next breakpoint |
| Run to Cursor | Ctrl+F10 | Run until the cursor's line |

**Step Into Specific** lets you pick which method on a line to enter (when a line has nested calls).

---

## Inspecting state

### Watch and locals

- **Locals** window — all variables in scope.
- **Autos** — variables used near the current line.
- **Watch** — expressions you add (`order.Total * 1.1`, `list.Count`).

You can evaluate arbitrary expressions, including method calls (careful — side effects).

### DataTips and the Immediate window

- Hover over a variable to see its value (DataTip), pin it to keep it visible.
- **Immediate window** — run expressions interactively while paused: `? order.Items.Count`, or even mutate: `order.Status = Status.Done`.

### Call stack and threads

- **Call Stack** window — the chain of method calls. Double-click a frame to inspect its locals.
- **Threads** / **Parallel Stacks** — for multithreaded/async code, see all threads and async call chains. Essential for diagnosing deadlocks (see [Chapter 08 §17](../08-Concurrency/17-CommonAsyncBugs.md)).

---

## Async debugging

Async code's logical call stack is split across continuations. Modern debuggers reconstruct it:
- **Async call stacks** show the logical `await` chain, not just the physical thread stack.
- **Tasks** window shows running/scheduled tasks and their status.
- "Just My Code" hides framework async plumbing.

For deadlocks: pause, open **Parallel Stacks**, look for threads blocked on `.Result`/`.Wait()` while holding a context — the classic sync-over-async deadlock.

---

## Debugger attributes

Shape how your types appear in the debugger:

```csharp
[DebuggerDisplay("{Name} (Id={Id})")]
public class Customer {
    public int Id { get; set; }
    public string Name { get; set; } = "";

    [DebuggerBrowsable(DebuggerBrowsableState.Never)]
    private string _internal = "";   // hidden in the debugger
}

[DebuggerStepThrough]                  // never step into this method
public int Trivial() => 42;
```

`[DebuggerDisplay]` gives a meaningful one-line summary instead of the type name. `[DebuggerStepThrough]`/`[DebuggerHidden]` skip boilerplate.

---

## Conditional compilation for debug

```csharp
[Conditional("DEBUG")]
static void Trace(string msg) => Console.WriteLine(msg);   // calls removed in Release

Debug.Assert(index >= 0, "index must be non-negative");    // Debug-only assertion
Debug.WriteLine($"value = {value}");
```

`Debug.Assert` fires only in Debug builds — a cheap invariant check. `[Conditional("DEBUG")]` elides call sites in Release (see [Chapter 12 §03](../12-Reflection/03-Attributes.md)).

---

## Production diagnostics — dotnet tools

When you can't attach a debugger (production, containers), use the diagnostic CLI tools (install as global tools):

```bash
dotnet tool install -g dotnet-dump
dotnet tool install -g dotnet-trace
dotnet tool install -g dotnet-gcdump
dotnet tool install -g dotnet-counters
```

### `dotnet-dump` — capture and analyze process dumps

```bash
dotnet-dump collect -p <pid>           # capture a full memory dump
dotnet-dump analyze core_dump          # interactive SOS-style analysis

# Inside analyze:
> clrstack            # managed call stack
> clrthreads          # all managed threads
> dumpheap -stat      # heap object counts by type (find leaks)
> gcroot <address>    # why is this object still alive?
> pstacks             # parallel stacks (deadlock diagnosis)
```

Dumps are the production equivalent of pausing in the debugger — capture the state, analyze offline. `dumpheap -stat` + `gcroot` is the classic memory-leak hunt. See [Chapter 09 §13](../09-MemoryPerformance/13-MemoryLeaks.md).

### `dotnet-trace` — collect performance traces

```bash
dotnet-trace collect -p <pid>                         # CPU sampling + events
dotnet-trace collect -p <pid> --profile gc-verbose    # GC events
# produces a .nettrace, view in PerfView / Visual Studio / speedscope
```

Low-overhead, no debugger needed — safe in production. See [08-Profiling.md](08-Profiling.md).

### `dotnet-gcdump` — heap snapshot for leaks

```bash
dotnet-gcdump collect -p <pid>     # lightweight managed heap snapshot
# open the .gcdump in Visual Studio to see object counts and retention paths
```

### `dotnet-stack` — quick stack dump

```bash
dotnet-stack report -p <pid>       # print managed stacks of all threads (great for hangs)
```

---

## Symbols and SourceLink

To map native/IL back to your source:
- **PDB files** carry debug symbols. `<DebugType>portable</DebugType>` produces cross-platform PDBs.
- **SourceLink** embeds source-control info so the debugger fetches the exact source for a package/commit — step into NuGet dependencies and your own published builds.

```xml
<PropertyGroup>
  <DebugType>portable</DebugType>
  <PublishRepositoryUrl>true</PublishRepositoryUrl>
  <EmbedUntrackedSources>true</EmbedUntrackedSources>
</PropertyGroup>
```

---

## Common bugs / gotchas

### Debugging a Release build

Optimizations reorder/inline code; locals may be "optimized away," breakpoints may not bind. Debug with a Debug build, or set `<Optimize>false</Optimize>` temporarily. For Release-only bugs, use a dump + trace.

### Heisenbugs from evaluating expressions

Watch expressions that call methods can have side effects (mutating state, advancing iterators). Use the "Tools → no implicit function evaluation" / pure-only options when inspecting.

### Async "step over" skipping awaits oddly

Stepping across `await` can jump unexpectedly because of continuations. Use async call stacks and set breakpoints after the `await` rather than stepping.

### Attaching too late

For startup bugs, `Debugger.Launch()` / `System.Diagnostics.Debugger.Break()` in code, or `DOTNET_DefaultDiagnosticPortSuspend`, lets you attach before the interesting code runs.

---

## Summary

- Use breakpoints (incl. **conditional**, **tracepoints**, **data breakpoints**) and stepping (F10/F11/Shift+F11) to control execution.
- Inspect via Locals/Watch/Immediate; use Call Stack and Parallel/Async stacks for multithreaded and deadlock diagnosis.
- Shape debugger output with `[DebuggerDisplay]`; skip noise with `[DebuggerStepThrough]`; assert invariants with `Debug.Assert`.
- For production (no debugger): `dotnet-dump` (state + heap), `dotnet-trace` (perf), `dotnet-gcdump` (leaks), `dotnet-stack` (hangs).
- Ship portable PDBs + SourceLink to step into source.
- Debug with Debug builds; for Release-only issues, capture dumps/traces.

→ Next: [08-Profiling.md](08-Profiling.md)
