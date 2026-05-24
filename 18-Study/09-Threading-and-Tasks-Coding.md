# 09 — Threading & Tasks — Coding Questions ⭐

> Predict the output / find the bug. (Concepts: [09-Threading-and-Tasks.md](09-Threading-and-Tasks.md))

---

### Q1 — Does Task.Run help here?
```csharp
// I/O-bound:
var data = await Task.Run(async () => await httpClient.GetStringAsync(url));
```
<details><summary>Answer</summary>

**Pointless** — `Task.Run` burns a pool thread just to *start* async I/O that uses no thread while waiting anyway. Just `await httpClient.GetStringAsync(url)`. `Task.Run` is for **CPU-bound** work, not I/O.
</details>

---

### Q2 — What's the output (ordering)?
```csharp
Console.WriteLine("1");
var t = Task.Run(() => Console.WriteLine("2"));
Console.WriteLine("3");
await t;
Console.WriteLine("4");
```
<details><summary>Answer</summary>

**`1`, `3`, `4`** in order, with **`2`** appearing somewhere after `1` (concurrently — could be before or after `3`, but always before `4` since we await `t`). Output is non-deterministic for `2` vs `3`. Common answers: `1 2 3 4` or `1 3 2 4`.
</details>

---

### Q3 — Concurrent or sequential?
```csharp
var sw = Stopwatch.StartNew();
await DelayAsync();   // each takes ~1s
await DelayAsync();
await DelayAsync();
Console.WriteLine(sw.Elapsed.TotalSeconds);   // ~?
static Task DelayAsync() => Task.Delay(1000);
```
<details><summary>Answer</summary>

**~3 seconds** — awaiting each in turn runs them **sequentially**. To run concurrently (~1s): start them first, then `await Task.WhenAll(DelayAsync(), DelayAsync(), DelayAsync())`.
</details>

---

### Q4 — WhenAll result order
```csharp
var tasks = new[] { SlowAsync(3), SlowAsync(1), SlowAsync(2) };
int[] results = await Task.WhenAll(tasks);
Console.WriteLine(string.Join(",", results));
static async Task<int> SlowAsync(int x) { await Task.Delay(x*100); return x; }
```
<details><summary>Answer</summary>

**`3,1,2`** — `WhenAll` preserves the **order of the tasks array**, not completion order. (Even though `1` finishes first, results map to task positions.)
</details>

---

### Q5 — Unobserved exception
```csharp
var t = Task.Run(() => throw new InvalidOperationException("boom"));
Console.WriteLine("started");
// (never await t)
```
<details><summary>Answer</summary>

"started" prints; the exception is **stored on the faulted Task** but never observed (no `await`). It won't crash immediately, but it's a fire-and-forget bug — the failure is silently lost. Always `await` or handle the task. (In old runtimes, unobserved task exceptions could crash on GC.)
</details>

---

### Q6 — TaskCompletionSource
```csharp
var tcs = new TaskCompletionSource<int>();
_ = Task.Run(async () => { await Task.Delay(100); tcs.SetResult(42); });
int result = await tcs.Task;
Console.WriteLine(result);
```
<details><summary>Answer</summary>

**`42`**. `tcs.Task` stays pending until `SetResult(42)` completes it — `TaskCompletionSource` bridges an external signal/callback into an awaitable. Output after ~100ms.
</details>

---

### Q7 — Thread pool starvation: what happens under load?
```csharp
// In an ASP.NET Core controller action, called by 1000 concurrent requests:
public IActionResult Get() {
    var data = _service.GetDataAsync().Result;   // blocks
    return Ok(data);
}
```
<details><summary>Answer</summary>

**Thread-pool starvation** — each request blocks a pool thread on `.Result`. With 1000 concurrent requests blocking faster than the pool injects threads, the queue grows, latency spikes, throughput collapses. **Fix:** `public async Task<IActionResult> Get() => Ok(await _service.GetDataAsync());`
</details>

---

### Q8 — WhenAny
```csharp
var t1 = Task.Delay(1000).ContinueWith(_ => "slow");
var t2 = Task.Delay(100).ContinueWith(_ => "fast");
var first = await Task.WhenAny(t1, t2);
Console.WriteLine(await first);
```
<details><summary>Answer</summary>

**`fast`** — `WhenAny` returns the first task to complete (`t2` after ~100ms). Useful for timeouts (race work against `Task.Delay`). Note the other task keeps running unless cancelled.
</details>

---

### Q9 — Bounded fan-out vs flood
```csharp
// 10,000 URLs:
await Task.WhenAll(urls.Select(u => httpClient.GetStringAsync(u)));   // problem?
```
<details><summary>Answer</summary>

Fires **all 10,000 requests at once** → socket/connection exhaustion, server overload, possible timeouts. **Fix:** bound concurrency — `await Parallel.ForEachAsync(urls, new ParallelOptions{MaxDegreeOfParallelism=20}, async (u,ct) => await httpClient.GetStringAsync(u,ct));`
</details>

---

### Q10 — LongRunning hint (senior)
```csharp
// A method that loops forever doing blocking work:
Task.Run(() => { while (true) { BlockingPoll(); } });
```
<details><summary>Answer</summary>

This **occupies a thread-pool thread forever** → fewer threads for other work (potential starvation). **Fix:** `new Thread(...) { IsBackground = true }.Start();` or `Task.Factory.StartNew(..., TaskCreationOptions.LongRunning)` to get a dedicated thread off the pool. Long-running blocking loops shouldn't live on the pool.
</details>
