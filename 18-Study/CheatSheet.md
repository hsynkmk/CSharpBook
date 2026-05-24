# CheatSheet — Last-Minute Scan

> Read this the morning of. All the comparison tables + traps + "say this" in one place.

---

## 🗣️ How to answer (always)

- **Restate → clarify → think aloud → answer the direct question first, then add depth.**
- "X vs Y" → one-sentence difference → when to use each → a trap.
- Coding → approach + **Big-O** before typing → test edge cases after.
- Don't bluff internals — reason aloud, state assumptions.

---

## ⭐ Concurrency — the most-tested area

| Use | Pick |
|---|---|
| CPU-bound offload | `Task.Run` |
| I/O-bound | `async/await` (no thread blocked) |
| atomic single op | `Interlocked` |
| critical section | `lock` (private readonly object) |
| async critical section | `SemaphoreSlim.WaitAsync` |
| flag visibility | `volatile` (not atomic!) |
| producer/consumer | `Channel<T>` (bounded = backpressure) |
| bounded async fan-out | `Parallel.ForEachAsync` |
| cross-process lock | `Mutex` |

**The deadly trio (have crisp answers):**
- **Sync-over-async** (`.Result`/`.Wait()`) → deadlock (sync context) / thread-pool starvation (ASP.NET Core). Fix: async all the way.
- **`async void`** → uncatchable exceptions, crashes process. Fix: `async Task`.
- **Deadlock** → circular lock waits. Fix: consistent lock ordering / timeouts / fewer locks.

**Remember:** `await` ≠ new thread (it *suspends*). `volatile` ≠ atomic (`x++` still races). Can't `await` inside `lock`. `Interlocked` for counters.

---

## Value vs Reference

| | Value (struct) | Reference (class) |
|---|---|---|
| Holds | data | pointer to heap object |
| Copy | bytes | reference |
| Default | zeroed | null |
| Equality | structural | identity |

**Boxing** = value type → heap object (allocates). Avoid with generics. `string` = reference but value `==`.

## override vs new

`override` = runtime virtual dispatch (object type wins). `new` = compile-time hiding (declared type wins, **not** polymorphic).

## Abstract class vs Interface

Abstract = shared state/impl, single inheritance. Interface = contract, multiple, no instance state (DIMs exist).

## Equals / GetHashCode

Override **both together**. Equal ⇒ equal hash. Records generate value equality. Mutable key with changing hash = lost in dictionary.

## LINQ

Deferred until enumerated. `IEnumerable` (memory) vs `IQueryable` (→ SQL). Multiple enumeration re-runs (`ToList` once). `Any()` > `Count()>0`.

## Collections Big-O

| | lookup | insert |
|---|---|---|
| `List<T>` | O(1) idx / O(n) val | O(1)* append |
| `Dictionary`/`HashSet` | O(1) | O(1)* |
| `SortedDictionary` | O(log n) | O(log n) |

Dictionary = hash → bucket → chain → `Equals`. Not thread-safe (use `ConcurrentDictionary`).

## GC / Memory

Generational (0/1/2) + compacting; traces from roots; unreachable = garbage. **LOH ≥85KB** (not compacted). Leak = unintended reachability (events/statics/caches/closures). `IDisposable`+`using` for unmanaged resources; prefer `SafeHandle` over finalizers. Reduce **allocation rate** for perf.

## DI lifetimes

Singleton (app, thread-safe) / Scoped (request, e.g. DbContext) / Transient (each). **Captive dependency** = Singleton holding Scoped → corruption. `IOptionsMonitor` in singletons (not `IOptionsSnapshot`).

## ASP.NET Core

Pipeline order: **ExceptionHandler → Routing → Authentication → Authorization → Endpoints**. Middleware = every request; filters = around actions. Minimal API (light) vs MVC (full). Errors → ProblemDetails.

## EF Core

`DbContext` = Unit of Work + Repository, **scoped**. **N+1** → `Include`/projection. `AsNoTracking` for reads. `RowVersion` for optimistic concurrency. `ExecuteUpdate` for bulk.

## Caching / HTTP / Resilience

`IHttpClientFactory` (not `new HttpClient`) → no socket exhaustion / stale DNS. Cache: Memory (local) / Distributed (Redis) / Hybrid (+stampede). Polly: **retry (backoff+jitter) → circuit breaker → timeout**. Retry only idempotent ops.

## Security

Authn (who) before Authz (what). JWT: validate **signature/iss/aud/exp**, stateless. OAuth2 (authz) vs OIDC (authn). Hash passwords (salted KDF). Secrets → Key Vault + managed identity. Cookies → CSRF (anti-forgery).

## Runtime

C# → IL → JIT → native. Tiered (tier-0 startup, tier-1 hot). AOT = fast startup, no reflection-emit. CLR = GC + types + JIT + thread pool.

## SOLID

**S**ingle responsibility, **O**pen/closed, **L**iskov, **I**nterface segregation, **D**ependency inversion. Compose over inherit. DI realizes DIP.

## Observability / Messaging

3 pillars: logs/metrics/traces (OpenTelemetry). Structured logging (named fields). At-least-once → **idempotent** handlers. **Outbox** = atomic DB+publish. Liveness (restart) vs readiness (route).

---

## 🪤 Top traps (don't fall for these)

1. `.Result`/`.Wait()` on async → deadlock/starvation.
2. `async void` (except event handlers).
3. `override` vs `new` confusion.
4. Override `Equals` without `GetHashCode`.
5. `volatile` ≠ atomic; `x++` races.
6. `lock(this)`/`lock(typeof(X))` → use a private object.
7. Captive dependency (Singleton ← Scoped).
8. N+1 queries; tracking read-only queries.
9. `new HttpClient()` per request.
10. Retry without idempotency/jitter.
11. Middleware order (authz before authn).
12. Multiple enumeration of a deferred LINQ query.
13. Mutable dictionary key.
14. Swallowing exceptions (`catch {}`); `throw ex;` (use `throw;`).
15. Microservices/optimization before measuring.

---

## 💬 Power phrases (signal seniority)

- "It depends on the access pattern / scale — let me clarify…"
- "The trade-off here is X vs Y; I'd choose … because …"
- "I'd **measure** first with `dotnet-counters`/a profiler before optimizing."
- "That's at-least-once delivery, so the handler needs to be **idempotent**."
- "I'd keep it async all the way to avoid thread-pool starvation."
- "Default to a monolith; extract services only when the complexity is justified."
- "`DbContext` is already a Unit of Work, so I wouldn't add a generic repository."

---

## ✅ 5-step coding protocol

1. Restate + clarify (edge cases, constraints).
2. State approach + **Big-O**.
3. Code cleanly, talk through it.
4. Test: empty / single / duplicates / overflow.
5. Note improvements & trade-offs.

## 🌟 STAR (behavioral)

**S**ituation (brief) → **T**ask (brief) → **A**ction (**the bulk, say "I"**) → **R**esult (**quantified**). Have 5 stories: tech challenge (use a concurrency/perf one), ownership, conflict, failure, learning.
