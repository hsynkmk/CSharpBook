# ConfigureAwait and SynchronizationContext

## What it is

When you `await` a task, by default the continuation runs on the **captured context** — the SynchronizationContext or TaskScheduler that was active when the await began. `ConfigureAwait(false)` says "I don't need the original context; run me on any thread."

```csharp
public async Task<int> Lib() {
    // Caller had a UI context (e.g., WPF main thread)
    var data = await httpClient.GetStringAsync(url);   // continuation back on UI thread
    var data2 = await httpClient.GetStringAsync(url2).ConfigureAwait(false);
    // continuation on any thread (whichever finishes the http call)
}
```

`ConfigureAwait(false)` matters for **library code** running on threads with a SynchronizationContext (WinForms, WPF, old ASP.NET). For ASP.NET Core, console apps, and worker services — there's no SynchronizationContext, and ConfigureAwait does nothing.

---

## Why it exists

### SynchronizationContext recap

UI frameworks have a "main thread" rule: only the UI thread can touch UI controls. Pre-async/await, you'd manually marshal back to the UI thread via `Control.Invoke` or `Dispatcher.BeginInvoke`. Painful.

async/await automated this by **capturing the SynchronizationContext** at every await and resuming the continuation on it:

```csharp
public async void Button_Click(object s, EventArgs e) {
    var data = await httpClient.GetStringAsync(url);
    label.Text = data;   // ← back on UI thread automatically
}
```

`await` magically posts the continuation to the UI thread. The label update works. No `Invoke` needed.

### The library problem

Library code shouldn't care which thread it runs on:

```csharp
// In a library
public async Task<string> DownloadAsync() {
    using var client = new HttpClient();
    var response = await client.GetAsync(url);   // captures UI context
    var body = await response.Content.ReadAsStringAsync();   // captures UI context AGAIN
    return body;
}
```

Each await marshals back to the UI thread for the next line. That:
1. **Wastes UI thread time** doing work that doesn't need it.
2. **Risks deadlock** if the caller did `.Result` or `.Wait()` (blocking the UI thread that the continuation needs).

`ConfigureAwait(false)` skips the marshaling:

```csharp
public async Task<string> DownloadAsync() {
    using var client = new HttpClient();
    var response = await client.GetAsync(url).ConfigureAwait(false);
    var body = await response.Content.ReadAsStringAsync().ConfigureAwait(false);
    return body;
}
```

Now continuations run on any free thread pool thread. Faster, no deadlock risk. The library doesn't care about the caller's thread.

---

## The rule

**In libraries** (code that doesn't know the caller's context): `ConfigureAwait(false)` everywhere.

**In application code**: skip it. Either you're on a UI thread (and want to come back), or you're in ASP.NET Core / Console (no context to capture).

This is the "asynchronous all the way" + "ConfigureAwait(false) all the way in libraries" pattern.

---

## When you'd want ConfigureAwait(true)

`true` is the default — you rarely write it explicitly. You'd want it (or just leave the default) when:
- You're a UI app handler that genuinely wants the continuation on the UI thread.
- You're a Razor Pages / Blazor handler that needs the rendering context.

Otherwise: `false` in libraries, default in app code.

---

## ASP.NET Core has NO SynchronizationContext

```csharp
// ASP.NET Core controller — no context to capture
public async Task<IActionResult> Get(CancellationToken ct) {
    var data = await _db.Users.ToListAsync(ct);   // continuation on any thread
    return Ok(data);
}
```

In ASP.NET Core (and Console, and Worker Services), the SynchronizationContext is null. `await` captures null → resumes on whatever thread completes the task. `ConfigureAwait(false)` is **a no-op** — same behavior.

So in ASP.NET Core application code, you don't need to write `ConfigureAwait(false)`. Library code shared across frameworks still should.

This is a confusing point: tutorials from old ASP.NET (pre-Core) advised always writing `ConfigureAwait(false)`. ASP.NET Core dropped the SynchronizationContext, making it unnecessary in app code. The library guidance hasn't changed.

