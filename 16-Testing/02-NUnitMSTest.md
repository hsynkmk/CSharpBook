# NUnit and MSTest

## The three frameworks

.NET has three mainstream test frameworks. They're ~90% interchangeable in capability; the differences are in attribute names, lifecycle, and philosophy.

| | xUnit | NUnit | MSTest |
|---|---|---|---|
| Test attribute | `[Fact]` / `[Theory]` | `[Test]` / `[TestCase]` | `[TestMethod]` / `[DataRow]` |
| Test class marker | (none needed) | `[TestFixture]` (optional) | `[TestClass]` (required) |
| Per-test setup | constructor | `[SetUp]` | `[TestInitialize]` |
| Per-test teardown | `Dispose` | `[TearDown]` | `[TestCleanup]` |
| Per-class setup | `IClassFixture` | `[OneTimeSetUp]` | `[ClassInitialize]` |
| Per-class teardown | fixture `Dispose` | `[OneTimeTearDown]` | `[ClassCleanup]` |
| New instance per test | yes | no (one per fixture by default) | no |
| Parallel by default | yes (collections) | opt-in | opt-in |
| Assertions | `Assert.Equal` | `Assert.That(x, Is.EqualTo(y))` | `Assert.AreEqual` |

**xUnit** is the most common choice for new projects. **NUnit** is mature and feature-rich. **MSTest** ships with Visual Studio and is common in Microsoft-centric/enterprise shops.

---

## NUnit

```csharp
using NUnit.Framework;

[TestFixture]
public class CalculatorTests {
    private Calculator _calc = null!;

    [SetUp]                          // before EACH test
    public void Setup() => _calc = new Calculator();

    [TearDown]                       // after EACH test
    public void Cleanup() { /* ... */ }

    [OneTimeSetUp]                   // once before all tests in the fixture
    public void FixtureSetup() { }

    [Test]
    public void Add_ReturnsSum() {
        Assert.That(_calc.Add(2, 3), Is.EqualTo(5));
    }
}
```

### NUnit parameterized tests

```csharp
[TestCase(2, 3, 5)]
[TestCase(-1, 1, 0)]
public void Add(int a, int b, int expected) {
    Assert.That(new Calculator().Add(a, b), Is.EqualTo(expected));
}

// Sources for complex data
[TestCaseSource(nameof(Cases))]
public void Add_FromSource(int a, int b, int expected) { ... }
public static IEnumerable<TestCaseData> Cases() {
    yield return new TestCaseData(2, 3, 5);
    yield return new TestCaseData(0, 0, 0).SetName("Zeros");
}

// Combinatorial — every combination of values
[Test]
public void Combo([Values(1, 2)] int a, [Values(10, 20)] int b) { ... }   // 4 cases
```

### NUnit constraint model assertions

NUnit's `Assert.That(actual, Is...)` reads like English and is very expressive:

```csharp
Assert.That(result, Is.EqualTo(5));
Assert.That(list, Has.Count.EqualTo(3));
Assert.That(list, Does.Contain(42));
Assert.That(value, Is.GreaterThan(0).And.LessThan(100));
Assert.That(text, Does.StartWith("Hello").IgnoreCase);
Assert.That(() => Foo(), Throws.TypeOf<ArgumentException>());
Assert.That(collection, Is.All.GreaterThan(0));

// Multiple assertions
Assert.Multiple(() => {
    Assert.That(result.Count, Is.EqualTo(3));
    Assert.That(result.IsValid, Is.True);
});
```

A key NUnit quirk: by default it creates **one fixture instance for all tests** (state in fields persists unless reset in `[SetUp]`). This differs from xUnit's per-test instance.

---

## MSTest

