# Chapter 16 — Testing

> Unit tests, integration tests, mocking, property-based testing, and benchmarking. If it's not tested it doesn't work.

**Prerequisites**: most of the book — testing is how you exercise everything you've learned.

**Time to read**: ~4-6 hours.

---

## What's in this chapter

| File | What it covers |
|---|---|
| [01-xUnit.md](01-xUnit.md) | xUnit is the de-facto standard. `[Fact]`, `[Theory]`, `IClassFixture`, `IAsyncLifetime`, parallelism, the assertion vocabulary. |
| [02-NUnitMSTest.md](02-NUnitMSTest.md) | Quick comparison: NUnit's `[TestCase]`, MSTest's `[DataRow]`, what differs in setup/teardown. |
| [03-Mocking.md](03-Mocking.md) | Moq, NSubstitute, the difference between stubs and mocks, when not to mock. |
| [04-Assertions.md](04-Assertions.md) | FluentAssertions, Shouldly, why a good failure message saves debugging time. |
| [05-IntegrationTesting.md](05-IntegrationTesting.md) | `WebApplicationFactory<TEntryPoint>`, the in-memory test server, replacing services, testing against real databases via Testcontainers. |
| [06-PropertyBased.md](06-PropertyBased.md) | FsCheck, Bogus for data generation, when property-based testing finds bugs unit tests miss. |
| [07-BenchmarkDotNet.md](07-BenchmarkDotNet.md) | `[Benchmark]`, `[MemoryDiagnoser]`, `[Params]`, common pitfalls (dead code elimination, cold cache), reading the output. |
| [Questions.md](Questions.md) | ~20 questions. |
| [Coding.md](Coding.md) | ~10 problems: write xUnit tests for a tricky async method, design a mock for a third-party SDK. |

→ Begin: [01-xUnit.md](01-xUnit.md)
