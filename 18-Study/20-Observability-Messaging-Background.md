# 20 — Observability, Messaging & Background Processing

## ⚡ 30-second answer

**Observability** = understanding a running system from its outputs: the three pillars are **logs** (events), **metrics** (numbers over time), and **traces** (a request's path across services). Use **structured logging** (`ILogger` with named placeholders) and **OpenTelemetry** to export all three. **Messaging** decouples services via **queues/brokers** (RabbitMQ, Azure Service Bus, Kafka) or in-process **`Channel<T>`**; delivery is **at-least-once**, so handlers must be **idempotent**, and the **outbox pattern** solves the "save DB + publish message atomically" problem. **Background processing** runs work off the request thread via **`BackgroundService`/`IHostedService`** (with a DI scope per item).

---

## Core mechanics

**Structured logging** (named properties, not string concat):
```csharp
logger.LogInformation("Order {OrderId} placed for {Amount}", id, amount);  // queryable fields
using (logger.BeginScope("Tenant {TenantId}", tenantId)) { … }              // contextual scope
```
Log **levels**: Trace < Debug < Information < Warning < Error < Critical. Don't log secrets/PII.

**The three pillars**:
- **Logs** — discrete events (`ILogger`, Serilog).
- **Metrics** — counters/gauges/histograms (`Meter` API) — request rate, latency, queue depth.
- **Traces** — `Activity`/spans correlated by a trace id across services (distributed tracing).
- **OpenTelemetry** unifies and exports them (Application Insights, Grafana/Tempo/Prometheus, Datadog).

**Health checks**: `/health` (readiness — dependencies OK, route traffic) vs `/alive` (liveness — process up, else restart) — consumed by Kubernetes ([21](21-Deployment-Perf-Tooling.md)).

**Messaging**:
```csharp
// In-process pipeline:
await channel.Writer.WriteAsync(work, ct);                 // Channel<T> ([12])
// Cross-service: publish to a broker; consumers process independently.
```
Delivery is typically **at-least-once** → design **idempotent** consumers (dedupe by message id).

**Outbox pattern** (atomic DB + publish):
```
1. In the SAME transaction as the business change, write the event to an "outbox" table.
2. A separate process reads the outbox and publishes to the broker, marking rows sent.
→ event is published iff the change committed (solves the dual-write problem).
```

**Background processing**:
```csharp
public class Worker(IServiceScopeFactory scopes) : BackgroundService {
    protected override async Task ExecuteAsync(CancellationToken stop) {
        while (!stop.IsCancellationRequested) {
            using var scope = scopes.CreateScope();        // fresh scoped services per item ([15])
            // do work; respect 'stop' for graceful shutdown
        }
    }
}
```

---

## Comparison tables

| Pillar | Answers | Tool |
|---|---|---|
| **Logs** | "what happened?" | `ILogger`, Serilog |
| **Metrics** | "how much / how fast?" | `Meter`, Prometheus |
| **Traces** | "where did the request go / what was slow?" | `Activity`, OpenTelemetry |

| Messaging option | When |
|---|---|
| `Channel<T>` | in-process producer/consumer ([12](12-Concurrent-Parallel-AsyncBugs.md)) |
| Queue (Service Bus / RabbitMQ) | reliable commands/work, ordering, dead-letter |
| Event stream (Kafka / Event Hubs) | high-volume telemetry/events, replay |
| Pub/sub (topics / Event Grid) | broadcast events to many consumers |

---

## 🪤 Traps & gotchas

- **String-concatenated logs** (`$"Order {id}"`) lose structure (can't query/filter by `OrderId`). Use **message templates** with named placeholders.
- **Logging too much / in hot loops / expensive args** — costs CPU even when filtered. Check level or use `LoggerMessage` source-gen.
- **Logging secrets/PII** — scrub before it reaches logs/APM.
- **Non-idempotent message handlers** — at-least-once delivery means a message can arrive twice; double-processing results. Dedupe by id / make handlers idempotent.
- **Dual-write problem** — committing the DB then publishing to a broker can lose the message if the publish fails → DB and downstream diverge. Use the **outbox**.
- **Fire-and-forget for background work** — exceptions vanish, no lifecycle ([12](12-Concurrent-Parallel-AsyncBugs.md)). Use `BackgroundService` (or a queue) and observe failures.
- **Not creating a DI scope per item** in a long-lived `BackgroundService` — reusing one scoped `DbContext` across items leaks/corrupts. Create a scope per item ([15](15-DI-Hosting-Config.md)).
- **Ignoring the stopping token** — work killed mid-flight on shutdown. Respect the `CancellationToken` for graceful drain.
- **Broken trace correlation** — a non-instrumented hop drops the trace id; use instrumented `HttpClient`/frameworks so W3C trace context flows.

---

## ❓ Likely questions

**Q: What are the three pillars of observability?**
A: Logs (events), metrics (numeric time series), traces (a request's path across services, correlated by trace id). OpenTelemetry collects and exports all three.

**Q: Why structured logging?**
A: Named placeholders capture fields you can query/filter/aggregate (`OrderId`, `Amount`), unlike string-concatenated messages. Better diagnostics and correlation.

**Q: Why must message handlers be idempotent?**
A: Brokers deliver at-least-once — a message can be redelivered (failure, retry). A non-idempotent handler double-processes. Dedupe by message id or design idempotent operations.

**Q: What problem does the outbox pattern solve?**
A: The dual-write problem — you can't atomically write the DB and publish to a broker. The outbox writes the event in the same DB transaction, and a separate process publishes it, guaranteeing publish iff the change committed.

**Q: How do you run background work in ASP.NET Core?**
A: `BackgroundService`/`IHostedService` hosted by the Generic Host. Create a DI scope per work item, respect the stopping token for graceful shutdown, and observe exceptions.

**Q: Liveness vs readiness health checks?**
A: Liveness (`/alive`) = is the process healthy (else restart). Readiness (`/health`) = can it serve now, incl. dependencies (else stop routing traffic). A liveness check shouldn't depend on a downstream DB (restart loops).

**Q: Logs vs metrics vs traces — when to use which?**
A: Logs for event detail/debugging; metrics for trends/alerting (rate/latency/errors); traces to find *where* a distributed request spent time or failed.

---

## 🎓 Senior Extra

- **Distributed tracing internals**: a trace = a tree of **spans**; context propagates via **W3C `traceparent`** headers; `Activity`/`ActivitySource` is the .NET API. Sampling controls volume at scale.
- **Metric cardinality**: high-cardinality tag values (user id, request id) explode metric series and cost — keep tags low-cardinality; put high-cardinality detail in logs/traces.
- **Outbox + inbox + idempotency keys** together give exactly-once *effects* over at-least-once delivery; **sagas** coordinate multi-service workflows with compensating actions; **dead-letter queues** isolate poison messages.
- **Ordering & partitioning**: Kafka/Service Bus sessions guarantee order within a partition/session key, not globally — design keys accordingly.
- **Backpressure**: bounded `Channel<T>` / consumer concurrency limits prevent unbounded memory growth ([12](12-Concurrent-Parallel-AsyncBugs.md)).
- **Scheduling**: `PeriodicTimer` for simple intervals; **Hangfire/Quartz.NET** for persistent, distributed, cron-style jobs with retries; Worker Service projects for dedicated hosts.
- **OpenTelemetry over vendor SDKs**: instrument with OTel (vendor-neutral) and export to your APM — the same telemetry works locally (Aspire dashboard) and in prod.
- **Log providers/sinks**: structured logs → JSON sinks (Seq/ELK/Loki); enrich with correlation/trace ids so logs link to traces.

→ Deeper: [`../DotNetBook/12-Observability/`](../DotNetBook/12-Observability/README.md), [`../DotNetBook/07-Messaging/`](../DotNetBook/07-Messaging/README.md), [`../DotNetBook/08-BackgroundProcessing/`](../DotNetBook/08-BackgroundProcessing/README.md)
