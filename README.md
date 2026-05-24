# The C# Book — Beginner to Expert

> A comprehensive, self-contained reference for the C# language, covering everything from "Hello, World" to expert-level runtime internals. Current as of **C# 14 / .NET 10 (November 2025, LTS)**.

This book is for:
- **Beginners** with no C# experience — start at Chapter 0 and read in order.
- **Intermediate developers** who want to fill gaps — jump to any chapter; prerequisites are listed.
- **Senior engineers** preparing for interviews or modernizing legacy codebases — chapters 8–17 cover the depth most never reach.

Every concept includes:
- *What it is* + mental model
- *Why it exists* (the problem it solves)
- *How it works under the hood* (IL, JIT, runtime)
- Common patterns + gotchas + performance notes
- Drilling questions and coding problems with full solutions

---

## 📚 Table of Contents

### Level 0 — Orientation
- [Chapter 00 — Introduction](00-Introduction/README.md) — What C# is, .NET ecosystem, SDK setup, your first program

### Level 1 — Beginner
- [Chapter 01 — Fundamentals](01-Fundamentals/README.md) — Variables, types, operators, control flow, methods, arrays, exceptions
- [Chapter 02 — Object-Oriented Programming](02-OOP/README.md) — Classes, properties, inheritance, polymorphism, interfaces

### Level 2 — Intermediate
- [Chapter 03 — The Type System](03-TypeSystem/README.md) — Value vs reference, structs, records, enums, tuples, nullable, pattern matching
- [Chapter 04 — Generics](04-Generics/README.md) — Generic types, constraints, variance, generic math
- [Chapter 05 — Delegates & Events](05-DelegatesEvents/README.md) — Delegates, lambdas, closures, events, expression trees
- [Chapter 06 — LINQ](06-LINQ/README.md) — Query syntax, deferred execution, custom operators, IQueryable
- [Chapter 07 — Collections](07-Collections/README.md) — Array, List, Dictionary, ImmutableX, FrozenX, equality contract

### Level 3 — Advanced
- [Chapter 08 — Concurrency](08-Concurrency/README.md) — async/await, Tasks, locks, semaphores, channels, TPL, memory model
- [Chapter 09 — Memory & Performance](09-MemoryPerformance/README.md) — GC, Span, Memory, ArrayPool, stackalloc, ref structs, unsafe
- [Chapter 10 — Advanced Language Features](10-AdvancedLanguage/README.md) — NRT, deep patterns, records deep, raw strings, interpolated handlers

### Level 4 — Expert
- [Chapter 11 — Modern Features (Version by Version)](11-ModernFeatures/README.md) — C# 7 through C# 14 + .NET 10 runtime improvements
- [Chapter 12 — Reflection & Metaprogramming](12-Reflection/README.md) — Reflection, attributes, source generators, compile-time tricks
- [Chapter 13 — I/O & Serialization](13-IO/README.md) — Files, streams, JSON, XML, encoding, globalization
- [Chapter 14 — Interop & AOT](14-InteropAOT/README.md) — P/Invoke, SafeHandle, Native AOT, trimming, publishing

### Level 5 — Mastery
- [Chapter 15 — Build & Tooling](15-BuildTooling/README.md) — dotnet CLI, csproj, NuGet, MSBuild, analyzers, profiling
- [Chapter 16 — Testing](16-Testing/README.md) — xUnit, mocking, integration tests, BenchmarkDotNet
- [Chapter 17 — Best Practices & Idioms](17-BestPractices/README.md) — Naming, async idioms, API design, anti-patterns

### Reference
- [GLOSSARY](GLOSSARY.md) — Every term defined
- [INDEX](INDEX.md) — Find any topic fast

---

## 🛣️ Learning Paths

### Absolute beginner (0 → working developer, ~6-8 weeks)
1. Chapter 00 (Introduction) — 1 day
2. Chapter 01 (Fundamentals) — 1 week
3. Chapter 02 (OOP) — 2 weeks
4. Chapter 03 (Type System) — 1 week
5. Chapter 06 (LINQ basics) + Chapter 07 (Collections) — 1 week
6. Chapter 08 first half (async/await fundamentals) — 1 week

### Java/C++ developer learning C# (~2 weeks)
- Skim 01 (Fundamentals) for syntax differences
- Read 02 (OOP), 03 (Type System), 04 (Generics) carefully — semantics differ
- Read 08 (Concurrency) and 09 (Memory) — async/await and ref structs are unique
- Chapter 11 (Modern Features) gives a tour of what's new since you last looked

### Preparing for a senior C# interview (~3-4 weeks)
- Chapter 09 (Memory & Performance) — GC, IDisposable, leaks
- Chapter 08 (Concurrency) — read in full
- Chapter 02 + 03 (OOP + Type System) — polymorphism + value/reference + boxing
- Chapter 07 (Collections) — Dictionary internals, equality contract
- Every `Coding.md` — drill, drill, drill

### Just want the latest features (~1 day)
- Chapter 11 (Modern Features) — version-by-version walkthrough
- Chapter 11 §07 (C# 14) + Chapter 11 §08 (.NET 10 runtime) for the bleeding edge

---

## 📋 Each Chapter Contains

Every chapter folder follows the same layout:

```
NN-ChapterName/
├── README.md             ← chapter overview (you are here when you click it)
├── 01-SubTopic1.md       ← deep dive #1
├── 02-SubTopic2.md       ← deep dive #2
├── ...                   ← (typically 5-15 sub-files)
├── Questions.md          ← classic Q&A drill
└── Coding.md             ← 10-15 coding problems with solutions
```

**Sub-files** are structured uniformly:
1. *What it is* + mental model
2. *Why it exists*
3. *Syntax + minimal example*
4. *Deep mechanics* (under the hood)
5. *Common patterns + idioms*
6. *Gotchas and pitfalls*
7. *Performance notes*
8. *When to use / when to avoid*

---

## 🎯 Conventions

- **Code samples** target **C# 14 / .NET 10**. Older syntax shown only when discussing history.
- `csharp` code blocks are runnable unless marked with a `// ⚠` to indicate a deliberate bug.
- "Trap" sections call out the bugs interviewers love to ask about.
- `<details><summary>Solution</summary>` blocks hide answers — try the problem first.
- Each file is independently readable, but earlier chapters are prerequisites for later ones.

---

## ✍️ About This Book

This book modernizes and dramatically expands the community-favorite "C# Notes For Professionals" (originally written for C# 7 in 2017). Everything in the original is here, plus seven major language versions of new features (8, 9, 10, 11, 12, 13, 14), plus the .NET runtime advances that shaped how modern C# is written.

The book is **opinionated about quality**: every claim is fact-checked against the official C# language specification, the .NET API reference, and the dotnet/csharplang + dotnet/runtime GitHub repositories. Where official documentation disagrees with common practice, both are presented.

Read in order, jump around, drill the questions — whatever fits how you learn. The goal is for you to finish this book knowing C# as well as anyone you'll work with.

Now go open [Chapter 00](00-Introduction/README.md) and start.
