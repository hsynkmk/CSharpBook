# RapidFire — One-Line Q&A (self-quiz)

> Cover the answers, read each Q, answer **out loud**, star the ones you miss. Concurrency is over-represented on purpose.

## C# Type System & OOP

1. **Value vs reference type?** Value holds data directly & copies by value; reference holds a pointer to a heap object & copies the reference.
2. **Is `string` value or reference?** Reference, but immutable with value-based `==`.
3. **What is boxing?** Wrapping a value type in a heap object to use as `object`/interface — allocates.
4. **Avoid boxing how?** Generics (`List<int>` never boxes).
5. **`struct` default equality?** Value (field-by-field), but slow (reflection) unless `IEquatable<T>`/`record struct`.
6. **`default(int)` vs `default(string)`?** `0` vs `null`.
7. **Does `int?` allocate?** No — `Nullable<int>` is a struct.
8. **`const` vs `readonly`?** `const` = compile-time, baked into callers; `readonly` = set at runtime (ctor), per-instance.
9. **`override` vs `new`?** `override` = runtime virtual dispatch; `new` = compile-time hiding (declared type wins, not polymorphic).
10. **Are methods virtual by default?** No — opt-in with `virtual`.
11. **Abstract class vs interface?** Abstract = shared state/impl, single inheritance; interface = contract, multiple, no instance state.
12. **Why not call virtual methods in a ctor?** Derived override runs before derived fields are initialized.
13. **Access modifiers?** public, private, protected, internal, protected internal (OR), private protected (AND), file.
14. **Four OOP pillars?** Encapsulation, inheritance, polymorphism, abstraction.
15. **`sealed` benefit?** Enables devirtualization/inlining + expresses intent.

## Generics, Delegates, LINQ