---

## Deadlock scenario

The classic:

```csharp
// UI thread:
var result = LibAsync().Result;   // ⚠ blocks UI thread
```

Without `ConfigureAwait(false)` in the library:
```csharp
public async Task<int> LibAsync() {
    await Task.Delay(100);
    return 42;
}
```

What happens:
1. `LibAsync()` runs, hits `await Task.Delay(100)`.
2. Captures the UI SynchronizationContext.
3. Returns the incomplete task.
4. `.Result` BLOCKS the UI thread.
5. After 100 ms, Task.Delay completes.
6. Continuation tries to post to the UI thread — which is blocked.
7. **Deadlock.**

With `ConfigureAwait(false)`:
```csharp
public async Task<int> LibAsync() {
    await Task.Delay(100).ConfigureAwait(false);
    return 42;
}
```

The continuation runs on any thread → no deadlock.

But the **real fix** is "never use `.Result`." `ConfigureAwait(false)` patches over the symptom. Async-all-the-way is the cure.

---

## Where to put ConfigureAwait(false)

**Every await in a library:**

```csharp
public async Task<T> Get<T>(string url) {
    var resp = await _http.GetAsync(url).ConfigureAwait(false);
    var stream = await resp.Content.ReadAsStreamAsync().ConfigureAwait(false);
    return await JsonSerializer.DeserializeAsync<T>(stream).ConfigureAwait(false);
}
```

Some teams use `Task.ConfigureAwait(false)` analyzers (e.g., CA2007) to enforce this everywhere. Strict but reliable.

---

## ConfigureAwait(false) is contagious only one level

```csharp
public async Task A() {
    await B().ConfigureAwait(false);
    // I'm on any thread now.
    await C();   // ⚠ captures context AGAIN unless I add ConfigureAwait(false)
}
```

Each `await` has its own context decision. Adding `ConfigureAwait(false)` to the first call doesn't help subsequent awaits in the same method. You need it everywhere.

---

## The `.Result` / `.Wait()` problem

If you blocked the UI thread, ConfigureAwait won't help much beyond the immediate continuation. The async work might still be calling back to your method, and the inner async chain might still capture context.

```csharp
// UI thread
public void OnClick() {
    var data = FetchData().Result;   // ⚠ blocks UI
}

async Task<string> FetchData() {
    return await Helper().ConfigureAwait(false);   // helps here
}

async Task<string> Helper() {
    return await DoIt();   // ⚠ captures context — but we already blocked above
}
```

The chain is only as safe as its weakest link.

**Real fix**: make `OnClick` `async void` and await:

```csharp
public async void OnClick() {
    var data = await FetchData();
    label.Text = data;
}
```

Now `.Result` is gone. The UI stays responsive. Continuations come back to the UI thread naturally to update the label.

---

## ConfigureAwait(false) is not "always faster"

In ASP.NET Core, `ConfigureAwait(false)` is a no-op. Adding it everywhere clutters code without benefit.

The advice "always ConfigureAwait(false)" was for the old ASP.NET (pre-Core) era. In modern Core, library code still benefits (shared libraries should be agnostic), but application code doesn't.

For .NET 8+, the **performance** difference of ConfigureAwait(false) in ASP.NET Core is effectively zero. The behavior is the same (no context to capture). The only reason to write it is "library code that might run somewhere with a context."

---

## Internals — what context capture actually does

When you `await someTask`:

1. The compiler-generated state machine calls `someTask.GetAwaiter().OnCompleted(continuation)` (roughly).
2. The awaiter checks `SynchronizationContext.Current` (and `TaskScheduler.Current` if it differs from the default).
3. If non-null, **captures** it — stores a reference for use when scheduling the continuation.
4. When the task completes, the runtime posts the continuation to the captured context's `Post` method.

`ConfigureAwait(false)`:
- The awaiter ignores `SynchronizationContext.Current`.
- The continuation is queued to the thread pool directly.

UI's SynchronizationContext's `Post` method posts a message to the UI message queue. The UI thread eventually dispatches it.

