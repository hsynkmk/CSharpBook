# Chapter 17 — Best Practices — Q & A

---

### Q1. Casing for types, locals, private fields, constants?

Types/members/public constants → **PascalCase**. Locals/parameters → **camelCase**. Private fields → **`_camelCase`**. Note C# constants are PascalCase (`MaxRetries`), **not** `SCREAMING_SNAKE_CASE`.

---

### Q2. How are acronyms cased?

Two-letter acronyms all-caps (`IOStream`, `IPAddress`); three+ letters PascalCase (`HtmlParser`, `XmlReader`). So it's `HttpClient`, not `HTTPClient`.

---

### Q3. Why always use braces, even for single statements?

Prevents the "goto fail" class of bugs — adding a second line under a brace-less `if` silently changes control flow. Enforce `csharp_prefer_braces = true` in `.editorconfig`.

---

### Q4. When should comments be used?

To explain **why**, not what — non-obvious decisions, workarounds (with issue links), constraints, algorithm approach. Code should be self-documenting through names. Delete commented-out code; keep comments updated. Public APIs get XML docs.

---

### Q5. What's the first rule of performance optimization?

**Measure first.** Profile to find the actual hot path before optimizing. Most code should optimize for clarity; micro-optimization belongs only in the proven-hot 5%.

---

### Q6. Name three allocation-reducing idioms.

`Span<T>`/`ReadOnlySpan<T>` (slice without copying), `stackalloc` (small stack buffers), `ArrayPool<T>` (reuse larger buffers). Also `StringBuilder` for string building and `params ReadOnlySpan<T>` for zero-alloc variadics.

---

### Q7. Why avoid LINQ in hot paths (but not elsewhere)?

LINQ allocates delegates, closures, and iterator state machines per call. In a measured hot loop, a plain `foreach` is faster and allocation-free. Everywhere else, LINQ's readability wins — don't rewrite it without a measured reason.

---

### Q8. Why is a public API a "contract"?

Once published, every public member is a promise; changing it breaks consumers at compile or runtime. Keep the surface small (default `internal`/`sealed`), promote deliberately, and follow SemVer for breaking changes.

---

### Q9. Why avoid boolean parameters in public APIs?

They're unreadable at the call site (`Parse(true, false)` — what do they mean?). Use enums or options objects so the call self-documents (`Parse(ParseOptions.Strict)`).

---

### Q10. Where does `CancellationToken` go in a parameter list?

**Last**, with a `default` value. And you must **forward it** to every async call you make — accepting a token but not passing it makes cancellation a no-op (analyzer CA2016 flags this).

---

### Q11. When throw vs return an error (Try-pattern/result)?

**Throw** for caller bugs / broken invariants / unrecoverable conditions (rare). **Return** a result/Try-pattern for expected, routine failures (validation, parsing, not-found). Exceptions are for exceptional conditions, not control flow — they're expensive.

---

### Q12. Why prefer `throw;` over `throw ex;`?

`throw ex;` **resets the stack trace** to the rethrow location, hiding where the exception originated. `throw;` preserves the original stack. To add context, wrap: `throw new XException("...", ex)` with the inner exception.

---

### Q13. Why never `async void`?

It can't be awaited, so exceptions escape to the synchronization context (crashing the process) and callers can't observe completion. Use `async Task`. Only event handlers may be `async void` (with internal try/catch).

---

### Q14. `ConfigureAwait(false)` — where and why?

In **library** code, on awaits, to avoid capturing the synchronization context — improves performance and prevents UI/legacy-ASP.NET deadlocks. In **app/UI** code it's usually unnecessary (ASP.NET Core has no sync context) or undesirable (UI code wants to resume on the UI thread).

---

### Q15. What type should methods accept vs return for collections?

**Accept** the least-specific type you need (`IEnumerable<T>` if you only iterate). **Return** the most restrictive useful type (`IReadOnlyList<T>`) — never your internal mutable `List<T>`, which lets callers corrupt your state.

---

### Q16. Why return empty collections instead of null?

A null return forces null checks everywhere; one missed check is a `NullReferenceException`. Return `[]` / `Array.Empty<T>()` so callers can `foreach` safely and use `.Count == 0` as the "no results" signal.

---

### Q17. What's the difference between `IReadOnlyList<T>` and a true immutable snapshot?

