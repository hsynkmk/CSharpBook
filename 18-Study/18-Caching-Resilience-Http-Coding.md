# 18 — Caching, Resilience & HttpClient — Coding Questions

> Find the bug / predict. (Concepts: [18-Caching-Resilience-Http.md](18-Caching-Resilience-Http.md))

---

### Q1 — Socket exhaustion
```csharp
public async Task<string> GetAsync(string url) {
    using var client = new HttpClient();      // ?
    return await client.GetStringAsync(url);
}
// called thousands of times
```
<details><summary>Answer</summary>

**Socket exhaustion** — each `new HttpClient()`/dispose leaves sockets in TIME_WAIT; under load you run out of ports. **Fix:** inject and reuse via **`IHttpClientFactory`** (`AddHttpClient<T>()`), which pools and recycles handlers (also fixing stale DNS).
</details>

---

### Q2 — Stale DNS
```csharp
static readonly HttpClient _client = new();   // one static client forever
```
<details><summary>Answer</summary>

Avoids socket exhaustion but **never picks up DNS changes** (the handler caches connections forever) — a problem when the backend's IP changes (failover, scaling). `IHttpClientFactory` periodically recycles handlers to refresh DNS. (Or set `PooledConnectionLifetime` on a `SocketsHttpHandler`.)
</details>

---

### Q3 — Cache stampede
```csharp
async Task<Data> GetAsync() {
    if (!_cache.TryGetValue("key", out Data? d)) {
        d = await LoadExpensiveAsync();   // 2s
        _cache.Set("key", d, TimeSpan.FromMinutes(5));
    }
    return d!;
}
// 500 requests hit a cold/expired key simultaneously
```
<details><summary>Answer</summary>

**Stampede** — all 500 see the miss and call `LoadExpensiveAsync` at once, hammering the source. **Fix:** `HybridCache` (built-in stampede protection) or a `SemaphoreSlim` so only one recomputes while others await the result.
</details>

---

### Q4 — Retry without idempotency
```csharp
pipeline.Execute(() => httpClient.PostAsync("/charge", content));   // retries on failure
```
<details><summary>Answer</summary>

**Dangerous** — if the POST succeeded but the response was lost, the **retry charges again** (double charge). Retry only **idempotent** ops, or send an **idempotency key** the server dedupes on. Don't blindly retry non-idempotent writes.
</details>

---

### Q5 — Retry without jitter
```csharp
.AddRetry(new() { MaxRetryAttempts = 5, BackoffType = DelayBackoffType.Exponential })
// 10,000 clients all failed at the same instant
```
<details><summary>Answer</summary>

**Thundering herd** — all clients retry at the same exponential instants (T+1s, T+2s…), crushing the recovering service in synchronized waves. **Fix:** add **`UseJitter = true`** to randomize delays so retries spread out.
</details>

---

### Q6 — Circuit breaker built per call
```csharp
async Task<string> CallAsync() {
    var pipeline = new ResiliencePipelineBuilder().AddCircuitBreaker(new()).Build();   // ?
    return await pipeline.ExecuteAsync(t => FetchAsync(t));
}
```
<details><summary>Answer</summary>

**Breaker never trips** — circuit-breaker **state must persist across calls**, but a new pipeline per call resets it every time. **Fix:** build/register the pipeline **once** (DI) and reuse it; one breaker per dependency.
</details>

---

### Q7 — IMemoryCache vs IDistributedCache
```csharp
// App runs on 3 instances behind a load balancer. You cache user sessions in IMemoryCache.
```
<details><summary>Answer</summary>

**Bug at scale** — `IMemoryCache` is **per-instance**; a request hitting instance B won't see data cached on instance A (and restarts lose it). For shared, multi-instance state use **`IDistributedCache`** (Redis) or `HybridCache` (L1+L2).
</details>

---

### Q8 — Standard resilience one-liner (senior)
```csharp
builder.Services.AddHttpClient<ApiClient>()
    .AddStandardResilienceHandler();
```
<details><summary>Answer</summary>

Correct — applies a pre-tuned Polly pipeline (**total timeout → retry (backoff+jitter) → circuit breaker → per-attempt timeout**) to the typed client. The ordering matters: a total timeout bounds all retries; per-attempt timeout bounds each try; the breaker sits between. Customize via options if needed.
</details>
