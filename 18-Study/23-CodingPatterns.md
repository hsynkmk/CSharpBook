# 23 — Coding Patterns (incl. multithreaded problems) ⭐

## ⚡ How to approach any coding problem (say this out loud)

1. **Restate** the problem in your words; **clarify** inputs, edge cases, constraints (empty? nulls? size? sorted?).
2. State the **approach + Big-O** *before* coding ("I'll use a hash map for O(n) time, O(n) space").
3. Write clean code; **talk as you go**.
4. **Test** with a small example + edge cases (empty, single, duplicates, overflow).
5. Mention improvements/trade-offs.

> Interviewers grade *communication + correctness + complexity*, not just a working answer. (See [23-CodingPatterns-Coding.md](23-CodingPatterns-Coding.md) for full practice problems.)

---

## Core algorithmic patterns

**Two pointers** — sorted array / pair problems, O(n):
```csharp
int l = 0, r = a.Length - 1;
while (l < r) {
    int sum = a[l] + a[r];
    if (sum == target) return (l, r);
    if (sum < target) l++; else r--;
}
```

**Sliding window** — subarray/substring with a condition, O(n):
```csharp
var seen = new Dictionary<char,int>(); int start = 0, best = 0;
for (int end = 0; end < s.Length; end++) {
    if (seen.TryGetValue(s[end], out var idx) && idx >= start) start = idx + 1;
    seen[s[end]] = end;
    best = Math.Max(best, end - start + 1);
}
```

**Hash map** — counting/lookup/dedup, O(n):
```csharp
var counts = new Dictionary<int,int>();
foreach (var x in a) counts[x] = counts.GetValueOrDefault(x) + 1;
```

**Binary tree traversal** — recursion (and know the iterative form):
```csharp
int Depth(TreeNode? n) => n is null ? 0 : 1 + Math.Max(Depth(n.Left), Depth(n.Right));
// In-order (BST → sorted): Left, Node, Right.  Level-order: a Queue (BFS).
```

**Recursion vs iteration**: recursion is cleaner for trees/divide-conquer; convert to iteration (explicit stack/queue) if depth risks stack overflow.

---

## 🧵 Multithreaded coding problems (study these hard)

**1. Print alternating numbers (two threads, even/odd in order)** — coordination:
```csharp
var evenTurn = new SemaphoreSlim(1, 1);   // even goes first
var oddTurn  = new SemaphoreSlim(0, 1);
Task Even() => Task.Run(async () => {
    for (int i = 0; i <= n; i += 2) { await evenTurn.WaitAsync(); Console.WriteLine(i); oddTurn.Release(); }
});
Task Odd()  => Task.Run(async () => {
    for (int i = 1; i <= n; i += 2) { await oddTurn.WaitAsync();  Console.WriteLine(i); evenTurn.Release(); }
});
await Task.WhenAll(Even(), Odd());
// Two semaphores ping-pong the turn. (Monitor.Wait/Pulse is the lock-based alternative.)
```

**2. Producer/consumer (bounded buffer)** — backpressure with `Channel<T>`:
```csharp
var ch = Channel.CreateBounded<int>(capacity: 10);   // bounded → producer waits when full
var producer = Task.Run(async () => {
    for (int i = 0; i < 100; i++) await ch.Writer.WriteAsync(i);
    ch.Writer.Complete();
});
var consumer = Task.Run(async () => { await foreach (var item in ch.Reader.ReadAllAsync()) Process(item); });
await Task.WhenAll(producer, consumer);
```

**3. Thread-safe counter** — `Interlocked`, not `lock`, for a single op:
```csharp
private long _count;
public void Inc() => Interlocked.Increment(ref _count);   // atomic, lock-free ([11])
// _count++ across threads is a RACE; volatile alone does NOT fix it.
```

**4. Run N async calls with bounded concurrency** (don't flood):
```csharp
await Parallel.ForEachAsync(urls,
    new ParallelOptions { MaxDegreeOfParallelism = 10 },
    async (url, ct) => await httpClient.GetStringAsync(url, ct));
```

**5. Dining philosophers (deadlock avoidance)** — consistent lock ordering:
```csharp
var (first, second) = left < right ? (left, right) : (right, left);
lock (forks[first]) lock (forks[second]) { Eat(); }   // ordered acquisition prevents the wait-cycle ([11])
```

**6. Async lock / one-at-a-time async section** — can't `lock` across `await`:
```csharp
await _gate.WaitAsync();           // SemaphoreSlim(1,1) as an async mutex
try { await DoCriticalAsync(); }
finally { _gate.Release(); }
```

---

## ❓ Likely questions

**Q: How do you make two threads print numbers in order?**
A: Coordinate turns — two `SemaphoreSlim`s (or `Monitor.Wait`/`Pulse`) where each thread waits for its signal, prints, then releases the other.

**Q: Producer/consumer in .NET?**
A: `Channel<T>` (bounded for backpressure) — thread-safe, async, no manual locking. `BlockingCollection<T>` is the older blocking equivalent.

**Q: Thread-safe increment?**
A: `Interlocked.Increment(ref x)` — atomic, lock-free. `x++` is a race even with `volatile`; `lock` works but is heavier for one op.

**Q: How do you avoid deadlock with multiple locks?**
A: Acquire them in a consistent global order, use lock timeouts (`Monitor.TryEnter`), or reduce to a single lock / lock-free design.

**Q: How do you limit concurrency of many async calls?**
A: `Parallel.ForEachAsync` with `MaxDegreeOfParallelism`, or a `SemaphoreSlim(n)` you `WaitAsync` before each call.

**Q: Iterative vs recursive tree traversal?**
A: Recursion is clean but risks stack overflow on deep trees; convert to iteration with an explicit `Stack` (DFS) or `Queue` (BFS).

**Q: Time/space of your approach?**
A: Always state it — hash-map counting O(n)/O(n); two-pointer on sorted O(n)/O(1); tree traversal O(n) time, O(h) space for recursion.

---

## 🎓 Senior Extra

- **Choosing the coordination primitive**: single atomic op → `Interlocked`; short critical section → `lock`; async section → `SemaphoreSlim.WaitAsync`; producer/consumer → `Channel<T>`; one-time init → `Lazy<T>`; many readers/rare writers → `ReaderWriterLockSlim` ([11](11-Synchronization-and-MemoryModel.md)).
- **Correctness first, then contention**: get it race-free, then minimize the critical section / shard / go lock-free only if profiling shows contention ([21](21-Deployment-Perf-Tooling.md)).
- **`Monitor.Wait/Pulse`** is the classic lock-based condition-variable pattern (alternating-print, bounded-buffer) — know it as the lower-level alternative to semaphores/channels.
- **Cancellation in coordination**: pass a `CancellationToken` to `WaitAsync`/`ReadAllAsync`/`Delay` so workers stop cleanly ([10](10-AsyncAwait.md)).
- **Big-O nuance**: amortized (List append), average vs worst (hash map O(1) vs O(n) with bad hashing — [05](05-Collections.md)), and recursion space (call stack = O(height)).
- **Talk about trade-offs**: brute force → optimal, time vs space, readability vs micro-perf — signals seniority.
- **Pattern map**: pair/sorted → two pointers; contiguous subarray/substring → sliding window; counting/dedup/lookup → hash map; "k largest/stream" → heap (`PriorityQueue`); hierarchical → tree DFS/BFS; combinations → backtracking; shortest path/levels → BFS.

→ Full practice problems with solutions: [23-CodingPatterns-Coding.md](23-CodingPatterns-Coding.md)