`IReadOnlyList<T>` is a read-only **view** — the caller can't mutate through it, but if you mutate the underlying list, they see the change. For a true snapshot independent of later mutations, return `.ToArray()` or an `ImmutableArray<T>`.

---

### Q18. `ImmutableArray<T>` vs `ImmutableList<T>`?

`ImmutableArray<T>` is array-backed: fast reads/iteration, but every mutation copies the whole array (best for rarely-changed collections). `ImmutableList<T>` is tree-backed with structural sharing: O(log n) updates without full copies, but slower reads (best for frequently-updated shared collections).

---

### Q19. What is shallow immutability and why is it a trap?

A `record` (immutable) containing a mutable member (`List<T>`, array) isn't truly immutable — callers mutate the contained collection (`record.Items.Add(...)`). Use `IReadOnlyList<T>`/`ImmutableArray<T>` for members. Also, `with` does a shallow copy — nested mutable objects are shared.

---

### Q20. What are the modern guard-clause helpers?

`ArgumentNullException.ThrowIfNull(x)`, `ArgumentException.ThrowIfNullOrWhiteSpace(s)`, `ArgumentOutOfRangeException.ThrowIfNegative(n)`/`ThrowIfNegativeOrZero`, `ObjectDisposedException.ThrowIf(_disposed, this)`. They use `[CallerArgumentExpression]` to auto-fill the parameter name in the message.

---

### Q21. Where should you validate input?

At **boundaries** — public APIs, deserialization, user input (validate untrusted data thoroughly). Internally, where preconditions are already established, trust the input and use `Debug.Assert` to document invariants (removed in Release). Don't re-validate at every layer.

---

### Q22. What is primitive obsession and the fix?

Using primitives (`string`, `int`) for domain concepts — no type safety, easy argument transposition, scattered validation. Fix: wrap concepts in value objects/records (`AccountId`, `Money`) so the compiler prevents mixing them up.

---

### Q23. What is an anemic domain model?

An object that holds data but no behavior, with domain logic living in separate "service" classes — invariants aren't enforced on the entity. Fix: put behavior and invariants on the entity (rich model) so state can't become inconsistent. (Pure DTOs are a legitimate exception.)

---

### Q24. Why is the service-locator pattern an anti-pattern?

Dependencies are hidden (pulled from a global container inside methods) rather than declared in the constructor — you can't tell what a class needs without reading its body, and it's hard to test. Fix: constructor injection makes dependencies explicit and testable.

---

### Q25. How should you implement thread-safe lazy initialization?

Use `Lazy<T>` (thread-safe by default): `private static readonly Lazy<Service> _x = new(() => new Service());`. Avoid hand-rolled double-checked locking — it's error-prone (memory-model bugs).

---

### Q26. What's the fix for "disposing what you don't own"?

A `StreamReader` disposes its underlying stream by default. When you don't own the stream, pass `leaveOpen: true`. For `HttpClient`, don't `new` one per call (socket exhaustion) — use `IHttpClientFactory` or a shared instance.

---

### Q27. What does the Single Responsibility Principle actually mean?

"One reason to change" — a class should encapsulate one concern, so a change to one concern doesn't risk others. It's about **cohesion**, not "tiny classes everywhere": an orchestrating class is fine; what matters is that things changing for different reasons are separated.

---

### Q28. What's the classic Liskov Substitution Principle violation?

`Square : Rectangle` — overriding `Width`/`Height` setters to keep sides equal breaks code that sets them independently (`r.Width=5; r.Height=4` no longer gives area 20). Symptoms: overrides that throw `NotSupportedException`, strengthened preconditions, callers type-checking subtypes. Fix: model correctly (composition / common interface) when "is-a" doesn't truly hold.

---

### Q29. How does the Open/Closed Principle relate to `switch` statements?

