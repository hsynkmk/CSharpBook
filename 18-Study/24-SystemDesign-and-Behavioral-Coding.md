# 24 — System Design & Behavioral — Practice Prompts

> Mini design prompts (with model answers) + a behavioral bank. Practice answering **out loud**. (Concepts: [24-SystemDesign-and-Behavioral.md](24-SystemDesign-and-Behavioral.md))

---

## System design mini-prompts

### D1 — Design a URL shortener
<details><summary>Model answer (bullets)</summary>

- **Clarify**: read-heavy (redirects ≫ creates)? scale (QPS)? custom aliases? analytics?
- **API**: `POST /shorten` (url → short code), `GET /{code}` → 301/302 redirect.
- **Code gen**: base62 of an auto-increment id (short, no collisions) **or** hash + collision check (unpredictable).
- **Storage**: key→URL in a DB (indexed on code). **Cache** hot codes in Redis (reads dominate).
- **Scale reads**: cache + read replicas; the redirect is cacheable.
- **Analytics**: emit click events to a **queue** → async worker (don't block the redirect).
- **Trade-off**: sequential ids (predictable, enumerable) vs random/hash (unpredictable, needs collision handling).
</details>

### D2 — Design a rate limiter
<details><summary>Model answer</summary>

- **Clarify**: per user/IP/API-key? limit (e.g., 100/min)? distributed (multi-instance)?
- **Algorithms**: fixed window (simple, burst at edges), **sliding window** (smoother), **token bucket** (allows bursts up to capacity — common).
- **State**: in-memory for single instance; **Redis** for distributed (atomic INCR + TTL, or Lua for token bucket).
- **Response**: 429 + `Retry-After` header.
- ASP.NET Core has built-in rate limiting (`UseRateLimiter`) with these policies ([16](16-AspNetCore.md)).
</details>

### D3 — Design a notification system (email/SMS/push)
<details><summary>Model answer</summary>

- **Decouple**: API enqueues a notification request to a **message broker**; workers consume and send.
- **At-least-once + idempotency**: dedupe by notification id so retries don't double-send ([20](20-Observability-Messaging-Background.md)).
- **Outbox**: if the trigger is a DB change, use the **outbox pattern** to publish atomically.
- **Resilience**: retry with backoff+jitter, **circuit breaker** per provider, dead-letter for poison messages ([18](18-Caching-Resilience-Http.md)).
- **Scale**: partition by channel/provider; bounded consumer concurrency.
- **Observe**: metrics (sent/failed rate), traces, alerting.
</details>

### D4 — Design a read-heavy product catalog
<details><summary>Model answer</summary>

- **Cache-aside** with Redis for product reads (TTL + invalidate on write); guard against **stampede** (HybridCache/lock).
- **DB**: indexed queries; read replicas; project to DTOs (avoid over-fetch / N+1 — [17](17-EFCore.md)).
- **CDN** for images/static.
- **Search**: a dedicated search index (Elasticsearch) if full-text/faceting needed.
- **Consistency**: eventual is fine for catalog (stale-by-seconds acceptable); strong for inventory/checkout.
</details>

### D5 — Ensure a payment isn't processed twice
<details><summary>Model answer</summary>

- **Idempotency key**: client sends a unique key per payment attempt; server records it and returns the same result for retries (so a lost-response retry doesn't double-charge).
- **At-least-once messaging** → idempotent handler (dedupe by key).
- **Outbox** for atomic "charge recorded + event published".
- **Optimistic concurrency** (`RowVersion`) on the account/balance row to prevent lost updates ([17](17-EFCore.md)).
</details>

### D6 — Scale a stateful app to many instances
<details><summary>Model answer</summary>

- **Make it stateless**: externalize session/cache to Redis, files to blob storage, data to the DB.
- **Load-balance** across instances; any instance handles any request.
- **Sticky sessions** only if unavoidable (weaker scaling).
- **Graceful shutdown** (SIGTERM/cancellation) + health probes for rolling deploys ([21](21-Deployment-Perf-Tooling.md)).
</details>

---

## Behavioral bank (answer with STAR — say "I", quantify the Result)

> Prepare 5 stories and map them to these. Use a **concurrency/perf** story for the "difficult problem" prompt given this interview.

### B1 — "Tell me about a difficult technical problem you solved."
<details><summary>STAR skeleton (fill with your story)</summary>

- **S**: a concrete symptom (e.g., "API latency spiked under load").
- **T**: "I owned diagnosing and fixing it."
- **A**: tooling + hypothesis + fix + trade-off — e.g., *"`dotnet-counters` showed thread-pool queue climbing; a `.Result` call was blocking pool threads. I refactored to async all-the-way and added a bounded `SemaphoreSlim` for the external API."*
- **R**: quantified — *"p95 dropped ~60%, stalls gone; I added a load test to catch regressions."*
</details>

### B2 — "A time you disagreed with a teammate."
<details><summary>Approach</summary>

Frame as resolving a *technical* disagreement with **data/a prototype**, staying respectful, reaching a decision, and committing to it even if it wasn't yours. Result: better outcome + preserved relationship.
</details>

### B3 — "A time you failed / made a mistake."
<details><summary>Approach</summary>

Own a **real** mistake (no blame-shifting), focus on **what you learned** and the **safeguard you added** so it can't recur (test, alert, process). Shows growth and accountability.
</details>

### B4 — "A time you took ownership / led."
<details><summary>Approach</summary>

Drove something end-to-end, made a call under ambiguity, or mentored someone. Emphasize **your** decisions and the impact (scope/business outcome), not just team effort.
</details>

### B5 — "Learning something new quickly."
<details><summary>Approach</summary>

Picked up an unfamiliar tech/domain under a deadline: how you ramped (docs, small spikes, asking the right people) and delivered. Shows adaptability.
</details>

---

## 🪤 Interview-day reminders
- **Clarify scale before designing**; state **trade-offs** explicitly (no single right answer).
- Don't reach for microservices/fancy tech unprompted — **start simple, justify complexity**.
- Behavioral: **"I" not "we"**, short S/T, long **A**, quantified **R**, blameless tone for failures.
- If unsure: reason aloud and state assumptions — never bluff.