16. **Covariance vs contravariance?** `out` = use more-derived T (output); `in` = use more-base T (input); reference types only.
17. **Func vs Action vs Predicate?** Func returns a value; Action returns void; Predicate = Func<T,bool>.
18. **What is a closure?** A lambda + captured variables (by reference, hoisted to heap).
19. **Loop capture bug?** A `for` loop's `() => i` all capture the same `i`; copy to a per-iteration local (`foreach` is fine).
20. **Delegate vs event?** Event restricts outsiders to `+=`/`-=` (can't invoke/overwrite).
21. **Deferred execution?** LINQ operators build a pipeline that runs only when enumerated.
22. **IEnumerable vs IQueryable?** In-memory (delegates) vs translated to SQL (expression trees, runs in DB).
23. **Multiple enumeration risk?** A deferred query re-runs each time iterated (repeat DB hits) — `ToList()` to materialize once.
24. **`Any()` vs `Count() > 0`?** `Any()` short-circuits (EXISTS); `Count()` may count all.
25. **`First` vs `Single`?** `First` = first (throws if none); `Single` = exactly one (throws if 0 or >1).
26. **`Select` vs `SelectMany`?** Project 1→1 vs flatten 1→many.

## Collections & Equality

27. **Dictionary lookup Big-O?** Average O(1); O(n) worst with bad hashing.
28. **How does Dictionary work?** Hash the key → bucket → resolve collisions by chaining → compare with `Equals`.
29. **List vs LinkedList?** Array-backed (O(1) index, cache-friendly) vs O(1) node insert but O(n) search. Prefer List.
30. **HashSet vs List for Contains?** O(1) vs O(n).
31. **List capacity growth?** Doubles, copying the array — amortized O(1) append.
32. **Override `Equals` — what else?** `GetHashCode` (equal objects ⇒ equal hash).
33. **`==` vs `Equals`?** `==` static/compile-time (default reference for classes); `Equals` virtual/runtime.
34. **GetHashCode contract?** Equal ⇒ equal hash; collisions allowed; stable while used as a key.
35. **Mutable dictionary key danger?** If its hash changes after insert, you can never find it.
36. **Concurrent vs Immutable vs Frozen?** Safe mutation / never-changes (copy on edit) / build-once-read-fast.

## ⭐ Async & Threading

37. **Thread vs Task?** Thread = heavy OS unit; Task = lightweight work on the reused thread pool.
38. **`Task.Run` vs `async/await`?** `Task.Run` = offload **CPU** work; `async/await` = **I/O** (no thread blocked while waiting).
39. **Does `await` use a thread while waiting on I/O?** No — it suspends; no thread consumed.
40. **What does `await` do under the hood?** Builds a state machine; returns to caller and resumes via a continuation when the awaited op completes.
41. **Why does `.Result` deadlock?** Blocks the thread the continuation needs (sync-context) → both wait forever.
42. **Sync-over-async in ASP.NET Core?** Thread-pool starvation (blocked pool threads) under load.
43. **`async void`?** Can't be awaited; exceptions crash the process. Only for event handlers.
44. **`ConfigureAwait(false)` — where?** In **library** code; no-op in ASP.NET Core.
45. **`Task` vs `ValueTask`?** ValueTask avoids allocation when often-synchronous; await once, no early `.Result`.
46. **`lock` vs `Interlocked` vs `volatile`?** Critical section / atomic single op / visibility of one flag (not atomic).
47. **Can you `await` inside `lock`?** No (monitor is thread-affine) — use `SemaphoreSlim.WaitAsync`.
48. **What to lock on?** A private readonly dedicated object — never `this`/`typeof`/strings.
49. **Is `volatile int x; x++` atomic?** No — read-modify-write race; use `Interlocked.Increment`.
50. **Race condition?** Unsynchronized shared mutation where the result depends on timing.
51. **Deadlock & prevention?** Circular lock waits; fix with consistent lock ordering / timeouts / fewer locks.
52. **Thread-pool starvation symptom?** Growing queue length + thread count (dotnet-counters) under blocking load.
53. **`ConcurrentDictionary` safe how?** Lock-free reads + lock-striped writes; `GetOrAdd` factory may run >1x.
54. **Producer/consumer in .NET?** `Channel<T>` (bounded = backpressure), async-first.
55. **Limit concurrency of N async calls?** `Parallel.ForEachAsync` (MaxDegreeOfParallelism) or `SemaphoreSlim`.
56. **`Task.WhenAll` multiple exceptions?** `await` surfaces the first; inspect `task.Exception.InnerExceptions`.
57. **`SemaphoreSlim` vs `Mutex`?** In-process/async/N-holders vs cross-process/OS-level.
58. **CAS?** `Interlocked.CompareExchange` — swap only if unchanged; basis of lock-free.
59. **`Thread.Sleep` in async code?** Never — it blocks a thread; use `await Task.Delay`.
60. **Fire-and-forget risk?** Lost exceptions, broken ordering/lifecycle; await or manage it.

## Memory & GC

61. **How does GC decide what to collect?** Traces from roots; unreachable = garbage. Generational + compacting.
62. **Why generations?** Most objects die young — Gen0 collected often/cheaply, Gen2 rarely.
63. **LOH threshold?** ≥85KB; collected with Gen2, not compacted (fragments) — pool buffers.
64. **Can you leak in a GC'd runtime?** Yes — unintended reachability (events, statics, caches, closures, timers).
65. **`IDisposable` for?** Deterministic release of unmanaged/external resources (`using`).
66. **When write a finalizer?** Rarely — prefer `SafeHandle`; finalizers delay collection, non-deterministic.
67. **`GC.SuppressFinalize` in Dispose — why?** Avoids the object surviving an extra GC for finalization.
68. **`Span<T>` vs `Memory<T>`?** Span = stack-only (no field/async); Memory = heap-storable, yields a Span.
69. **Should you call `GC.Collect()`?** No (except niche) — the GC self-tunes.
70. **Server vs Workstation GC?** Server = per-core heaps/threads, throughput (default ASP.NET Core); Workstation = low latency.
71. **#1 GC lever?** Reduce allocation rate (Span/pooling/structs/avoid LINQ in hot loops).
72. **Find a leak how?** `dotnet-dump` → `dumpheap -stat` → `gcroot`.

## Runtime / Platform

73. **Is C# interpreted?** No — compiled to IL, JIT-compiled to native at runtime.
74. **Tiered compilation?** Tier-0 fast (startup), tier-1 optimizes hot methods (with OSR/PGO).
75. **JIT vs AOT vs R2R?** Runtime compile (best steady-state) / native AOT (fast startup, restricted) / precompiled + JIT (fast startup, no restrictions).
76. **CLR provides?** GC, type system, JIT, exceptions, thread pool, metadata/reflection, assembly loading.
77. **Native AOT restriction?** No runtime codegen, limited reflection, trim-only — use source generators.

## DI / Hosting / Config

78. **Three DI lifetimes?** Singleton (app), Scoped (request), Transient (each resolution).
79. **Captive dependency?** A longer-lived service capturing a shorter-lived one (Singleton holding Scoped) — corruption/leak.
80. **`DbContext` lifetime?** Scoped (not thread-safe); `IDbContextFactory` for parallel/background.
81. **`IOptions` vs `Snapshot` vs `Monitor`?** Read-once singleton / per-request scoped / live singleton (use Monitor in singletons).
82. **Config precedence?** Later providers win: appsettings → env-specific → user secrets → env vars → command line.
83. **Service locator — why avoid?** Hides dependencies, hurts testability; use constructor injection.

## ASP.NET Core / EF Core

84. **Auth middleware order?** Routing → Authentication → Authorization → endpoints (exception handler first).
85. **Middleware vs filters?** Every request (pipeline) vs around MVC actions (model/action context).
86. **Minimal API vs MVC?** Lightweight function endpoints vs controllers + full filter pipeline.
87. **ProblemDetails?** RFC 7807 standard error response.
88. **What patterns does `DbContext` implement?** Unit of Work + Repository (`DbSet`).
89. **N+1 problem?** 1 query + N lazy loads in a loop; fix with `Include`/projection.
90. **`AsNoTracking` when?** Read-only queries (skips change tracking).
91. **Optimistic concurrency?** A `RowVersion` checked in the UPDATE; conflict → `DbUpdateConcurrencyException`.
92. **Bulk update efficiently?** `ExecuteUpdate`/`ExecuteDelete` (no load).

## Caching / Resilience / Security

93. **Why `IHttpClientFactory`?** Avoids socket exhaustion (`new HttpClient` per call) and stale DNS (one static client).
94. **`IMemoryCache` vs `IDistributedCache`?** In-process/fast/not-shared vs Redis/shared/survives restart.
95. **Cache stampede fix?** `HybridCache` built-in protection or a lock/semaphore around recompute.
96. **Circuit breaker?** Trips open after sustained failures → fail fast → half-open to test recovery.
97. **Why jitter on retries?** Prevents synchronized retry storms (thundering herd).
98. **Retry danger?** Non-idempotent ops duplicate; use idempotency keys.
99. **Authn vs authz?** Who you are vs what you're allowed; authn first.
100. **JWT validation checks?** Signature, issuer, audience, expiry.
101. **OAuth2 vs OIDC?** Authorization (access token) vs authentication (id token).
102. **JWT vs cookie?** Stateless header token (APIs) vs auto-sent encrypted ticket (web, CSRF-prone).
103. **How store passwords?** Hashed with a salted slow KDF (PBKDF2/bcrypt/Argon2) — never plaintext/encrypted.
104. **Where store secrets?** User-secrets (dev), Key Vault + managed identity (prod) — never in code.

## Observability / Messaging / Patterns

105. **Three pillars of observability?** Logs, metrics, traces.
106. **Why structured logging?** Named fields are queryable (`{OrderId}`) vs concatenated strings.
107. **Why idempotent handlers?** At-least-once delivery → messages can repeat.
108. **Outbox pattern solves?** Dual-write — atomic DB change + reliable publish.
109. **Liveness vs readiness?** Process up (restart) vs can-serve incl. deps (route traffic).
110. **Background work in ASP.NET Core?** `BackgroundService`/`IHostedService` with a DI scope per item.
111. **SOLID?** Single responsibility, Open/closed, Liskov, Interface segregation, Dependency inversion.
112. **DIP vs DI?** Principle (depend on abstractions) vs technique (inject them).
113. **CQRS when overkill?** Simple CRUD where read/write models don't diverge.
114. **Repository over EF Core needed?** Usually no — DbContext is already UoW+repo.
115. **Singleton pattern vs DI singleton?** Prefer DI singleton lifetime (injectable/testable) over a static instance.

## Perf / Tooling / Coding

116. **#1 perf rule?** Measure, don't guess — profile, fix, re-measure.
117. **BenchmarkDotNet over Stopwatch?** Handles JIT warmup, dead-code elimination, GC noise, allocations.
118. **Live triage tool?** `dotnet-counters` (GC, allocation, thread-pool queue, exceptions).
119. **Most common perf issues?** Allocations (GC), N+1 queries, sync-over-async starvation.
120. **Two threads print in order — how?** Two `SemaphoreSlim`s ping-ponging the turn (or `Monitor.Wait`/`Pulse`).
121. **Thread-safe counter?** `Interlocked.Increment` (not `x++`, not just `volatile`).
122. **Avoid deadlock with 2 locks?** Acquire in a consistent global order.
123. **Coding approach?** Restate → clarify → state approach + Big-O → code aloud → test edges.
124. **STAR?** Situation, Task, Action (the bulk, "I"), Result (quantified).