A growing `switch` you must edit for every new case violates OCP (you modify tested code). Polymorphism/strategy lets you add a case by adding a class. But a `switch` over a **stable, closed** set (an enum that won't grow) is fine — don't force polymorphism on something that won't vary. Apply OCP where variation is real.

---

### Q30. What is the Interface Segregation Principle?

Clients shouldn't depend on methods they don't use. Prefer many small, focused interfaces over one fat one — a read-only consumer shouldn't depend on write methods. The BCL's `IReadOnlyList<T>`/`ICollection<T>`/`IList<T>` capability ladder models this.

---

### Q31. Which GoF patterns has modern C# made obsolete?

Iterator (`IEnumerable<T>` + `yield`), Singleton (`Lazy<T>` / DI), Observer (`event` / `IObservable<T>`), Command/Strategy (delegates), Prototype & data-Builder (`record` + `with` / `init`), simple Adapter (extension methods). Reach for a language feature before a pattern class.

---

### Q32. Which design patterns are still genuinely useful?

Factory (complex creation), Decorator (cross-cutting concerns like logging/caching/retry — e.g., `Stream` and ASP.NET middleware), Strategy when behavior is complex (a small interface), State machine, and Mediator at application scale. Use them only when they reduce complexity for a real problem.

---

### Q33. Why is constructor injection preferred over property/method injection?

The object is fully initialized after construction (no half-built state), dependencies are required and explicit (the constructor documents them), and they can be `readonly`/immutable. Property/method injection is only for genuinely optional or per-call dependencies.

---

### Q34. What is the composition root?

The single place at application startup where the dependency graph is wired (e.g., `Program.cs` registrations). Everywhere else, classes just declare dependencies and let them be supplied. Avoid `new`-ing collaborators or pulling from the container (service location) outside this root.

---

### Q35. Is dependency injection the same as a DI container?

No. DI is a **principle** — supply dependencies from outside, depend on abstractions. You can do "pure DI" by constructing the graph by hand (valid for small apps/libraries). A container just automates the graph and manages lifetimes/disposal for large apps.

---

### Q36. What is a captive dependency?

A longer-lived service capturing a shorter-lived one — e.g., a Singleton injecting a Scoped `DbContext`, holding it for the app's lifetime and breaking per-request semantics (and thread-safety). Rule: depend only on services of **equal or longer** lifetime; use a scope factory for the exception.

---

### Q37. When should you throw vs return a result?

**Throw** for caller bugs / broken invariants / unrecoverable conditions (rare, exceptional). **Return** a result/Try-pattern for expected, routine failures (validation, not-found, parse, business rejections). Exceptions are expensive and noisy — don't use them for control flow.

---

### Q38. Why `throw;` instead of `throw ex;`?

`throw ex;` resets the stack trace to the rethrow location, hiding the origin. `throw;` preserves the original stack. To add context, wrap with an inner exception: `throw new XException("context", ex)`.

---

### Q39. Where should errors be handled?

At strategic **boundaries** — a global handler (ASP.NET `UseExceptionHandler` → `ProblemDetails`), and integration edges (DB/HTTP/file) where failure is expected. Domain code mostly propagates. Catching and re-logging the same exception at every layer floods logs; **log once** at the handling boundary.

---

### Q40. What is the cardinal rule of equality and hashing?

If `a.Equals(b)` is true, `a.GetHashCode()` must equal `b.GetHashCode()`. Hash collections bucket by hash first, then compare with `Equals` within the bucket — so inconsistent hashes mean equal objects land in different buckets and are never matched. Override **both** `Equals` and `GetHashCode`, or neither.

---

### Q41. What's the easiest way to get value equality right?

Use a `record` (or `record struct`) — it synthesizes correct, consistent `Equals`/`GetHashCode`/`==`/`IEquatable<T>`. Reach for it before hand-writing equality. If hand-writing, implement `IEquatable<T>`, override `Equals`+`GetHashCode` (with `HashCode.Combine`), handle null.

---

### Q42. Why must hash keys be immutable?

`Dictionary`/`HashSet` place an entry in a bucket based on its hash at insertion time. If you mutate a field used in `GetHashCode` afterward, the hash changes and lookups search the wrong bucket — the entry is effectively lost and the collection is corrupted. Use immutable types (records, `readonly struct`) as keys.

---

### Q43. How do you customize equality without changing the type?

Supply an `IEqualityComparer<T>` to the collection or LINQ operator — e.g., `new Dictionary<string,int>(StringComparer.OrdinalIgnoreCase)` or `customers.Distinct(new ByEmailComparer())`. This decouples equality from the type and lets different contexts use different equality.

---

### Q44. What's the gotcha with records that have collection members?

Record value equality compares members with *their* `Equals`. A `List<T>`/array member compares **by reference**, so two records with equal-but-distinct lists are unequal. Use an immutable value-equatable member or override `Equals` to compare contents.

---

→ Next: [Coding.md](Coding.md)
