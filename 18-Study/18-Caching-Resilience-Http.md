# 18 — Caching, Resilience & HttpClient

## ⚡ 30-second answer

**Caching** trades freshness for speed/load: **`IMemoryCache`** (in-process, fast, per-instance), **`IDistributedCache`** (Redis/SQL, shared across instances), **`HybridCache`** (.NET 9, combines both + built-in **stampede** protection). For outbound HTTP, never `new HttpClient()` per call — use **`IHttpClientFactory`** (manages connection pooling/handler lifetime, avoids socket exhaustion). For unreliable dependencies, add **resilience** with **Polly** (`Microsoft.Extensions.Resilience`): **retry** with exponential backoff + **jitter**, **circuit breaker**, and **timeout**. The theme: networks fail — design for transient failure and don't hammer a struggling service.

---

## Core mechanics

**Caching**:
```csharp
var data = await cache.GetOrCreateAsync(key, async e => {
    e.AbsoluteExpirationRelativeToNow = TimeSpan.FromMinutes(5);
    return await LoadAsync();
});
```
- **`IMemoryCache`** — local, fastest, but per-instance (not shared; lost on restart).
- **`IDistributedCache`** — Redis/SQL; shared across a scaled-out app; serialized (slower).
- **`HybridCache`** — L1 (memory) + L2 (distributed) with **stampede protection** and tag-based invalidation.

**HttpClient — the right way**:
```csharp
builder.Services.AddHttpClient<ApiClient>(c => c.BaseAddress = new("https://api…"))
    .AddStandardResilienceHandler();      // retry + circuit breaker + timeouts in one line
```
`IHttpClientFactory` pools `HttpMessageHandler`s (reusing connections) and recycles them periodically (picks up DNS changes) — solving both **socket exhaustion** (from `new HttpClient` per request) and **stale DNS** (from a single static client).

**Polly resilience strategies**:
```csharp
new ResiliencePipelineBuilder()
    .AddRetry(new() { MaxRetryAttempts = 3, BackoffType = DelayBackoffType.Exponential, UseJitter = true })
    .AddCircuitBreaker(new() { FailureRatio = 0.5, BreakDuration = TimeSpan.FromSeconds(15) })
    .AddTimeout(TimeSpan.FromSeconds(10))
    .Build();
```

---

## Comparison tables

| Cache | Scope | Speed | Survives restart / shared? |
|---|---|---|---|
| `IMemoryCache` | in-process | fastest | no |
| `IDistributedCache` (Redis) | shared | network hop | yes |
| `HybridCache` | both (L1+L2) | fast (L1) + shared (L2) | yes + stampede protection |

| Polly strategy | Handles | Behavior |
|---|---|---|
| **Retry** (+ backoff + jitter) | transient blips | re-attempt, spacing out |
| **Circuit breaker** | sustained outage | trip open → fail fast, let it recover |
| **Timeout** | slow calls | cancel after a deadline |
| **Fallback** | unrecoverable | return a default/degraded result |
| **Rate limiter / bulkhead** | overload | bound concurrency |

---

## 🪤 Traps & gotchas

- **`new HttpClient()` per request** → **socket exhaustion** (sockets stuck in TIME_WAIT). Use `IHttpClientFactory`.
- **One static `HttpClient` forever** → **stale DNS** (never sees IP changes). The factory recycles handlers to fix this.
- **Retry without idempotency** → duplicates: if a POST succeeded but the response was lost, retrying charges twice. Retry only idempotent ops, or use an **idempotency key** ([20](20-Observability-Messaging-Background.md)).
- **Retry without jitter** → **thundering herd**: many clients retry in lockstep and crush the recovering service. Always add jitter.
- **Retry without a circuit breaker** → retrying a *sustained* outage piles load on a down service. Combine: retry for blips, breaker for outages.
- **Cache stampede**: when a hot key expires, many concurrent requests all recompute it at once. Use `HybridCache` (built-in protection) or a lock/`SemaphoreSlim`.
- **Caching without invalidation** → stale data bugs; set sensible expiry and invalidate on writes.
- **Building a Polly pipeline per call** resets circuit-breaker state (it must persist across calls) — register/build once and reuse.

---

## ❓ Likely questions

**Q: Why not `new HttpClient()` per request?**
A: Each disposes a handler holding sockets that linger in TIME_WAIT → socket exhaustion under load. `IHttpClientFactory` pools and recycles handlers, fixing that and stale DNS.

**Q: `IMemoryCache` vs `IDistributedCache`?**
A: Memory cache is in-process (fastest, not shared, lost on restart); distributed cache (Redis) is shared across instances and survives restarts but has a network/serialization cost. `HybridCache` combines both.

**Q: What is a cache stampede and how do you prevent it?**
A: Many requests recomputing the same expired hot key simultaneously, overloading the source. Prevent with `HybridCache`'s built-in protection or a lock/semaphore so only one recomputes.

**Q: How does a circuit breaker work?**
A: It tracks failures; when they exceed a threshold it "trips open" and fails calls instantly (without calling the dependency) for a cooldown, then half-opens to test recovery. Prevents cascading failure.

**Q: Why add jitter to retries?**
A: Without it, many clients that failed together retry at the same instants (thundering herd), re-overloading the recovering service. Jitter randomizes delays so retries spread out.

**Q: When is retrying dangerous?**
A: For non-idempotent operations — if the first attempt succeeded but the response was lost, the retry duplicates it (double charge). Retry only idempotent ops or use an idempotency key.

**Q: What's the standard resilience setup for HttpClient?**
A: `AddHttpClient<T>().AddStandardResilienceHandler()` — a pre-tuned Polly pipeline (total timeout → retry → circuit breaker → per-attempt timeout).

---

## 🎓 Senior Extra

- **Strategy ordering matters** in Polly: standard order is **total timeout → retry → circuit breaker → per-attempt timeout → call**. Breaker *inside* vs *outside* retry behaves very differently.
- **Circuit breaker state is shared** across calls (the point) — build the pipeline once (DI), one breaker **per dependency** (B's failures shouldn't open C's circuit).
- **Idempotency keys + dedup** make retries safe for writes; combine with the **outbox pattern** for reliable messaging ([20](20-Observability-Messaging-Background.md)).
- **Cache invalidation strategies**: absolute vs sliding expiry, tag-based eviction (HybridCache), write-through vs cache-aside; correctness is the hard part.
- **Distributed cache serialization** cost matters — keep cached payloads small; consider compression; Redis data types/TTL.
- **Resilience telemetry**: Polly/`Microsoft.Extensions.Resilience` emit metrics (retry counts, breaker state) — wire to OpenTelemetry so retry spikes/open circuits are visible ([20](20-Observability-Messaging-Background.md)).
- **Typed clients vs named/basic**: `AddHttpClient<T>` gives a strongly-typed client with DI; add `DelegatingHandler`s for auth/logging/correlation propagation.
- **Hedging** (Polly v8): race a second attempt for tail-latency-sensitive reads — advanced, idempotent-only.

→ Deeper: [`../DotNetBook/06-DataAndCaching/`](../DotNetBook/06-DataAndCaching/README.md), [`../DotNetBook/09-NetworkingAndHttp/`](../DotNetBook/09-NetworkingAndHttp/README.md), [`../DotNetBook/11-Resilience/`](../DotNetBook/11-Resilience/README.md)
