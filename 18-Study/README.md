# C# / .NET Interview Prep — Condensed & Comprehensive

> Everything you need, distilled from the full `CSharpBook/` and `DotNetBook/` into a set you can read in **5 days**. Self-contained — you only need this folder. **Multithreading is weighted heaviest** (it'll dominate your interview), but everything important is here.

---

## How the files are organized

Each topic has **two** files:
- **`NN-Topic.md`** — the concepts, in a scannable format.
- **`NN-Topic-Coding.md`** — **"what's the output / find the bug / fix this"** questions with hidden answers (interviews love these).

Every concept file follows the same structure:
1. **⚡ 30-second answer** — what to say out loud first.
2. **Core mechanics** — the bullets + key code that matter.
3. **Comparison tables** — the side-by-sides interviewers love.
4. **🪤 Traps & gotchas** — the "wrong answer" pitfalls.
5. **❓ Likely questions** — crisp Q→A you can fire back.
6. **🎓 Senior Extra** — deeper internals to stand out.

Aim: **understand the 30-second answer + traps for every file**, and **work the `-Coding.md` questions** (predicting output/finding bugs is what most interviews test).

---

## Quick-reference (use daily + morning-of)
| File | Use |
|---|---|
| [RapidFire.md](RapidFire.md) | 120+ one-line Q&A to memorize and self-quiz |
| [CheatSheet.md](CheatSheet.md) | Last-minute scannable: top traps, all tables, power phrases, coding protocol |

## Topic map (each row also has a `-Coding.md`)

### C# Language
| # | Concepts | Coding |
|---|---|---|
| 01 | [TypeSystem](01-TypeSystem.md) — value/reference, **boxing**, struct/class/record, nullable | [Coding](01-TypeSystem-Coding.md) |
| 02 | [OOP](02-OOP.md) — **override vs new**, abstract vs interface, access, sealed | [Coding](02-OOP-Coding.md) |
| 03 | [Generics-Delegates-Events](03-Generics-Delegates-Events.md) — variance, **closures**, events | [Coding](03-Generics-Delegates-Events-Coding.md) |
| 04 | [LINQ](04-LINQ.md) — **deferred execution**, IEnumerable vs IQueryable, N+1 | [Coding](04-LINQ-Coding.md) |
| 05 | [Collections](05-Collections.md) — **Dictionary internals**, Big-O, concurrent/frozen | [Coding](05-Collections-Coding.md) |
| 06 | [Equality](06-Equality.md) — **Equals/GetHashCode**, records, comparers | [Coding](06-Equality-Coding.md) |
| 07 | [Exceptions-Idioms](07-Exceptions-Idioms.md) — try-pattern, defensive code, anti-patterns | [Coding](07-Exceptions-Idioms-Coding.md) |
| 08 | [ModernCSharp](08-ModernCSharp.md) — pattern matching, records, NRT, C# 7→14 | [Coding](08-ModernCSharp-Coding.md) |

### Concurrency & Async — **HEAVIEST EMPHASIS**
| # | Concepts | Coding |
|---|---|---|
| 09 | [Threading-and-Tasks](09-Threading-and-Tasks.md) — thread vs Task, **thread pool**, WhenAll | [Coding](09-Threading-and-Tasks-Coding.md) |
| 10 | [AsyncAwait](10-AsyncAwait.md) — **state machine**, ValueTask, ConfigureAwait, deadlocks | [Coding](10-AsyncAwait-Coding.md) |
| 11 | [Synchronization-and-MemoryModel](11-Synchronization-and-MemoryModel.md) — lock/**Interlocked**/volatile, deadlocks | [Coding](11-Synchronization-and-MemoryModel-Coding.md) |
| 12 | [Concurrent-Parallel-AsyncBugs](12-Concurrent-Parallel-AsyncBugs.md) — **Channels**, TPL, **classic async bugs** | [Coding](12-Concurrent-Parallel-AsyncBugs-Coding.md) |
| 13 | [MemoryAndGC](13-MemoryAndGC.md) — **GC generations/LOH**, IDisposable, **leaks** | [Coding](13-MemoryAndGC-Coding.md) |

### .NET Platform
| # | Concepts | Coding |
|---|---|---|
| 14 | [Runtime-CLR-JIT](14-Runtime-CLR-JIT.md) — **JIT/tiered/AOT**, GC modes | [Coding](14-Runtime-CLR-JIT-Coding.md) |
| 15 | [DI-Hosting-Config](15-DI-Hosting-Config.md) — **lifetimes + captive dependency**, IOptions | [Coding](15-DI-Hosting-Config-Coding.md) |
| 16 | [AspNetCore](16-AspNetCore.md) — **middleware ordering**, Minimal vs MVC, filters | [Coding](16-AspNetCore-Coding.md) |
| 17 | [EFCore](17-EFCore.md) — DbContext=UoW, tracking, **N+1**, concurrency | [Coding](17-EFCore-Coding.md) |
| 18 | [Caching-Resilience-Http](18-Caching-Resilience-Http.md) — caches, IHttpClientFactory, **Polly** | [Coding](18-Caching-Resilience-Http-Coding.md) |
| 19 | [Security-Auth](19-Security-Auth.md) — **authn vs authz**, JWT, OAuth/OIDC, CSRF | [Coding](19-Security-Auth-Coding.md) |
| 20 | [Observability-Messaging-Background](20-Observability-Messaging-Background.md) — logs/metrics/traces, **outbox** | [Coding](20-Observability-Messaging-Background-Coding.md) |
| 21 | [Deployment-Perf-Tooling](21-Deployment-Perf-Tooling.md) — Docker/K8s, AOT, **profiling tools** | [Coding](21-Deployment-Perf-Tooling-Coding.md) |

### Architecture, Coding, Design, Behavioral
| # | Concepts | Coding |
|---|---|---|
| 22 | [Architecture-Patterns](22-Architecture-Patterns.md) — **SOLID**, GoF, CQRS, anti-patterns | [Coding](22-Architecture-Patterns-Coding.md) |
| 23 | [CodingPatterns](23-CodingPatterns.md) — trees, two-pointer, **multithreaded problems** | [Coding](23-CodingPatterns-Coding.md) |
| 24 | [SystemDesign-and-Behavioral](24-SystemDesign-and-Behavioral.md) — design checklist + **STAR** | [Coding](24-SystemDesign-and-Behavioral-Coding.md) |

---

## 5-day study plan

| Day | Focus | Files (read concept + work the `-Coding`) |
|---|---|---|
| **1** | C# language core | 01–08 |
| **2** | **Concurrency & async (the big one)** | 09–13 |
| **3** | .NET platform | 14–18 |
| **4** | Platform + architecture | 19–22 |
| **5** | Coding + design + behavioral, then review quick-ref | 23, 24, RapidFire, CheatSheet |

**Each day**: read the concept files → re-read the **⚡30-second answers** and **🪤traps** → **work the matching `-Coding.md`** (cover the answer, predict, reveal) → self-quiz with RapidFire. Memorize the *mental model* and *traps*, not code verbatim.

---

## ⏰ Morning-of (90-minute final skim)

1. **CheatSheet.md** (15 min) — all tables + power phrases.
2. **RapidFire.md** (30 min) — answer every Q out loud, star misses.
3. **Concurrency traps + coding** (30 min) — re-skim 🪤 of files **10, 11, 12** and their `-Coding` (deadlock, async void, race, `lock` vs `Interlocked` vs `volatile`).
4. **15 min** — your 5 STAR stories ([24](24-SystemDesign-and-Behavioral.md)).

---

## How to talk in the interview (always)

- **Restate** the question → **clarify** assumptions → **think out loud** → answer the *direct* question first, then add depth.
- "X vs Y": one-sentence difference → when to use each → a trap.
- Code: state **approach + Big-O** before typing; verify with an example after.
- Don't bluff internals — reason aloud, state assumptions.

> Go-deeper links point at `../CSharpBook/` and `../DotNetBook/`, but you don't need them — this folder is complete.