```csharp
using Microsoft.VisualStudio.TestTools.UnitTesting;

[TestClass]
public class CalculatorTests {
    private Calculator _calc = null!;

    [TestInitialize]                  // before EACH test
    public void Setup() => _calc = new Calculator();

    [TestCleanup]                     // after EACH test
    public void Cleanup() { }

    [ClassInitialize]                 // once before all (must be static, takes TestContext)
    public static void ClassSetup(TestContext ctx) { }

    [TestMethod]
    public void Add_ReturnsSum() {
        Assert.AreEqual(5, _calc.Add(2, 3));
    }
}
```

### MSTest parameterized tests

```csharp
[TestMethod]
[DataRow(2, 3, 5)]
[DataRow(-1, 1, 0)]
public void Add(int a, int b, int expected) {
    Assert.AreEqual(expected, new Calculator().Add(a, b));
}

// Dynamic data
[TestMethod]
[DynamicData(nameof(Cases))]
public void Add_Dynamic(int a, int b, int expected) { ... }
public static IEnumerable<object[]> Cases => new[] {
    new object[] { 2, 3, 5 },
};
```

### MSTest assertions

```csharp
Assert.AreEqual(expected, actual);
Assert.AreNotEqual(a, b);
Assert.IsTrue(condition);
Assert.IsNull(obj);
Assert.AreSame(a, b);                  // reference equality
StringAssert.Contains(text, "sub");
CollectionAssert.Contains(list, item);
CollectionAssert.AreEqual(expected, actual);
Assert.ThrowsException<ArgumentException>(() => Foo());
await Assert.ThrowsExceptionAsync<TimeoutException>(() => BarAsync());
```

MSTest has separate `StringAssert`/`CollectionAssert` classes (xUnit and NUnit fold these into the main assert API).

---

## Choosing a framework

- **New projects / OSS** → **xUnit**. Clean lifecycle, parallel by default, widely adopted, the default in many templates.
- **Existing NUnit codebase / want rich constraint assertions / combinatorial tests** → **NUnit**.
- **Microsoft/enterprise shop, tight VS integration, or migrating legacy MSTest** → **MSTest** (now `MSTest` v3 / `Microsoft.Testing.Platform` is modern and fast).

All three integrate with `dotnet test`, CI, coverage tools, and IDEs. The choice rarely matters much — pick one and be consistent. Don't mix frameworks in one project.

---

## Migrating between them

The mechanical mapping (attributes/asserts) is straightforward, but watch the **lifecycle difference**: moving from NUnit/MSTest (shared fixture instance) to xUnit (per-test instance) can expose tests that secretly depended on shared field state. That's usually a *good* thing — it surfaces hidden coupling — but expect some failures to fix during migration.

---

## Common bugs / gotchas

### NUnit shared instance surprises

Because NUnit reuses one fixture instance, fields mutated by one test leak into the next unless reset in `[SetUp]`. xUnit avoids this with per-test instances.

### Forgetting `[TestClass]`/`[TestMethod]` in MSTest

MSTest requires both the class and method attributes; missing either silently excludes the test. xUnit needs neither beyond `[Fact]`.

### Static `[ClassInitialize]`/`[OneTimeSetUp]` state

Class-level setup runs once and its state is shared — mutating it from tests creates order dependence. Keep one-time setup read-only after init.

### Mixing assertion styles inconsistently

Within a project, pick one assertion approach (built-in, FluentAssertions, NUnit constraints) for readable, consistent tests.

---

## Summary

- xUnit, NUnit, MSTest are ~interchangeable; differ in attribute names, lifecycle, and assertion style.
- **xUnit**: constructor/`Dispose`, per-test instance, parallel by default — the modern default.
- **NUnit**: `[SetUp]`/`[TearDown]`, one fixture instance, expressive `Assert.That(...Is...)` constraints, combinatorial tests.
- **MSTest**: `[TestInitialize]`/`[TestCleanup]`, `[DataRow]`, separate `StringAssert`/`CollectionAssert`, strong VS integration.
- Pick one per project and stay consistent; the biggest migration gotcha is xUnit's per-test instance exposing shared-state coupling.

→ Next: [03-Mocking.md](03-Mocking.md)
