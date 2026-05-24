# 23 — Coding Patterns — Practice Problems ⭐

> Full problems with solutions + complexity. Attempt before revealing. Multithreaded problems emphasized (the interview's focus). (Patterns: [23-CodingPatterns.md](23-CodingPatterns.md))

---

### P1 — Two Sum (hash map)
Return indices of two numbers summing to `target`.
<details><summary>Solution — O(n) time, O(n) space</summary>

```csharp
int[] TwoSum(int[] nums, int target) {
    var seen = new Dictionary<int,int>();          // value → index
    for (int i = 0; i < nums.Length; i++) {
        if (seen.TryGetValue(target - nums[i], out var j)) return [j, i];
        seen[nums[i]] = i;
    }
    return [];
}
```
One pass; for each element check if its complement was seen. Brute force is O(n²) — mention then improve.
</details>

---

### P2 — Longest substring without repeating chars (sliding window)
<details><summary>Solution — O(n) time</summary>

```csharp
int LengthOfLongest(string s) {
    var last = new Dictionary<char,int>(); int start = 0, best = 0;
    for (int end = 0; end < s.Length; end++) {
        if (last.TryGetValue(s[end], out var idx) && idx >= start) start = idx + 1;
        last[s[end]] = end;
        best = Math.Max(best, end - start + 1);
    }
    return best;
}
```
Window `[start, end]`; jump `start` past the last duplicate.
</details>

---

### P3 — Reverse a linked list (pointers)
<details><summary>Solution — O(n) time, O(1) space</summary>

```csharp
Node? Reverse(Node? head) {
    Node? prev = null;
    while (head != null) { var next = head.Next; head.Next = prev; prev = head; head = next; }
    return prev;
}
```
Iterative pointer flip. Recursive version is O(n) stack.
</details>

---

### P4 — Validate a BST (tree + bounds)
<details><summary>Solution — O(n)</summary>

```csharp
bool IsValid(TreeNode? n, long min = long.MinValue, long max = long.MaxValue) =>
    n == null || (n.Val > min && n.Val < max
        && IsValid(n.Left, min, n.Val) && IsValid(n.Right, n.Val, max));
```
Pass down allowed `(min, max)` bounds — each node must fit, not just be greater than its parent.
</details>

---

### 🧵 P5 — Two threads print 1..N alternately (odd/even)
<details><summary>Solution — semaphore ping-pong</summary>

```csharp
async Task PrintAlternating(int n) {
    var evenTurn = new SemaphoreSlim(1, 1);   // even (0) first
    var oddTurn  = new SemaphoreSlim(0, 1);
    var even = Task.Run(async () => {
        for (int i = 0; i <= n; i += 2) { await evenTurn.WaitAsync(); Console.WriteLine(i); oddTurn.Release(); }
    });
    var odd = Task.Run(async () => {
        for (int i = 1; i <= n; i += 2) { await oddTurn.WaitAsync(); Console.WriteLine(i); evenTurn.Release(); }
    });
    await Task.WhenAll(even, odd);
}
```
Two semaphores hand the turn back and forth. (Lock-based alternative: `Monitor.Wait`/`Pulse` on a shared lock with a turn flag.)
</details>

---

### 🧵 P6 — Bounded producer/consumer (backpressure)
<details><summary>Solution — Channel&lt;T&gt;</summary>

```csharp
async Task Run() {
    var ch = Channel.CreateBounded<int>(10);                 // bounded → backpressure
    var producer = Task.Run(async () => {
        for (int i = 0; i < 1000; i++) await ch.Writer.WriteAsync(i);   // awaits when full
        ch.Writer.Complete();
    });
    var consumers = Enumerable.Range(0, 4).Select(_ => Task.Run(async () => {
        await foreach (var item in ch.Reader.ReadAllAsync()) Process(item);   // multiple consumers
    }));
    await Task.WhenAll(consumers.Prepend(producer));
}
```
`Channel<T>` gives thread-safe handoff, backpressure, and multiple consumers — no manual locking. `Complete()` ends the readers.
</details>

---

### 🧵 P7 — Thread-safe counter / aggregate
<details><summary>Solution — Interlocked (lock-free)</summary>

```csharp
long _total = 0;
void Add(int x) => Interlocked.Add(ref _total, x);
// Parallel sum with no lock:
long ParallelSum(int[] data) {
    long sum = 0;
    Parallel.ForEach(data,
        () => 0L,                                   // thread-local seed
        (x, _, local) => local + x,                 // accumulate locally (no contention)
        local => Interlocked.Add(ref sum, local));  // merge once per thread
    return sum;
}
```
`x++`/`sum += x` across threads is a **race**. Use `Interlocked`, and the thread-local accumulate pattern to minimize contention.
</details>

---

### 🧵 P8 — Run N async calls, max K concurrent
<details><summary>Solution — SemaphoreSlim throttle (or Parallel.ForEachAsync)</summary>

```csharp
async Task<string[]> FetchAll(string[] urls, int k) {
    var sem = new SemaphoreSlim(k);
    var tasks = urls.Select(async url => {
        await sem.WaitAsync();
        try { return await _http.GetStringAsync(url); }
        finally { sem.Release(); }
    });
    return await Task.WhenAll(tasks);
}
// Or simply:
await Parallel.ForEachAsync(urls, new ParallelOptions{ MaxDegreeOfParallelism = k },
    async (url, ct) => await _http.GetStringAsync(url, ct));
```
Bounds concurrency to `k` (avoids flooding). `Release()` in `finally`.
</details>

---

### 🧵 P9 — Dining philosophers (no deadlock)
<details><summary>Solution — ordered lock acquisition</summary>

```csharp
void Philosopher(int i, object[] forks) {
    int left = i, right = (i + 1) % forks.Length;
    var (a, b) = left < right ? (left, right) : (right, left);   // GLOBAL order
    lock (forks[a]) lock (forks[b]) { Eat(); }
}
```
Always grab the lower-indexed fork first → breaks the circular-wait condition → no deadlock. (Alternative: a waiter/`SemaphoreSlim` limiting concurrent diners to N-1.)
</details>

---

### 🧵 P10 — Async lazy singleton (thread-safe, one init)
<details><summary>Solution — Lazy&lt;Task&lt;T&gt;&gt;</summary>

```csharp
private static readonly Lazy<Task<Config>> _config =
    new(() => LoadConfigAsync());                 // factory runs once
public static Task<Config> GetConfigAsync() => _config.Value;
```
`Lazy<Task<T>>` ensures the async factory runs **exactly once** and all callers await the same task — thread-safe lazy async init without locks or double-init. (Plain `ConcurrentDictionary.GetOrAdd` factory can run >1×; this doesn't.)
</details>

---

### Pattern recognition cheat
| Clue | Pattern |
|---|---|
| sorted array / pair | two pointers |
| contiguous subarray/substring + condition | sliding window |
| count / dedup / lookup | hash map |
| k largest/smallest / streaming top-k | heap (`PriorityQueue`) |
| hierarchical / levels | tree DFS/BFS |
| "in order" across threads | semaphore handoff / Monitor.Wait-Pulse |
| producer/consumer | `Channel<T>` |
| shared counter | `Interlocked` |
| bound concurrency | `SemaphoreSlim` / `Parallel.ForEachAsync` |
| avoid deadlock | consistent lock ordering |
