# Chapter 16 — Testing — Q & A

---

### Q1. `[Fact]` vs `[Theory]`?

`[Fact]` is a single test with no parameters (one scenario). `[Theory]` is parameterized — runs the same logic with multiple data sets (`InlineData`/`MemberData`/`ClassData`/`TheoryData`), each row appearing as a separate test case.

---

### Q2. How does xUnit handle setup/teardown without `[SetUp]`?

It uses the language: the test class **constructor** is per-test setup, `Dispose` (or `IAsyncLifetime.DisposeAsync`) is teardown. xUnit creates a **fresh instance per test method**, so tests are isolated with no shared mutable state.

---

### Q3. `IClassFixture` vs `ICollectionFixture`?

`IClassFixture<T>` shares one fixture instance across tests in a single class. `ICollectionFixture<T>` shares one instance across multiple classes in a named collection (and serializes them). Both exist to reuse expensive resources (DB, web host).

---

### Q4. How do you test async code in xUnit?

Make the test `async Task` and `await` the operation. Never use `async void` (the runner can't observe completion/exceptions) and never block with `.Result`/`.Wait()` (deadlock risk, swallowed exceptions).

---

### Q5. What does xUnit run in parallel?

Test **collections** run in parallel (different classes are different collections unless grouped). Tests **within** a class run sequentially. Tests sharing mutable global state must be grouped into a collection to serialize them.

---

### Q6. Biggest lifecycle difference between xUnit and NUnit?

xUnit creates a **new test-class instance per test** (isolated). NUnit (by default) uses **one fixture instance for all tests** in the class, so field state persists between tests unless reset in `[SetUp]`. This surfaces hidden coupling when migrating to xUnit.

---

### Q7. Stub vs mock?

A **stub** provides canned return values (indirect input to the unit). A **mock** is a stub that *also verifies* it was called as expected (checks indirect output/interactions). Casually "mock" covers both; the meaningful distinction is whether you assert on interactions.

---

### Q8. Moq vs NSubstitute syntactically?

Moq: `new Mock<IFoo>()`, `.Setup(x => ...).Returns(...)`, `.Verify(...)`, pass `mock.Object`. NSubstitute: `Substitute.For<IFoo>()`, `foo.Method().Returns(...)`, `foo.Received().Method()`, use the substitute directly (no `.Object`). NSubstitute reads more cleanly; both are equivalent in power.

---

### Q9. What can mocking libraries substitute, and why does it matter?

Only **interfaces** and **virtual/abstract** members (they generate a proxy subclass at runtime). Non-virtual methods, sealed classes, static methods, and extension methods can't be overridden. This is why depending on interfaces (DI) makes code testable.

---

### Q10. When should you NOT mock?

Don't mock types you don't own (wrap them behind your own interface first), DTOs/value objects (just construct them), or over-verify incidental interactions (brittle). Prefer in-memory **fakes** for stateful dependencies like repositories.

---

### Q11. How do you make `DateTime.Now` testable?

Don't call it directly — inject an abstraction. Modern .NET provides `TimeProvider`; pass it in and use `FakeTimeProvider` (from `Microsoft.Extensions.TimeProvider.Testing`) in tests to control the clock. No static mocking needed.

---

### Q12. Why use FluentAssertions/Shouldly over built-in asserts?

Better **failure messages** — they read as sentences and pinpoint exactly what differs (e.g., "item[2].Name to be Bob but found Bobby"). Over a project's life, clearer messages save substantial debugging time. Built-in asserts are fine for simple checks.

---

### Q13. `Be` vs `BeEquivalentTo` in FluentAssertions?

`Be` uses the type's equality (reference equality for classes, value equality for records). `BeEquivalentTo` does deep **structural** property comparison recursively, ignoring reference identity. Use `BeEquivalentTo` for "same data" comparisons of object graphs.

---

### Q14. What's the difference between unit and integration tests?

Unit tests verify one class in isolation with mocked dependencies. Integration tests verify components working together — real DI container, middleware pipeline, real/realistic database — catching wiring bugs (DI misconfig, routing, serialization, EF queries) that unit tests can't.

---

### Q15. What does `WebApplicationFactory<Program>` do?

Boots your real ASP.NET Core app in memory (in-process `TestServer`, no network port) and provides an `HttpClient` wired to its pipeline. `CreateClient()` requests flow through real routing, middleware, controllers, and services — fast end-to-end tests.

---

### Q16. How do you replace a real service with a fake in integration tests?

`factory.WithWebHostBuilder(b => b.ConfigureTestServices(services => { services.RemoveAll<IFoo>(); services.AddSingleton<IFoo, FakeFoo>(); }))`. `ConfigureTestServices` runs after normal registration, so overrides win — letting you stub payments/email/external HTTP while keeping the real pipeline.

---

### Q17. Three database fidelity options for integration tests?

1. **EF in-memory provider** — fast, but no real SQL/constraints (low fidelity, diverges from production). 2. **SQLite in-memory** — real SQL engine, medium fidelity (keep the connection open or the DB vanishes). 3. **Testcontainers** — the real DB (Postgres/SQL Server) in Docker, highest fidelity, needs Docker and is slower.

---

### Q18. What does `public partial class Program;` enable?

It makes the top-level-statement `Program` class accessible so `WebApplicationFactory<Program>` can reference the app's entry point. Without it (or `InternalsVisibleTo`), the factory can't find `Program`.

---

### Q19. What is property-based testing?

Testing that an **invariant holds for hundreds of randomly generated inputs**, instead of specific hand-picked cases. It explores edge cases you wouldn't enumerate and, on failure, **shrinks** the input to a minimal counterexample. FsCheck is the .NET library.

---

### Q20. What is shrinking and why is it valuable?

When a property fails on a large random input, the runner repeatedly simplifies the input while it still fails, reporting the **smallest** failing case (e.g., `[0, -1]` instead of a 6-element array). This turns "failed on random garbage" into an obvious minimal reproducer pointing at the bug.

---

### Q21. Name three good property patterns.

Round-trip/inverse (`Deserialize(Serialize(x)) == x`), idempotence (`Sort(Sort(x)) == Sort(x)`), commutativity (`Add(a,b) == Add(b,a)`). Others: invariants (sorted output, same length), oracle comparison (against a known-good impl), associativity, metamorphic relations.

---

### Q22. FsCheck vs Bogus?

FsCheck generates **adversarial** edge-case data to falsify properties (and shrinks failures). Bogus generates **realistic** plausible domain data (names, emails, addresses) for seeding tests. They complement each other; use a fixed seed with Bogus for reproducibility.

---

### Q23. Why not benchmark with `Stopwatch`?

It ignores JIT warmup (measuring compilation, not steady state), GC interference, dead-code elimination, CPU frequency scaling, and provides no statistics. BenchmarkDotNet handles warmup, many iterations, outlier detection, and variance analysis for trustworthy numbers.

---

### Q24. Why must BenchmarkDotNet run in Release?

Debug builds disable optimizations (no inlining, extra checks), making timings meaningless. BenchmarkDotNet refuses to run optimized benchmarks in Debug and warns. Always `dotnet run -c Release` in a dedicated console project.

---

### Q25. How do you prevent dead-code elimination in a benchmark?

**Return** the computed result from the `[Benchmark]` method so the JIT can't delete the work as unused. Also avoid constant inputs (constant folding) — use fields or `[Arguments]` so computation actually happens at runtime.

---

### Q26. When is a benchmark difference NOT significant?

When the Mean ± Error intervals of two benchmarks **overlap**, the difference isn't statistically significant — don't claim one is faster. Check Error/StdDev, not just the Mean. Also remember `[MemoryDiagnoser]`'s Allocated column is often the more important result.

---

→ Next: [Coding.md](Coding.md)
