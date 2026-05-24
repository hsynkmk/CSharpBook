# 24 — System Design & Behavioral

## ⚡ 30-second answer

**System design**: clarify requirements + scale → sketch components (client → API → service → data + cache + queue) → discuss the hard parts (scaling, caching, consistency, failure). There's rarely one right answer; show you reason about **trade-offs**. **Behavioral**: use **STAR** (Situation, Task, Action, Result) — most of your time on **Action** (what *you* did) and a concrete, quantified **Result**. Prepare ~5 stories you can flex to any prompt. (Practice prompts: [24-SystemDesign-and-Behavioral-Coding.md](24-SystemDesign-and-Behavioral-Coding.md).)

---

## System design — a repeatable approach

1. **Clarify & scope**: functional requirements, **scale** (users, RPS, data size, read/write ratio), latency/availability targets.
2. **Sketch the components**:
   ```
   Client → Load Balancer → API (stateless, N instances) → Service layer
                                   │                          ├── Database (source of truth)
                                   ├── Cache (Redis)          ├── Queue/broker (async work)
                                   └── CDN (static)           └── Background workers
   ```
3. **Drill into the hard parts**:

| Concern | Options / talking points |
|---|---|
| **Scale out** | stateless app instances behind a load balancer; externalize state ([21](21-Deployment-Perf-Tooling.md)) |
| **Database scale** | read replicas, indexing, partitioning/sharding, SQL vs NoSQL by access pattern |
| **Caching** | cache-aside with Redis; TTL + invalidation; stampede protection ([18](18-Caching-Resilience-Http.md)) |
| **Async / decoupling** | queue heavy work to background workers; smooth spikes ([20](20-Observability-Messaging-Background.md)) |
| **Consistency** | strong vs eventual; idempotency keys; outbox for atomic DB+publish ([20](20-Observability-Messaging-Background.md)) |
| **Resilience** | retries+jitter, circuit breakers, timeouts, bulkheads ([18](18-Caching-Resilience-Http.md)) |
| **Observability** | logs/metrics/traces, health checks, alerting ([20](20-Observability-Messaging-Background.md)) |
| **Security** | authn/authz, secrets, rate limiting ([19](19-Security-Auth.md), [16](16-AspNetCore.md)) |

4. **State trade-offs explicitly**: CAP (consistency vs availability under partition), latency vs freshness (caching), cost vs isolation (multi-tenancy — [22](22-Architecture-Patterns.md)), complexity vs need (monolith vs microservices).

**Mini-example — "design a URL shortener"**: API to create/redirect; short key = base62 of an id (or hash); store key→URL in a DB; **cache** hot redirects in Redis (reads dominate); 301/302 redirect; scale reads with cache + replicas; analytics via an async **queue**. Trade-off: counter ids (sequential) vs random/hash (unpredictable, collision handling).

---

## Behavioral — STAR

| Letter | What | Keep it |
|---|---|---|
| **S**ituation | context | brief (1–2 sentences) |
| **T**ask | your responsibility/goal | brief |
| **A**ction | **what YOU did** (decisions, trade-offs) | **the bulk — use "I", not "we"** |
| **R**esult | outcome, **quantified** | concrete ("cut latency 40%", "removed the deadlock") |

**Prepare ~5 flexible stories**:
1. **Technical challenge** — a hard bug/perf problem you diagnosed and fixed (use a *concurrency/perf* one for this interview).
2. **Ownership/leadership** — drove something end-to-end / mentored / made a call.
3. **Conflict/disagreement** — resolved a technical disagreement with data.
4. **Failure/mistake** — what went wrong, what you learned, how you prevented recurrence.
5. **Learning/adaptation** — picking up new tech/domain under pressure.

**Example (tech challenge, concurrency-flavored)**:
> *S*: Our API stalled under load. *T*: I owned finding and fixing it. *A*: `dotnet-counters` showed the thread-pool queue climbing — a sync-over-async `.Result` was blocking pool threads. I refactored to async-all-the-way and added a bounded `SemaphoreSlim` for an external API. *R*: p95 latency dropped ~60% and the stalls disappeared; I added a load test to catch regressions.

---

## 🪤 Traps & gotchas

- **Designing before clarifying scale** — ask about RPS/data size/read-write ratio first; the design changes at different scales.
- **Jumping to microservices/fancy tech** unprompted — start simple, justify complexity ([22](22-Architecture-Patterns.md)).
- **No trade-offs** — "I'd use Redis" without *why* and *what you give up* reads junior. Always name the alternative and the cost.
- **Behavioral: "we" instead of "I"** — interviewers want *your* contribution.
- **No quantified result** — "it went well" is weak. Give numbers/outcomes.
- **Rambling Situation** — keep S/T short; spend time on Action.
- **Bluffing** — reason out loud and state assumptions rather than inventing facts.

---

## ❓ Likely questions

**Q: How do you approach a system design question?**
A: Clarify requirements and scale, sketch the components, then drill into the hard parts (scaling, caching, consistency, failure) discussing trade-offs. Reasoning matters most.

**Q: How do you scale a read-heavy service?**
A: Stateless app instances behind a load balancer, a cache (Redis, cache-aside with TTL/invalidation) for hot reads, read replicas + good indexing, and offload heavy work asynchronously.

**Q: Strong vs eventual consistency?**
A: Strong = every read sees the latest write (simpler, higher latency). Eventual = replicas converge over time (higher availability/performance, must handle stale reads). Choose per use case; CAP forces a trade under partitions.

**Q: How do you ensure a payment isn't processed twice?**
A: Idempotency key — client sends a unique key; the server dedupes so retries are processed once; combine with the outbox for reliable, atomic publish ([20](20-Observability-Messaging-Background.md)).

**Q: Tell me about a difficult technical problem.**
A: (STAR) Pick a concrete one — ideally concurrency/perf — focus on *your* diagnosis (tooling, hypothesis), the fix and trade-offs, and a quantified result + prevention.

**Q: Tell me about a time you failed.**
A: (STAR) Own a real mistake, emphasize what you learned and the safeguard you added so it can't recur. Show growth, not blame.

**Q: A disagreement with a teammate?**
A: (STAR) Resolving a technical question with data/a prototype, staying respectful, reaching a decision you supported even if it wasn't yours.

---

## 🎓 Senior Extra

- **Capacity estimation**: do rough math out loud (RPS × payload, storage/day, cache hit ratio) — shows you size systems.
- **Data modeling choice**: SQL (relational, transactions, joins) vs NoSQL (document/key-value, partition-key access, scale) — justify by **access pattern**; for NoSQL, the **partition key** is the critical decision.
- **Failure modes & blast radius**: single points of failure, retries causing cascades (need circuit breakers — [18](18-Caching-Resilience-Http.md)), graceful degradation, dead-letter queues, idempotency.
- **Consistency patterns at scale**: outbox/inbox, sagas (compensating transactions), CQRS read models, eventual consistency with reconciliation ([22](22-Architecture-Patterns.md)).
- **Observability as a requirement**: how you'd detect/debug the system in prod (3 pillars, health checks, alerting) — seniors design for operability ([20](20-Observability-Messaging-Background.md)).
- **Behavioral depth**: senior stories should show **scope and influence** (cross-team decisions, mentoring, architecture calls, owning incidents/postmortems), with quantified business impact and a blameless tone.

→ Deeper: behavioral bank in [`../06-Behavioral/`](../06-Behavioral/), process notes in [`../00-Process/`](../00-Process/)