ASP.NET Core's SynchronizationContext is null — `Post` falls through to "queue on the thread pool."

---

## `SynchronizationContext.Current`

You can inspect (or set) the current context:

```csharp
var ctx = SynchronizationContext.Current;
// In ASP.NET Core / Console: null
// In WinForms: WindowsFormsSynchronizationContext
// In WPF: DispatcherSynchronizationContext
```

For testing async code, you can install a custom context to control where continuations run:

```csharp
SynchronizationContext.SetSynchronizationContext(new MyContext());
```

Most code never does this. Used by some testing frameworks (xUnit's parallel test runner, for example) to control execution order.

---

## Common patterns

### Library method

```csharp
public async Task<string> FetchAsync(string url, CancellationToken ct = default) {
    using var resp = await _http.GetAsync(url, ct).ConfigureAwait(false);
    return await resp.Content.ReadAsStringAsync(ct).ConfigureAwait(false);
}
```

`ConfigureAwait(false)` on every await. Library doesn't care about caller's thread.

### App code (ASP.NET Core)

```csharp
public async Task<IActionResult> Get() {
    var data = await _service.FetchAsync(url);   // no need for ConfigureAwait(false)
    return Ok(data);
}
```

No context, no need.

### App code (WPF / WinForms / Blazor Server)

```csharp
public async Task ButtonClick() {
    var data = await _service.FetchAsync(url);
    label.Text = data;   // we WANT to come back to UI thread
}
```

Default `ConfigureAwait(true)` lets the UI update happen on the right thread. Don't add ConfigureAwait(false) here.

### Mixed

```csharp
public async Task ButtonClick() {
    var data = await _heavyService.ProcessAsync().ConfigureAwait(false);
    // I'm on a thread pool thread now — fine for CPU work
    var result = TransformData(data);
    // To update UI, switch back:
    await _uiDispatcher.SwitchToUiAsync();   // pseudo-API; in real code, use Dispatcher.InvokeAsync
    label.Text = result;
}
```

Used when you specifically want some work off the UI thread.

---

## Common bugs

### Forgetting ConfigureAwait in a library

```csharp
public async Task<int> LibAsync() {
    await SomeAsync();   // captures whatever context the caller had
    return 42;
}
```

Caller in a UI app blocks → deadlock potential. Caller in ASP.NET Core: no problem. Be defensive in libraries.

### Adding ConfigureAwait everywhere in ASP.NET Core

```csharp
[ApiController]
public class Controller : ControllerBase {
    public async Task<IActionResult> Get() {
        var data = await _db.Users.ToListAsync().ConfigureAwait(false);
        return Ok(data);
    }
}
```

Harmless but pointless. Reads as cargo-cult. Skip ConfigureAwait(false) in ASP.NET Core app code.

### Mixing ConfigureAwait(false) with `Dispatcher.Invoke`

```csharp
public async Task M() {
    await DoAsync().ConfigureAwait(false);
    label.Text = "...";    // ⚠ — on a non-UI thread
}
```

After `ConfigureAwait(false)`, you're on whatever thread. UI updates need the UI thread. Either drop the ConfigureAwait(false) here or explicitly marshal back.

---

## Performance

- `ConfigureAwait(false)` saves one `SynchronizationContext.Post` call per await.
- For ASP.NET Core (null context), no perf difference — both paths skip the post.
- For UI apps with hundreds of awaits per operation, a tiny win — usually irrelevant compared to the I/O.

The win is **correctness** (no deadlocks) more than performance.

---

## Summary

- **Libraries**: `ConfigureAwait(false)` everywhere. Always.
- **App code in ASP.NET Core / Console / Worker**: don't bother. No effect.
- **App code in WPF / WinForms / Blazor Server**: don't add it. Default lets continuations return to UI thread.
- The deeper fix for "deadlock" issues is **never block on async** (`.Result` / `.Wait()`). ConfigureAwait(false) only patches half the problem.

→ Next: [07-AsyncStreams.md](07-AsyncStreams.md)
